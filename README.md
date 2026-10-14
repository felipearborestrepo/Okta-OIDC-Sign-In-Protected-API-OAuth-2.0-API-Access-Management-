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

## Step 1: Configure the authorization server

**Goal:** prepare the token "factory." Before any app can request a `reports:read` permission, the authorization server has to know that permission exists and what to put in the tokens it issues.

**Okta Admin → Security → API → Authorization Servers → `default`**

Note the **Issuer URI** and the **Audience**:

| Setting | Value |
|---|---|
| Issuer | `https://integrator-1703499.okta.com/oauth2/default` |
| Audience | `api://default` |

<img width="1135" height="643" alt="Screenshot 2026-10-13 at 17 11 27" src="https://github.com/user-attachments/assets/e763a9dc-a9f2-4d93-8d58-e35640db9035" />

> **Why a custom authorization server and not the org one?** The org authorization server (`https://<org>.okta.com`) issues tokens meant only for Okta's own APIs. You can't add your own scopes or claims to it. The `default` custom authorization server is the one meant for protecting **your** APIs.
### 1a. Add a custom scope

**Scopes tab → Add Scope**

| Field | Value |
|---|---|
| Name | `reports:read` |
| Description | Read reports |
| User consent | Implicit |

<img width="705" height="658" alt="Screenshot 2026-10-13 at 17 12 53" src="https://github.com/user-attachments/assets/ee8bee67-d1ed-4b2f-a7e0-adc9031a365d" />

**Why:** a scope is a named permission. The app asks for it at login, Okta decides whether to grant it, and the granted scopes are listed in the access token's `scp` claim. The API then checks for `reports:read` before returning data. The `resource:action` naming style keeps permissions clear as an API grows (for example `reports:write` later).

### 1b. Add a custom claim

**Claims tab → Add Claim**

| Field | Value |
|---|---|
| Name | `department` |
| Include in token type | Access Token, Always |
| Value type | Expression |
| Value | `user.department` |
| Include in | Any scope |

<img width="696" height="552" alt="Screenshot 2026-10-13 at 17 14 40" src="https://github.com/user-attachments/assets/d31a2675-6aa0-42b9-825d-2df3fccb7710" />

**Why:** claims let the API make decisions without a separate lookup. By copying the user's `department` profile attribute into the access token, the API can know "this caller is in Engineering" just by reading the token.

- `user.department` is **Okta Expression Language**: it reads the `department` attribute from the user's Okta profile at the moment the token is issued.
- **Access Token, Always** means the claim goes to the API, not just the app.
- **Any scope** means it's included no matter which scopes were requested.

### 1c. Check the access policy

**Access Policies tab:** the **Default Policy** is assigned to **All Clients**, with a **Default Rule** that allows the **Authorization Code** grant and **any scopes**.

<img width="1003" height="600" alt="Screenshot 2026-10-13 at 17 17 10" src="https://github.com/user-attachments/assets/e464c8c9-a15c-4ea9-adf0-a68b71bcab7f" />

<img width="698" height="657" alt="Screenshot 2026-10-13 at 17 17 32" src="https://github.com/user-attachments/assets/b8a73a23-83ee-48a7-a026-7edfd5ece008" />

An authorization server will **not issue any token** unless an access policy allows it. A policy decides *which clients* it applies to, and its rules decide *which grant types, users, and scopes* are allowed, plus token lifetimes.

---

## Step 2: Create the OIDC app in Okta

**Goal:** register the Flask app with Okta as a **client**, so Okta knows who is asking for tokens, where it's allowed to send users back, and who may use it.

**Applications → Create App Integration → OIDC - OpenID Connect → Web Application → Next**

A *Web Application* is a server-side app that can keep a client secret safe, which is what Flask is.

<img width="794" height="591" alt="Screenshot 2026-10-13 at 17 18 58" src="https://github.com/user-attachments/assets/7f5c9a42-1b4f-4477-8874-03a15164a91c" />

- **OIDC** rather than SAML: this lab needs tokens an API can check (JWT access tokens). SAML handles sign-in for web apps but doesn't give you OAuth access tokens for APIs.
- **Web Application** rather than Single-Page App or Native: the Flask app runs on a server, so it can safely keep a **client secret**. Browser-only (SPA) and mobile apps can't hide a secret, so they rely on PKCE alone.
### General settings

| Field | Value |
|---|---|
| App integration name | `My Web App` |
| Client definition | Use Okta-generated client ID |
| Grant type | Authorization Code |
| Sign-in redirect URI | `http://localhost:8080/authorization-code/callback` |
| Sign-out redirect URI | `http://localhost:8080/` |

<img width="936" height="603" alt="Screenshot 2026-10-13 at 17 21 34" src="https://github.com/user-attachments/assets/25518ca9-bad2-474d-b974-c1c84c51078f" />

