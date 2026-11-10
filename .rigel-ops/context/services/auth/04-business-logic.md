---
service_id: ticketing-app/auth
service_name: auth
prompt_id: 04-business-logic
prompt_label: Business Logic Extraction
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.182205
schema_version: "2.0"
---

# Business Processes: Auth Service

## Process: Application Startup

**Trigger**: Node.js process execution via `pnpm start` or `pnpm dev` scripts (runs `services/auth/src/index.ts`)

**Input**: Environment variables `JWT_KEY` and `MONGO_URI`

**Preconditions**: None

**Steps**:
1. [`start()` in `services/auth/src/index.ts:5-26`] — Async startup function invoked immediately
2. [Environment validation in `services/auth/src/index.ts:6-8`] — Checks if `process.env.JWT_KEY` is defined
   - If `JWT_KEY` is falsy: throws `Error('JWT_KEY must be defined')` and process crashes
   - Else: continues
3. [Environment validation in `services/auth/src/index.ts:10-12`] — Checks if `process.env.MONGO_URI` is defined
   - If `MONGO_URI` is falsy: throws `Error('MONGO_URI must be defined')` and process crashes
   - Else: continues
4. [MongoDB connection in `services/auth/src/index.ts:14-20`] — Calls `mongoose.connect(process.env.MONGO_URI!)`
   - If connection succeeds: logs `'Connected to MongoDB!'` to console
   - If connection throws: logs the error to console and calls `process.exit(1)`
5. [HTTP server start in `services/auth/src/index.ts:22-24`] — Calls `app.listen(3000, callback)`
   - On successful bind: logs `'Listening on port 3000!'`

**Business rules**:
- JWT_KEY environment variable is mandatory — application will not start without it
- MONGO_URI environment variable is mandatory — application will not start without it
- MongoDB connection failure is fatal — process exits with code 1
- Server binds to port 3000 unconditionally (hardcoded)

**Output**: HTTP server listening on port 3000

**Post-conditions**: 
- Express application is ready to receive HTTP requests
- Mongoose is connected to MongoDB

**Error scenarios**:
- Missing `JWT_KEY`: throws `Error`, process crashes with uncaught exception
- Missing `MONGO_URI`: throws `Error`, process crashes with uncaught exception
- MongoDB connection failure: error logged, `process.exit(1)` called

---

## Process: User Registration (Sign-Up)

**Trigger**: HTTP POST request to `/api/users/sign-up`

**Input**: JSON request body with `email` (string) and `password` (string)

**Preconditions**: None (unauthenticated endpoint)

**Steps**:
1. [`json()` middleware in `services/auth/src/app.ts:26`] — Parses JSON request body from `body-parser`
2. [`cookieSession()` middleware in `services/auth/src/app.ts:27-32`] — Initializes cookie-based session (unsigned, secure except in test environment)
3. [`currentUserMiddleware` in `services/auth/src/app.ts:37`] — Extracts JWT from session cookie and populates `req.currentUser` (call target not statically resolvable — from `@b.anik/ticketing-common`)
4. [`body('email').trim().isEmail()` in `services/auth/src/routes/sign-up.ts:13`] — Validates email field
   - If email is missing or not valid email format: validation error recorded
5. [`body('password').trim().isLength({ min: 4, max: 20 })` in `services/auth/src/routes/sign-up.ts:14-17`] — Validates password field
   - If password length < 4 or > 20 characters after trimming: validation error recorded
6. [`validateRequestMiddleware` in `services/auth/src/routes/sign-up.ts:18`] — Checks express-validator results (call target not statically resolvable — from `@b.anik/ticketing-common`)
   - If validation errors exist: throws error with 400 status (inferred based on test expectations)
   - Else: continues to route handler
7. [Route handler in `services/auth/src/routes/sign-up.ts:19-52`] — Extracts `email` and `password` from `req.body`
8. [`User.findOne({ email })` in `services/auth/src/routes/sign-up.ts:22`] — Database read: queries `users` collection for existing user with same email
   - **Database read**: MongoDB `users` collection, query `{ email: <provided_email> }`
9. [Duplicate check in `services/auth/src/routes/sign-up.ts:24-26`] — Evaluates `existingUser`
   - If `existingUser` is truthy: throws `BadRequestError('Email in use')` (400 status)
   - Else: continues
10. [`User.build({ email, password })` in `services/auth/src/routes/sign-up.ts:30-33`] — Creates new Mongoose document instance via static factory method
11. [`user.save()` in `services/auth/src/routes/sign-up.ts:35`] — Persists user document to MongoDB
    - **Pre-save hook triggers** [`userSchema.pre('save')` in `services/auth/src/models/user.ts:41-48`]:
      - Checks `this.isModified('password')`
      - If password is modified: calls `Password.toHash(this.get('password'))`
