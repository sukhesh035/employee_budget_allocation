# Employee Budget Allocation Platform - Architecture Diagrams

## 1. System Context Diagram (C4 Level 1)

The platform sits at the center, serving four user roles and integrating with two external systems.

```mermaid
flowchart TB
    subgraph Actors
        EMP["Employee\n(View own comp & position)"]
        MGR["Manager\n(View subtree, propose budgets)"]
        HR["HR Admin\n(Manage employees, comp, CSV import)"]
        FIN["Finance Admin\n(Approve budgets, view rollups)"]
    end

    subgraph External Systems
        AUTH0["Auth0\n(SSO / OAuth 2.0 OIDC)"]
        SPLIT["Split.io\n(Feature Flags)"]
    end

    PLATFORM["Employee Budget\nAllocation Platform"]

    EMP -->|HTTPS| PLATFORM
    MGR -->|HTTPS| PLATFORM
    HR -->|HTTPS| PLATFORM
    FIN -->|HTTPS| PLATFORM

    PLATFORM -->|OAuth 2.0 PKCE| AUTH0
    PLATFORM -->|SDK polling| SPLIT
```

---

## 2. Container Diagram (C4 Level 2)

All external traffic enters through the NestJS BFF. The .NET API is never publicly exposed. Writes use normalized tables; reads use materialized views.

```mermaid
flowchart TB
    USER["Browser"]

    subgraph Frontend ["Frontend (Module Federation)"]
        SHELL["Shell App\n(Layout, routing, error boundaries)\nServed from S3/CloudFront"]
        MFE1["MFE Hierarchy\n(Org tree visualization)"]
        MFE2["MFE Compensation\n(Compensation management)"]
        MFE3["MFE Budget\n(Budget allocation & tracking)"]
        MFE4["MFE Admin\n(Employee CRUD, CSV import)"]
        SHELL --> MFE1 & MFE2 & MFE3 & MFE4
    end

    subgraph Backend ["Backend (EKS)"]
        BFF["NestJS BFF\n(Auth guard, RBAC guard,\ncircuit breaker, HMAC signing)"]
        API[".NET 9 API\n(MediatR 12 CQRS,\nHMAC verify, domain events)"]
    end

    subgraph Data
        PG["PostgreSQL 17\n(ltree, RLS, outbox table,\nmaterialized views)"]
        REDIS["ElastiCache Redis 7.4\n(Cache-aside, stale fallback)"]
    end

    subgraph Async
        S3["S3\n(CSV uploads)"]
        SNS["SNS\n(Domain event topics)"]
        SQS["SQS\n(Fan-out queues + DLQ)"]
    end

    USER -->|HTTPS| SHELL
    SHELL -->|HTTPS REST| BFF
    BFF -->|HTTP + HMAC headers| API
    BFF -->|GET/SET| REDIS
    API -->|EF Core| PG
    API -->|Publish| SNS
    SNS -->|Fan-out| SQS
    SQS -->|Consume| API
    API -->|Presigned URL| S3
    SHELL -->|Upload| S3
```

---

## 3. Infrastructure Diagram (AWS)

Multi-AZ deployment inside a VPC with public and private subnets. WAF protects the edge; all backend services live in private subnets.

