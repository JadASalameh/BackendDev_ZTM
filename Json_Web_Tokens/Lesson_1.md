Introduction to JWT and Access Tokens
=====================================

What is JWT (JSON Web Token)?
-----------------------------

A **JWT** is a special string the server gives you after you log in.

-   **Definition:** A compact, URL-safe string.

-   **Purpose:** You get it once (at login), and you must show it to the server every time you make an API request.


Why, Where, and When JWT is Used
--------------------------------

### 1\. Why JWT is used

The main purpose of a JWT is **authentication** and **authorization** in web systems.

-   After login, the server needs a way to remember who the user is for future requests.

-   Instead of storing sessions in memory or a DB (**stateful**), JWTs provide a **stateless** solution.

-   The server encodes identity and permissions into the JWT.

-   Each request carries the JWT, and the server checks its **signature** to confirm authenticity.

**Advantages:**

-   **Stateless:** server keeps no session data.

-   **Scalable:** works across multiple servers / microservices.

-   **Secure:** signatures prevent tampering.

-   **Compact:** easy to send in HTTP headers or cookies.

**Example -- Online Gym Website:**

-   Manager logs in → system gives a "pass" (the JWT).

-   Without it: manager must re-enter email/password for every action.

-   With it: each request automatically carries the JWT.

-   Any service (members, payments, attendance) can verify:

    -   *"This is Jad, he's a manager, and he's allowed to see this page."*

-   If expired → system asks for login again (or uses refresh token flow).

 The **"pass"** is your **JWT access token**.\
 It's used on **every request** after login, until it expires.



### 2\. Where JWT is used

-   **Between client and server**\
    Example:

    `GET /members
    Authorization: Bearer <jwt>`

    The backend checks the JWT before serving the response.

-   **Between microservices**\
    Example: Attendance service → Auth service. The JWT proves the request is authenticated.

-   **In distributed systems / cloud environments**\
    Multiple servers can independently validate tokens without consulting a central DB.

### 3\. When JWT is used

JWTs come into play **after authentication**:

1.  **Login phase**

    -   User enters email + password.

    -   Server validates credentials.

    -   Server issues a JWT (access token).

2.  **API calls phase**

    -   Client attaches JWT to every protected API request.

    -   Server verifies **signature** and **expiry**.

3.  **Until expiration**

    -   Access tokens are short-lived (often **5--15 minutes**).

    -   After expiration, the client must request a new token (usually via a **refresh token**, explained later).

JWT Access Token: Execution Flow
-----------------------------

### 0) Setup (one-time, on the server)


-   Define protected API routes (e.g., `/members`, `/payments`).

-   Decide a short lifetime for access tokens (e.g., **15 minutes**).

-   Have a user store (e.g., `users` table with hashed passwords).


### 1) Login request

1.  **Client** shows a login form and sends:

    `POST /auth/login
    { email, password }`

2.  **Server**:

    -   Looks up the user by email.

    -   Verifies the password against the stored hash.

    -   If invalid → return **401 Unauthorized** (stop here).

    -   If valid → proceed.



### 2) Issue access token

1.  **Server** creates an **access token** that represents:

    -   *Who* the user is (their ID; optionally role/permissions).

    -   *When* the token expires (e.g., now + 15 minutes).

2.  **Server** returns:

    `{
      "access_token": "<token string>",
      "token_type": "Bearer",
      "expires_in": 900
    }`

3.  **Client** stores the access token (e.g., in memory or a safe store in the app).



### 3) Calling protected APIs

1.  **Client** calls any protected endpoint and **attaches the token**:

    `GET /members
    Authorization: Bearer <access_token>`

2.  **Server** has an **auth middleware** in front of protected routes. For every request:

    -   Extracts the token from `Authorization` header.

    -   **Validates** it (authentic + not expired).

    -   If valid → attaches the user identity to the request context (e.g., `req.user`).

    -   If invalid/expired/missing → returns **401 Unauthorized**.



### 4) Authorization (permissions)

1.  **Route handler** runs *after* the token is validated:

    -   Uses the user identity/role from the request context.

    -   Checks permissions (e.g., manager can view payments).

    -   If allowed → perform the action and return **200 OK** with data.

    -   If not allowed → return **403 Forbidden**.



### 5) Token expiration

1.  Access tokens are **short-lived** by design:

    -   If the token is **still valid** → requests keep succeeding.

    -   If the token **expires**:

        -   The next protected request will hit the middleware, fail validation, and receive **401 Unauthorized** (commonly with a message like "token expired").

2.  **Client reaction** (app logic):

    -   Option A (reactive): When it sees **401 expired**, it triggers the refresh/login flow, then retries the request.

    -   Option B (proactive): The app watches the token's expiry time and refreshes just before it expires, so users don't see failures.

    -   (Refresh is a separate topic; if not implemented, the user is sent back to the login page.)



### 6) Logout

1.  **Client** logs out by:

    -   Deleting the locally stored access token.

    -   (If you also use refresh tokens, you'd call a logout endpoint to revoke them. Without refresh tokens, simply removing the access token ends the session on the client.)

2.  Already-issued access tokens (if any still exist on that device) simply **stop working** once they expire.



### 7) Multi-service usage (if you have several backend services)

1.  The **same access token** can be presented to multiple services in your system:

    -   Each service protects its own routes with the same kind of middleware.

    -   Each service independently validates the token on every request.

    -   No central "who's logged in" lookup is required for each call.



### 8) Typical errors & handling

1.  **Missing token** → 401 Unauthorized.

2.  **Expired token** → 401 Unauthorized (app should refresh or send user to login).

3.  **Not enough permissions** (user is valid but not allowed) → 403 Forbidden.

4.  **Malformed token** (can't be read) → 401 Unauthorized.



### TL;DR 

-   **Login:** client sends credentials → server returns **access token** with a short expiry.

-   **Use:** client sends token on every protected request → server validates per request → handler runs.

-   **Expire:** when token expires, requests return **401** → client refreshes (if implemented) or re-logs in.

-   **Logout:** client deletes token; access naturally ends when no valid token is sent.
