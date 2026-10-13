# Okta OIDC Sign-In + Protected API (OAuth 2.0 / API Access Management)

A hands-on lab that signs users in with **Okta** using **OpenID Connect (OIDC)** and protects an API with **OAuth 2.0 access tokens**. A small Flask app (`oidc_lab.py`) does the full Authorization Code flow, shows what's inside the tokens, and tests a protected API endpoint against valid, invalid, and under-privileged tokens.

![Okta](https://img.shields.io/badge/Okta-Integrator%20Free%20Plan-007DC1)
![Python](https://img.shields.io/badge/Python-3.9-blue)
![Flask](https://img.shields.io/badge/Flask-3.1-lightgrey)
![OIDC](https://img.shields.io/badge/OIDC-Authorization%20Code-green)

---

## Table of contents

1. [What this project shows](#what-this-project-shows)
2. [Key concepts](#key-concepts)
3. [Architecture](#architecture)
4. [Prerequisites](#prerequisites)
5. [Step 1: Configure the authorization server](#step-1-configure-the-authorization-server)
6. [Step 2: Create the OIDC app in Okta](#step-2-create-the-oidc-app-in-okta)
7. [Step 3: Run the app](#step-3-run-the-app)
8. [Step 4: Sign in with MFA](#step-4-sign-in-with-mfa)
9. [Step 5: Inspect the tokens](#step-5-inspect-the-tokens)
10. [Step 6: Test the protected API](#step-6-test-the-protected-api)
11. [Step 7: Missing scope returns 403](#step-7-missing-scope-returns-403)
12. [Step 8: Least privilege with group assignment](#step-8-least-privilege-with-group-assignment)
13. [Terminal log of the full flow](#terminal-log-of-the-full-flow)
14. [Results summary](#results-summary)
15. [Interview questions this lab answers](#interview-questions-this-lab-answers)
16. [Security notes](#security-notes)

---

## What this project shows

- Configuring an Okta **custom authorization server** with a custom **scope** (`reports:read`) and a custom **claim** (`department`)
- Registering an **OIDC Web Application** in Okta and limiting it to one group
- Running the **Authorization Code flow** end to end, with MFA (Okta Verify)
- Reading and explaining the claims in an **ID token** and an **access token**
- Validating JWTs in an API: **signature, issuer, audience, expiry, and scope**
- The difference between **401 Unauthorized** and **403 Forbidden**
- **Centralized access control**: Okta blocks users who aren't assigned to the app before a token is ever issued

---

## Key concepts

Every app has to answer two questions:

| Question | Protocol | Token | Example |
|---|---|---|---|
| **Who are you?** (authentication) | OpenID Connect (OIDC) | **ID token** | "This is Alex Eng, alex.eng@…" |
| **What are you allowed to do?** (authorization) | OAuth 2.0 | **Access token** | "This app may read reports" |

### The hotel analogy

- The **front desk (Okta)** checks your ID.
- They give you a **receipt with your name (ID token)** so the hotel knows who you are.
- They give you a **room key card (access token)**. The **door (API)** doesn't care who you are. It only checks that the card is real, not expired, and opens this door (the **scope**).

### Terms

| Term | Meaning |
|---|---|
| **Authorization server** | The part of Okta that issues tokens. This lab uses the built-in one named `default`. |
| **Scope** | A permission, such as `reports:read`. |
| **Claim** | One piece of information inside a token, such as `email` or `department`. |
| **JWT** | The token format: a signed bundle of claims. Anyone can read it, but nobody can change it without breaking Okta's signature. |
| **Authorization Code flow** | The secure login method. Okta sends a one-time **code** (not the tokens) back to the app, and the app trades it for tokens server-side. |
| **PKCE** | An extra check that makes a stolen authorization code useless without a secret only the app holds. |
| **Audience (`aud`)** | Who the token is meant for. The ID token is for the **app**; the access token is for the **API**. |

---

## Architecture

```
┌──────────┐   1. click "Sign in"     ┌──────────────┐
│ Browser  │ ───────────────────────▶ │  Flask app   │
│          │ ◀─── 2. redirect ─────── │ localhost:   │
└────┬─────┘                          │    8080      │
     │ 3. login + MFA                 └──────┬───────┘
     ▼                                       │
┌──────────────────────┐  4. one-time code   │
│ Okta authorization   │ ──(via browser)────▶│
│ server (default)     │                     │
│                      │ ◀── 5. code + secret│
│                      │ ─── 6. ID token + ─▶│
└──────────────────────┘     access token    │
                                             ▼
                              7. call GET /api/reports
                                 with the access token
                                             │
                              ┌──────────────▼──────────┐
                              │ Protected API           │
                              │ checks signature, iss,  │
                              │ aud, exp, scope         │
                              │ → 200 / 401 / 403       │
                              └─────────────────────────┘
```

**No ngrok needed.** Okta never calls the local machine directly. The browser goes to Okta and comes back to `localhost`, and the app calls Okta over the internet. Both directions work without a tunnel.

---

## Prerequisites

- An Okta org with **API Access Management** (the free Integrator plan includes the `default` authorization server)
- Python 3.9+
- Test users and groups in Okta. This lab uses:
  - `alex.eng@yourlabdomain.com`: member of **Dept-Engineering**, enrolled in Okta Verify
  - `sam.eng@yourlabdomain.com`: **not** assigned to the app (used for the negative test)
- The user profile attribute `department` set (Alex: `Engineering`)

---

## Step 1: Configure the authorization server

**Okta Admin → Security → API → Authorization Servers → `default`**

Note the **Issuer URI** and the **Audience**:

| Setting | Value |
|---|---|
| Issuer | `https://integrator-1703499.okta.com/oauth2/default` |
| Audience | `api://default` |

![Authorization server settings](screenshots/01-auth-server-settings.png)

### 1a. Add a custom scope

**Scopes tab → Add Scope**

| Field | Value |
|---|---|
| Name | `reports:read` |
| Description | Read reports |
| User consent | Implicit |

![Add reports:read scope](screenshots/02-add-scope.png)

### 1b. Add a custom claim

**Claims tab → Add Claim**

| Field | Value |
|---|---|
| Name | `department` |
| Include in token type | Access Token, Always |
| Value type | Expression |
| Value | `user.department` |
| Include in | Any scope |

![Add department claim](screenshots/03-add-claim.png)

### 1c. Check the access policy

**Access Policies tab:** the **Default Policy** is assigned to **All Clients**, with a **Default Rule** that allows the **Authorization Code** grant and **any scopes**.

![Access policies](screenshots/04-access-policy.png)
![Default rule](screenshots/05-default-rule.png)

---

## Step 2: Create the OIDC app in Okta

**Applications → Create App Integration → OIDC - OpenID Connect → Web Application → Next**

A *Web Application* is a server-side app that can keep a client secret safe, which is what Flask is.

![Create app integration](screenshots/06-create-app-integration.png)

### General settings

| Field | Value |
|---|---|
| App integration name | `My Web App` |
| Client definition | Use Okta-generated client ID |
| Grant type | Authorization Code |
| Sign-in redirect URI | `http://localhost:8080/authorization-code/callback` |
| Sign-out redirect URI | `http://localhost:8080/` |

![General settings](screenshots/07-general-settings.png)
![Redirect URIs](screenshots/08-redirect-uris.png)

> Port **8080** is used instead of 5000 because macOS AirPlay Receiver takes port 5000.

### Assignments

**Limit access to selected groups → `Dept-Engineering`**

Only members of this group can sign in to the app. Step 8 tests this.

![Assignments](screenshots/09-assignments.png)

### Client credentials

After saving, copy the **Client ID** and **Client Secret** from the **General** tab. The secret is a password. Never commit it.

![Client credentials](screenshots/10-client-credentials.png)

> Optional hardening: check **Require PKCE as additional verification**.

---

## Step 3: Run the app

Install the dependencies:

```bash
cd ~/Downloads
pip3 install flask requests "pyjwt[crypto]"
```

![pip install](screenshots/11-pip-install.png)

Set the configuration as environment variables (never hard-code the secret) and start the app:

```bash
export OKTA_ISSUER="https://integrator-1703499.okta.com/oauth2/default"
export OKTA_CLIENT_ID="<your-client-id>"
export OKTA_CLIENT_SECRET="<your-client-secret>"
python3 oidc_lab.py
```

![Environment variables](screenshots/12-env-vars.png)

The app prints its configuration and starts on port 8080:

```
=== Okta OIDC + API Access Management lab ===
Issuer:        https://integrator-1703499.okta.com/oauth2/default
Client ID:     0oa18ok7etkF5TL4N698
Redirect URI:  http://localhost:8080/authorization-code/callback
API audience:  api://default   required scope: reports:read
Open:          http://localhost:8080
```

![App started](screenshots/13-app-started.png)

> The `NotOpenSSLWarning` from urllib3 comes from macOS's built-in Python using LibreSSL. It is harmless for this lab.

---

## Step 4: Sign in with MFA

1. Open a private browser window and go to `http://localhost:8080`.
2. Click **Sign in with Okta (with reports:read)**.

![App home page](screenshots/14-app-home.png)

3. The app redirects to Okta. Sign in as `alex.eng@yourlabdomain.com` and enter the **Okta Verify** code.

![Okta MFA prompt](screenshots/15-okta-mfa.png)

4. Okta redirects back to the app's callback with a one-time code. The app exchanges the code for tokens and shows the signed-in page.

---

## Step 5: Inspect the tokens

Click **Tokens**. The page decodes both JWTs and explains every claim. Both are signed by Okta with RS256.

### ID token: who the user is (OIDC)

![ID token claims](screenshots/16-id-token.png)

| Claim | Value | Why it matters |
|---|---|---|
| `aud` | `0oa18ok7etkF5TL4N698` | **The Client ID**: this token is meant for the app |
| `iss` | `https://integrator-1703499.okta.com/oauth2/default` | Issued by the custom authorization server |
| `name` / `email` | Alex Eng / alex.eng@yourlabdomain.com | Identity of the user |
| `amr` | `["mfa", "otp", "pwd", "okta_verify"]` | Proves MFA with Okta Verify was used |
| `nonce` | random | Ties the token to this login request (replay protection) |
| `exp` | 1 hour after `iat` | Expiry |

### Access token: what the app may do (OAuth 2.0)

![Access token claims](screenshots/17-access-token.png)

| Claim | Value | Why it matters |
|---|---|---|
| `aud` | `api://default` | **The API**: this token is meant for the resource server |
| `scp` | `["openid", "profile", "reports:read", "email"]` | Includes the custom scope |
| `department` | `Engineering` | Custom claim added in Step 1b |
| `cid` | `0oa18ok7etkF5TL4N698` | The app that requested the token |
| `exp` | 1 hour after `iat` | Matches the 1-hour access token lifetime in the rule |

**Key takeaway:** the two tokens have **different audiences**. That is why an API must reject an ID token, which Step 6 proves.

---

## Step 6: Test the protected API

Click **API tests**. The app calls its own `GET /api/reports` endpoint four ways. The API validates **signature** (using Okta's public keys from the JWKS endpoint), **issuer**, **audience**, **expiry**, and **scope**.

| # | Test | Result | API response |
|---|---|---|---|
| 1 | No token | **401** | `Missing bearer token` |
| 2 | ID token instead of access token | **401** | `Wrong audience (expected api://default)` |
| 3 | Tampered access token | **401** | `Signature verification failed` |
| 4 | Real access token | **200** | `Access granted to protected reports API` |

![API tests 401s](screenshots/18-api-tests-401.png)
![API test 200](screenshots/19-api-test-200.png)

The successful response returns the caller, the `department` claim, the granted scopes, and the report data:

```json
{
  "message": "Access granted to protected reports API",
  "caller": "alex.eng@yourlabdomain.com",
  "department": "Engineering",
  "scopes": ["openid", "profile", "reports:read", "email"],
  "reports": [
    { "id": 1, "name": "Q3 Access Review" },
    { "id": 2, "name": "Dormant Accounts" }
  ]
}
```

### 401 vs. 403

| Code | Meaning |
|---|---|
| **401 Unauthorized** | "I don't know who you are, or your token is bad." (missing, wrong audience, forged, expired) |
| **403 Forbidden** | "I know who you are, but you're not allowed to do this." (valid token, missing scope) |

---

## Step 7: Missing scope returns 403

1. Click **Sign out**.
2. Click **Sign in without reports:read** and sign in again as Alex.

![Sign in without scope](screenshots/20-sign-in-without-scope.png)

3. Run the API tests again. Tests 1–3 still return 401, but **test 4 now returns 403 `insufficient_scope`**. The token is genuine and valid; it just doesn't carry the permission.

![API test 403](screenshots/21-api-test-403.png)

---

## Step 8: Least privilege with group assignment

The app is assigned only to **Dept-Engineering** (Step 2). Signing in as `sam.eng@yourlabdomain.com`, who is not assigned to the app:

![Sign in as Sam](screenshots/22-sign-in-sam.png)

Okta refuses at the authorization server, with **`access_denied`: User is not assigned to the client application**. No code and no tokens are ever issued, so the app and the API never see the request.

![User not assigned](screenshots/23-user-not-assigned.png)

**Why this matters:** access is decided **centrally in Okta**, not inside each app. Removing a user from the group removes their access everywhere the group is used.

> **Going further:** the same idea works at the scope level. An authorization server access policy assigned to this app, with a rule like *"if user is in Dept-Engineering, grant `reports:read`"*, lets other users sign in but never receive the scope, so the API returns 403 instead of the login being blocked.

---

## Terminal log of the full flow

The app logs each stage: `LOGIN` (redirect to Okta), `CALLBACK` (code received), `TOKEN` (tokens received), and every `API` decision.

```
17:43:53  CALLBACK received authorization code; exchanging it at /token
17:43:55  TOKEN    received ID token + access token for alex.eng@yourlabdomain.com
                   scope='openid profile reports:read email'
17:49:29  API      401 invalid_token: Missing bearer token
17:49:30  API      401 invalid_token: Wrong audience (expected api://default)
17:49:30  API      401 invalid_token: Invalid token: Signature verification failed
17:49:30  API      200 OK for alex.eng@yourlabdomain.com  scopes=['openid','profile','reports:read','email']
17:51:18  LOGIN    redirecting to Okta /authorize  scope='openid profile email'
17:51:51  TOKEN    received ID token + access token for alex.eng@yourlabdomain.com
                   scope='openid profile email'
17:51:54  API      401 invalid_token: Missing bearer token
17:51:54  API      401 invalid_token: Wrong audience (expected api://default)
17:51:54  API      401 invalid_token: Invalid token: Signature verification failed
17:51:54  API      403 insufficient_scope: Token lacks required scope reports:read
```

![Terminal log part 1](screenshots/24-terminal-log-1.png)
![Terminal log part 2](screenshots/25-terminal-log-2.png)

---

## Results summary

| Scenario | Expected | Actual |
|---|---|---|
| Sign in with MFA | ID + access token issued | ✅ `amr` includes `mfa`, `okta_verify` |
| Custom scope in access token | `reports:read` in `scp` | ✅ |
| Custom claim in access token | `department: Engineering` | ✅ |
| API: no token | 401 | ✅ 401 |
| API: ID token | 401 (wrong audience) | ✅ 401 |
| API: tampered token | 401 (bad signature) | ✅ 401 |
| API: valid token with scope | 200 | ✅ 200 |
| API: valid token without scope | 403 | ✅ 403 |
| User not assigned to app | Login blocked | ✅ 400 `access_denied` |

---

## Interview questions this lab answers

**What's the difference between OAuth 2.0 and OIDC?**
OAuth 2.0 is for authorization: it gives an app an access token to call an API. OIDC is built on top of OAuth and adds authentication: an ID token that says who the user is.

**What's the difference between an ID token and an access token?**
The ID token is for the app, to know who signed in. The access token is for the API, to decide what's allowed. You never send the ID token to an API. In this lab the API rejected it because its audience was the Client ID, not `api://default`.

**How does an API validate a token?**
It verifies the signature with Okta's public keys (JWKS), then checks the issuer, audience, expiry, and the required scope.

**What's the difference between 401 and 403?**
401 means the token is missing or invalid. 403 means the token is valid but lacks the permission. Test 4 showed both: 200 with `reports:read`, 403 without it.

**Why use the Authorization Code flow and PKCE?**
Tokens never pass through the browser URL. Only a one-time code does, and it's exchanged server-side with the client secret. PKCE adds a code verifier only the app knows, so an intercepted code can't be redeemed.

**Where should access decisions live?**
Centrally in the identity provider. Group-based app assignment blocked an unassigned user before any token existed.

---

## Security notes

- The client secret is passed through an environment variable and is **never committed**. Add a `.gitignore` for any `.env` file.
- Screenshots have the client secret redacted. If a secret is ever exposed, rotate it in Okta (**Generate new secret**, then deactivate the old one).
- The Flask development server is for local testing only, not production.
- Possible next steps: enable **Require PKCE**, add **refresh tokens**, and move scope decisions into an authorization server **access policy rule**.

---

