# COST-8520 External OIDC TDD Test Plan

> **For agentic workers:** Implement test-first. For each behavior, write the failing test, run it to confirm the expected failure, implement the smallest change, then rerun the test and relevant suite. Do not implement production code before observing the test fail.

**Goal:** Define the red-first test coverage for replacing the required Keycloak-specific authentication integration with external OIDC while preserving Insights RBAC and Metrics Operator uploads.

**Architecture:** `koku-service-operator` configures the generic OIDC UI proxy and Envoy JWT validation/claim mapping. Envoy preserves the existing `X-Rh-Identity` contract and maps the provider’s organization-admin app-role claim to `is_org_admin`, which continues to activate the full Insights RBAC admin-default bundle. `koku-metrics-operator` gains a configurable OAuth scope for its client-credentials flow. SCIM/principal provisioning is out of scope for COST-8520 and must be tracked separately if pre-provisioning parity is required.

**Tech Stack:** Go, controller-runtime, Ginkgo/Gomega, Envoy YAML, oauth2-proxy, Kubernetes CRDs; Go/Ginkgo for the Metrics Operator token client; existing Koku/Insights RBAC integration tests for identity and permission behavior.

**Spec:** `docs/design/COST-8520-external-oidc-design.md`

## Global Constraints

- COST-8520 changes external authentication only; Insights RBAC remains the authorization engine.
- Preserve the full existing admin-default permission bundle: Cost Management, User Access/RBAC, Sources, and currently seeded platform admin roles.
- `ENHANCED_ORG_ADMIN` remains false; admin identity must not bypass Insights RBAC.
- Decision: Envoy validates a Cost Management API **access token** for UI API requests, matching the token category used at the Hybrid Cloud Console API boundary. Migrate the current Keycloak UI path to this contract; its observed ID-token behavior is a baseline to characterize, not a mode to preserve.
- Backend services continue receiving the existing `X-Rh-Identity` contract; caller-provided identity headers are not trusted.
- Entra `tid` maps to `org_id` with hyphens stripped (32 hex chars), only under the one-Entra-tenant-to-one-Cost-Management-org assumption; `account_number` is mapped independently.
- No CRD backwards compatibility: `spec.auth.keycloak` is deleted; Keycloak is tested as a generic OIDC profile. Old CRs must be rejected with a message naming `spec.auth.oidc`.
- SCIM/Graph principal provisioning is not implemented or tested in this plan. Integration tests seed or provision RBAC Principals through an already-supported test/setup path.
- Full-stack support is claimed only for providers that pass user login, API access-token audience/claim, admin-role, and Metrics Operator token/upload checks.

## Current Keycloak Baseline (SNO, read-only check)

- The live SNO deployment is installed from `cost-onprem-chart` (deprecated; baseline evidence only, never an edit target). Its `cost-onprem-ui` oauth2-proxy is v7.6.0 and runs `--provider=keycloak-oidc` plus `--pass-authorization-header`; it does **not** set `--pass-access-token`.
- oauth2-proxy v7.6.0 documents `--pass-authorization-header` as forwarding the OIDC **ID token** in `Authorization`; `--pass-access-token` forwards the OAuth access token separately in `X-Forwarded-Access-Token`.
- The live Envoy JWT filter reads the default bearer `Authorization` header and accepts audiences including `cost-management-ui`, `cost-management-operator`, and `account`.
- A controlled direct-grant comparison using the UI client found that both returned tokens had `org_id`, `account_number`, and the `org-admin` role; both separately received HTTP 200 from the read-only Koku `user-access` endpoint. Only claim summaries and status codes were retained; raw tokens were not printed or saved. This was **not** a browser authorization-code capture, so the proxy flag semantics plus a browser/E2E check should confirm the exact live header.
- The Keycloak install guide says Envoy reads the access token and its protocol-mapper instructions set the required claims on access tokens. This conflicts with the oauth2-proxy flag’s documented ID-token behavior. The decision is to migrate the UI path to API access tokens; tests must make that change explicit rather than assume the current flag already does it.
- The live Metrics Operator CR uses `type: service-account` and the Keycloak token endpoint; its most recent status was `202 Accepted`. It also has `validate_cert: false`, which is a separate TLS-hardening concern to address by configuring trusted CA validation.

