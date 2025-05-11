# Deactivation Flow

## Overview
The deactivation flow manages the process of deactivating users, subscriptions, and employers in the EmployeeSure system. This includes handling various types of deactivations, security measures, and associated business logic.

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
        DeactivationAPI[Deactivation API]
        SubscriptionAPI[Subscription API]
        AuthAPI[Auth API]
        ValidationAPI[Validation API]
        PolicyAPI[Policy API]
    end

    subgraph Service Layer
        DeactivationService[Deactivation Service]
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
    Web --> DeactivationAPI
    Mobile --> DeactivationAPI
    AdminPortal --> DeactivationAPI
    EmployerPortal --> DeactivationAPI

    %% API Layer Connections
    AuthAPI --> AuthService
    DeactivationAPI --> DeactivationService
    SubscriptionAPI --> SubscriptionService
    ValidationAPI --> ValidationService
    PolicyAPI --> PolicyService

    %% Service Layer Connections
    DeactivationService --> ValidationService
    SubscriptionService --> ValidationService
    PolicyService --> ValidationService
    DeactivationService --> NotificationService
    SubscriptionService --> NotificationService
    PolicyService --> NotificationService
    DeactivationService --> CacheService
    SubscriptionService --> CacheService
    PolicyService --> CacheService

    %% Service to Data Connections
    DeactivationService --> DB
    SubscriptionService --> DB
    PolicyService --> DB
    DeactivationService --> Cache
    SubscriptionService --> Cache
    PolicyService --> Cache
    DeactivationService --> Queue
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
    class DeactivationAPI,SubscriptionAPI,AuthAPI,ValidationAPI,PolicyAPI api
    class DeactivationService,SubscriptionService,NotificationService,AuthService,ValidationService,CacheService,PolicyService service
    class DB,S3,Cache,Queue data
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant OTP
    participant DeactivationService
    participant SubscriptionService
    participant Database
    participant NotificationService
    participant PaymentService

    %% OTP Generation Flow
    Client->>Server: POST /api/v1/deactivate/generate-otp
    Note over Client,Server: Request Body:<br/>{<br/>  "reason": "deactivation_reason",<br/>  "medium": "email/sms"<br/>}
    Server->>OTP: Generate OTP
    OTP-->>Server: OTP Generated
    Server->>NotificationService: Send OTP
    NotificationService-->>Client: OTP Delivered
    Server-->>Client: OTP Request ID
    Note over Server,Client: Response Body:<br/>{<br/>  "request_id": "otp_request_id",<br/>  "success": true<br/>}

    %% Deactivation Flow
    Client->>Server: POST /api/v1/deactivate
    Note over Client,Server: Request Body:<br/>{<br/>  "request_id": "otp_request_id",<br/>  "reason": "deactivation_reason",<br/>  "one_time_password": "otp",<br/>  "value": [deactivation_details]<br/>}
    Server->>OTP: Validate OTP
    OTP-->>Server: OTP Validated
    Server->>DeactivationService: Process Deactivation
    DeactivationService->>SubscriptionService: Deactivate Subscriptions
    SubscriptionService->>Database: Update Subscription Status
    Database-->>SubscriptionService: Status Updated
    DeactivationService->>PaymentService: Deactivate Payment
    PaymentService-->>DeactivationService: Payment Deactivated
    DeactivationService->>NotificationService: Send Deactivation Notification
    NotificationService-->>Client: Deactivation Notification
    Server-->>Client: Deactivation Confirmation
    Note over Server,Client: Response Body:<br/>{<br/>  "success": true,<br/>  "message": "Successfully deactivated"<br/>}
```

## API Endpoints

### Generate OTP
```http
POST /api/v1/deactivate/generate-otp
Content-Type: application/json

{
    "reason": "deactivation_reason",
    "medium": "email/sms"
}
```

### Deactivate
```http
POST /api/v1/deactivate
Content-Type: application/json

{
    "request_id": "otp_request_id",
    "reason": "deactivation_reason",
    "one_time_password": "otp",
    "value": [
        {
            "id": "user_id",
            "type": "user_type"
        }
    ]
}
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