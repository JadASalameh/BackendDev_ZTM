JWT Creation and Verification Flow
=====================================

So far, we know what a JWT is (a token with header, payload, and signature).\
Now we'll see **how JWTs are created by the server, sent to the client, and later verified on every request.**

This flow has two sides:

1.  **Creation (server builds the JWT at login)**

2.  **Verification (server checks the JWT on each API request)**


1\. Creating a JWT (Server Side at Login)
-----------------------------------------

**Step 1: User logs in**

-   Client sends email + password to the server.

-   Server checks credentials in the database.

**Step 2: Build the header**

-   Example:

    `{ "alg": "HS256", "typ": "JWT" }`

-   Encoded into Base64URL.

**Step 3: Build the payload**

-   Contains the user's identity and expiry time.

-   Example:

    `{ "sub": "123", "role": "manager", "exp": 1724659335 }`

-   Encoded into Base64URL.

### Step 4: Create the Signature

Now that the server has the **encoded header** and **encoded payload**, it needs to create the **signature**.

1.  **Combine the two parts**

    `header_b64 + "." + payload_b64`

2.  **Apply the signing algorithm**

    -   The header says `"alg": "HS256"`.

    -   HS256 = HMAC (Hash-based Message Authentication Code) using SHA-256.

    -   The server takes:

        -   the combined string (`header.payload`)

        -   the **secret key** (known only to the server)

    -   And produces a hash → this becomes the signature.

3.  **Encode the signature**

    -   The result is still binary.

    -   It's then encoded in **Base64URL** to make it safe for web transport.


#### Example (simplified)

-   Header (encoded):

    `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`

-   Payload (encoded):

    `eyJzdWIiOiIxMjMiLCJyb2xlIjoibWFuYWdlciIsImV4cCI6MTcyNDY1OTMzNX0`

-   Combined:

    `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJyb2xlIjoibWFuYWdlciIsImV4cCI6MTcyNDY1OTMzNX0`

-   Signing with HS256 + secret key (`my_secret`):

    `HMAC_SHA256(combined, "my_secret")`

-   Result (Base64URL):

    `TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ`

**Step 5: Combine all three parts**

-   Format:

    `header.payload.signature`

-   That's the **JWT string** (the access token).

**Step 6: Send the JWT to the client**

-   Server returns a response like:

    `{
      "access_token": "<jwt_string>",
      "token_type": "Bearer",
      "expires_in": 900
    }`

-   Client stores it (in memory or secure storage).

2\. Using the JWT (Client Side)
-------------------------------

-   Whenever the client calls a protected API, it attaches the JWT in the **Authorization header**:

`GET /members
Authorization: Bearer <jwt_string>`


3\. Verifying a JWT (Server Side on Each Request)
-------------------------------------------------

**Step 1: Extract the token**

-   Server middleware reads the `Authorization` header.

**Step 2: Split into parts**

-   JWT has 3 parts: `header.payload.signature`.

**Step 3: Recompute the signature**

-   Server takes the header + payload (from the request).

-   Uses its **secret key** and the `alg` field from the header.

-   Recomputes what the signature *should be*.

**Step 4: Compare signatures**

-   If recomputed signature == signature from the client's token → authentic.

-   If not → reject (token was tampered with).

**Step 5: Check expiry**

-   Look at `exp` in the payload.

-   If `exp < now` → token expired → reject.

**Step 6: Accept and attach identity**

-   If valid → server trusts the claims (e.g., user ID, role).

-   Attaches identity to request (e.g., `req.user = { id: 123, role: "manager" }`).


4\. What Happens if Verification Fails
--------------------------------------

-   **Missing token** → 401 Unauthorized.

-   **Invalid signature** → 401 Unauthorized (tampered token).

-   **Expired token** → 401 Unauthorized (client must refresh or log in again).


5\. Summary
-----------

-   **Creation:** server issues JWT at login (header + payload + signature).

-   **Usage:** client attaches JWT on every request.

-   **Verification:** server checks the signature and expiry on each request.

-   If valid → request is allowed.

-   If invalid → rejected with 401.