## Review Focus

1. **Baseline and migration:** the live SNO args use `--pass-authorization-header`; oauth2-proxy v7.6.0 documents that as forwarding the OIDC ID token, while the Keycloak install guide says Envoy reads the access token. The live ID and access token pair were both accepted by a read-only Koku endpoint. Preserve this as current-state evidence, then migrate the UI-to-API contract to an API-audience access token; do not retain ID-token acceptance as a compatibility path.
2. UI proxy forwards an API-audience access token through nginx to Envoy; an ID token with the UI client audience must be rejected by the API gateway in the proposed target design.
3. Provider-specific claim shapes must map to the same stable identity and fail closed when required org/account claims are absent or invalid.
4. The Entra admin role must map to the full RBAC admin-default bundle; non-admin roles must remain scoped and must not inherit admin access.
5. Metrics Operator service tokens use the Cost Management API audience and configured scope; token identity must not accidentally gain user-admin privileges.
6. No backwards compatibility: old Keycloak-field CRs are rejected explicitly, Keycloak works through the generic OIDC profile, and external OIDC operation must not require Keycloak resources or APIs.

---

## Test-First Workstreams

### Workstream A: Generic OIDC CRD and configuration validation (`koku-service-operator`)

**Likely files:**
- Test: `api/v1alpha1/costmanagementserviceconfig_webhook_test.go` (or a focused auth validation test file)
- Modify: `api/v1alpha1/costmanagementserviceconfig_types.go`
- Generated: CRD/deepcopy/bundle trees per `AGENTS.md`

**Tests to write first:**

- Valid external OIDC settings accept issuer URL, JWKS URI, non-empty audience list, TLS settings, and claim mappings.
- Invalid issuer scheme, malformed URLs, empty audience list, control characters/newlines, and missing required fields are rejected by CR/webhook validation.
- A missing or malformed admin claim mapping does not silently make users administrators.
- A CR still using the deleted `spec.auth.keycloak` field is rejected with a message naming `spec.auth.oidc` (WHY: proves the break is explicit, not silent).
- A Keycloak realm expressed as a generic OIDC profile (Keycloak issuer/JWKS/audiences, `realm_access.roles` claim path) validates with zero special-case code (WHY: proves Keycloak stays supported through the generic fields).

**Red/green verification:** run the focused validation test and confirm the new-schema cases fail before implementation. After implementation, run the focused test and then the operator suite. If CRD markers change, run `make generate manifests` and `make bundle`; verify generated trees rather than hand-editing them.

### Workstream B: Envoy JWT extraction and identity mapping (`koku-service-operator`)

**Likely files:**
- Test: `internal/resources/envoy_test.go`
- Modify: `internal/resources/envoy.go`

**Tests to write first:**

- Generated Envoy YAML parses and contains configured issuer, JWKS URI, audiences, TLS validation, and accepted token sources: UI access-token header and Metrics Operator bearer `Authorization` header.
- Entra fixture token with `roles` containing the configured admin role maps to `is_org_admin=true`; missing, unrelated, wrong-type, or case-variant role claims map to false (WHY exact match: prefix/case-insensitive matching would let crafted role strings escalate to admin).
- Configured tenant claim maps to `org_id` with hyphens stripped (fixture `tid` `a1b2c3d4-e5f6-…` → `org_id` `a1b2c3d4e5f6…`); configured account mapping supplies `account_number`; Entra `oid` maps to the RBAC identity key while display email/username remains separate.
- Missing/invalid tenant or account data fails closed with a 401 and does not use a shared/default tenant.
- Envoy replaces an incoming forged `X-Rh-Identity` header with a header built from the validated token and configured mappings.
- The backend request does NOT contain `X-Bearer-Token` (WHY: proves the forwarding removal; update the old `envoy_test.go` assertion rather than deleting it silently).
- A UI ID token with `aud` equal only to the UI OAuth client is rejected for API access; a valid API access token with the configured API audience is accepted.
- Invalid signature, wrong issuer, wrong audience, expired/not-yet-valid token, missing token, and JWKS fetch failures have explicit expected outcomes.
- Existing admin mapping preserves the full default role bundle through Insights RBAC; no new claims or headers bypass RBAC.

