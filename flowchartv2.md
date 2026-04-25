# 🧪 ChemWatch — System Flowcharts

## 1. System Architecture Overview

```mermaid
graph TB
    subgraph Client["🌐 Client Browser"]
        React["React 19 SPA<br/>Vite 8 + TypeScript"]
    end

    subgraph Docker["🐳 Docker Compose"]
        subgraph Frontend["Frontend Container"]
            Vite["Vite Dev Server :5173"]
            Nginx["Nginx :80 (prod)"]
        end

        subgraph Backend["Backend Container"]
            API[".NET 10 Web API :5000"]
            EF["Entity Framework Core 9"]
            JWT["JWT Auth Middleware"]
            Swagger["Swagger UI"]
        end

        subgraph Database["Database Container"]
            PG["PostgreSQL 16 Alpine"]
            InitDB["initdb/01_restore.sql"]
        end

        subgraph Backup["Backup Container"]
            Cron["Cron Scheduler"]
            BkScript["backup.sh / rotate.sh"]
        end
    end

    React -->|"HTTP/REST (Axios)"| Vite
    Vite -->|"Proxy /api/*"| API
    Nginx -->|"Proxy /api/*"| API
    API --> JWT
    JWT --> EF
    EF -->|"Npgsql"| PG
    InitDB -.->|"First run only"| PG
    Cron --> BkScript
    BkScript -->|"pg_dump"| PG

    style Client fill:#1e293b,stroke:#6366f1,color:#e2e8f0
    style Docker fill:#0f172a,stroke:#334155,color:#e2e8f0
    style Frontend fill:#1e1b4b,stroke:#818cf8,color:#e2e8f0
    style Backend fill:#172554,stroke:#60a5fa,color:#e2e8f0
    style Database fill:#14532d,stroke:#4ade80,color:#e2e8f0
    style Backup fill:#431407,stroke:#fb923c,color:#e2e8f0
```

---

## 2. Authentication & Authorization Flow

```mermaid
flowchart TD
    Start([User visits app]) --> CheckToken{JWT token<br/>in localStorage?}

    CheckToken -->|No| LoginPage["Login Page"]
    CheckToken -->|Yes| ValidateToken{Token valid<br/>& not expired?}

    ValidateToken -->|No| LoginPage
    ValidateToken -->|Yes| PostLogin["Post-Login Loading<br/>(fetch user profile)"]

    LoginPage -->|"POST /api/auth/login"| AuthAPI["Auth Controller"]
    AuthAPI --> VerifyPW{BCrypt verify<br/>password?}
    VerifyPW -->|❌ Fail| LoginError["Show error message"]
    LoginError --> LoginPage
    VerifyPW -->|✅ Pass| GenerateJWT["Generate JWT Token<br/>(userId, role, email)"]
    GenerateJWT --> PostLogin

    PostLogin --> CheckRole{User Role?}

    CheckRole -->|Admin| FullAccess["Full Access<br/>All modules + Admin panel"]
    CheckRole -->|Focal Point| ApproverAccess["Approver Access<br/>All modules + Check-In + QR + Approvals"]
    CheckRole -->|User| UserAccess["Basic Access<br/>Catalog, Cart, Orders, Checkout, Reports"]

    FullAccess --> Dashboard["Dashboard Page"]
    ApproverAccess --> Dashboard
    UserAccess --> Dashboard

    subgraph ProtectedRoute["🔐 ProtectedRoute Component"]
        RouteGuard{Has required role?}
        RouteGuard -->|Yes| RenderPage["Render Page"]
        RouteGuard -->|No| Unauthorized["Unauthorized Page"]
    end

    Dashboard --> ProtectedRoute

    style Start fill:#6366f1,stroke:#818cf8,color:#fff
    style Dashboard fill:#22c55e,stroke:#4ade80,color:#fff
    style LoginError fill:#ef4444,stroke:#f87171,color:#fff
    style Unauthorized fill:#ef4444,stroke:#f87171,color:#fff
```

---

## 3. Order Workflow (Purchase Request Lifecycle)

