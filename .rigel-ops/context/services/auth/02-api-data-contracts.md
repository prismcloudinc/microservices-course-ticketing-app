---
service_id: ticketing-app/auth
service_name: auth
prompt_id: 02-api-data-contracts
prompt_label: API & Data Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.20587500000000003
schema_version: "2.0"
---

# Auth Service API Contract

## Part A: HTTP API Endpoints

### GET /api/users/health

- **Controller**: `services/auth/src/routes/health.ts`, anonymous arrow function (line 5)
- **Authentication**: None
- **Authorization**: None
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers: None
- **Processing**:
  1. Returns HTTP 200 with empty object immediately
- **Response**:
  - Success: `200 OK`
    ```json
    {}
    ```
  - Error cases: None
- **Side effects**: None

---

### GET /api/users/current-user

- **Controller**: `services/auth/src/routes/current-user.ts`, anonymous arrow function (line 5-8)
- **Authentication**: `currentUserMiddleware` from `@b.anik/ticketing-common` (applied globally in `app.ts:37`) — extracts JWT from session cookie and populates `req.currentUser`
- **Authorization**: None — returns `null` if unauthenticated rather than rejecting
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers: Session cookie containing JWT (optional)
- **Processing**:
  1. `currentUserMiddleware` parses JWT from `req.session.jwt` if present
  2. If valid JWT, populates `req.currentUser` with decoded payload `{ id, email }`
  3. Route handler returns `req.currentUser` or `null`
- **Response**:
  - Success (authenticated): `200 OK`
    ```json
    {
      "currentUser": {
        "id": "<string>",
        "email": "<string>"
      }
    }
    ```
  - Success (unauthenticated): `200 OK`
    ```json
    {
      "currentUser": null
    }
    ```
  - Error cases: None
- **Side effects**: None

---

### POST /api/users/sign-up

- **Controller**: `services/auth/src/routes/sign-up.ts`, anonymous async arrow function (line 19-52)
- **Authentication**: None
- **Authorization**: None
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body:
    ```typescript
    {
      email: string;    // Required, must be valid email format (validated by express-validator body('email').isEmail())
      password: string; // Required, 4-20 characters (validated by body('password').isLength({ min: 4, max: 20 }))
    }
    ```
  - Headers: `Content-Type: application/json`
- **Processing**:
  1. `validateRequestMiddleware` checks express-validator results — throws validation error if invalid (`sign-up.ts:18`)
  2. Extract `email` and `password` from request body (`sign-up.ts:20`)
  3. Query MongoDB for existing user with same email (`sign-up.ts:22`)
  4. If user exists, throw `BadRequestError('Email in use')` (`sign-up.ts:24-26`)
  5. Create new User document via `User.build({ email, password })` (`sign-up.ts:30-33`)
  6. Save user — pre-save hook hashes password using `Password.toHash()` (`models/user.ts:41-48`)
  7. Generate JWT containing `{ id, email }` signed with `JWT_KEY` env var (`sign-up.ts:37-42`)
  8. Store JWT in session cookie (`sign-up.ts:45-47`)
  9. Return 201 with user data (`sign-up.ts:49-51`)
- **Response**:
  - Success: `201 Created`
    ```json
    {
      "user": {
        "id": "<string>",
        "email": "<string>"
      }
    }
    ```
    Headers: `Set-Cookie: session=<base64-encoded-jwt>; path=/; httponly`
  - Error cases:
    - `400 Bad Request` — Validation error (invalid email format, password length)
      ```json
      {
        "errors": [
          {
            "message": "Email must be valid",
            "field": "email"
          }
        ]
      }
      ```
      or
      ```json
      {
        "errors": [
          {
            "message": "Password must be between 4 and 20 characters",
            "field": "password"
          }
        ]
      }
      ```
    - `400 Bad Request` — Duplicate email
      ```json
      {
        "errors": [
          {
            "message": "Email in use"
          }
        ]
      }
      ```
