# SECURITY.md: Halfsies

Version: 1.0.0-draft
Status: Draft for review
Scope: The security posture of the Halfsies iOS app, Android app, Halfsies API, static web content, and the build and deploy pipeline: threat model, security requirements, controls, trust-boundary enforcement, and secure coding rules.
Companion docs: `REQUIREMENTS.md` (source of truth for behavior, actors, data, integrations, privacy and regulatory constraints), `ARCHITECTURE.md` (source of truth for components, trust boundaries, data flows, interfaces), `DESIGN.md` (UI-facing behavior and rendering surfaces), `DEPENDENCIES.md` (to be written)

## How to read this document

- **Precedence.** `REQUIREMENTS.md` wins on behavior. `ARCHITECTURE.md` wins on structure. This file adds security constraints but never silently overrides either one. Where a security constraint conflicts with them, the conflict is recorded in section 12 (Open Security Questions) and the other document stays authoritative until the conflict is resolved.
- **Markers.**
  - `UNKNOWN`: a fact the input documents do not provide.
  - `TO BE DECIDED`: a decision that has not been made.
  - `ASSUMPTION`: an inference the input documents do not support directly. Every assumption must be confirmed or removed before M1 exits.
  - Proposed resolutions in `ARCHITECTURE.md` (for example AD-14, AD-17) are marked **proposed** here. They are not confirmed requirements.
- **Keywords** follow RFC 2119.
- **IDs are stable.**
  - `T-*`: threats
  - `SC-*`: controls
  - `TBE-*`: trust-boundary enforcement
  - `JS-*`, `RN-*`, `NODE-*`: secure coding rules
  - `SD-*`: security decisions recorded here
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
| Supported versions | The two most recent released app versions and the current API (API-02). Nothing has been released yet |

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
6. **Availability** at the 99.5% SLO (NFR-AV-01), including under abuse.

### 2.2 Baseline standards

These baselines are named in section 7 of `REQUIREMENTS.md` and are binding:

- OWASP MASVS v2 (mobile apps)
- OWASP ASVS 5.0 Level 2 (API)
- OWASP API Security Top 10 2023
- RFC 9700 (OAuth 2.0 Security BCP)
- RFC 8252 (OAuth for native apps)
- RFC 9449 (DPoP)
- NIST SP 800-63B
- NIST SP 800-207 (for service-to-service calls)

Supporting guidance used to derive the controls and rules below. It is not binding beyond what the controls state:

- OWASP Top 10 Proactive Controls 2024
- OWASP Cheat Sheets: REST Security, Authentication, Session Management, Input Validation, Secrets Management, Logging, Error Handling, CI/CD Security, Software Supply Chain Security, Infrastructure as Code Security, Vulnerable Dependency Management, and XSS Prevention (static web only)
- NIST SSDF 1.1 (SP 800-218)
- NIST SP 800-63B-4, for authenticator lifecycle only (section 7.2). SQ-02 covers the version mismatch with `REQUIREMENTS.md`

Secure coding rule sources (section 8): the internal secure coding prompts for JavaScript ES2026, React Native 0.87, and Node.js 26.

---

## 3. System summary for threat modeling

This section restates the parts of `ARCHITECTURE.md` that the threat model uses. If the two differ, `ARCHITECTURE.md` wins.

### 3.1 Assets

| ID | Asset | Class (ARCHITECTURE 11.2) | Where it lives |
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
| AS-10 | Security audit logs | Confidential | CloudWatch `halfsies-security` (1 year) |
| AS-11 | Source, build pipeline, signing keys | Secret (keys), Internal (source) | GitHub, GitHub Actions, App Store Connect, Play App Signing, Secrets Manager |
| AS-12 | Provider budget and spend | Internal | Redis counters, provider billing |

### 3.2 Actors

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

### 3.3 Entry points

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

### 3.4 Trust boundaries

These are the boundaries TB-1 to TB-6 from `ARCHITECTURE.md` section 11.1, plus the participant-to-participant boundary, which this document names **TB-P**. Section 6 describes how each one is enforced.

---

## 4. Threat model

Method: STRIDE per trust boundary and asset, plus abuse cases specific to location sharing. This satisfies QA-09 for the design stage. It MUST be updated when a data flow changes and before sessions and invites are implemented (M2).

Likelihood (L) and impact (I) use High, Medium, and Low. They are design-time estimates (ASSUMPTION), to be recalibrated after the first penetration test (QA-07).

### 4.1 Spoofing

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-01 | A stolen refresh or access token is replayed from another device | M | H | SC-AUTH-03, SC-AUTH-04, SC-AUTH-05, SC-MOB-01 | Low if DPoP ships (SD-01). Medium if it is deferred |
| T-02 | Account takeover through IdP account linking by email (a different IdP account with the same or a reused email) | M | H | SC-AUTH-07 | Low |
| T-03 | A forged or misissued IdP ID token (algorithm confusion, wrong audience, replayed nonce) | L | H | SC-AUTH-02 | Low |
| T-04 | A forged Halfsies access token (`alg: none`, HS/ES confusion, unknown `kid`) | L | H | SC-AUTH-01 | Low |
| T-05 | A malicious app on the device intercepts the invite link or the OIDC redirect | M | H | SC-INV-03, SC-AUTH-06 | Low. See SQ-05 for Apple sign-in on Android |
| T-06 | Invite secrets are brute-forced online | L | H | SC-INV-01, SC-INV-02, SC-RL-02 | Negligible (256-bit secret) |
| T-07 | A forwarded invite link is redeemed by someone the Initiator did not intend | M | M | SC-INV-04, FR-SES-04, FR-SES-07 | Accepted. The link is a bearer credential by design. The Initiator sees who joined and can end the session |
| T-08 | A scripted client impersonates the app to farm provider calls | H | M | SC-RL-01, SC-RL-04, SC-COST-01 | Medium. Attestation is not the sole control (SEC-RL-05) |

### 4.2 Tampering

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-09 | BOLA: a caller reads or writes a session they are not in (API1:2023) | H | H | SC-AZ-01, SC-AZ-02 | Low |
| T-10 | A participant sets or changes the other participant's origin or mode | M | H | SC-AZ-03 | Low |
| T-11 | The proposer accepts their own proposal, or a race creates two Plans | M | M | SC-AZ-04, SC-DATA-03 | Low |
| T-12 | Mass assignment or role injection through extra body fields (API3:2023) | M | H | SC-VAL-01, SC-AZ-05 | Low |
| T-13 | Prototype pollution through JSON bodies, headers, or provider responses | M | H | SC-VAL-04, JS-OBJ-01 to JS-OBJ-04 | Low |
| T-14 | A malicious or malformed provider response injects fields, non-https URLs, or oversized data (API10:2023) | L | M | SC-PROV-01, SC-PROV-02 | Low |
| T-15 | An origin ciphertext is copied into another participant's row | L | H | SC-CRYPTO-02 | Low |
| T-16 | Invite or refresh token redemption races produce double use | M | H | SC-DATA-01, SC-DATA-02 | Low |
| T-17 | A deep link triggers a state change the user did not confirm | M | M | SC-MOB-05, RN-LNK-03 | Medium until SQ-06 is resolved |
| T-18 | A tampered app binary or JS bundle, or a rooted or jailbroken device, bypasses client checks | H | M | SC-MOB-07, SC-RL-04 | Accepted. All decisions are enforced server-side |

### 4.3 Repudiation

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-19 | A security-relevant action (token reuse, invite brute force, deletion) cannot be reconstructed | M | M | SC-LOG-01, SC-LOG-03 | Low |
| T-20 | Audit logs are tampered with or deleted by an insider | L | M | SC-LOG-04 | Medium. Log immutability is TO BE DECIDED (SQ-03) |