```mermaid
flowchart TD
    Browse["👤 Browse Catalog"] -->|Add to cart| Cart["🛒 Cart Page"]
    Cart -->|Submit order| CreatePR["POST /api/orders<br/>Create Purchase Request"]

    CreatePR --> PRStatus["Status: pending_approval"]

    PRStatus --> ApprovalQueue["📋 Approval Queue<br/>(Admin / Focal Point)"]

    ApprovalQueue --> ReviewPR{Review Items}

    ReviewPR -->|"Approve"| Approved["Status: approved"]
    ReviewPR -->|"Reject"| Rejected["Status: rejected"]
    ReviewPR -->|"Modify (change qty/items)"| Modified["Status: modified"]

    Modified -->|"Requester reviews"| ReSubmit{Accept changes?}
    ReSubmit -->|Yes| PRStatus
    ReSubmit -->|No| Rejected

    Approved -->|"Assign PO Number"| POAssigned["Status: po_assigned"]
    POAssigned -->|"Mark as ordered"| Ordered["Status: ordered"]
    Ordered -->|"Delivery arrives"| PendingDelivery["📦 Pending Delivery Page"]

    PendingDelivery -->|"Check in items"| CheckIn["Pending Delivery Check-In<br/>POST /api/checkin/pending-delivery"]
    CheckIn --> CreateLot["Create Inventory Lot<br/>(lot#, qty, expiry, lab)"]
    CreateLot --> StockTx["Record Stock Transaction<br/>(type: check_in)"]
    StockTx --> UpdatePR["Update PR Item Status to received"]
    UpdatePR --> Complete["Status: received ✅"]

    Rejected --> End["❌ Order Closed"]

    style Browse fill:#6366f1,stroke:#818cf8,color:#fff
    style Complete fill:#22c55e,stroke:#4ade80,color:#fff
    style Rejected fill:#ef4444,stroke:#f87171,color:#fff
    style End fill:#ef4444,stroke:#f87171,color:#fff
    style ApprovalQueue fill:#f59e0b,stroke:#fbbf24,color:#000
```

---

## 4. Inventory Lifecycle (Check-In → Checkout)

```mermaid
flowchart TD
    subgraph CheckInMethods["📥 Check-In Methods"]
        Manual["Manual Check-In<br/>(Admin/Focal Point)"]
        PendingDel["Pending Delivery<br/>(from approved order)"]
        QRScan["QR Code Scan<br/>(scan label → auto check-in)"]
    end

    Manual --> CreateLot
    PendingDel --> CreateLot
    QRScan --> CreateLot

    CreateLot["Create Inventory Lot"] --> ActiveLot["Active Lot<br/>(item, lot#, qty, lab, expiry)"]

    ActiveLot --> Monitor{Monitoring}

    Monitor -->|"Check expiry"| ExpiryCheck{Days until expiry?}
    ExpiryCheck -->|"> 30 days"| Safe["✅ Safe"]
    ExpiryCheck -->|"≤ 30 days"| NearExpire["⚠️ Near Expire"]
    ExpiryCheck -->|"Expired"| Expired["❌ Expired"]

    Monitor -->|"Check peroxide"| PeroxideCheck{Peroxide-forming<br/>chemical?}
    PeroxideCheck -->|Yes| PeroxideTrack["Peroxide Tracking<br/>(test schedule)"]
    PeroxideCheck -->|No| Regular["Regular monitoring"]

    PeroxideTrack --> TestDue{Test due?}
    TestDue -->|Yes| RecordTest["Record Peroxide Test<br/>(pass/fail/quarantine)"]
    TestDue -->|No| WaitSchedule["Wait for schedule"]

    Monitor -->|"Check stock level"| StockCheck{Qty vs Min Stock?}
    StockCheck -->|"Above min"| StockOK["✅ Stock OK"]
    StockCheck -->|"Below min"| LowStock["🔴 Low Stock Alert"]

    ActiveLot --> Checkout["Checkout / Consume<br/>POST /api/checkout"]
    Checkout --> DeductQty["Deduct Quantity"]
    DeductQty --> RecordTx["Record Stock Transaction<br/>(type: check_out)"]
    RecordTx --> CheckEmpty{Remaining qty = 0?}
    CheckEmpty -->|Yes| Depleted["Lot Depleted"]
    CheckEmpty -->|No| ActiveLot

    NearExpire -->|"Admin extends"| ExtendShelfLife["Extend Shelf Life<br/>POST /api/shelflife"]
    ExtendShelfLife --> ActiveLot

    style CreateLot fill:#6366f1,stroke:#818cf8,color:#fff
    style ActiveLot fill:#22c55e,stroke:#4ade80,color:#fff
    style Expired fill:#ef4444,stroke:#f87171,color:#fff
    style LowStock fill:#ef4444,stroke:#f87171,color:#fff
    style Depleted fill:#64748b,stroke:#94a3b8,color:#fff
```

---

## 5. QR Code Workflow