```mermaid
flowchart TB
    INET["Internet"]
    WAF["AWS WAF"]
    CF["CloudFront CDN"]
    S3_STATIC["S3\n(MFE static assets:\nshell/, mfe-hierarchy/,\nmfe-compensation/, mfe-budget/,\nmfe-admin/)"]
    APIGW["API Gateway"]

    subgraph VPC ["VPC (10.0.0.0/16)"]
        subgraph PUB ["Public Subnets (3 AZs)"]
            NAT["NAT Gateway"]
            ALB["Application Load Balancer"]
        end

        subgraph PRIV ["Private Subnets (3 AZs)"]
            subgraph EKS ["EKS Cluster"]
                NEST_POD["NestJS BFF Pods\n(HPA, 2-6 replicas)"]
                DOTNET_POD[".NET API Pods\n(HPA, 2-6 replicas)"]
            end
            RDS_PRI["RDS PostgreSQL 17\n(Primary, AZ-1)"]
            RDS_REP["RDS PostgreSQL 17\n(Read Replica, AZ-2)"]
            EC_PRI["ElastiCache Redis\n(Primary, AZ-1)"]
            EC_REP["ElastiCache Redis\n(Replica, AZ-2)"]
        end
    end

    INET --> WAF --> CF
    CF --> S3_STATIC
    CF --> APIGW
    APIGW --> ALB
    ALB --> NEST_POD
    NEST_POD --> DOTNET_POD
    DOTNET_POD --> RDS_PRI
    RDS_PRI -.->|Replication| RDS_REP
    NEST_POD --> EC_PRI
    EC_PRI -.->|Replication| EC_REP
    DOTNET_POD --> NAT
    NAT --> INET
```

---

## 4. Data Flow Diagrams

### 4a. Authentication Flow

PKCE flow with Auth0, followed by JWT validation at BFF and HMAC-signed internal headers verified by the .NET API. PostgreSQL RLS provides the final enforcement layer.

```mermaid
sequenceDiagram
    actor User
    participant SPA as Shell + MFE
    participant Auth0
    participant BFF as NestJS BFF
    participant API as .NET API
    participant DB as PostgreSQL

    User->>SPA: Click "Sign In"
    SPA->>Auth0: Authorization request (PKCE)
    Auth0->>User: Login page
    User->>Auth0: Credentials
    Auth0->>SPA: Authorization code
    SPA->>Auth0: Exchange code for tokens
    Auth0->>SPA: Access token + ID token (JWT)

    SPA->>BFF: API request + Bearer JWT
    BFF->>BFF: Validate JWT (AuthGuard)
    BFF->>BFF: Check RBAC (RbacGuard)
    BFF->>BFF: Sign headers with HMAC<br/>(X-User-Id, X-User-Role, X-Subtree-Ids)
    BFF->>API: Forward request + HMAC-signed headers

    API->>API: Verify HMAC signature
    API->>API: Re-validate RBAC (.NET policy)
    API->>DB: SET app.current_user_id = '{userId}'
    API->>DB: Execute query (RLS enforced)
    DB-->>API: Filtered results
    API-->>BFF: Response
    BFF-->>SPA: Response
    SPA-->>User: Rendered data
```

### 4b. Hierarchy Tree Query Flow

Cache-aside pattern: check Redis first, fall through to the .NET API on miss, populate cache after. Circuit breaker serves stale cache if the .NET API is unavailable.

```mermaid
sequenceDiagram
    actor User
    participant SPA as Shell + MFE
    participant BFF as NestJS BFF
    participant Redis as ElastiCache Redis
    participant API as .NET API
    participant DB as PostgreSQL

    User->>SPA: View org tree
    SPA->>BFF: GET /hierarchy/tree?rootId=123
    BFF->>Redis: GET hierarchy:tree:123
    alt Cache HIT
        Redis-->>BFF: Cached tree JSON
    else Cache MISS
        Redis-->>BFF: null
        BFF->>API: GET /internal/hierarchy/tree?rootId=123
        API->>DB: SELECT * FROM employee_tree_view<br/>WHERE path <@ '123'
        DB-->>API: Tree rows
        API-->>BFF: Tree response
        BFF->>Redis: SET hierarchy:tree:123 (TTL 5min)
    end
    BFF-->>SPA: Tree JSON
    SPA-->>User: Rendered org chart
```

### 4c. Compensation Update Flow

Compensation records are append-only. After the write, a domain event is published via the transactional outbox. SNS fans out to multiple SQS consumers for async processing.