- **Side effects**:
  - Creates a new document in MongoDB `users` collection
  - Sets session cookie with JWT

---

### POST /api/users/sign-in

- **Controller**: `services/auth/src/routes/sign-in.ts`, anonymous async arrow function (line 17-45)
- **Authentication**: None
- **Authorization**: None
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body:
    ```typescript
    {
      email: string;    // Required, must be valid email format (validated by body('email').isEmail())
      password: string; // Required, must not be empty (validated by body('password').notEmpty())
    }
    ```
  - Headers: `Content-Type: application/json`
- **Processing**:
  1. `validateRequestMiddleware` checks express-validator results — throws validation error if invalid (`sign-in.ts:16`)
  2. Extract `email` and `password` from request body (`sign-in.ts:18`)
  3. Query MongoDB for user with matching email (`sign-in.ts:20`)
  4. If user not found, throw `BadRequestError('Invalid credentials')` (`sign-in.ts:22-24`)
  5. Compare supplied password with stored hash using `Password.compare()` (`sign-in.ts:26`)
  6. If passwords don't match, throw `BadRequestError('Invalid credentials')` (`sign-in.ts:28-30`)
  7. Generate JWT containing `{ id, email }` signed with `JWT_KEY` env var (`sign-in.ts:32-38`)
  8. Store JWT in session cookie (`sign-in.ts:40-42`)
  9. Return 200 with empty object (`sign-in.ts:44`)
- **Response**:
  - Success: `200 OK`
    ```json
    {}
    ```
    Headers: `Set-Cookie: session=<base64-encoded-jwt>; path=/; httponly`
  - Error cases:
    - `400 Bad Request` — Validation error (invalid email format, empty password)
      ```json
      {
        "errors": [
          {
            "message": "Email must be valid",
            "field": "email"
          }
        ]
      }
      ```
      or
      ```json
      {
        "errors": [
          {
            "message": "Password must be provided",
            "field": "password"
          }
        ]
      }
      ```
    - `400 Bad Request` — Invalid credentials (user not found or wrong password)
      ```json
      {
        "errors": [
          {
            "message": "Invalid credentials"
          }
        ]
      }
      ```
- **Side effects**:
  - Sets session cookie with JWT

---

### POST /api/users/sign-out

- **Controller**: `services/auth/src/routes/sign-out.ts`, anonymous arrow function (line 7-10)
- **Authentication**: `currentUserMiddleware` (global, `app.ts:37`) + `requireAuthMiddleware` (route-level, `sign-out.ts:7`)
- **Authorization**: Must be authenticated (JWT must be valid)
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers: Session cookie containing valid JWT (required)
- **Processing**:
  1. `currentUserMiddleware` extracts JWT and populates `req.currentUser`
  2. `requireAuthMiddleware` verifies `req.currentUser` exists — throws 401 if not (`sign-out.ts:7`)
  3. Clear session by setting `req.session = null` (`sign-out.ts:8`)
  4. Return 200 with empty object (`sign-out.ts:10`)
- **Response**:
  - Success: `200 OK`
    ```json
    {}
    ```
    Headers: `Set-Cookie: session=; path=/; expires=Thu, 01 Jan 1970 00:00:00 GMT; httponly`
  - Error cases:
    - `401 Unauthorized` — Not authenticated (no JWT or invalid JWT)
      ```json
      {
        "errors": [
          {
            "message": "Not authorized"
          }
        ]
      }
      ```
- **Side effects**:
  - Clears session cookie

---

### ALL /* (404 Handler)

- **Controller**: `services/auth/src/app.ts`, anonymous arrow function (line 45-47)
- **Authentication**: `currentUserMiddleware` (runs before, `app.ts:37`)
- **Authorization**: None
- **Request**: Any unmatched route
- **Processing**:
  1. Throws `NotFoundError()` from `@b.anik/ticketing-common`
  2. `errorHandlerMiddleware` catches and formats the error
- **Response**:
  - Error: `404 Not Found`
    ```json
    {
      "errors": [
        {
          "message": "Not Found"
        }
      ]
    }
    ```