```mermaid
flowchart LR
    subgraph Generate["🏷️ Generate"]
        Admin["Admin / Focal Point"]
        Admin -->|"POST /api/qrcode/generate"| GenQR["Generate QR Code<br/>(encodes lot ID + URL)"]
        GenQR --> Label["Print Label<br/>/qr/print/:lotId"]
        GenQR --> BatchPrint["Batch Print<br/>/qr/print-batch"]
    end

    subgraph Scan["📱 Scan"]
        ScanQR["Scan QR Label"]
        ScanQR -->|"Opens URL"| ScanPage["/qr/scan/:lotId"]
        ScanPage --> AutoFill["Auto-fill lot info"]
        AutoFill --> ConfirmCheckIn["Confirm Check-In/Status Update"]
        ConfirmCheckIn -->|"POST /api/qrcode/scan"| LogScan["Log QR Scan<br/>+ Update Lot"]
    end

    Label --> ScanQR
    BatchPrint --> ScanQR

    style Generate fill:#1e1b4b,stroke:#818cf8,color:#e2e8f0
    style Scan fill:#172554,stroke:#60a5fa,color:#e2e8f0
```

---

## 6. Peroxide Safety Monitoring Flow

```mermaid
flowchart TD
    NewLot["Chemical Lot Created"] --> IsPeroxide{Requires<br/>Peroxide Monitoring?}

    IsPeroxide -->|No| NormalTrack["Normal inventory tracking"]
    IsPeroxide -->|Yes| AddToPeroxide["Add to Peroxide Tracking<br/>/monitoring/peroxide"]

    AddToPeroxide --> CalcSchedule["Calculate Test Schedule<br/>(based on PeroxideConfigRule)"]

    CalcSchedule --> Tracking["Active Peroxide Lot"]

    Tracking --> CheckDue{Test overdue?}
    CheckDue -->|"Not yet"| WaitIcon["⏳ Wait for test date"]
    CheckDue -->|"Due / Overdue"| AlertDue["🔴 Peroxide Due Alert<br/>(Dashboard card)"]

    AlertDue --> PerformTest["Perform Peroxide Test"]
    WaitIcon -.->|"Date arrives"| AlertDue

    PerformTest --> TestResult{Test Result?}
    TestResult -->|"Pass (< threshold)"| RecordPass["✅ Record: Pass<br/>Next test date calculated"]
    TestResult -->|"Fail (≥ threshold)"| RecordFail["❌ Record: Fail"]

    RecordPass --> Tracking
    RecordFail --> Quarantine["🟡 Quarantine Lot"]

    Quarantine --> Decision{Admin Decision}
    Decision -->|"Dispose"| Dispose["Dispose / Remove"]
    Decision -->|"Retest"| PerformTest

    style NewLot fill:#6366f1,stroke:#818cf8,color:#fff
    style AlertDue fill:#ef4444,stroke:#f87171,color:#fff
    style RecordPass fill:#22c55e,stroke:#4ade80,color:#fff
    style Quarantine fill:#f59e0b,stroke:#fbbf24,color:#000
    style Dispose fill:#64748b,stroke:#94a3b8,color:#fff
```

---

## 7. Deployment Pipeline (CI/CD)

```mermaid
flowchart LR
    subgraph Trigger["🔫 Trigger"]
        Push["Git Push"]
    end

    subgraph CI["⚙️ GitHub Actions (ci-cd.yml)"]
        Build["Build<br/>Docker images"]
        Test["Test<br/>dotnet test"]
        Deploy["Deploy<br/>to target env"]
    end

    subgraph DeployScript["🚀 deploy.sh"]
        Backup["1. Pre-deploy backup"]
        Pull["2. Pull/build images"]
        Migrate["3. Run DB migrations"]
        Rolling["4. Rolling restart"]
        Health["5. Health check"]
    end

    subgraph Prod["🏭 Production (Blue-Green)"]
        NginxProd["Nginx"]
        Blue["Backend Blue"]
        Green["Backend Green"]
        DB["PostgreSQL"]
    end

    Push -->|"main branch"| CI
    Push -->|"develop branch"| CI
    CI --> Build --> Test --> Deploy
    Deploy --> DeployScript
    DeployScript --> Backup --> Pull --> Migrate --> Rolling --> Health

    Rolling -->|"Switch upstream"| NginxProd
    NginxProd --> Blue
    NginxProd -.->|"standby"| Green
    Blue --> DB
    Green --> DB

    style Trigger fill:#6366f1,stroke:#818cf8,color:#fff
    style CI fill:#1e1b4b,stroke:#818cf8,color:#e2e8f0
    style DeployScript fill:#172554,stroke:#60a5fa,color:#e2e8f0
    style Prod fill:#14532d,stroke:#4ade80,color:#e2e8f0
```

---

