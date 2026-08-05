# OAuth2

> **In one line —** a protocol for granting an application limited access to your data on another service, without ever giving it your password.

| | |
|---|---|
| **Category** | Authorization Framework |
| **Architectural Layer** | Authentication / Authorization |
| **Standard** | RFC 6749, plus many extensions |
| **Related notes** | [OpenID Connect](OpenID%20Connect.md) · [JWT](JWT.md) · [Authentication](Authentication.md) · [Authorization](Authorization.md) · [API Design](../04%20-%20Networking%20and%20Internet/API%20Design.md) |

---

## 1. Short Definition

*What is it?*

OAuth2 is a **delegated authorisation** framework. It lets a user grant one application scoped access to their data held by another, using a token instead of sharing credentials.

> [!IMPORTANT]
> **OAuth2 is authorisation, not authentication.** It answers "may this app read your calendar?", not "who are you?". Using it for login requires **[OpenID Connect](OpenID%20Connect.md)**, which is a layer built on top of it. This distinction is the single most common source of confusion, and getting it wrong has produced real vulnerabilities.

---

## 2. Purpose

*What is its main purpose?*

To eliminate password sharing between services, and to make access **scoped, revocable and expiring** rather than total and permanent.

---

## 3. Problem

*What engineering problem does it solve?*

```text
BEFORE OAUTH
"Give us your Gmail password so we can find your friends"
    ↓
The app has FULL access, forever, and stores your password
A breach at that app is a breach of your email

WITH OAUTH
"This app wants to read your contacts"  → you approve at Google
    ↓
The app receives a token scoped to contacts, expiring, revocable
It never sees your password
```

---

## 4. The four roles

```text
RESOURCE OWNER        you, the user
CLIENT                the application requesting access
AUTHORIZATION SERVER  issues tokens (Google, GitHub, your Keycloak)
RESOURCE SERVER       holds the data and accepts the token (the API)
```

---

## 5. Architecture Position

```text
User
  ↓ approves
Authorization Server  ──issues token──►  Client application
                                              ↓ Bearer token
                                         Resource Server (API)
```

---

## 6. Authorization Code flow with PKCE — the one to use

> [!IMPORTANT]
> This is the **only** flow you should implement for new applications, whether web, mobile or single-page. PKCE (Proof Key for Code Exchange) is now recommended for all client types, not just mobile.

```text
1. Client generates a random verifier, sends its SHA-256 hash (challenge)
    ↓
2. Redirect to the authorization server with the challenge + state
    ↓
3. User authenticates and approves the requested scopes
    ↓
4. Redirect back with a short-lived AUTHORIZATION CODE
    ↓
5. Client exchanges the code + original verifier for tokens
       ← an intercepted code is useless without the verifier
    ↓
6. Access token (short) + refresh token (revocable)
    ↓
7. Client calls the API with:  Authorization: Bearer <access token>
```

---

## 7. The deprecated flows

```text
✗ IMPLICIT             returned tokens directly in the URL fragment
                       → leaked via history, referrers, logs. Deprecated.

✗ PASSWORD (ROPC)      the app collects the user's password directly
                       → defeats the entire purpose. Deprecated.

✓ CLIENT CREDENTIALS   machine-to-machine, no user involved. Still correct.

✓ DEVICE CODE          TVs and CLIs — "go to this URL and enter this code"
```

> [!CAUTION]
> If a tutorial shows the implicit flow or asks for the user's password, it is outdated. Both were removed from current OAuth 2.1 guidance.

---

## 8. Scopes

```text
scope=contacts.read calendar.write
    ↓
The token grants exactly those permissions, and nothing else
```

> [!TIP]
> **Request the minimum scope you need.** Users notice broad permission requests, and an over-scoped token is a larger prize if your application is ever compromised.

---

## 9. Real World Example

- **"Sign in with Google/GitHub/Apple"** — OAuth2 plus OpenID Connect.
- **A calendar app reading your Google Calendar** — pure OAuth2, no login involved.
- **GitHub Actions accessing repositories** with a scoped token.
- **Internal microservices** using the client credentials flow.

---

## 10. Communication and Dependencies

- **An authorization server** — Google, GitHub, Auth0, Keycloak, Ory
- **HTTPS everywhere** — non-negotiable; redirects carry codes and tokens
- **Registered redirect URIs**, matched exactly
- **A token store** for refresh tokens

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use OAuth2 when a third-party application needs access to user data you hold, or when you want to access another service's API on a user's behalf. Use **OIDC** when you want login.

> [!CAUTION]
> Do not use OAuth2 for authentication on its own — an access token proves an app was granted access, not who the user is. This confusion has produced real account-takeover vulnerabilities. If you want to know who someone is, you need an **ID token** from OIDC.
>
> Also, do not build your own authorization server. It is a genuinely difficult specification with many subtle requirements; use Keycloak, Ory, or a managed provider.

