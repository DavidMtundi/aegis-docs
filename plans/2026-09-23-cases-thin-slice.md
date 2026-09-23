# S1 Cases Thin Slice — Implementation Plan

> Execute backend then console. Commit after each green test cycle.

**Goal:** Cases module + APIs + console so demo reaches disposition + audit.

## Backend tasks

1. Domain + ports in `Aegis.Modules.Cases`
2. EF config, DbSet, migration, CaseRepository, DI
3. CasesController + AlertsController.create-case
4. Integration test CaseWorkflowTests
5. Commit `feat: cases thin slice with create-from-alert and close`

## Console tasks

1. API helpers + types
2. Cases list/detail pages + nav
3. Alert detail Create case + actions
4. Commit `feat(console): cases investigation workflow`