## 8. Frontend Page Navigation Map

```mermaid
flowchart TD
    Login["/login"] -->|"Auth success"| Loading["/auth/entering"]
    Loading --> Dashboard["/  Dashboard"]

    Dashboard --> Orders
    Dashboard --> Inventory
    Dashboard --> Monitoring
    Dashboard --> Reports
    Dashboard --> Admin

    subgraph Orders["📦 Orders"]
        Catalog["/orders/catalog"]
        Cart["/orders/cart"]
        MyOrders["/orders/my-orders"]
        Approval["/orders/approval-queue 🔒"]
    end

    subgraph Inventory["🏗️ Inventory"]
        PendingDel["/inventory/check-in/pending-delivery"]
        ManualCI["/inventory/check-in/manual 🔒"]
        Checkout["/inventory/checkout"]
        Lots["/inventory/lots"]
        Transactions["/inventory/transactions"]
        QRCodes["/admin/qr-codes 🔒"]
        ExtendSL["/inventory/extend-shelf-life 🔒"]
    end

    subgraph Monitoring["⚠️ Monitoring"]
        Peroxide["/monitoring/peroxide"]
    end

    subgraph Reports["📊 Reports"]
        OrderStatus["/reports/orders"]
        MinStock["/reports/min-stock"]
        Expired["/reports/expired"]
        PeroxideDue["/reports/peroxide-due"]
        AuditLog["/reports/transactions"]
        Regulatory["/reports/regulatory"]
    end

    subgraph Admin["🔐 Admin (Admin only)"]
        Users["/admin/users"]
        CreateUser["/admin/users/create"]
        EditUser["/admin/users/:id/edit"]
        Locations["/admin/locations"]
        Roles["/admin/roles"]
        Vendors["/admin/vendors"]
        Categories["/admin/categories"]
        Items["/admin/items"]
        LabSettings["/admin/item-lab-settings"]
        PO["/admin/po-references"]
    end

    style Login fill:#6366f1,stroke:#818cf8,color:#fff
    style Dashboard fill:#22c55e,stroke:#4ade80,color:#fff
    style Orders fill:#1e1b4b,stroke:#818cf8,color:#e2e8f0
    style Inventory fill:#172554,stroke:#60a5fa,color:#e2e8f0
    style Monitoring fill:#431407,stroke:#fb923c,color:#e2e8f0
    style Reports fill:#14532d,stroke:#4ade80,color:#e2e8f0
    style Admin fill:#4c0519,stroke:#f43f5e,color:#e2e8f0
```

> 🔒 = Requires `admin` or `focal_point` role

---

## 9. Data Model (Entity Relationships)

