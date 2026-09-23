# SECURITY.md: Halfsies

Version: 1.0.0-draft
Status: Draft for review
Scope: The security posture of the Halfsies iOS app, Android app, Halfsies API, static web content, and the build and deploy pipeline: security requirements, threat model, controls, trust-boundary enforcement, and secure coding rules.
Companion docs: `REQUIREMENTS.md` (product behavior, API contract, privacy requirements), `ARCHITECTURE.md` (components, trust boundaries, data flows, interfaces), `DESIGN.md` (UI rules), `DEPENDENCIES.md` (not yet written; SEC-SC-01)

## How to read this document

- **Ownership.** This file owns every security requirement and control. `REQUIREMENTS.md` owns product behavior and `ARCHITECTURE.md` owns structure; this file cites them by ID and does not restate them.
- **Conflicts.** A conflict between this file and another spec file is recorded in section 11 and marked `TO BE DECIDED`. Until it is resolved, the more secure reading applies.
- **Markers.**
  - `UNKNOWN`: a fact the input documents do not provide.
  - `TO BE DECIDED`: a decision that has not been made.
  - `ASSUMPTION`: an inference the input documents do not support directly. Every assumption must be confirmed or removed before M1 exits.
  - **Proposed**: a security decision (section 8) that its owner has not confirmed.
- **Keywords** follow RFC 2119.
- **IDs are stable.**
  - `SEC-*`: security requirements
  - `SC-*`: controls added by this document
  - `T-*`: threats
  - `TBE-*`: trust-boundary enforcement
  - `JS-*`, `RN-*`, `NODE-*`: secure coding rules
  - `SD-*`: security decisions
  - `SQ-*`: open security questions
- **Status.** No implementation exists yet. Every control in this document is a design obligation, not a claim about deployed behavior.

---

## 1. Reporting a vulnerability

| Item | Value |
|---|---|
| Security contact | TO BE DECIDED (SQ-01) |
| Private reporting channel | TO BE DECIDED. GitHub private vulnerability reporting is one option; it is not enabled today (UNKNOWN) |
| Response targets (acknowledge, triage, fix) | TO BE DECIDED |
| Safe harbor and disclosure policy | TO BE DECIDED |
| Supported versions | The app versions API-02 supports, and the current API. Nothing has been released yet |

Until a channel exists, do not file security issues in the public issue tracker.

---

## 2. Security objectives and baselines

### 2.1 Objectives

In priority order. When objectives conflict, the higher one wins.

1. **Protect each participant's precise location from everyone else**, including the other participant (PRIV-01 to PRIV-06, API-SHP-01).
2. **Only participants can see or change a session** (SEC-AZ-01 to SEC-AZ-04).
3. **Credentials cannot be stolen or replayed** (SEC-AUTH-01 to SEC-AUTH-08, SEC-INV-01 to SEC-INV-06).
4. **Provider keys and spend are protected** (SEC-SEC-01 to SEC-SEC-04, NFR-COST-02).
5. **Releases can be trusted** (SEC-SC-01 to SEC-SC-04).
6. **Availability** at the NFR-AV-01 SLO, including under abuse.

### 2.2 Baseline standards

Binding:

- OWASP MASVS v2 (mobile apps)
- OWASP ASVS 5.0 Level 2 (API)
- OWASP API Security Top 10 2023
- RFC 9700 (OAuth 2.0 Security BCP)
- RFC 8252 (OAuth for native apps)
- RFC 9449 (DPoP)
- NIST SP 800-63B-4 (authentication)
- NIST SP 800-207 (Zero Trust principles for service-to-service calls)

Supporting guidance used to derive the controls and rules below. It is not binding beyond what the controls state:

- OWASP Top 10 Proactive Controls 2024
- OWASP Cheat Sheets: REST Security, Authentication, Session Management, Input Validation, Secrets Management, Logging, Error Handling, CI/CD Security, Software Supply Chain Security, Infrastructure as Code Security, Vulnerable Dependency Management, and XSS Prevention (static web only)
- NIST SSDF 1.1 (SP 800-218)

---

## 3. System model for threat modeling

Locations below name components defined in `ARCHITECTURE.md`.

### 3.1 Data classification

| Class | Examples | Handling |
|---|---|---|
| Restricted | Precise origins, addresses, autocomplete query text | Encrypted at the application level (origins). Never logged (PRIV-05). Purged on session end |
| Secret | Refresh tokens, invite secrets, provider keys, signing keys | Stored only as hashes (tokens, invite secrets) or in Secrets Manager (keys). Redacted from logs |
| Confidential | Display name, email, IdP subject, snapped area, Plan | Encryption at rest. Access controlled by authorization. Included in export |
| Internal | Metrics, trace timing, correlation IDs | No personal data in labels |

### 3.2 Assets

| ID | Asset | Class | Where it lives |
|---|---|---|---|
| AS-01 | Precise starting point (coordinates) | Restricted | App memory during entry. `participants.origin_ciphertext` (KMS envelope, `origin-cmk`). Provider requests |
| AS-02 | Address text and autocomplete queries | Restricted | App input, the `GET /v1/places/autocomplete` query string, provider requests |
| AS-03 | Snapped area and area label | Confidential | `participants.approx_*`, `area_label`, API responses to the other participant |
| AS-04 | Access tokens, refresh tokens, DPoP private key | Secret | App Keychain or Keystore, Secure Enclave or StrongBox. Only hashes are stored server-side |
| AS-05 | Invite secret | Secret | URL fragment, the OS share sheet, `invites.secret_hash` |
| AS-06 | Provider, IdP, APNs, FCM credentials, JWT signing key, `k_search` | Secret | Secrets Manager |
| AS-07 | Profile (display name, email, IdP subject), Plan | Confidential | Aurora |
| AS-08 | Travel times `tA` and `tB` | Confidential. They are derived from AS-01 and could be used to trilaterate it (PRIV-06) | API responses, `results` |
| AS-09 | Push tokens | Confidential | `devices` |
| AS-10 | Security audit logs | Confidential | CloudWatch `halfsies-security` |
| AS-11 | Source, build pipeline, signing keys | Secret (keys), Internal (source) | GitHub, GitHub Actions, App Store Connect, Play App Signing, Secrets Manager |
| AS-12 | Provider budget and spend | Internal | Redis counters, provider billing |

### 3.3 Actors

| ID | Actor | Trust |
|---|---|---|
| AC-01 | Participant A (Initiator, account) | Trusted only for their own data |
| AC-02 | Participant B (Invitee, account or guest) | Trusted only for their own data |
| AC-03 | **The other participant, acting maliciously**: for example an abuser or stalker who wants the counterpart's location | Adversary for AS-01 and AS-02 of the counterpart. This is the primary privacy threat |
| AC-04 | Anyone holding a forwarded or leaked invite link | Untrusted |
| AC-05 | Anonymous internet client, scripted or a modified app | Untrusted |
| AC-06 | Malicious app on the same device | Untrusted |
| AC-07 | Person with physical access to an unlocked or stolen device | Untrusted |
| AC-08 | Compromised or malfunctioning third-party provider (Google Maps Platform, IdP, APNs, FCM) | Untrusted responses |
| AC-09 | Compromised dependency or CI component | Untrusted |
| AC-10 | Insider with cloud or repository access | Partially trusted; least privilege applies. The access model is UNKNOWN (SQ-03) |

### 3.4 Entry points

| ID | Entry point | Reached by |
|---|---|---|
| EP-01 | `/v1/*` API (the endpoints in section 4.2 of `REQUIREMENTS.md`) through CloudFront, WAF, and ALB | AC-01 to AC-05 |
| EP-02 | Static web: `/.well-known/*`, legal pages, invite fallback `/i` (INT-06) | Anyone |
| EP-03 | Deep links: Universal Links and App Links for `/i` and the OIDC redirect | AC-04 to AC-06 |
| EP-04 | Push notification delivery to the app | APNs and FCM |
| EP-05 | Provider responses to the API | AC-08 |
| EP-06 | IdP token and JWKS responses to the API | AC-08 |
| EP-07 | App UI input: address search, pin drop, display name, filters, meeting time (`DESIGN.md` Components) | The device user |
| EP-08 | CI/CD: pull requests, dependency updates, GitHub Actions to AWS via OIDC | Contributors, AC-09 |
| EP-09 | SQS queues, EventBridge Scheduler, worker | Internal only |

### 3.5 Trust boundaries

`ARCHITECTURE.md` section 11.1 defines TB-1 to TB-6 and the participant-to-participant boundary TB-P. Section 6 describes how each one is enforced.

---

## 4. Threat model

Method: STRIDE per trust boundary and asset, plus abuse cases specific to location sharing. This is the threat model QA-09 requires. T-22 and RR-01 record the PRIV-06 residual risk.

Likelihood (L) and impact (I) use High, Medium, and Low. They are design-time estimates (ASSUMPTION), to be recalibrated after the first penetration test (QA-07).

