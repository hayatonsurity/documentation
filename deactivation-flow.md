# Deactivation Flow Documentation

## Overview
The deactivation system handles various types of deactivations in the EmployeeSure platform, including employee deactivation, family member deactivation, employer deactivation, and partial deactivations. This document outlines the different deactivation flows and their implementation details.

## Types of Deactivation

### 1. Employee Deactivation
- **Trigger**: When an employee leaves the company or needs to be deactivated
- **Flow**:
  1. Validates leaving date and reason
  2. Deactivates primary subscription
  3. Handles associated family member subscriptions
  4. Updates beneficiary status
  5. Revokes admin access if applicable

### 2. Family Member Deactivation
- **Trigger**: When a family member needs to be removed from coverage
- **Flow**:
  1. Validates the member is not a primary member
  2. Updates subscription status
  3. Updates parent subscription records
  4. Updates invited members status

### 3. Employer Deactivation
- **Trigger**: When an entire employer account needs to be deactivated
- **Flow**:
  1. Deactivates employer account
  2. Deactivates all employee subscriptions
  3. Deactivates employer cpanel users
  4. Handles partial deactivation if specified
  5. Sends deactivation notification emails

### 4. Partner Integration Deactivation
- **Trigger**: When deactivation is initiated through partner integration
- **Flow**:
  1. Validates invite ID and reason
  2. Implements locking mechanism to prevent concurrent deactivations
  3. Processes deactivation through subscription helper
  4. Handles response formatting for partner API

## API Endpoints

### Internal APIs
1. `/api/v1/subscription/deactivate` - Main deactivation endpoint
2. `/api/v1/subscription/deactivate-employer` - Employer deactivation endpoint
3. `/api/v1/employer/beneficiary/deactivate` - Beneficiary deactivation endpoint

### Partner APIs
1. Partner deactivation endpoint with invite ID and reason

## Deactivation Process Flow

```mermaid
graph TD
    A[Deactivation Request] --> B{Type Check}
    B -->|Employee| C[Employee Deactivation]
    B -->|Family Member| D[Family Member Deactivation]
    B -->|Employer| E[Employer Deactivation]
    B -->|Partner| F[Partner Deactivation]
    
    C --> G[Validate Dates]
    C --> H[Update Subscription]
    C --> I[Handle Family Members]
    C --> J[Revoke Access]
    
    D --> K[Validate Member]
    D --> L[Update Subscription]
    D --> M[Update Parent]
    
    E --> N[Deactivate Employer]
    E --> O[Deactivate Employees]
    E --> P[Send Notifications]
    
    F --> Q[Validate Request]
    F --> R[Acquire Lock]
    F --> S[Process Deactivation]
    F --> T[Release Lock]
```

## Detailed Control Flows

### 1. Employee Deactivation Flow
```mermaid
sequenceDiagram
    participant API as /api/v1/subscription/deactivate
    participant SH as SubscriptionHelper
    participant DB as Database
    participant Redis as Redis Lock
    participant Notify as Notification

    API->>SH: deactivateSubscriptionV2()
    SH->>DB: Validate joining/leaving dates
    SH->>DB: Find primary subscription
    SH->>DB: Find family member subscriptions
    SH->>DB: Update primary subscription status
    SH->>DB: Update family member subscriptions
    SH->>DB: Update beneficiary status
    SH->>DB: Revoke admin access
    SH->>Notify: Send deactivation email
    SH->>API: Return success/failure
```

### 2. Family Member Deactivation Flow
```mermaid
sequenceDiagram
    participant API as /api/v1/subscription/deactivate
    participant SH as SubscriptionHelper
    participant DB as Database
    participant Redis as Redis Lock
    participant Notify as Notification

    API->>SH: deactivateFamilyMember()
    SH->>DB: Validate member type
    SH->>DB: Find parent subscription
    SH->>DB: Update member subscription
    SH->>DB: Update parent subscription
    SH->>DB: Update invited members
    SH->>Notify: Send deactivation email
    SH->>API: Return success/failure
```

### 3. Employer Deactivation Flow
```mermaid
sequenceDiagram
    participant API as /api/v1/subscription/deactivate-employer
    participant SH as SubscriptionHelper
    participant DB as Database
    participant Redis as Redis Lock
    participant Notify as Notification

    API->>SH: deactivateEmployer()
    SH->>DB: Find employer account
    SH->>DB: Find all employee subscriptions
    SH->>DB: Deactivate employer account
    SH->>DB: Deactivate employee subscriptions
    SH->>DB: Deactivate cpanel users
    SH->>Notify: Send deactivation email
    SH->>API: Return success/failure
```

### 4. Partner Integration Deactivation Flow
```mermaid
sequenceDiagram
    participant API as Partner API
    participant SH as SubscriptionHelper
    participant DB as Database
    participant Redis as Redis Lock
    participant Notify as Notification

    API->>Redis: Acquire deactivation lock
    Redis-->>API: Lock acquired
    API->>SH: deactivateMembershipSubscription()
    SH->>DB: Validate invite ID
    SH->>DB: Find subscription
    SH->>DB: Update subscription status
    SH->>DB: Update member status
    SH->>Notify: Send deactivation email
    SH->>API: Return success/failure
    API->>Redis: Release deactivation lock
```

## Key Components

### 1. Subscription Helper
- Handles core deactivation logic
- Manages subscription status updates
- Processes different types of deactivations

### 2. Deactivation Locking
- Implements Redis-based locking
- Prevents concurrent deactivations
- Ensures data consistency

### 3. Notification System
- Sends deactivation emails
- Updates activity logs
- Handles integration notifications

## Error Handling

1. **Validation Errors**
   - Invalid dates
   - Missing required fields
   - Invalid member types

2. **Process Errors**
   - Concurrent deactivation attempts
   - Database operation failures
   - Integration failures

3. **Recovery Mechanisms**
   - Automatic lock release
   - Error logging
   - Status tracking

## Best Practices

1. **Before Deactivation**
   - Validate all input parameters
   - Check for active processes
   - Verify member status

2. **During Deactivation**
   - Use transaction where applicable
   - Implement proper locking
   - Log all actions

3. **After Deactivation**
   - Send notifications
   - Update audit logs
   - Clean up temporary data

## Monitoring and Logging

1. **Activity Tracking**
   - Deactivation requests
   - Success/failure status
   - Processing time

2. **Error Monitoring**
   - Failed deactivations
   - System errors
   - Integration issues

3. **Audit Logs**
   - User actions
   - System changes
   - Integration events 