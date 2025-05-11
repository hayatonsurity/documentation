# Hospital Integration Flow

## Overview
The hospital integration flow manages the integration with hospital systems for claim processing and verification.

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
        HospitalAPI[Hospital API]
        ClaimAPI[Claim API]
        AuthAPI[Auth API]
        ValidationAPI[Validation API]
        PolicyAPI[Policy API]
    end

    subgraph Service Layer
        HospitalService[Hospital Service]
        ClaimService[Claim Service]
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
    HospitalPortal --> AuthAPI
    Web --> HospitalAPI
    Mobile --> HospitalAPI
    AdminPortal --> HospitalAPI
    HospitalPortal --> HospitalAPI
    Web --> ClaimAPI
    Mobile --> ClaimAPI
    AdminPortal --> ClaimAPI
    HospitalPortal --> ClaimAPI

    %% API Layer Connections
    AuthAPI --> AuthService
    HospitalAPI --> HospitalService
    ClaimAPI --> ClaimService
    ValidationAPI --> ValidationService
    PolicyAPI --> PolicyService

    %% Service Layer Connections
    HospitalService --> ValidationService
    ClaimService --> ValidationService
    PolicyService --> ValidationService
    HospitalService --> NotificationService
    ClaimService --> NotificationService
    PolicyService --> NotificationService
    HospitalService --> CacheService
    ClaimService --> CacheService
    PolicyService --> CacheService

    %% Service to Data Connections
    HospitalService --> DB
    ClaimService --> DB
    PolicyService --> DB
    HospitalService --> Cache
    ClaimService --> Cache
    PolicyService --> Cache
    HospitalService --> Queue
    ClaimService --> Queue
    PolicyService --> Queue

    %% Data Layer Connections
    Queue --> NotificationService
    CacheService --> Cache

    classDef client fill:#f9f,stroke:#333,stroke-width:2px
    classDef api fill:#bbf,stroke:#333,stroke-width:2px
    classDef service fill:#bfb,stroke:#333,stroke-width:2px
    classDef data fill:#fbb,stroke:#333,stroke-width:2px

    class Web,Mobile,AdminPortal,HospitalPortal client
    class HospitalAPI,ClaimAPI,AuthAPI,ValidationAPI,PolicyAPI api
    class HospitalService,ClaimService,NotificationService,AuthService,ValidationService,CacheService,PolicyService service
    class DB,S3,Cache,Queue data
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant HospitalController
    participant CashlessController
    participant Database
    participant NotificationService
    participant ExternalAPI

    %% Hospital Listing Flow
    Client->>Server: GET /api/v1/hospitals
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token<br/>Query Parameters:<br/>- city: string<br/>- speciality: string<br/>- page: number<br/>- limit: number
    Server->>HospitalController: Get Hospitals
    HospitalController->>Database: Query Hospitals
    Database-->>HospitalController: Hospital Data
    HospitalController-->>Client: Hospital List
    Note over HospitalController,Client: Response Body:<br/>{<br/>  "hospitals": [{<br/>    "id": "hospital_id",<br/>    "name": "name",<br/>    "address": "address",<br/>    "specialities": ["speciality"]<br/>  }],<br/>  "total": "total_count",<br/>  "page": "current_page"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% Cashless Hospital Flow
    Client->>Server: POST /api/v1/cashless/hospital/verify
    Note over Client,Server: Request Body:<br/>{<br/>  "hospital_id": "hospital_id",<br/>  "beneficiary_id": "beneficiary_id",<br/>  "treatment_type": "treatment_type"<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>CashlessController: Process Request
    CashlessController->>Database: Check Eligibility
    Database-->>CashlessController: Eligibility Status
    CashlessController->>ExternalAPI: Verify Hospital
    ExternalAPI-->>CashlessController: Hospital Status
    CashlessController->>NotificationService: Send Status
    NotificationService-->>Client: Status Notification
    CashlessController-->>Client: Verification Result
    Note over CashlessController,Client: Response Body:<br/>{<br/>  "status": "status",<br/>  "message": "verification_message",<br/>  "details": {<br/>    "hospital": "hospital_details",<br/>    "eligibility": "eligibility_details"<br/>  }<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% Hospital Registration Flow
    Client->>Server: POST /api/v1/hospitals/register
    Note over Client,Server: Request Body:<br/>{<br/>  "name": "hospital_name",<br/>  "address": "address",<br/>  "city": "city",<br/>  "specialities": ["speciality"],<br/>  "contact": "contact",<br/>  "email": "email"<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>HospitalController: Register Hospital
    HospitalController->>Database: Store Hospital Data
    HospitalController->>ExternalAPI: Verify Hospital Details
    ExternalAPI-->>HospitalController: Verification Result
    HospitalController->>NotificationService: Send Registration Status
    NotificationService-->>Client: Registration Notification
    HospitalController-->>Client: Registration Result
    Note over HospitalController,Client: Response Body:<br/>{<br/>  "status": "success",<br/>  "hospital_id": "hospital_id",<br/>  "message": "registration_message"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% Hospital Update Flow
    Client->>Server: PUT /api/v1/hospitals/:id
    Note over Client,Server: Request Body:<br/>{<br/>  "name": "hospital_name",<br/>  "address": "address",<br/>  "specialities": ["speciality"],<br/>  "contact": "contact"<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: application/json
    Server->>HospitalController: Update Hospital
    HospitalController->>Database: Update Hospital Data
    Database-->>HospitalController: Update Confirmation
    HospitalController->>NotificationService: Send Update Notification
    NotificationService-->>Client: Update Notification
    HospitalController-->>Client: Update Result
    Note over HospitalController,Client: Response Body:<br/>{<br/>  "status": "success",<br/>  "message": "update_message",<br/>  "hospital": {<br/>    "id": "hospital_id",<br/>    "name": "updated_name",<br/>    "status": "updated_status"<br/>  }<br/>}<br/>Headers:<br/>- Content-Type: application/json
```

## API Endpoints

### Hospital Listing
```