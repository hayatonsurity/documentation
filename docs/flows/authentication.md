# Authentication Flow

## Overview
The authentication flow in EmployeeSure handles user authentication, authorization, and session management.

## High-Level Design

```mermaid
graph TB
    subgraph Client Layer
        Web[Web Client]
        Mobile[Mobile Client]
        AdminPortal[Admin Portal]
        EmployerPortal[Employer Portal]
        HospitalPortal[Hospital Portal]
    end

    subgraph API Layer
        AuthAPI[Auth API]
        ValidationAPI[Validation API]
        PolicyAPI[Policy API]
        SubscriptionAPI[Subscription API]
    end

    subgraph Service Layer
        AuthService[Auth Service]
        ValidationService[Validation Service]
        NotificationService[Notification Service]
        CacheService[Cache Service]
        PolicyService[Policy Service]
        SubscriptionService[Subscription Service]
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
    Web --> ValidationAPI
    Mobile --> ValidationAPI
    AdminPortal --> ValidationAPI
    EmployerPortal --> ValidationAPI
    HospitalPortal --> AuthAPI

    %% API Layer Connections
    AuthAPI --> AuthService
    ValidationAPI --> ValidationService
    PolicyAPI --> PolicyService
    SubscriptionAPI --> SubscriptionService

    %% Service Layer Connections
    AuthService --> ValidationService
    ValidationService --> NotificationService
    PolicyService --> ValidationService
    AuthService --> CacheService
    ValidationService --> CacheService
    PolicyService --> CacheService
    SubscriptionService --> ValidationService
    AuthService --> NotificationService
    PolicyService --> NotificationService
    SubscriptionService --> NotificationService

    %% Service to Data Connections
    AuthService --> DB
    ValidationService --> DB
    PolicyService --> DB
    SubscriptionService --> DB
    AuthService --> Cache
    ValidationService --> Cache
    PolicyService --> Cache
    SubscriptionService --> Cache
    AuthService --> Queue
    ValidationService --> Queue
    PolicyService --> Queue
    SubscriptionService --> Queue

    %% Data Layer Connections
    Queue --> NotificationService
    CacheService --> Cache

    classDef client fill:#f9f,stroke:#333,stroke-width:2px
    classDef api fill:#bbf,stroke:#333,stroke-width:2px
    classDef service fill:#bfb,stroke:#333,stroke-width:2px
    classDef data fill:#fbb,stroke:#333,stroke-width:2px

    class Web,Mobile,AdminPortal,EmployerPortal,HospitalPortal client
    class AuthAPI,ValidationAPI,PolicyAPI,SubscriptionAPI api
    class AuthService,ValidationService,NotificationService,CacheService,PolicyService,SubscriptionService service
    class DB,S3,Cache,Queue data
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant Redis
    participant JWT
    participant Database

    %% Initial Login
    Client->>Server: POST /api/v1/auth/login
    Note over Client,Server: Request Body:<br/>{<br/>  "email": "user@example.com",<br/>  "password": "password"<br/>}<br/>Headers:<br/>- Content-Type: application/json
    Server->>Database: Verify Credentials
    Database-->>Server: User Data
    
    alt Valid Credentials
        Server->>JWT: Generate Token
        JWT-->>Server: JWT Token
        Server->>Redis: Store Session
        Redis-->>Server: Session Stored
        Server-->>Client: Return Token & User Data
        Note over Server,Client: Response Body:<br/>{<br/>  "token": "jwt_token",<br/>  "user": {<br/>    "id": "user_id",<br/>    "email": "user@example.com",<br/>    "role": "user_role"<br/>  }<br/>}<br/>Headers:<br/>- Content-Type: application/json
    else Invalid Credentials
        Server-->>Client: 401 Unauthorized
        Note over Server,Client: Response Body:<br/>{<br/>  "error": "Invalid credentials"<br/>}<br/>Headers:<br/>- Content-Type: application/json
    end

    %% Token Validation
    Client->>Server: GET /api/v1/protected-route
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token
    Server->>JWT: Verify Token
    alt Valid Token
        JWT-->>Server: Decoded User Info
        Server->>Redis: Check Session
        Redis-->>Server: Session Status
        Server-->>Client: Authenticated Response
        Note over Server,Client: Response Body:<br/>{<br/>  "data": "protected_data"<br/>}<br/>Headers:<br/>- Content-Type: application/json
    else Invalid Token
        JWT-->>Server: Invalid Token
        Server-->>Client: 401 Unauthorized
        Note over Server,Client: Response Body:<br/>{<br/>  "error": "Invalid token"<br/>}<br/>Headers:<br/>- Content-Type: application/json
    end

    %% Session Management
    Client->>Server: POST /api/v1/auth/logout
    Note over Client,Server: Headers:<br/>- Authorization: Bearer jwt_token
    Server->>Redis: Delete Session
    Redis-->>Server: Session Deleted
    Server-->>Client: Logout Confirmation
    Note over Server,Client: Response Body:<br/>{<br/>  "message": "Successfully logged out"<br/>}<br/>Headers:<br/>- Content-Type: application/json
```

## API Endpoints

### Login
```http
POST /api/v1/auth/login
Content-Type: application/json

{
    "email": "user@example.com",
    "password": "password"
}
```

### Token Validation
```http
GET /api/v1/protected-route
Authorization: Bearer <jwt_token>
```

### Logout
```http
POST /api/v1/auth/logout
Authorization: Bearer <jwt_token>
```

## Data Models

### User Model
```javascript
{
    id: String,
    email: String,
    password: String (hashed),
    role: String,
    status: String,
    created_at: Date,
    updated_at: Date
}
```

### Session Model
```javascript
{
    session_id: String,
    user_id: String,
    token: String,
    expires_at: Date,
    created_at: Date
}
```

## Security Considerations

1. **Token Security**
   - JWT tokens are signed with a secure secret key
   - Tokens have a limited expiration time
   - Token payload is encrypted

2. **Password Security**
   - Passwords are hashed using bcrypt
   - Password complexity requirements enforced
   - Rate limiting on login attempts

3. **Session Security**
   - Sessions are stored in Redis with expiration
   - Session invalidation on logout
   - Multiple device session management

4. **API Security**
   - Rate limiting on authentication endpoints
   - CORS configuration
   - Helmet security headers

## Error Handling

### Common Error Codes
- 401: Unauthorized - Invalid or expired token
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

1. **Redis Integration**
   - Session storage
   - Token blacklisting
   - Rate limiting

2. **Database Integration**
   - User authentication
   - Role management
   - Session tracking

3. **External Services**
   - OAuth providers (if implemented)
   - SSO integration
   - MFA services

## Best Practices

1. **Token Management**
   - Use short-lived access tokens
   - Implement refresh token rotation
   - Secure token storage on client

2. **Session Management**
   - Regular session cleanup
   - Session activity tracking
   - Concurrent session handling

3. **Security Headers**
   - Implement security headers
   - Use secure cookies
   - Enable HTTPS only

4. **Monitoring**
   - Track failed login attempts
   - Monitor session usage
   - Log security events 