```mermaid
erDiagram
    %% ══════════════════════════════════════════════════
    %% MASTER DATA
    %% ══════════════════════════════════════════════════

    Role {
        uuid Id PK
        string Name UK
        string DisplayName
    }

    Location {
        uuid Id PK
        string Name
        string Code UK
    }

    Lab {
        uuid Id PK
        uuid LocationId FK
        string Name
        string Code
    }

    User {
        uuid Id PK
        string Email UK
        string FullName
        uuid RoleId FK
        string LocationScopeType
    }

    UserLocation {
        uuid Id PK
        uuid UserId FK
        uuid LocationId FK
    }

    Vendor {
        uuid Id PK
        string Name UK
        string Code
    }

    ItemCategory {
        uuid Id PK
        string Name UK
        string Code UK
    }

    RegulatoryType {
        uuid Id PK
        string Name UK
        string Code UK
    }

    PeroxideConfigRule {
        uuid Id PK
        string PeroxideClass UK "alternate key"
        bool TestBeforeUse
    }

    Item {
        uuid Id PK
        string ItemName
        uuid CategoryId FK
        uuid RegulatoryTypeId FK
        uuid DefaultVendorId FK
        string PeroxideClass FK
        decimal TotalMinStock
    }

    ItemLabSetting {
        uuid Id PK
        uuid ItemId FK
        uuid LabId FK
        decimal MinStock
    }

    PoReference {
        uuid Id PK
        string PoNumber UK
        uuid CategoryId FK
        uuid LabId FK
        uuid VendorId FK
    }

    %% ══════════════════════════════════════════════════
    %% INVENTORY CORE
    %% ══════════════════════════════════════════════════

    InventoryLot {
        uuid Id PK
        uuid ItemId FK
        uuid LabId FK
        uuid LocationId FK
        uuid VendorId FK
        string LotNumber
        int QuantityReceived
        int QuantityRemaining
        datetime ExpiryDate
        string Status
    }

    StockTransaction {
        uuid Id PK
        string TransactionType
        uuid UserId FK
        uuid LotId FK
        uuid ItemId FK
        decimal QuantityBase
        string Type
    }

    %% ══════════════════════════════════════════════════
    %% ORDER WORKFLOW
    %% ══════════════════════════════════════════════════

    PurchaseRequest {
        uuid Id PK
        string PoNumber
        uuid PoReferenceId FK
        uuid LabId FK
        uuid RequestedBy FK
        string Status
    }

    PurchaseRequestItem {
        uuid Id PK
        uuid PurchaseRequestId FK
        uuid ItemId FK
        uuid VendorId FK
        decimal QuantityOrdered
        string Status
    }

    PurchaseRequestItemRevision {
        uuid Id PK
        uuid PurchaseRequestItemId FK
        uuid PurchaseRequestId FK
        string Action
        uuid RevisedBy FK
    }

    %% ══════════════════════════════════════════════════
    %% MONITORING & TRACKING
    %% ══════════════════════════════════════════════════

    PeroxideTest {
        uuid Id PK
        uuid InventoryLotId FK
        datetime TestDate
        uuid TestedByUserId FK
        string ResultType
        decimal PpmResult
        string Classification
    }

    ShelfLifeExtension {
        uuid Id PK
        uuid InventoryLotId FK
        int ExtensionDays
        string Reason
        uuid AuthorizedByUserId FK
    }

    QrScanLog {
        uuid Id PK
        uuid InventoryLotId FK
        uuid ScannedByUserId FK
        string Action
    }

    %% ══════════════════════════════════════════════════
    %% RELATIONSHIPS
    %% ══════════════════════════════════════════════════

    Role ||--o{ User : "has many"
    Location ||--o{ Lab : "has many"
    User ||--o{ UserLocation : "has many"
    Location ||--o{ UserLocation : "has many"

    ItemCategory ||--o{ Item : "has many"
    RegulatoryType ||--o{ Item : "has many"
    Vendor ||--o{ Item : "default vendor"
    PeroxideConfigRule ||--o{ Item : "PeroxideClass FK"

    Item ||--o{ ItemLabSetting : "has many"
    Lab ||--o{ ItemLabSetting : "has many"

    ItemCategory ||--o| PoReference : "optional"
    Lab ||--o| PoReference : "optional"
    Vendor ||--o| PoReference : "optional"

    Item ||--o{ InventoryLot : "has many"
    Lab ||--o{ InventoryLot : "has many"
    Location ||--o{ InventoryLot : "has many"
    Vendor ||--o{ InventoryLot : "optional"

    InventoryLot ||--o{ StockTransaction : "has many"
    User ||--o{ StockTransaction : "performed by"

    Lab ||--o{ PurchaseRequest : "for lab"
    User ||--o{ PurchaseRequest : "requested by"
    PoReference ||--o| PurchaseRequest : "optional"

    PurchaseRequest ||--o{ PurchaseRequestItem : "has many"
    Item ||--o{ PurchaseRequestItem : "for item"

    PurchaseRequest ||--o{ PurchaseRequestItemRevision : "has many"

    InventoryLot ||--o{ PeroxideTest : "has many"
    InventoryLot ||--o{ ShelfLifeExtension : "has many"
    InventoryLot ||--o{ QrScanLog : "has many"
```

> **Note:** For a fully detailed ER diagram with all columns and constraints, refer to the `er_diagram.md` artifact document.

---

## 10. Request-Response Flow (API Lifecycle)

```mermaid
sequenceDiagram
    participant Browser as 🌐 React SPA
    participant Axios as Axios Client
    participant API as .NET Web API
    participant JWT as JWT Middleware
    participant Controller as Controller
    participant EF as EF Core
    participant DB as PostgreSQL

    Browser->>Axios: User action (e.g., submit order)
    Axios->>Axios: Attach JWT from localStorage
    Axios->>API: HTTP Request + Bearer Token

    API->>JWT: Validate token
    alt Token Invalid/Expired
        JWT-->>Axios: 401 Unauthorized
        Axios-->>Browser: Redirect to /login
    end

    JWT->>Controller: Authenticated request
    Controller->>Controller: Check role authorization
    alt Insufficient Role
        Controller-->>Axios: 403 Forbidden
        Axios-->>Browser: Show Unauthorized page
    end

    Controller->>EF: Query/Mutate data
    EF->>DB: SQL via Npgsql
    DB-->>EF: Result set
    EF-->>Controller: Entity objects
    Controller-->>API: JSON response
    API-->>Axios: HTTP Response
    Axios-->>Browser: Update UI state
```
