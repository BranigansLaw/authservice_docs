**Software Design Specification**  
CP11 Auth Service

[[_TOC_]]

## Overview

* **Status:** In Progress
* **Stakeholders:**
    * **Responsible:**
        * @tyler.findlay
        * @luke.norman
    * **Accountable:**
        * @surya.manchikanti
    * **Consulted:**
        * @phillip_array
    * **Informed:**
        * 
* **Related Links:**
    * **Product Brief:** [DBP Auth Service/Supercomponent Integration](https://docs.google.com/document/d/1t0M2GRUd_5qkCdYkm2ZsaIxftC2i2TBFPNEjFJOAu6k)
    * **Shortcut Epic:** [CP11 integration](https://app.shortcut.com/arrayus/epic/52574)
    * **AuthService API Spec:** [AuthService API](https://gitlab.com/array.com-internal/engineering/specifications/api-specifications/-/blob/squad-4-deploy-fi/sc-52367/design-documents-for-cp11-authservice/_api-authservice.zed)
    * **User API Spec v4:** [User API v4 spec](https://gitlab.com/array.com-internal/engineering/specifications/api-specifications/-/blob/v4/api-user.zed)
    * **Super Component Documentation:** [Super Component Documentation](https://gitlab.com/array.com-internal/engineering/specifications/design-specifications/-/blob/squad-4-deploy-fi/sc-50177/create-fe-sds-template/js/website-webcom-svelte/001_SuperComponent.md)
* **Summary:**

This specification describes a new auth service which serves the purpose of allowing DBPs to authenticate with Array using the DBPs user ID. The AuthService also manages different enrollment types (auto enroll and KBA bypass) based on the appKey configured settings in the CaaS.

## Context

### Goal

The existing authentication API requires DBPs to store Array user IDs and manage the enrollment process within each platform. The goal of this new API is to provide an alternate API route that allows for authentication with Array without having to store Array user IDs, and specify the enrollment process based on DBP needs.

### Problem

1. Currently, authentication must happen using Array enrollment components. This API will offer a route to pass an authentication token to a super component (still in development) that would automatically render an enrollment flow for unenrolled users, or a credit report to enrolled users without the DBP needing to know if a given user is enrolled or not.
2. DBPs are required to handle the mapping of one of their user ID's to the respective Array user ID. This service will use the DBP user ID to identify users.

### Limitations

None identified.

### Success Criteria

1. An API endpoint that
    - returns an auth token based on the appKey and DBP user ID and PII information provided
    - if the appId has `autoEnroll` enabled, enrolls that user immediately and returns an auth token
2. An API endpoint that can use an auth token to retrieve
   - A 200 reponse if the user is already registered including a user token
   - A 202 response if the user is not registered and auth enrolled is not enabled including any Customer PII information stored
   - A 203 if the user is not registered, auto enrolled is enabled, and consent is enabled
3. An API endpoint that exchanges an finalizes an enrollment with KBA
4. A data store that:
    - maps appKey/dbpUserId pairs to Array user IDs
    - stores authTokens

## Architecture

### Diagrams

#### API Flow Chart

##### `POST /api/auth/v1/`
```mermaid
graph TD
    A(POST /api/auth/v1/) --> B{Request OK?}
    B -->|YES| C{Is KBA Bypass Enabled?}
    C -->|YES| C1[Create User]
    C1 --> C11[Generate AuthToken]
    C11 --> C111(201 with Token)
    C -->|NO| C2[Generate AuthToken]
    C2 --> C21(200 OK with Token)
    B -->|NO| D[Missing or Invalid Fields]
    D --> E(400 Bad Request)
    B -->|NO| F[Missing Server Token]
    F --> G(401 Unauthorized)
    B -->|NO| H[Invalid Server Token]
    H --> I(403 Forbidden)
    B -->|NO| J[The CaaS configuration and b2b_clients<br>configuration for KBA bypass are conflicting]
    J --> K(409 Conflict)
    B -->|NO| L[Server Error]
    L --> M(500 External Error)
```

##### `GET /api/auth/v1/{authToken}`
```mermaid
graph TD
    A("GET /api/auth/v1/{authToken}") --> B{Request OK?}
    B -->|YES| C{Is the user already enrolled?}
    C --YES--> C1[User is already enrolled. Return a userToken]
    C1 ---> C11(200 OK with userToken)
    C --NO--> C2{Is auto enroll enabled?}
    C2 --YES--> C21{Is KBA Bypass Enabled?}
    C21 --NO--> C1
    C21 --YES--> C211(203 Promp user to verify enrollment)
    C2 --NO--> C22[Normal Enrollment Flow with KBA questions]
    C22 --> C221(202 with PII info)
    B -->|NO| H[authToken was not found or is expired]
    H --> I(404 Not Found)
    B -->|NO| J[Server Error]
    J --> K(500 External Error)
```

##### `PATCH /api/auth/v1/{authToken}`
```mermaid
graph TD
    A("PATCH /api/auth/v1/{authToken}") --> B{Request OK?}
    B -->|YES| B1[authToken is valid, finalize user creation]
    B1 --YES--> B11(203 with userToken)
    B --NO--> D[authToken is valid, but is for an<br>already registered user, or an appId<br>that does not have KBA consent enabled]
    D --> I(404 Not Found)
    B -->|NO| H[authToken was not found, expired, or<br>not valid for consent based enrollment]
    H --> I
    B -->|NO| J[Server Error]
    J --> K(500 External Error)
```

#### Sequence Diagram

##### Enrolled user

```mermaid
sequenceDiagram
    participant DBP
    participant Super Component
    participant POST /api/auth/v1/
    participant GET /api/auth/v1/{authToken}
    participant AuthService DB
    participant POST /api/user/v4/token
    DBP->>+POST /api/auth/v1/: dbpUserId, PII ...
    POST /api/auth/v1/->>+AuthService DB: lookup dbpUser's Array UserId
    AuthService DB->>-POST /api/auth/v1/: dbp's Array user ID
    POST /api/auth/v1/->>+AuthService DB: generate authToken linked to Array userID and PII
    AuthService DB->>-POST /api/auth/v1/: newly generated authToken
    POST /api/auth/v1/->>-DBP: 200 response with authToken
    DBP->>+Super Component: authToken
    Super Component->>+GET /api/auth/v1/{authToken}: authToken
    GET /api/auth/v1/{authToken}->>+AuthService DB: check token is valid
    AuthService DB->>-GET /api/auth/v1/{authToken}: valid and return Array user ID
    GET /api/auth/v1/{authToken}->>+POST /api/user/v4/token: get user token
    POST /api/user/v4/token->>-GET /api/auth/v1/{authToken}: userToken
    GET /api/auth/v1/{authToken}->>-Super Component: 200 OK - userToken
    Super Component->>-DBP: Render User Credit Overview page
```

##### Unenrolled user with Auto Enroll disabled

```mermaid
sequenceDiagram
    participant DBP
    participant Super Component
    participant POST /api/auth/v1/
    participant GET /api/auth/v1/{authToken}
    participant AuthService DB
    participant CaaS
    participant b2b_clients Table
    DBP->>+POST /api/auth/v1/: dbpUserId, PII ...
    POST /api/auth/v1/->>+AuthService DB: lookup dbpUserId's Array UserId
    AuthService DB->>-POST /api/auth/v1/: no association found
    POST /api/auth/v1/->>+CaaS: check if appKey has auto enroll or KBA Bypass Enabled
    CaaS->>-POST /api/auth/v1/: appKey has auto enroll disabled
    POST /api/auth/v1/->>+b2b_clients Table: check if trust_auth is true
    b2b_clients Table->>-POST /api/auth/v1/: trust_auth is false
    POST /api/auth/v1/->>+AuthService DB: generate authToken linked to PII and dbpUserId
    AuthService DB->>-POST /api/auth/v1/: newly generated authToken
    POST /api/auth/v1/->>-DBP: 201 response with authToken
    DBP->>+Super Component: authToken
    Super Component->>+GET /api/auth/v1/{authToken}: authToken
    GET /api/auth/v1/{authToken}->>+AuthService DB: check token is valid
    AuthService DB->>-GET /api/auth/v1/{authToken}: valid and is already registered
    GET /api/auth/v1/{authToken}->>-Super Component: 202 response with PII
    Super Component->>-DBP: Render Enrollment form (prefilled)
```

##### Unenrolled user with Auto Enroll enabled and KBA Bypass Disabled

```mermaid
sequenceDiagram
    participant DBP
    participant Super Component
    participant POST /api/auth/v1/
    participant GET /api/auth/v1/{authToken}
    participant AuthService DB
    participant CaaS
    participant b2b_clients Table
    participant POST /api/user/v4
    DBP->>+POST /api/auth/v1/: dbpUserId, PII ...
    POST /api/auth/v1/->>+AuthService DB: lookup dbpUserId's Array UserId
    AuthService DB->>-POST /api/auth/v1/: no association found
    POST /api/auth/v1/->>+CaaS: check if appKey has auto enroll or KBA Bypass Enabled
    CaaS->>-POST /api/auth/v1/: appKey has auto enroll enabled and KBA bypass is disabled
    POST /api/auth/v1/->>+b2b_clients Table: check if trust_auth is true
    b2b_clients Table->>-POST /api/auth/v1/: trust_auth is true
    POST /api/auth/v1/->>+POST /api/user/v4: create user
    POST /api/user/v4->>-POST /api/auth/v1/: new Array userId and userToken
    POST /api/auth/v1/->>+AuthService DB: generate authToken linked to PII and dbpUserId
    AuthService DB->>-POST /api/auth/v1/: return authToken
    POST /api/auth/v1/->>-DBP: 200 response with authToken
    DBP->>+Super Component: authToken
    Super Component->>+GET /api/auth/v1/{authToken}: authToken
    GET /api/auth/v1/{authToken}->>+AuthService DB: check token is valid
    AuthService DB->>-GET /api/auth/v1/{authToken}: valid and return Array user ID
    GET /api/auth/v1/{authToken}->>+POST /api/user/v4/token: get user token
    POST /api/user/v4/token->>-GET /api/auth/v1/{authToken}: userToken
    GET /api/auth/v1/{authToken}->>-Super Component: 200 response with userToken
    Super Component->>-DBP: Render Credit Overview page
```

##### Unenrolled user with Auto Enroll enabled and KBA Bypass Enabled

```mermaid
sequenceDiagram
    participant DBP
    participant Super Component
    participant POST /api/auth/v1/
    participant GET /api/auth/v1/{authToken}
    participant PATCH /api/auth/v1/{authToken}
    participant AuthService DB
    participant CaaS
    participant b2b_clients Table
    participant POST /api/user/v4
    DBP->>+POST /api/auth/v1/: dbpUserId, PII ...
    POST /api/auth/v1/->>+AuthService DB: lookup dbpUserId's Array UserId
    AuthService DB->>-POST /api/auth/v1/: no association found
    POST /api/auth/v1/->>+CaaS: check if appKey has auto enroll or KBA Bypass Enabled
    CaaS->>-POST /api/auth/v1/: appKey has auto enroll enabled and KBA bypass enabled
    POST /api/auth/v1/->>+b2b_clients Table: check if trust_auth is true
    b2b_clients Table->>-POST /api/auth/v1/: trust_auth is false
    POST /api/auth/v1/->>+AuthService DB: generate authToken linked to PII and dbpUserId
    AuthService DB->>-POST /api/auth/v1/: return authToken
    POST /api/auth/v1/->>-DBP: authToken
    DBP->>+Super Component: 200 response with authToken
    Super Component->>+GET /api/auth/v1/{authToken}: authToken
    GET /api/auth/v1/{authToken}->>+AuthService DB: validate authToken
    AuthService DB->>-GET /api/auth/v1/{authToken}: token is valid, user consent is required for registration
    GET /api/auth/v1/{authToken}->>-Super Component: 203 response with no body
    Super Component->>-DBP: Render Consent page
    DBP ->>+ Super Component: User gives consent
    Super Component ->>+PATCH /api/auth/v1/{authToken}: authToken
    PATCH /api/auth/v1/{authToken}->>+AuthService DB: verify token
    AuthService DB->>-PATCH /api/auth/v1/{authToken}: validate token was waiting for consent
    PATCH /api/auth/v1/{authToken}->>+POST /api/user/v4: create user
    POST /api/user/v4->>-PATCH /api/auth/v1/{authToken}: new Array userId, userToken
    PATCH /api/auth/v1/{authToken}->>+AuthService DB: associate dbpUserId with new Array userId
    AuthService DB->>-PATCH /api/auth/v1/{authToken}: association created
    PATCH /api/auth/v1/{authToken}->>-Super Component: 200 response with userToken
    Super Component->>-DBP: Render Credit Overview page
```

##### Conflicting CaaS and `b2b_clients` settings

```mermaid
sequenceDiagram
    participant DBP
    participant POST /api/auth/v1/
    participant AuthService DB
    participant CaaS
    participant b2b_clients Table
    DBP->>+POST /api/auth/v1/: dbpUserId, PII ...
    POST /api/auth/v1/->>+AuthService DB: lookup dbpUserId's Array UserId
    AuthService DB->>-POST /api/auth/v1/: no association found
    POST /api/auth/v1/->>+CaaS: check if appKey has auto enroll or KBA Bypass Enabled
    CaaS->>-POST /api/auth/v1/: appKey has auto enroll enabled and KBA bypass enabled
    POST /api/auth/v1/->>+b2b_clients Table: check if trust_auth is true
    b2b_clients Table->>-POST /api/auth/v1/: trust_auth is false
    POST /api/auth/v1/->>-DBP: 409 Conflict
```

### Connected Systems

```mermaid
graph LR
    A[DBP] --> B[Super Component]
    B --> C[AuthService]
    A --> C
    C --> D["/api/user/v4"]
    C --> E[AuthService DB]
    C --> F[b2b_clients Table]
    C --> G[CaaS]
```

## Timeline

* One engineer is required for completion of this feature, but two working together might be beneficial due to complexity.
* Feature is critical to the Supercomponent, as well as CP11 integration with DBPs on the Deploy FI team (Alkami, Banno, Q2).

#### How would this design change if we had infinite resources?

* Fast response time from the auth generation endpoint so there would be no requirement for cacheing tokens on the DBP end
