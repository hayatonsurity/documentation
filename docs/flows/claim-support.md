# Claim Support Flow

## Overview
The claim support flow in EmployeeSure handles the submission, verification, and settlement of insurance claims.

## High-Level Design

```mermaid
graph TB
    subgraph Client Layer
        Web[Web Client]
        Mobile[Mobile Client]
        AdminPortal[Admin Portal]
        HospitalPortal[Hospital Portal]
    end

    subgraph API Layer
        ClaimAPI[Claim API]
        DocumentAPI[Document API]
        AuthAPI[Auth API]
        BeneficiaryAPI[Beneficiary API]
        PolicyAPI[Policy API]
    end

    subgraph Service Layer
        ClaimService[Claim Service]
        DocumentService[Document Service]
        NotificationService[Notification Service]
        AuthService[Auth Service]
        ValidationService[Validation Service]
        CacheService[Cache Service]
        BillingService[Billing Service]
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
    HospitalPortal --> AuthAPI
    Web --> ClaimAPI
    Mobile --> ClaimAPI
    AdminPortal --> ClaimAPI
    HospitalPortal --> ClaimAPI
    Web --> BeneficiaryAPI
    Mobile --> BeneficiaryAPI
    AdminPortal --> BeneficiaryAPI
    HospitalPortal --> BeneficiaryAPI

    %% API Layer Connections
    AuthAPI --> AuthService
    ClaimAPI --> ClaimService
    ClaimAPI --> DocumentAPI
    ClaimAPI --> PolicyAPI
    DocumentAPI --> DocumentService
    BeneficiaryAPI --> ClaimService
    PolicyAPI --> ClaimService

    %% Service Layer Connections
    ClaimService --> ValidationService
    DocumentService --> ValidationService
    ClaimService --> DB
    ClaimService --> Cache
    DocumentService --> S3
    ClaimService --> Queue
    DocumentService --> Queue
    ClaimService --> NotificationService
    DocumentService --> NotificationService
    ClaimService --> CacheService
    DocumentService --> CacheService
    ClaimService --> BillingService
    DocumentService --> BillingService

    %% Data Layer Connections
    Queue --> NotificationService
    CacheService --> Cache

    classDef client fill:#f9f,stroke:#333,stroke-width:2px
    classDef api fill:#bbf,stroke:#333,stroke-width:2px
    classDef service fill:#bfb,stroke:#333,stroke-width:2px
    classDef data fill:#fbb,stroke:#333,stroke-width:2px

    class Web,Mobile,AdminPortal,HospitalPortal client
    class ClaimAPI,DocumentAPI,AuthAPI,BeneficiaryAPI,PolicyAPI api
    class ClaimService,DocumentService,NotificationService,AuthService,ValidationService,CacheService,BillingService service
    class DB,S3,Cache,Queue data
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant ClaimController
    participant HospitalController
    participant Database
    participant NotificationService
    participant PaymentService
    participant DocumentService

    %% Claim Submission Flow
    Client->>Server: POST /api/v1/claim/submit
    Note over Client,Server: Request Body:<br/>{<br/>  "policy_id": "policy_id",<br/>  "hospital_id": "hospital_id",<br/>  "treatment_type": "treatment_type",<br/>  "amount": "amount",<br/>  "documents": ["document_urls"]<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>ClaimController: Submit Claim
    ClaimController->>HospitalController: Validate Hospital
    HospitalController-->>ClaimController: Hospital Validation
    ClaimController->>Database: Store Claim
    ClaimController->>DocumentService: Store Documents
    DocumentService-->>ClaimController: Document Confirmation
    ClaimController->>NotificationService: Send Claim Notification
    NotificationService-->>Client: Claim Submission Notification
    ClaimController-->>Client: Claim Details
    Note over ClaimController,Client: Response Body:<br/>{<br/>  "claim_id": "claim_id",<br/>  "status": "status",<br/>  "document_urls": ["urls"],<br/>  "message": "submission_message"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% Claim Verification Flow
    Client->>Server: POST /api/v1/claim/:id/verify
    Note over Client,Server: Request Body:<br/>{<br/>  "verification_status": "status",<br/>  "verification_notes": "notes"<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>ClaimController: Verify Claim
    ClaimController->>Database: Update Claim Status
    Database-->>ClaimController: Update Confirmation
    ClaimController->>NotificationService: Send Verification Notification
    NotificationService-->>Client: Verification Notification
    ClaimController-->>Client: Verification Status
    Note over ClaimController,Client: Response Body:<br/>{<br/>  "claim_id": "claim_id",<br/>  "verification_status": "status",<br/>  "message": "verification_message"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% Claim Settlement Flow
    Client->>Server: POST /api/v1/claim/:id/settle
    Note over Client,Server: Request Body:<br/>{<br/>  "settlement_amount": "amount",<br/>  "settlement_notes": "notes"<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>ClaimController: Settle Claim
    ClaimController->>PaymentService: Process Payment
    PaymentService-->>ClaimController: Payment Confirmation
    ClaimController->>Database: Update Settlement Status
    ClaimController->>NotificationService: Send Settlement Notification
    NotificationService-->>Client: Settlement Notification
    ClaimController-->>Client: Settlement Status
    Note over ClaimController,Client: Response Body:<br/>{<br/>  "claim_id": "claim_id",<br/>  "settlement_status": "status",<br/>  "settlement_amount": "amount",<br/>  "message": "settlement_message"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% Claim Status Flow
    Client->>Server: GET /api/v1/claim/:id/status
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token
    Server->>ClaimController: Get Claim Status
    ClaimController->>Database: Fetch Claim Data
    Database-->>ClaimController: Claim Data
    ClaimController-->>Client: Claim Status
    Note over ClaimController,Client: Response Body:<br/>{<br/>  "claim_id": "claim_id",<br/>  "status": "status",<br/>  "treatment_type": "type",<br/>  "amount": "amount",<br/>  "verification_status": "status",<br/>  "settlement_status": "status"<br/>}<br/>Headers:<br/>- Content-Type: application/json
```

