# REQUIREMENTS.md: Halfsies

Version: 1.0.0-draft
Status: Draft for review
Scope: iOS app, Android app, and the Halfsies API. No web application in this release.
Companion docs: `DESIGN.md` (UI rules), `BRAND_GUIDELINES.md`, `ARCHITECTURE.md` (to be written), `SECURITY.md` (to be written)

Requirement keywords follow RFC 2119. Every requirement has a stable ID. Requirements marked **[AC]** include acceptance criteria that MUST be implemented as automated tests.

---

## 1. Product summary

Two people each enter a starting point. Halfsies returns restaurants and activities where the travel time for both people is roughly equal, ranked by fairness and total travel time. Either person can propose a place, the other accepts, and both get directions.

### 1.1 In scope (v1)

- Native mobile apps for iOS and Android
- Public HTTPS JSON API consumed only by the Halfsies mobile apps
- Exactly two participants per session
- Travel modes: drive, transit, walk, bike
- Place categories: restaurants, cafes, bars, and activities (parks, museums, entertainment)
- Invite by link, shared through the OS share sheet
- Push notifications for session events

### 1.2 Out of scope (v1)

- Web application or web client of any kind (the only web-served content is the static files in section 9.6)
- Sessions with more than two participants
- Reservations, ticketing, or payments
- In-app chat
- User-generated reviews or ratings
- Third-party API access (no public developer API)
- Advertising and ad SDKs

### 1.3 Glossary

| Term | Definition |
|---|---|
| Session | One planning instance between two participants |
| Initiator | Participant A. Creates the session. Requires an account |
| Invitee | Participant B. Joins via invite link. May join as a guest |
| Starting point | A participant's origin coordinate for travel time calculation |
| Candidate | A place considered for ranking |
| `tA`, `tB` | Estimated travel time in seconds from each starting point to a candidate, using each participant's selected mode |
| Fairness delta | `abs(tA - tB)` |
| Even trip | A candidate whose fairness delta is within tolerance (FR-SRCH-06) |
| Proposal | A candidate one participant has suggested to the other |
| Plan | A proposal accepted by both participants |

---

## 2. Platforms and baseline

| ID | Requirement |
|---|---|
| PLAT-01 | The iOS app MUST support the current and previous major iOS version at release time. |
| PLAT-02 | The Android app MUST support Android API level 29 (Android 10) and above, and MUST target the API level required by Google Play at submission time. |
| PLAT-03 | Both apps MUST be native or use a single cross-platform framework chosen in `ARCHITECTURE.md`. The choice MUST NOT rely on WebViews for any application screen. |
| PLAT-04 | Both apps MUST support phone form factors in portrait. Tablet layouts SHOULD scale without breaking. |
| PLAT-05 | The API MUST be the sole backend for both apps. Apps MUST NOT call place or routing providers directly, except for rendering map tiles through the platform map SDK (see SEC-SEC-02). |

---

## 3. Functional requirements

### 3.1 Accounts and identity

| ID | Requirement |
|---|---|
| FR-ACC-01 | The Initiator MUST sign in before creating a session. Supported identity providers: Sign in with Apple and Google. |
| FR-ACC-02 | The Invitee MAY join a session as a guest without creating an account. Guest identity is scoped to one session. |
| FR-ACC-03 | A guest MAY upgrade to a full account at any time; the current session MUST carry over. |
| FR-ACC-04 **[AC]** | Users MUST be able to delete their account in-app (App Store Guideline 5.1.1(v), Google Play account deletion policy). Deletion MUST remove or irreversibly anonymize all personal data within 30 days and immediately revoke all tokens. AC: after `DELETE /v1/me`, all refresh tokens for the user fail with 401 and the user record is unrecoverable via any API. |
| FR-ACC-05 | Users MUST be able to export their personal data in JSON (`GET /v1/me/export`). |
| FR-ACC-06 | Profile data is limited to: display name, avatar initial color choice, identity provider subject identifier, email (only if provided by the IdP). No phone number, birthdate, or contacts access. |

### 3.2 Sessions and invites