**Test method:** use a local deterministic OIDC fixture with generated signing keys and a mock discovery/JWKS endpoint. Round-trip generated YAML through the repository’s YAML parser; string-only assertions are insufficient for Envoy config.

### Workstream C: UI OAuth proxy and nginx access-token path (`koku-service-operator`)

**Likely files:**
- Test: `internal/resources/ui_test.go`
- Modify: `internal/resources/ui.go`

**Tests to write first:**

- Add a token-forwarding characterization test for the current SNO Keycloak UI deployment: it includes `--pass-authorization-header`, omits `--pass-access-token`, nginx does not explicitly forward an access-token header, and Envoy extracts JWTs from `Authorization`. Compare this with oauth2-proxy v7.6.0 flag semantics and `docs/install/keycloak.md`, which currently describes an access token. The live SNO chart is a baseline; implementation changes belong in `koku-service-operator`.
- Before changing the behavior, perform a non-production token-flow check that records only decoded claim names/values (never raw tokens): determine whether the token reaching Envoy is an ID token or access token, its `aud`, and whether required `org_id`, `account_number`, and `org-admin` claims are present.
- Preserve the SNO baseline observation as a fixture/assertion: the UI client’s ID token and access token both carried required identity/admin claims and both independently received HTTP 200 at the read-only `user-access` endpoint. This direct-grant probe is not a substitute for a browser authorization-code capture.
- oauth2-proxy is configured with generic OIDC provider, issuer discovery, authorization-code/PKCE, delegated API scope, provider CA trust, secure cookies, and token refresh settings.
- oauth2-proxy has an explicit session email-claim mapping for Entra (e.g. `preferred_username`), verified against the deployed proxy version's `--help`; a login flow with a token carrying no `email` claim still builds a session (WHY: Entra rarely emits `email`, and without a mapped claim every browser login fails before any API call).
- oauth2-proxy requests/forwards the API access token (e.g. `--pass-access-token`), not only the ID token.
- nginx forwards the access-token header to Envoy without dropping or trusting a browser-supplied replacement; `/api/me` still returns the intended display username/email headers.
- logout always clears the local proxy session; optional provider end-session behavior is tested only when configured/supported.
- Missing/invalid provider CA, issuer discovery failure, and absent UI client credentials fail visibly without disabling JWT validation.

**Integration assertion:** a browser-style request with a valid API access token traverses oauth2-proxy → nginx → Envoy and receives the expected `X-Rh-Identity`; an ID token does not pass as API authentication.

**Resolved migration decision:** migrate Keycloak and external OIDC UI flows together to the API-audience JWT access-token contract. Envoy rejects ID tokens for API access, including tokens whose audience is only the UI client. The implementation plan must specify the proxy/nginx token-forwarding path and any one-time configuration migration; it must not introduce a legacy ID-token acceptance mode. This matches the Console's access-token boundary while retaining the distinct on-prem oauth2-proxy → nginx → Envoy → `X-Rh-Identity` architecture.

### Workstream D: Metrics Operator scope and token request (`koku-metrics-operator`)

**Likely files:**
- Test: `internal/crhchttp/config_test.go`
- Modify: `internal/crhchttp/config.go`, `api/v1beta1/metricsconfig_types.go`, defaults/controller wiring as needed
- Generated: CRD/deepcopy/manifests/bundle per the repository guide

**Tests to write first:**

- Default/omitted scope preserves the existing `api.console` behavior for backward compatibility.
- Custom scope is sent exactly in the `client_credentials` form body; Entra fixture uses `api://<api-resource-id>/.default`.
- Token URL remains configurable; the token request includes the configured client ID/secret and scope and does not log secrets or tokens.
- Unsupported auth types do not make a token request; token endpoint errors and malformed responses fail with useful errors and no bearer token set.
- CRD defaulting/serialization preserves existing CRs and includes the new field in generated manifests and bundle.