### 4.4 Information disclosure

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-21 | **The other participant gets the precise origin** from API responses (AC-03) | H | H | SC-PRIV-01, SC-PRIV-02 | Low |
| T-22 | **The other participant trilaterates the origin** from `tA`, `tB`, and results across repeated searches while moving their own origin (PRIV-06) | M | H | SC-PRIV-03, SC-RL-03 | **Accepted residual risk for v1 (SD-03).** Rounding is proposed, not confirmed (OD-06) |
| T-23 | The other participant infers the counterpart's region from the geometry of the result set | M | M | SC-PRIV-02 | Accepted. This is inherent to the product. It is bounded by the snapped area already disclosed |
| T-24 | Coordinates or addresses leak into logs, traces, metrics, crash reports, WAF, CloudFront, or ALB logs (PRIV-05) | H | H | SC-LOG-02, SC-LOG-05, SC-MOB-06 | Medium until SQ-07 is resolved (autocomplete query string) |
| T-25 | Location or address in push payloads or on the lock screen | M | H | SC-NOT-01 | Low |
| T-26 | Location in invite share text or directions deep links | M | H | SC-INV-05, SC-MOB-08 | Low |
| T-27 | Session existence probing or ID enumeration | M | L | SC-AZ-01 (404), API-06 | Low |
| T-28 | Errors leak internals (stack traces, SQL, hostnames, provider payloads) | M | M | SC-ERR-01 | Low |
| T-29 | Database, snapshot, or backup exfiltration exposes origins | L | H | SC-CRYPTO-01, SC-CRYPTO-03, SC-DATA-04 | Low |
| T-30 | Device backup, filesystem extraction, or screenshots expose tokens, origins, or the cached Plan | M | M | SC-MOB-01, SC-MOB-02, SC-MOB-03 | Low |
| T-31 | Provider API keys leak from the app bundle or the repository | M | H | SC-SEC-01, SC-SEC-02, SC-SC-04 | Low |
| T-32 | Data export (`GET /v1/me/export`) is abused with a stolen token | L | M | T-01 controls, SC-RL-01 | Low |
| T-33 | Precise origins are retained past the purge obligations in section 6.3 of `REQUIREMENTS.md` | M | H | SC-DATA-04, SC-DATA-05 | Low |

### 4.5 Denial of service

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-34 | **Denial of wallet**: automated searches or autocomplete calls drain the provider budget (API6:2023) | H | H | SC-COST-01, SC-COST-02, SC-RL-03, SC-RL-04 | Medium. Budgets are UNKNOWN (SQ-08) |
| T-35 | Request floods, slow clients, oversized bodies (API4:2023) | H | M | SC-RL-01, SC-HTTP-01, SC-HTTP-02 | Low |
| T-36 | ReDoS, or deep or huge JSON exhausting CPU or memory | M | M | SC-VAL-02, JS-RGX-01, JS-JSON-01 | Low |
| T-37 | A provider outage stalls request handlers | M | M | SC-PROV-03 (NFR-PERF-05, NFR-AV-02) | Low |
| T-38 | Invite DoS: an attacker who knows an invite ID burns its 5 failed attempts | L | L | SC-RL-02 | Accepted. Only link holders know the invite ID, and the Initiator can rotate (FR-SES-04) |

### 4.6 Elevation of privilege

| ID | Threat | L | I | Controls | Residual |
|---|---|---|---|---|---|
| T-39 | A guest token reaches `/v1/me` account endpoints (SEC-AZ-03) | M | M | SC-AZ-06 | Low |
| T-40 | Guest upgrade links a guest to someone else's account | L | H | SC-AUTH-08 | Low |
| T-41 | A malicious dependency or install script runs in CI or production (AC-09) | M | H | SC-SC-01, SC-SC-02, SC-SC-03, NODE-MOD-01 | Medium |
| T-42 | A compromised CI workflow deploys to production or exfiltrates secrets | L | H | SC-CICD-01, SC-CICD-02 | Low |
| T-43 | A tampered release or over-the-air update ships to users | L | H | SC-SC-05, SC-MOB-09 | Low |
| T-44 | SSRF: the server fetches a URL chosen by a client or a provider | L | H | SC-PROV-02, NODE-NET-01 | Low. By design, no server code fetches a caller-supplied URL |
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

## 5. Security requirements

Section 7 of `REQUIREMENTS.md` holds the normative security requirements (SEC-*), and section 6 holds the privacy requirements (PRIV-*). This document does not restate them. Section 7 below turns each one into controls, and section 8 adds coding rules that apply to every change.

The requirements delegate these items to `SECURITY.md`:

| Delegating requirement | What this document must record | Where |
|---|---|---|
| SEC-AUTH-06 | The DPoP decision, and compensating controls if DPoP is deferred | SD-01 |
| SEC-TLS-03 | Pinning design and kill switch, if pinning is implemented | SD-02 |
| PRIV-06 | The trilateration residual risk in the threat model | T-22, RR-01, SD-03 |
| QA-09 | The threat model | Section 4 |

---

## 6. Trust-boundary enforcement

Each row names what enforces the boundary, where, and how it is verified. "Verify" refers to the tests in section 10.

| ID | Boundary | Enforcement | Verify |
|---|---|---|---|
| TBE-01 | **TB-1 to TB-2**: device and internet to the edge | TLS 1.2 minimum, TLS 1.3 preferred, HSTS on the API domain (SEC-TLS-01). WAF managed rules and rate-based rules. Shield Standard. The WAF redacts query strings and the `authorization` and `dpop` headers in its logs | TLS scan in staging. WAF log inspection in the PRIV-05 test |
| TBE-02 | **TB-2 to TB-3**: edge to application | The ALB accepts only the CloudFront origin-facing prefix list **and** a rotated secret origin header. TLS from CloudFront to the ALB. The API computes client IPs only from the trusted hop count (NODE-HTTP-05) | DAST attempts direct-to-ALB requests. A unit test covers the client-IP derivation |
| TBE-03 | **Application entry** (inside TB-3) | The ordered request pipeline in ARCHITECTURE 5.2: version gate, correlation ID, per-IP limit, authentication (JWT and DPoP), per-token limit, schema validation, route authorization policy, idempotency, handler, allowlist serializer. Routes without a declared policy fail at startup | A startup test enumerates every route and its policy. The BOLA suite |
| TBE-04 | **TB-P**: participant to participant | Writes are scoped to `participants/me` only. Participant identity comes from the token subject, never from the path or body. The other participant is serialized through a dedicated DTO holding only `{ areaLabel, approxLat, approxLng }` snapped to geohash-6. Proposal and accept checks read roles from the database | API-SHP-01 contract test, PRIV-03 property test, FR-ORG-05 and FR-RES-04 tests |
| TBE-05 | **TB-3 to TB-4**: application to data | IAM database authentication through RDS Proxy. TLS required. Security groups allow only the app tier. Queries are parameterized through Kysely (SEC-VAL-05). Only the api task role can `kms:Decrypt` with `origin-cmk`, and only with a matching encryption context. The worker has no decrypt permission | IaC policy checks. An IAM Access Analyzer review (ASSUMPTION: Access Analyzer is available in the organization) |
| TBE-06 | **TB-3 to TB-5**: application to third parties | Egress only through NAT with fixed IPs. Only coordinates are sent to providers, never user identifiers. Every provider response is validated against a Zod schema. Provider hosts are an exact allowlist. Redirects are refused. Every call has a deadline | Fixture tests with malformed responses (QA-03) |
| TBE-07 | **TB-6**: build and deploy to AWS | GitHub OIDC federation to per-environment roles with no long-lived keys. Production deploys require a protected branch and an approval. Terraform is applied only by CI | OWASP CI/CD cheat sheet review before M1 exits |
| TBE-08 | **Device to app**: other apps and the OS | Verified Universal Links and App Links only. The invite secret travels in the URL fragment. Credentials live in Keychain or Keystore. Backups are excluded. No exported Android components beyond the verified link intent filters | MASTG tests (QA-07) |
| TBE-09 | **Static web to visitors** | Static files only, with no server-side code. The fallback page has no script that reads `location.hash`. A strict CSP (SC-WEB-01) | Automated header check. Page source review |

