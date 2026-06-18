# Frontend Testing — channel-integrations FE (CI-7194)

> **Status:** DRAFT | **Owner:** Clarissa Audrey | **Jira:** [CI-7194](https://hootsuite.atlassian.net/browse/CI-7194) | **Last updated:** 2026-06-18

Companion to [`design-doc.md`](./design-doc.md) and [`phased-plan.md`](./phased-plan.md). The four-tier pyramid, decisions D1–D7, principles, and triggers in those docs all carry forward; this page lists the FE deltas.

# Scope

| Surface | What it is | Ownership | Source |
| --- | --- | --- | --- |
| `fe-global/channel-integrations/` (**24 team-owned `fe-chan-*` packages**; 27 in the namespace total, 3 carved out to `@hootsuite/frontend-platform`) | React + JS/TS mix + styled-components 5/6 + RTK + Bento `@ds/*`, jest with `*.jest.*` colocated tests, MSW for API tests in two top packages, Storybook in most UI packages | **Team-owned** | `.github/CODEOWNERS:28`: `#channel-integrations/ @hootsuite/ci-hippogriff` (disabled); per-package overrides at `.github/CODEOWNERS:67-69` for the 3 carve-outs. |

## Team-owned packages

The 24 in-scope packages, by shape:

| Category | Count | Packages |
| --- | --- | --- |
| Apps (mounted UIs) | 2 | `fe-chan-ig-auth-app`, `fe-chan-multiselect-auth-app` |
| Libraries (no UI) | 2 | `fe-chan-lib-constants`, `fe-chan-lib-darklaunch` |
| Modals / dialogs | 15 | `fe-chan-auth-success-modal`, `fe-chan-comp-add-social-account-modal`, `fe-chan-comp-auth-focus-modal`, `fe-chan-comp-auth-wizard-modal`, `fe-chan-comp-bluesky-connect-modal`, `fe-chan-comp-ig-auth-success-modal`, `fe-chan-comp-igb-auth-process-modal`, `fe-chan-comp-igb-steal-downgrade-error-modal`, `fe-chan-comp-permissions-modal`, `fe-chan-comp-reauth-modal`, `fe-chan-comp-reddit-disclaimer-modal`, `fe-chan-comp-select-instagram-type-modal`, `fe-chan-comp-tiktokbusiness-modal`, `fe-chan-comp-tiktokbusiness-paywall-modal`, `fe-chan-remove-confirmation-modal` |
| Other components | 5 | `fe-chan-comp-i18n-context-provider`, `fe-chan-comp-permissions-messaging`, `fe-chan-comp-pii-mask`, `fe-chan-comp-profile-warning-banner`, `fe-chan-comp-question-select` |

# Current state

| Aspect | Today |
| --- | --- |
| Stack | React + JS/TS mix + styled-components 5/6 + RTK + Bento `@ds/*` |
| Tier 1 | jest with `*.jest.*` colocated tests; MSW for API tests in two top packages (`fe-chan-comp-reauth-modal`, `fe-chan-multiselect-auth-app`); plain `jest.mock` elsewhere |
| Storybook | Present in most UI packages |
| Tier 2 (component / integration) | None standardised; per-package fan-out |
| Tier 3 (cross-repo) | None — Playwright is not wired per-package; `platform/fe-lib-visual-testing` exists but not in this namespace's CI |
| Tier 4 (post-deploy) | None |
| BE coupling | `fe-chan-comp-reauth-modal` calls `/service/channel-auth-audit` and `/service/social-network-error-inspection`; `fe-chan-multiselect-auth-app` calls `/app/social-network/fetch` and `/ajax/instagrambusiness/get-refreshed-social-profiles` |
| Hardest gap | Mock fidelity not enforced; per-package fan-out across 24 packages with mixed test conventions (MSW vs `jest.mock`) and dual legacy/RTK codepaths in `fe-chan-multiselect-auth-app` |

# Per-tier mapping

## Tier 1 — fast unit + MSW handler fidelity

* **New tier-1 cell — MSW handler fidelity:** a backend OpenAPI change must invalidate FE MSW handlers in the same PR. Concretely:
  * Specmatic-validate `fe-chan-comp-reauth-modal`'s MSW handlers against `channel-auth-audit`'s OpenAPI and SNEI's OpenAPI.
  * Specmatic-validate `fe-chan-multiselect-auth-app`'s handlers against `dashboard-som`'s OpenAPI.
  * **This is the FE-side analogue of the consumer mock-fidelity sub-tier in [`design-doc.md`](./design-doc.md).**
* **Standardize on jest + MSW.** Today some packages use MSW (multiselect, reauth-modal), others use plain `jest.mock`. Pick one — MSW is the better bet because it lets us share fixtures with the backend Tier 1 mock-fidelity tests.
* **Storybook MSW addon as canonical pattern** for visually-verifiable component states.

## Tier 2 — in-package integration

For FE, "in-repo integration" means **component tests against MSW that share fixtures with backend Tier 1**. Two work items:

1. **Standardize on Storybook + MSW addon** for all team-owned `fe-chan-*` packages so Tier 2 can be a single command per package.
2. **Shared fixture format** between FE MSW handlers and backend Tier 1 mock-fidelity tests, so a backend OpenAPI change invalidates both in the same PR.

Heavier lift than backend because of the per-package fan-out (24 team-owned packages) and the legacy/RTK dual codepaths in `fe-chan-multiselect-auth-app`.

## Tier 3 — cross-repo harness (FE side)

The Phase 3 harness in `channel-auth-tool` is backend-only. Adding FE Playwright on top is **deferred to a Phase 3.5** unless the team has an immediate FE regression risk.

If revisited:

* Add a `playwright/` suite to `channel-auth-tool` that boots a host shell rendering the team's `fe-chan-*` packages and drives them against the backend Compose stack.
* Verifies UI reauth flow round-trip end-to-end against the harness — including the disconnect-event consumers (per **D7** in [`design-doc.md`](./design-doc.md)) so the FE sees the post-disconnect state correctly.

## Tier 4 — post-deploy smoke

No FE Tier 4 today. Two options:

* **Visual regression** via `platform/fe-lib-visual-testing` Playwright in CI for the 24 team-owned packages. Snapshot the Storybook for each `fe-chan-*` UI component on dev deploys.
* **Synthetic UI flow** against staging dashboard exercising real reauth or multiselect modals. Heavier; defer until visual regression is in place.

# Trigger matrix delta

The [Triggers and cadence section in `design-doc.md`](./design-doc.md#triggers-and-cadence) applies to backend services. FE deltas:

* **No weekly cron equivalent for FE.** The backend services use `gitOpsPipeline.cron = 'H H * * H'`; FE has nothing comparable.
* **Tier 4 cadence for FE would need to be wired manually** (e.g., a `pipelineTriggers([cron(...)])` block on a future visual-regression Jenkinsfile, mirroring `dashboard/localPhpImage.Jenkinsfile:7`).

# Sequencing

If leadership wants the FE work to land:

1. **MSW handler fidelity (Tier 1, new)** for `fe-chan-comp-reauth-modal` and `fe-chan-multiselect-auth-app` first — the two packages already use MSW, so the marginal cost is just the Specmatic gate.
2. **Storybook + MSW addon standardization** across the rest of the 24 team-owned packages, prioritising the 15 modal/dialog packages (high reuse, similar shape) before the long tail.
3. **Visual regression (Tier 4)** once Tier 1 fidelity is in place and a deploy cadence is wired.
4. **FE Playwright in the harness (Phase 3.5)** only if the disconnect/reauth UI starts breaking in ways Tier 1+2 cannot catch.

OQ-FE-1 (CODEOWNERS hygiene) can land in parallel at any time — it is not a blocker for the work above.

# Open questions

## OQ-FE-1 — Shared fixture format with backend Tier 1

If the team commits to MSW + jest as the FE Tier 1 standard, the FE MSW handlers and the backend mock-fidelity tests should share a single fixture format so that a backend OpenAPI change invalidates both in the same PR. Concrete decision needed: where do shared fixtures live, and what generates them? Options: hand-written JSON in IDL-central next to each spec; generated from the OpenAPI examples; generated from VCR-style recordings against staging.

## OQ-FE-2 — FE Playwright vs visual regression

When Tier 4 is on the roadmap, decide between:

* **Visual regression** (Storybook snapshots via `platform/fe-lib-visual-testing`) — cheaper, already-integrated infra, but only catches rendering bugs.
* **Synthetic UI flow** against staging dashboard — heavier, but catches regressions in real user flows (reauth modal, multiselect).

Default proposal: visual regression first, synthetic later only if a real production incident motivates it.

# Out of scope

* Backend testing (covered in [`design-doc.md`](./design-doc.md) and [`phased-plan.md`](./phased-plan.md)).
* Visual-regression / Playwright infrastructure design beyond pointing at `platform/fe-lib-visual-testing`.
* These are explicitly out of scope for any FE testing work this design proposes, and are listed here only so future readers do not relitigate ownership:
  * `fe-chan-comp-social-profile-pill` — owned by `@hootsuite/frontend-platform`
  * `fe-chan-comp-a11y-dialog` — owned by `@hootsuite/frontend-platform`
  * `fe-chan-comp-generic-dialog` — owned by `@hootsuite/frontend-platform`
  * `fe-ae-comp-social-profile-avatar` — owned by `@hootsuite/global-adespresso`
  * `fe-pnc-comp-grouped-profiles` — owned by `@hootsuite/global-plancreate`