```mermaid
sequenceDiagram
    actor HR as HR Admin
    participant SPA as Shell + MFE
    participant BFF as NestJS BFF
    participant API as .NET API
    participant DB as PostgreSQL
    participant SNS
    participant SQS_MV as SQS: MV Refresh
    participant SQS_CACHE as SQS: Cache Invalidation
    participant SQS_AUDIT as SQS: Audit Log

    HR->>SPA: Submit compensation change
    SPA->>BFF: POST /employees/456/compensation
    BFF->>BFF: Validate JWT + RBAC + HMAC sign
    BFF->>API: POST /internal/employees/456/compensation

    API->>DB: BEGIN transaction
    API->>DB: INSERT INTO compensation (append-only)
    API->>DB: INSERT INTO outbox_events
    API->>DB: COMMIT

    Note over API,DB: Outbox publisher polls outbox table

    API->>SNS: Publish CompensationUpdated event
    API->>DB: Mark outbox event as published

    par Fan-out
        SNS->>SQS_MV: CompensationUpdated
        SQS_MV->>DB: REFRESH MATERIALIZED VIEW
    and
        SNS->>SQS_CACHE: CompensationUpdated
        SQS_CACHE->>BFF: Invalidate Redis keys
    and
        SNS->>SQS_AUDIT: CompensationUpdated
        SQS_AUDIT->>DB: INSERT INTO audit_log
    end

    API-->>BFF: 201 Created
    BFF-->>SPA: Success
    SPA-->>HR: Confirmation
```

### 4d. CSV Import Flow

Large CSV imports use S3 presigned URLs to avoid BFF memory pressure. An S3 event triggers async processing.

```mermaid
sequenceDiagram
    actor HR as HR Admin
    participant SPA as Shell + MFE
    participant BFF as NestJS BFF
    participant API as .NET API
    participant S3
    participant Consumer as .NET Consumer
    participant DB as PostgreSQL
    participant SNS

    HR->>SPA: Select CSV file
    SPA->>BFF: POST /imports/presigned-url
    BFF->>API: POST /internal/imports/presigned-url
    API->>S3: Generate presigned PUT URL
    S3-->>API: Presigned URL
    API-->>BFF: Presigned URL
    BFF-->>SPA: Presigned URL

    SPA->>S3: PUT CSV file (direct upload)
    S3-->>SPA: 200 OK

    S3->>Consumer: S3 Event Notification
    Consumer->>S3: GET CSV file
    Consumer->>Consumer: Parse & validate rows
    Consumer->>DB: Batch INSERT employees/compensation
    Consumer->>SNS: Publish CSVImportCompleted event

    SNS->>BFF: Notification via SQS
    BFF->>SPA: WebSocket: import complete
    SPA-->>HR: Import results summary
```

---

## 5. RBAC Visibility Diagram

Managers see only their subtree. In this example, "Manager B" (highlighted) can see their direct reports and all descendants, but cannot see peers, other subtrees, or upward in the hierarchy.

```mermaid
flowchart TB
    CEO["CEO\n(hidden)"]:::hidden
    VP1["VP Engineering\n(hidden)"]:::hidden
    VP2["VP Sales\n(hidden)"]:::hidden

    DIR1["Director Platform\n(hidden)"]:::hidden
    DIR2["Director Product\n(hidden)"]:::hidden

    MGR_A["Manager A\n(hidden)"]:::hidden
    MGR_B["Manager B\n(CURRENT USER)"]:::current

    IC1["IC: Alice\n(hidden)"]:::hidden
    IC2["IC: Bob\n(hidden)"]:::hidden
    IC3["IC: Carol\n(visible)"]:::visible
    IC4["IC: Dave\n(visible)"]:::visible
    IC5["IC: Eve\n(visible)"]:::visible

    LEAD["Tech Lead: Frank\n(visible)"]:::visible
    IC6["IC: Grace\n(visible)"]:::visible
    IC7["IC: Hank\n(visible)"]:::visible

    CEO --> VP1 & VP2
    VP1 --> DIR1 & DIR2
    VP2 --> MGR_A
    DIR1 --> MGR_B
    DIR2 --> IC1 & IC2
    MGR_A --> IC1 & IC2
    MGR_B --> IC3 & IC4 & LEAD
    LEAD --> IC5 & IC6
    IC3 ~~~ IC7
    MGR_B --> IC5

    classDef hidden fill:#f5f5f5,stroke:#ccc,color:#bbb
    classDef current fill:#1a73e8,stroke:#1557b0,color:#fff
    classDef visible fill:#34a853,stroke:#1e8e3e,color:#fff
```