12. [`Password.toHash()` in `services/auth/src/services/password.ts:7-11`] — Hashes the password
    - Generates 8 random bytes, converts to hex for salt (16 hex chars)
    - Uses `scrypt` with password, salt, and keylen=64 to derive hash
    - Returns `${hash_hex}.${salt_hex}` format
13. [Pre-save hook completion in `services/auth/src/models/user.ts:44`] — Sets hashed password on document
14. [Document save in `services/auth/src/routes/sign-up.ts:35`] — MongoDB insert operation
    - **Database write**: MongoDB `users` collection, inserts `{ email: <email>, password: <hashed_password> }`
15. [`jwt.sign()` in `services/auth/src/routes/sign-up.ts:37-43`] — Generates JWT containing `{ id: user.id, email: user.email }`, signed with `process.env.JWT_KEY`
16. [Session assignment in `services/auth/src/routes/sign-up.ts:45-47`] — Sets `req.session = { jwt: userJwt }`
    - `cookie-session` middleware will serialize this to `Set-Cookie` header
17. [Response in `services/auth/src/routes/sign-up.ts:49-51`] — Returns HTTP 201 with JSON body `{ user: <user_object> }`

**Business rules**:
- Email must be valid email format after trimming
- Password must be between 4 and 20 characters inclusive after trimming
- Email addresses must be unique across all users in the system — duplicate registration is rejected
- Passwords are hashed using scrypt with 64-byte output and random 8-byte salt before storage
- JWT payload contains user `id` and `email`
- JWT is signed with `JWT_KEY` environment variable
- Session cookie is not signed (`signed: false` in `services/auth/src/app.ts:29`)
- Session cookie is secure (HTTPS-only) unless `NODE_ENV === 'test'` (`services/auth/src/app.ts:30`)

**Output**: 
- HTTP 201 status code
- JSON body: `{ user: { id: <string>, email: <string> } }` (password excluded by JSON transform)
- `Set-Cookie` header with session containing JWT

**Post-conditions**:
- New user document exists in MongoDB `users` collection with hashed password
- Client has session cookie containing valid JWT

**Error scenarios**:
- Invalid email format: 400 status, validation error message `'Email must be valid'` (`services/auth/src/routes/sign-up.ts:13`)
- Password too short/long: 400 status, validation error message `'Password must be between 4 and 20 characters'` (`services/auth/src/routes/sign-up.ts:17`)
- Duplicate email: 400 status, `BadRequestError` with message `'Email in use'` (`services/auth/src/routes/sign-up.ts:25`)

---

## Process: User Authentication (Sign-In)

**Trigger**: HTTP POST request to `/api/users/sign-in`

**Input**: JSON request body with `email` (string) and `password` (string)

**Preconditions**: None (unauthenticated endpoint)

**Steps**:
1. [`json()` middleware in `services/auth/src/app.ts:26`] — Parses JSON request body
2. [`cookieSession()` middleware in `services/auth/src/app.ts:27-32`] — Initializes session
3. [`currentUserMiddleware` in `services/auth/src/app.ts:37`] — Extracts current user from existing session (if any)
4. [`body('email').trim().isEmail()` in `services/auth/src/routes/sign-in.ts:14`] — Validates email field
   - If email is missing or not valid email format: validation error recorded
5. [`body('password').trim().notEmpty()` in `services/auth/src/routes/sign-in.ts:15`] — Validates password field
   - If password is empty after trimming: validation error recorded
6. [`validateRequestMiddleware` in `services/auth/src/routes/sign-in.ts:16`] — Checks validation results
   - If validation errors exist: throws error with 400 status (inferred)
   - Else: continues
7. [Route handler in `services/auth/src/routes/sign-in.ts:17-45`] — Extracts credentials from body
8. [`User.findOne({ email })` in `services/auth/src/routes/sign-in.ts:20`] — Database read: looks up user by email
   - **Database read**: MongoDB `users` collection, query `{ email: <provided_email> }`
9. [User existence check in `services/auth/src/routes/sign-in.ts:22-24`] — Evaluates result
   - If `user` is falsy: throws `BadRequestError('Invalid credentials')` (400 status)
   - Else: continues
