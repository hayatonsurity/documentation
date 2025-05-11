# Deactivation Flow

## Overview
The deactivation flow manages the process of deactivating users, subscriptions, and policies in the EmployeeSure system.

## High-Level Design

```mermaid
graph TB
    %% Client Layer
    subgraph ClientLayer[Client Layer]
        Web[Web Client]
        Mobile[Mobile Client]
        AdminPortal[Admin Portal]
        EmployerPortal[Employer Portal]
    end

    %% API Layer
    subgraph APILayer[API Layer]
        DeactivationAPI[Deactivation API]
        ValidationAPI[Validation API]
        AuthAPI[Auth API]
        PolicyAPI[Policy API]
        BillingAPI[Billing API]
        DocumentAPI[Document API]
    end

    %% Service Layer
    subgraph ServiceLayer[Service Layer]
        subgraph CoreServices[Core Services]
            DeactivationService[Deactivation Service]
            PolicyService[Policy Service]
            BillingService[Billing Service]
        end
        
        subgraph InternalServices[Internal Support Services]
            ValidationService[Validation Service]
            AuthService[Auth Service]
            CacheService[Cache Service]
            AuditService[Audit Service]
        end

        subgraph ExternalServices[External Services]
            NotificationService[Notification Service]
            DocumentService[Document Service]
            PaymentService[Payment Service]
        end
    end

    %% Data Layer
    subgraph DataLayer[Data Layer]
        DB[(MongoDB)]
        S3[(S3 Storage)]
        Cache[(Redis Cache)]
        Queue[(AWS SQS)]
    end

    %% Client to API Connections
    Web --> DeactivationAPI
    Web --> AuthAPI
    Web --> ValidationAPI
    Web --> PolicyAPI
    Web --> BillingAPI
    Web --> DocumentAPI
    Mobile --> DeactivationAPI
    Mobile --> AuthAPI
    Mobile --> ValidationAPI
    Mobile --> PolicyAPI
    Mobile --> BillingAPI
    Mobile --> DocumentAPI
    AdminPortal --> DeactivationAPI
    AdminPortal --> AuthAPI
    AdminPortal --> ValidationAPI
    AdminPortal --> PolicyAPI
    AdminPortal --> BillingAPI
    AdminPortal --> DocumentAPI
    EmployerPortal --> DeactivationAPI
    EmployerPortal --> AuthAPI
    EmployerPortal --> ValidationAPI
    EmployerPortal --> PolicyAPI
    EmployerPortal --> BillingAPI
    EmployerPortal --> DocumentAPI

    %% API to Service Connections
    DeactivationAPI --> DeactivationService
    ValidationAPI --> ValidationService
    AuthAPI --> AuthService
    PolicyAPI --> PolicyService
    BillingAPI --> BillingService
    DocumentAPI --> DocumentService

    %% Core Service Connections
    DeactivationService --> ValidationService
    DeactivationService --> NotificationService
    DeactivationService --> PolicyService
    DeactivationService --> BillingService
    DeactivationService --> CacheService
    DeactivationService --> DocumentService
    DeactivationService --> AuditService
    DeactivationService --> PaymentService
    PolicyService --> ValidationService
    PolicyService --> CacheService
    PolicyService --> DocumentService
    PolicyService --> AuditService
    BillingService --> ValidationService
    BillingService --> CacheService
    BillingService --> NotificationService
    BillingService --> AuditService
    BillingService --> PaymentService

    %% Service to Data Connections
    DeactivationService --> DB
    DeactivationService --> Queue
    PolicyService --> DB
    PolicyService --> Cache
    BillingService --> DB
    BillingService --> Queue
    ValidationService --> DB
    ValidationService --> Cache
    DocumentService --> DB
    DocumentService --> S3
    AuditService --> DB
    AuditService --> S3
    CacheService --> Cache

    %% Styling
    classDef client fill:#f9f,stroke:#333,stroke-width:2px
    classDef api fill:#bbf,stroke:#333,stroke-width:2px
    classDef coreService fill:#bfb,stroke:#333,stroke-width:2px
    classDef internalService fill:#fbb,stroke:#333,stroke-width:2px
    classDef externalService fill:#ff9,stroke:#333,stroke-width:2px
    classDef data fill:#ddd,stroke:#333,stroke-width:2px

    class Web,Mobile,AdminPortal,EmployerPortal client
    class DeactivationAPI,ValidationAPI,AuthAPI,PolicyAPI,BillingAPI,DocumentAPI api
    class DeactivationService,PolicyService,BillingService coreService
    class ValidationService,AuthService,CacheService,AuditService internalService
    class NotificationService,DocumentService,PaymentService externalService
    class DB,S3,Cache,Queue data
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant DeactivationController
    participant ValidationService
    participant SubscriptionService
    participant Database
    participant NotificationService
    participant DocumentService

    %% Deactivation Request Flow
    Client->>Server: POST /api/v1/deactivation/request
    Note over Client,Server: Request Body:<br/>{<br/>  "user_id": "user_id",<br/>  "reason": "reason",<br/>  "effective_date": "date",<br/>  "documents": ["document_urls"]<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>DeactivationController: Process Request
    DeactivationController->>ValidationService: Validate Request
    ValidationService-->>DeactivationController: Validation Result
    DeactivationController->>SubscriptionService: Check Active Subscriptions
    SubscriptionService-->>DeactivationController: Subscription Status
    DeactivationController->>Database: Store Request
    Database-->>DeactivationController: Request Stored
    DeactivationController->>DocumentService: Process Documents
    DocumentService-->>DeactivationController: Document Processing
    DeactivationController->>NotificationService: Send Request Notification
    NotificationService-->>Client: Request Confirmation
    DeactivationController-->>Client: Request Status
    Note over DeactivationController,Client: Response Body:<br/>{<br/>  "request_id": "request_id",<br/>  "status": "status",<br/>  "message": "request_message"<br/>}

    %% Get Deactivation Status Flow
    Client->>Server: GET /api/v1/deactivation/:id/status
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token
    Server->>DeactivationController: Get Status
    DeactivationController->>Database: Fetch Request
    Database-->>DeactivationController: Request Data
    DeactivationController-->>Client: Status Response
    Note over DeactivationController,Client: Response Body:<br/>{<br/>  "request_id": "request_id",<br/>  "status": "status",<br/>  "details": {<br/>    "reason": "reason",<br/>    "effective_date": "date",<br/>    "created_at": "timestamp"<br/>  }<br/>}

    %% Deactivation Approval Flow
    Client->>Server: POST /api/v1/deactivation/:id/approve
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token
    Server->>DeactivationController: Process Approval
    DeactivationController->>ValidationService: Validate Approval
    ValidationService-->>DeactivationController: Validation Result
    DeactivationController->>SubscriptionService: Cancel Subscriptions
    SubscriptionService-->>DeactivationController: Cancellation Confirmation
    DeactivationController->>Database: Update Status
    Database-->>DeactivationController: Status Updated
    DeactivationController->>NotificationService: Send Approval Notification
    NotificationService-->>Client: Approval Notification
    DeactivationController-->>Client: Approval Status
    Note over DeactivationController,Client: Response Body:<br/>{<br/>  "request_id": "request_id",<br/>  "status": "status",<br/>  "message": "approval_message"<br/>}

    %% Deactivation Rejection Flow
    Client->>Server: POST /api/v1/deactivation/:id/reject
    Note over Client,Server: Request Body:<br/>{<br/>  "reason": "rejection_reason"<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>DeactivationController: Process Rejection
    DeactivationController->>ValidationService: Validate Rejection
    ValidationService-->>DeactivationController: Validation Result
    DeactivationController->>Database: Update Status
    Database-->>DeactivationController: Status Updated
    DeactivationController->>NotificationService: Send Rejection Notification
    NotificationService-->>Client: Rejection Notification
    DeactivationController-->>Client: Rejection Status
    Note over DeactivationController,Client: Response Body:<br/>{<br/>  "request_id": "request_id",<br/>  "status": "status",<br/>  "message": "rejection_message"<br/>}

    %% Cancel Deactivation Request Flow
    Client->>Server: POST /api/v1/deactivation/:id/cancel
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token
    Server->>DeactivationController: Process Cancellation
    DeactivationController->>ValidationService: Validate Cancellation
    ValidationService-->>DeactivationController: Validation Result
    DeactivationController->>Database: Update Status
    Database-->>DeactivationController: Status Updated
    DeactivationController->>NotificationService: Send Cancellation Notification
    NotificationService-->>Client: Cancellation Notification
    DeactivationController-->>Client: Cancellation Status
    Note over DeactivationController,Client: Response Body:<br/>{<br/>  "request_id": "request_id",<br/>  "status": "status",<br/>  "message": "cancellation_message"<br/>}
```