| ID | Requirement |
|---|---|
| FR-SES-01 | The Initiator creates a session and receives an invite link. |
| FR-SES-02 **[AC]** | A session MUST have exactly two participant slots. AC: a second successful invite redemption on a session with two participants is rejected with 409. |
| FR-SES-03 **[AC]** | Invite links MUST expire 24 hours after creation and MUST be single-use. AC: a redeemed token returns 410 on second use; an expired token returns 410. |
| FR-SES-04 | The Initiator MUST be able to revoke an unredeemed invite and generate a new one. |
| FR-SES-05 | Either participant MUST be able to leave a session. Leaving removes that participant's starting point immediately. |
| FR-SES-06 | Sessions MUST expire at the earlier of: 24 hours after a scheduled meeting time, or 7 days after creation. Expired sessions are read-only for their Plan summary and purge location data per section 6.3. |
| FR-SES-07 | The Initiator MUST be able to end a session at any time. |
| FR-SES-08 | Share text for the invite MUST follow `BRAND_GUIDELINES.md` section 6.3 and MUST NOT include either participant's location. |

### 3.3 Starting points

| ID | Requirement |
|---|---|
| FR-ORG-01 | Each participant sets their own starting point by address search, current device location, or dropping a pin on a map. |
| FR-ORG-02 | Address search MUST use API-proxied autocomplete (`GET /v1/places/autocomplete`). Minimum 3 characters before a request; client debounce of at least 300 ms. |
| FR-ORG-03 | Current location MUST be requested only on explicit user action, using "while in use" or one-time permission. The app MUST NOT request background location. |
| FR-ORG-04 | Each participant selects their own travel mode (drive, transit, walk, bike). Modes MAY differ between participants. |
| FR-ORG-05 **[AC]** | A participant MUST NOT be able to set or modify the other participant's starting point or mode. AC: `PUT /v1/sessions/{id}/participants/{otherId}/origin` returns 403. |
| FR-ORG-06 | Participants MAY change their starting point until a Plan is confirmed. Changing it invalidates existing search results. |

### 3.4 Search and ranking

| ID | Requirement |
|---|---|
| FR-SRCH-01 | Search runs only when both participants have set a starting point and mode. |
| FR-SRCH-02 | Filters: category (multi-select), open at meeting time, price level, maximum travel time per person (default 60 minutes, range 10 to 120). |
| FR-SRCH-03 | Meeting time: "now" or a scheduled time up to 14 days ahead. Travel time estimates MUST use the meeting time for transit schedules and traffic where the provider supports it. |
| FR-SRCH-04 | Candidate generation MUST NOT rely only on the geographic midpoint. It MUST sample candidates from the region where both participants' reachable areas overlap (for example, intersecting isochrones, or a search area around multiple points along the travel-time-balanced region). The method is documented in `ARCHITECTURE.md`. |
| FR-SRCH-05 **[AC]** | For each candidate the API MUST compute `tA` and `tB` using each participant's own mode and the meeting time. AC: fixture test with known matrix returns expected `tA`, `tB` per candidate. |
| FR-SRCH-06 **[AC]** | A candidate is an Even trip when `abs(tA - tB) <= max(0.15 * max(tA, tB), 300)` seconds. The 0.15 ratio and 300-second floor are server-side configuration (`FAIRNESS_RATIO`, `FAIRNESS_FLOOR_SECONDS`). AC: boundary tests at exactly the threshold, one second over, and one second under. |
| FR-SRCH-07 **[AC]** | Default ranking score: `score = max(tA, tB) + W * abs(tA - tB)`, lower is better, with `W` configurable (default 1.0). Ties break by provider rating, then by place ID for determinism. AC: identical inputs produce identical ordering across 100 runs. |
| FR-SRCH-08 | Candidates where either `tA` or `tB` exceeds the per-person maximum MUST be excluded. |
| FR-SRCH-09 | The API MUST return between 0 and 25 results per search. |
| FR-SRCH-10 | If no results are found, the API MUST return a structured reason (`no_overlap`, `filters_too_narrow`, `provider_unavailable`) so the app can show a directive empty state. |
| FR-SRCH-11 | Search results MUST be cached per session and input hash for 15 minutes to control provider cost. Cache keys MUST NOT contain raw coordinates in logs or metrics. |

### 3.5 Results, proposals, and plans

