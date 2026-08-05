# OpenID Connect

> **In one line —** the identity layer OAuth2 was missing: it adds an ID token that actually says *who* the user is.

| | |
|---|---|
| **Category** | Authentication Protocol |
| **Architectural Layer** | Authentication |
| **Abbreviation** | OIDC |
| **Built on** | [OAuth2](OAuth2.md) |
| **Related notes** | [OAuth2](OAuth2.md) · [JWT](JWT.md) · [Authentication](Authentication.md) · [Session Store](../08%20-%20Databases%20and%20Data/Session%20Store.md) |

---

## 1. Short Definition

*What is it?*

OpenID Connect is a thin authentication layer on top of [OAuth2](OAuth2.md). It adds an **ID token** — a [JWT](JWT.md) containing verified claims about the user — plus a standard `/userinfo` endpoint and a discovery document.

```text
OAuth2   "this app may read your calendar"        → authorisation
OIDC     "this user is ana@example.com, verified" → authentication
```

---

## 2. Purpose

*What is its main purpose?*

To make "Sign in with X" a standard, verifiable protocol rather than a different bespoke integration for every provider.

---

## 3. Problem

*What engineering problem does it solve?*

OAuth2 alone gives you an access token, which proves an app was granted access — not who the user is. Applications routinely misused it for login, which produced real account-takeover vulnerabilities.

```text
✗ OAuth2 misused for login
    receive an access token → call an API → "this returned Ana's data,
    so the user must be Ana"
    ↓
    A token stolen from another application also returns Ana's data.
    Nothing here proves the user authenticated with YOU.

✓ OIDC
    receive an ID TOKEN → cryptographically signed by the provider
    → contains sub, iss, aud, nonce
    → verifiably issued FOR YOUR APPLICATION, for THIS login
```

> [!IMPORTANT]
> This is the whole reason OIDC exists. The `aud` and `nonce` claims are what make an ID token unusable if replayed from somewhere else.

---

## 4. What OIDC adds to OAuth2

| Addition | Purpose |
|---|---|
| **ID token** | A signed JWT of identity claims |
| **`openid` scope** | Requests identity, not just resource access |
| **`/userinfo` endpoint** | Fetch profile claims with the access token |
| **Discovery document** | `/.well-known/openid-configuration` — endpoints and keys |
| **JWKS endpoint** | Public keys for verification, supporting rotation |
| **`nonce`** | Binds the token to this specific login attempt |

---

## 5. The ID token

```json
{
  "iss": "https://accounts.google.com",   // who issued it
  "aud": "your-client-id",                // WHO IT IS FOR ← verify this
  "sub": "108423...",                     // stable user identifier
  "email": "ana@example.com",
  "email_verified": true,
  "nonce": "a1b2c3",                      // ties it to your login request
  "exp": 1740000000,
  "iat": 1739996400
}
```

> [!CAUTION]
> **`sub` is the identity, not `email`.** People change email addresses, and some providers allow reuse. Key your user records on `iss` + `sub`, and treat email as a mutable attribute.
>
> Also check **`email_verified`**. An unverified email from a provider is a claim, not a fact — and account-linking on an unverified address is an account-takeover path.

---

## 6. Architecture Position

```text
User
    ↓
Your application (the OIDC "relying party")
    ↓  redirect with scope=openid, state, nonce, PKCE
Identity Provider  (Google, Keycloak, Okta)
    ↓  authorisation code
Your application — exchanges it server-side
    ↓
ID token + access token (+ refresh token)
    ↓
Verify the ID token → create YOUR OWN session
```

> [!TIP]
> The last step matters. After verifying the ID token you normally issue **your own session or token**. The ID token is proof of a login event, not a long-lived credential — it is not meant to be presented on every subsequent request.

---

## 7. Real World Example

- **"Sign in with Google / Microsoft / Apple"** — all OIDC.
- **Corporate single sign-on** — Okta, Entra ID and Keycloak all speak it.
- **Kubernetes** authenticates users against an OIDC provider.
- **Cross-application SSO** inside one organisation, without each application handling passwords.

---

## 8. OIDC vs SAML

```text
SAML     XML, browser POST bindings, enterprise legacy, still widespread
OIDC     JSON + JWT, mobile- and SPA-friendly, simpler, the modern default
```

If you are integrating with older enterprise identity systems you will still meet SAML. For anything new, OIDC.

---

## 9. Communication and Dependencies

- **An identity provider** supporting OIDC
- **The discovery document** — endpoints and JWKS, fetched and cached
- **HTTPS** and exactly-matched redirect URIs
- **A JWT library** that verifies signature, `iss`, `aud`, `exp` and `nonce`

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use OIDC whenever you want users to sign in with an existing account, or when an organisation needs single sign-on. **Delegating authentication means you never store passwords** — which removes an entire category of risk from your system.

> [!CAUTION]
> It is not free of trade-offs: you depend on the provider's availability, users without such an account are excluded, and account linking (same person, two providers) is genuinely fiddly. Many products offer both OIDC and a local password option, which means maintaining both paths.