---

## 6. Event-Driven Architecture Diagram

All domain events flow through the transactional outbox to guarantee at-least-once delivery. SNS fans out to SQS queues with dedicated DLQs for each consumer.

```mermaid
flowchart LR
    API[".NET API\n(Domain Event Raised)"]
    OUTBOX["outbox_events table\n(PostgreSQL)"]
    PUBLISHER["Outbox Publisher\n(Background Service)"]
    SNS_TOPIC["SNS Topic:\nDomainEvents"]

    subgraph "SQS Fan-out"
        SQS1["SQS: MaterializedViewRefresh"]
        SQS2["SQS: CacheInvalidation"]
        SQS3["SQS: AuditLog"]
        SQS4["SQS: NotificationService"]
    end

    subgraph DLQs ["Dead Letter Queues"]
        DLQ1["DLQ: MV Refresh"]
        DLQ2["DLQ: Cache"]
        DLQ3["DLQ: Audit"]
        DLQ4["DLQ: Notification"]
    end

    subgraph Consumers
        C1["MV Refresh Consumer\nREFRESH MATERIALIZED VIEW"]
        C2["Cache Consumer\nDEL Redis keys"]
        C3["Audit Consumer\nINSERT audit_log"]
        C4["Notification Consumer\nEmail / WebSocket"]
    end

    API -->|"BEGIN tx: write + outbox INSERT, COMMIT"| OUTBOX
    PUBLISHER -->|Poll & publish| OUTBOX
    PUBLISHER -->|Publish| SNS_TOPIC

    SNS_TOPIC --> SQS1 & SQS2 & SQS3 & SQS4

    SQS1 --> C1
    SQS2 --> C2
    SQS3 --> C3
    SQS4 --> C4

    SQS1 -.->|After 3 retries| DLQ1
    SQS2 -.->|After 3 retries| DLQ2
    SQS3 -.->|After 3 retries| DLQ3
    SQS4 -.->|After 3 retries| DLQ4
```

---

## 7. CI/CD Pipeline Diagram

PRs trigger CI checks. Merges to `develop`, `release/*`, or `main` trigger environment-specific CD pipelines with progressively stricter deployment strategies.

```mermaid
flowchart LR
    subgraph CI ["CI (GitHub Actions)"]
        PR["PR Created"]
        LINT["Lint\n(ESLint, dotnet format)"]
        TEST_UNIT["Unit Tests\n(Jest + xUnit)"]
        TEST_INT["Integration Tests\n(Testcontainers)"]
        TEST_CONTRACT["Contract Tests\n(Pact)"]
        SCAN["Security Scan\n(Snyk, Trivy)"]
        COV["Coverage Check\n(80% biz / 60% overall)"]
    end

    subgraph CD_TEST ["deploy-test.yml"]
        MERGE_DEV["Merge to develop"]
        BUILD_T["Build Docker images"]
        PUSH_T["Push to ECR\n(test-{sha})"]
        DEPLOY_T["Deploy to test EKS\n(direct rollout)"]
    end

    subgraph CD_BETA ["deploy-beta.yml"]
        MERGE_REL["Merge to release/*"]
        BUILD_B["Build Docker images"]
        PUSH_B["Push to ECR\n(beta-{sha})"]
        CANARY_B["Argo Canary\n50%→100%"]
    end

    subgraph CD_PROD ["deploy-prod.yml"]
        MERGE_MAIN["Merge to main"]
        APPROVE["Manual Approval\n(2 reviewers)"]
        BUILD_P["Build Docker images"]
        PUSH_P["Push to ECR\n(prod-{sha})"]
        CANARY_P["Argo Canary\n20%→40%→80%→100%\n+ analysis gates"]
        ROLLBACK["Auto-rollback"]
    end

    PR --> LINT --> TEST_UNIT --> TEST_INT --> TEST_CONTRACT --> SCAN --> COV

    MERGE_DEV --> BUILD_T --> PUSH_T --> DEPLOY_T
    MERGE_REL --> BUILD_B --> PUSH_B --> CANARY_B
    MERGE_MAIN --> APPROVE --> BUILD_P --> PUSH_P --> CANARY_P
    CANARY_P -->|Fail| ROLLBACK
```

