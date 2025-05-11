# Billing Flow

## Overview
The billing flow handles subscription management, payment processing, and billing operations.

## High-Level Design

```mermaid
graph TB
    subgraph Client Layer
        Web[Web Client]
        Mobile[Mobile Client]
        AdminPortal[Admin Portal]
        EmployerPortal[Employer Portal]
    end

    subgraph API Layer
        BillingAPI[Billing API]
        SubscriptionAPI[Subscription API]
        AuthAPI[Auth API]
        ValidationAPI[Validation API]
        PolicyAPI[Policy API]
    end

    subgraph Service Layer
        BillingService[Billing Service]
        SubscriptionService[Subscription Service]
        NotificationService[Notification Service]
        AuthService[Auth Service]
        ValidationService[Validation Service]
        CacheService[Cache Service]
        PolicyService[Policy Service]
    end

    subgraph Data Layer
        DB[(MongoDB)]
        S3[(S3 Storage)]
        Cache[(Redis Cache)]
        Queue[(AWS SQS)]
    end

    %% Client Layer Connections
    Web --> AuthAPI
    Mobile --> AuthAPI
    AdminPortal --> AuthAPI
    EmployerPortal --> AuthAPI
    Web --> BillingAPI
    Mobile --> BillingAPI
    AdminPortal --> BillingAPI
    EmployerPortal --> BillingAPI
    Web --> SubscriptionAPI
    Mobile --> SubscriptionAPI
    AdminPortal --> SubscriptionAPI
    EmployerPortal --> SubscriptionAPI

    %% API Layer Connections
    AuthAPI --> AuthService
    BillingAPI --> BillingService
    SubscriptionAPI --> SubscriptionService
    ValidationAPI --> ValidationService
    PolicyAPI --> PolicyService

    %% Service Layer Connections
    BillingService --> ValidationService
    SubscriptionService --> ValidationService
    PolicyService --> ValidationService
    BillingService --> NotificationService
    SubscriptionService --> NotificationService
    PolicyService --> NotificationService
    BillingService --> CacheService
    SubscriptionService --> CacheService
    PolicyService --> CacheService

    %% Service to Data Connections
    BillingService --> DB
    SubscriptionService --> DB
    PolicyService --> DB
    BillingService --> Cache
    SubscriptionService --> Cache
    PolicyService --> Cache
    BillingService --> Queue
    SubscriptionService --> Queue
    PolicyService --> Queue

    %% Data Layer Connections
    Queue --> NotificationService
    CacheService --> Cache

    classDef client fill:#f9f,stroke:#333,stroke-width:2px
    classDef api fill:#bbf,stroke:#333,stroke-width:2px
    classDef service fill:#bfb,stroke:#333,stroke-width:2px
    classDef data fill:#fbb,stroke:#333,stroke-width:2px

    class Web,Mobile,AdminPortal,EmployerPortal client
    class BillingAPI,SubscriptionAPI,AuthAPI,ValidationAPI,PolicyAPI api
    class BillingService,SubscriptionService,NotificationService,AuthService,ValidationService,CacheService,PolicyService service
    class DB,S3,Cache,Queue data
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant BillingController
    participant PaymentGateway
    participant S3
    participant Database
    participant NotificationService

    %% Personal Bills Flow
    Client->>Server: GET /api/v1/billing/personal-bills
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token<br/>Query Parameters:<br/>- page: number<br/>- limit: number
    Server->>BillingController: Get Billing Details
    BillingController->>Database: Fetch Bills
    Database-->>BillingController: Bill Data
    BillingController-->>Client: Return Bills
    Note over BillingController,Client: Response Body:<br/>{<br/>  "bills": [{<br/>    "id": "bill_id",<br/>    "amount": "amount",<br/>    "status": "status"<br/>  }],<br/>  "total": "total_amount",<br/>  "page": "current_page"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% Payment Verification Flow
    Client->>Server: POST /api/v1/billing/verify-personal-bills-payment
    Note over Client,Server: Request Body:<br/>{<br/>  "invoice_id": "invoice_id",<br/>  "pay_hash": "payment_hash"<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>BillingController: Verify Payment
    BillingController->>PaymentGateway: Verify Payment Hash
    PaymentGateway-->>BillingController: Payment Status
    BillingController->>Database: Update Payment Status
    BillingController->>NotificationService: Send Payment Confirmation
    NotificationService-->>Client: Payment Notification
    BillingController-->>Client: Payment Confirmation
    Note over BillingController,Client: Response Body:<br/>{<br/>  "status": "success",<br/>  "message": "Payment verified"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% Wellness Invoice Flow
    Client->>Server: POST /api/v1/billing/generate-wellness-invoice
    Note over Client,Server: Request Body:<br/>{<br/>  "beneficiary_id": "beneficiary_id",<br/>  "amount": "amount",<br/>  "description": "description"<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>BillingController: Create Invoice
    BillingController->>Database: Store Invoice Data
    BillingController->>S3: Store Invoice Document
    S3-->>BillingController: Invoice URL
    BillingController->>NotificationService: Send Invoice Notification
    NotificationService-->>Client: Invoice Notification
    BillingController-->>Client: Invoice Details
    Note over BillingController,Client: Response Body:<br/>{<br/>  "invoice_id": "invoice_id",<br/>  "url": "invoice_url",<br/>  "amount": "amount"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% E-Invoicing Flow
    Client->>Server: GET /api/v1/billing/e-invoicing
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token<br/>Query Parameters:<br/>- start_date: date<br/>- end_date: date
    Server->>BillingController: Process E-Invoice
    BillingController->>Database: Fetch Invoice Data
    Database-->>BillingController: Invoice Data
    BillingController->>S3: Generate E-Invoice
    S3-->>BillingController: E-Invoice URL
    BillingController-->>Client: E-Invoice Details
    Note over BillingController,Client: Response Body:<br/>{<br/>  "invoices": [{<br/>    "id": "invoice_id",<br/>    "url": "invoice_url",<br/>    "amount": "amount"<br/>  }]<br/>}<br/>Headers:<br/>- Content-Type: application/json
```

## API Endpoints

### Personal Bills
```