---

## 11. Advantages and Disadvantages

**Advantages**
- You never handle or store passwords
- Standardised — one implementation works with many providers
- MFA, breach detection and account recovery become the provider's problem
- Enterprise SSO out of the box
- Discovery and JWKS make key rotation automatic

**Disadvantages**
- A hard dependency on an external provider
- Users must have an account with a supported provider
- Account linking across providers is awkward
- More moving parts than a password form
- Provider outages become your outages

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Login** | Several redirects — a one-time cost per session |
| **ID token verification** | Signature check; cache the JWKS keys |
| **`/userinfo`** | An extra HTTP call — use ID token claims where possible |
| **Ongoing requests** | None, once you have issued your own session |

---

## 13. Security Considerations

> [!CAUTION]
> Verifying an ID token means checking **all** of these. Skipping any one of them has produced real vulnerabilities:

```text
signature   against the provider's JWKS keys
iss         exactly the expected issuer
aud         exactly YOUR client id        ← prevents token replay from another app
exp / iat   not expired
nonce       matches the one you sent      ← prevents replay of an old login
```

- **Verify `state`** on the redirect — CSRF protection, as in OAuth2
- **Use PKCE**, for every client type
- **Key on `iss` + `sub`**, never on email alone
- **Check `email_verified`** before linking accounts by email address
- **Issue your own session** rather than accepting the ID token on every request
- **Handle provider outages** — decide in advance what happens when login is unavailable

---

## 14. Mental Model

> [!NOTE]
> **OAuth2 is a hotel key card; OIDC adds a photo ID.**
>
> The key card says which doors open. The photo ID says who is holding it — issued by a recognised authority, with your name on it, and verifiably meant for you rather than borrowed from someone else. The `aud` claim is the "your name on it" part.

---

## 15. Mini Architecture Diagram

```text
Browser
    ↓
Your app → redirect (openid scope, state, nonce, PKCE)
    ↓
Identity Provider — user authenticates, possibly with MFA
    ↓
code → your app (server-side exchange)
    ↓
ID token + access token
    ↓
Verify: signature · iss · aud · exp · nonce
    ↓
Create your own session → normal application flow
```

---

## 16. Complete Request Flow

```text
User clicks "Sign in with Google"
    ↓
App generates state, nonce and a PKCE verifier
    ↓
Redirect with scope=openid email profile
    ↓
User authenticates at Google (MFA is Google's responsibility)
    ↓
Redirect back with a code and the same state
    ↓
State verified → code exchanged server-side with the PKCE verifier
    ↓
ID token received and verified:
    signature via JWKS · iss = accounts.google.com · aud = our client id
    exp valid · nonce matches
    ↓
Look up or create a user, keyed on (iss, sub)
    ↓
YOUR OWN session issued — httpOnly, Secure, SameSite
    ↓
The ID token has done its job and is discarded
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> OIDC adds a verifiable ID token to OAuth2, so you can prove who the user is — verify `aud` and `nonce`, key on `sub`, and then issue your own session.

---

## 18. Common Mistakes

- **Using OAuth2 without OIDC for login**
- **Not verifying `aud`** — accepting tokens issued for another application
- **Not verifying `nonce`** — replay of an old login
- **Keying users on email** instead of `sub`
- **Ignoring `email_verified`** when linking accounts
- **Using the ID token as a session token** on every request
- **Not caching JWKS**, or not handling key rotation
- **Hand-rolling verification** instead of using a certified library

---

## 19. Open Source Technologies

- **Keycloak**, **Ory Hydra/Kratos**, **Authentik**, **Zitadel** — self-hosted providers
- **Auth0**, **Okta**, **Entra ID**, **Clerk** — managed
- **authlib**, **openid-client** (Node), **Spring Security OAuth2 Client**, **MSAL**
- **`/.well-known/openid-configuration`** — read a real one; it is short and clarifying
- **OpenID Foundation certified libraries** — prefer these over your own verification code

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Fetch a provider's `/.well-known/openid-configuration` and identify the JWKS and token endpoints.
- [ ] Check whether your integration verifies `aud` and `nonce`, not only the signature.
- [ ] Decide what your application does if the identity provider is unavailable.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
User → your app → identity provider → ID token → verification → your session
```

## 2. Request Flow

```text
Input       a login request redirected to an identity provider
    ↓
Processing  code exchanged for an ID token; signature, iss, aud, exp, nonce verified
    ↓
Output      a verified identity, converted into your own session
```

## 3. Real-World Usage

**Every "Sign in with…" button** is OIDC. Its practical value is that you never store a password, never handle MFA, and never manage account recovery — all of which are difficult to do well and expensive to get wrong.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An identity layer on top of OAuth2, adding a verifiable ID token |
| **Why does it exist?** | Because OAuth2 alone cannot prove who the user is |
| **Where does it belong?** | Between your application and an identity provider |
| **When should I use it?** | For social login and enterprise SSO — verifying every claim |