| ID | Requirement |
|---|---|
| FR-RES-01 | Each result MUST show place name, category, price level, open status at meeting time, `tA`, `tB`, both mode icons, and the Even trip badge when applicable (`DESIGN.md` 5.2). |
| FR-RES-02 | Results MUST be viewable as a list and on a map. The list is the accessible equivalent of the map. |
| FR-RES-03 | Either participant MAY propose a result. Only one active proposal per session; a new proposal replaces the previous one. |
| FR-RES-04 **[AC]** | Only the participant who did not create the proposal can accept it. AC: proposer calling accept returns 403. |
| FR-RES-05 | Accepting a proposal creates the Plan and notifies both participants. |
| FR-RES-06 | From a Plan, each participant MUST be able to open directions in Apple Maps or Google Maps using the place coordinate as destination. The deep link MUST NOT include the other participant's starting point. |
| FR-RES-07 | A participant MAY add the Plan to their device calendar through the OS calendar API with explicit permission. |

### 3.6 Notifications

| ID | Requirement |
|---|---|
| FR-NOT-01 | Push events: invitee joined, results ready, proposal received, plan confirmed, participant left, session ending soon. |
| FR-NOT-02 | Push payloads MUST NOT contain addresses, coordinates, or neighborhood labels. Place names are permitted. |
| FR-NOT-03 | Push permission is requested after the user creates or joins their first session, not at first launch. |
| FR-NOT-04 | Users MUST be able to disable each event type in settings. |

---

## 4. API requirements

### 4.1 Contract

| ID | Requirement |
|---|---|
| API-01 | The API contract MUST be defined in OpenAPI 3.1 before implementation and versioned in the repository. Code is generated or validated against it in CI. |
| API-02 | All endpoints are under `/v1`. Breaking changes require `/v2`. The API MUST support at least the two most recent released app versions. |
| API-03 | Request and response bodies are JSON (`application/json`, UTF-8). |
| API-04 | Errors MUST use RFC 9457 Problem Details (`application/problem+json`). Error responses MUST NOT include stack traces, SQL, provider responses, or internal hostnames. |
| API-05 | All `POST` endpoints that create resources MUST accept an `Idempotency-Key` header and return the original response for repeated keys within 24 hours. |
| API-06 | Resource identifiers MUST be random and non-sequential (UUIDv4 or 128-bit random). |
| API-07 | The API MUST reject requests from app versions below a configured minimum with 426 and a Problem Details body the app uses to prompt an update. |

### 4.2 Endpoints (v1)

| Method | Path | Caller | Purpose |
|---|---|---|---|
| POST | `/v1/auth/token` | App | Exchange authorization code or refresh token (see SEC-AUTH) |
| POST | `/v1/auth/revoke` | App | Revoke refresh token on sign-out |
| GET | `/v1/me` | Account | Profile |
| PATCH | `/v1/me` | Account | Update display name, color |
| DELETE | `/v1/me` | Account | Delete account |
| GET | `/v1/me/export` | Account | Data export |
| POST | `/v1/devices` | Participant | Register push token |
| DELETE | `/v1/devices/{deviceId}` | Participant | Unregister push token |
| POST | `/v1/sessions` | Account | Create session |
| GET | `/v1/sessions/{sessionId}` | Participant | Session state |
| DELETE | `/v1/sessions/{sessionId}` | Initiator | End session |
| POST | `/v1/sessions/{sessionId}/invites` | Initiator | Create or rotate invite |
| DELETE | `/v1/sessions/{sessionId}/invites/{inviteId}` | Initiator | Revoke invite |
| POST | `/v1/invites/redeem` | Anyone with token | Join session; returns guest or account-bound credentials |
| PUT | `/v1/sessions/{sessionId}/participants/me/origin` | Participant | Set own starting point and mode |
| DELETE | `/v1/sessions/{sessionId}/participants/me` | Participant | Leave session |
| GET | `/v1/places/autocomplete` | Participant or Account | Proxied address autocomplete |
| POST | `/v1/sessions/{sessionId}/searches` | Participant | Run search |
| GET | `/v1/sessions/{sessionId}/searches/{searchId}` | Participant | Search results |
| POST | `/v1/sessions/{sessionId}/proposals` | Participant | Propose a place |
| POST | `/v1/sessions/{sessionId}/proposals/{proposalId}/accept` | Other participant | Accept proposal |
| DELETE | `/v1/sessions/{sessionId}/proposals/{proposalId}` | Proposer | Withdraw proposal |