## API Endpoints

### Claim Submission
```http
POST /api/v1/claim/submit
Content-Type: application/json
Authorization: Bearer <jwt_token>

{
    "policy_id": "string",
    "hospital_id": "string",
    "treatment_type": "string",
    "amount": "number",
    "documents": ["string"]
}
```

### Claim Verification
```http
POST /api/v1/claim/:id/verify
Content-Type: application/json
Authorization: Bearer <jwt_token>

{
    "verification_status": "string",
    "verification_notes": "string"
}
```

### Claim Settlement
```http
POST /api/v1/claim/:id/settle
Content-Type: application/json
Authorization: Bearer <jwt_token>

{
    "settlement_amount": "number",
    "settlement_notes": "string"
}
```

### Claim Status
```http
GET /api/v1/claim/:id/status
Authorization: Bearer <jwt_token>
```

## Data Models

### Claim Model
```javascript
{
    id: String,
    policy_id: String,
    hospital_id: String,
    treatment_type: String,
    amount: Number,
    status: String,
    verification_status: String,
    verification_notes: String,
    settlement_status: String,
    settlement_amount: Number,
    settlement_notes: String,
    documents: [String],
    created_at: Date,
    updated_at: Date
}
```

## Security Considerations

1. **Access Control**
   - Role-based access control for claim management
   - Document encryption
   - Secure payment processing
   - Audit logging

2. **Data Validation**
   - Claim data validation
   - Document validation
   - Payment validation

## Error Handling

### Common Error Codes
- 400: Bad Request - Invalid claim data
- 401: Unauthorized - Invalid token
- 403: Forbidden - Insufficient permissions
- 404: Not Found - Claim not found
- 409: Conflict - Claim already exists
- 422: Unprocessable Entity - Invalid payment data
- 500: Internal Server Error - Server-side issues

### Error Response Format
```javascript
{
    "status": "error",
    "code": "ERROR_CODE",
    "message": "Error description",
    "details": {
        "field": "error_details"
    }
}
```

## Integration Points

1. **Hospital Service**
   - Hospital validation
   - Treatment verification
   - Network status check

2. **Payment Service**
   - Payment processing
   - Transaction tracking

3. **Document Service**
   - Document storage
   - Document verification
   - Document retrieval

4. **Notification Service**
   - Claim submission notifications
   - Verification notifications
   - Settlement notifications
   - Status updates

## Best Practices

1. **Claim Management**
   - Regular status checks
   - Automated verification
   - Payment tracking
   - Document management

2. **Data Management**
   - Regular data backups
   - Audit logging
   - Data retention policies
   - Privacy compliance

3. **Performance**
   - Caching frequently accessed claims
   - Optimized database queries
   - Efficient document handling

4. **Monitoring**
   - Track claim submission rates
   - Monitor verification success rates
   - Track settlement processing
   - Alert on claim issues