---

## 12. Advantages and Disadvantages

**Advantages**
- Passwords are never shared with third parties
- Access is scoped, expiring and revocable
- An industry standard with mature libraries everywhere
- Users can review and revoke access centrally

**Disadvantages**
- Genuinely complex — many flows, extensions and edge cases
- Easy to implement insecurely
- Requires HTTPS and careful redirect handling
- Dependency on an external provider's availability
- The authorisation/authentication distinction trips up most people

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Login** | Several redirects — a one-time cost |
| **API calls** | Token verification only; no extra round trip if it is a JWT |
| **Token introspection** | Opaque tokens require a call to the auth server per request |
| **Refresh** | Occasional; should be transparent to the user |

---

## 14. Security Considerations

> [!CAUTION]
> Most OAuth2 vulnerabilities come from the same handful of omissions:

- **Always use the `state` parameter** and verify it on return — this is the CSRF defence for the redirect
- **Always use PKCE**, for every client type
- **Validate redirect URIs exactly** — an open redirect lets an attacker steal the authorisation code
- **Never put tokens in URLs** — they end up in browser history, referrer headers and server logs
- **Verify the token audience** — a token issued for another application must be rejected
- **Store refresh tokens securely**, and revoke them on logout
- **Do not treat an access token as proof of identity** — that is the OIDC ID token's job

---

## 15. Mental Model

> [!NOTE]
> **OAuth2 is a hotel key card.**
>
> Reception (the authorization server) verifies you and issues a card that opens your room and the gym, expires on checkout, and can be cancelled at any moment. You never hand out the master key, and the card says nothing about who you are — it only says which doors open. That last part is exactly why login needs OpenID Connect on top.

---

## 16. Mini Architecture Diagram

```text
User ──approves──► Authorization Server
                          │ authorization code
                          ▼
                      Client app
                          │ code + PKCE verifier
                          ▼
                   Authorization Server
                          │ access + refresh tokens
                          ▼
                      Client app
                          │ Bearer access token
                          ▼
                   Resource Server (API)
```

---

## 17. Complete Request Flow

```text
User clicks "Connect Google Calendar"
    ↓
App generates a PKCE verifier and a random state
    ↓
Redirect to Google with client_id, scope, redirect_uri, state, code_challenge
    ↓
User authenticates at Google and approves the calendar scope
    ↓
Google redirects back with a code and the same state
    ↓
App VERIFIES the state matches  ← CSRF defence
    ↓
App exchanges code + verifier for tokens, server-side
    ↓
Access token (1 h) + refresh token stored securely
    ↓
API calls: Authorization: Bearer <access token>
    ↓
Access token expires → refresh silently
    ↓
User revokes access at Google → refresh fails → the app loses access
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> OAuth2 delegates scoped, revocable access without sharing passwords — it authorises rather than authenticates, and login requires OpenID Connect on top of it.

---

## 19. Common Mistakes

- **Using OAuth2 alone for login** instead of OIDC
- **Omitting or not verifying `state`** — a CSRF hole
- **Skipping PKCE**
- **Loose redirect URI matching**, enabling code theft
- **Implicit or password flows** in new code
- **Requesting excessive scopes**
- **Not verifying the token audience**
- **Building your own authorization server**

---

## 20. Open Source Technologies

- **Keycloak**, **Ory Hydra**, **Authentik** — self-hosted authorization servers
- **Auth0**, **Okta**, **Clerk** — managed providers
- **authlib** (Python), **Passport.js**, **Spring Security OAuth**, **MSAL**
- **OAuth 2.1** and the **Security Best Current Practice** RFC — current guidance
- **PKCE (RFC 7636)** — required for all clients

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Check whether your OAuth integration uses PKCE and verifies the `state` parameter.
- [ ] List the scopes your application requests and remove any it does not actually use.
- [ ] Trace one full login redirect chain in DevTools and identify each parameter.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
User → Authorization Server → code → Client → tokens → Resource Server
```

## 2. Request Flow

```text
Input       a user's consent to a scoped access request
    ↓
Processing  authorization code exchanged with PKCE for scoped tokens
    ↓
Output      an access token granting limited, expiring, revocable access
```

## 3. Real-World Usage

**Every "Connect your account" integration** — calendar, storage, payments, repositories — is OAuth2. Before it existed, these integrations genuinely asked users for their passwords, and the consequences were exactly as bad as they sound.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A framework for delegated, scoped access to a user's data |
| **Why does it exist?** | So applications never need to hold another service's password |
| **Where does it belong?** | Between a client, an authorization server and a resource API |
| **When should I use it?** | Third-party data access — with OIDC on top when you need login |