Participant-scoped endpoints use `participants/me`. The API MUST NOT expose any endpoint that writes another participant's data.

### 4.3 Response shaping

| ID | Requirement |
|---|---|
| API-SHP-01 **[AC]** | `GET /v1/sessions/{id}` MUST return the caller's own starting point at full precision and the other participant's starting point only as `{ areaLabel, approxLat, approxLng }`, snapped per PRIV-03. AC: contract test asserts the other participant object contains no field with more than 3 decimal places of coordinate precision and no address field. |
| API-SHP-02 | Response schemas MUST be explicit allowlists. Serializers MUST NOT serialize persistence entities directly. |
| API-SHP-03 | Search results MUST NOT include the raw provider payload. Only fields defined in the OpenAPI schema are returned. |

---

## 5. Non-functional requirements

### 5.1 Performance

| ID | Requirement |
|---|---|
| NFR-PERF-01 | Search: p95 end-to-end latency under 4 seconds, p50 under 2 seconds, measured at the API edge for cache misses. |
| NFR-PERF-02 | All other API endpoints: p95 under 300 ms excluding upstream provider time. |
| NFR-PERF-03 | Autocomplete: p95 under 500 ms. |
| NFR-PERF-04 | App cold start to interactive: under 2 seconds on a mid-tier device defined in `ARCHITECTURE.md`. |
| NFR-PERF-05 | Upstream provider calls MUST have explicit timeouts (connect 2 s, total 5 s) and bounded retries (max 2, exponential backoff with jitter, idempotent calls only). |

### 5.2 Availability and resilience

| ID | Requirement |
|---|---|
| NFR-AV-01 | API monthly availability SLO: 99.5 percent. |
| NFR-AV-02 | If the routing provider is unavailable, search MUST fail fast with `provider_unavailable`. The API MUST NOT fall back to straight-line distance and present it as travel time. |
| NFR-AV-03 | The app MUST handle offline state: show cached session and plan read-only, queue nothing that changes location, and show a clear offline message. |

### 5.3 Cost control

| ID | Requirement |
|---|---|
| NFR-COST-01 | Maximum candidates per search sent to the travel time matrix: 50 (configurable). |
| NFR-COST-02 | Per-user and global daily budgets for provider calls MUST be enforced server-side, with alerting at 80 percent of budget. |
| NFR-COST-03 | Provider data caching MUST comply with each provider's terms of service. Allowed retention per provider is recorded in `ARCHITECTURE.md`. |

### 5.4 Accessibility

| ID | Requirement |
|---|---|
| NFR-A11Y-01 | Apps MUST meet WCAG 2.2 AA as applied to native apps (W3C guidance on applying WCAG to mobile) and `DESIGN.md` section 8. |
| NFR-A11Y-02 | Support Dynamic Type (iOS) and font scaling (Android) up to 200 percent without clipped text. |
| NFR-A11Y-03 | Every interactive element has an accessibility label. Travel time pairs are announced as a sentence per `DESIGN.md` A-4. |
| NFR-A11Y-04 | Tested with VoiceOver and TalkBack before each release. |

### 5.5 Localization

| ID | Requirement |
|---|---|
| NFR-L10N-01 | v1 ships in US English. All user-facing strings are externalized. |
| NFR-L10N-02 | Distance units follow device locale. Times display in the device time zone; the API stores and transmits UTC ISO 8601. |

---

## 6. Privacy requirements

Location is the most sensitive data Halfsies handles. These requirements apply to the apps, API, logs, analytics, and backups.

### 6.1 Principles

| ID | Requirement |
|---|---|
| PRIV-01 | Collect only what is needed to compute travel times and run the session. No location collection outside an active session. |
| PRIV-02 | No sale or sharing of personal data for advertising. No third-party ad, attribution, or data-broker SDKs in either app. |

### 6.2 Location handling