---

## Part B: Database Schema

### User

- **Database collection name**: `users` (inferred — Mongoose lowercases and pluralizes model name "User")
- **Purpose**: Stores user account credentials for authentication
- **Source**: `services/auth/src/models/user.ts`

| Field | Type | Required | Default | Index | Description |
|-------|------|----------|---------|-------|-------------|
| `_id` | ObjectId | Yes | Auto-generated | Primary | MongoDB document ID; transformed to `id` in JSON output (`user.ts:31`) |
| `email` | String | Yes | None | None | User's email address; used for login lookup (`user.ts:20-22`) |
| `password` | String | Yes | None | None | Scrypt-hashed password in `<hash>.<salt>` format; omitted from JSON output (`user.ts:32`) |
| `__v` | Number | Yes | 0 | None | Mongoose version key; omitted from JSON output (`user.ts:33`) |

- **Relationships**: None — User is a standalone entity in the auth service
- **Enums**: None
- **Compound indexes**: None defined
- **Common query patterns**:
  - Find by email: `User.findOne({ email })` (`sign-in.ts:20`, `sign-up.ts:22`) — used for login and duplicate detection

**JSON Transformation** (`user.ts:30-36`):
When serialized to JSON, the User document is transformed:
- `_id` → `id` (rename)
- `password` → removed (security)
- `__v` → removed (internal)

**Pre-save Hook** (`user.ts:41-48`):
Before saving, if `password` field is modified:
1. Hash password using `Password.toHash()` (scrypt with random 8-byte salt, 64-byte output)
2. Store as `<hex-hash>.<hex-salt>` format

---

## Part C: Type Definitions & DTOs

### UserAttrs

- **Name**: `UserAttrs`
- **File path**: `services/auth/src/models/user.ts:4-7`
- **Full definition**:
  ```typescript
  interface UserAttrs {
    email: string;
    password: string;
  }
  ```
- **Where used**: Input parameter for `User.build()` static method — type-safe user creation (`sign-up.ts:30-33`)

---

### UserDoc

- **Name**: `UserDoc`
- **File path**: `services/auth/src/models/user.ts:13-16`
- **Full definition**:
  ```typescript
  interface UserDoc extends mongoose.Document {
    email: string;
    password: string;
  }
  ```
- **Where used**: Return type of Mongoose queries and `User.build()`; represents a saved User document

---

### UserModel

- **Name**: `UserModel`
- **File path**: `services/auth/src/models/user.ts:9-11`
- **Full definition**:
  ```typescript
  interface UserModel extends mongoose.Model<any> {
    build(attrs: UserAttrs): UserDoc;
  }
  ```
- **Where used**: Type for the User model exported from `user.ts:56`; enables TypeScript-aware static `build()` method

---

### JWT Payload (inferred from usage)

- **Name**: JWT Payload (not explicitly named in code)
- **File path**: Generated in `services/auth/src/routes/sign-in.ts:32-36` and `services/auth/src/routes/sign-up.ts:37-41`
- **Full definition**:
  ```typescript
  {
    id: string;    // User's MongoDB ObjectId as string
    email: string; // User's email address
  }
  ```
- **Where used**: 
  - Encoded into JWT on sign-in and sign-up
  - Decoded by `currentUserMiddleware` and stored in `req.currentUser`
  - Returned by `/api/users/current-user` endpoint

---

### Session Shape (inferred from usage)

- **Name**: Session (managed by cookie-session)
- **File path**: Set in `services/auth/src/routes/sign-in.ts:40-42` and `services/auth/src/routes/sign-up.ts:45-47`
- **Full definition**:
  ```typescript
  {
    jwt: string; // JWT token string
  }
  ```
- **Where used**: Stored in session cookie; read by `currentUserMiddleware`

---

### Request Body: Sign-Up (inferred from validation)

