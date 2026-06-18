# Phased Plan and Sequencing — CI-7194

> **Status:** DRAFT | **Owner:** Clarissa Audrey | **Jira:** [CI-7194](https://hootsuite.atlassian.net/browse/CI-7194) | **Last updated:** 2026-06-18

Companion to [`design-doc.md`](./design-doc.md). Frontend companion: [`frontend-testing.md`](./frontend-testing.md).

> **Read also:** [Hootsuite services — integration and contract test inventory (CI-7194)](https://hootsuite.atlassian.net/wiki/spaces/CI/pages/13967753294) — patterns we are borrowing (SCUM subproject, FunctionOverrider) and avoiding (shared dev RDS, ignore-and-continue gates).

# Phase 1 — Make `channel-auth-tool` agent-driveable (foundation)

This unblocks everything else. Today the tool cannot be the harness because it cannot run unattended.

* **Refactor to a library**: add `pyproject.toml`, package as `channel_auth_tool`, make `services/` and `creators/` importable modules. Keep `create-profile.py` / `refresh-profile-token.py` as thin CLI shims.
* **Abstract the OAuth callback**: replace `input()` in `create-profile.py` and `atprotooauthclient.py` with an `OAuthCallbackHandler` interface. CLI uses a stdin handler; the harness uses a local HTTP listener on `127.0.0.1:8080` that captures the redirect automatically.
* **Add `channel-auth` as a target service** (per **D1**): add `services/channel_auth.py` alongside the existing `services/core.py` and `services/som.py`. Scenarios drive `channel-auth`'s `/v1/oauth/...` endpoints to exercise OAuth orchestration, then read Core/SOM directly to assert the encrypted token and profile state landed correctly. Keep the existing direct Core/SOM paths — they are the verification surface, not legacy.
* **Pytest unit suite** (Tier 1 for the tool itself): mock HTTP with the `responses` library; one test file per `services/*.py` and `creators/*.py`.
* **Fix latent bugs surfaced during the audit**: `GoogleBusinessProfileTokenRefresher` uses `YoutubeChannelClient` (likely wrong); `.env` has `${self.client_id}` literals never substituted; Threads is a stub returning `"mock"`.
* **Add `agents.Makefile`** matching the Scala/Go services' `slow-verify` contract.

# Phase 2 — Add a real integration tier in each backend service

Each service today has Tier 1 only (with `service-core`'s DAO tests as a non-hermetic outlier and `service-social-profiles` / SNEI as zero-coverage outliers). Introduce a clean Tier 2 that satisfies **D2**, **D3**, and **D4** simultaneously, and add a small Tier 1 mock-fidelity sub-suite.

## Common shape across all five backend services

Every Tier 2 suite has these four test classes (named differently per language):

1. **DataLayerIntegrationTests** — DAO/repository tests against Testcontainers MySQL, with Flyway/equivalent migrations applied per run. (N/A for `service-social-profiles`, which is Redis-only — see its subsection.)
2. **InfraIntegrationTests** — Redis, kafka-bridge, and other infra clients against Testcontainers.
3. **HttpIntegrationTests** — bind the real production HTTP server (Chi/http4s/Pekko-HTTP) on an ephemeral port; tests issue real HTTP requests; partner clients and Core/SOM are stubbed via WireMock.
4. **OpenApiConformanceTests** (per **D3**) — load the published OpenAPI spec, drive the bound HTTP server with Specmatic/hootpact, fail on any spec/code divergence.

Plus a Tier 1 sub-suite, sitting alongside unit tests:

5. **MockFidelityTests (Tier 1, new)** — for each upstream we mock (Core, SOM, partner networks where the spec is public), load the upstream's published OpenAPI and replay our mock responses through Specmatic. Failure means our mock has drifted from the upstream contract. Fast (< 5s per upstream), blocks PR. Estimated 0.5 day per upstream per repo. **This is the consumer-side cell of the integration-vs-contract matrix** — see the matrix in [`design-doc.md`](./design-doc.md).

## channel-auth (Go)

* Introduce build tags. Existing `_test.go` files stay tag-free; new `_integration_test.go` files use `//go:build integration`.
* Replace `localhost:3306` and `localhost:6379` in `internal/authenticator/bluesky/bluekskyauthdb/db_test.go` and `internal/refreshlock/refreshlock_test.go` with Testcontainers. Kill the silent Redis skip.
* Add `make test-integration` (`go test -tags=integration ./...`) and run it in Jenkins `unitTest` after `make up`.
* Add `HttpIntegrationTests` that boots the generated Chi router from `internal/gen` on `httptest.NewServer`. WireMock containers stub partner APIs and Core/SOM. Currently zero HTTP-layer coverage.
* Add `OpenApiConformanceTests`: load `schema/channel-auth/openapi.yaml`, drive the bound server with Specmatic, fail on drift.
* Add `MockFidelityTests`: validate fakes/stubs of Core's `/v1/socialIntegrationAccounts` and SOM's `/socialProfiles/{id}` against their published OpenAPI specs in `IDL-central`.
* `agents.Makefile` `slow-verify` becomes `lint + test (Tier 1) + test-integration (Tier 2) + secscan`.

## social-network-auth (Scala)

* Populate the vestigial `IntegrationTest` config. Add `src/it/scala` and a `service/it:test` sbt target. Add `make test-it` to the Makefile.
* Per **D2**: rewrite endpoint-orchestration tests in `src/it` against a real http4s server bound to an ephemeral port. The Tapir stub interpreter pattern in `OAuthEndpointsTest.scala` stays in `src/test` for Tier 1 only.
* Per **D3**: add `OpenApiConformanceTest` in `src/it` that drives the bound server via Specmatic against `docs/openapi.yaml`. This is in addition to the existing post-deploy hootpact suite — the conformance test runs at PR time, hootpact runs after deploy.
* Add `MockFidelityTests` in `src/test` (Tier 1): validate the fake Core SDK client against `service-core`'s OpenAPI in IDL-central, and the fake SOM client against SOM's OpenAPI. Cheaper than Tier 2 and catches mock drift earlier.
* **Parallelism investigation** (per **D5**): before flipping `Test / parallelExecution`, audit shared state in test fixtures. Suspects: `mockito-scala` global state, shared `BaseSocialNetworkAuthSpec` mutable fields, cross-suite log-buffer interleaving. Fix mutable fixtures first; only then enable parallel for `IntegrationTest`. Document findings in the migration PR.

## service-core (Scala)

> **Highest-stakes change in the entire plan:** replace the shared dev RDS connection in `service/src/test/resources/reference.conf` with Testcontainers MySQL. Today every developer's tests fight over the same database — the single biggest agentic-first blocker because no agent can verify a `service-core` change in isolation.

* Move DAO tests from `service/src/test` into `src/it/scala`; revive the declared-but-unused `IntegrationTest` config in `project/Settings.scala`. Run via `sbt service/it:test`.
* Per **D2**: replace the in-process `EndpointTestHelper.executeTestRequest` injector with a real Pekko-HTTP listener bound to an ephemeral port for Tier 2 endpoint tests. The cake-pattern composition stays — only the request-injection swaps.
* Per **D3**: add an `OpenApiConformanceTest` driving the bound server. **Open question:** service-core does not currently publish an OpenAPI spec for its VIP endpoints — generating one is a prerequisite. If the team rejects spec-first for Core, fall back to a hootpact-style example-driven suite at this tier.
* Add `MockFidelityTests` for the service-crypto client. service-crypto publishes an OpenAPI spec; our circuit-breaker-wrapped fake should be validated against it. This is the most leveraged mock-fidelity test in the plan because the re-encryption pipeline currently has zero coverage.
* **Add reencryption-pipeline coverage** — `ReEncryptionPipeline` and the writable `dbReEncrypt` pool currently have **zero** tests. With Testcontainers in place this becomes feasible: integration tests walk rows through `social_network_reencryption` against a real DB with service-crypto mocked.
* **Parallelism investigation** (per **D5**): same drill as social-network-auth, plus a specific suspect — the shared-RDS legacy means tests probably rely on each other's row IDs. Untangle that before parallelizing.

## service-social-profiles (Scala)

> **Hygiene blocker:** `Makefile:73` defines `it: ## no-op` while `README.md:75-78` advertises `make it` as if it ran integration tests. Either revive (this phase) or delete in a same-PR hygiene fix. The current state is worse than absent — the contract is actively misleading and silently passes in CI.

Revive the empty `integration/` subproject. Four classes:

1. **HttpIntegrationTests** — bind real Tapir+http4s on an ephemeral port; drive via sttp client; mock downstreams with WireMock containers. No Testcontainers DB needed — the service is Redis-only.
2. **InfraIntegrationTests (Redis)** — Testcontainers Redis (`redis:7-alpine`); validate cache-invalidation and locks for real instead of via mocked `RedisCommands`.
3. **EventBusIntegrationTests** — POST a real protobuf `EventList` containing `SocialProfileDisconnectDiscovered` to `/event-bus/event`; assert the disconnect processor's downstream calls (mocked SOM, mocked SNA) fire correctly. **First end-to-end coverage of the disconnect ingress path.**
4. **OpenApiConformanceTests** — Tapir publishes its OpenAPI at runtime; pin a snapshot of the spec to disk and Specmatic-conform against the bound server. Per **D3**.

Plus the Tier 1 mock-fidelity wins:

* **MockFidelityTests** against the six vendored OpenAPI specs in `src/main/resources/specs/` (`core.yaml`, `sna.yaml`, `organization.yaml`, `memberfavourites.yaml`, `pushNotification.yaml`, `socialCommunication.yaml`). The specs are already pinned for client codegen, so adding Specmatic validation on top is high-leverage and cheap (~3 days).
* **Add tests for `EventHandler` and `/events`**: today the disconnect processor is well-tested but the Kafka HTTP ingress path itself (`EventHandler`, `EventsApi`) has zero coverage.

Replace the dead `make it*` scaffolding so the Make targets and Jenkinsfile actually do what their names imply. Estimated **2 weeks**.

## social-network-error-inspection / SNEI (Scala)

The biggest land-grab in Phase 2. Empty `src/it/scala` becomes:

1. **DataLayerIntegrationTests** — Testcontainers MySQL with Flyway migrations (`Makefile.db.mk` already drives Flyway). DAO layer currently mocked or fake-faked; this gives real coverage.
2. **HttpIntegrationTests** — real Tapir+http4s Blaze server on an ephemeral port. Tests both internal endpoints and the **public Aperture-fronted endpoints** that today are only exercised by staging Wasabi.
3. **EventBusIntegrationTests** — POST `EventList` to `/event-bus/event`; assert the disconnect parser writes the expected row to MySQL. End-to-end Kafka-bridge to DB coverage.
4. **OpenApiConformanceTests** — **prerequisite work:** SNEI does not publish OpenAPI today (Tapir generates Swagger at runtime). Either generate the spec to a checked-in file as part of the build, or fall back to hootpact-style example-driven tests. **See Open Question OQ-S1.**

Plus the Tier 1 mock-fidelity wins:

* **MockFidelityTests** for the Core, social-network-auth, and Organization client stubs against their published OpenAPI specs in IDL-central. ~2 days.
* **Add `DefaultOrganizationService` tests**: the org client has zero direct unit tests today. The `LoggingTopsAuthorizationService` is mocked at the route layer but the underlying client is untested.

Replace the dead `make it*` scaffolding (the `it-staging` Jenkinsfile invocation runs an empty sbt IT plus Wasabi today — same misleading-contract problem as service-social-profiles). Estimated **2–3 weeks**.

## Critical-path ordering inside Phase 2

For agents in particular, the order matters more than the parallelism:

1. **`service-core` RDS to Testcontainers** must land first. Until it does, no agent (or human) can run service-core tests in isolation.
2. **Real HTTP server binding** (D2) is a prerequisite for OpenAPI conformance (D3). Land in that order per service.
3. **Mock-fidelity sub-suite** can land in parallel with Tier 2 work; it is independent of the HTTP-binding migration.
4. **Parallelism investigation** (D5) is the last step in each Scala service's Phase 2 — it is an outcome, not an input. Defer until the suite is green and hermetic.
5. **`service-social-profiles` and SNEI** can land in parallel with the OAuth-trio work — they share Testcontainers/WireMock patterns but no source code with the OAuth services.

# Phase 3 — Build the cross-repo harness in `channel-auth-tool`

The agent-runnable verification surface this SPIKE is really about. Per **D7**, this phase includes the disconnect-event contract end-to-end — the harness is not just OAuth.

## Compose stack

`docker-compose.harness.yml` at the root of `channel-auth-tool`:

* Brings up `channel-auth` + `social-network-auth` + `service-core` + `dashboard-som` + MySQL + Redis from their respective image tags.
* WireMock containers per partner API (Bluesky, Reddit, TikTok, Pinterest, Google, Meta).
* **Kafka (or Redpanda — lighter)** plus the **kafka-bridge** sidecar.
* `service-social-profiles` and SNEI as containers subscribed to `hs.app.social_profile.social_profile_disconnect_discovered`.

Adding Kafka to the harness Compose is the single biggest scope item in Phase 3 — but per **D7** it stays inside the base scope rather than being deferred, because the disconnect contract is the highest-value cross-service integration the team owns.

## WireMock fixture library

Pre-recorded responses for each partner OAuth + profile flow, checked in under `channel-auth-tool/fixtures/`. Recorded once via VCR-style capture against staging; hand-edited where partner data is sensitive.

## Scenario directory

`scenarios/` directory: pytest scenarios using `channel-auth-tool`'s library modules. Two markers:

* `@pytest.mark.harness` — runs against the local Compose stack with WireMock partners and the in-stack Kafka. Hermetic. **This is what an agent runs to verify a change.**
* `@pytest.mark.dev` — same scenarios, run against deployed dev with real partners. Run by humans / nightly cron, not by agents-in-loop.

## Scenario coverage (minimum)

* One create-profile + one refresh-token scenario per supported network. Bluesky's ATProto flow gets its own pair because it bypasses Core.
* **Disconnect-event scenarios** (per **D7**):
  1. Run the create-profile flow through `channel-auth`.
  2. Trigger a token refresh failure that emits `social_profile_disconnect_discovered`.
  3. Assert SNEI's MySQL has a parsed disconnect row.
  4. Assert `service-social-profiles`' Redis cache reflects the disconnect.

## One command

`make harness` boots Compose, waits for health (including Kafka and the consumer subscriptions), runs `pytest -m harness scenarios/`, tears down.

# Phase 4 — Harmonize post-deploy smoke (Tier 4)

## Decision: provider-published over consumer-driven (D6)

We start with **provider-published OpenAPI contracts**, not consumer-driven contracts (CDC).

|  | Provider-published (chosen) | Consumer-driven (deferred) |
| --- | --- | --- |
| Source of truth | OpenAPI in IDL-central / `schema/` | Contracts hosted by each consumer |
| Adoption cost | Per-repo, can land in parallel | Cross-team coordination required |
| Drift detection | Tier 2 conformance suite (D3) | Provider runs each consumer's contracts against itself |
| Hootsuite precedent | None at PR time (gap our plan addresses) | SCUM's `consumer-contract-test-staging-service-mobile-api` (see [services inventory](https://hootsuite.atlassian.net/wiki/spaces/CI/pages/13967753294)) |

**Migration trigger:** revisit consumer-driven once at least two of `social-network-proxy`, `social-communication`, or `dashboard` express interest in owning their channel-auth/core/social-network-auth contracts. Until then, the spec wins; consumers consume.

## Phase 4 work items per service

* **`channel-auth`** has no contract tests today. Add hootpact (or specmatic) contracts under `channel-auth/hootpact_service/` covering `/v1/oauth/{network}/...` for each supported network. Wire into Jenkinsfile `postDeployDev`.
* **`social-network-auth`** hootpact only covers Facebook token-status today. Extend `hootpact_service/hootpact-service.openapi.yaml` and `hootpact-service_examples/` to cover Pinterest, TikTok, Reddit, Bluesky.
* **`service-core`** uses external Amplify Wasabi today. Either migrate to hootpact for consistency, or document the divergence; either way, drive the reencryption pipeline endpoints in the contract.
* **`service-social-profiles`** has nothing today. Add hootpact contracts for the Tapir API. Wire into a new `postDeployDev` closure in the Jenkinsfile (the file has neither postDeployDev nor postDeployStaging today). The vendored consumer specs in `src/main/resources/specs/` cover the inverse direction; provider hootpact is independent.
* **SNEI** keeps the existing Wasabi staging suite for the Aperture/JWT/CSRF dashboard-facing path — it's the only thing exercising those middlewares today and has six real cases. **Add hootpact** for the internal API to give post-dev signal (currently `postDeployDev` is empty; only `postDeployStaging` runs `make it-staging`, which itself runs an empty sbt IT plus Wasabi). Two-gate ladder: hootpact at dev (fast feedback), Wasabi at staging (Aperture authenticity).
* The `it-dev` / `it-staging` / `it-production` `.PHONY` placeholders in `social-network-auth/Makefile` should be implemented or removed — they are listed but not defined today.

# Phase 5 — Agent runbook and governance

* Document in each repo's `AGENTS.md` (and at the root of `channel-auth-tool`):
  * "To verify your change: run `make test` (Tier 1, includes mock-fidelity), `make test-integration` (Tier 2), and — for cross-service changes — `cd channel-auth-tool && make harness` (Tier 3)."
  * **Hard rule: agents never run Tier 4.** Real-network smoke is post-deploy on real infra, not an agent verification surface.
* Add a single source-of-truth `TESTING.md` in `channel-auth-tool` (the harness home) explaining the four tiers, where each lives, and which one to add a new test to.

# Sequencing and investment estimate

| Phase | Estimate | Notes |
| --- | --- | --- |
| **1** Tool refactor | ~1–2 weeks for one engineer | Unlock for Phase 3. Without it, Phase 3 is blocked. |
| **2** Per-repo Tier 2 + Tier 1 mock fidelity | ~2 weeks per service, parallelizable | Real-HTTP migration + OpenAPI conformance + mock-fidelity sub-suite + parallelism investigation. `service-core`'s RDS migration is the riskiest single task. SNEI's OpenAPI gap (OQ-S1) is the riskiest unknown. Mock-fidelity adds ~0.5 day per upstream per repo. |
| **3** Cross-repo harness (incl. Kafka + the two consumers per D7) | ~3–4 weeks, single chunk | Delivers "agent verifies its own change" plus end-to-end disconnect-event coverage. Adding Kafka is the biggest single scope item but is in base scope per **D7**. |
| **4** Post-deploy smoke | Incremental | Add networks one at a time as they break in staging. |
| **5** Governance | Ongoing, lightweight | Documentation tax. |

**If leadership wants a single deliverable that answers "does agentic development work safely on these repos":** ship Phases 1 and 3. Phase 2 is hygiene that pays off over months but is essential for D3 (OpenAPI conformance at PR time). Phase 4 is the safety net for partner API drift.

# Open questions

- [ ] **OQ-1:** Does `service-core` generate an OpenAPI spec for D3 (Tier 2 conformance), or does it fall back to hootpact-style examples? Decision needed before Phase 2 can finish for Core.
- [ ] **OQ-2:** Owner / staffing for Phase 1 (channel-auth-tool refactor). It is the unlock and likely the smallest single piece — who picks it up?
- [ ] **OQ-3:** Image-tag strategy for Phase 3 Compose stack: dev image-tags (latest staging), pinned versions, or per-PR builds? Affects how trustworthy the harness is across branches.
- [ ] **OQ-4:** WireMock fixture refresh cadence: when do partner-API recordings go stale, and who refreshes them? Likely a Phase 4 trigger — if Tier 4 catches a regression, refresh Tier 3 fixtures.
- [ ] **OQ-5:** Mock-fidelity scope: do we validate every mocked upstream, or only those whose specs change frequently? Default proposal: every upstream with a published OpenAPI in IDL-central; revisit if maintenance cost outweighs value.
- [ ] **OQ-6 — Cron / scheduled test triggers (backend-platform clarification needed).** `gitOpsPipeline.cron = 'H H * * H'` is set in all five backend Jenkinsfiles. The design doc treats this as a candidate Tier 1/2/4 cadence trigger but does not commit. Owner needs to confirm with backend-platform: (a) is `gitOpsPipeline.cron` a supported testing primitive (versus an undocumented carryover), (b) is there a separate standalone test runner planned (e.g., a Kubernetes CronJob primitive, a `gitOpsPipeline.scheduledTest` stage), (c) if neither, what is the supported way to detect partner-API drift between deploys, given that hootpact today only fires on `postDeployDev` and will not catch a partner regression until something redeploys, (d) **specifically for `service-social-profiles`**, the weekly cron rerun combined with `deployProduction = true` auto-deploys to prod weekly with no code change — this combination needs deliberate sign-off if the cron stays. Cross-referenced from the [design doc Triggers and cadence section](./design-doc.md#triggers-and-cadence).
- [ ] **OQ-S1 — SNEI does not publish OpenAPI.** Tapir generates Swagger at runtime but nothing is checked in to IDL-central. D3 (Tier 2 OpenAPI conformance) requires a checked-in spec. Generate an OpenAPI spec from Tapir to a checked-in file as part of `make build`, or fall back to hootpact-style example-driven tests at Tier 2? Decision needed before SNEI's Phase 2 can finish.
- [ ] **OQ-S2 — `service-social-profiles` `make it` no-op.** `README.md:75-78` references `make it`; `Makefile:73` defines `it: ## no-op`. The `it-dev` / `it-staging` / `it-production` `.PHONY` lines are declared but not implemented. Phase 2 fixes this. Until then, decide: revive (Phase 2) or delete in a hygiene PR? Worse than absent — the contract is actively misleading.
- [ ] **OQ-S3 — SNEI's `postDeployStaging` runs an empty sbt IT plus Wasabi.** `Jenkinsfile:postDeployStaging` runs `make it-staging`, which is `sbt it:test` (zero tests) plus the Wasabi suite. The empty sbt IT is harmless but creates the same misleading-contract problem as `service-social-profiles`' `make it`. Phase 2 work fixes it; until then, a one-line comment in the Makefile or Jenkinsfile would clarify intent.

# Out of scope

* Performance/load testing (separate effort).
* Security testing beyond existing Snyk pre-PR gates.
* Test infrastructure for repos outside the team's backend channel-auth ecosystem (`token-refresher-scheduler`, `channel-auth-audit`, etc. — add via follow-up if relevant).
* Migrating any service away from Disqlose, VIP server, the Hootsuite cake-pattern DI, or other deprecated-but-pinned legacy frameworks. Tier 2 wraps these as-is.
* Migrating SNEI off ScalikeJDBC or migrating `service-social-profiles` to use MySQL.
* Consumer-driven contracts (deferred per **D6**); revisit when consumer teams want to own their expectations.
* **Frontend** (`channel-integrations/` FE in `fe-global`) — separate companion in [`frontend-testing.md`](./frontend-testing.md), with its own toolchain (jest + MSW, Storybook, optional Playwright), its own ownership-confirmation gate, and its own cadence story.
