# File Upload Flow

## Overview
The file upload flow in EmployeeSure handles the secure upload, storage, and management of various document types.

## High-Level Design

```mermaid
graph TB
    %% Styling
    classDef client fill:#f9f,stroke:#333,stroke-width:2px,color:#333,font-weight:bold
    classDef api fill:#bbf,stroke:#333,stroke-width:2px,color:#333,font-weight:bold
    classDef service fill:#bfb,stroke:#333,stroke-width:2px,color:#333,font-weight:bold
    classDef data fill:#fbb,stroke:#333,stroke-width:2px,color:#333,font-weight:bold
    classDef layer fill:#fff,stroke:#333,stroke-width:2px,stroke-dasharray: 5 5

    %% Layers
    subgraph ClientLayer["📱 Client Layer"]
        direction TB
        Web["🌐 Web Client"]
        Mobile["📱 Mobile Client"]
        AdminPortal["👨‍💼 Admin Portal"]
        EmployerPortal["🏢 Employer Portal"]
    end

    subgraph APILayer["🔌 API Layer"]
        direction TB
        UploadAPI["📤 Upload API"]
        DocumentAPI["📄 Document API"]
        AuthAPI["🔐 Auth API"]
        ValidationAPI["✓ Validation API"]
    end

    subgraph ServiceLayer["⚙️ Service Layer"]
        direction TB
        UploadService["📤 Upload Service"]
        DocumentService["📄 Document Service"]
        NotificationService["📢 Notification Service"]
        AuthService["🔐 Auth Service"]
        ValidationService["✓ Validation Service"]
        CacheService["💾 Cache Service"]
    end

    subgraph DataLayer["💾 Data Layer"]
        direction TB
        DB["🗄️ MongoDB"]
        S3["📦 S3 Storage"]
        Cache["⚡ Redis Cache"]
        Queue["📨 AWS SQS"]
    end

    %% Client to API Connections
    ClientLayer --> APILayer
    Web --> AuthAPI
    Mobile --> AuthAPI
    AdminPortal --> AuthAPI
    EmployerPortal --> AuthAPI
    Web --> UploadAPI
    Mobile --> UploadAPI
    AdminPortal --> UploadAPI
    EmployerPortal --> UploadAPI

    %% API to Service Connections
    APILayer --> ServiceLayer
    AuthAPI --> AuthService
    UploadAPI --> UploadService
    DocumentAPI --> DocumentService
    ValidationAPI --> ValidationService

    %% Service to Data Connections
    ServiceLayer --> DataLayer
    UploadService --> DB
    DocumentService --> DB
    UploadService --> S3
    DocumentService --> S3
    UploadService --> Cache
    DocumentService --> Cache
    UploadService --> Queue
    DocumentService --> Queue

    %% Service to Service Connections
    UploadService --> ValidationService
    DocumentService --> ValidationService
    UploadService --> NotificationService
    DocumentService --> NotificationService
    UploadService --> CacheService
    DocumentService --> CacheService

    %% Data Layer Connections
    Queue --> NotificationService
    CacheService --> Cache

    %% Apply styles
    class Web,Mobile,AdminPortal,EmployerPortal client
    class UploadAPI,DocumentAPI,AuthAPI,ValidationAPI api
    class UploadService,DocumentService,NotificationService,AuthService,ValidationService,CacheService service
    class DB,S3,Cache,Queue data
    class ClientLayer,APILayer,ServiceLayer,DataLayer layer
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant UploadController
    participant DocumentService
    participant Database
    participant S3
    participant NotificationService
    participant SecurityService

    %% File Upload Flow
    Client->>Server: POST /api/v1/upload
    Note over Client,Server: Request Body:<br/>{<br/>  "file": "file_data",<br/>  "type": "document_type",<br/>  "metadata": {<br/>    "name": "file_name",<br/>    "size": "file_size",<br/>    "content_type": "mime_type"<br/>  }<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: multipart/form-data
    Server->>UploadController: Process Upload
    UploadController->>SecurityService: Validate File
    SecurityService-->>UploadController: Validation Result
    UploadController->>S3: Store File
    S3-->>UploadController: Storage Confirmation
    UploadController->>Database: Store File Metadata
    Database-->>UploadController: Metadata Confirmation
    UploadController->>NotificationService: Send Upload Notification
    NotificationService-->>Client: Upload Notification
    UploadController-->>Client: Upload Status
    Note over UploadController,Client: Response Body:<br/>{<br/>  "file_id": "file_id",<br/>  "status": "status",<br/>  "url": "file_url",<br/>  "message": "upload_message"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% File Retrieval Flow
    Client->>Server: GET /api/v1/files/:id
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token
    Server->>UploadController: Get File
    UploadController->>Database: Fetch File Metadata
    Database-->>UploadController: File Metadata
    UploadController->>S3: Retrieve File
    S3-->>UploadController: File Data
    UploadController-->>Client: File Data
    Note over UploadController,Client: Response Body:<br/>{<br/>  "file_id": "file_id",<br/>  "url": "file_url",<br/>  "metadata": {<br/>    "name": "file_name",<br/>    "type": "file_type",<br/>    "size": "file_size"<br/>  }<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% File Update Flow
    Client->>Server: PUT /api/v1/files/:id
    Note over Client,Server: Request Body:<br/>{<br/>  "file": "file_data",<br/>  "metadata": {<br/>    "name": "file_name",<br/>    "type": "file_type"<br/>  }<br/>}<br/>Headers:<br/>- Authorization: Bearer jwt_token<br/>- Content-Type: multipart/form-data
    Server->>UploadController: Update File
    UploadController->>SecurityService: Validate File
    SecurityService-->>UploadController: Validation Result
    UploadController->>S3: Update File
    S3-->>UploadController: Update Confirmation
    UploadController->>Database: Update File Metadata
    Database-->>UploadController: Metadata Update
    UploadController->>NotificationService: Send Update Notification
    NotificationService-->>Client: Update Notification
    UploadController-->>Client: Update Status
    Note over UploadController,Client: Response Body:<br/>{<br/>  "file_id": "file_id",<br/>  "status": "updated_status",<br/>  "url": "updated_url",<br/>  "message": "update_message"<br/>}<br/>Headers:<br/>- Content-Type: application/json

    %% File Delete Flow
    Client->>Server: DELETE /api/v1/files/:id
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token
    Server->>UploadController: Delete File
    UploadController->>S3: Remove File
    S3-->>UploadController: Deletion Confirmation
    UploadController->>Database: Remove File Metadata
    Database-->>UploadController: Metadata Deletion
    UploadController->>NotificationService: Send Deletion Notification
    NotificationService-->>Client: Deletion Notification
    UploadController-->>Client: Deletion Status
    Note over UploadController,Client: Response Body:<br/>{<br/>  "file_id": "file_id",<br/>  "status": "deleted",<br/>  "message": "deletion_message"<br/>}<br/>Headers:<br/>- Content-Type: application/json
```

## API Endpoints

### File Upload
```