---

## 7. Security controls

Each control lists the requirement IDs it implements, the ASVS 5.0 chapter or MASVS group it relates to, and how it is verified. Controls with no requirement ID are additions from this document; each cites the guidance it came from.

### 7.1 Authorization (ASVS V8; API1, API3, API5)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-AZ-01 | One shared authorization guard, deny by default. It loads the caller's participant row by the path `sessionId` **and** the token subject. If there is no row it returns 404 | SEC-AZ-01 | BOLA test for every endpoint in section 4.2 of `REQUIREMENTS.md` |
| SC-AZ-02 | Every route declares a policy (`public`, `account`, `participant`, `initiator`, `proposer`, `nonProposer`). A route without one fails at startup | SEC-AZ-01, SEC-AZ-02 | Startup test |
| SC-AZ-03 | No route writes another participant's data. Origin and leave routes exist only as `participants/me` | FR-ORG-05, section 4.2 | Route inventory test. See SQ-10 for the FR-ORG-05 acceptance criterion |
| SC-AZ-04 | Accept runs inside a locked transaction that checks `proposed_by <> caller` | FR-RES-04 | FR-RES-04 test |
| SC-AZ-05 | Roles and participant identity come only from the database and the token, never from request fields | SEC-AZ-04 | Mass-assignment test |
| SC-AZ-06 | Guest tokens (`sub_type=guest`) are rejected on every `/v1/me*` route. Guest upgrade is the only exception | SEC-AZ-03 | Test for each route |

### 7.2 Authentication and tokens (ASVS V6, V7, V9, V10; MASVS-AUTH)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-AUTH-01 | The access token verifier accepts only ES256, a known `kid`, `iss`, `aud=api.<domain>`, `exp` within 15 minutes, and a `cnf.jkt` that matches the DPoP proof. `alg` values outside the allowlist are rejected before any key lookup | SEC-AUTH-04 | Negative tests: `none`, HS256, unknown `kid`, expired, wrong `aud` |
| SC-AUTH-02 | ID token validation per IdP: signature against that IdP's JWKS, `alg` pinned per IdP, plus `iss`, `aud`, `exp`, `iat` skew, and `nonce` equal to the stored value | SEC-AUTH-03 | Fixture tests with tampered tokens |
| SC-AUTH-03 | Refresh tokens: 256-bit CSPRNG values, stored as SHA-256, rotated on every use by a conditional update. Reuse revokes the family and emits a `token_reuse` event | SEC-AUTH-05 | SEC-AUTH-05 test |
| SC-AUTH-04 | DPoP proofs are required on every request once SD-01 is confirmed. The API checks the signature, `htm`, `htu`, `iat` within ±60 s, `ath`, and `jti` uniqueness in Redis | SEC-AUTH-06 | Replay and wrong-key tests |
| SC-AUTH-05 | Absolute refresh lifetime: 30 days for accounts, session lifetime for guests. Sign-out revokes the family. Account deletion and family revocation set the Redis subject denylist for 15 minutes | SEC-AUTH-07, SEC-AUTH-08, FR-ACC-04 | FR-ACC-04 test |
| SC-AUTH-06 | OIDC Authorization Code flow with PKCE S256 in `ASWebAuthenticationSession` or Custom Tabs, or the native Apple API on iOS. Redirects use verified `https` links. `state` and `nonce` are single-use and bound to the flow. No WebView login. No implicit or password grants | SEC-AUTH-01, SEC-AUTH-02 | MASTG. Code review. SQ-05 |
| SC-AUTH-07 | Accounts are keyed by (`idp_provider`, `idp_subject`) only. They are **never** linked or merged by email address, and email is not used as an identifier | FR-ACC-06; OWASP Authentication Cheat Sheet | Test: same email from a different IdP creates a separate account |
| SC-AUTH-08 | Guest upgrade requires both a valid guest access token (with DPoP) and a fresh IdP code in the same request. It re-points exactly one participant row and revokes the guest token family | FR-ACC-03 | Test that one guest cannot upgrade into another guest's session |
| SC-AUTH-09 | The token manager single-flights refreshes, so a legitimate client never triggers reuse detection | SEC-AUTH-05 | Concurrency test in the app |

Authenticator lifecycle (NIST SP 800-63B-4): Halfsies holds no passwords or OTP authenticators. Authentication is delegated to Apple and Google, so AAL is inherited from the IdP (ASSUMPTION: AAL1 is sufficient for v1; SQ-02). Account recovery is IdP recovery.

### 7.3 Invites (SEC-INV)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-INV-01 | The token is `inviteId.secret`. The secret is 256 bits from `crypto.randomBytes`. Only SHA-256(secret) is stored, and comparison uses `crypto.timingSafeEqual` on equal-length digests | SEC-INV-01, SEC-INV-02 | Unit tests |
| SC-INV-02 | Single use with a 24-hour expiry, enforced by a conditional `UPDATE ... RETURNING` in the same transaction as the participant insert. The second use returns 410 | FR-SES-03 | FR-SES-03 test, concurrent redemption test |
| SC-INV-03 | The link is `https://<domain>/i#<token>`. The token is in the fragment and is claimed by Universal Links and App Links. Custom schemes never carry it. The handler removes it from navigation state after redemption | SEC-INV-03, SEC-INV-06 | MASTG deep link tests |
| SC-INV-04 | Redemption issues guest credentials scoped to exactly one session | SEC-INV-04 | Guest scope test |
| SC-INV-05 | Share text comes from a fixed template with no location fields | FR-SES-08 | Snapshot test |

