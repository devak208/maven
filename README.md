# Authentication System Documentation

## Table of Contents
1. [Database Schema Overview](#database-schema-overview)
2. [Authentication Flow Diagrams](#authentication-flow-diagrams)
3. [API Endpoints Flow](#api-endpoints-flow)
4. [Role-Based Access Control](#role-based-access-control)
5. [Session Management](#session-management)
6. [Security Features](#security-features)

---

## Database Schema Overview

### 1. **Users Table** (`users`)
**Purpose**: Stores core user information and roles for authentication

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `id` | text (PK) | Unique user identifier | `user_abc123def456` |
| `name` | text | User's display name | `John Doe` |
| `email` | text (UNIQUE) | User's email address | `john@example.com` |
| `emailVerified` | boolean | Email verification status | `true/false` |
| `image` | text | Profile image URL | `https://example.com/avatar.jpg` |
| `role` | text | User role for access control | `admin`, `user`, `moderator` |
| `createdAt` | timestamp | Account creation time | `2025-06-30T10:00:00Z` |
| `updatedAt` | timestamp | Last update time | `2025-06-30T10:00:00Z` |

### 2. **Accounts Table** (`accounts`)
**Purpose**: Stores authentication provider information and credentials

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `id` | text (PK) | Unique account identifier | `acc_xyz789abc123` |
| `accountId` | text | Provider-specific account ID | `email_provider_id` |
| `providerId` | text | Authentication provider | `email`, `google`, `github` |
| `userId` | text (FK) | Reference to users table | `user_abc123def456` |
| `accessToken` | text | OAuth access token | `at_1234567890abcdef` |
| `refreshToken` | text | OAuth refresh token | `rt_abcdef1234567890` |
| `idToken` | text | OAuth ID token | `id_fedcba0987654321` |
| `accessTokenExpiresAt` | timestamp | Token expiration time | `2025-07-01T10:00:00Z` |
| `refreshTokenExpiresAt` | timestamp | Refresh token expiration | `2025-07-30T10:00:00Z` |
| `scope` | text | OAuth scope permissions | `read,write,email` |
| `password` | text | Hashed password (email provider) | `$2b$10$...` |
| `createdAt` | timestamp | Account creation time | `2025-06-30T10:00:00Z` |
| `updatedAt` | timestamp | Last update time | `2025-06-30T10:00:00Z` |

### 3. **Sessions Table** (`sessions`)
**Purpose**: Manages active user sessions for authentication state

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `id` | text (PK) | Unique session identifier | `sess_qwe123rty456` |
| `expiresAt` | timestamp | Session expiration time | `2025-07-07T10:00:00Z` |
| `token` | text (UNIQUE) | Session token | `better-auth.session_token_value` |
| `createdAt` | timestamp | Session creation time | `2025-06-30T10:00:00Z` |
| `updatedAt` | timestamp | Last session update | `2025-06-30T10:00:00Z` |
| `ipAddress` | text | Client IP address | `192.168.1.100` |
| `userAgent` | text | Client user agent | `Mozilla/5.0...` |
| `userId` | text (FK) | Reference to users table | `user_abc123def456` |

### 4. **Verifications Table** (`verifications`)
**Purpose**: Handles email verification, password reset tokens, etc.

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `id` | text (PK) | Unique verification identifier | `ver_asd123fgh456` |
| `identifier` | text | Email or user identifier | `john@example.com` |
| `value` | text | Verification token/code | `verify_token_12345` |
| `expiresAt` | timestamp | Token expiration time | `2025-06-30T11:00:00Z` |
| `createdAt` | timestamp | Token creation time | `2025-06-30T10:00:00Z` |
| `updatedAt` | timestamp | Last update time | `2025-06-30T10:00:00Z` |

---

## Authentication Flow Diagrams

### 1. **User Registration Flow**

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │    │  Auth API   │    │ Better Auth │    │  Database   │
│ (Frontend)  │    │ (Express)   │    │   Library   │    │(PostgreSQL) │
└─────┬───────┘    └─────┬───────┘    └─────┬───────┘    └─────┬───────┘
      │                  │                  │                  │
      │ POST /api/auth/  │                  │                  │
      │ sign-up/email    │                  │                  │
      ├─────────────────►│                  │                  │
      │ {email, password,│                  │                  │
      │  name}           │                  │                  │
      │                  │                  │                  │
      │                  │ Create Request   │                  │
      │                  │ Object with URL  │                  │
      │                  ├─────────────────►│                  │
      │                  │                  │                  │
      │                  │                  │ 1. Hash Password │
      │                  │                  │ 2. Generate IDs  │
      │                  │                  │ 3. Validate Data │
      │                  │                  │                  │
      │                  │                  │ INSERT INTO      │
      │                  │                  │ users (...)      │
      │                  │                  ├─────────────────►│
      │                  │                  │                  │
      │                  │                  │ INSERT INTO      │
      │                  │                  │ accounts (...)   │
      │                  │                  ├─────────────────►│
      │                  │                  │                  │
      │                  │                  │ INSERT INTO      │
      │                  │                  │ sessions (...)   │
      │                  │                  ├─────────────────►│
      │                  │                  │                  │
      │                  │ Response with    │ Return User &    │
      │                  │ User & Session   │ Session Data     │
      │                  │◄─────────────────┤                  │
      │                  │                  │                  │
      │ 201 Created      │                  │                  │
      │ {user, session}  │                  │                  │
      │◄─────────────────┤                  │                  │
      │ + Set-Cookie     │                  │                  │
      │                  │                  │                  │
```

### 2. **User Login Flow**

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │    │  Auth API   │    │ Better Auth │    │  Database   │
└─────┬───────┘    └─────┬───────┘    └─────┬───────┘    └─────┬───────┘
      │                  │                  │                  │
      │ POST /api/auth/  │                  │                  │
      │ sign-in/email    │                  │                  │
      ├─────────────────►│                  │                  │
      │ {email, password}│                  │                  │
      │                  │                  │                  │
      │                  │ Create Request   │                  │
      │                  │ Object           │                  │
      │                  ├─────────────────►│                  │
      │                  │                  │                  │
      │                  │                  │ SELECT FROM      │
      │                  │                  │ accounts WHERE   │
      │                  │                  │ email = ?        │
      │                  │                  ├─────────────────►│
      │                  │                  │                  │
      │                  │                  │ Compare Password │
      │                  │                  │ Hash             │
      │                  │                  │                  │
      │                  │                  │ CREATE NEW       │
      │                  │                  │ SESSION          │
      │                  │                  ├─────────────────►│
      │                  │                  │                  │
      │                  │ 200 OK           │ Return User &    │
      │                  │ {user, session}  │ Session          │
      │                  │◄─────────────────┤                  │
      │                  │                  │                  │
      │ 200 OK           │                  │                  │
      │ + Session Cookie │                  │                  │
      │◄─────────────────┤                  │                  │
      │                  │                  │                  │
```

### 3. **Protected Route Access Flow**

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │    │Auth Middleware│   │ Better Auth │    │  Database   │
└─────┬───────┘    └─────┬───────┘    └─────┬───────┘    └─────┬───────┘
      │                  │                  │                  │
      │ GET /api/v1/auth/│                  │                  │
      │ me               │                  │                  │
      ├─────────────────►│                  │                  │
      │ Cookie: session  │                  │                  │
      │                  │                  │                  │
      │                  │ Extract Session  │                  │
      │                  │ Token from       │                  │
      │                  │ Headers/Cookies  │                  │
      │                  ├─────────────────►│                  │
      │                  │                  │                  │
      │                  │                  │ SELECT FROM      │
      │                  │                  │ sessions WHERE   │
      │                  │                  │ token = ?        │
      │                  │                  ├─────────────────►│
      │                  │                  │                  │
      │                  │                  │ Check Expiration │
      │                  │                  │                  │
      │                  │                  │ SELECT FROM      │
      │                  │                  │ users WHERE      │
      │                  │                  │ id = session.    │
      │                  │                  │ userId           │
      │                  │                  ├─────────────────►│
      │                  │                  │                  │
      │                  │ Session Valid    │ Return User Data │
      │                  │ + User Data      │                  │
      │                  │◄─────────────────┤                  │
      │                  │                  │                  │
      │                  │ Attach user to   │                  │
      │                  │ req.user         │                  │
      │                  │                  │                  │
      │                  │ Call next()      │                  │
      │                  │ middleware       │                  │
      │                  │                  │                  │
      │ 200 OK           │                  │                  │
      │ {user: {...}}    │                  │                  │
      │◄─────────────────┤                  │                  │
      │                  │                  │                  │
```

---

## API Endpoints Flow

### Authentication Endpoints Structure

```
/api/auth/                          ← Better Auth Handler
├── sign-up/email                   ← Registration
├── sign-in/email                   ← Login
├── sign-out                        ← Logout
├── forget-password                 ← Password Reset Request
├── reset-password                  ← Password Reset Execution
└── verify-email                    ← Email Verification

/api/v1/auth/                       ← Custom Auth Routes
├── me                              ← Get Current User
├── users                           ← Get All Users (Admin)
└── users/role                      ← Update User Role (Admin)
```

### Route Protection Levels

```
┌─────────────────────────────────────────────────────────────┐
│                    Route Protection Matrix                   │
├─────────────────────────────────────────────────────────────┤
│ Route                    │ Protection Level │ Required Role  │
├─────────────────────────────────────────────────────────────┤
│ POST /sign-up/email      │ None            │ Public         │
│ POST /sign-in/email      │ None            │ Public         │
│ POST /sign-out           │ Session         │ Authenticated  │
│ GET /api/v1/auth/me      │ Session         │ Authenticated  │
│ GET /api/v1/auth/users   │ Role-based      │ Admin          │
│ PUT /api/v1/auth/users/  │ Role-based      │ Admin          │
│     role                 │                 │                │
│ PATCH /devices/:id/      │ Role-based      │ Admin          │
│       approve            │                 │                │
└─────────────────────────────────────────────────────────────┘
```

---

## Role-Based Access Control

### Role Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                      Role Hierarchy                         │
└─────────────────────────────────────────────────────────────┘

                    ┌─────────────┐
                    │    ADMIN    │
                    │             │
                    │ • Full Access│
                    │ • User Mgmt  │
                    │ • Device Mgmt│
                    │ • System Cfg │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  MODERATOR  │
                    │             │
                    │ • Read Users │
                    │ • Device Mgmt│
                    │ • Limited Cfg│
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    USER     │
                    │             │
                    │ • View Own   │
                    │   Profile    │
                    │ • Basic Info │
                    └─────────────┘
```

### Role Implementation Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Request   │    │Auth Middleware│   │  Database   │    │  Response   │
└─────┬───────┘    └─────┬───────┘    └─────┬───────┘    └─────┬───────┘
      │                  │                  │                  │
      │ Protected Route  │                  │                  │
      │ Request          │                  │                  │
      ├─────────────────►│                  │                  │
      │                  │                  │                  │
      │                  │ 1. Check Session │                  │
      │                  │ 2. Get User ID   │                  │
      │                  │                  │                  │
      │                  │ SELECT role FROM │                  │
      │                  │ users WHERE      │                  │
      │                  │ id = ?           │                  │
      │                  ├─────────────────►│                  │
      │                  │                  │                  │
      │                  │ Return user role │                  │
      │                  │◄─────────────────┤                  │
      │                  │                  │                  │
      │                  │ 3. Check if role │                  │
      │                  │    matches       │                  │
      │                  │    required      │                  │
      │                  │                  │                  │
      │                  │ ✓ Role Valid     │                  │
      │                  │ Call next()      │                  │
      │                  │                  │                  │
      │                  │ OR               │                  │
      │                  │                  │                  │
      │ 403 Forbidden    │ ✗ Role Invalid   │                  │
      │ Access Denied    │ Return Error     │                  │
      │◄─────────────────┤                  │                  │
      │                  │                  │                  │
```

---

## Session Management

### Session Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    Session Lifecycle                        │
└─────────────────────────────────────────────────────────────┘

1. SESSION CREATION (Login)
   ┌─────────────┐
   │   Login     │
   │  Request    │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐    ┌─────────────────────────────────┐
   │  Generate   │    │ Session Properties:             │
   │ Session ID  │    │ • id: unique identifier         │
   │   & Token   │    │ • token: authentication token   │
   └──────┬──────┘    │ • expiresAt: 7 days from now   │
          │           │ • userId: reference to user     │
          ▼           │ • ipAddress: client IP          │
   ┌─────────────┐    │ • userAgent: client info        │
   │   Store in  │    └─────────────────────────────────┘
   │  Database   │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │ Set Cookie  │
   │   in User   │
   │  Browser    │
   └─────────────┘

2. SESSION VALIDATION (Each Request)
   ┌─────────────┐
   │  Request    │
   │ with Cookie │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │  Extract    │
   │ Session     │
   │   Token     │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐    ┌─────────────┐
   │   Lookup    │    │    Check    │
   │  Session    ├───►│ Expiration  │
   │ in Database │    │    Time     │
   └─────────────┘    └──────┬──────┘
                             │
                             ▼
                      ┌─────────────┐
                      │   Valid?    │
                      └──────┬──────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
             ┌─────────────┐   ┌─────────────┐
             │    Allow    │   │   Reject    │
             │   Request   │   │   Request   │
             └─────────────┘   └─────────────┘

3. SESSION EXPIRATION & CLEANUP
   ┌─────────────┐
   │   Session   │
   │  Reaches    │
   │ Expiry Time │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │   Delete    │
   │  Session    │
   │from Database│
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │ Clear User  │
   │   Cookie    │
   └─────────────┘
```

### Session Security Features

```
┌─────────────────────────────────────────────────────────────┐
│                   Security Features                         │
├─────────────────────────────────────────────────────────────┤
│ Feature              │ Implementation                       │
├─────────────────────────────────────────────────────────────┤
│ Token Generation     │ Cryptographically secure random     │
│ Session Expiration   │ 7 days default, configurable        │
│ IP Tracking          │ Store client IP for audit           │
│ User Agent Tracking  │ Store browser info for security     │
│ Automatic Cleanup    │ Remove expired sessions              │
│ Cookie Security      │ HttpOnly, Secure, SameSite          │
│ Session Rotation     │ New token on sensitive operations   │
└─────────────────────────────────────────────────────────────┘
```

---

## Security Features

### Password Security

```
┌─────────────────────────────────────────────────────────────┐
│                    Password Security                        │
└─────────────────────────────────────────────────────────────┘

1. PASSWORD REGISTRATION
   User Password: "password123"
           │
           ▼
   ┌─────────────┐
   │    Salt     │ ← Random bytes generated
   │ Generation  │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │   Bcrypt    │ ← Hash with salt (10 rounds)
   │   Hashing   │
   └──────┬──────┘
          │
          ▼
   Stored Hash: "$2b$10$abcd...xyz"

2. PASSWORD VERIFICATION
   Login Password: "password123"
           │
           ▼
   ┌─────────────┐
   │   Extract   │
   │ Stored Hash │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │   Bcrypt    │
   │  Compare    │
   └──────┬──────┘
          │
      ┌───▼───┐
      │Match? │
      └───┬───┘
          │
    ┌─────┴─────┐
    │           │
    ▼           ▼
   Allow      Deny
  Access     Access
```

### Complete Authentication Entity Relationship

```
┌─────────────────────────────────────────────────────────────┐
│              Complete Entity Relationship                   │
└─────────────────────────────────────────────────────────────┘

    ┌─────────────────┐
    │      USERS      │
    │                 │
    │ • id (PK)       │◄──┐
    │ • name          │   │
    │ • email         │   │ One User
    │ • emailVerified │   │ Has Many
    │ • image         │   │ Accounts
    │ • role          │   │
    │ • createdAt     │   │
    │ • updatedAt     │   │
    └─────────────────┘   │
             │            │
             │ One User   │
             │ Has Many   │
             │ Sessions   │
             │            │
             ▼            │
    ┌─────────────────┐   │
    │    SESSIONS     │   │
    │                 │   │
    │ • id (PK)       │   │
    │ • expiresAt     │   │
    │ • token         │   │
    │ • createdAt     │   │
    │ • updatedAt     │   │
    │ • ipAddress     │   │
    │ • userAgent     │   │
    │ • userId (FK)   │───┘
    └─────────────────┘
                          
    ┌─────────────────┐   
    │    ACCOUNTS     │   
    │                 │   
    │ • id (PK)       │   
    │ • accountId     │   
    │ • providerId    │   
    │ • userId (FK)   │───┘
    │ • accessToken   │
    │ • refreshToken  │
    │ • idToken       │
    │ • password      │
    │ • scope         │
    │ • ...ExpiresAt  │
    │ • createdAt     │
    │ • updatedAt     │
    └─────────────────┘
             │
             │ One Account
             │ May Have
             │ Verifications
             ▼
    ┌─────────────────┐
    │  VERIFICATIONS  │
    │                 │
    │ • id (PK)       │
    │ • identifier    │
    │ • value         │
    │ • expiresAt     │
    │ • createdAt     │
    │ • updatedAt     │
    └─────────────────┘
```

This comprehensive documentation covers every aspect of your authentication system, from database design to API flows and security features. Each diagram shows how data flows through your system and how security is maintained at every level.
