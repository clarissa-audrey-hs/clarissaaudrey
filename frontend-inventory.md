# fe-global FE testing inventory — integration and contract patterns (CI-7194)

> **Status:** DRAFT | **Owner:** Clarissa Audrey | **Jira:** [CI-7194](https://hootsuite.atlassian.net/browse/CI-7194) | **Last updated:** 2026-06-18

Companion to [`frontend-testing.md`](./frontend-testing.md). This page inventories what other namespaces in `fe-global` have for integration and contract testing today, so the `channel-integrations/` testing strategy can borrow proven in-monorepo patterns instead of inventing new ones. It is a snapshot, not a survey.

The single most important fact this audit established:

> **No namespace in `fe-global` has any consumer-side contract validation.** No Pact, no hootpact, no Specmatic, no OpenAPI conformance. The "MSW handler fidelity" sub-tier proposed in [`frontend-testing.md`](./frontend-testing.md) and the consumer-side Tier 1 in [`design-doc.md`](./design-doc.md) would be a Hootsuite FE first.

# Methodology

Searched `/Users/clarissa.audrey/repos/fe-global` for:

| Signal | Classified as |
| --- | --- |
| `setupServer` / `from 'msw'` | MSW in Jest (network-layer integration) |
| `parameters.msw` / `msw-storybook-addon` | MSW in Storybook |
| `axios-mock-adapter` / `global.axiosMock` | Axios-level HTTP mocking (legacy/alternate) |
| `*.cypress.*`, `cypress.config.*`, Cypress deps | Cypress |
| `*.contract.*`, `pact/`, `hootpact/`, `specmatic` | Contract tests |
| `@storybook/test-runner`, `play:` in stories | Storybook interaction tests |
| `@chromatic-com`, `percy` | Hosted visual diff |
| `visual:check`, `fe-lib-visual-testing`, Playwright configs | Visual regression |

A namespace is included only when at least one package has evidence beyond plain Jest + `@testing-library/react`. Namespaces with zero hits are intentionally **omitted** from the per-namespace section: `engage`, `design-system`, `dashboard`, `inbox`, `streams`, `reports`, `mobile`, `offers`, `contentlab`, `appdir`, `social-network`, `acquisition`, `adespresso`.

**Absences confirmed repo-wide** (not inferred):

* 0 `*.cypress.*` files, 0 `cypress.config.*`
* 0 `*.contract.{test,spec}.*`, 0 `pact/` or `hootpact/` directories
* 0 `@storybook/test-runner` deps or `test-storybook` scripts
* 0 `play:` functions in any `*.stories.*` file
* 0 `@chromatic-com/storybook` or Percy usage

# Per-namespace inventory

Ordered by maturity of integration testing, most-mature first.

## `adpromotion` (~53 packages) — most mature MSW usage

| Toolchain | Coverage |
| --- | --- |
| **MSW `setupServer` in Jest** | ~9 packages, ~14 test files (audience-field, edit components, FB/LI modals, `fe-adp-lib-client-organization`, `fe-adp-lib-client-ads-management`, `fe-adp-lib-client-ad-promotion`) |
| **MSW in Storybook (`parameters.msw`)** | ~5 packages, 8 story files |
| **Shared handler modules** | `__mocks__/apiHandlers.ts` (486 lines) reused across Jest and Storybook |

* **Canonical Jest test:** `adpromotion/fe-adp-comp-fb-audience-field/src/FbAudienceField.jest.tsx:1-49` — `setupServer(handleGetSavedAudiences, handleGetAudience, getReachMockedServer, handleGetCustomAudiences)` plus a `jest.mock('fe-comp-aperture', ...)` that routes Aperture API calls through `window.fetch` so MSW intercepts them.
* **Canonical handler module:** `adpromotion/fe-adp-comp-fb-audience-field/src/__mocks__/apiHandlers.ts:1-486` — exports named `http.get`/`http.post` handlers with query-param branches (`no-result`, `error`, default fixtures).
* **Why it matters for `channel-integrations/`:** This is the most direct precedent for the "shared handler module + dual-use across Jest and Storybook" pattern that **OQ-FE-1** in [`frontend-testing.md`](./frontend-testing.md) is asking about. `fe-chan-comp-reauth-modal` and `fe-chan-multiselect-auth-app` could converge on this shape with low marginal cost.

## `plancreate` (~121 packages) — biggest namespace, dual mocking strategies

| Toolchain | Coverage |
| --- | --- |
| **MSW `setupServer` in Jest** | 4 data packages (`fe-pnc-data-drafts`, `fe-pnc-data-favorites`, `fe-pnc-data-facebook-albums`, `fe-pnc-data-predictive-compliance`) |
| **MSW in Storybook** | 6 comp packages (social-network-picker, perch-nav, message-editor, lexical-editor, hashtags-suggestion-pane, etc.) |
| **`axios-mock-adapter`** | 1 lib package (`fe-pnc-lib-api/src/api.jest.js`, ~733+ lines) |

* **Storybook MSW example:** `plancreate/fe-pnc-comp-social-network-picker/src/SocialNetworkPicker.stories.js:45` — `parameters.msw`.
* **Axios mock example:** `plancreate/fe-pnc-lib-api/src/api.jest.js:19-40` — `MockAdapter(axios)` per test suite.
* **Caveat:** `fe-pnc-lib-api` is a legacy axios consumer; the MSW-versus-axios-mock split here is incidental to age, not a design choice. New `fe-chan-*` work should standardise on MSW.

## `member-management` (~7 packages) — Storybook-first MSW

| Toolchain | Coverage |
| --- | --- |
| **MSW in Storybook** | 4 packages (sso-config, invite-org-members, create-team-modal stories) |
| **MSW `setupServer` in Jest** | 1 package (`fe-member-comp-create-team-modal`) |
| **Per-package `mocks/` handler files** | `fe-member-comp-sso-config/src/mocks/organizationMembers.ts:12`, `organizationsSSO.ts:4,50` |

* **Critical detail:** `member-management/fe-member-comp-create-team-modal/src/CreateTeamModal.test.tsx:16-24` calls `axiosMock.restore()` **before** `server.listen()` to avoid the global `axiosMock` (set up in `__tapas__/config/jest/jest.setup.js:40`) intercepting `fe-comp-aperture` calls before MSW gets to them. This gotcha applies to anyone introducing MSW into a package that uses Aperture.

## `unified-composer` (~9 packages) — only namespace using `visual:check`

| Toolchain | Coverage |
| --- | --- |
| **MSW in Storybook** | 3 packages (message-composition, hashtag-suggestions, text-editor) |
| **`visual:check` story tags** | 1 package — `fe-uc-comp-message-composition` (5+ stories tagged `visual:check`, e.g. `MessageComposition.stories.tsx:217,225`) |

* **Storybook MSW richest example:** `unified-composer/fe-uc-comp-message-composition/src/components/MessageComposition/MessageComposition.stories.tsx:165-184` — multiple named handler groups (owly-writer, entitlements).
* **Why interesting:** the only namespace currently feeding stories into `platform/fe-lib-visual-testing`. If `channel-integrations/` ever wires Tier 4 visual regression, this is the precedent for tagging.

## `channel-integrations` (~27 packages, 24 team-owned) — the baseline

For completeness — the team's own current state, summarised here so this page can be read standalone. Detail in [`frontend-testing.md`](./frontend-testing.md).

| Toolchain | Coverage |
| --- | --- |
| **MSW `setupServer` in Jest** | 1 of 24 — `fe-chan-multiselect-auth-app/src/api/fetchProfiles.jest.ts:1-123` (`onUnhandledRequest: 'error'` per `:7-10`) |
| **MSW in Storybook** | 1 of 24 — `fe-chan-comp-reauth-modal/src/index.stories.js:238-240` plus mocks at `src/mocks/disconnectionReasonMocks.js:1-76` |

The team is **not starting from zero** on FE integration — both surfaces (Jest + Storybook MSW) have an in-namespace precedent. The work in [`frontend-testing.md`](./frontend-testing.md) is to extend these patterns to the rest of the 24 packages and bolt on the contract-validation (mock fidelity) layer that no namespace in `fe-global` has yet.

## Smaller namespaces with non-trivial integration

* **`enterprise-xp`** — 1 package (`fe-epe-lib-approvals`, MSW in Storybook). OpenAPI mentioned only in `aidlc-docs/` prose, **not** in test code.
* **`billing`** — 1 package (`fe-billing-comp-payment-update`, MSW `setupServer` in Jest, 2 test files).
* **`global-libs`** — 1 package (`fe-lib-async-app/src/hsAppLoaderV3.jest.ts:5-41`, mocks `async-apps` URL fetches).
* **`analytics`** (~19 packages) — has `fe-anl-lib-test-utils` (`componentSpy`, `WithMockImport`, type-safe assertions, `src/index.ts:1-9`) used by ~10+ chart packages, but it is **unit-test helpers only**, not MSW/network integration. Omitted as an integration precedent.

# Cross-namespace shared infra

## `platform/fe-storybook` (Storybook host)

* **What it is:** Private workspace package; the central Storybook entry point. Every namespace's stories run here.
* **What it exposes:** Global Storybook config; **MSW addon initialised globally** at `platform/fe-storybook/src/preview.tsx:27-59` with `initialize()`, `mswLoader`, `onUnhandledRequest: 'bypass'`. Eight platform-wide decorators mock `fe-lib-hs`, `fe-comp-aperture`, GraphQL, DarkLaunch, Optimizely, Pendo, etc. (`platform/fe-storybook/src/mocks/`).
* **Deps:** `msw` and `msw-storybook-addon` declared at `platform/fe-storybook/package.json:59-60,74-77`; the worker lives in `public/`.
* **Why it matters:** Per-package MSW work for `channel-integrations/` only needs to add `parameters.msw.handlers` per story — the platform layer is already wired.
* **Run via:** `make storybook` (`Makefile:138-140`).

## `platform/fe-lib-visual-testing` (Playwright visual regression — scaffolding only)

* **What it is:** Playwright config (`src/playwright.config.ts:1-31`) plus a runner (`src/test.stories.spec.ts:1-57`) that fetches Storybook's `index.json` and screenshots stories tagged `visual:check`.
* **Deps:** `@playwright/test` only (`package.json:14-22`).
* **What it does NOT have:** No `__snapshots__/` committed in repo; no production CI invocation (the `visual-test` Jenkins stage at `Jenkinsfile:366-383` is **commented out**).
* **Who uses it today:** Only `unified-composer/fe-uc-comp-message-composition` tags stories `visual:check`.
* **Implication:** Adopting visual regression for `channel-integrations/` Tier 4 means **platform work first** to uncomment the CI stage and seed baselines — not just adding tags to stories.

## Root Jest setup — `__tapas__/config/jest/jest.setup.js`

* **Lines 3-4:** `whatwg-fetch` polyfill (load-bearing for MSW in jsdom).
* **Line 40:** `global.axiosMock = new AxiosMockAdapter(axios)` — every Jest run gets a default axios interceptor unless a test calls `axiosMock.restore()`. This is the source of the MSW-vs-axios-mock collision documented above for `member-management`.

## Root `package.json:80`

* `msw` 2.6.5 hoisted at monorepo root — packages do not need a local `msw` dep to import it.

## What does NOT exist as cross-namespace shared infra

* No `fe-lib-test-fixtures` / `fe-*-lib-mocks-*` package for shared HTTP fixtures across namespaces.
* No `fe-lib-contract-test-utils` / `fe-lib-specmatic` / Pact helper.
* No shared OpenAPI loader for the FE side.

> **The mock-fidelity sub-tier in [`design-doc.md`](./design-doc.md) and [`frontend-testing.md`](./frontend-testing.md) would either need to (a) live inside each consuming package's tests or (b) ship as a new `fe-chan-lib-test-fixtures` (or similar) package. There is no existing FE infra to plug into.**

# CI integration findings

## Root `Jenkinsfile`

| Stage | Lines | What runs |
| --- | --- | --- |
| **Integration** | `333-386` | Parallel `test` + `lint` only |
| **`test`** | `337-339` | `make test-ci` → `./tapas plan --step TEST --raw \| ./tapas test --ci` |
| **`visual-test`** | `366-383` | **Commented out** — would run `yarn workspace fe-lib-visual-testing test` and S3 baseline sync |
| **Build** | `312-316` | `make build-ci` |
| **Storybook test-runner / Cypress / contract / Pact** | — | **Not referenced anywhere** |

## Root `Makefile`

| Target | Lines | Purpose |
| --- | --- | --- |
| `test-ci` | `70-71` | Tapas-planned Jest only |
| `storybook-visual-testing` | `142-144` | Local-only — starts Storybook for impacted TEST packages |
| `storybook-visual-testing-ci` | `146-148` | CI variant (port 6006, no HTTPS) — invoked by the commented-out Jenkins stage |
| `mono e2e` / `mono integration` | — | **Not defined** anywhere in the monorepo |

## `docs/dev/testing.md`

Documents **only** Jest + React Testing Library (`:1-55`). No mention of MSW, Storybook testing, Cypress, or contract tests. The team docs describe the same one-tier reality the CI enforces.

**Bottom line:** CI's "Integration" stage is a misnomer — it runs **Tapas-planned Jest only**. There is no FE PR-time integration gate beyond unit tests, and no FE post-deploy gate at all.

# Patterns to borrow for `channel-integrations/`

These three patterns are the strongest in-monorepo precedents for the work proposed in [`frontend-testing.md`](./frontend-testing.md). All three keep the migration cost low because they reuse infra we already pay for.

## 1. MSW `setupServer` in Jest for API clients (already started in-namespace)

* **Pioneer:** `adpromotion` (mature). **Sister precedent in `channel-integrations`:** `fe-chan-multiselect-auth-app/src/api/fetchProfiles.jest.ts`.
* **What to copy:** `setupServer` lifecycle, `onUnhandledRequest: 'error'`, per-test `server.use(http.post(...))`, `window.hs` fixture setup.
* **Migration cost for `fe-chan-*`:** **Low.** Extend to other API modules across the 24 team-owned packages. No new deps; root `msw` already available. If a target package uses Aperture, add the `jest.mock('fe-comp-aperture', ...)` shim that `adpromotion` uses (route Aperture calls through `window.fetch` so MSW intercepts).

## 2. Shared `mocks/` handlers + Storybook `parameters.msw` (already started in-namespace)

* **Pioneer:** `member-management` (`mocks/*.ts` + story meta). **Sister precedent:** `fe-chan-comp-reauth-modal`.
* **What to copy:** Export `http.get`/`http.post` handlers from `src/mocks/`, wire them into story meta as `parameters.msw.handlers` (the pattern at `fe-chan-comp-reauth-modal/src/index.stories.js:238-240`).
* **Migration cost:** **Low–medium.** Extend to multiselect-auth-app stories, Bluesky connect modal, permissions flows, and the rest of the 15 modal/dialog packages identified in [`frontend-testing.md`](./frontend-testing.md). Reuse the same handler files in Jest via `setupServer(handler)` — the `adpromotion` dual-use pattern.

## 3. Package-level `__mocks__/apiHandlers.ts` shared between Jest and Storybook

* **Pioneer:** `adpromotion/fe-adp-comp-fb-audience-field` (486-line handler module).
* **What to copy:** One file per API surface (channel-auth-audit, social-network-error-inspection, dashboard-som ajax paths) with query- or body-branching for error states.
* **Migration cost:** **Medium.** Requires cataloguing `fe-chan-*` outbound URLs and aligning fixture shapes with real backend responses. Pays off when multiple `fe-chan-*` packages hit the same backend (channel-auth-audit and SNEI in particular — both consumed by `fe-chan-comp-reauth-modal` per [`frontend-testing.md`](./frontend-testing.md)).
* **Resolves OQ-FE-1:** the "shared fixture format" question in [`frontend-testing.md`](./frontend-testing.md) has a working in-monorepo answer — copy `adpromotion`'s file shape.

# Patterns to avoid (grounded in repo evidence)

| Pattern | Why avoid |
| --- | --- |
| **FE contract / hootpact / Pact / OpenAPI validation** | **Zero implementation anywhere in `fe-global`.** Only design-doc prose in `enterprise-xp/aidlc-docs/`. There is no precedent to borrow — we would be pioneering. The mock-fidelity sub-tier we are proposing in [`design-doc.md`](./design-doc.md) and [`frontend-testing.md`](./frontend-testing.md) is exactly this kind of new work, but we should be honest that it is novel for FE. |
| **Cypress component or E2E** | Zero `cypress.config.*`, zero `*.cypress.*` test files, zero `cypress` in any `package.json`. Adopting Cypress means picking up infra no other team in `fe-global` runs. |
| **Storybook `play:` interaction tests + `@storybook/test-runner`** | The `fe-lib-visual-testing` README documents `play` as optional (`platform/fe-lib-visual-testing/README.md:7-16`) but **zero** `play:` functions exist in any story file across the monorepo, and `@storybook/test-runner` is not a declared dep. |
| **Playwright visual regression in CI today** | Infra exists but the Jenkins stage is **commented out** (`Jenkinsfile:366-383`); no committed baselines. Adopting now means platform work first — re-enabling the stage, seeding baselines, and signing up for baseline maintenance. Defer to Tier 4 / Phase 3.5 in [`frontend-testing.md`](./frontend-testing.md). |
| **Chromatic / Percy** | Not used. Adopting means sourcing budget and a vendor relationship the team does not currently have. |
| **`axios-mock-adapter`-only for new `fetch`/MSW paths** | Conflicts with the global `axiosMock` (`__tapas__/config/jest/jest.setup.js:40`); `member-management` had to `axiosMock.restore()` explicitly. Use MSW for new `channel-integrations/` work. `axios-mock-adapter` is acceptable only for legacy axios-only packages like `plancreate/fe-pnc-lib-api`, and `channel-integrations/` has no such legacy. |
| **Handlers that drift from consumer URLs** | `adpromotion` handlers use wildcard paths (`*/ad-promotion/...`) — they only stay correct because they're maintained alongside the components. `channel-integrations/fetchProfiles.jest.ts` already matches `*/app/social-network/fetch`; new handlers must track dashboard/ajax route changes or tests give false confidence. **This is the gap that the proposed mock-fidelity sub-tier closes.** |

# Summary

The `fe-global` integration story is **Jest + MSW**, split across two surfaces — `msw/node` `setupServer` in `*.jest.*` files (`adpromotion`, `billing`, `plancreate` data layer, `channel-integrations` multiselect, `member-management`) and `msw-storybook-addon` via per-story `parameters.msw.handlers` (`plancreate`, `member-management`, `unified-composer`, `adpromotion`, `channel-integrations` reauth modal, `enterprise-xp`) — all bootstrapped from `platform/fe-storybook/src/preview.tsx`. There is **no Cypress, no Storybook test-runner, no `play` interaction tests in practice, no Chromatic/Percy, and no consumer-side contract / Pact / hootpact validation anywhere**. Playwright visual regression exists as scaffolding (`platform/fe-lib-visual-testing`) with a commented-out Jenkins stage and one consuming namespace. CI's "Integration" stage runs Tapas-planned Jest only. For `channel-integrations/`, the strongest borrow path is to extend the in-namespace MSW patterns toward `adpromotion`'s shared-handler-module shape — and to be honest in [`design-doc.md`](./design-doc.md) and [`frontend-testing.md`](./frontend-testing.md) that the consumer-side mock-fidelity work is **net-new for FE at Hootsuite**, not a port of an existing pattern.

# Maintenance

This inventory is a one-time snapshot taken during CI-7194. We will not keep it current on a schedule. If a namespace ships a major new test pattern that the `channel-integrations/` work could benefit from, append a dated entry to the relevant section above and link the PR.
