# Agentic-First Integration Testing — Design Doc (CI-7194)

> **Status:** DRAFT | **Owner:** Clarissa Audrey | **Jira:** [CI-7194](https://hootsuite.atlassian.net/browse/CI-7194) | **Last updated:** 2026-06-18

> **Read next:** [`phased-plan.md`](./phased-plan.md) — Phases 1 through 5, sequencing, critical-path ordering, open questions, out of scope.
>
> **Frontend companion:** [`frontend-testing.md`](./frontend-testing.md) — `channel-integrations/` FE testing strategy. Different toolchain (jest + MSW, Storybook), different cadence story, contingent on Hippogriff ownership confirmation.
>
> **Read also:** [`services-inventory.md`](./services-inventory.md) — what other Hootsuite services (SCUM, SNP, dashboard) have for integration and contract tests today, patterns we are borrowing, and patterns we are avoiding.

# TL;DR

Adopt a **four-tier test pyramid** across the team's backend surface — the five services that own the social-network OAuth + disconnect lifecycle (`channel-auth`, `social-network-auth`, `service-core`, `service-social-profiles`, `social-network-error-inspection`) plus the cross-repo harness in `channel-auth-tool` — so that agentic development can verify its own changes before completing them.

| Tier | Where | Purpose | Audience |
| --- | --- | --- | --- |
| **1** Fast unit | Per repo | Logic correctness + mock fidelity | Agent + human, every save |
| **2** In-repo integration | Per repo | HTTP wiring + DB + OpenAPI conformance | Agent + human, blocking PR gate |
| **3** Cross-repo harness | `channel-auth-tool` | End-to-end profile lifecycle through real services, including the disconnect event flowing to `service-social-profiles` and SNEI | Agent (verify-and-iterate), opt-in |
| **4** Post-deploy smoke | Per repo (hootpact / specmatic) | Real partner-API regression detection | Cron / humans, never agents |

Per-repo Tier 1+2 stays the PR gate. Tier 3 is opt-in and agent-runnable, the surface this SPIKE is really about. Tier 4 is post-deploy.

`channel-auth-tool` is repurposed from a CLI utility into the Tier 3 cross-repo harness, refactored into an importable Python library plus a Docker Compose stack with WireMock partner stubs, Kafka (or Redpanda) plus the kafka-bridge sidecar, and `service-social-profiles` and SNEI subscribed as event consumers — so the harness can verify the full disconnect contract end-to-end.

> **Highest-impact single change:** replace the shared dev RDS connection in `service-core`'s test config with Testcontainers MySQL. Today every developer's `service-core` tests fight over the same database, the single biggest agentic-first blocker.

# Background

## Current state across the team's backend services

Read-only audit findings:

| Repo | Today | Biggest agentic-first gap |
| --- | --- | --- |
| `channel-auth` (Go) | 27 colocated `_test.go`, mockery v3 + httptest, no tier separation, no build tags, real MySQL via `make up`, Redis tests soft-skip | No `integration` tier. Redis tests silently skip in CI. No real-network or contract tests. |
| `social-network-auth` (Scala) | 36 unit suites, Mockito + fake SDK clients, `parallelExecution := false`, hootpact via `make it` hits deployed dev, 3 example contracts (Facebook only) | Vestigial `IntegrationTest` config. Hootpact only covers token-status for Facebook. Per-network gap. |
| `service-core` (Scala) | 22 test files, IntegrationTest sbt config declared but unused, **DAO tests hit a shared dev RDS cluster**, crypto mocked, Amplify Wasabi post-deploy | DAO tests are non-hermetic. Re-encryption pipeline has zero coverage. |
| `service-social-profiles` (Scala) | Scala 2.13 + Tapir+http4s + Cats Effect, Redis only (no MySQL), macwire DI, kafka-bridge HTTP consumer, 21 unit suites (~120 cases), six vendored OpenAPI specs in `src/main/resources/specs/` for codegen only | **Zero integration tests** despite an `integration/` sbt subproject scaffold. `make it` is a **no-op** despite README and Makefile referencing it. No Kafka end-to-end coverage of the disconnect ingress path. |
| `social-network-error-inspection` (SNEI, Scala) | Scala 2.13 + Tapir+http4s (Blaze) + **MySQL via ScalikeJDBC + HikariCP + Flyway**, kafka-bridge consumer + producer, ~32 unit classes (parser-heavy: 17 disconnect parsers), Wasabi staging suite (1 spec, 6 cases) | **Zero integration tests** in empty `src/it/scala`. No real MySQL tests. Tapir generates Swagger at runtime but **nothing checked in to IDL-central**, so D3 (Tier 2 OpenAPI conformance) has no spec to conform against. Public Aperture-fronted API only tested via staging Wasabi. |
| `channel-auth-tool` (Python) | Zero tests. CLI-only. Hardcoded URLs. Calls Core + SOM + network APIs directly, does **not** call channel-auth. `input()` on stdin for OAuth callback. | Not a library. Cannot be driven by an agent. Wrong neighbor set to be the channel-auth integration tester. |

> **Critical fact:** `channel-auth-tool` does not currently call `channel-auth`. If we want it to be the cross-repo harness, that is a refactor, not a re-purposing.

> **Cross-service note:** `service-social-profiles` and SNEI are both **consumers of** `social_profile_disconnect_discovered` — the event `channel-auth` emits when token refresh fails. social-profiles invalidates its Redis cache; SNEI parses and persists the disconnect to MySQL. This is the highest-value cross-service contract the team owns and is the load-bearing reason Tier 3 must include Kafka and both consumers.

# Decisions

| # | Decision | Rationale |
| --- | --- | --- |
| **D1** | Add `channel-auth` as a target service **alongside** existing direct Core/SOM calls. Do **not** replace them. | Scenarios drive flows through `channel-auth`'s public API and assert ground truth via direct Core/SOM reads. Gold-standard integration pattern. |
| **D2** | Tier 2 binds a **real HTTP server on an ephemeral port**. No in-process routing (no Tapir stub interpreter, no `EndpointTestHelper.executeTestRequest`, no direct Chi handler calls). | Catches the bug class agents introduce most: routing, JSON codec, content-type, and OpenAPI-spec drift. |
| **D3** | OpenAPI / hootpact conformance is a **Tier 2 concern**, not just Tier 4. Each service gets a conformance test class. | Catches spec-vs-code drift at PR time, not post-deploy. Highest-leverage agentic-first lever. |
| **D4** | Tier 2 is a **blocking PR gate with no time budget**. Quality over speed. | Trustworthy gate. If suites grow unmaintainable, split or move to Tier 3 with evidence, not preemptively. |
| **D5** | Investigate `Test / parallelExecution := false` before changing. The Scala services (`social-network-auth`, `service-core`, `service-social-profiles`, SNEI) disable parallelism today. | Probable shared-state bug. Flipping blindly will green-then-red. Treat parallelism as a Phase 2 outcome, not input. |
| **D6** | Phase 4 contract approach: start **provider-published** (OpenAPI from IDL-central / `schema/`), defer **consumer-driven**. | Provider-published has lower bootstrap cost and reuses an existing source of truth. Migrate to consumer-driven later, modeled on SCUM. See Integration-vs-contract section below. |
| **D7** | Tier 3 cross-repo harness includes Kafka (or Redpanda) plus kafka-bridge plus `service-social-profiles` and SNEI as event consumers, **not as an opt-in extension**. | The disconnect-event contract from `channel-auth` to its two consumers is the highest-value integration the team owns and cannot be verified at any lower tier. Keeping it inside Phase 3 base scope means the harness verifies the full team-owned contract end-to-end. |


# Q1: Per-repo or multi-repo?

**Both, but at different tiers.** Forcing a binary choice produces bad outcomes either way:

* **Per-repo only**: contract drift between `channel-auth` <-> `social-network-auth` <-> `service-core`, and between `channel-auth` and its event consumers (`service-social-profiles`, SNEI), is invisible until staging. Roughly today's state.
* **Multi-repo only**: every PR boots the whole world. CI bottleneck, flaky on partner APIs, agents wait minutes per iteration.
* **Tiered (recommended)**: per-repo Tier 1+2 stays the PR gate; cross-repo Tier 3 is opt-in and agent-runnable; Tier 4 is post-deploy.

# Q2: What does an agentic-first tech plan look like?

The classical pyramid was designed around **human cognitive load**. Agents flip that constraint: writing the 100th scenario is the same as the 1st. New constraints:

| Old (human) | New (agentic) |
| --- | --- |
| Don't write too many integration tests | Don't write **flaky** integration tests |
| Keep tests readable | Keep failure output **diff-able** by an LLM |
| Minimize maintenance burden | Maximize **reproducibility** |
| Use stack traces for debugging | Use **named, isolated scenarios** an agent can re-run by ID |

Five non-negotiable principles:

1. **Hermetic** — no shared dev RDS, no localhost-or-skip, no rate-limited partner APIs in any tier below post-deploy. Use Testcontainers and WireMock.
2. **One command per tier** — `make test`, `make test-integration`, `make harness`, `make smoke`. No flags to remember.
3. **Self-describing failures** — Mockito's wanted-but-not-invoked stack traces are agent-hostile. Prefer ScalaTest `clue()`, Go `t.Logf` with structured context, OpenAPI/hootpact contract diffs.
4. **Stable scenario names** — `pytest scenarios/test_pinterest.py::test_create_then_refresh` is re-runnable. Anonymous Mockito spies are not.
5. **Pre-seeded, deterministic fixtures** — kill `input()`. The harness must run unattended.

# Q3: What does the test pyramid look like?

Four tiers, slightly top-heavier than classical because (a) Tier 3 catches cross-repo contract drift, (b) partner OAuth surfaces change often. Heavier != inverted: Tier 1 still has 10x the cases of Tier 2.

```
Tier 4: Post-deploy smoke (real partner APIs, real envs)
        ~10-20 scenarios per service, hootpact / specmatic
   |
Tier 3: Cross-repo harness (channel-auth-tool, Docker Compose + WireMock + Kafka)
        ~30-50 scenarios across all networks plus the disconnect event
        contract through service-social-profiles and SNEI;
        OPT-IN, agent-runnable
   |
Tier 2: In-repo integration (Testcontainers, real HTTP on ephemeral port)
        OpenAPI conformance suite, fakes for Core/SOM/network clients
        ~100 cases per service, blocking PR gate, no time budget
   |
Tier 1: In-repo fast unit (fully mocked, hermetic, in-process)
        ~500-2000 cases per service, PR gate, fast
        + mock-fidelity sub-suite: Specmatic checks that our mocks of
          upstreams match the upstreams' published OpenAPI specs
```

The **mock-fidelity sub-suite** is new in this plan. It is fast (< 5s per upstream), runs alongside unit tests, and catches "we mocked it wrong" before Tier 2 even gets to compile. Implementation lands as a Phase 2 task per repo — see [`phased-plan.md`](./phased-plan.md).

# Triggers and cadence

The pyramid above answers _what_ runs at each tier. This section answers _when_ and _how_ each tier fires — which is critical for the agentic-first goal because the agent's iteration speed depends on knowing which gate fires at which moment.

We rely only on triggers I have direct evidence of in the team's backend repos. The Hootsuite-wide trigger surface is `gitOpsPipeline` (Jenkins shared library) plus per-developer git hooks; there is no Kubernetes CronJob test runner and no GitHub Actions in any of the audited backend repos.

## Tier 1 — fast unit + mock fidelity

| Trigger | Cadence | Status today | Evidence |
| --- | --- | --- | --- |
| Agent / local `make test` | On every change or save | Available everywhere | All repos |
| Pre-push git hook (`agents.Makefile slow-verify`) | Per developer push | Available — `channel-auth`, `service-core`, `social-network-auth` | Each repo's `AGENTS.md` Pre-PR Gate block |
| Jenkins `unitTest` stage | Per PR push and per main push | Available everywhere via `gitOpsPipeline.unitTest` | All five service Jenkinsfiles plus `channel-auth-tool` |
| Jenkins weekly pipeline cron rerun | Weekly | **Open question** — see callout below | `cron = 'H H * * H'` set in `channel-auth/Jenkinsfile`, `service-core/Jenkinsfile`, `social-network-auth/Jenkinsfile`, `service-social-profiles/Jenkinsfile`, `social-network-error-inspection/Jenkinsfile` |

## Tier 2 — in-repo integration

Same trigger surface as Tier 1, but each row requires implementation in **Phase 2** before it fires.

| Trigger | Cadence | Status | Phase |
| --- | --- | --- | --- |
| Agent / local `make test-integration` | On-demand | New | Phase 2 task per repo |
| Pre-push hook (extend `slow-verify` to include `test-integration`) | Per developer push | New | Phase 2 task per repo |
| Jenkins `unitTest` stage extended to also run `test-integration`, **blocking per D4** | Per PR push and per main push | New | Phase 2 task per repo |
| Jenkins weekly pipeline cron rerun | Weekly | **Open question** — inherits Tier 1's pipeline cron if pipeline-level | Phase 2 — depends on cron clarification |

## Tier 3 — cross-repo harness in `channel-auth-tool`

| Trigger | Cadence | Status | Phase |
| --- | --- | --- | --- |
| Agent / local `make harness` | On-demand | New | Phase 3 |
| Jenkins build for `channel-auth-tool` repo only (boots Compose stack and runs harness scenarios, including Kafka + the two event consumers per **D7**) | Per PR push to `channel-auth-tool` | New | Phase 3 |
| Periodic harness against deployed dev (e.g., weekly) | Weekly | **Not available today** — would need either a `gitOpsPipeline` host repo or a new platform primitive | **Open question** — see callout below |

Explicit non-trigger by design (per Q1): Tier 3 is **not** a PR gate for the five component repos. Each ships its own Tier 1+2 gates; Tier 3 is the cross-repo agent verification surface, opt-in.

## Tier 4 — post-deploy smoke

| Trigger | Cadence | Status | Source / Phase |
| --- | --- | --- | --- |
| Jenkins `postDeployDev` closure | After every dev deploy | Active in `social-network-auth` (`make it`) and SCUM (`make integration-test`); planned for `channel-auth`, `service-core`, `service-social-profiles`, and SNEI in Phase 4 | `social-network-auth/Jenkinsfile:33-36`, `service-social-communication/Jenkinsfile:34-66`, Phase 4 plan |
| Jenkins `postDeployStaging` closure | After every staging deploy | Active in `service-core` (Amplify Wasabi contract test); active in SNEI (`make it-staging`, runs Wasabi and a currently-empty sbt IT) | `service-core/Jenkinsfile:38-54`, `social-network-error-inspection/Jenkinsfile` |
| Jenkins weekly pipeline cron rerun (re-exercises `postDeployDev` and the hootpact suite) | Weekly | **Open question** — same cron line as Tiers 1/2 | See callout below |
| Manual re-deploy or manual Jenkins re-run | On-demand | Available everywhere | Standard Jenkins capability |

## Agent verification surface (which command runs which tier)

| Command | Tier | Notes |
| --- | --- | --- |
| `make test` | Tier 1 | Includes the mock-fidelity sub-suite once Phase 2 lands |
| `make test-integration` | Tier 2 | New in Phase 2 |
| `cd channel-auth-tool && make harness` | Tier 3 | New in Phase 3, opt-in, agent-runnable; covers the full disconnect contract per **D7** |
| _no agent command_ | Tier 4 | **Hard rule from Phase 5: agents never run Tier 4.** Real-network smoke is post-deploy on real infra. |

## Open question — cron / scheduled test triggers

> **Routing to backend-platform:** `gitOpsPipeline.cron = 'H H * * H'` is set in all five backend Jenkinsfiles (`channel-auth/Jenkinsfile:10`, `service-core/Jenkinsfile:24`, `social-network-auth/Jenkinsfile:13`, `service-social-profiles/Jenkinsfile:9`, `social-network-error-inspection/Jenkinsfile:9`), which reruns the entire pipeline (including `unitTest` and `postDeployDev`) once a week. The doc treats this as a candidate Tier 1/2/4 cadence trigger but does **not** commit until backend-platform confirms:
>
> 1. Is the `gitOpsPipeline.cron` parameter a supported testing primitive (versus an undocumented carryover)?
> 2. Is there a separate standalone test runner planned (e.g., a Kubernetes CronJob primitive, a `gitOpsPipeline.scheduledTest` stage)?
> 3. If neither, what is the supported way to detect partner-API drift between deploys, given that hootpact today only fires on `postDeployDev` and won't catch a partner regression until something redeploys?
> 4. **Specific concern for `service-social-profiles`:** that repo combines the weekly cron with `deployProduction = true`, so the same pipeline rerun also auto-deploys to prod weekly with no code change. If the cron stays, this combination needs a deliberate sign-off from backend-platform — it compounds the operational concern of relying on the cron as a testing primitive.

This question is also tracked in [`phased-plan.md`](./phased-plan.md#open-questions).

# Integration vs contract tests — where each lives in the pyramid

Integration tests and contract tests answer different questions. The four-tier pyramid covers both, but they sit in different cells.

* **Integration test:** does my service work end-to-end with realistic collaborators (DB, HTTP server, fakes for upstreams)?
* **Contract test:** does the interface between two parties match the agreed spec, on either side?

The two-by-two:

|  | **Provider side** (we own the API) | **Consumer side** (we call someone else's API) |
| --- | --- | --- |
| **Integration** | Tier 2: real HTTP server bound to an ephemeral port; handlers wired through real router/middleware; fakes for downstreams. Verifies handler wiring, serialization, error mapping. | Tier 2: real HTTP client against a fake of the upstream. Verifies our outbound shape (URL, headers, body, retries) — _given a fake we trust_. |
| **Contract** | Tier 2: **OpenAPI conformance** via Specmatic — every endpoint we expose conforms to the published spec, run as a blocking PR gate (decision **D3**). | **Tier 1 mock-fidelity sub-suite (new):** our hand-written mocks of Core/SOM/network APIs are validated against the published OpenAPI of those upstreams. Catches "we mocked it wrong" drift before it ships. |

Cross-service end-to-end behavior (Tier 3) and live-partner validation (Tier 4) sit outside this matrix; they are the integration tests that no in-repo tier can replace, and they do not gate PRs.

The disconnect-event contract from `channel-auth` to `service-social-profiles` and SNEI is the team's most consequential cross-service contract and is verified end-to-end at Tier 3 per **D7**.

## Phase-4 contract approach: start provider-published, not consumer-driven (D6)

We start contract testing with **provider-published OpenAPI** as the source of truth, not consumer-driven contracts (CDC). Three reasons:

1. **We already publish OpenAPI** in `IDL-central` and per-repo `schema/` directories (with the SNEI exception called out in the Phase 2 SNEI section of [`phased-plan.md`](./phased-plan.md)). CDC would duplicate a source of truth we already maintain.
2. **Most consumers today are SDK consumers**, not contract-test owners. Our Scala SDK is generated; PHP and Go consumers do not host contract tests yet.
3. **Bootstrap cost is lower.** Provider-published lets each repo gate its own merges immediately. CDC requires inter-repo coordination and mature consumer teams.

We migrate to **consumer-driven** later, mirroring SCUM's `consumer-contract-test-*` pattern (see [`services-inventory.md`](./services-inventory.md)), once we have at least two consumer teams who want to own their expectations.

---

**Continue to:** [`phased-plan.md`](./phased-plan.md) for Phases 1–5, sequencing, critical-path ordering, open questions, and out-of-scope. **Or:** [`frontend-testing.md`](./frontend-testing.md) for the FE companion.