10. [`Password.compare(user.password, password)` in `services/auth/src/routes/sign-in.ts:26`] — Verifies supplied password against stored hash
11. [`Password.compare()` in `services/auth/src/services/password.ts:14-19`] — Password verification logic
    - Splits stored password by `.` to extract `[hashedPassword, salt]`
    - Re-hashes supplied password with same salt using `scrypt` (keylen=64)
    - Returns `true` if resulting hash matches stored hash, `false` otherwise
12. [Password match check in `services/auth/src/routes/sign-in.ts:28-30`] — Evaluates comparison result
    - If `passwordsMatch` is false: throws `BadRequestError('Invalid credentials')` (400 status)
    - Else: continues
13. [`jwt.sign()` in `services/auth/src/routes/sign-in.ts:32-38`] — Generates JWT containing `{ id: user.id, email: user.email }`, signed with `process.env.JWT_KEY`
14. [Session assignment in `services/auth/src/routes/sign-in.ts:40-42`] — Sets `req.session = { jwt: userJwt }`
15. [Response in `services/auth/src/routes/sign-in.ts:44`] — Returns HTTP 200 with empty JSON body `{}`

**Business rules**:
- Email must be valid email format after trimming
- Password must not be empty after trimming
- User with provided email must exist in database
- Supplied password must match stored hashed password when verified with same salt
- Same generic error message `'Invalid credentials'` used for both non-existent user and wrong password (security best practice to prevent user enumeration)
- JWT payload contains user `id` and `email` (same structure as sign-up)

**Output**:
- HTTP 200 status code
- JSON body: `{}` (empty object)
- `Set-Cookie` header with session containing JWT

**Post-conditions**:
- No database changes
- Client has session cookie containing valid JWT for authenticated user

**Error scenarios**:
- Invalid email format: 400 status, validation error message `'Email must be valid'` (`services/auth/src/routes/sign-in.ts:14`)
- Empty password: 400 status, validation error message `'Password must be provided'` (`services/auth/src/routes/sign-in.ts:15`)
- User not found by email: 400 status, `BadRequestError` with message `'Invalid credentials'` (`services/auth/src/routes/sign-in.ts:23`)
- Password mismatch: 400 status, `BadRequestError` with message `'Invalid credentials'` (`services/auth/src/routes/sign-in.ts:29`)

---

## Process: User Sign-Out

**Trigger**: HTTP POST request to `/api/users/sign-out`

**Input**: None (body ignored)

**Preconditions**: 
- User must be authenticated (valid JWT in session cookie)
- Enforced by `requireAuthMiddleware` (call target not statically resolvable — from `@b.anik/ticketing-common`)

**Steps**:
1. [`json()` middleware in `services/auth/src/app.ts:26`] — Parses request (body typically empty)
2. [`cookieSession()` middleware in `services/auth/src/app.ts:27-32`] — Loads existing session
3. [`currentUserMiddleware` in `services/auth/src/app.ts:37`] — Extracts JWT from session, populates `req.currentUser`
4. [`requireAuthMiddleware` in `services/auth/src/routes/sign-out.ts:7`] — Validates authentication
   - If `req.currentUser` is not set: returns 401 status (inferred behavior from middleware name)
   - Else: continues
5. [Route handler in `services/auth/src/routes/sign-out.ts:7-11`] — Sets `req.session = null`
   - `cookie-session` middleware will set `Set-Cookie` header with expired cookie
6. [Response in `services/auth/src/routes/sign-out.ts:10`] — Returns HTTP 200 with empty JSON body `{}`

**Business rules**:
- Only authenticated users can sign out (protected by `requireAuthMiddleware`)
- Session destruction is achieved by setting `req.session = null`
- Cookie expiration is handled by `cookie-session` middleware (sets expiry to epoch: `Thu, 01 Jan 1970 00:00:00 GMT` per test at `services/auth/src/routes/__tests__/sign-out.test.ts:15-17`)

**Output**:
- HTTP 200 status code
- JSON body: `{}`
- `Set-Cookie` header: `session=; path=/; expires=Thu, 01 Jan 1970 00:00:00 GMT; httponly`

**Post-conditions**:
- No database changes
- Client's session cookie is invalidated/expired

**Error scenarios**:
- Unauthenticated request: 401 status (inferred from `requireAuthMiddleware` behavior — call target not statically resolvable)

---

## Process: Get Current User

**Trigger**: HTTP GET request to `/api/users/current-user`

**Input**: Session cookie containing JWT (optional)

**Preconditions**: None (works for both authenticated and unauthenticated requests)

