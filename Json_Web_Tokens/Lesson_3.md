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

6\. GO Example
---------------

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"time"
)

// b64url encodes bytes using Base64URL **without padding** (what JWTs expect).
func b64url(b []byte) string {
	return base64.RawURLEncoding.EncodeToString(b)
}

// MakeJWT creates an HS256 JWT string.
// sub: user id, role: app role (example), ttl: token lifetime, secret: HS256 secret key.
func MakeJWT(sub, role string, ttl time.Duration, secret []byte) (string, error) {
	// 1) Header (JSON)
	header := map[string]any{
		"alg": "HS256",
		"typ": "JWT",
	}
	headerJSON, err := json.Marshal(header)
	if err != nil {
		return "", fmt.Errorf("marshal header: %w", err)
	}
	headerB64 := b64url(headerJSON)

	// 2) Payload (claims)
	now := time.Now().Unix()
	payload := map[string]any{
		"sub": sub,        // subject (user id)
		"role": role,      // custom claim (example)
		"iat":  now,       // issued at
		"exp":  now + int64(ttl.Seconds()), // expiration time
		"iss":  "https://auth.example.com", // issuer (example)
		"aud":  "mma-api", // audience (example)
	}
	payloadJSON, err := json.Marshal(payload)
	if err != nil {
		return "", fmt.Errorf("marshal payload: %w", err)
	}
	payloadB64 := b64url(payloadJSON)

	// 3) Signing input = "<header>.<payload>"
	signingInput := headerB64 + "." + payloadB64

	// 4) Signature = HMAC_SHA256(signingInput, secret), then Base64URL (no padding)
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(signingInput))
	sig := mac.Sum(nil)
	sigB64 := b64url(sig)

	// 5) Final token
	return signingInput + "." + sigB64, nil
}

func main() {
	// Example "after login": you verified email+password already.
	secret := []byte("my_super_secret_key_change_me") // store via env/secret manager in real apps

	token, err := MakeJWT(
		"550e8400-e29b-41d4-a716-446655440000", // sub (user id)
		"manager",                               // role
		15*time.Minute,                          // 15m access token
		secret,
	)
	if err != nil {
		panic(err)
	}

	fmt.Println("Access JWT:")
	fmt.Println(token)
}
```