## API Endpoints

### Request Deactivation
```http
POST /api/v1/deactivation/request
Content-Type: application/json
Authorization: Bearer <jwt_token>

{
    "user_id": "string",
    "reason": "string",
    "effective_date": "date",
    "documents": ["string"]
}
```

### Approve Deactivation
```http
POST /api/v1/deactivation/:id/approve
Authorization: Bearer <jwt_token>
```

### Reject Deactivation
```http
POST /api/v1/deactivation/:id/reject
Content-Type: application/json
Authorization: Bearer <jwt_token>

{
    "reason": "string"
}
```

### Get Deactivation Status
```http
GET /api/v1/deactivation/:id/status
Authorization: Bearer <jwt_token>
```

### List Deactivation Requests
```http
GET /api/v1/deactivation/requests
Authorization: Bearer <jwt_token>
Query Parameters:
- status: string (optional)
- user_id: string (optional)
- start_date: date (optional)
- end_date: date (optional)
```

### Cancel Deactivation Request
```http
POST /api/v1/deactivation/:id/cancel
Authorization: Bearer <jwt_token>
```

## Data Models

### Deactivation Request
```javascript
{
    request_id: String,
    reason: String,
    one_time_password: String,
    value: Array,
    created_at: Date,
    updated_at: Date
}
```