**Verification commands:** focused `go test` for `internal/crhchttp`; then `make fmt && make test && make verify-manifests` (and bundle generation/validation if required by the repo’s CRD workflow).

### Workstream E: Cross-component authorization and upload acceptance

**Likely suites:**
- `koku-service-operator/internal/resources/envoy_test.go`, `ui_test.go`, and existing controller/readiness tests
- `koku-metrics-operator/internal/crhchttp/config_test.go` and controller tests
- Existing Koku RBAC/middleware tests or operator E2E harness; no Koku production change is assumed

**Scenarios:**

- **Admin:** Entra app-role claim → `X-Rh-Identity.is_org_admin=true` → Insights RBAC returns the entire current admin-default bundle, including Cost Management, User Access/RBAC, and Sources capabilities.
- **Non-admin:** absence of admin role → regular Insights RBAC lookup; seeded viewer permissions allow only configured resource/verb access and denied resources remain filtered/blocked.
- **Revocation:** remove admin app-role membership, issue a fresh token, and verify admin-default access is absent after provider propagation and the effective Koku/Insights RBAC cache bounds.
- **Operator upload:** use a mock token endpoint to issue a service access token with the API audience and configured scope, send an upload through Envoy to Ingress, and verify identity/tenant/account mapping. The service identity never receives the human admin role.
- **Tenant isolation:** two tenant fixtures cannot read each other’s data; an Entra `tid` maps as `org_id` with hyphens stripped, only under the documented one-to-one mapping.
- **Failure:** wrong issuer/audience, ID token instead of API access token, invalid signature, expired token, missing org/account mapping, invalid TLS, and RBAC unavailable all fail according to the fail-closed contract.
- **Keycloak profile:** a Keycloak realm configured through the generic OIDC fields (Keycloak issuer/JWKS/audiences, `realm_access.roles` claim path, access-token mappers) passes the same login, API-audience, admin-role, and upload checks as Entra; old `spec.auth.keycloak` CRs are rejected with a message naming `spec.auth.oidc`.

## Provider Conformance Matrix for Tests

- **Entra:** required first full-stack profile: user access token for the Cost Management API, `roles` app-role claim, tenant/account mapping, and Metrics Operator client-credentials `/.default` flow.
- **Okta:** full-stack only after a custom authorization server issues the API-audience JWT and configured role claim, and the Metrics Operator grant/audience is proven. Do not assume `sub` is immutable.
- **Ping Identity:** test the exact product/profile; require API-audience JWT, stable user identity, admin claim, and client-credentials upload flow before claiming full support.
- **Google Cloud Identity / Workspace:** OIDC user login may be testable, but standard tokens do not provide the same admin-role or API client-credentials behavior. Mark login-only until a supported role signal and Metrics Operator token flow are implemented and tested.
- **SCIM/principal provisioning:** explicitly excluded from COST-8520 tests; tests use seeded RBAC Principals. A separate SCIM follow-up must test create/update/deactivate/delete and safe full-sync pruning.

## Open Decisions to Lock Before the Implementation Plan

1. Is Entra the only full-stack provider in the first release, with Okta/Ping as conformance candidates and Google login-only pending a machine-token strategy?
2. Confirm `org_id=tid` (hyphens stripped) only for one Entra tenant per Cost Management org; supply `account_number` independently per org (test value recorded in implementation plan Task 0).
3. DECIDED: Entra `oid` is the stable RBAC identity key and the corresponding `X-Rh-Identity` username mapping.
4. DECIDED: no migration — `spec.auth.keycloak` is deleted; Keycloak is configured as a generic OIDC profile. The token contract is resolved: Keycloak and external OIDC use API-audience access tokens, and ID tokens are rejected for API access.
5. DECIDED: old Keycloak CR configuration is rejected with a message naming `spec.auth.oidc`; no legacy support.
6. Confirm the Entra API audience, UI delegated scope, organization-admin app-role value, and Metrics Operator application permission/scope.
7. Confirm SCIM remains a dependent follow-up and not part of COST-8520 implementation.
