```mermaid
gitGraph
   commit id: "init"
   branch main
   checkout main
   commit id: "prod-ready N"
   commit id: "ongoing dev"

   branch feature-ABC-123
   checkout feature-ABC-123
   commit id: "work"
   commit id: "complete"

   checkout main
   branch release-1.0.x
   commit id: "stabilize"
   checkout release-1.0.x
   merge feature-ABC-123
   commit id: "rc-merge" tag: "rc-1"

   commit id: "RC builds"
   commit id: "v1.0.0" tag: "v1.0.0"

   checkout main
   merge release-1.0.x

   branch hotfix-1.0.1
   commit id: "hotfix change"
   
   checkout release-1.0.x
   merge hotfix-1.0.1
   commit id: "v1.0.1" tag: "v1.0.1"
   
   checkout main
   merge hotfix-1.0.1

   checkout main
   branch release-1.1.x
   commit id: "stabilize next"
```

```mermaid
flowchart TD
  A[Release branch build starts] --> B[Script queries git for existing tags]
  B --> C{"Determine next semver<br/>major.minor.patch"}
  C --> D[Create git tag for the build]
  D --> E[Create Docker image tag with same value]
  E --> F{Build type?}
  F -->|Release Candidate/Release| G["Use full version tag<br/>e.g. v1.2.0-rc.1 or v1.2.0"]
  F -->|Feature build| H["Use feature build tagging<br/>e.g. feature-shortsha"]
  B -.-> I[No version files - avoids merge conflicts]
```

```mermaid
flowchart LR
  subgraph Inputs
    O["Octopus Deploy API<br/>current PROD version"]
    GE["GitHub Environments<br/>version metadata"]
  end

  R["Release Branch<br/>e.g. release/1.0.x"] --> M{Select target env}
  M -->|PROD| P["Deploy N current"]
  M -->|Active| A[Deploy N+1]
  M -->|Future| F["Deploy N+2 single slot"]
  O --> M
  GE --> M

  P --> Z{After PROD deployment?}
  Z -->|Yes| X["Delete release branch<br/>tags preserved"]
  Z -->|No| R

  classDef meta fill:#f7f7f7,stroke:#bbb,color:#333
  class O,GE meta
```

```mermaid
flowchart TD
  A[Incident in PROD] --> B["Create hotfix branch<br/>from PROD tag e.g. v1.0.0"]
  B --> C["Commit fix + PR to release branches"]
  C --> D{Automation triggers}
  D -->|On hotfix merge| E[Create forward-merge PRs]
  D -->|ServiceNow ticket closed| E
  E --> F[Auto merge to main]
  E --> G["Auto open PRs to all<br/>active release branches"]
  F --> H[Prevents loss of fix in future releases]
  G --> H
```

## Alternative Chart Types

### Timeline Chart
```mermaid
timeline
    title Release Timeline
    section Q1 2025
        January : Feature Development
                 : DEV-INT Testing
        February : FUTURE Environment
                 : Partner UAT
        March    : STAGING Validation
                 : Production Release
    section Q2 2025
        April    : Hotfix v1.1.1
                 : Performance Optimization
        May      : Feature v2.0 Planning
                 : Architecture Review
        June     : v2.0 Development
                 : Integration Testing
```

### Gantt Chart
```mermaid
gantt
    title Release Schedule
    dateFormat  YYYY-MM-DD
    section Phase 1
    Development     :dev, 2025-01-01, 30d
    DEV-INT Testing :test1, after dev, 7d
    section Phase 2
    FUTURE Deploy   :future, after test1, 3d
    UAT Testing     :uat, after future, 14d
    section Phase 3
    STAGING Deploy  :staging, after uat, 3d
    Production      :prod, after staging, 1d
```

### Pie Chart
```mermaid
pie title Environment Usage Distribution
    "DEV-INT" : 35
    "FUTURE" : 15
    "ACTIVE" : 20
    "UAT" : 10
    "STAGING" : 10
    "PROD" : 10
```

### Mindmap
```mermaid
mindmap
  root((Deployment Strategy))
    Environments
      DEV-INT
        Auto-deploy
        Integration testing
        Fast feedback
      FUTURE
        Next version
        Feature testing
        Manual deploy
      ACTIVE
        Current stable
        Release candidate
        Manual deploy
      UAT
        Partner testing
        User validation
        Approval required
      STAGING
        Pre-production
        Final validation
        Approval required
      PROD
        Live environment
        Customer facing
        Multiple approvals
    Workflows
      Auto Deploy
        Push to main
        DEV-INT only
      Manual Deploy
        Any environment
        Validation required
      Promotion
        Between environments
        Approval gates
        Path validation
```

### State Diagram
```mermaid
stateDiagram-v2
    [*] --> DEV_INT : Push to main
    DEV_INT --> FUTURE : Manual deploy
    DEV_INT --> ACTIVE : Manual deploy
    FUTURE --> ACTIVE : Promote
    ACTIVE --> UAT : Promote (1 approval)
    ACTIVE --> STAGING : Promote (1 approval)
    UAT --> STAGING : Promote (1 approval)
    STAGING --> PROD : Promote (2+ approvals)
    PROD --> [*] : Stable
    
    note right of DEV_INT : Auto-deploy on main push
    note right of UAT : Partner testing
    note right of STAGING : Pre-production validation
    note right of PROD : Live customer environment
```

