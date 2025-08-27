Structure of a JWT
==================

A JSON Web Token (JWT) is made up of **three distinct parts**, separated by dots (`.`):

`header.payload.signature`

Each part has its own purpose, and together they make the token both useful and secure.

1\. Header
----------

The header is a **small JSON object** that describes metadata about the token.

-   **alg**: the algorithm used to sign the token (e.g., HS256).
    - `Signing` here means the server adds a special proof (using a secret key) that the token really came from the server and hasn't been changed. We'll learn the details in the verification flow later, but for now just know it's a way to prevent tampering.

-   **typ**: the type of token, usually `"JWT"`.

Example (before encoding):

`{ "alg": "HS256", "typ": "JWT" }`

This JSON is then encoded with **Base64URL** so it becomes a compact string, safe to send in web requests.

Encoded example:

`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`

**Purpose**: tells the server *how* the token was signed so it can verify it correctly.

2\. Payload
-----------

The payload is the **main content** of the JWT.\
It contains **claims**, which are pieces of information about the user and the token itself.

-   **Standard claims** (commonly used):

    -   `sub`: subject (user id).

    -   `exp`: expiration time.

    -   `iat`: issued at time.

-   **Custom claims** (specific to your app):

    -   `role`: user's role (manager, receptionist, etc.).

    -   `permissions`: what the user is allowed to do.

Example (before encoding):

`{ "sub": "123", "role": "manager", "exp": 1724659335 }`

After Base64URL encoding, it looks like:

`eyJzdWIiOiIxMjMiLCJyb2xlIjoibWFuYWdlciIsImV4cCI6MTcyNDY1OTMzNX0`

**Purpose**: carries the actual information the server needs (who the user is, when the token expires, what they're allowed to do).


3\. Signature
-------------

The signature is the **proof** that the token hasn't been changed.

-   It's created by combining the encoded header and payload, then signing them with the server's secret key.

-   Result is also encoded in Base64URL.

Example (shortened):

`TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ`

**Purpose**: ensures the token is **authentic**. If someone tampers with the header or payload, the signature won't match, and the server will reject it.


4\. Full JWT Example
--------------------

When you put all three parts together, you get the complete JWT:

`<header_b64>.<payload_b64>.<signature_b64>`

Example (shortened):

`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjMiLCJyb2xlIjoibWFuYWdlciIsImV4cCI6MTcyNDY1OTMzNX0.
TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ`


Summary
-------

-   **Header** → metadata (algorithm + type).

-   **Payload** → claims (user id, expiry, roles).

-   **Signature** → proof it hasn't been tampered with.

-   Together they form the **JWT**, a compact string used as the access token in web systems.
