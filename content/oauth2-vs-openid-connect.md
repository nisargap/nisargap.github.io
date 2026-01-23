+++
title = "OAuth2 vs OpenID Connect: Authentication and Authorization Explained"
date = 2026-01-23
description = "OAuth2 handles authorization, OpenID Connect handles authentication. Understanding the difference is critical for building secure systems that know who you are and what you can access."
[taxonomies]
tags = ["security", "authentication", "authorization", "oauth2", "openid-connect", "web-security"]
categories = ["Engineering"]
+++

When building modern web applications, you need two fundamental capabilities: knowing who your users are (authentication) and controlling what they can do (authorization). OAuth2 and OpenID Connect are the industry standards for these problems, but they're often confused because OpenID Connect is built on top of OAuth2. Understanding the distinction is essential for implementing security correctly.

<!-- more -->

## The Core Distinction

**OAuth2** is an authorization framework. It answers the question: "What can this user access?" It allows applications to obtain limited access to user resources without exposing passwords.

**OpenID Connect (OIDC)** is an authentication protocol built on OAuth2. It answers the question: "Who is this user?" It provides a standardized way to verify identity and get basic profile information.

The confusion arises because OpenID Connect extends OAuth2. Every OpenID Connect implementation uses OAuth2, but not every OAuth2 implementation provides authentication. Using OAuth2 alone for authentication is a common security mistake.

## Why OAuth2 Exists: The Delegation Problem

Before OAuth2, if you wanted to grant a third-party application access to your data on another service, you had to give it your password. This was catastrophic for security:

- The third-party app could access everything, not just what you intended
- You couldn't revoke access without changing your password everywhere
- The app could store your password indefinitely
- No way to limit what the app could do

Consider a concrete example: You want a photo printing service to access your photos stored in Google Drive. Pre-OAuth2, you'd give the printing service your Google password. Now it can read your emails, delete files, access your calendar—everything you can do.

OAuth2 solves this with delegated authorization. You grant the printing service specific, limited access without revealing your password. You can revoke access anytime without changing your password.

## How OAuth2 Works: The Authorization Flow

OAuth2 defines several "flows" (grant types). The most common is the Authorization Code flow, which works like this:

```
User                    Client App              Authorization Server    Resource Server
  |                         |                           |                     |
  |-- 1. Click "Login" ---->|                           |                     |
  |                         |                           |                     |
  |<-- 2. Redirect to ------|                           |                     |
  |    authorization URL --------------------------->|                     |
  |                         |                           |                     |
  |<-- 3. Login prompt ----------------------------------                    |
  |                         |                           |                     |
  |-- 4. Enter credentials & grant permission --------->|                     |
  |                         |                           |                     |
  |<-- 5. Redirect with authorization code --------------|                     |
  |-- Send code ----------->|                           |                     |
  |                         |                           |                     |
  |                         |-- 6. Exchange code for -->|                     |
  |                         |    access token           |                     |
  |                         |                           |                     |
  |                         |<-- 7. Access token -------|                     |
  |                         |                           |                     |
  |                         |-- 8. Request resource with token ------------->|
  |                         |                           |                     |
  |                         |<-- 9. Protected resource -----------------------|
```

**Step-by-step breakdown**:

1. User clicks "Login with Google" (or similar)
2. App redirects to authorization server (e.g., Google's OAuth endpoint)
3. Authorization server shows login prompt and permission request
4. User authenticates and grants permission
5. Authorization server redirects back to app with authorization code
6. App exchanges code for access token (happening server-to-server)
7. App receives access token (and optionally refresh token)
8. App uses access token to access protected resources
9. Resource server validates token and returns data

The authorization code is single-use and short-lived (minutes). The exchange in step 6 happens server-to-server with client authentication, preventing interception attacks. The access token is what actually grants access to resources.

## OAuth2 Components

**Resource Owner**: The user who owns the data. You.

**Client**: The application requesting access. The photo printing service.

**Authorization Server**: Issues access tokens after authenticating the user and getting consent. Google's OAuth server.

**Resource Server**: Hosts the protected resources. Validates tokens and returns data. Google Drive API.

Often the authorization server and resource server are the same system, but they're logically separate.

## OAuth2 Tokens

**Access Token**: Credentials used to access protected resources. Usually a JWT (JSON Web Token) or opaque string. Short-lived (minutes to hours). The client includes this in API requests: `Authorization: Bearer <access_token>`

**Refresh Token**: Long-lived credentials used to obtain new access tokens without requiring the user to log in again. Stored securely by the client. When the access token expires, exchange the refresh token for a new access token.

Access tokens are intentionally short-lived to limit exposure if compromised. Refresh tokens are long-lived (days to months) but require client authentication to use, making them harder to exploit.

## OAuth2 Scopes

Scopes define the permissions being requested. They're space-separated strings that represent specific access levels:

```
https://authorization-server.com/authorize?
  client_id=abc123&
  redirect_uri=https://client-app.com/callback&
  scope=read:photos write:photos&
  response_type=code
```

Common scope patterns:
- `read:photos` - Read access to photos
- `write:photos` - Write access to photos
- `openid profile email` - OpenID Connect scopes (we'll get to this)

The authorization server shows these scopes to the user: "Photo Printing Service wants to: View your photos, Upload photos." The user can accept or deny.

Scopes are critical for least privilege. Request only what you need. If you just need to read photos, don't request write access.

## OAuth2 Grant Types

OAuth2 defines several flows for different scenarios:

**Authorization Code Flow**: Most secure. Used by web apps with a backend. The authorization code is exchanged server-side where client secrets are secure.

**Authorization Code Flow with PKCE**: Enhanced version for mobile and single-page apps that can't securely store client secrets. PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks.

**Implicit Flow**: Deprecated. Tokens returned directly in the URL fragment. Used for SPAs before PKCE existed. Don't use this anymore.

**Client Credentials Flow**: Machine-to-machine authentication. No user involved. The app authenticates with its own credentials to access resources it owns.

**Resource Owner Password Credentials Flow**: User provides username/password directly to the client app, which exchanges them for tokens. Only use this for first-party apps where the client is completely trusted.

For modern applications: use Authorization Code with PKCE for SPAs and mobile apps, Authorization Code for traditional web apps, and Client Credentials for service-to-service.

## The Problem: OAuth2 Isn't Authentication

Here's the critical mistake: OAuth2 access tokens tell you what someone can access, not who they are.

Say you build an app that uses "Login with Google." You get an access token that allows accessing Google APIs. But that access token doesn't tell you which user authorized it. Two different users could theoretically produce tokens that look identical to your app.

The access token format is undefined in OAuth2. It might be a JWT with user info, or an opaque string. You could call the resource server API to get user data, but there's no standard endpoint or format. Different providers implement this differently.

Worse, if you use the access token as a session identifier (a common mistake), you're vulnerable to attacks. Access tokens are meant for API access, not session management. They're often passed in URLs, logged, cached—all dangerous for session tokens.

OAuth2 explicitly states it's not an authentication protocol. Using it as one requires workarounds that often have security flaws.

## Enter OpenID Connect: Authentication Done Right

OpenID Connect (OIDC) extends OAuth2 to add a standardized authentication layer. It defines exactly how to verify user identity and retrieve user information.

OIDC adds three key things to OAuth2:

1. **ID Token**: A JWT that contains verified identity claims
2. **UserInfo Endpoint**: A standard API for retrieving user profile information
3. **Standard Scopes and Claims**: Defined ways to request specific user data

When you use OpenID Connect, the authorization flow returns both an access token (for API access) and an ID token (for authentication).

## The ID Token

The ID token is a JWT (JSON Web Token) that contains claims about the authenticated user:

```json
{
  "iss": "https://accounts.google.com",
  "sub": "10769150350006150715113082367",
  "aud": "1234567890.apps.googleusercontent.com",
  "exp": 1311281970,
  "iat": 1311280970,
  "name": "Jane Doe",
  "email": "jane@example.com",
  "email_verified": true,
  "picture": "https://example.com/photo.jpg"
}
```

Key claims:

- `iss` (issuer): Who issued this token
- `sub` (subject): Unique identifier for the user (never changes, never reused)
- `aud` (audience): Your application's client ID (prevents token reuse)
- `exp` (expiration): When this token expires
- `iat` (issued at): When this token was issued
- Additional profile claims: name, email, picture, etc.

The ID token is signed by the authorization server. Your app verifies the signature using the server's public key. This proves:

1. The token came from the legitimate authorization server
2. The token hasn't been tampered with
3. The token is meant for your application (aud claim)
4. The token is still valid (exp claim)

You don't call an API to verify the user—you validate the cryptographic signature. This makes authentication fast and doesn't require network calls.

## OpenID Connect Scopes

OIDC defines standard scopes:

- `openid`: Required. Signals that you want OIDC authentication, not just OAuth2 authorization
- `profile`: Grants access to name, family_name, given_name, middle_name, nickname, picture, etc.
- `email`: Grants access to email and email_verified
- `address`: Grants access to address claim
- `phone`: Grants access to phone_number and phone_number_verified

Request what you need:

```
https://authorization-server.com/authorize?
  client_id=abc123&
  redirect_uri=https://client-app.com/callback&
  scope=openid profile email&
  response_type=code
```

The ID token will include the requested claims (depending on what the authorization server supports and the user consents to).

## UserInfo Endpoint

For claims not included in the ID token, OIDC defines a standard UserInfo endpoint:

```bash
GET /userinfo
Authorization: Bearer <access_token>
```

Response:
```json
{
  "sub": "10769150350006150715113082367",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "email_verified": true,
  "picture": "https://example.com/photo.jpg",
  "updated_at": 1311280970
}
```

The `sub` claim in the UserInfo response matches the `sub` in the ID token, tying them together. Use the access token to call UserInfo, then match `sub` values to ensure they refer to the same user.

## OpenID Connect Flow

The Authorization Code flow with OIDC looks nearly identical to OAuth2, but with OIDC-specific additions:

```
User                    Client App              Authorization Server
  |                         |                           |
  |-- 1. Click "Login" ---->|                           |
  |                         |                           |
  |<-- 2. Redirect with ----------------------------------
  |    scope=openid profile email                      |
  |                         |                           |
  |-- 3. Authenticate and consent -------------------->|
  |                         |                           |
  |<-- 4. Redirect with authorization code -------------|
  |-- Send code ----------->|                           |
  |                         |                           |
  |                         |-- 5. Exchange code ------>|
  |                         |                           |
  |                         |<-- 6. Access token + -----|
  |                         |    ID token               |
  |                         |                           |
  |                         |-- 7. Validate ID token ---|
  |                         |    (verify signature)     |
  |                         |                           |
  |<-- 8. User logged in ---|                           |
```

The key differences:

1. Request includes `openid` scope
2. Response includes ID token alongside access token
3. Client validates ID token signature and claims
4. User is now authenticated (you know who they are)

You can optionally call the UserInfo endpoint with the access token for additional claims.

## When to Use OAuth2 vs OpenID Connect

**Use OAuth2 when**:
- You need to access user data on another service (delegated authorization)
- Machine-to-machine communication (client credentials flow)
- You're building an authorization system but not handling user authentication
- Example: "Allow photo printing service to access my Google Drive photos"

**Use OpenID Connect when**:
- You need to authenticate users (know who they are)
- Implementing "Login with Google/GitHub/etc."
- Single Sign-On (SSO) across multiple applications
- You need verified user identity and basic profile information
- Example: "Login with Google" to access your web app

**Use both when**:
- You need authentication AND access to user data
- Example: "Login with Google and sync your calendar" (OIDC for authentication, OAuth2 scopes for calendar access)

Most "Login with X" implementations should use OpenID Connect, not plain OAuth2.

## Common Patterns and Best Practices

**1. Always validate ID tokens**

Don't just decode the JWT—verify the signature, issuer, audience, and expiration:

```javascript
const decoded = jwt.verify(idToken, publicKey, {
  algorithms: ['RS256'],
  audience: clientId,
  issuer: 'https://accounts.google.com'
});

const userId = decoded.sub;
```

**2. Use `sub` as the user identifier**

The `sub` (subject) claim is the stable, unique identifier. Email addresses can change. Names can change. `sub` never changes and is never reused.

**3. Don't use access tokens for authentication**

Access tokens are for authorization (API access). ID tokens are for authentication. Don't create sessions based on access tokens.

**4. Implement PKCE for SPAs and mobile apps**

Authorization Code flow with PKCE prevents authorization code interception attacks. It's now recommended for all public clients (apps that can't securely store secrets).

**5. Store tokens securely**

Access tokens and refresh tokens are credentials:
- SPAs: Store in memory, not localStorage (vulnerable to XSS)
- Mobile apps: Use secure storage (Keychain on iOS, Keystore on Android)
- Backend: Encrypted database or secure key management system

**6. Use short-lived access tokens**

Access tokens should expire quickly (minutes to hours). Use refresh tokens to obtain new access tokens. This limits damage if tokens are compromised.

**7. Request minimal scopes**

Follow least privilege. Only request scopes you actually need. Users are more likely to grant narrow permissions.

## Security Considerations

**Authorization Code Interception**: Without PKCE, attackers can intercept the authorization code in redirects. PKCE mitigates this by requiring a code verifier that only the legitimate client knows.

**Token Leakage**: Tokens in URLs, logs, or error messages are vulnerable. Use POST for token exchange, don't log tokens, use secure storage.

**Phishing**: Attackers can create fake authorization screens. Users should verify the domain matches the legitimate authorization server. Apps should use platform browsers (not WebViews) where users can see the URL.

**Redirect URI Validation**: Authorization servers must strictly validate redirect URIs. Attackers can hijack flows by registering malicious redirect URIs.

**Client Secret Security**: For confidential clients (web apps), never expose client secrets in frontend code. Keep them server-side.

**Token Replay**: Access tokens can be replayed if intercepted. Use HTTPS everywhere, short token lifetimes, and consider token binding for high-security scenarios.

## Real-World Implementations

**Google Identity**: Supports OAuth2 and OpenID Connect. Widely used for "Sign in with Google." Provides ID tokens with user profile information.

**Auth0**: Identity-as-a-Service platform. Implements OIDC on top of OAuth2. Supports social login, enterprise SSO, and custom identity management.

**Keycloak**: Open-source identity and access management. Full OIDC implementation with extensive customization options.

**Okta**: Enterprise identity platform. OIDC for SSO across applications. Centralized user management and authentication.

**GitHub OAuth**: Pure OAuth2 (not OIDC). You must call the `/user` API endpoint to get identity information. Shows why OIDC is better for authentication.

## OIDC Discovery and Configuration

OpenID Connect providers expose a discovery endpoint that returns metadata:

```
GET /.well-known/openid-configuration
```

Response includes:
```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "profile", "email"],
  "response_types_supported": ["code", "token", "id_token"],
  "grant_types_supported": ["authorization_code", "refresh_token"]
}
```

This allows clients to dynamically configure themselves without hardcoding endpoints. The `jwks_uri` provides the public keys for verifying ID token signatures.

## Implementing Your Own: Should You?

Building OAuth2/OIDC providers is complex and security-critical. Most applications should:

1. **Use existing providers** for user authentication (Google, GitHub, etc.)
2. **Use identity platforms** (Auth0, Okta, Keycloak) if you need custom identity management
3. **Only build your own** if you have specific requirements and security expertise

If you must implement:
- Use established libraries (don't roll your own crypto)
- Follow the specifications exactly
- Get security audits
- Implement all required validations
- Use secure token generation

The specifications are complex. OAuth2 (RFC 6749) is 76 pages. OIDC Core is 70 pages. Security Best Current Practice (RFC 8252) adds more requirements. Getting this right requires careful implementation.

## Debugging and Testing

**Tools**:
- [jwt.io](https://jwt.io): Decode and verify JWTs
- [OAuth2 Playground](https://www.oauth.com/playground/): Test OAuth2 flows
- [OIDC Playground](https://openidconnect.net): Test OIDC flows

**Common issues**:

- **Invalid signature**: Check public key, algorithm, token issuer
- **Token expired**: Check exp claim, system clock synchronization
- **Invalid audience**: Ensure aud claim matches your client ID
- **Redirect URI mismatch**: Must exactly match registered URI (including trailing slashes, query params)
- **Invalid grant**: Authorization code already used or expired

Enable verbose logging in OAuth libraries during development. Capture full HTTP requests/responses (excluding secrets) when debugging.

## The Evolution: From OAuth 1.0 to OAuth 2.1

**OAuth 1.0** (2007): Required cryptographic signatures on every request. Complex but didn't require HTTPS. Largely obsolete.

**OAuth 2.0** (2012): Simplified by requiring HTTPS. Removed signature requirements. Introduced multiple grant types. Current widely-deployed version.

**OAuth 2.1** (Draft): Consolidates best practices. Mandates PKCE for all flows. Deprecates implicit flow and password grant. Removes bearer token in query strings.

**OpenID Connect** (2014): Built on OAuth 2.0. Became the standard for authentication. Superseded OpenID 2.0 (different, older protocol).

The trend is toward simpler, more secure defaults. OAuth 2.1 codifies what's already recommended practice.

## Why This Matters

Authentication and authorization are fundamental security controls. Getting them wrong exposes users to account takeover, data breaches, and privacy violations.

OAuth2 and OpenID Connect are battle-tested standards used by billions of users daily. They handle:
- Password-less authentication
- Limited delegated access
- Single Sign-On
- Mobile and web security models
- Token-based authorization

Understanding the distinction prevents critical mistakes:
- Using OAuth2 for authentication (insecure)
- Requesting excessive scopes (privacy violation)
- Storing tokens insecurely (credential leakage)
- Improper token validation (spoofing attacks)

Modern applications are distributed systems with multiple services, third-party integrations, and diverse clients. OAuth2 and OIDC provide the foundation for secure identity and access management in this environment.

You don't need to implement these protocols from scratch—use libraries and platforms. But understanding how they work makes you better at:
- Integrating authentication securely
- Debugging authorization issues
- Designing APIs with proper access controls
- Evaluating identity platforms
- Building secure distributed systems

OAuth2 handles authorization. OpenID Connect handles authentication. Know which you need, and use the right tool for the job. Your users' security depends on it.