| ID | Requirement |
|---|---|
| PRIV-03 **[AC]** | The other participant's starting point MUST be snapped server-side before it is sent to any client: coordinates rounded to the center of a geohash precision 6 cell, plus a neighborhood or locality label. AC: property test over 10,000 random coordinates verifies returned approx coordinate is the geohash-6 cell center and no response field reveals the raw coordinate. |
| PRIV-04 | Precise starting point coordinates MUST be encrypted at rest with a key separate from general database encryption (application-level or column-level encryption). |
| PRIV-05 **[AC]** | Precise coordinates and addresses MUST NOT appear in application logs, traces, metrics labels, analytics events, crash reports, or error messages. AC: log-scrubbing test injects known coordinates into a full session flow and asserts zero matches across captured log and trace output. |
| PRIV-06 | Travel times that could allow trilateration of the other participant's origin are an accepted residual risk for v1 and MUST be documented in the threat model. Mitigations to evaluate: rounding `tA`/`tB` shown to the other participant to the nearest minute, limiting searches per session (SEC-RL-03). |

### 6.3 Retention

| Data | Retention |
|---|---|
| Precise starting points | Deleted when the session expires, ends, or the participant leaves |
| Snapped area label | Deleted with the session record |
| Search results and cache | 15 minutes cache; result records deleted when the session expires |
| Plan summary (place name, time) | Retained for account holders until account deletion; deleted with session for guests |
| Guest identity | Deleted 7 days after session expiry |
| Security audit logs | 1 year, no precise location |
| Backups | Rolling 30 days; deletion requests are honored when backups age out |

### 6.4 Consent and transparency

| ID | Requirement |
|---|---|
| PRIV-07 | Pre-permission explanation before any OS location, notification, or calendar prompt (`DESIGN.md` P-2). |
| PRIV-08 | The in-app "Who can see what" screen (`DESIGN.md` P-4) MUST accurately reflect API behavior. It is covered by an end-to-end test. |
| PRIV-09 | App Store privacy nutrition labels and the Google Play Data safety form MUST match actual data flows, reviewed each release. |
| PRIV-10 | Privacy policy MUST address CCPA/CPRA for California users. GDPR compliance is required before any EU launch. |
| PRIV-11 | Product analytics MUST be first-party or a vendor under a data processing agreement, MUST NOT receive location data, and MUST respect an in-app opt-out. On iOS, no tracking as defined by App Tracking Transparency. |

---

## 7. Security requirements

Baseline standards: OWASP MASVS v2 (mobile apps), OWASP ASVS 5.0 Level 2 (API), OWASP API Security Top 10 2023, RFC 9700 (OAuth 2.0 Security BCP), RFC 8252 (OAuth for native apps), NIST SP 800-63B (authentication), NIST SP 800-207 (Zero Trust principles for service-to-service calls).

### 7.1 Authentication (ASVS V6, V10; MASVS-AUTH)

| ID | Requirement |
|---|---|
| SEC-AUTH-01 | Apps MUST authenticate users via OpenID Connect Authorization Code flow with PKCE (S256), using the system browser or platform auth session (`ASWebAuthenticationSession`, Android Custom Tabs) per RFC 8252. Embedded WebViews for login are prohibited. Sign in with Apple MAY use the native `AuthenticationServices` API. |
| SEC-AUTH-02 | Implicit grant and Resource Owner Password Credentials grant MUST NOT be used (RFC 9700 sections 2.1.2, 2.4). |
| SEC-AUTH-03 | The Halfsies API issues its own access and refresh tokens after validating the IdP ID token (signature, `iss`, `aud`, `exp`, `nonce`). Apps MUST NOT send IdP tokens to the API for ongoing authorization. |
| SEC-AUTH-04 | Access tokens: lifetime at most 15 minutes, audience-restricted to the Halfsies API. |
| SEC-AUTH-05 **[AC]** | Refresh tokens MUST rotate on every use. Reuse of a previously used refresh token MUST revoke the entire token family (RFC 9700 section 4.14.2). AC: replaying a rotated refresh token returns 401 and invalidates the current token. |
| SEC-AUTH-06 | Tokens SHOULD be sender-constrained with DPoP (RFC 9449) using a non-exportable key held in Secure Enclave (iOS) or hardware-backed Android Keystore where available. If DPoP is deferred, the decision and compensating controls are recorded in `SECURITY.md`. |
| SEC-AUTH-07 | Refresh token absolute lifetime: 30 days for accounts, session lifetime for guests. |
| SEC-AUTH-08 | Sign-out MUST revoke the refresh token server-side and delete local credentials. |