### Deactivation Log
```javascript
{
    deactivation_id: String,
    user_id: String,
    type: String,
    reason: String,
    status: String,
    created_at: Date,
    updated_at: Date
}
```

## Security Considerations

1. **OTP Verification**
   - OTP generation and validation
   - OTP expiration handling
   - Rate limiting on OTP requests

2. **Access Control**
   - Role-based access control
   - Permission validation
   - Audit logging

3. **Data Security**
   - Secure data transmission
   - Data encryption
   - Session management

4. **API Security**
   - Rate limiting
   - Input validation
   - Error handling

## Error Handling

### Common Error Codes
- 400: Bad Request - Invalid input
- 401: Unauthorized - Invalid OTP
- 403: Forbidden - Insufficient permissions
- 429: Too Many Requests - Rate limit exceeded
- 500: Internal Server Error - Server-side issues

### Error Response Format
```javascript
{
    "status": "error",
    "code": "ERROR_CODE",
    "message": "Error description",
    "details": {} // Optional additional details
}
```

## Integration Points

1. **Payment System**
   - Payment deactivation
   - Refund processing
   - Payment status updates

2. **Notification System**
   - OTP delivery
   - Deactivation notifications
   - Status updates

3. **Cache System**
   - Session management
   - Data caching
   - Cache invalidation

4. **Activity Logging**
   - Audit trails
   - Activity tracking
   - Log management

## Best Practices

1. **Deactivation Process**
   - Validate all dependencies
   - Handle partial deactivations
   - Maintain data consistency

2. **Security**
   - Implement proper authentication
   - Use secure communication
   - Follow security guidelines

3. **Performance**
   - Optimize database queries
   - Use caching effectively
   - Handle concurrent requests

4. **Monitoring**
   - Track deactivation metrics
   - Monitor system performance
   - Log important events
``` 