### 4.1 Spoofing

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-01 | A stolen refresh or access token is replayed from another device | M | H | SEC-AUTH-05, SEC-AUTH-06, SC-AUTH-05, SEC-MOB-01 | Low if DPoP ships (SD-01). Medium if it is deferred |
| T-02 | Account takeover through IdP account linking by email (a different IdP account with the same or a reused email) | M | H | SC-AUTH-07 | Low |
| T-03 | A forged or misissued IdP ID token (algorithm confusion, wrong audience, replayed nonce) | L | H | SEC-AUTH-03 | Low |
| T-04 | A forged Halfsies access token (`alg: none`, HS/ES confusion, unknown `kid`) | L | H | SEC-AUTH-04 | Low |
| T-05 | A malicious app on the device intercepts the invite link or the OIDC redirect | M | H | SEC-INV-03, SD-04, SEC-AUTH-01, RN-LNK-01 | Low. See SQ-05 for Apple sign-in on Android |
| T-06 | Invite secrets are brute-forced online | L | H | SEC-INV-01, SEC-INV-02, SEC-RL-02 | Negligible (256-bit secret) |
| T-07 | A forwarded invite link is redeemed by someone the Initiator did not intend | M | M | SEC-INV-04, FR-SES-04, FR-SES-07 | Accepted. The link is a bearer credential by design. The Initiator sees who joined and can end the session |
| T-08 | A scripted client impersonates the app to farm provider calls | H | M | SEC-RL-01, SEC-RL-05, SC-COST-01 | Medium. Attestation is not the sole control (SEC-RL-05) |

### 4.2 Tampering

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-09 | BOLA: a caller reads or writes a session they are not in (API1:2023) | H | H | SEC-AZ-01, SC-AZ-02 | Low |
| T-10 | A participant sets or changes the other participant's origin or mode | M | H | FR-ORG-05, SC-AZ-03 | Low |
| T-11 | The proposer accepts their own proposal, or a race creates two Plans | M | M | FR-RES-03, FR-RES-04 (locked accept and unique indexes, `ARCHITECTURE.md` 7.2) | Low |
| T-12 | Mass assignment or role injection through extra body fields (API3:2023) | M | H | SEC-VAL-01, SEC-AZ-04 | Low |
| T-13 | Prototype pollution through JSON bodies, headers, or provider responses | M | H | SC-VAL-04, JS-OBJ-01 to JS-OBJ-04 | Low |
| T-14 | A malicious or malformed provider response injects fields, non-https URLs, or oversized data (API10:2023) | L | M | SEC-SEC-03 | Low |
| T-15 | An origin ciphertext is copied into another participant's row | L | H | SC-CRYPTO-02 | Low |
| T-16 | Invite or refresh token redemption races produce double use | M | H | SC-INV-02, SEC-AUTH-05 | Low |
| T-17 | A deep link triggers a state change the user did not confirm | M | M | SEC-MOB-06, RN-LNK-03 | Low |
| T-18 | A tampered app binary or JS bundle, or a rooted or jailbroken device, bypasses client checks | H | M | RN-TRUST-01, SEC-RL-05 | Accepted. All decisions are enforced server-side |

### 4.3 Repudiation

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-19 | A security-relevant action (token reuse, invite brute force, deletion) cannot be reconstructed | M | M | SEC-LOG-01, SEC-LOG-03 | Low |
| T-20 | Audit logs are tampered with or deleted by an insider | L | M | SC-LOG-04 | Medium. Log immutability is TO BE DECIDED (SQ-03) |

### 4.4 Information disclosure

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-21 | **The other participant gets the precise origin** from API responses (AC-03) | H | H | PRIV-03, API-SHP-01, API-SHP-02, SC-PRIV-01 | Low |
| T-22 | **The other participant trilaterates the origin** from `tA`, `tB`, and results across repeated searches while moving their own origin (PRIV-06) | M | H | SC-PRIV-03, SEC-RL-03 | **Accepted residual risk for v1 (SD-03).** Rounding is proposed, not confirmed (OD-06) |
| T-23 | The other participant infers the counterpart's region from the geometry of the result set | M | M | API-SHP-02, API-SHP-03 | Accepted. This is inherent to the product. It is bounded by the snapped area already disclosed |
| T-24 | Coordinates or addresses leak into logs, traces, metrics, crash reports, WAF, CloudFront, or ALB logs (PRIV-05) | H | H | SEC-LOG-02, SC-LOG-05, SC-MOB-06 | Medium until SQ-07 is resolved (autocomplete query string) |
| T-25 | Location or address in push payloads or on the lock screen | M | H | FR-NOT-02, SC-NOT-01 | Low |
| T-26 | Location in invite share text or directions deep links | M | H | SC-INV-05, FR-RES-06, SC-MOB-08 | Low |
| T-27 | Session existence probing or ID enumeration | M | L | SEC-AZ-01 (404), API-06 | Low |
| T-28 | Errors leak internals (stack traces, SQL, hostnames, provider payloads) | M | M | API-04, NODE-DIAG-04 | Low |
| T-29 | Database, snapshot, or backup exfiltration exposes origins | L | H | SC-CRYPTO-01, SC-CRYPTO-03, SC-DATA-04 | Low |
| T-30 | Device backup, filesystem extraction, or screenshots expose tokens, origins, or the cached Plan | M | M | SEC-MOB-01, SEC-MOB-02, SEC-MOB-03, SC-MOB-03 | Low |
| T-31 | Provider API keys leak from the app bundle or the repository | M | H | SEC-SEC-01, SEC-SEC-02, SEC-SEC-04 | Low |
| T-32 | Data export (`GET /v1/me/export`) is abused with a stolen token | L | M | T-01 controls, SEC-RL-01 | Low |
| T-33 | Precise origins are retained past the purge obligations in section 6.3 of `REQUIREMENTS.md` | M | H | SC-DATA-04, SC-DATA-05 | Low |

### 4.5 Denial of service

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-34 | **Denial of wallet**: automated searches or autocomplete calls drain the provider budget (API6:2023) | H | H | SC-COST-01, NFR-COST-01, FR-SRCH-11, SEC-RL-03, SEC-RL-05 | Medium. Budgets are TO BE DECIDED (OD-08) |
| T-35 | Request floods, slow clients, oversized bodies (API4:2023) | H | M | SEC-RL-01, NODE-HTTP-01 to NODE-HTTP-04, SC-HTTP-02 | Low |
| T-36 | ReDoS, or deep or huge JSON exhausting CPU or memory | M | M | JS-JSON-01, JS-RGX-01 | Low |
| T-37 | A provider outage stalls request handlers | M | M | SC-PROV-03 | Low |
| T-38 | Invite DoS: an attacker who knows an invite ID burns its 5 failed attempts | L | L | SEC-RL-02 | Accepted. Only link holders know the invite ID, and the Initiator can rotate (FR-SES-04) |

### 4.6 Elevation of privilege

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-39 | A guest token reaches `/v1/me` account endpoints | M | M | SEC-AZ-03 | Low |
| T-40 | Guest upgrade links a guest to someone else's account | L | H | SC-AUTH-08 | Low |
| T-41 | A malicious dependency or install script runs in CI or production (AC-09) | M | H | SEC-SC-01, SEC-SC-02, NODE-MOD-01 | Medium |
| T-42 | A compromised CI workflow deploys to production or exfiltrates secrets | L | H | SC-CICD-01, SC-CICD-02 | Low |
| T-43 | A tampered release or over-the-air update ships to users | L | H | SEC-SC-03, SEC-SC-04, AD-03 | Low |
| T-44 | SSRF: the server fetches a URL chosen by a client or a provider | L | H | SEC-SEC-03, NODE-NET-01 | Low. By design, no server code fetches a caller-supplied URL |
| T-45 | Script injection on static web pages steals invite fragments or hijacks the fallback page | L | M | SC-WEB-01, SC-WEB-02 | Low |

### 4.7 Abuse cases specific to location sharing

| ID | Abuse case | Response |
|---|---|---|
| AB-01 | An abusive partner uses Halfsies to learn a victim's home area | The victim chooses their own starting point, which does not have to be home (FR-ORG-01). The counterpart only ever sees the snapped area (PRIV-03). The victim can leave at any time (FR-SES-05). There is no in-product block or report function in v1 (UNKNOWN whether one is needed, SQ-09) |
| AB-02 | Repeated sessions with the same person, to trilaterate over time | Each session exposes only snapped areas and rounded times. Whether to limit repeat sessions between the same accounts is TO BE DECIDED (SQ-09) |
| AB-03 | A shared or borrowed device reveals past Plans | Offline cache holds only the Plan summary (NFR-AV-03). Sign-out clears user-scoped state (SC-MOB-04) |

### 4.8 Accepted residual risks

| ID | Risk | Accepted by | Revisit |
|---|---|---|---|
| RR-01 | Trilateration from travel times (T-22, PRIV-06) | TO BE DECIDED (Security and Product, OD-06) | M4 |
| RR-02 | Region inference from result geometry (T-23) | TO BE DECIDED | M4 |
| RR-03 | A forwarded invite is a bearer credential (T-07) | TO BE DECIDED | M2 |
| RR-04 | Software-backed DPoP keys on Android devices without StrongBox or TEE (SD-01) | TO BE DECIDED | M1 |
| RR-05 | Client-side checks are bypassable on rooted or jailbroken devices (T-18) | Inherent; server enforcement is the control | Never |

---

## 5. Security requirements and controls

`SEC-*` rows are the normative security requirements. `SC-*` rows are controls this document adds; each traces to the requirement or guidance it serves. Rows marked **[AC]** carry acceptance criteria tested per QA-02. "Verify" refers to the activities in section 9.