### 7.2 Invite tokens and guest access

| ID | Requirement |
|---|---|
| SEC-INV-01 | Invite tokens MUST contain at least 128 bits of entropy from a CSPRNG. |
| SEC-INV-02 | The API MUST store only a hash (SHA-256) of invite tokens and compare in constant time. |
| SEC-INV-03 | Invite links MUST use Universal Links (iOS) and verified Android App Links. Custom URL schemes MUST NOT carry invite tokens, since any app can register them. |
| SEC-INV-04 | Redemption issues guest credentials scoped to exactly one session, following SEC-AUTH-04 through SEC-AUTH-06. |
| SEC-INV-05 | Invite redemption is rate limited per SEC-RL-02 and all failed redemptions are logged as security events. |
| SEC-INV-06 | The invite token MUST be removed from app navigation state and not persisted after redemption. |

### 7.3 Authorization (ASVS V8; API Top 10 API1, API3, API5)

| ID | Requirement |
|---|---|
| SEC-AZ-01 **[AC]** | Every session-scoped endpoint MUST verify the caller is a current participant of that session, enforced in a single shared authorization layer, deny by default. AC: for every endpoint in section 4.2, an automated test calls it with a valid token from a non-participant and expects 404 (to avoid confirming existence). |
| SEC-AZ-02 | Initiator-only actions (end session, manage invites) MUST check the participant role server-side. |
| SEC-AZ-03 | Guest credentials MUST NOT access any `/v1/me` account endpoint except as defined for guest upgrade. |
| SEC-AZ-04 | Authorization decisions MUST NOT rely on client-supplied role or participant fields. |

### 7.4 Input validation (ASVS V1, V2; API Top 10 API8)

| ID | Requirement |
|---|---|
| SEC-VAL-01 | All request bodies MUST be validated against the OpenAPI schema with unknown properties rejected (`additionalProperties: false`). |
| SEC-VAL-02 | Latitude in [-90, 90], longitude in [-180, 180], finite numbers only. Travel mode, category, and price are enums. |
| SEC-VAL-03 | Strings have maximum lengths: display name 50, autocomplete query 120, area label 80. Display names are Unicode-normalized (NFC) and stripped of control and bidi override characters. |
| SEC-VAL-04 | Meeting time MUST be within [now minus 5 minutes, now plus 14 days]. |
| SEC-VAL-05 | Database access MUST use parameterized queries or a query builder that parameterizes by default. |

### 7.5 Rate limiting and abuse (API Top 10 API4, API6)

| ID | Requirement |
|---|---|
| SEC-RL-01 | Per-token and per-IP rate limits on all endpoints, returning 429 with `Retry-After`. |
| SEC-RL-02 | Invite redemption: maximum 10 attempts per IP per hour and 5 failures per session before the invite is revoked. |
| SEC-RL-03 | Searches: maximum 20 per session per hour. Autocomplete: maximum 60 per minute per user. |
| SEC-RL-04 | Account creation and session creation limits per account and device to deter automated abuse. |
| SEC-RL-05 | The API SHOULD verify app integrity using App Attest (iOS) and Play Integrity API (Android) on session creation, invite redemption, and search. Failed attestation increases rate-limit strictness; it MUST NOT be the sole control. |

### 7.6 Transport (ASVS V12; MASVS-NETWORK)

| ID | Requirement |
|---|---|
| SEC-TLS-01 | All traffic uses TLS 1.2 or higher; TLS 1.3 preferred. The API domain sends HSTS. |
| SEC-TLS-02 | iOS App Transport Security MUST NOT be disabled. Android `usesCleartextTraffic` MUST be false via Network Security Config. |
| SEC-TLS-03 | Certificate pinning MAY be implemented. If implemented, it MUST pin to public keys with at least one backup pin and include a remote-configurable kill switch process documented in `SECURITY.md`. |

### 7.7 Mobile data storage (MASVS-STORAGE, MASVS-CRYPTO)