---

## 8. Deployment Topology

Sizing varies by environment. The diagram below shows the **prod** topology (3 AZs, read replica, multi-AZ Redis). **test** uses a single AZ with 2 EKS nodes; **beta** uses 2 AZs with 3 nodes. See CLAUDE.md for the full per-environment config comparison.

```mermaid
flowchart TB
    ALB["Application Load Balancer\n(Cross-AZ)"]

    subgraph AZ1 ["Availability Zone 1"]
        NEST1["NestJS BFF\nPod x2"]
        DOTNET1[".NET API\nPod x2"]
        RDS_P["RDS PostgreSQL\n(PRIMARY)\ndb.r6g.xlarge"]
        EC_P["ElastiCache Redis\n(PRIMARY)\n3-node cluster"]
    end

    subgraph AZ2 ["Availability Zone 2"]
        NEST2["NestJS BFF\nPod x2"]
        DOTNET2[".NET API\nPod x2"]
        RDS_R["RDS PostgreSQL\n(READ REPLICA)"]
        EC_R["ElastiCache Redis\n(REPLICA)"]
    end

    subgraph AZ3 ["Availability Zone 3"]
        NEST3["NestJS BFF\nPod x2"]
        DOTNET3[".NET API\nPod x2"]
    end

    ALB --> NEST1 & NEST2 & NEST3
    NEST1 --> DOTNET1
    NEST2 --> DOTNET2
    NEST3 --> DOTNET3

    DOTNET1 --> RDS_P
    DOTNET2 --> RDS_P
    DOTNET3 --> RDS_P

    DOTNET2 -.->|Reads| RDS_R
    DOTNET3 -.->|Reads| RDS_R

    NEST1 --> EC_P
    NEST2 --> EC_P
    NEST3 --> EC_P

    RDS_P -.->|Async replication| RDS_R
    EC_P -.->|Async replication| EC_R
```

---

## 8b. Environment Promotion Flow

Code flows from feature branches through three environments with progressively stricter gates before reaching production.

```mermaid
flowchart LR
    FB["Feature Branch"]
    DEV_BR["develop branch"]
    REL_BR["release/* branch"]
    MAIN_BR["main branch"]

    TEST_ENV["test env\n(direct deploy)"]
    BETA_ENV["beta env\n(canary 50%→100%)"]
    PROD_ENV["prod env\n(canary 20%→40%→80%→100%\n+ analysis gates)"]

    FB -->|PR + CI checks| DEV_BR
    DEV_BR -->|auto-deploy| TEST_ENV
    DEV_BR -->|cut release branch| REL_BR
    REL_BR -->|auto-deploy| BETA_ENV
    REL_BR -->|PR to main| MAIN_BR
    MAIN_BR -->|manual approval\n(2 reviewers)| PROD_ENV
```

---

## 9. Caching Architecture

Three cache layers with distinct TTLs. Invalidation flows from domain events through SQS to clear Redis, which in turn causes fresh reads from the materialized view.

