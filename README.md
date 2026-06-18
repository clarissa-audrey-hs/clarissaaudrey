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
| [`frontend-testing.md`](./frontend-testing.md) | **Frontend** companion doc — channel-integrations FE testing strategy, MSW handler fidelity, jest/Storybook/Playwright surfaces, contingent on Hippogriff ownership confirmation. |