### 7.4 Input validation and business logic (ASVS V1, V2; API8)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-VAL-01 | Every request body, query, and path parameter is validated against the OpenAPI schema with `additionalProperties: false`, type coercion off, and enums for mode, category, and price | SEC-VAL-01, SEC-VAL-02 | Contract tests (QA-04), Schemathesis |
| SC-VAL-02 | Size limits are enforced before parsing: a body limit per route (TO BE DECIDED, default ASSUMPTION 16 KiB), plus string maximums (display name 50, autocomplete query 120, area label 80) | SEC-VAL-03; JS-JSON-01 | Oversize tests |
| SC-VAL-03 | Coordinates must be finite and in range. Meeting time must be within [now − 5 min, now + 14 days], parsed from ISO 8601 UTC with an explicit format. Display names are normalized to NFC, and control and bidi override characters are stripped | SEC-VAL-02 to SEC-VAL-04 | Boundary tests |
| SC-VAL-04 | The JSON parser rejects `__proto__` and `constructor.prototype` keys (Fastify's secure JSON parsing, `protoAction` and `constructorAction` set to `error`). Duplicate-key handling is TO BE DECIDED (SQ-11) | JS-OBJ-02, JS-JSON-03 | Negative tests |
| SC-VAL-05 | Parameterized queries only. The Kysely `sql.raw` / `sql.lit` escape hatches are banned outside reviewed migrations | SEC-VAL-05 | Lint rule (TO BE DECIDED) and code review |
| SC-VAL-06 | `Idempotency-Key` values are bounded in length and charset, and keyed per subject | API-05 | Unit tests |

### 7.5 Privacy and location (PRIV)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-PRIV-01 | The other participant's DTO contains only a geohash-6 cell center and an area label reverse-geocoded from the cell center | PRIV-03, API-SHP-01 | PRIV-03 property test, API-SHP-01 contract test |
| SC-PRIV-02 | Response serialization uses explicit mappers and Fastify response schemas. Entities are never serialized directly, and provider payloads are never passed through | API-SHP-02, API-SHP-03 | Contract tests |
| SC-PRIV-03 | Displayed `tA` and `tB` are rounded to 60 s (**proposed**, AD-17 and OD-06). Ranking and the Even trip flag use unrounded values | PRIV-06 | Unit tests, once confirmed |
| SC-PRIV-04 | The "Who can see what" screen text is generated from the same field inventory as the DTO allowlist, and covered end-to-end | PRIV-08, `DESIGN.md` P-4 | E2E test |
| SC-PRIV-05 | No ad, attribution, or data broker SDKs. Analytics, if added, receives no location data and honors opt-out | PRIV-02, PRIV-11 | Dependency review (SEC-SC-01). SQ-12 |

### 7.6 Cryptography and secrets (ASVS V11, V13; MASVS-CRYPTO)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-CRYPTO-01 | Origins use envelope encryption through the AWS Encryption SDK, with the dedicated `origin-cmk` key, separate from database encryption | PRIV-04 | IaC review, decryption-scope test |
| SC-CRYPTO-02 | The encryption context `{ purpose: "origin", sessionId, participantId }` is required on decryption. A mismatch fails closed | PRIV-04 | Row-swap test |
| SC-CRYPTO-03 | Every data store is encrypted at rest with customer managed KMS keys, with annual key rotation | ASVS V11 | IaC policy (Checkov) |
| SC-CRYPTO-04 | All randomness comes from `crypto.randomBytes`, `crypto.randomUUID`, or `gen_random_uuid()`. `Math.random` is banned in security paths | API-06, SEC-INV-01 | Lint rule |
| SC-SEC-01 | Provider, IdP, APNs, and FCM credentials and signing keys live only in Secrets Manager. They are loaded at startup into frozen configuration and never placed in `process.argv`, environment files baked into images, or logs | SEC-SEC-01; NODE-ENV-03 | Image scan, secret scanning |
| SC-SEC-02 | The only keys in the app are Google Maps SDK display keys, restricted by bundle ID, package name, and certificate fingerprint, with quotas. Server keys are restricted to the NAT egress IPs | SEC-SEC-02 | Release checklist |
| SC-SEC-03 | Rotation: JWT signing key, `k_search`, and origin header secret every 90 days. KMS keys annually. Provider credentials per the provider | Secrets Management Cheat Sheet | Runbook (TO BE DECIDED) |

### 7.7 Rate limiting, cost, and availability (API4, API6)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-RL-01 | Per-IP (WAF and app) and per-token limits on every endpoint. Exceeding one returns 429 with `Retry-After` | SEC-RL-01 | Load tests |
| SC-RL-02 | Invite redemption: 10 per IP per hour. After 5 failures against an invite, that invite is revoked. Every failure is logged | SEC-RL-02, SEC-INV-05 | Tests. See SQ-13 for the "per session" interpretation |
| SC-RL-03 | 20 searches per session per hour. 60 autocomplete calls per minute per user | SEC-RL-03 | Tests |
| SC-RL-04 | App Attest and Play Integrity on session creation, redemption, and search. A failure makes the limits stricter but never blocks on its own | SEC-RL-05 | Tests with an attestation stub |
| SC-RL-05 | Limits on account and session creation per account and device | SEC-RL-04 | Values TO BE DECIDED |
| SC-COST-01 | Per-user and global daily provider budgets in Redis, checked before any provider call. Alert at 80% | NFR-COST-02 | Budget exhaustion test. Values UNKNOWN (SQ-08) |
| SC-COST-02 | At most 50 candidates per search. Search results are cached for 15 minutes, keyed by an HMAC input hash | NFR-COST-01, FR-SRCH-11 | Unit tests |

### 7.8 Transport and HTTP hardening (ASVS V12, V13; MASVS-NETWORK)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-TLS-01 | TLS 1.2 minimum and HSTS at CloudFront. iOS ATS stays on. Android `cleartextTrafficPermitted="false"` with no user CAs in release trust anchors | SEC-TLS-01, SEC-TLS-02 | MASTG, config test |
| SC-HTTP-01 | Node server limits: finite `requestTimeout`, `headersTimeout`, and `keepAliveTimeout`, bounded `maxRequestsPerSocket`, and a header size matching the ALB. Values TO BE DECIDED | NODE-HTTP-01 to NODE-HTTP-04 | Config test |
| SC-HTTP-02 | Responses set `Cache-Control: no-store` on authenticated routes, `Content-Type` with charset, and `X-Content-Type-Options: nosniff` | REST Security Cheat Sheet | Contract tests |
| SC-ERR-01 | RFC 9457 Problem Details only. Unknown errors become a generic 500 with the correlation ID. No stack, SQL, hostname, or provider body in any response | API-04; Error Handling Cheat Sheet | DAST, negative tests |

### 7.9 Logging and monitoring (ASVS V16)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-LOG-01 | The security events listed in SEC-LOG-01 are logged with timestamp, correlation ID, actor ID, and outcome to `halfsies-security`, which keeps them 1 year | SEC-LOG-01, section 6.3 | Event emission tests |
| SC-LOG-02 | `pino` redaction of `authorization`, `dpop`, `cookie`, `set-cookie`, `req.query.q`, and every body field. A pattern scrubber masks coordinate pairs, the invite token format, and JWT-shaped strings. Request logs record the route template, not the raw URL | PRIV-05, SEC-LOG-02 | PRIV-05 log-scrubbing test |
| SC-LOG-03 | Alarms on refresh token reuse, spikes in authorization failures, and invite brute-force patterns | SEC-LOG-03 | Alarm tests in staging |
| SC-LOG-04 | Security log integrity: CloudTrail enabled and protected by SCP. Immutability of the log group (for example export to a locked archive account) is TO BE DECIDED | Logging Cheat Sheet | SQ-03 |
| SC-LOG-05 | CloudFront logging uses a field allowlist without the query string. ALB access logs are disabled. WAF redacts the query string. OpenTelemetry drops `url.query` | PRIV-05 (ARCHITECTURE 11.5) | PRIV-05 test covers each sink |

### 7.10 Mobile app (MASVS)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-MOB-01 | Tokens and the MMKV key are kept in Keychain (`AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY` or stricter) and Android Keystore-backed storage. The DPoP key is non-exportable in hardware where available | SEC-MOB-01 | MASTG |
| SC-MOB-02 | Precise origins are never persisted. Offline cache holds only the session summary and the Plan, encrypted, and excluded from backups | SEC-MOB-02, SEC-MOB-03, NFR-AV-03 | MASTG storage tests |
| SC-MOB-03 | Screens showing precise origins or the invite are protected from screenshots and the app switcher preview (ASSUMPTION; `DESIGN.md` does not specify this; SQ-14) | MASVS-PLATFORM | TO BE DECIDED |
| SC-MOB-04 | Sign-out, account deletion, and credential revocation clear tokens, the DPoP key, the TanStack Query cache, MMKV, and user-scoped files | SEC-AUTH-08; RN-DATA-05 | App test |
| SC-MOB-05 | The deep link handler checks scheme, host, and path against an allowlist, takes exactly one fragment parameter, and ignores the rest. Redemption requires on-screen user confirmation (SQ-06) | SEC-MOB-06 | Deep link fuzz tests |
| SC-MOB-06 | Sentry: `sendDefaultPii: false`. `beforeSend` and breadcrumb scrubbing drops URLs, query strings, and bodies. No console logging in release builds | OBS-03, PRIV-05, SEC-MOB-04 | PRIV-05 test includes Sentry |
| SC-MOB-07 | The server enforces every rule. The client never recomputes the Even trip threshold (`DESIGN.md` R-2) or makes authorization decisions | SEC-AZ-04 | Code review |
| SC-MOB-08 | Directions deep links carry only the place coordinate. `Linking.openURL` targets come from an exact allowlist (Apple Maps, Google Maps, the legal page URLs) | FR-RES-06; RN-LNK-04 | Unit tests |
| SC-MOB-09 | Release builds have no dev menu, debuggable flag, packager access, or debug certificates. Over-the-air JS updates are disabled (ARCHITECTURE AD-03) | SEC-MOB-04 | Release checklist |
| SC-MOB-10 | No WebView for any application screen, and no WebView dependency in v1 | PLAT-03 | Dependency check in CI |
| SC-MOB-11 | Location, notification, and calendar permissions are requested only at the point of need after a pre-prompt. No background location | FR-ORG-03, FR-NOT-03, PRIV-07 | MASTG, UI tests |
| SC-MOB-12 | Clipboard is never used for tokens. Invites are shared only through the share sheet | SEC-MOB-05 | Code review |

### 7.11 Notifications

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-NOT-01 | Push payloads come from an allowlisted template per event: event type, session ID, and at most a place name. The app validates payloads before acting on them | FR-NOT-02; RN-DATA-04 | Payload snapshot tests |
| SC-NOT-02 | Push tokens are unique per (`platform`, `token`). Re-registration moves ownership to the current authenticated subject | Section 8 data model | Unit tests |

### 7.12 Data lifecycle

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-DATA-01 | Single-use invite update (SC-INV-02) | FR-SES-03 | Concurrency test |
| SC-DATA-02 | Conditional refresh rotation (SC-AUTH-03) | SEC-AUTH-05 | Concurrency test |
| SC-DATA-03 | A partial unique index enforces one active proposal. A unique (`session_id`, `role`) constraint enforces two participant slots | FR-RES-03, FR-SES-02 | FR-SES-02 test |
| SC-DATA-04 | Purge jobs follow section 6.3 of `REQUIREMENTS.md`. Origin columns are nulled on expiry, end, or leave. Backups roll off after 30 days | Section 6.3 | Retention tests against a clock-controlled database |
| SC-DATA-05 | Account deletion is synchronous and transactional, with the token denylist set, and completed asynchronously. Security logs keep only the random actor ID | FR-ACC-04 | FR-ACC-04 test |

### 7.13 Providers

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-PROV-01 | Every provider response is validated with Zod. Malformed responses are dropped and logged by shape, never by content | SEC-SEC-03 | Fixture tests |
| SC-PROV-02 | Provider-supplied URLs are returned to clients only if they are `https`. The server never fetches them | SEC-SEC-03 | Unit tests |
| SC-PROV-03 | Timeouts of 2 s to connect and 5 s total. At most 2 retries with jitter, on idempotent calls only. A search deadline of 8 s. No straight-line fallback | NFR-PERF-05, NFR-AV-02 | Fault-injection tests |

### 7.14 Static web

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-WEB-01 | Static pages send `Content-Security-Policy: default-src 'none'; img-src 'self'; style-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'` (plus `script-src` only if a page needs script, TO BE DECIDED), along with `Referrer-Policy: no-referrer` and `X-Content-Type-Options: nosniff` | INT-06; XSS Prevention Cheat Sheet | Header test |
| SC-WEB-02 | The invite fallback page contains no script that reads `location.hash` and no third-party resources | INT-06 | Page review |
| SC-WEB-03 | `logo.svg` and any SVG asset contain no scripts, event handlers, `foreignObject`, or external references | `DESIGN.md` Brand and Logo | CI check (TO BE DECIDED) |

### 7.15 Supply chain and CI/CD (NIST SSDF; SEC-SC)

| ID | Control | Implements | Verify |
|---|---|---|---|
| SC-SC-01 | A dependency review is recorded in `DEPENDENCIES.md` before adoption | SEC-SC-01 | PR checklist |
| SC-SC-02 | One committed lockfile. `npm ci` everywhere. Lifecycle scripts are disabled by default (`--ignore-scripts`), with an allowlist for reviewed packages that need them | SEC-SC-02; NODE-MOD-04 | CI |
| SC-SC-03 | SCA (Dependabot, OSV-Scanner) blocks on critical or high findings. Exceptions are documented with an expiry date. Patch SLAs are TO BE DECIDED (SQ-15) | SEC-SC-02 | CI |
| SC-SC-04 | Secret scanning with push protection and gitleaks on every push | SEC-SEC-04 | CI |
| SC-SC-05 | Release builds come only from CI on protected branches. Platform-managed app signing. Upload keys in Secrets Manager via OIDC. An SBOM for each release | SEC-SC-03, SEC-SC-04 | Release audit |
| SC-CICD-01 | GitHub Actions: `permissions:` minimized per workflow, third-party actions pinned to a commit SHA, no `pull_request_target` running untrusted code, production environments with required reviewers | CI/CD Security Cheat Sheet | Workflow lint (TO BE DECIDED) |
| SC-CICD-02 | AWS deploy roles trust only this repository's OIDC `sub` for the named environment and branch. Each role is scoped to deploy actions | CI/CD Security Cheat Sheet | IaC review |
| SC-IAC-01 | Terraform: remote encrypted state, `plan` reviewed on the PR, Checkov blocking, no secrets in variables or state outputs | IaC Security Cheat Sheet | CI |

---

## 8. Secure coding rule system

These rules apply to every change to Halfsies code. They are adapted from the JavaScript ES2026, React Native 0.87, and Node.js 26 secure coding prompts and narrowed to the stack in `ARCHITECTURE.md`. Source rules that cannot apply to this stack are listed in 8.5 with the reason.

### 8.1 How the rules are applied

- **Scope tags.** `API` means `services/api` (the api and worker services). `APP` means `apps/mobile`, including native modules. `WEB` means `/web` static pages. `ALL` means everything, including shared `packages/*`, `infra/`, and scripts.
- **Order.** `JS-*` applies first. `RN-*` and `NODE-*` add to it for their scope. A stricter rule wins.
- **Enforcement.** `Lint` means an ESLint rule or plugin (the specific plugins are TO BE DECIDED, SQ-16). `Type` means TypeScript compile. `Test` means an automated test. `CI` means a pipeline gate. `Review` means a pull request checklist item (QA-08 applies to AI-generated code too). `Config` means a checked-in configuration asserted by a test.
- **Exceptions.** A deviation needs a code comment citing the rule ID and an entry in the exceptions log (location TO BE DECIDED, SQ-16), approved by a Security reviewer, with an expiry date.
- **Versions.** The rule sources target React Native 0.87 and Node.js 26. `ARCHITECTURE.md` selects Node.js 24 LTS and does not pin a React Native version. Rules that depend on a specific version are marked **[v]**, and SQ-04 records the conflict.

### 8.2 JavaScript and TypeScript rules (`JS-*`)

**Trust boundaries and validation**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-VAL-01 | Validate every untrusted value at runtime (requests, provider responses, deep links, push payloads, storage reads, native module results). TypeScript types are not validation | ALL | Review, Test |
| JS-VAL-02 | Use allowlist schemas that reject unknown properties: OpenAPI with Ajv on the API, Zod for provider responses and app-side validation | ALL | CI (contract), Test |
| JS-VAL-03 | Check type, size, and encoding first, then normalize (for example NFC for display names), then validate the normalized value, and use only that value from then on | ALL | Test |
| JS-VAL-04 | Keep raw input and validated domain values in distinct types (branded types, for example `ValidatedOrigin`), so unchecked data cannot reach a domain function | ALL | Type |
| JS-VAL-05 | Authorization happens on the server for every protected operation. Client-side checks are only UX | APP, API | Review |

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
| JS-OBJ-02 | Reject `__proto__`, `prototype`, and `constructor` keys before any merge or assignment of untrusted objects. The API JSON parser errors on them (SC-VAL-04) | ALL | Config, Test |
| JS-OBJ-03 | No recursive deep-merge helpers on untrusted data unless their pollution behavior is tested | ALL | Review |
| JS-OBJ-04 | Use `Object.hasOwn` when presence of a property decides control flow or authorization | ALL | Lint |
| JS-OBJ-05 | Deep-freeze validated security configuration (limits, fairness constants, allowlists) at startup, or keep private copies | API | Test |

**URLs and network targets**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-URL-01 | Parse URLs with `new URL()` and check protocol, hostname, port, and credentials. Never use substring, prefix, or suffix matching for trust decisions | ALL | Lint, Review |
| JS-URL-02 | Remote destinations must be `https:`. Provider URLs passed to clients must be `https:` (SEC-SEC-03) | ALL | Test |
| JS-URL-03 | Deep link, directions, and legal-page URLs are compared against exact origin and path allowlists | APP | Test |

**Structured data**

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-JSON-01 | Bound input bytes before `JSON.parse` (Fastify `bodyLimit` per route, and a response size cap for provider calls) | API | Config |
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
| JS-NUM-01 | Reject `NaN`, `Infinity`, partial parses, and implicit coercion in numeric input. Coordinates must be finite (SEC-VAL-02). Use `Number.isSafeInteger` for counts, limits, and seconds | ALL | Test |
| JS-NUM-02 | Tokens and IDs come only from a CSPRNG (`crypto.randomBytes`, `crypto.randomUUID`, `crypto.getRandomValues`). Never `Math.random` | ALL | Lint |
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
| JS-DEP-01 | Keep a pinned lockfile. Install from the expected registry. Review lifecycle scripts and native binaries | ALL | CI |
| JS-DEP-02 | Keep test fixtures, debug code, and private source maps out of release bundles. Source maps go only to access-controlled crash tooling | APP, WEB | CI |
| JS-DEP-03 | Review transitive dependency diffs before promoting a build | ALL | Review |

**Browser DOM** (static web only)

| ID | Rule | Scope | Enforce |
|---|---|---|---|
| JS-DOM-01 | Static pages use no `innerHTML`-family sinks and no runtime HTML construction. If script is ever added, use text APIs only, plus a CSP with `require-trusted-types-for 'script'` | WEB | Review, Config |

### 8.3 React Native rules (`RN-*`)

These add to `JS-*` for `APP`. The source also requires React 19 and TypeScript 7 rule sets. Those were not supplied (SQ-17).

**Client trust**

| ID | Rule | Enforce |
|---|---|---|
| RN-TRUST-01 | Treat the bundle, device state, component state, and client decisions as attacker controlled. Authentication, authorization, fairness, ranking, and limits are enforced on the server | Review |
| RN-TRUST-02 | The bundle and assets contain no API secrets, private keys, or admin endpoints. Map display keys are the only exception (SEC-SEC-02) | CI (secret scanning on the built bundle) |

**Networking**

| ID | Rule | Enforce |
|---|---|---|
| RN-NET-01 | HTTPS only, with platform certificate and hostname verification on. ATS is not disabled. Android Network Security Config sets `cleartextTrafficPermitted="false"`. Release builds have no user or development CAs | Config, MASTG |
| RN-NET-02 | No certificate pinning unless SD-02 is revisited with a rotation, backup-pin, and kill-switch design | Review |
| RN-NET-03 | Bound request sizes, retries, and timeouts in the API client. No offline mutation queue (NFR-AV-03) | Test |
| RN-NET-04 | Credentials never appear in URLs, analytics, crash reports, network logs, the clipboard, or screenshots | Test (PRIV-05), Review |

**Local data**

| ID | Rule | Enforce |
|---|---|---|
| RN-DATA-01 | Credentials and keys are stored only through `expo-secure-store` or the Keychain and Keystore. AsyncStorage and plain files are never used for secrets or personal data | Lint (ban AsyncStorage import), Review |
| RN-DATA-02 | Persisted data is encrypted with a Keystore- or Keychain-protected key and excluded from backups (`NSURLIsExcludedFromBackupKey`, `dataExtractionRules`, `allowBackup="false"`) | Config, MASTG |
| RN-DATA-03 | Cache only the minimum (session summary, Plan). Never precise origins (SEC-MOB-02) | Test |
| RN-DATA-04 | Validate every server response, persisted value, native module result, and push payload at runtime before use | Test |
| RN-DATA-05 | Clear all user-scoped state on sign-out, account switch, guest upgrade, and revocation (a 401 after a failed refresh) | Test |

**Deep links and outbound URLs**

| ID | Rule | Enforce |
|---|---|---|
| RN-LNK-01 | Only verified Universal Links and App Links. No custom scheme carries tokens, codes, or session state | Config, MASTG |
| RN-LNK-02 | Validate scheme, host, path, and parameters, and correlate OIDC redirects with the pending flow's `state`, before dispatch | Test |
| RN-LNK-03 | A deep link must not trigger a consequential action (invite redemption, leaving, ending) without on-screen confirmation by the authenticated user. This conflicts with the current ARCHITECTURE 4.4 wording (SQ-06) | Test |
| RN-LNK-04 | `Linking.openURL` targets come only from an exact allowlist. `canOpenURL` is not a safety check | Test |

**WebViews**

| ID | Rule | Enforce |
|---|---|---|
| RN-WV-01 | No WebView dependency in v1 (PLAT-03). If one is ever added, JavaScript and bridges are off by default and navigation is restricted, after a security review | CI (dependency deny-list) |

**Native modules and permissions**

| ID | Rule | Enforce |
|---|---|---|
| RN-NAT-01 | `halfsies-device-security` exposes narrow, typed methods (generate key, sign DPoP, attest, assert). Arguments are validated again in Swift and Kotlin. The private key is never exported | Review, Test |
| RN-NAT-02 | Location, calendar, and notification permissions are requested only at the point of need, after the `DESIGN.md` P-2 pre-prompt. Denial is handled without weakening any control | UI test |
| RN-NAT-03 | No exported Android activities, services, or receivers beyond the verified link intent filters and required SDK components. Merged manifests are reviewed every release | CI (manifest check, TO BE DECIDED) |
| RN-NAT-04 | Third-party native modules are reviewed for exported components, permissions, deep links, storage, network use, and binary provenance (SEC-SC-01) | Review |

**Logging and release**

| ID | Rule | Enforce |
|---|---|---|
| RN-REL-01 | Redact credentials, personal data, request and response bodies, and local paths from JS and native logs. Release builds have no console output | Lint, Config |
| RN-REL-02 | Release builds have no dev menu, remote debugging, packager access, debug certificates, or test endpoints | CI (release config check) |
| RN-REL-03 | Signed by platform-managed signing. No OTA updates (AD-03) | Config |
| RN-REL-04 | **[v]** Use the stable public React Native API. No deep imports into `react-native/Libraries/*` and no path-alias workarounds | Lint, Type |
| RN-REL-05 | Before each major release, test on rooted and jailbroken devices and under proxy interception, deep link spoofing, native bridge abuse, and backup extraction (QA-07) | Test (MASTG) |

### 8.4 Node.js rules (`NODE-*`)

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
| NODE-PERM-02 | The permission model is defense in depth, not a sandbox. Container isolation (non-root, read-only root) remains required | Config |

**Child processes, addons, FFI**

| ID | Rule | Enforce |
|---|---|---|
| NODE-PROC-01 | The api and worker services do not import `child_process`, `node:ffi`, or native addons. Start with `--no-addons` unless a reviewed dependency needs one | Lint, Config |

**HTTP server**

| ID | Rule | Enforce |
|---|---|---|
| NODE-HTTP-01 | Never enable `insecureHTTPParser`. Keep strict header validation | Config test |
| NODE-HTTP-02 | Finite `requestTimeout` (below the 300 s default), `headersTimeout`, `keepAliveTimeout` (above the ALB idle timeout), and `server.timeout`, plus a bounded `maxRequestsPerSocket` | Config test |
| NODE-HTTP-03 | Bounded `maxHeadersCount` (never 0). Header size matches the ALB and CloudFront limits | Config test |
| NODE-HTTP-04 | Reject CR, LF, and control characters in any value copied into a response header | Test |
| NODE-HTTP-05 | Derive client identity from the trusted proxy chain only: the fixed hop count of CloudFront then ALB. Never take the left-most `X-Forwarded-For` value | Test |

**Outbound requests and TLS**

| ID | Rule | Enforce |
|---|---|---|
| NODE-NET-01 | Outbound hosts form an exact allowlist (Google Maps Platform, Apple, Google IdP, APNs, FCM, Play Integrity, AWS endpoints). No request goes to a caller-supplied or provider-supplied URL. `redirect: 'error'` | Test, Review |
| NODE-NET-02 | Every request has an `AbortSignal` deadline. The undici dispatcher sets `connections`, `connectTimeout`, and queue bounds per provider | Config test |
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
| NODE-CRY-01 | Randomness from `crypto.randomBytes`, `crypto.randomInt`, or `crypto.randomUUID` | Lint |
| NODE-CRY-02 | Any direct AES-GCM use sets `authTagLength: 16` and releases plaintext only after `final()` succeeds. Origin encryption goes through the AWS Encryption SDK, not hand-rolled GCM | Review |
| NODE-CRY-03 | Signing keys (JWT ES256) are loaded as `KeyObject`s from Secrets Manager and never logged or serialized | Review |

**Modules and loaders**

| ID | Rule | Enforce |
|---|---|---|
| NODE-MOD-01 | Production images use `npm ci --omit=dev` from the committed lockfile. Unreviewed trees install with `--ignore-scripts` | CI |
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
| NODE-DIAG-03 | Redact `authorization`, `dpop`, `cookie`, `set-cookie`, and token-bearing query parameters before logs or traces (SC-LOG-02) | Test |
| NODE-DIAG-04 | Clients get generic error bodies. Stacks, module paths, and runtime versions stay in access-controlled telemetry (SC-ERR-01) | Test |

**Deployment**

| ID | Rule | Enforce |
|---|---|---|
| NODE-OPS-01 | Non-root user, read-only root filesystem, immutable image. Patching means rebuilding the image | Config (IaC) |
| NODE-OPS-02 | On `SIGTERM`: stop accepting connections, drain against a deadline, then close database, Redis, and socket handles | Test |
| NODE-OPS-03 | Health endpoints reveal no configuration, versions, dependency inventory, or stack traces | Test |

### 8.5 Source rules that do not apply

| Source rule area | Why it does not apply |
|---|---|
| JS: cookies, CSRF, `postMessage`, browser storage for session tokens | There is no web client and no cookies (section 1.2 of `REQUIREMENTS.md`). The apps use bearer tokens with DPoP in native secure storage |
| JS: DOMPurify, Trusted Types policy modules, sanitizer configuration | No HTML rendering of untrusted content anywhere. Static pages have no dynamic content (JS-DOM-01 covers the residual) |
| JS: Subresource Integrity for external scripts | Static pages load no third-party scripts (SC-WEB-02) |
| RN: WebView hardening details | No WebViews in v1 (PLAT-03, RN-WV-01) |
| Node: filesystem path handling, uploads, archives, `Content-Disposition` | The API handles no files or uploads (section 4.2 of `REQUIREMENTS.md`). Revisit if uploads are added |
| Node: `node:sqlite` | The database is Aurora PostgreSQL (ARCHITECTURE AD-07) |
| Node: child process hardening details | Child processes are banned in services (NODE-PROC-01) |
| Node: `--allow-openssl-store`, `--secure-heap` | Not needed by the design. `--secure-heap` is TO BE DECIDED as defense in depth |
| Node: HTTP/2 listener limits | The Node server speaks HTTP/1.1 to the ALB (ASSUMPTION; `ARCHITECTURE.md` does not specify the ALB-to-target protocol). APNs HTTP/2 is outbound only |

---

## 9. Security decisions

| ID | Decision | Status | Rationale and compensating controls |
|---|---|---|---|
| SD-01 | **DPoP (SEC-AUTH-06, OD-04).** Ship DPoP in v1 with hardware-backed P-256 keys, and a software Keystore fallback on Android devices without StrongBox or a TEE | **Proposed** (ARCHITECTURE AD-14). Owner: Security. TO BE DECIDED by M1 | If deferred, the compensating controls are: 15-minute access tokens, refresh rotation with reuse detection, attestation on sensitive endpoints, the subject denylist, and alerting on refresh from a new device or ASN (the new-ASN signal is an ASSUMPTION; not in `ARCHITECTURE.md`) |
| SD-02 | **Certificate pinning (SEC-TLS-03)** is not implemented in v1 | **Proposed** (ARCHITECTURE 11.5) | Because pinning is not implemented, no kill switch is required. The threats pinning addresses (interception with a rogue CA) are reduced by DPoP binding, platform trust stores with no user CAs in release builds, and attestation. The outage risk from certificate rotation is avoided |
| SD-03 | **Trilateration (PRIV-06)** is an accepted residual risk for v1. Mitigations: round displayed times to 60 s (proposed, AD-17) and limit searches to 20 per session per hour | **Proposed**. Owners: Security and Product (OD-06) | See T-22 and RR-01. Evaluate a quantitative trilateration test (how precisely an origin can be recovered with 20 searches) before M4 |
| SD-04 | **Invite secret in the URL fragment**, so it never reaches server, CDN, or WAF logs | Adopted in ARCHITECTURE 4.4 | Satisfies INT-06 and SEC-LOG-02 by construction |
| SD-05 | **The API redeems IdP codes as a confidential client** | Adopted in ARCHITECTURE 8.1 | Keeps the Apple client secret server-side and gives one validation path (SEC-AUTH-03) |
| SD-06 | **Accounts are linked by IdP subject only, never by email** | Adopted here (SC-AUTH-07) | Prevents cross-IdP takeover (T-02) |

---

## 10. Verification

| Activity | Covers | When | Requirement |
|---|---|---|---|
| Unit and fixture tests naming the requirement or control ID | Every **[AC]** requirement and every control marked Test | Every PR | QA-01 to QA-03 |
| Contract tests and Schemathesis | SC-VAL-*, SC-PRIV-02, SC-ERR-01 | Every PR | QA-04 |
| BOLA suite: every endpoint called as a non-participant, a guest, the other participant, and the proposer | SC-AZ-* | Every PR, and DAST | SEC-AZ-01, QA-06 |
| PRIV-05 leak test across application logs, X-Ray, WAF, CloudFront, and Sentry | SC-LOG-02, SC-LOG-05, SC-MOB-06 | Nightly on staging and before release | PRIV-05 |
| PRIV-03 property test (10,000 coordinates) | SC-PRIV-01 | Every PR | PRIV-03 |
| SAST (CodeQL), SCA, secret scanning, IaC scan, container scan, SBOM | SC-SC-*, SC-IAC-01 | Every PR | QA-05 |
| DAST (OWASP ZAP API scan) and direct-to-ALB bypass attempts | TBE-01 to TBE-03 | Before each release | QA-06 |
| MASVS v2 testing using MASTG | SC-MOB-*, RN-* | Before first public release and after major changes | QA-07 |
| Threat model review | Section 4 | Before M2, and whenever a data flow changes | QA-09 |
| External penetration test | All | TO BE DECIDED | UNKNOWN |

---

## 11. Security operations

| Area | Status |
|---|---|
| Incident response plan, on-call, severity levels | TO BE DECIDED. `ARCHITECTURE.md` names SNS alerting to an on-call tool but not the tool itself (UNKNOWN) |
| Breach notification obligations (CCPA/CPRA; GDPR before any EU launch, PRIV-10) | TO BE DECIDED with Legal |
| Vulnerability and patch SLAs (dependencies, Node runtime, base images, mobile SDKs) | TO BE DECIDED (SQ-15) |
| Key and secret rotation runbooks | TO BE DECIDED (SC-SEC-03 sets the cadence) |
| Production access for humans (break-glass, just-in-time access, MFA) | UNKNOWN (SQ-03) |
| Security training and review ownership | UNKNOWN |

---

## 12. Open Security Questions

This section includes material conflicts between the security inputs and `REQUIREMENTS.md` or `ARCHITECTURE.md`. Until a question is resolved, the document named as authoritative stays in force.

| ID | Question or conflict | Affects | Owner | Needed by |
|---|---|---|---|---|
| SQ-01 | Who is the security contact, and what is the private reporting channel, response targets, and disclosure policy? | Section 1 | TO BE DECIDED | Before public release |
| SQ-02 | `REQUIREMENTS.md` cites "NIST SP 800-63B" without a revision, while the security guidance references SP 800-63B-4. Which revision is binding, and is AAL1 through the IdPs sufficient? | SC-AUTH-*, section 7.2 | Security | M1 |
| SQ-03 | Human access model for AWS and GitHub (SSO, MFA, just-in-time, break-glass), and security log immutability. None of the documents specify these | T-20, SC-LOG-04, AC-10 | Engineering, Security | M1 |
| SQ-04 | **Conflict (version).** The Node.js rule source targets Node.js 26, which becomes LTS in October 2026. `ARCHITECTURE.md` selects Node.js 24 LTS. Rules marked **[v]** (the `--allow-net` permission, `module.registerHooks`, `Temporal`) do not apply on Node 24. The React Native source targets 0.87, but `ARCHITECTURE.md` pins no React Native version, and Expo SDK compatibility with 0.87 is UNKNOWN. Recommendation: adopt Node.js 26 LTS before M1 and pin React Native in `ARCHITECTURE.md`. `ARCHITECTURE.md` remains authoritative until it changes | Section 8 | Engineering | M1 |
| SQ-05 | Sign in with Apple on Android (ARCHITECTURE AQ-01): does Apple's web flow support PKCE as SEC-AUTH-01 requires? If not, choose between dropping Apple on Android and a documented exception | SC-AUTH-06, T-05 | Security | M1 |
| SQ-06 | **Conflict (deep link consent).** The React Native rule (RN-LNK-03) requires confirmation before a deep link triggers a consequential action. ARCHITECTURE 4.4 has the handler call `POST /v1/invites/redeem` directly. Proposed: show a "Join session?" confirmation screen before redemption. This does not conflict with `REQUIREMENTS.md` | SC-MOB-05, T-17 | Engineering, Design | M2 |
| SQ-07 | `GET /v1/places/autocomplete` puts address text in the query string (section 4.2 of `REQUIREMENTS.md`), which makes PRIV-05 depend on logging configuration in four places (ARCHITECTURE AQ-02). Should it become `POST`? | T-24, SC-LOG-05 | Engineering | M3 |
| SQ-08 | Provider budget values (per user per day, global per day) are UNKNOWN | SC-COST-01, T-34 | Product, Engineering | M4 |
| SQ-09 | Abuse handling. Should v1 have block or report, or limits on repeat sessions between the same pair of accounts (AB-01, AB-02)? Neither is in scope in `REQUIREMENTS.md` | Section 4.7 | Product, Security | M2 |
| SQ-10 | **Conflict (inside REQUIREMENTS).** The FR-ORG-05 acceptance criterion expects `PUT /v1/sessions/{id}/participants/{otherId}/origin` to return 403, but section 4.2 defines only `participants/me` and forbids endpoints that write another participant's data. SEC-AZ-01 prefers 404. Proposed: change the AC to assert that no such route exists (404 or 405) | SC-AZ-03 | Product, Security | M3 |
| SQ-11 | Duplicate JSON keys: reject them on auth and redemption bodies (needs a parser that reports duplicates), or document why last-wins is harmless given schema validation | JS-JSON-03, SC-VAL-04 | Engineering | M1 |
| SQ-12 | Product analytics: no endpoint or vendor exists (ARCHITECTURE AQ-03). Also, the DPA status for Sentry is UNKNOWN, although ARCHITECTURE 4.3 states Sentry acts under one | SC-PRIV-05, PRIV-11, OBS-03 | Legal, Product | M5 |
| SQ-13 | SEC-RL-02 says "5 failures per session". The design enforces it per invite (ARCHITECTURE 8.2), since a failed guess identifies only the invite. Confirm this reading | SC-RL-02 | Security | M2 |
| SQ-14 | Screenshot and app switcher protection for screens showing the user's own precise origin or an invite. This is not specified in `DESIGN.md` or `REQUIREMENTS.md` | SC-MOB-03 | Design, Security | M3 |
| SQ-15 | Vulnerability and patch SLAs for dependencies, the Node runtime, base images, and mobile SDKs | SC-SC-03, NODE-RT-01 | Security | M1 |
| SQ-16 | Choice of lint plugins that enforce section 8, and the location of the rule exceptions log | Section 8.1 | Engineering | M1 |
| SQ-17 | The React Native rule source requires React 19 and TypeScript 7 rule sets first. They were not provided, so this document has no `REACT-*` or `TS-*` rules | Section 8.3 | Security | M1 |
| SQ-18 | Account deletion versus backups. `FR-ACC-04` requires irreversible removal "within 30 days", and backups roll off after 30 days. Is a backup that still holds deleted data on day 30 compliant? (ASSUMPTION: yes, if retention is exactly 30 days, but Legal should confirm) | SC-DATA-05 | Legal | M1 |
| SQ-19 | The proposed decisions (SD-01 DPoP, SD-02 no pinning, SD-03 trilateration, AD-17 rounding, AD-20 region) need sign-off from their owners | Section 9 | Security, Product | M1 to M4 |

---

## 13. References

- OWASP ASVS 5.0.0: https://github.com/OWASP/ASVS/releases/tag/v5.0.0_release
- OWASP MASVS and MASTG: https://mas.owasp.org/
- OWASP API Security Top 10 2023: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- OWASP Top 10 Proactive Controls 2024: https://top10proactive.owasp.org/archive/2024/the-top-10/
- OWASP Cheat Sheet Series: https://cheatsheetseries.owasp.org/ (REST Security, Authentication, Session Management, Input Validation, Secrets Management, Logging, Error Handling, XSS Prevention, CI/CD Security, Software Supply Chain Security, Infrastructure as Code Security, Vulnerable Dependency Management)
- NIST SSDF 1.1 (SP 800-218): https://csrc.nist.gov/pubs/sp/800/218/final
- NIST SP 800-63B-4: https://pages.nist.gov/800-63-4/sp800-63b.html
- RFC 9700, RFC 8252, RFC 9449, RFC 9457
