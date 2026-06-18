# Hootsuite services inventory — integration and contract testing (CI-7194)

> **Status:** DRAFT | **Owner:** Clarissa Audrey | **Jira:** [CI-7194](https://hootsuite.atlassian.net/browse/CI-7194) | **Last updated:** 2026-06-18

Companion to [`design-doc.md`](./design-doc.md) and [`phased-plan.md`](./phased-plan.md). This page inventories integration- and contract-testing patterns at Hootsuite **outside the channel-auth ecosystem**, to inform the four-tier pyramid by learning from existing investments. It is a snapshot, not a survey — we sampled three services across two languages and three teams.

# Scope

| Service | Team | Language | Why sampled |
| --- | --- | --- | --- |
| `service-social-communication` (SCUM) | sn-integration-cerberus | Scala 2.12 / sbt / Pekko (with classic Akka still wired alongside, e.g. `AkkaTestKit`) | Owns the most mature integration-tests subproject in the org; consumer-driven contract pattern |
| `social-network-proxy` (SNP) | sn-integration-cerberus | Scala 2.13 / Play 3 | Adjacent to channel-auth-ecosystem; consumes our specs; closest "Tier 1 only" peer to compare against |
| `dashboard` | backend-platform | PHP 8.3 | Long-lived consumer of channel-auth and core; PHP test conventions, FunctionOverrider hermeticity guard |

Out of scope for this snapshot (worth a follow-up): `service-crypto`, `token-refresher-scheduler`, `social-profiles`, `organization`, `event-bus` consumers.

# Comparison summary

| Dimension | SCUM | SNP | dashboard |
| --- | --- | --- | --- |
| Language / framework | Scala 2.12 / sbt / Pekko (+ classic Akka) | Scala 2.13 / Play 3 / Guice | PHP 8.3 / Smarty / DashFwd |
| Tier-1 home | `*/src/test/scala` per subproject | `*/src/test/scala` per subproject (53 spec classes across `service`, `domain`, `domain-wiring`) | `tests/php/` (legacy) + `src/core/Features/*/Tests/` (DashFwd) |
| Tier-2 home | **`integration-tests/` sbt subproject** with `src/it/scala/` (24 test classes + tag taxonomy + helpers) | **Absent** — sbt `IntegrationTest` config wired in `project/Settings.scala:25-29` and `ITProjectHelper.scala` defined but **never referenced**, **0 classes** in `src/it` | `tests/php/docker-compose.yml` boots real MySQL/Mongo/Redis containers per test run |
| Tier-2 split mechanism | sbt subproject + ScalaTest tags `Flaky`/`Slow`/`Manual`/`SNDeprecated` | None | PHPUnit suites + Docker Compose for infra |
| Tier-2 enforcement | All 24 IT classes live in the subproject; tags actively used for quarantine via `make integration-test` (`-l "Flaky Slow Manual SNDeprecated"`, `Makefile:73`) | **Phantom contract:** `it`, `it-dev`, `it-staging`, `it-production` declared in `Makefile:86` `.PHONY` only — **no recipes**; would fail with "No rule to make target" | All tests run together; hermeticity enforced via `FunctionOverrider` |
| Real infra in tests | Deployed dev SCUM + real partner APIs (anti-pattern, chronic flakes) | None — unit tests mock `WSClient` directly | Local MySQL 8 / Mongo 7 / Redis (RedLock + auth-Redis-cache) via Docker Compose (`tests/php/docker-compose.yml:1-57`) |
| Hermeticity guardrail | None | None | `FunctionOverrider` (`src/core/Features/Infrastructure/Tests/Utilities/FunctionOverrider.php`) overrides `curl_*`, `file_get_contents`, `fsockopen`, etc. — tests cannot make outbound HTTP without an explicit allow |
| In-process HTTP test surface | `EndpointTestHelper` injects requests through Pekko router (no socket) | Play `stubControllerComponents` + `FakeRequest`, controller methods called directly with mocked deps | None — PHP tests call controllers directly |
| Contract tests | **Consumer-driven**: provider Makefile clones consumer repo, runs consumer's contracts against staging SCUM (`Makefile:56-67`); plus partial Hootpact coverage for Facebook | **Absent** — SNP consumes `channelauth.yaml` and `crypto.yaml` for client codegen only (`domain/src/main/resources/specs/`); no spec for SNP itself, no Specmatic, no hootpact, no consumer-contract pattern | None in-repo (SOA + Wasabi tests live in `dashboard-test-automation` repo; Playwright tests live in `hootsuite-playwright` repo) |
| Spec source of truth | OpenAPI per area + Hootpact (Facebook only) | `domain/src/main/resources/specs/{channelauth,crypto}.yaml` — codegen inputs only, **never validated** | None |
| CI integration | Jenkins `postDeployDev` runs `make integration-test`, but **on failure prompts an operator to "Continue deploy" or "Retry tests"** (`Jenkinsfile:34-66`) | `Jenkinsfile` has only `unitTest` + `build` + deploy flags; **no `postDeployDev` / `postDeployStaging` / `postDeployProduction`**, **no hootpact, no Wasabi, no synthetic** | `commands.Jenkinsfile` runs PHPUnit; SOA/Wasabi/Playwright orchestrated in `integrationTests.Jenkinsfile` against deployed envs |
| Cron / scheduled runs | Inherits `gitOpsPipeline` cron behavior of host repo | **Absent** — no `gitOpsPipeline.cron` or `pipelineTriggers([cron(...)])` | Multiple Jenkinsfiles (`localPhpImage.Jenkinsfile`, `prodDeploy.Jenkinsfile`, etc.) with their own cron/triggers |


# Per-service detail

## SCUM (`service-social-communication`)

* **Tier 2 lives in a dedicated sbt subproject:** `integrationTests` in `build.sbt`, sources under `integration-tests/src/it/scala/com/hootsuite/service/sdk/socialcommunication/`. **24 test classes** (excluding tags and helpers): `OAuthTests`, `FacebookTests`, `FacebookPageTests`, `InstagramTests`, `InstagramBusinessTests`, `LinkedInV2Tests`, `LinkedInV2CompanyTests`, `TwitterTests`, `BlueskyTests`, `YouTubeChannelsTests`, `YouTubeVideosTests`, `YouTubeCommentsTests`, `YouTubeRatingsTests`, `YouTubeSubscriptionsTests`, `YouTubeModerateTests`, plus 8 `CrossNetworks*Tests`/`*EndPoint` classes and `FacebookPageUnpublishedPostsTests`.
* **ScalaTest tag taxonomy** under `integration-tests/src/it/scala/.../tags/`:
  * `Flaky.scala` — known-flaky, excluded by default
  * `Slow.scala` — long-running, opt-in
  * `Manual.scala` — requires human action (token refresh, captcha)
  * `SNDeprecated.scala` — covers paths slated for removal
* **`make integration-test`** runs `sbt 'it:testOnly -- -l "Flaky Slow Manual SNDeprecated"'` (`Makefile:72-73`) — the tags actively gate which subset runs.
* **Real infra in IT:** `reference.conf` points at the **deployed dev SCUM** with **real partner APIs**. Brittle: dev DB drift, partner API rate limits, expired tokens are routine breakers.
* **Consumer-driven contract testing**: SCUM Makefile defines `consumer-contract-test` template (`Makefile:56-62`) that **clones the consumer's repo and runs the consumer's contract tests against staging SCUM**. Today only `service-mobile-api` is wired (`Makefile:66-67`); a `dashboard-contract-tests` line is commented out at `Makefile:70` ("temporary exclusion while we figure out how to get a php container into the new jenkinsfile hotness"). This is the cleanest example in the org of "consumers own their expectations."
* **CI gate is theatre, not enforcement.** `Jenkinsfile:34-66` runs IT in `postDeployDev` inside a `while (tryIntegrationTests)` loop. On failure it prompts the operator via Slack with a `RELEASE_SCOPE` choice: `"Continue deploy"` or `"Retry tests"`. If the operator picks Continue, the deploy proceeds with `"Ignoring failed integration tests and continuing with deploy"` logged (`Jenkinsfile:53-57`). The gate exists; the enforcement does not.

**Takeaway:** subproject + tag taxonomy + consumer-driven contracts is the pattern to copy. Real partner APIs in IT and "ignore-and-continue" gates are the patterns to discard.

## SNP (`social-network-proxy`)

* **Tier 1 only.** ~53 spec classes across three sbt subprojects: `service` (24), `domain` (12 processor specs, one per network), `domain-wiring` (17). Mockito-driven; `WSClient` mocked directly (`BaseDefaultApiSpec.scala:31-44`).
* **No DB, no Testcontainers.** SNP is stateless — no `mysql`, `jdbc`, `slick`, `redis` deps anywhere.
* **Tier 2 is scaffolding only.** `project/Settings.scala:25-29` configures `Defaults.itSettings` and an `IntegrationTest` config; `ITProjectHelper.scala:12-30` defines `makeITProject` with forked IT config and `parallelExecution := true`. **`makeITProject` is never referenced**, and `src/it/scala` does not exist. The setup looks like Tier 2 from the outside but compiles to nothing.
* **Phantom Makefile targets.** `Makefile:86` declares `it`, `it-dev`, `it-staging`, `it-production` in `.PHONY` with no recipes anywhere in the file. Invoking any of them fails with "No rule to make target". Same misleading-contract anti-pattern as `service-social-profiles`' `make it` (called out in [`phased-plan.md`](./phased-plan.md) OQ-S2).
* **Tier 4 is absent.** `Jenkinsfile` (38 lines total) has only `initialize` / `unitTest` / `build` / deploy flags — **no `postDeployDev`, no `postDeployStaging`, no `postDeployProduction`**. No hootpact, no Wasabi, no synthetic.
* **No `gitOpsPipeline.cron`** in the Jenkinsfile, no `pipelineTriggers`, no GitHub Actions. Scheduled smoke is not evidenced anywhere.
* **Contract: consumer only.** SNP vendors `domain/src/main/resources/specs/channelauth.yaml` and `domain/src/main/resources/specs/crypto.yaml` for **outbound client codegen** via `HsOpenApiCodegenPlugin` (`build.sbt:30`). It does **not publish its own OpenAPI** despite ~20 callers (`AGENTS.md:81-108`). No Specmatic, no hootpact, no consumer-contract loop.
* **Hermeticity by convention only.** No sttp backend wrapper guard, no `RoundTripper` egress block. New tests can accidentally make real network calls if they bypass the mock pattern.
* **Xray non-gating.** `xrayScan = "permissive"` (`Jenkinsfile:18-19`) — security findings do not fail the build.

**Takeaway:** SNP is a useful counter-example. It has substantial unit coverage but **the same Tier-2/Tier-4 holes the channel-auth ecosystem has**, so any pattern we adopt for `channel-auth` / `social-network-auth` / `service-core` should ship to SNP at the same time, by the same team. The "phantom .PHONY targets" pattern is a Tier-2 misleading-contract liability we should explicitly delete in the channel-auth ecosystem and not let it spread.

## `dashboard`

* **Tier 1**: PHPUnit suites in `tests/php/` (legacy) and `src/core/Features/*/Tests/` (DashFwd-namespaced). Most extend `TestCase` or the project-internal `HsTestCase`.
* **Local infra via Docker Compose** (`tests/php/docker-compose.yml:1-57`):
  * MySQL 8 (root, plus `dev_team` user)
  * MongoDB 7
  * Redis 6 (no auth — used for RedLock)
  * Redis 6 (auth, `requirepass test-cache-password` — used for HsRedis cache)
* **Mocking layered:**
  * **AspectMock** for global bytecode mocking (used in `FunctionOverrider.php:10` and many test bootstraps; works, but high blast radius and aspectmock has deprecation pressure on PHP 8.3+).
  * **Mockery** for traditional class mocks.
  * **`FunctionOverrider`** (`src/core/Features/Infrastructure/Tests/Utilities/FunctionOverrider.php`, 1180 lines) overrides `curl_*`, `file_get_contents`, `fsockopen`, etc. **Tests cannot make outbound HTTP** without an explicit allow.
* **No in-repo contract tests** against channel-auth or core — those live in separate consumer SDK repos and Wasabi.
* **CI is split across multiple Jenkinsfiles:**
  * `commands.Jenkinsfile` — PHPUnit unit tests (Tier 1).
  * `integrationTests.Jenkinsfile` — orchestrates **SOA tests** (PHPUnit-style, parallel via Arbiter), **Wasabi tests** (cloned from `web/dashboard-test-automation.git`), and **Playwright UI tests** (cloned from `hootsuite/hootsuite-playwright`). Runs against staging or canary.
  * `pipeline.Jenkinsfile`, `prodDeploy.Jenkinsfile`, `canary.Jenkinsfile`, `localPhpImage.Jenkinsfile`, `opslevel.Jenkinsfile` — deploy and ancillary pipelines.
* **External-repo orchestration** is dashboard's distinguishing pattern: contract / integration / UI test code lives in repos owned by other teams (`dashboard-test-automation`, `hootsuite-playwright`) but is **invoked from the dashboard pipeline**. This is the inverse of SCUM's "clone the consumer repo from the provider side" — here the **provider clones the test repos that exercise it**.

**Takeaway:** the FunctionOverrider hermeticity guard and the docker-compose-real-infra pattern transfer well to the channel-auth ecosystem. The split-Jenkinsfile + external-test-repo pattern is heavier than what the channel-auth ecosystem needs at its current scale, but the **idea** of separating fast-feedback (`commands.Jenkinsfile`) from heavyweight integration (`integrationTests.Jenkinsfile`) is exactly what our **D4** Tier-2 PR gate does at a smaller scale.

# Patterns to borrow

1. **Subproject for Tier 2** — SCUM's `integration-tests/` cleanly isolates slow/integration code from fast unit. Mirror this for `social-network-auth` (`src/it/scala`, scaffolded but empty), `service-core`, `service-social-profiles`, and SNEI in Phase 2 of [`phased-plan.md`](./phased-plan.md).
2. **Tag taxonomy for quarantine** — `Flaky`/`Slow`/`Manual`/`SNDeprecated` from SCUM. Tag, do not delete; failed tags get a Jira ticket and a deadline. The actual gate is `make integration-test` excluding those tags by default (`SCUM Makefile:73`), so the tags carry their weight.
3. **Consumer-driven contracts (eventually)** — SCUM's clone-the-consumer-repo loop in `Makefile:56-67` is the cleanest example in the org. Adopt the _receiving_ end of this pattern in `channel-auth` and `social-network-auth` once consumers stabilize. **For Phase 4 we start with provider-published contracts** per **D6** in [`design-doc.md`](./design-doc.md); revisit consumer-driven once at least two consumers want to own their expectations.
4. **Hermeticity guardrail at the network boundary** — Dashboard's `FunctionOverrider`. Equivalents:
   * Go (channel-auth): a test-only `http.RoundTripper` wired into shared HTTP clients that fails on any unstubbed host.
   * Scala (sn-auth, core, social-profiles, SNEI): a test-only `SttpBackend` wrapper that fails on any unstubbed `Uri`.
5. **Real local infra over shared dev** — Docker Compose (Dashboard) and Testcontainers (proposed) over the shared-RDS pattern that today plagues `service-core`.
6. **In-process HTTP routing for fast handler tests** — `EndpointTestHelper` (SCUM, service-core today) is fine for unit-of-handler tests, but our Tier 2 binds a real socket per **D2** so serialization and middleware run.
7. **Jenkinsfile separation by tier** — Dashboard's `commands.Jenkinsfile` (fast) vs `integrationTests.Jenkinsfile` (slow, post-deploy) is a useful precedent for how a team can ladder gates without conflating PR feedback with deploy verification.

# Patterns to avoid

1. **Phantom Makefile targets** (SNP `Makefile:86`, also `service-social-profiles`' `make it`, also SNEI's `it-staging` running `sbt it:test` over zero tests). The contract is actively misleading and silently passes in CI. Delete or implement; do not leave declared-but-undefined.
2. **Dead IT scaffolding** (SNP `ITProjectHelper.scala:12-30` plus `Defaults.itSettings` plus `IntegrationTest` config with zero `src/it` tests). Creates false impression of Tier 2 coverage during compliance audits.
3. **Tag-only splits without CI enforcement** — applies to any service that declares an `IntegrationTest` config but never runs it. SCUM passes this bar (the tags actually gate `make integration-test`); a service that just defines tags without a gate fails it.
4. **Real partner APIs inside Tier 2** (SCUM `integration-tests/`) — moves to Tier 4 only. Tier 2 uses WireMock or recorded fixtures per **D2**.
5. **"Ignore and continue" gates** (SCUM `Jenkinsfile:46-57`) — either gate or do not gate. Halfway is neither signal nor speed. The Slack-prompt-into-input pattern is operator toil that does not produce reliable signal.
6. **Provider with no published spec** (SNP) when ~20 services consume the API — contract drift is undetected at CI time and only surfaces in production incidents.
7. **`xrayScan = "permissive"`** (SNP `Jenkinsfile:18-19`) — security findings that do not fail the build are findings that do not get fixed. Out of scope for CI-7194 but worth flagging adjacent to it.
8. **Jenkins-vs-`agents.Makefile` drift** (SNP) — Jenkins runs `make test` (lint + test); `agents.Makefile slow-verify` runs `lint + test + secscan`. Contributors can pass CI without ever running the local Snyk gate if hooks are not installed. Symmetry between local and CI gates is what `agents.Makefile` was supposed to deliver.

# Gaps our plan addresses (not covered by any service we sampled)

1. **Provider-side OpenAPI conformance at PR time.** None of these services validate that handler wiring matches the published OpenAPI before merge. SCUM has hootpact on Facebook only and at `postDeployDev`, not at PR. SNP has no provider spec at all. Dashboard has no in-repo contract gate. Our **D3** (Tier 2 Specmatic conformance) would be a Hootsuite-first.
2. **Hermetic, agent-runnable cross-service stack.** SCUM's IT relies on deployed dev. SNP has no IT. Dashboard's `integrationTests.Jenkinsfile` runs against staging. Our **Phase 3** Compose + WireMock + Kafka harness gives the agent a reproducible cross-service environment per **D7** — including the disconnect-event contract end-to-end.
3. **Consumer-side mock fidelity.** No service we sampled checks that _its mocks of upstreams_ match the upstream's published spec. SCUM, SNP, and dashboard all hand-write mocks against partner / sibling APIs and let them drift silently. Our new **Tier 1 mock-fidelity sub-tier** (see [`design-doc.md`](./design-doc.md) and Phase 2 of [`phased-plan.md`](./phased-plan.md)) closes that loop.
4. **Symmetric local + CI gates** via `agents.Makefile`. SNP demonstrates the failure mode (CI runs `make test`, local pre-push runs `slow-verify`); `channel-auth`, `service-core`, and `social-network-auth` already enforce symmetry via the kairos-pilot-3 pre-PR gate. Phase 5 of [`phased-plan.md`](./phased-plan.md) extends the contract to `service-social-profiles` and SNEI.

# Maintenance

This inventory is a one-time snapshot taken during CI-7194. We will not keep it current on a schedule. If a service makes a major change to its test infra that the channel-auth ecosystem could benefit from, append a dated entry to the relevant per-service section above and link the commit.