**Steps**:
1. [`json()` middleware in `services/auth/src/app.ts:26`] — Parses request
2. [`cookieSession()` middleware in `services/auth/src/app.ts:27-32`] — Loads session if cookie present
3. [`currentUserMiddleware` in `services/auth/src/app.ts:37`] — Extracts JWT from `req.session.jwt` and verifies signature
   - If valid JWT: decodes payload and sets `req.currentUser = { id, email }`
   - If no JWT or invalid: `req.currentUser` remains `undefined`
   - (call target not statically resolvable — from `@b.anik/ticketing-common`)
4. [Route handler in `services/auth/src/routes/current-user.ts:5-9`] — Reads `req.currentUser`
5. [Response in `services/auth/src/routes/current-user.ts:6-8`] — Returns JSON with `currentUser` property
   - If `req.currentUser` exists: returns `{ currentUser: { id, email } }`
   - Else: returns `{ currentUser: null }` (explicit null via `|| null` fallback)

**Business rules**:
- Endpoint is always accessible (no auth middleware)
- Returns `null` for `currentUser` when not authenticated (not undefined)
- User data comes from JWT payload, not database lookup (no DB query in this flow)

**Output**:
- HTTP 200 status code
- JSON body: `{ currentUser: { id: <string>, email: <string> } | null }`

**Post-conditions**: None (read-only operation)

**Error scenarios**: None explicitly handled — invalid JWT is treated as unauthenticated (`currentUser: null`)

---

## Process: Health Check

**Trigger**: HTTP GET request to `/api/users/health`

**Input**: None

**Preconditions**: None

**Steps**:
1. [Route handler in `services/auth/src/routes/health.ts:5`] — Immediately returns response
   - Note: `healthRouter` is mounted before `currentUserMiddleware` (`services/auth/src/app.ts:34` vs `services/auth/src/app.ts:37`), so no JWT processing occurs

**Business rules**:
- Health check bypasses authentication middleware (mounted before `currentUserMiddleware`)
- Always returns success if application is running

**Output**:
- HTTP 200 status code
- JSON body: `{}` (empty object)

**Post-conditions**: None

**Error scenarios**: None — if the process is running and accepting connections, health check succeeds

---

## Process: Request to Unknown Route (404 Handler)

**Trigger**: HTTP request (any method) to any path not matching defined routes

**Input**: Any HTTP request

**Preconditions**: Request did not match any defined route handler

**Steps**:
1. [Middleware chain execution] — Request passes through all route handlers without match
2. [`app.all('*')` in `services/auth/src/app.ts:45-47`] — Catch-all route handler fires
3. [Handler in `services/auth/src/app.ts:46`] — Throws `NotFoundError()` (from `@b.anik/ticketing-common`)
4. [`errorHandlerMiddleware` in `services/auth/src/app.ts:48`] — Catches error and formats response
   - (call target not statically resolvable — from `@b.anik/ticketing-common`)

**Business rules**:
- All unmatched routes result in `NotFoundError` being thrown
- Error handler middleware formats the error into consistent JSON response (format not statically resolvable)

**Output**: HTTP 404 status (inferred from error class name)

**Post-conditions**: None

**Error scenarios**: This IS the error scenario — handled by design

---

## Supporting Service: Password Hashing

**Location**: `services/auth/src/services/password.ts`

This is not a standalone business process but a supporting service used by sign-up and sign-in.

### Password.toHash(password: string)

**Steps** (`services/auth/src/services/password.ts:7-11`):
1. Generate 8 random bytes using `crypto.randomBytes(8)`
2. Convert to hex string (produces 16-character salt)
3. Derive 64-byte key using `scrypt(password, salt, 64)`
4. Return `${hash_hex}.${salt_hex}` (128 hex chars for hash + `.` + 16 hex chars for salt)

**Formula**: `scrypt(plaintext_password, random_salt, keylen=64)` → 128 hex chars

### Password.compare(storedPassword: string, suppliedPassword: string)

**Steps** (`services/auth/src/services/password.ts:14-19`):
1. Split `storedPassword` by `.` to extract `[hashedPassword, salt]`
2. Derive key using `scrypt(suppliedPassword, salt, 64)`
3. Compare resulting hash (hex) with stored `hashedPassword`
4. Return `true` if equal, `false` otherwise

---

## Data Transformations

### User Document JSON Serialization

**Location**: `services/auth/src/models/user.ts:30-36`

When a User document is serialized to JSON (via `toJSON` schema option):
1. `ret.id = ret._id` — Copies MongoDB ObjectId to `id` field
2. `delete ret.password` — Removes password hash from output
3. `delete ret.__v` — Removes Mongoose version key
4. `delete ret._id` — Removes original `_id` field

**Result**: API responses contain `{ id, email }` only, never `password` or MongoDB internals.