### 5.1 Authentication and tokens (ASVS V6, V7, V9, V10; MASVS-AUTH)

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SEC-AUTH-01 | Apps MUST authenticate users via OpenID Connect Authorization Code flow with PKCE (S256), using the system browser or platform auth session (`ASWebAuthenticationSession`, Android Custom Tabs) per RFC 8252. Embedded WebViews for login are prohibited. Sign in with Apple MAY use the native `AuthenticationServices` API. Control: OIDC redirects arrive only on verified links (RN-LNK-01); `state` and `nonce` are single-use, bound to the pending flow, and checked before dispatch | RFC 8252 | MASTG. Code review. SQ-05 |
| SEC-AUTH-02 | Implicit grant and Resource Owner Password Credentials grant MUST NOT be used (RFC 9700 sections 2.1.2, 2.4) | RFC 9700 | Code review |
| SEC-AUTH-03 | The Halfsies API issues its own access and refresh tokens after validating the IdP ID token (signature, `iss`, `aud`, `exp`, `nonce`). Apps MUST NOT send IdP tokens to the API for ongoing authorization. Control: per IdP, the signature is checked against that IdP's JWKS with `alg` pinned per IdP, plus `iat` skew and `nonce` equal to the stored value. The API redeems codes itself (SD-05) | ASVS V10 | Fixture tests with tampered tokens |
| SEC-AUTH-04 | Access tokens: lifetime at most 15 minutes, audience-restricted to the Halfsies API. Control: JWT signed with ES256, claims `iss`, `aud=api.<domain>`, `sub`, `sub_type` (`account` or `guest`), `sid` (guest only), `cnf.jkt`, `iat`, `exp`, `jti`. The verifier accepts only ES256, a known `kid`, the expected `iss` and `aud`, an unexpired `exp`, and a `cnf.jkt` that matches the DPoP proof. `alg` values outside the allowlist are rejected before any key lookup. Signing keys: SC-SEC-03 | ASVS V9 | Negative tests: `none`, HS256, unknown `kid`, expired, wrong `aud` |
| SEC-AUTH-05 **[AC]** | Refresh tokens MUST rotate on every use. Reuse of a previously used refresh token MUST revoke the entire token family (RFC 9700 section 4.14.2). AC: replaying a rotated refresh token returns 401 and invalidates the current token. Control: refresh tokens are 256-bit CSPRNG values stored as SHA-256 and rotated by a conditional update (`ARCHITECTURE.md` 7.2); reuse emits a `token_reuse` event. The app's token manager single-flights refreshes, so a legitimate client never presents a token twice and any reuse is treated as theft | RFC 9700 | SEC-AUTH-05 test. Server and app concurrency tests |
| SEC-AUTH-06 | Tokens SHOULD be sender-constrained with DPoP (RFC 9449) using a non-exportable key held in Secure Enclave (iOS) or hardware-backed Android Keystore where available. If DPoP is deferred, the decision and compensating controls are recorded in SD-01. Control: `halfsies-device-security` creates a P-256 key on first sign-in (Secure Enclave; StrongBox or TEE; software Keystore fallback per SD-01). Once SD-01 is confirmed, every request carries a proof (`htm`, `htu`, `iat`, `jti`, and `ath` for resource requests). The API checks the signature, compares the key thumbprint with `cnf.jkt`, requires `iat` within ±60 s, and rejects a repeated `jti` using Redis. Refresh tokens are bound to the `jkt` at issuance | RFC 9449 | Replay and wrong-key tests |
| SEC-AUTH-07 | Refresh token absolute lifetime: 30 days for accounts, session lifetime for guests | ASVS V7 | TO BE DECIDED |
| SEC-AUTH-08 | Sign-out MUST revoke the refresh token server-side and delete local credentials. Control: `POST /v1/auth/revoke` revokes the whole family; the app clears state per SC-MOB-04 | ASVS V7 | App test |
| SC-AUTH-05 | Immediate revocation: the subject denylist (`ARCHITECTURE.md` 7.3) is checked on every request and set for the access token lifetime on account deletion and on family revocation | FR-ACC-04, SEC-AUTH-05 | FR-ACC-04 test |
| SC-AUTH-07 | Accounts are keyed by (`idp_provider`, `idp_subject`) only. They are **never** linked or merged by email address, and email is not used as an identifier | FR-ACC-06; OWASP Authentication Cheat Sheet | Test: same email from a different IdP creates a separate account |
| SC-AUTH-08 | Guest upgrade (`grant_type=guest_upgrade`) requires both a valid guest access token (with DPoP) and a fresh IdP authorization code in the same request. In one transaction it re-points exactly one participant row from `guest_id` to `user_id`, deletes the guest row, and revokes the guest token family | FR-ACC-03 | Test that one guest cannot upgrade into another guest's session |

Authenticator lifecycle (NIST SP 800-63B-4): Halfsies holds no passwords or OTP authenticators. Authentication is delegated to Apple and Google, so AAL is inherited from the IdP (ASSUMPTION: AAL1 is sufficient for v1; SQ-02). Account recovery is IdP recovery.

### 5.2 Invites and guest access

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SEC-INV-01 | Invite tokens MUST contain at least 128 bits of entropy from a CSPRNG. Control: the token is `inviteId.secret`, and the secret is 256 bits from `crypto.randomBytes` | ASVS V7 | Unit tests |
| SEC-INV-02 | The API MUST store only a hash (SHA-256) of invite tokens and compare in constant time. Control: only the secret part is hashed and compared, per JS-CRY-03 | ASVS V11 | Unit tests |
| SEC-INV-03 | Invite links MUST use Universal Links (iOS) and verified Android App Links. Custom URL schemes MUST NOT carry invite tokens, since any app can register them. The secret travels in the URL fragment (SD-04) | MASVS-PLATFORM | MASTG deep link tests |
| SEC-INV-04 | Redemption issues guest credentials scoped to exactly one session (`sub_type=guest`, `sid` set), following SEC-AUTH-04 through SEC-AUTH-06 | ASVS V8 | Guest scope test |
| SEC-INV-05 | Invite redemption is rate limited per SEC-RL-02 and all failed redemptions are logged as security events | ASVS V16 | Tests |
| SEC-INV-06 | The invite token MUST be removed from app navigation state and not persisted after redemption | MASVS-STORAGE | MASTG deep link tests |
| SC-INV-02 | Redemption is race-safe: the single-use update and the participant insert share one transaction (`ARCHITECTURE.md` 7.2) | FR-SES-03 | FR-SES-03 test, concurrent redemption test |
| SC-INV-05 | Share text comes from a fixed template with no location fields | FR-SES-08 | Snapshot test |

### 5.3 Authorization (ASVS V8; API1, API3, API5)

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SEC-AZ-01 **[AC]** | Every session-scoped endpoint MUST verify the caller is a current participant of that session, enforced in a single shared authorization layer, deny by default. AC: for every endpoint in section 4.2 of `REQUIREMENTS.md`, an automated test calls it with a valid token from a non-participant and expects 404 (to avoid confirming existence). Control: the guard loads the caller's participant row by the path `sessionId` **and** the token subject; no row returns 404 | API1:2023 | BOLA suite |
| SEC-AZ-02 | Initiator-only actions (end session, manage invites) MUST check the participant role server-side | API5:2023 | BOLA suite |
| SEC-AZ-03 | Guest credentials MUST NOT access any `/v1/me` account endpoint except as defined for guest upgrade (SC-AUTH-08). Control: tokens with `sub_type=guest` are rejected on every `/v1/me*` route | API5:2023 | Test for each route |
| SEC-AZ-04 | Authorization decisions MUST NOT rely on client-supplied role or participant fields. Roles and participant identity come only from the database and the token | API3:2023 | Mass-assignment test |
| SC-AZ-02 | Every route declares a policy (`public`, `account`, `participant`, `initiator`, `proposer`, `nonProposer`). A route without one fails at startup | SEC-AZ-01, SEC-AZ-02 | Startup test that enumerates every route and its policy |
| SC-AZ-03 | Participant writes exist only as `participants/me` routes (section 4.2 of `REQUIREMENTS.md`); CI checks the route inventory | FR-ORG-05 | Route inventory test |

### 5.4 Input validation and business logic (ASVS V1, V2; API8)

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SEC-VAL-01 | All request bodies MUST be validated against the OpenAPI schema with unknown properties rejected (`additionalProperties: false`). Control: query and path parameters are validated the same way, with type coercion off (Ajv, `ARCHITECTURE.md` 5.1) | API8:2023 | Contract tests (QA-04), Schemathesis |
| SEC-VAL-02 | Latitude in [-90, 90], longitude in [-180, 180], finite numbers only. Travel mode, category, and price are enums | ASVS V2 | Boundary tests |
| SEC-VAL-03 | Strings have maximum lengths: display name 50, autocomplete query 120, area label 80. Display names are Unicode-normalized (NFC) and stripped of control and bidi override characters | ASVS V2 | Oversize and boundary tests |
| SEC-VAL-04 | Meeting time MUST be within [now minus 5 minutes, now plus 14 days], parsed per JS-NUM-03 | ASVS V2 | Boundary tests |
| SEC-VAL-05 | Database access MUST use parameterized queries or a query builder that parameterizes by default. Control: the Kysely `sql.raw` and `sql.lit` escape hatches are banned outside reviewed migrations | ASVS V1 | Lint rule (TO BE DECIDED) and code review |
| SC-VAL-04 | The JSON parser rejects `__proto__` and `constructor.prototype` keys (Fastify secure JSON parsing, `protoAction` and `constructorAction` set to `error`) | JS-OBJ-02 | Negative tests |
| SC-VAL-06 | `Idempotency-Key` values are bounded in length and charset, and keyed per subject | API-05 | Unit tests |

