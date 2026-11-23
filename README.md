# Okta IAM Lab – Foothill Family Clinic

A hands-on identity and access management lab built in an Okta Integrator Free Plan org. I set up a fictional healthcare clinic and configured group-based access, MFA policies, OIDC single sign-on, and user lifecycle (onboarding/offboarding).

I work in healthcare IT support, where I handle account provisioning and access in Active Directory and Entra ID. I built this lab to get hands-on with Okta and to practice designing access policies, not just operating them.

> **Note:** This is a lab environment. All users are fictional, and some policies were scoped differently than I would use in production (noted below).

---

## Environment

| Item | Details |
|---|---|
| Platform | Okta Workforce Identity (Integrator Free Plan) |
| Scenario | Foothill Family Clinic (fictional) |
| Users | 6 test users + 1 admin |
| Groups | Clinical Staff, Front Office, IT Department |
| Test app | Clinic Patient Portal (Test), an OIDC web app tested with [OIDC Debugger](https://oidcdebugger.com) |

---

## 1. Users and groups

Created three department groups and assigned users by role.

| User | Group |
|---|---|
| Maria Lopez | Clinical Staff |
| James Carter | Clinical Staff |
| Sofia Nguyen | Front Office |
| Derek Wilson | Front Office |
| Priya Shah | IT Department |

Access is granted through **group membership**, not individual assignments, so access follows a person's role.

![Groups](screenshots/01-groups.png)

---

## 2. MFA and sign-on policies

Different users carry different risk, so they get different requirements.

| Group | Policy | Why |
|---|---|---|
| IT Department | MFA (Okta Verify) **every sign-in**, 2-hour max session, 30-minute idle timeout | Admin accounts are high-value targets |
| Clinical Staff / Front Office | Password, re-authenticate after 1 hour | Shared clinic workstations; balances security and usability |

**Result:** Priya (IT) was forced to enroll in Okta Verify at first sign-in. Maria (Clinical) signed in with a password only.


### Troubleshooting: policy layering
At first, Maria was also forced into MFA. Okta evaluates **two layers** of policy:
1. **Global Session Policy**: signing in to Okta itself
2. **Authentication (app sign-in) policies**: per app, including the Okta Dashboard

The Okta Dashboard's default app policy required MFA for everyone, which overrode my group-based intent. I added a group-scoped rule above the catch-all rule (rules evaluate top to bottom, first match wins).

Okta flagged this rule as **weak assurance** for the Dashboard. **In production, I would require MFA for all users**; I scoped it here to demonstrate group-based policy differences.

---

## 3. SSO with OIDC

Created an OIDC web app, **Clinic Patient Portal (Test)**, assigned only to **Clinical Staff**.

- Scopes: `openid profile email` (only what the app needs to identify the user)
- Tested the authorization code flow (with PKCE) using OIDC Debugger

| Test | Result |
|---|---|
| Maria (Clinical Staff) | ✅ Success, authorization code returned and state matched |
| Admin account (not in Clinical Staff) | ❌ `access_denied – User is not assigned to the client application` |

Being an Okta admin does **not** grant app access. Access comes only from assignment (least privilege).

![SSO success](screenshots/03-sso-success.png)

### Step-up authentication
The Patient Portal kept its own app policy requiring MFA, even though the Dashboard allows password-only for staff. Since a patient portal would handle PHI, it requires stronger authentication. Maria enrolled a **passkey** (phishing-resistant, Face ID).

### Troubleshooting: enrollment email never arrived
MFA enrollment was blocked by an email verification challenge (Okta Account Management Policy). I updated the test user's email using Gmail plus-addressing, but no email arrived.

I exported the **System Log** and confirmed:
- `Okta Account Management Policy evaluation resulted in CHALLENGE` for the user
- **No `system.email.delivery` events** after the change, so Okta never sent the email (a free-tier limitation, not a mail filter issue)

Fix: added a scoped, lab-only enrollment rule for Clinical Staff above the catch-all. In production I would keep email verification for authenticator enrollment.

---

## 4. Onboarding and offboarding

**Onboarding:** created new hire **Jordan Reyes** and added him to Clinical Staff. He received the Patient Portal **automatically** through group membership, with a temporary password he must change at first sign-in.

![Onboarding](screenshots/05-onboarding.png)

**Offboarding:** deactivated **Derek Wilson**. All app access was removed, and the deactivation was recorded in the System Log for audit.

In a real environment I would typically **suspend** first (reversible) and **deactivate** after HR confirms the termination.

![Offboarding](screenshots/06-offboarding.png)

---

## System Log evidence

Key events from the org's System Log export (times UTC, Nov 23, 2026):

| Time | Event | What it shows |
|---|---|---|
| 01:32:33 | `user.mfa.factor.activate`: FIDO_WEBAUTHN, Maria Lopez | Passkey enrolled (phishing-resistant MFA) |
| 01:32:40 | `app.oauth2.authorize.code`: SUCCESS, Maria Lopez | SSO to Clinic Patient Portal worked |
| 01:32:42 | `app.oauth2.token.grant`: FAILURE `invalid_token_endpoint_auth_method` | Expected. Exchanging the code for tokens requires the app's client secret from a backend, which OIDC Debugger doesn't send |
| 01:36:14 | `user.lifecycle.create` / `activate`: Jordan Reyes | New hire created |
| 01:37:37 | `group.user_membership.add`: Jordan to Clinical Staff | Added to role group |
| 01:37:38 | `application.user_membership.add`: Jordan to Patient Portal | App access granted **automatically**, one second after the group add |
| 01:38:37 | `user.lifecycle.deactivate`: Derek Wilson | Offboarding, with an audit trail |


---

## What I learned
- How Okta's policy layers (global session, app sign-in, account management) interact, and how rule order affects outcomes
- Using the System Log to trace *why* something happened instead of guessing
- Designing access by role and risk: stricter controls for admins and sensitive apps (step-up authentication)
- OIDC basics: scopes, authorization code flow, PKCE, and state validation
- Group-based provisioning and clean offboarding

## Next steps
- Automate user onboarding with PowerShell or Python against the Okta API
- Add a SAML app integration