### Class Diagram
```mermaid
classDiagram
    class Environment {
        +String name
        +String branch
        +Boolean autoDeploy
        +Int approvalRequired
        +String purpose
        +deploy()
        +validate()
    }
    
    class Workflow {
        +String type
        +String trigger
        +List~Job~ jobs
        +run()
        +validate()
    }
    
    class Job {
        +String name
        +String runner
        +List~Step~ steps
        +String condition
        +execute()
    }
    
    class Step {
        +String name
        +String action
        +Map inputs
        +Map outputs
        +run()
    }
    
    class Promotion {
        +String sourceEnv
        +String targetEnv
        +String releaseId
        +Boolean approved
        +promote()
        +validate()
    }
    
    Environment ||--o{ Workflow : contains
    Workflow ||--o{ Job : executes
    Job ||--o{ Step : runs
    Environment ||--o{ Promotion : participates
```

### User Journey Map
```mermaid
journey
    title Developer Release Journey
    section Development
      Create feature branch    : 5: Developer
      Write code              : 4: Developer
      Create PR               : 3: Developer
      Code review             : 2: Developer
      Merge to main           : 5: Developer
    section Deployment
      Auto-deploy DEV-INT     : 5: System
      Test in DEV-INT         : 4: Developer
      Deploy to FUTURE        : 3: Developer
      Test next version       : 4: QA
      Promote to ACTIVE       : 3: Developer
    section Release
      Promote to UAT          : 2: QA Lead
      Partner testing         : 3: Partner
      Promote to STAGING      : 2: Senior Dev
      Final validation        : 4: QA
      Promote to PROD         : 1: Engineering Manager
      Monitor production      : 5: Operations
```

### Git Graph (Alternative Style)
```mermaid
gitGraph
    commit id: "Initial"
    branch develop
    checkout develop
    commit id: "Feature A"
    commit id: "Feature B"
    checkout main
    merge develop
    commit id: "Release v1.0" tag: "v1.0"
    branch hotfix
    checkout hotfix
    commit id: "Bug fix"
    checkout main
    merge hotfix
    commit id: "Hotfix v1.0.1" tag: "v1.0.1"
    checkout develop
    merge hotfix
    commit id: "Merge hotfix"
```

### Sequence Diagram
```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant CI as CI/CD
    participant DEV as DEV-INT
    participant FUT as FUTURE
    participant UAT as UAT
    participant PROD as PROD
    
    Dev->>GH: Push to main
    GH->>CI: Trigger workflow
    CI->>DEV: Auto-deploy
    DEV-->>Dev: Deploy complete
    
    Dev->>GH: Manual deploy FUTURE
    GH->>CI: Trigger workflow
    CI->>FUT: Deploy
    FUT-->>Dev: Deploy complete
    
    Dev->>GH: Promote FUTURE→UAT
    GH->>CI: Trigger promotion
    CI->>CI: Validate promotion
    CI->>CI: Wait for approval
    Note over CI: 1 reviewer required
    CI->>UAT: Deploy after approval
    UAT-->>Dev: Promotion complete
    
    Dev->>GH: Promote UAT→PROD
    GH->>CI: Trigger promotion
    CI->>CI: Validate promotion
    CI->>CI: Wait for approvals
    Note over CI: 2+ reviewers required
    CI->>PROD: Deploy after approval
    PROD-->>Dev: Production live
```

### ER Diagram (Entity Relationship)
```mermaid
erDiagram
    ENVIRONMENT ||--o{ DEPLOYMENT : has
    ENVIRONMENT ||--o{ PROMOTION : participates
    WORKFLOW ||--o{ JOB : contains
    JOB ||--o{ STEP : executes
    RELEASE ||--o{ DEPLOYMENT : creates
    USER ||--o{ APPROVAL : provides
    
    ENVIRONMENT {
        string name PK
        string branch
        boolean autoDeploy
        int approvalRequired
        string purpose
    }
    
    DEPLOYMENT {
        string id PK
        string environment FK
        string releaseId FK
        string status
        datetime deployedAt
    }
    
    PROMOTION {
        string id PK
        string sourceEnv FK
        string targetEnv FK
        string releaseId FK
        string status
        datetime requestedAt
    }
    
    WORKFLOW {
        string name PK
        string trigger
        string type
    }
    
    JOB {
        string name PK
        string workflow FK
        string runner
        string condition
    }
    
    STEP {
        string name PK
        string job FK
        string action
        json inputs
    }
    
    RELEASE {
        string id PK
        string version
        string branch
        string commit
    }
    
    USER {
        string username PK
        string email
        string role
    }
    
    APPROVAL {
        string id PK
        string promotionId FK
        string approver FK
        string status
        datetime approvedAt
    }
```

### Sankey Diagram
```mermaid
sankey-beta
    %%{wrap}%%
    DEV-INT,ACTIVE,100
    DEV-INT,FUTURE,50
    FUTURE,ACTIVE,40
    ACTIVE,UAT,60
    ACTIVE,STAGING,30
    UAT,STAGING,50
    STAGING,PROD,80