### 5.5 Rate limiting, cost, and abuse (API4, API6)

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SEC-RL-01 | Per-token and per-IP rate limits on all endpoints, returning 429 with `Retry-After`. Control: per-IP limits apply at the WAF (TBE-01) and in the app | API4:2023 | Load tests |
| SEC-RL-02 | Invite redemption: maximum 10 attempts per IP per hour and 5 failures per session before the invite is revoked. Control: failures are counted per invite (SQ-13) | API4:2023 | Tests |
| SEC-RL-03 | Searches: maximum 20 per session per hour. Autocomplete: maximum 60 per minute per user. Control: a search served from the FR-SRCH-11 cache does not count toward the limit | API6:2023 | Tests |
| SEC-RL-04 | Account creation and session creation limits per account and device to deter automated abuse. Values TO BE DECIDED | API6:2023 | TO BE DECIDED |
| SEC-RL-05 | The API SHOULD verify app integrity using App Attest (iOS) and Play Integrity API (Android) on session creation, invite redemption, and search. Failed attestation increases rate-limit strictness; it MUST NOT be the sole control | MASVS-RESILIENCE | Tests with an attestation stub |
| SC-COST-01 | Provider budget counters (NFR-COST-02) live in Redis and are checked before every provider call; a failed or exhausted check fails closed (JS-ASY-02) | NFR-COST-02 | Budget exhaustion test. Values TO BE DECIDED (OD-08) |

### 5.6 Transport and HTTP hardening (ASVS V12, V13; MASVS-NETWORK)

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SEC-TLS-01 | All traffic uses TLS 1.2 or higher; TLS 1.3 preferred. The API domain sends HSTS. Control: enforced at CloudFront | ASVS V12 | TLS scan in staging |
| SEC-TLS-02 | iOS App Transport Security MUST NOT be disabled. Android `usesCleartextTraffic` MUST be false via Network Security Config. Control: Network Security Config sets `cleartextTrafficPermitted="false"`; platform certificate and hostname verification stay on; release trust anchors contain no user or development CAs | MASVS-NETWORK | MASTG, config test |
| SEC-TLS-03 | Certificate pinning MAY be implemented. If implemented, it MUST pin to public keys with at least one backup pin and include a remote-configurable kill switch process documented in SD-02 | MASVS-NETWORK | Not implemented in v1 (SD-02) |
| SC-HTTP-02 | Responses set `Cache-Control: no-store` on authenticated routes, `Content-Type` with charset, and `X-Content-Type-Options: nosniff` | REST Security Cheat Sheet | Contract tests |

Node server limits are NODE-HTTP-01 to NODE-HTTP-04.

### 5.7 Privacy and location

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SC-PRIV-01 | Area labels are reverse-geocoded from the snapped cell center, never from the precise coordinate, so a label cannot carry more precision than the cell | PRIV-03 | PRIV-03 property test |
| SC-PRIV-03 | **Proposed** (SD-03, OD-06): the serializer rounds both `tA` and `tB` to the nearest 60 s in every response. Ranking and the `even` flag are computed on unrounded values before serialization | PRIV-06 | Unit tests, once confirmed |
| SC-PRIV-04 | The "Who can see what" screen text is generated from the same field inventory as the DTO allowlist | PRIV-08, `DESIGN.md` P-4 | E2E test |
| SC-PRIV-05 | The SEC-SC-01 dependency review checks every SDK against PRIV-02 and PRIV-11 | PRIV-02, PRIV-11 | Dependency review. SQ-12 |

### 5.8 Cryptography, keys, and secrets (ASVS V11, V13; MASVS-CRYPTO; API10)

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SC-CRYPTO-01 | Origins use envelope encryption under `origin-cmk`, which is separate from every data store key. Only the api service's task role has `kms:Decrypt` on it, and only with a matching `purpose` encryption context. The worker has no decrypt permission | PRIV-04 | IaC review, decryption-scope test |
| SC-CRYPTO-02 | The encryption context `{ purpose: "origin", sessionId, participantId }` is required on decryption, so a ciphertext copied into another row will not decrypt. A mismatch fails closed | PRIV-04 | Row-swap test |
| SC-CRYPTO-03 | Every data store (Aurora, Redis, S3, SQS, logs) is encrypted at rest with customer managed KMS keys (SC-SEC-03) | ASVS V11 | IaC policy (Checkov) |
| SEC-SEC-01 | Place, routing, and geocoding provider API keys MUST live only on the server, loaded from a secrets manager. Control: the same applies to IdP, APNs, and FCM credentials and signing keys (SC-SEC-03). They are loaded at startup into frozen configuration (NODE-ENV-02) and never placed in `process.argv`, image-baked environment files, or logs (NODE-ENV-03) | API10:2023 | Image scan, secret scanning |
| SEC-SEC-02 | The only keys permitted in the app are map SDK display keys, restricted by bundle ID and Android package name plus signing certificate fingerprint, with usage quotas set. Control: server keys are restricted by API and to the NAT egress IPs | API10:2023 | Release checklist. Secret scanning on the built bundle (RN-TRUST-02) |
| SEC-SEC-03 | Provider responses MUST be validated against expected schemas before use. Malformed responses are dropped and logged. Provider-supplied URLs (photos, websites) are validated as `https` before being returned to clients. Control: Zod schemas in each adapter; malformed responses are logged by shape, never by content; the server never fetches provider-supplied URLs | API10:2023 | Fixture tests (QA-03) |
| SC-SEC-03 | Keys and secrets are stored and rotated per the table below | Secrets Management Cheat Sheet | Runbook (TO BE DECIDED) |

| Key or secret | Store | Rotation |
|---|---|---|
| `origin-cmk` | KMS customer managed key | Automatic annual rotation |
| Aurora, Redis, S3, SQS, and log encryption keys | KMS customer managed keys, one per data class | Automatic annual rotation |
| JWT signing key (ES256, P-256) | Secrets Manager, loaded into memory at startup (NODE-CRY-03). KMS asymmetric signing was rejected because its request quotas are too low for the refresh volume at scale | 90 days, with two `kid`s overlapping |
| `k_search`, origin header secret | Secrets Manager | 90 days |
| Provider, IdP, APNs, and FCM credentials | Secrets Manager | According to each provider |

### 5.9 Logging and monitoring (ASVS V16)

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SEC-LOG-01 | Security events are logged with timestamp, correlation ID, actor ID, and outcome: sign-in, token refresh failure, token reuse detection, invite created/redeemed/failed/revoked, authorization failures, rate-limit triggers, account deletion. Control: they go to the dedicated CloudWatch Logs group `halfsies-security`, with retention per section 6.3 of `REQUIREMENTS.md` | ASVS V16 | Event emission tests |
| SEC-LOG-02 | Logs MUST comply with PRIV-05. Tokens, invite tokens, and authorization headers MUST be redacted. Control: `pino` redaction removes `authorization`, `dpop`, `cookie`, `set-cookie`, `req.query.q`, and every body field. A final serializer masks decimal coordinate pairs, the invite token format, and JWT-shaped strings. Request logs record route templates (`/v1/sessions/:sessionId`), not raw URLs. OpenTelemetry spans carry no request-parameter attributes; HTTP instrumentation drops `url.query` and provider request bodies | PRIV-05 | PRIV-05 leak test |
| SEC-LOG-03 | Alerts fire on refresh token reuse, spikes in authorization failures, and invite brute-force patterns. Control: CloudWatch metric filters and alarms, routed to on-call through SNS | ASVS V16 | Alarm tests in staging |
| SC-LOG-04 | Security log integrity: CloudTrail enabled and protected by SCP. Immutability of the log group (for example export to a locked archive account) is TO BE DECIDED | Logging Cheat Sheet | SQ-03 |
| SC-LOG-05 | Log sinks outside the application exclude query strings, because `GET /v1/places/autocomplete?q=` carries address text: CloudFront standard logging (v2) uses a field allowlist without the query string; ALB access logs are disabled because their format cannot omit it; WAF logs redact the query string and the `authorization` and `dpop` headers | PRIV-05 | PRIV-05 leak test covers each sink |