| ID | Requirement |
|---|---|
| SEC-MOB-01 | Tokens and DPoP keys MUST be stored in the iOS Keychain (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` or stricter) and Android Keystore-backed encrypted storage. Never in plain preferences, files, or logs. |
| SEC-MOB-02 | Precise starting points MUST NOT be persisted on device beyond the active session. |
| SEC-MOB-03 | Credentials and session data MUST be excluded from cloud and device backups. |
| SEC-MOB-04 | Release builds MUST disable debug logging, debuggable flags, and developer menus. |
| SEC-MOB-05 | Clipboard MUST NOT be used for tokens. Invite links are shared only through the OS share sheet. |
| SEC-MOB-06 | Deep link handlers MUST validate host, path, and parameters against an allowlist and ignore unknown parameters. |

### 7.8 Secrets and third-party APIs (API Top 10 API10)

| ID | Requirement |
|---|---|
| SEC-SEC-01 | Place, routing, and geocoding provider API keys MUST live only on the server, loaded from a secrets manager. |
| SEC-SEC-02 | The only keys permitted in the app are map SDK display keys, restricted by bundle ID and Android package name plus signing certificate fingerprint, with usage quotas set. |
| SEC-SEC-03 | Provider responses MUST be validated against expected schemas before use. Malformed responses are dropped and logged. Provider-supplied URLs (photos, websites) are validated as `https` before being returned to clients. |
| SEC-SEC-04 | Secrets MUST NOT be committed. CI runs secret scanning on every push and blocks merges on findings. |

### 7.9 Logging and monitoring (ASVS V16)

| ID | Requirement |
|---|---|
| SEC-LOG-01 | Security events are logged with timestamp, correlation ID, actor ID, and outcome: sign-in, token refresh failure, token reuse detection, invite created/redeemed/failed/revoked, authorization failures, rate-limit triggers, account deletion. |
| SEC-LOG-02 | Logs MUST comply with PRIV-05. Tokens, invite tokens, and authorization headers MUST be redacted. |
| SEC-LOG-03 | Alerts fire on refresh token reuse, spikes in authorization failures, and invite brute-force patterns. |

### 7.10 Supply chain

| ID | Requirement |
|---|---|
| SEC-SC-01 | Every third-party dependency (app and API) requires a documented review before adoption: known CVEs, maintenance activity, patch cadence, license, and transitive dependency count. Reviews are recorded in `DEPENDENCIES.md`. |
| SEC-SC-02 | Dependencies are version-pinned with lockfiles. Software composition analysis runs in CI and blocks on critical or high vulnerabilities without a documented exception. |
| SEC-SC-03 | Each release produces an SBOM (CycloneDX or SPDX) for both apps and the API. |
| SEC-SC-04 | Release builds are produced only by CI from protected branches. App signing keys are held in platform-managed signing (App Store Connect, Play App Signing) or an HSM-backed store. |

---

## 8. Data model (logical)

| Entity | Key fields | Notes |
|---|---|---|
| User | id, idpSubject, idpProvider, displayName, color, email?, createdAt | Accounts only |
| Guest | id, sessionId, displayName, createdAt | Scoped to one session |
| Session | id, initiatorUserId, status, meetingTime?, createdAt, expiresAt | status: open, planned, ended, expired |
| Invite | id, sessionId, tokenHash, expiresAt, redeemedAt?, revokedAt? | |
| Participant | id, sessionId, userId or guestId, role (A or B), mode, originEncrypted?, areaLabel?, joinedAt | origin encrypted per PRIV-04 |
| Search | id, sessionId, inputHash, filters, createdAt, expiresAt | |
| Result | searchId, placeId, name, category, priceLevel, lat, lng, tA, tB, even, score, rank | Provider-cacheable fields only |
| Proposal | id, sessionId, placeId, proposedBy, status, createdAt | |
| Plan | sessionId, placeId, placeName, meetingTime, confirmedAt | |
| Device | id, participantOrUserId, platform, pushToken, createdAt | |
| RefreshToken | id, family, subjectId, tokenHash, dpopJkt?, expiresAt, usedAt?, revokedAt? | |

---

## 9. Integrations

| ID | Requirement |
|---|---|
| INT-01 | Routing provider: MUST support travel time matrices for drive, transit, walk, and bike with departure time. Selection recorded in `ARCHITECTURE.md`. Candidates: Google Routes API, Mapbox Matrix API, self-hosted Valhalla or OSRM (transit requires GTFS-capable engine such as OpenTripPlanner). |
| INT-02 | Places provider: MUST support category search, opening hours, and price level. Candidates: Google Places API, Foursquare Places API. |
| INT-03 | Provider access MUST be behind internal interfaces so providers can be swapped without API contract changes. |
| INT-04 | Push: APNs (token-based auth) and Firebase Cloud Messaging HTTP v1. |
| INT-05 | Identity: Sign in with Apple, Google Identity (OIDC). |
| INT-06 | The only web-served content is `/.well-known/apple-app-site-association`, `/.well-known/assetlinks.json`, the privacy policy, terms, and a static invite fallback page linking to app stores. The fallback page MUST NOT read or render the invite token. |

---

## 10. Quality gates

| ID | Requirement |
|---|---|
| QA-01 | Unit test line coverage at least 80 percent for API and for app business logic (view models, domain, networking). UI code is covered by UI tests on critical flows. |
| QA-02 | Every **[AC]** requirement has at least one automated test referencing the requirement ID in its name or annotation. |
| QA-03 | Search and ranking logic has deterministic fixture tests using recorded provider responses. No live provider calls in unit tests. |
| QA-04 | API contract tests validate every endpoint against the OpenAPI document in CI. |
| QA-05 | CI pipeline: lint, unit tests, contract tests, SAST, SCA, secret scanning, SBOM generation. Any failure blocks merge. |
| QA-06 | DAST against a staging API before each release, including BOLA tests from SEC-AZ-01. |
| QA-07 | Mobile security testing against MASVS v2 using the OWASP MASTG before first public release and after major changes. |
| QA-08 | All AI-generated code passes the same gates and receives human review before merge. |
| QA-09 | Threat model (STRIDE or equivalent) completed before implementation of sessions and invites, and updated when data flows change. |

---

## 11. Observability

| ID | Requirement |
|---|---|
| OBS-01 | Distributed tracing across API and provider calls with correlation IDs returned in a response header. |
| OBS-02 | Metrics: search latency, provider latency and error rate, cache hit rate, provider spend per day, even-trip rate per search, empty-result rate by reason. |
| OBS-03 | Crash reporting in both apps with PII and location scrubbing verified by PRIV-05 tests. |

---

## 12. Delivery milestones

Each milestone is broken into small, single-responsibility tasks in the issue tracker. Each task references requirement IDs.

| Milestone | Scope |
|---|---|
| M1 | OpenAPI contract, auth (SEC-AUTH), accounts (FR-ACC-01, 04, 05), CI quality gates |
| M2 | Sessions and invites (FR-SES, SEC-INV, SEC-AZ), deep links |
| M3 | Starting points and autocomplete (FR-ORG), privacy snapping (PRIV-03 to PRIV-05) |
| M4 | Search and ranking (FR-SRCH), provider integrations, cost controls |
| M5 | Results, proposals, plans (FR-RES), push notifications (FR-NOT) |
| M6 | Accessibility pass, MASTG testing, DAST, store submission |

---

## 13. Open decisions

| ID | Decision | Owner | Needed by |
|---|---|---|---|
| OD-01 | Native (Swift/Kotlin) versus a single cross-platform framework | Engineering | M1 |
| OD-02 | Routing provider and whether transit is in v1 or v1.1 given cost | Product, Engineering | M4 |
| OD-03 | Places provider and caching limits under its terms | Engineering, Legal | M4 |
| OD-04 | DPoP in v1 or deferred with compensating controls (SEC-AUTH-06) | Security | M1 |
| OD-05 | Final fairness constants (`FAIRNESS_RATIO`, `FAIRNESS_FLOOR_SECONDS`, `W`) after user testing | Product | M4 |
| OD-06 | Rounding of `tA`/`tB` shown to the other participant to reduce trilateration risk (PRIV-06) | Security, Product | M4 |
| OD-07 | Hosting region and cloud provider | Engineering | M1 |