<img width="909" height="604" alt="Screenshot 2026-10-13 at 17 22 14" src="https://github.com/user-attachments/assets/f9d6f193-05ab-49e2-a467-295122807443" />

**Why this matters:** Okta will only send the authorization code to a URI on this list. That stops an attacker from crafting a login link that delivers the code to *their* site. The URI must match **exactly**, including port and path, or Okta returns a `redirect_uri` error.

> Port **8080** is used instead of 5000 because macOS AirPlay Receiver takes port 5000.

### Assignments

**Limit access to selected groups → `Dept-Engineering`**

Only members of this group can sign in to the app. Step 8 tests this.

<img width="888" height="370" alt="Screenshot 2026-10-13 at 17 22 29" src="https://github.com/user-attachments/assets/47dc3192-d7b5-4d6c-b207-3de589f22d5e" />

**Why:** this is the first layer of access control. Only members of Dept-Engineering can sign in to the app at all. Anyone else is stopped by Okta before a token exists (tested in Step 8). Assigning to groups instead of individual users means access follows group membership automatically when people join or leave.

### Client credentials

After saving, copy the **Client ID** and **Client Secret** from the **General** tab. The secret is a password. Never commit it.

<img width="724" height="601" alt="Screenshot 2026-10-13 at 17 34 51" src="https://github.com/user-attachments/assets/e511aeb5-bb90-4b3a-a974-73fcd8cc6cf0" />

> Optional hardening: check **Require PKCE as additional verification**.

---

## Step 3: Run the app


**Goal:** start the Flask app locally with the Okta settings

Install the dependencies:

```bash
cd ~/Downloads
pip3 install flask requests "pyjwt[crypto]"
```

<img width="563" height="327" alt="Screenshot 2026-10-13 at 17 26 00" src="https://github.com/user-attachments/assets/2a274593-2386-41a0-a6f3-64f1d62abcae" />

Set the configuration as environment variables (never hard-code the secret) and start the app:

```bash
export OKTA_ISSUER="https://integrator-1703499.okta.com/oauth2/default"
export OKTA_CLIENT_ID="<your-client-id>"
export OKTA_CLIENT_SECRET="<your-client-secret>"
python3 oidc_lab.py
```

<img width="576" height="165" alt="Screenshot 2026-10-13 at 17 38 17" src="https://github.com/user-attachments/assets/86ad5cad-2bb9-4df9-8a90-41b94b48c778" />

**Why environment variables:** the secret stays out of the source code, so the code can be pushed to GitHub safely. The same code can run against a different Okta org just by changing the variables.

The app prints its configuration and starts on port 8080:

```
=== Okta OIDC + API Access Management lab ===
Issuer:        https://integrator-1703499.okta.com/oauth2/default
Client ID:     0oa18ok7etkF5TL4N698
Redirect URI:  http://localhost:8080/authorization-code/callback
API audience:  api://default   required scope: reports:read
Open:          http://localhost:8080
```

<img width="562" height="280" alt="Screenshot 2026-10-13 at 17 36 44" src="https://github.com/user-attachments/assets/04df4b9a-bbc9-4920-b969-9db58b08642d" />

**What to check:** the issuer, client ID, and redirect URI must match exactly what's in Okta. The last line confirms what the API will demand: audience `api://default` and scope `reports:read`. Leave this terminal open; it shows a live log of every step.

---

## Step 4: Sign in with MFA

**Goal:** run the full Authorization Code flow as a real user and receive tokens.

1. Open a private browser window and go to `http://localhost:8080`.
2. Click **Sign in with Okta (with reports:read)**.

<img width="1060" height="562" alt="Screenshot 2026-10-13 at 17 40 34" src="https://github.com/user-attachments/assets/b5eb69f7-6a1f-4049-99db-bb9d1b813605" />

3. The app redirects to Okta. Sign in as `alex.eng@yourlabdomain.com` and enter the **Okta Verify** code.

<img width="626" height="535" alt="Screenshot 2026-10-13 at 17 43 16" src="https://github.com/user-attachments/assets/c26234e9-ee65-44f1-91ac-5d780135ff67" />

**Why this matters:** the user types the password and MFA code **only into Okta**, never into the app. The app never handles credentials, and the sign-on policy (including MFA) is enforced by Okta in one place for every app.

4. Okta redirects back to the app's callback with a one-time code. The app exchanges the code for tokens and shows the signed-in page.

---

## Step 5: Inspect the tokens

**Goal:** see exactly what Okta put in each token and understand why they're different.

Click **Tokens**. The page decodes both JWTs and explains every claim. Both are signed by Okta with RS256.

### ID token: who the user is (OIDC)

<img width="1191" height="664" alt="Screenshot 2026-10-13 at 17 44 56" src="https://github.com/user-attachments/assets/5f546c4d-f0f3-44ac-9285-f1fd9ddcc835" />