### 5.10 Mobile app (MASVS)

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SEC-MOB-01 | Tokens and DPoP keys MUST be stored in the iOS Keychain (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` or stricter) and Android Keystore-backed encrypted storage. Never in plain preferences, files, or logs. Control: this also covers the MMKV encryption key. Storage goes through `expo-secure-store`; AsyncStorage and plain files are never used for secrets or personal data | MASVS-STORAGE | MASTG. Lint bans the AsyncStorage import |
| SEC-MOB-02 | Precise starting points MUST NOT be persisted on device. Control: the offline cache (NFR-AV-03) holds only the session summary and the Plan | MASVS-STORAGE | MASTG storage tests |
| SEC-MOB-03 | Credentials and session data MUST be excluded from cloud and device backups. Control: persisted data is encrypted with a Keychain- or Keystore-protected key and excluded with `NSURLIsExcludedFromBackupKey`, `dataExtractionRules`, and `allowBackup="false"` | MASVS-STORAGE | Config test, MASTG |
| SEC-MOB-04 | Release builds MUST disable debug logging, debuggable flags, and developer menus. Control: release builds also have no console output, remote debugging, packager access, debug certificates, or test endpoints | MASVS-RESILIENCE | Release checklist. CI release config check |
| SEC-MOB-05 | Clipboard MUST NOT be used for tokens. Invite links are shared only through the OS share sheet | MASVS-PLATFORM | Code review |
| SEC-MOB-06 | Deep link handlers MUST validate host, path, and parameters against an allowlist and ignore unknown parameters. Control: the handler also checks the scheme, and the invite handler takes exactly one fragment parameter. Consequential actions need confirmation (RN-LNK-03) | MASVS-PLATFORM | Deep link fuzz tests |
| SC-MOB-03 | Screens showing precise origins or the invite are protected from screenshots and the app switcher preview (ASSUMPTION; `DESIGN.md` does not specify this; SQ-14) | MASVS-PLATFORM | TO BE DECIDED |
| SC-MOB-04 | On sign-out, account deletion, account switch, guest upgrade, and revocation (a 401 after a failed refresh), the app clears tokens, the DPoP key, the TanStack Query cache, MMKV, and user-scoped files | SEC-AUTH-08 | App test |
| SC-MOB-06 | Sentry runs with `sendDefaultPii: false`, and a `beforeSend` and breadcrumb scrubber drops URLs, query strings, and request bodies | OBS-03, PRIV-05 | PRIV-05 leak test includes Sentry |
| SC-MOB-08 | `Linking.openURL` targets come only from an exact allowlist: Apple Maps, Google Maps, and the legal page URLs. `canOpenURL` is not a safety check | FR-RES-06 | Unit tests |

### 5.11 Notifications

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SC-NOT-01 | Push payloads come from an allowlisted template per event: event type, session ID, and at most a place name. The app validates payloads before acting on them (JS-VAL-01) | FR-NOT-02 | Payload snapshot tests |
| SC-NOT-02 | Push tokens are unique per (`platform`, `token`). Re-registration moves ownership to the current authenticated subject | `ARCHITECTURE.md` 7.2 (`devices`) | Unit tests |

### 5.12 Data lifecycle

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SC-DATA-04 | The scheduled purge jobs (`ARCHITECTURE.md` 6.3) enforce the retention table in section 6.3 of `REQUIREMENTS.md` | `REQUIREMENTS.md` 6.3 | Retention tests against a clock-controlled database |
| SC-DATA-05 | Account deletion follows `ARCHITECTURE.md` 8.6. Security logs keep only the random actor ID | FR-ACC-04 | FR-ACC-04 test |

### 5.13 Providers

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SC-PROV-03 | Every provider call and the search as a whole have deadlines (NFR-PERF-05, `ARCHITECTURE.md` 9.3), and an outage fails fast (NFR-AV-02) | NFR-PERF-05, NFR-AV-02 | Fault-injection tests |

### 5.14 Static web

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SC-WEB-01 | Static pages send `Content-Security-Policy: default-src 'none'; img-src 'self'; style-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'` (plus `script-src` only if a page needs script, TO BE DECIDED), along with `Referrer-Policy: no-referrer` and `X-Content-Type-Options: nosniff` | INT-06; XSS Prevention Cheat Sheet | Header test |
| SC-WEB-02 | The invite fallback page MUST NOT read or render the invite token. It contains no script that reads `location.hash` and no third-party resources | INT-06 | Page review |
| SC-WEB-03 | `logo.svg` and every SVG asset, including logo variants, contain no scripts, event handlers, `foreignObject`, or external references | XSS Prevention Cheat Sheet | CI check (TO BE DECIDED) |

### 5.15 Supply chain and CI/CD (NIST SSDF)

| ID | Requirement or control | Traces to | Verify |
|---|---|---|---|
| SEC-SEC-04 | Secrets MUST NOT be committed. CI runs secret scanning on every push and blocks merges on findings. Control: GitHub secret scanning with push protection, and gitleaks | NIST SSDF | CI |
| SEC-SC-01 | Every third-party dependency (app and API) requires a documented review before adoption: known CVEs, maintenance activity, patch cadence, license, and transitive dependency count. Reviews are recorded in `DEPENDENCIES.md` | NIST SSDF | PR checklist |
| SEC-SC-02 | Dependencies are version-pinned with lockfiles. Software composition analysis runs in CI and blocks on critical or high vulnerabilities without a documented exception. Control: one committed lockfile and `npm ci` everywhere; lifecycle scripts are disabled by default (`--ignore-scripts`), with an allowlist for reviewed packages; SCA uses Dependabot and OSV-Scanner; exceptions carry an expiry date. Patch SLAs are TO BE DECIDED (SQ-15) | NIST SSDF | CI |
| SEC-SC-03 | Each release produces an SBOM (CycloneDX or SPDX) for both apps and the API | NIST SSDF | Release audit |
| SEC-SC-04 | Release builds are produced only by CI from protected branches. App signing keys are held in platform-managed signing (App Store Connect, Play App Signing) or an HSM-backed store. Release pipeline: `ARCHITECTURE.md` 14.3 | NIST SSDF | Release audit |
| SC-CICD-01 | GitHub Actions: `permissions:` minimized per workflow, third-party actions pinned to a commit SHA, no `pull_request_target` running untrusted code, production environments with required reviewers | CI/CD Security Cheat Sheet | Workflow lint (TO BE DECIDED) |
| SC-CICD-02 | AWS deploy roles trust only this repository's OIDC `sub` for the named environment and branch. Each role is scoped to deploy actions | CI/CD Security Cheat Sheet | IaC review |
| SC-IAC-01 | Terraform state is encrypted, and no secret appears in variables or state outputs. Plan review and Checkov: AD-11 | IaC Security Cheat Sheet | CI |

---

## 6. Trust-boundary enforcement

Each row names what enforces the boundary and how it is verified.

| ID | Boundary | Enforcement | Verify |
|---|---|---|---|
| TBE-01 | **TB-1 to TB-2**: device and internet to the edge | SEC-TLS-01. AWS WAF on CloudFront with the AWS managed core and known-bad-inputs rule sets, and rate-based rules per IP: a coarse limit globally and a tighter one on `/v1/invites/redeem` and `/v1/auth/*`. Shield Standard. WAF logging per SC-LOG-05 | TLS scan in staging. WAF log inspection in the PRIV-05 test |
| TBE-02 | **TB-2 to TB-3**: edge to application | The ALB accepts only the CloudFront origin-facing prefix list **and** a rotated secret origin header. TLS from CloudFront to the ALB. Client IPs come only from the trusted hop count (NODE-HTTP-05) | DAST attempts direct-to-ALB requests. A unit test covers the client-IP derivation |
| TBE-03 | **Application entry** (inside TB-3) | Every request passes the ordered pipeline in `ARCHITECTURE.md` 5.2. Route policies per SC-AZ-02 | Startup route test. The BOLA suite |
| TBE-04 | **TB-P**: participant to participant | Writes only through `participants/me` (SC-AZ-03), identity from the token (SEC-AZ-04), the other participant serialized per API-SHP-01 and PRIV-03, rounded travel times (SC-PRIV-03), search limits (SEC-RL-03), and proposal roles read from the database (FR-RES-04) | API-SHP-01 contract test, PRIV-03 property test, FR-ORG-05 and FR-RES-04 tests |
| TBE-05 | **TB-3 to TB-4**: application to data | Aurora: IAM database authentication through RDS Proxy, no static database passwords, TLS required (`rds.force_ssl`). Redis: reachable only from the app security group, TLS, IAM authentication. Security groups allow only the app tier. SEC-VAL-05. Origin decryption per SC-CRYPTO-01 | IaC policy checks. An IAM Access Analyzer review (ASSUMPTION: Access Analyzer is available in the organization) |
| TBE-06 | **TB-3 to TB-5**: application to third parties | Egress only through NAT with fixed IPs. Only coordinates are sent to providers, never user identifiers. Responses validated per SEC-SEC-03. Outbound host allowlist and no redirects (NODE-NET-01). Deadlines per JS-ASY-03 | Fixture tests with malformed responses (QA-03) |
| TBE-07 | **TB-6**: build and deploy to AWS | SC-CICD-01, SC-CICD-02, SC-IAC-01. No long-lived AWS keys. Terraform is applied only by CI (AD-11) | OWASP CI/CD cheat sheet review before M1 exits |
| TBE-08 | **Device to app**: other apps and the OS | Verified links only (RN-LNK-01, SEC-INV-03). SEC-MOB-01, SEC-MOB-03. No exported Android components beyond the verified link intent filters (RN-NAT-03) | MASTG tests (QA-07) |
| TBE-09 | **Static web to visitors** | Static files only, with no server-side code. SC-WEB-01, SC-WEB-02 | Automated header check. Page source review |

---

## 7. Secure coding rules

These rules apply to every change to Halfsies code. They are adapted from the JavaScript ES2026, React Native 0.87, and Node.js 26 secure coding prompts and narrowed to the stack in `ARCHITECTURE.md`. Source rules that cannot apply to this stack are listed in 7.5 with the reason.

### 7.1 How the rules are applied

- **Scope tags.** `API` means `services/api` (the api and worker services). `APP` means `apps/mobile`, including native modules. `WEB` means `/web` static pages. `ALL` means everything, including shared `packages/*`, `infra/`, and scripts.
- **Order.** `JS-*` applies first. `RN-*` and `NODE-*` add to it for their scope. A stricter rule wins.
- **Enforcement.** `Lint` means an ESLint rule or plugin (the specific plugins are TO BE DECIDED, SQ-16). `Type` means TypeScript compile. `Test` means an automated test. `CI` means a pipeline gate. `Review` means a pull request checklist item (QA-08 applies to AI-generated code too). `Config` means a checked-in configuration asserted by a test.
- **Exceptions.** A deviation needs a code comment citing the rule ID and an entry in the exceptions log (location TO BE DECIDED, SQ-16), approved by a Security reviewer, with an expiry date.
- **Versions.** The rule sources target React Native 0.87 and Node.js 26. `ARCHITECTURE.md` selects Node.js 24 LTS and does not pin a React Native version. Rules that depend on a specific version are marked **[v]**, and SQ-04 records the conflict.

### 7.2 JavaScript and TypeScript rules (`JS-*`)

**Trust boundaries and validation**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-VAL-01 | Validate every untrusted value at runtime (requests, provider responses, deep links, push payloads, storage reads, native module results). TypeScript types are not validation | ALL | Review, Test |
| JS-VAL-02 | Use allowlist schemas that reject unknown properties: OpenAPI with Ajv on the API, Zod for provider responses and app-side validation | ALL | CI (contract), Test |
| JS-VAL-03 | Check type, size, and encoding first, then normalize (for example NFC for display names), then validate the normalized value, and use only that value from then on | ALL | Test |
| JS-VAL-04 | Keep raw input and validated domain values in distinct types (branded types, for example `ValidatedOrigin`), so unchecked data cannot reach a domain function | ALL | Type |

**Dynamic code**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-DYN-01 | No `eval`, `new Function`, string-valued `setTimeout` or `setInterval`, or `vm` for untrusted data | ALL | Lint |
| JS-DYN-02 | No `import()` or `require()` with a specifier built from runtime data | ALL | Lint |
| JS-DYN-03 | No executable code in JSON, database fields, SSM parameters, feature flags, or push payloads. Operation names map through explicit dispatch tables | ALL | Review |

**Objects and prototypes**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-OBJ-01 | Use `Map` or `Object.create(null)` for dictionaries keyed by untrusted data (for example place IDs, idempotency keys) | ALL | Review |
| JS-OBJ-02 | Reject `__proto__`, `prototype`, and `constructor` keys before any merge or assignment of untrusted objects (API parser: SC-VAL-04) | ALL | Config, Test |
| JS-OBJ-03 | No recursive deep-merge helpers on untrusted data unless their pollution behavior is tested | ALL | Review |
| JS-OBJ-04 | Use `Object.hasOwn` when presence of a property decides control flow or authorization | ALL | Lint |
| JS-OBJ-05 | Deep-freeze validated security configuration (limits, fairness constants, allowlists) at startup, or keep private copies | API | Test |

**URLs and network targets**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-URL-01 | Parse URLs with `new URL()` and check protocol, hostname, port, and credentials. Never use substring, prefix, or suffix matching for trust decisions | ALL | Lint, Review |
| JS-URL-02 | Remote destinations must be `https:` | ALL | Test |

**Structured data**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-JSON-01 | Bound input bytes before `JSON.parse`: Fastify `bodyLimit` per route (value TO BE DECIDED; ASSUMPTION 16 KiB default), and a response size cap for provider calls | API | Config |
| JS-JSON-02 | Treat `JSON.parse` output as unknown until it has been schema-validated. Do not use revivers on untrusted JSON | ALL | Review |
| JS-JSON-03 | Duplicate keys in security-sensitive bodies (auth and redemption) must be rejected, or proven harmless. Mechanism TO BE DECIDED (SQ-11) | API | Test |

**Regular expressions and text**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-RGX-01 | No nested ambiguous quantifiers on untrusted text. Bound the length before any regex runs. Test security patterns (log scrubber, token format) with adversarial long inputs | ALL | Lint (TO BE DECIDED), Test |
| JS-RGX-02 | Escape user text inserted into a pattern with `RegExp.escape()` **[v]**, or avoid dynamic patterns | ALL | Review |

**Numbers, IDs, and time**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-NUM-01 | Reject `NaN`, `Infinity`, partial parses, and implicit coercion in numeric input. Use `Number.isSafeInteger` for counts, limits, and seconds | ALL | Test |
| JS-NUM-02 | Tokens, IDs, and security randomness come only from a CSPRNG (`crypto.randomBytes`, `crypto.randomInt`, `crypto.randomUUID`, `crypto.getRandomValues`, or `gen_random_uuid()`). Never `Math.random` | ALL | Lint |
| JS-NUM-03 | Parse times with an explicit ISO 8601 UTC format and compare instants, never locale strings (NFR-L10N-02, SEC-VAL-04) | ALL | Test |

**Asynchrony**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-ASY-01 | Await or return every promise on security paths. Floating promises are banned | ALL | Lint (`@typescript-eslint/no-floating-promises`) |
| JS-ASY-02 | Fail closed when validation, authorization, attestation, KMS decryption, or budget checks reject or time out | API | Test |
| JS-ASY-03 | Every remote call gets an `AbortSignal` deadline. An abort does not prove a remote side effect did not happen, so provider calls must be idempotent before they are retried | ALL | Review, Test |
| JS-ASY-04 | Stale responses must not overwrite newer identity or session state (for example a refresh that finishes after sign-out) | APP | Test |
| JS-ASY-05 | Release locks, database clients, and temporary secrets in `finally` blocks | ALL | Review |

**Secrets and cryptography**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-CRY-01 | No secrets in source, bundles, URLs, error text, analytics, or serialized state | ALL | CI (secret scanning) |
| JS-CRY-02 | Use platform crypto only (Node `crypto`, Web Crypto, AWS Encryption SDK, Secure Enclave or Keystore). No hand-written primitives | ALL | Review |
| JS-CRY-03 | Compare secret-derived values with `crypto.timingSafeEqual` on equal-length buffers, hashing first when lengths can differ | API | Test |
| JS-CRY-04 | Base64 and hex are encodings, not protection | ALL | Review |

**Storage and isolation**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-STO-01 | Privileged data never goes into caches shared across users or trust levels. Redis keys are scoped per subject or per session. The search cache is scoped per session (FR-SRCH-11) | API | Review |

**Dependencies and build output**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-DEP-01 | Install from the expected registry. Review lifecycle scripts and native binaries | ALL | CI |
| JS-DEP-02 | Keep test fixtures, debug code, and private source maps out of release bundles. Source maps go only to access-controlled crash tooling | APP, WEB | CI |
| JS-DEP-03 | Review transitive dependency diffs before promoting a build | ALL | Review |

**Browser DOM** (static web only)

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-DOM-01 | Static pages use no `innerHTML`-family sinks and no runtime HTML construction. If script is ever added, use text APIs only, plus a CSP with `require-trusted-types-for 'script'` | WEB | Review, Config |

### 7.3 React Native rules (`RN-*`)

These add to `JS-*` for `APP`. The source also requires React 19 and TypeScript 7 rule sets. Those were not supplied (SQ-17).

**Client trust**

| ID | Rule | Enforce |
|---|---|---|
| RN-TRUST-01 | Treat the bundle, device state, component state, and client decisions as attacker controlled. Authentication, authorization, fairness, ranking, and limits are enforced on the server; the client never recomputes them (`DESIGN.md` R-2) | Review |
| RN-TRUST-02 | The bundle and assets contain no API secrets, private keys, or admin endpoints. Map display keys are the only exception (SEC-SEC-02) | CI (secret scanning on the built bundle) |

**Networking**

| ID | Rule | Enforce |
|---|---|---|
| RN-NET-03 | Bound request sizes, retries, and timeouts in the API client. No offline mutation queue | Test |
| RN-NET-04 | Credentials never appear in URLs, analytics, crash reports, network logs, the clipboard, or screenshots | Test (PRIV-05), Review |

**Local data**

| ID | Rule | Enforce |
|---|---|---|

**Deep links and outbound URLs**

| ID | Rule | Enforce |
|---|---|---|
| RN-LNK-01 | Only verified Universal Links and App Links. No custom scheme carries tokens, codes, or session state | Config, MASTG |
| RN-LNK-03 | A deep link must not trigger a consequential action (invite redemption, leaving, ending) without on-screen confirmation by the authenticated user | Test |

**WebViews**

| ID | Rule | Enforce |
|---|---|---|
| RN-WV-01 | No WebView for any screen and no WebView dependency in v1 (PLAT-03). If one is ever added, JavaScript and bridges are off by default and navigation is restricted, after a security review | CI (dependency deny-list) |

**Native modules and permissions**

| ID | Rule | Enforce |
|---|---|---|
| RN-NAT-01 | `halfsies-device-security` exposes narrow, typed methods (generate key, sign DPoP, attest, assert). Arguments are validated again in Swift and Kotlin. The private key is never exported | Review, Test |
| RN-NAT-02 | When the user denies a permission (FR-ORG-03, FR-NOT-03, PRIV-07), the app handles it without weakening any control | UI test |
| RN-NAT-03 | No exported Android activities, services, or receivers beyond the verified link intent filters and required SDK components. Merged manifests are reviewed every release | CI (manifest check, TO BE DECIDED) |
| RN-NAT-04 | Third-party native modules are reviewed for exported components, permissions, deep links, storage, network use, and binary provenance (SEC-SC-01) | Review |

**Logging and release**

| ID | Rule | Enforce |
|---|---|---|
| RN-REL-01 | Redact credentials, personal data, request and response bodies, and local paths from JS and native logs | Lint, Config |
| RN-REL-04 | **[v]** Use the stable public React Native API. No deep imports into `react-native/Libraries/*` and no path-alias workarounds | Lint, Type |
| RN-REL-05 | Before each major release, test on rooted and jailbroken devices and under proxy interception, deep link spoofing, native bridge abuse, and backup extraction (QA-07) | Test (MASTG) |

### 7.4 Node.js rules (`NODE-*`)

These add to `JS-*` for `API`. Rules marked **[v]** exist only in the Node.js 26 release line. They do not apply on the Node.js 24 LTS runtime that `ARCHITECTURE.md` selects until SQ-04 is resolved.

**Runtime and startup**

| ID | Rule | Enforce |
|---|---|---|
| NODE-RT-01 | Pin the exact Node patch version in the image, CI, and `engines`. Verify `process.version` at startup. Take security releases promptly (SLA TO BE DECIDED, SQ-15) | CI, Test |
| NODE-ENV-01 | Validate required configuration at startup (presence, type, range) and exit non-zero on failure | Test |
| NODE-ENV-02 | Freeze validated configuration. Later changes to `process.env` have no effect | Test |
| NODE-ENV-03 | No secrets in `process.argv`. No `--env-file` in production. `NODE_OPTIONS`, `--require`, and `--import` are fixed in the image | Config |
| NODE-ENV-04 | Do not use `--no-warnings` or `--disable-warning` in production. Route warnings to monitoring. Tests run with `--throw-deprecation` | Config |

**Permission model**

| ID | Rule | Enforce |
|---|---|---|
| NODE-PERM-01 | Start with `--permission`. Grant `--allow-fs-read` only for the app directory. No `--allow-fs-write` (the root filesystem is read-only), no `--allow-child-process`, no `--allow-worker` unless justified, no `--allow-addons`, no `--allow-inspector`. **[v]** On Node 26, add `--allow-net` (network is required) together with the NAT egress control | Config, Test |
| NODE-PERM-02 | The permission model is defense in depth, not a sandbox. Container isolation (NODE-OPS-01) remains required | Config |

**Child processes, addons, FFI**

| ID | Rule | Enforce |
|---|---|---|
| NODE-PROC-01 | The api and worker services do not import `child_process`, `node:ffi`, or native addons. Start with `--no-addons` unless a reviewed dependency needs one | Lint, Config |

**HTTP server**

| ID | Rule | Enforce |
|---|---|---|
| NODE-HTTP-01 | Never enable `insecureHTTPParser`. Keep strict header validation | Config test |
| NODE-HTTP-02 | Finite `requestTimeout` (below the 300 s default), `headersTimeout`, `keepAliveTimeout` (above the ALB idle timeout), and `server.timeout`, plus a bounded `maxRequestsPerSocket`. Values TO BE DECIDED | Config test |
| NODE-HTTP-03 | Bounded `maxHeadersCount` (never 0). Header size matches the ALB and CloudFront limits | Config test |
| NODE-HTTP-04 | Reject CR, LF, and control characters in any value copied into a response header | Test |
| NODE-HTTP-05 | Derive client identity from the trusted proxy chain only: the fixed hop count of CloudFront then ALB. Never take the left-most `X-Forwarded-For` value | Test |

**Outbound requests and TLS**

| ID | Rule | Enforce |
|---|---|---|
| NODE-NET-01 | Outbound hosts form an exact allowlist (Google Maps Platform, Apple, Google IdP, APNs, FCM, Play Integrity, AWS endpoints). No request goes to a caller-supplied or provider-supplied URL. `redirect: 'error'` | Test, Review |
| NODE-NET-02 | The undici dispatcher sets `connections`, `connectTimeout`, and queue bounds per provider | Config test |
| NODE-NET-03 | Never set `rejectUnauthorized: false` or `NODE_TLS_REJECT_UNAUTHORIZED=0`, anywhere, including scripts and health checks. Keep the default TLS minimum and security level | Lint, CI grep |

**Streams and memory**

| ID | Rule | Enforce |
|---|---|---|
| NODE-MEM-01 | Use `Buffer.alloc` and `Buffer.from` only. No `allocUnsafe`, `new Buffer`, or `buffer.slice` | Lint |
| NODE-MEM-02 | Provider response bodies are read with a byte cap. Decompressed size is limited independently of compressed size | Test |
| NODE-MEM-03 | Set `--max-old-space-size` below the task memory limit so a spike ends the task rather than the host | Config |

**Cryptography**

| ID | Rule | Enforce |
|---|---|---|
| NODE-CRY-02 | Any direct AES-GCM use sets `authTagLength: 16` and releases plaintext only after `final()` succeeds. Origin encryption goes through the AWS Encryption SDK, not hand-rolled GCM | Review |
| NODE-CRY-03 | Signing keys (JWT ES256) are loaded as `KeyObject`s from Secrets Manager and never logged or serialized | Review |

**Modules and loaders**

| ID | Rule | Enforce |
|---|---|---|
| NODE-MOD-01 | Production images install with `npm ci --omit=dev` (lockfile and lifecycle scripts: SEC-SC-02) | CI |
| NODE-MOD-02 | No module customization hooks except ones owned by this repository. **[v]** On Node 26, use `module.registerHooks()`, never `module.register()` | Review |
| NODE-MOD-03 | TypeScript uses erasable syntax only if run through type stripping. Types are never runtime validation | Type |
| NODE-MOD-04 | Declare `exports` in internal packages to limit subpath imports. The module boundary lint rules (ARCHITECTURE 5.3) are the privilege boundary | Lint |

**Time and legacy APIs**

| ID | Rule | Enforce |
|---|---|---|
| NODE-TIME-01 | Compare expiry and replay instants numerically in UTC. **[v]** On Node 26, prefer `Temporal` | Review |
| NODE-LEG-01 | Use `new URL()`, never `url.parse()` | Lint |

**Isolation**

| ID | Rule | Enforce |
|---|---|---|
| NODE-ISO-01 | `node:vm` is not a sandbox and is not used. No untrusted code execution. Any `Worker` has `resourceLimits` and a validated `workerData` schema | Lint, Review |

**Diagnostics and logging**

| ID | Rule | Enforce |
|---|---|---|
| NODE-DIAG-01 | No `--inspect` or network inspection in production. Heap snapshots, diagnostic reports, and core dumps are treated as secrets and are disabled in production | Config |
| NODE-DIAG-02 | On `unhandledRejection` or `uncaughtException`: log a sanitized record and exit. ECS replaces the task | Test |
| NODE-DIAG-04 | Clients get generic error bodies. Stacks, module paths, and runtime versions stay in access-controlled telemetry (API-04) | Test |

**Deployment**

| ID | Rule | Enforce |
|---|---|---|
| NODE-OPS-01 | Non-root user, read-only root filesystem, immutable image. Patching means rebuilding the image | Config (IaC) |
| NODE-OPS-02 | On `SIGTERM`: stop accepting connections, drain against a deadline, then close database, Redis, and socket handles | Test |
| NODE-OPS-03 | Health endpoints reveal no configuration, versions, dependency inventory, or stack traces | Test |

### 7.5 Source rules that do not apply

| Source rule area | Why it does not apply |
|---|---|
| JS: cookies, CSRF, `postMessage`, browser storage for session tokens | There is no web client and no cookies (section 1.2 of `REQUIREMENTS.md`). The apps use bearer tokens with DPoP in native secure storage |
| JS: DOMPurify, Trusted Types policy modules, sanitizer configuration | No HTML rendering of untrusted content anywhere. Static pages have no dynamic content (JS-DOM-01 covers the residual) |
| JS: Subresource Integrity for external scripts | Static pages load no third-party resources (SC-WEB-02) |
| RN: WebView hardening details | No WebViews in v1 (RN-WV-01) |
| Node: filesystem path handling, uploads, archives, `Content-Disposition` | The API handles no files or uploads (section 4.2 of `REQUIREMENTS.md`). Revisit if uploads are added |
| Node: `node:sqlite` | The database is Aurora PostgreSQL (ARCHITECTURE AD-07) |
| Node: child process hardening details | Child processes are banned in services (NODE-PROC-01) |
| Node: `--allow-openssl-store`, `--secure-heap` | Not needed by the design. `--secure-heap` is TO BE DECIDED as defense in depth |
| Node: HTTP/2 listener limits | The Node server speaks HTTP/1.1 to the ALB (ASSUMPTION; `ARCHITECTURE.md` does not specify the ALB-to-target protocol). APNs HTTP/2 is outbound only |


---

## 8. Security decisions

| ID | Decision | Status | Rationale and compensating controls |
|---|---|---|---|
| SD-01 | **DPoP (SEC-AUTH-06, OD-04).** Ship DPoP in v1 with hardware-backed P-256 keys, and a software Keystore fallback on Android devices without StrongBox or a TEE | **Proposed**. Owner: Security. TO BE DECIDED by M1 | If deferred, the compensating controls are: 15-minute access tokens, refresh rotation with reuse detection, attestation on sensitive endpoints, the subject denylist, and alerting on refresh from a new device or ASN (the new-ASN signal is an ASSUMPTION; not in `ARCHITECTURE.md`) |
| SD-02 | **Certificate pinning (SEC-TLS-03)** is not implemented in v1, and no pinning code is added unless this decision is revisited with a rotation, backup-pin, and kill-switch design | **Proposed** | Because pinning is not implemented, no kill switch is required. The threats pinning addresses (interception with a rogue CA) are reduced by DPoP binding, platform trust stores with no user CAs in release builds (SEC-TLS-02), and attestation. The outage risk from certificate rotation is avoided |
| SD-03 | **Trilateration (PRIV-06)** is an accepted residual risk for v1. Mitigations: round displayed times (SC-PRIV-03) and limit searches (SEC-RL-03) | **Proposed**. Owners: Security and Product (OD-06) | See T-22 and RR-01. Evaluate a quantitative trilateration test (how precisely an origin can be recovered within the SEC-RL-03 limit) before M4 |
| SD-04 | **Invite secret in the URL fragment** (`ARCHITECTURE.md` 4.4). Browsers never send the fragment to a server, so it never reaches CloudFront, WAF, or fallback-page request logs, and the static fallback page cannot read it server-side | Adopted | Satisfies SC-WEB-02 and SEC-LOG-02 by construction |
| SD-05 | **The API redeems IdP codes as a confidential client** (`ARCHITECTURE.md` 8.1) | Adopted | Apple requires a client-secret JWT for code redemption, which cannot live in the app. Doing the same for Google gives one uniform validation path (SEC-AUTH-03) |
| SD-06 | **Accounts are linked by IdP subject only, never by email** | Adopted (SC-AUTH-07) | Prevents cross-IdP takeover (T-02) |

---

## 9. Verification

| Activity | Covers | When | Requirement |
|---|---|---|---|
| Unit and fixture tests naming the requirement or control ID | Every **[AC]** requirement and every row whose Verify names a test | Every PR | QA-01 to QA-03 |
| Contract tests and Schemathesis | SEC-VAL-*, API-SHP-02, API-SHP-03, API-04 | Every PR | QA-04 |
| BOLA suite: every endpoint called as a non-participant, a guest, the other participant, and the proposer | SEC-AZ-*, SC-AZ-* | Every PR, and DAST | SEC-AZ-01, QA-06 |
| PRIV-05 leak test: a full session flow on staging with known coordinates, then a search of the captured application logs, X-Ray, WAF, CloudFront, and Sentry output | SEC-LOG-02, SC-LOG-05, SC-MOB-06 | Nightly on staging and before release | PRIV-05 |
| PRIV-03 property test (10,000 coordinates), run against the other-participant serializer | PRIV-03, API-SHP-01, SC-PRIV-01 | Every PR | PRIV-03 |
| SAST (CodeQL), SCA, secret scanning, IaC scan, container scan, SBOM | SEC-SC-*, SEC-SEC-04, SC-IAC-01 | Every PR | QA-05 |
| DAST (OWASP ZAP API scan) and direct-to-ALB bypass attempts | TBE-01 to TBE-03, API-04 | Before each release | QA-06 |
| MASVS v2 testing using MASTG | SEC-MOB-*, SC-MOB-*, RN-* | Before first public release and after major changes | QA-07 |
| Threat model review | Section 4 | Per QA-09 | QA-09 |
| External penetration test | All | TO BE DECIDED | UNKNOWN |

---

## 10. Security operations

| Area | Status |
|---|---|
| Incident response plan, on-call, severity levels | TO BE DECIDED. `ARCHITECTURE.md` names SNS alerting to an on-call tool but not the tool itself (UNKNOWN) |
| Breach notification obligations (CCPA/CPRA; GDPR before any EU launch, PRIV-10) | TO BE DECIDED with Legal |
| Vulnerability and patch SLAs (dependencies, Node runtime, base images, mobile SDKs) | TO BE DECIDED (SQ-15) |
| Key and secret rotation runbooks | TO BE DECIDED (SC-SEC-03 sets the cadence) |
| Production access for humans (break-glass, just-in-time access, MFA) | UNKNOWN (SQ-03) |
| Security training and review ownership | UNKNOWN |

---

## 11. Open Security Questions

| ID | Question or conflict | Affects | Owner | Needed by |
|---|---|---|---|---|
| SQ-01 | Who is the security contact, and what is the private reporting channel, response targets, and disclosure policy? | Section 1 | TO BE DECIDED | Before public release |
| SQ-02 | Is AAL1, inherited from the IdPs, sufficient for v1? | Section 5.1 | Security | M1 |
| SQ-03 | Human access model for AWS and GitHub (SSO, MFA, just-in-time, break-glass), and security log immutability. None of the documents specify these | T-20, SC-LOG-04, AC-10 | Engineering, Security | M1 |
| SQ-04 | **Conflict (version).** The Node.js rule source targets Node.js 26, which becomes LTS in October 2026. `ARCHITECTURE.md` selects Node.js 24 LTS. Rules marked **[v]** (the `--allow-net` permission, `module.registerHooks`, `Temporal`) do not apply on Node 24. The React Native source targets 0.87, but `ARCHITECTURE.md` pins no React Native version, and Expo SDK compatibility with 0.87 is UNKNOWN. Recommendation: adopt Node.js 26 LTS before M1 and pin React Native in `ARCHITECTURE.md` | Section 7 | Engineering | M1 |
| SQ-05 | Sign in with Apple on Android uses Apple's web flow (`ARCHITECTURE.md` 8.1). Does it support PKCE as SEC-AUTH-01 requires? If not, choose between dropping Apple sign-in on Android and a documented exception | SEC-AUTH-01, T-05 | Security, Engineering | M1 |
| SQ-07 | `GET /v1/places/autocomplete` puts address text in the query string (section 4.2 of `REQUIREMENTS.md`), which makes PRIV-05 depend on logging configuration in several sinks (SC-LOG-05, SEC-LOG-02). Should the contract change it to `POST` with a body before M3? | T-24, SC-LOG-05 | Engineering | M3 |
| SQ-09 | Abuse handling. Should v1 have block or report, or limits on repeat sessions between the same pair of accounts (AB-01, AB-02)? Neither is in scope in `REQUIREMENTS.md` | Section 4.7 | Product, Security | M2 |
| SQ-11 | Duplicate JSON keys: reject them on auth and redemption bodies (needs a parser that reports duplicates), or document why last-wins is harmless given schema validation | JS-JSON-03 | Engineering | M1 |
| SQ-12 | Sentry must act under a DPA; its DPA status is UNKNOWN. Product analytics has no endpoint or vendor yet (`ARCHITECTURE.md` AQ-03) | SC-PRIV-05, SC-MOB-06, PRIV-11, OBS-03 | Legal, Product | M5 |
| SQ-13 | SEC-RL-02 says "5 failures per session". The design counts failures per invite, since a failed guess identifies the invite through the `inviteId` part of the token. Confirm this reading | SEC-RL-02 | Security | M2 |
| SQ-14 | Screenshot and app switcher protection for screens showing the user's own precise origin or an invite. This is not specified in `DESIGN.md` or `REQUIREMENTS.md` | SC-MOB-03 | Design, Security | M3 |
| SQ-15 | Vulnerability and patch SLAs for dependencies, the Node runtime, base images, and mobile SDKs | SEC-SC-02, NODE-RT-01 | Security | M1 |
| SQ-16 | Choice of lint plugins that enforce section 7, and the location of the rule exceptions log | Section 7.1 | Engineering | M1 |
| SQ-17 | The React Native rule source requires React 19 and TypeScript 7 rule sets first. They were not provided, so this document has no `REACT-*` or `TS-*` rules | Section 7.3 | Security | M1 |
| SQ-18 | Account deletion versus backups. `FR-ACC-04` requires irreversible removal "within 30 days", and backups roll off after 30 days. Is a backup that still holds deleted data on day 30 compliant? (ASSUMPTION: yes, if retention is exactly 30 days, but Legal should confirm) | SC-DATA-05 | Legal | M1 |

---

## 12. References

- OWASP ASVS 5.0.0: https://github.com/OWASP/ASVS/releases/tag/v5.0.0_release
- OWASP MASVS and MASTG: https://mas.owasp.org/
- OWASP API Security Top 10 2023: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- OWASP Top 10 Proactive Controls 2024: https://top10proactive.owasp.org/archive/2024/the-top-10/
- OWASP Cheat Sheet Series: https://cheatsheetseries.owasp.org/ (REST Security, Authentication, Session Management, Input Validation, Secrets Management, Logging, Error Handling, XSS Prevention, CI/CD Security, Software Supply Chain Security, Infrastructure as Code Security, Vulnerable Dependency Management)
- NIST SSDF 1.1 (SP 800-218): https://csrc.nist.gov/pubs/sp/800/218/final
- NIST SP 800-63B-4: https://pages.nist.gov/800-63-4/sp800-63b.html
- RFC 9700, RFC 8252, RFC 9449, RFC 9457