- **Name**: Sign-Up Request Body (not explicitly typed)
- **File path**: `services/auth/src/routes/sign-up.ts:13-17` (validation), line 20 (cast)
- **Full definition**:
  ```typescript
  {
    email: string;    // Must match email format (express-validator isEmail())
    password: string; // Must be 4-20 characters (express-validator isLength({ min: 4, max: 20 }))
  }
  ```
- **Where used**: Request body for `POST /api/users/sign-up`

---

### Request Body: Sign-In (inferred from validation)

- **Name**: Sign-In Request Body (not explicitly typed)
- **File path**: `services/auth/src/routes/sign-in.ts:14-15` (validation), line 18 (cast)
- **Full definition**:
  ```typescript
  {
    email: string;    // Must match email format (express-validator isEmail())
    password: string; // Must not be empty (express-validator notEmpty())
  }
  ```
- **Where used**: Request body for `POST /api/users/sign-in`

---

### Response: Current User

- **Name**: Current User Response (not explicitly typed)
- **File path**: `services/auth/src/routes/current-user.ts:6-8`
- **Full definition**:
  ```typescript
  {
    currentUser: {
      id: string;
      email: string;
    } | null;
  }
  ```
- **Where used**: Response body for `GET /api/users/current-user`

---

### Response: Sign-Up Success

- **Name**: Sign-Up Success Response (not explicitly typed)
- **File path**: `services/auth/src/routes/sign-up.ts:49-51`
- **Full definition**:
  ```typescript
  {
    user: {
      id: string;    // MongoDB ObjectId as string
      email: string; // User's email
      // password and __v are omitted by toJSON transform
    };
  }
  ```
- **Where used**: Response body for successful `POST /api/users/sign-up` (201)

---

### Password Service

- **Name**: `Password` (class with static methods)
- **File path**: `services/auth/src/services/password.ts:6-20`
- **Full definition**:
  ```typescript
  class Password {
    static async toHash(password: string): Promise<string>;
    // Returns: "<64-byte-scrypt-hash-hex>.<8-byte-salt-hex>"
    
    static async compare(storedPassword: string, suppliedPassword: string): Promise<boolean>;
    // storedPassword format: "<hash>.<salt>"
  }
  ```
- **Where used**: 
  - `toHash()`: Called in User pre-save hook (`models/user.ts:43`)
  - `compare()`: Called in sign-in route (`routes/sign-in.ts:26`)

---

## External Dependencies (from `@b.anik/ticketing-common`)

The following exports are imported from the external npm package `@b.anik/ticketing-common` (version ^0.1.2). The source code is not in this repository, so these definitions are inferred from usage patterns:

| Export | Type | Usage in Auth Service | Inferred Behavior |
|--------|------|----------------------|-------------------|
| `NotFoundError` | Error class | `app.ts:46` — thrown for unmatched routes | Produces 404 response with `{ errors: [{ message: "Not Found" }] }` |
| `BadRequestError` | Error class | `sign-in.ts:23,29`, `sign-up.ts:25` — thrown for validation/business errors | Produces 400 response; constructor takes message string |
| `currentUserMiddleware` | Express middleware | `app.ts:37` — global middleware | Extracts JWT from `req.session.jwt`, decodes it, and sets `req.currentUser` |
| `requireAuthMiddleware` | Express middleware | `sign-out.ts:7` — route-level protection | Throws 401 error if `req.currentUser` is not set |
| `validateRequestMiddleware` | Express middleware | `sign-in.ts:16`, `sign-up.ts:18` — after validators | Checks express-validator results; throws error with validation details if invalid |
| `errorHandlerMiddleware` | Express middleware | `app.ts:48` — final error handler | Catches errors and formats them to `{ errors: [...] }` JSON structure |

---

## Cookie Session Configuration

Configured in `services/auth/src/app.ts:27-32`:

```typescript
cookieSession({
  signed: false,           // Cookie is not cryptographically signed (JWT provides integrity)
  secure: process.env.NODE_ENV !== 'test',  // HTTPS-only except in test environment
})
```

Cookie name: `session` (cookie-session default)