| Claim | Value | Why it matters |
|---|---|---|
| `aud` | `0oa18ok7etkF5TL4N698` | **The Client ID**: this token is meant for the app |
| `iss` | `https://integrator-1703499.okta.com/oauth2/default` | Issued by the custom authorization server |
| `name` / `email` | Alex Eng / alex.eng@yourlabdomain.com | Identity of the user |
| `amr` | `["mfa", "otp", "pwd", "okta_verify"]` | Proves MFA with Okta Verify was used |
| `nonce` | random | Ties the token to this login request (replay protection) |
| `exp` | 1 hour after `iat` | Expiry |

### Access token: what the app may do (OAuth 2.0)

<img width="923" height="528" alt="Screenshot 2026-10-13 at 17 49 19" src="https://github.com/user-attachments/assets/0cf32529-cb83-471d-a442-2ba7404201f6" />

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

**Goal:** prove the API only accepts genuine, correctly targeted access tokens.

Click **API tests**. The app calls its own `GET /api/reports` endpoint four ways. The API validates **signature** (using Okta's public keys from the JWKS endpoint), **issuer**, **audience**, **expiry**, and **scope**.

| # | Test | Result | API response |
|---|---|---|---|
| 1 | No token | **401** | `Missing bearer token` |
| 2 | ID token instead of access token | **401** | `Wrong audience (expected api://default)` |
| 3 | Tampered access token | **401** | `Signature verification failed` |
| 4 | Real access token | **200** | `Access granted to protected reports API` |

<img width="1029" height="538" alt="Screenshot 2026-10-13 at 17 49 44" src="https://github.com/user-attachments/assets/c93d9164-e49c-4459-b088-0210f5fc8123" />

<img width="1008" height="410" alt="Screenshot 2026-10-13 at 17 49 55" src="https://github.com/user-attachments/assets/17e47cce-e34b-4115-be1f-fd26d03a91bf" />

**Why test 2 matters:** the ID token is genuine, signed by Okta, and not expired. Signature checks alone would accept it. Only the **audience check** catches it. Skipping the audience check is a common real-world bug that lets a token meant for one app unlock another.

**Why test 3 matters:** anyone can decode a JWT and edit its contents (for example adding `reports:write` to `scp`). The signature is what makes that useless: the API detects the change and rejects the token.

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

**Goal:** show that a valid, authenticated user still can't use the API without the right permission.

1. Click **Sign out**.
2. Click **Sign in without reports:read** and sign in again as Alex.

<img width="1008" height="544" alt="Screenshot 2026-10-13 at 17 50 58" src="https://github.com/user-attachments/assets/e67c82d5-6979-4780-bd15-c467aa528d96" />

3. Run the API tests again. Tests 1–3 still return 401, but **test 4 now returns 403 `insufficient_scope`**. The token is genuine and valid; it just doesn't carry the permission.

<img width="1063" height="649" alt="Screenshot 2026-10-13 at 17 52 05" src="https://github.com/user-attachments/assets/86ae4a81-3601-4de6-ad0a-12027b64f674" />

**What happened:** it's the same user, same MFA, and a perfectly valid token. Its signature, issuer, audience, and expiry all pass. The only difference is that `scp` no longer contains `reports:read`, because the app didn't ask for it and Okta didn't grant it. The API therefore knows *who* is calling but refuses the action.

**Why this matters:** this is **least privilege at the API level**. Apps should request only the scopes they need, and APIs should check for a specific scope on every endpoint rather than accepting any valid token.

---

## Step 8: Least privilege with group assignment

**Goal:** show that Okta can block a user centrally, before the app or the API is ever involved.

The app is assigned only to **Dept-Engineering** (Step 2). Signing in as `sam.eng@yourlabdomain.com`, who is not assigned to the app:

<img width="865" height="635" alt="Screenshot 2026-10-13 at 17 54 33" src="https://github.com/user-attachments/assets/576e81bc-be00-42d2-b961-d2c2f720d12c" />

Okta refuses at the authorization server, with **`access_denied`: User is not assigned to the client application**. No code and no tokens are ever issued, so the app and the API never see the request.

<img width="578" height="572" alt="Screenshot 2026-10-13 at 17 55 17" src="https://github.com/user-attachments/assets/30ef66ab-72c0-4e3a-ab01-6d5af8877c3d" />

**What happened:** after Sam authenticated, Okta checked whether he is assigned to *My Web App*. He isn't, so Okta stopped at the authorization step and **never issued a code or any tokens**. The app never sees a request, and the API is never called.

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

<img width="563" height="298" alt="Screenshot 2026-10-13 at 17 56 28" src="https://github.com/user-attachments/assets/466c77fb-f794-49bd-a65a-6edc9332e99b" />

<img width="563" height="343" alt="Screenshot 2026-10-13 at 17 56 05" src="https://github.com/user-attachments/assets/df5a0172-8118-4865-92f8-69d25f9b1323" />


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

---

