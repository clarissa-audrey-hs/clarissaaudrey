# Agentic-First Integration Testing — CI-7194

Design doc and phased plan for the agentic-first integration testing initiative tracked in [CI-7194](https://hootsuite.atlassian.net/browse/CI-7194).

> **Status:** DRAFT  
> **Owner:** Clarissa Audrey  
> **Last updated:** 2026-06-18

## Pages

| Page | What it covers |
| --- | --- |
| [`design-doc.md`](./design-doc.md) | **Backend** design doc — the four-tier test pyramid across the team's five backend services (`channel-auth`, `social-network-auth`, `service-core`, `service-social-profiles`, `social-network-error-inspection`) and the cross-repo harness in `channel-auth-tool`. Decisions D1–D7, principles, integration-vs-contract matrix, triggers and cadence. |
| [`phased-plan.md`](./phased-plan.md) | **Backend** phased plan — Phases 1 through 5, sequencing, critical-path ordering, per-service work items, open questions, out-of-scope. |
| [`frontend-testing.md`](./frontend-testing.md) | **Frontend** companion doc — channel-integrations FE testing strategy, MSW handler fidelity, jest/Storybook/Playwright surfaces, across the 24 team-owned `fe-chan-*` packages. |
| [`services-inventory.md`](./services-inventory.md) | **Backend inventory** — what other Hootsuite services have for integration and contract testing today. Sampled SCUM, SNP, dashboard. Patterns to borrow, patterns to avoid, gaps our plan addresses. |
| [`frontend-inventory.md`](./frontend-inventory.md) | **Frontend inventory** — what other `fe-global` namespaces have for integration and contract testing today. Audited 75 namespaces; covers `adpromotion`, `plancreate`, `member-management`, `unified-composer`, `platform`, and the smaller MSW adopters. Confirms zero contract / Pact / Cypress / Storybook test-runner usage anywhere in `fe-global`. |