```mermaid
flowchart TB
    USER["Browser"]
    CF["CloudFront\nEdge Cache\n(TTL: static assets only)"]
    BFF["NestJS BFF"]
    REDIS["ElastiCache Redis\nApplication Cache\n(TTL: 5 min)"]
    API[".NET API"]
    MV["PostgreSQL\nMaterialized View\n(Refreshed on events)"]
    TABLES["PostgreSQL\nNormalized Tables"]

    USER -->|1. Request| CF
    CF -->|2. Cache MISS for API calls| BFF
    BFF -->|3. GET cache key| REDIS

    REDIS -->|4a. HIT| BFF
    REDIS -->|4b. MISS| BFF

    BFF -->|5. Forward on MISS| API
    API -->|6. Query| MV
    API -->|7. Response| BFF
    BFF -->|8. SET cache key| REDIS
    BFF -->|9. Response| USER

    subgraph Invalidation ["Invalidation Flow"]
        EVT["Domain Event\n(via SQS)"]
        INV_REDIS["DEL Redis keys\n(pattern-based)"]
        INV_MV["REFRESH MATERIALIZED VIEW\nCONCURRENTLY"]
    end

    EVT --> INV_REDIS
    EVT --> INV_MV
    INV_MV --> MV
    TABLES -->|Source data| MV
```

---

## 10. Database Schema Diagram (ER)

Core tables with `ltree` for hierarchy. Compensation is append-only (no UPDATEs or DELETEs). The `employee_tree_view` materialized view denormalizes hierarchy + compensation for fast reads.

```mermaid
erDiagram
    departments {
        uuid id PK
        varchar name
        uuid parent_department_id FK
        ltree path
        uuid budget_owner_id FK
        timestamp created_at
        timestamp updated_at
    }

    employees {
        uuid id PK
        varchar first_name
        varchar last_name
        varchar email UK
        varchar auth0_id UK
        uuid department_id FK
        uuid manager_id FK
        ltree path
        varchar role "employee | manager | hr_admin | finance_admin"
        boolean is_active
        timestamp hire_date
        timestamp created_at
        timestamp updated_at
    }

    compensation {
        uuid id PK
        uuid employee_id FK
        decimal base_salary
        decimal bonus_target_pct
        decimal equity_units
        varchar currency
        date effective_date
        varchar change_reason
        uuid changed_by FK
        timestamp created_at
    }

    budgets {
        uuid id PK
        uuid department_id FK
        int fiscal_year
        varchar quarter
        decimal total_amount
        decimal allocated_amount
        decimal remaining_amount
        varchar status "draft | pending | approved | rejected"
        uuid proposed_by FK
        uuid approved_by FK
        timestamp created_at
        timestamp updated_at
    }

    audit_log {
        uuid id PK
        uuid user_id FK
        varchar entity_type
        uuid entity_id
        varchar action
        jsonb old_values
        jsonb new_values
        varchar correlation_id
        inet ip_address
        timestamp created_at
    }

    outbox_events {
        uuid id PK
        varchar event_type
        jsonb payload
        varchar status "pending | published | failed"
        int retry_count
        timestamp created_at
        timestamp published_at
    }

    employee_tree_view {
        uuid employee_id
        varchar full_name
        ltree path
        int depth
        uuid department_id
        varchar department_name
        decimal current_salary
        decimal bonus_target_pct
        decimal total_comp
        int direct_report_count
        decimal subtree_budget
    }

    employees ||--o{ employees : "manager_id"
    departments ||--o{ employees : "department_id"
    departments ||--o{ departments : "parent_department_id"
    employees ||--o{ compensation : "employee_id"
    departments ||--o{ budgets : "department_id"
    employees ||--o{ audit_log : "user_id"
    employees ||--o{ budgets : "proposed_by"
    employees ||--o{ budgets : "approved_by"
    employees ||--o{ compensation : "changed_by"
```
