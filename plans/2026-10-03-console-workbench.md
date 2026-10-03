# Console workbench implementation plan

**Goal:** implement `designs/2026-10-03-console-workbench.md`.

**Architecture:** additive API fields resolved through one batch name lookup (no N+1); console layout rebuilt around a sidebar shell and two-column workspaces; all formatting and summary logic in pure tested modules.

**Tech stack:** .NET 9 / EF Core 9 / xUnit integration tests against Postgres; Next 16 / React 19 / Tailwind 4; `node --test` for console libs.

## Global constraints
- No new npm or NuGet packages.
- Existing response fields keep their names and meaning.
- Each task ends green: `dotnet test` (backend) or `npm test && npm run build && npx eslint src` (console), then one commit.
- Branch `console-workbench` in `aegis` and `aegis-console`.

## Tasks

### 1. API: names on alerts, queue views, SLA settings
- Create `Aegis.Application/Lookups/IDisplayNameLookup.cs`: `Task<IReadOnlyDictionary<Guid, CustomerLabel>> CustomersAsync(TenantId, IEnumerable<Guid>, ct)` with `record CustomerLabel(Guid Id, string Name, string Country, string Type)`, and `Task<IReadOnlyDictionary<Guid, string>> UserNamesAsync(TenantId, IEnumerable<Guid>, ct)`. EF implementation in `Infrastructure/Lookups/DisplayNameLookup.cs`; customer name = legal name or "first last".
- `AlertsController`: `AlertResponse` gains `CustomerName`, `CustomerCountry`, `AssigneeName` (nullable). List and every single-alert response use the lookup.
- `GET /alerts`: `view` (`open` = not RESOLVED/DISMISSED/CLOSED; `mine` = open and assigned to caller; `unassigned` = open and unassigned; `pastSla` = open and triggered before now − alert SLA) and `customerId`. Implemented in `AlertListQuery` and `AlertRepository.ListByTenantAsync`.
- `SettingsController`: `GET /settings/sla` → `{ alertSlaDays, caseSlaDays }`, any authenticated user.
- Tests (`tests/Integration/Alerts/AlertQueueTests.cs`): names present on list and detail; each view returns the right alerts; unknown view → 400; SLA endpoint returns defaults.

### 2. API: names on cases, transactions, customer risk band
- `CaseResponse` gains `CustomerName`, `AssigneeName`, `LinkedAlerts` (`id, ruleName, severity, status`), resolved in one query per response set.
- Transaction list/detail items gain `CustomerName`.
- Customer list items gain `RiskBand` (latest `customer_risk_scores` row per customer, DISTINCT ON).
- Tests: extend case, transaction and customer integration tests with the new fields.

### 3. Console: formatting and headline libs
- `lib/format/money.ts` `formatMoney(amount: number | string, currency: string): string` → `KES 95,000.00`.
- `lib/format/time.ts` `relativeAge(iso, now): string` (`45m`, `5h`, `3d`), `slaState(iso, slaDays, now): "ok" | "due-soon" | "breached"` (due-soon = last 20 % of the window).
- `lib/format/labels.ts` `statusLabel("IN_REVIEW") → "In review"`.
- `lib/alerts/headline.ts` `alertHeadline(evaluatedValues, currency?): string | null` using `transaction_count*`, `transaction_sum*`, `max_single_amount*` and the window suffix (`_24h`, `_1h`, generic → "in the rule window").
- Tests beside each file. Types updated in `lib/api/types.ts` for task 1–2 fields.

### 4. Console: shell
- `components/shell/Sidebar.tsx` (grouped links, open-count badges from `GET /dashboard`), `TopBar.tsx` (jump box, user menu, mobile menu button), `ShellSkeleton.tsx`; `(app)/layout.tsx` becomes sidebar + main; `RequireAuth` renders the skeleton while hydrating. `AppNav.tsx` is removed.
- Nav groups defined in `lib/nav.ts` with a test that every link has a permission or is public.

### 5. Console: alert queue
- `components/ui/Tabs.tsx`, `components/ui/StatusChip.tsx`; `alerts/page.tsx` uses `view` + `severity` from the URL; `AlertTable` columns per design; SLA from `/settings/sla` via `lib/api/settings.ts`.

### 6. Console: alert workspace
- `components/customers/CustomerCard.tsx` (overview + risk fetch); `alerts/[id]/page.tsx` two-column layout; headline; `EvidencePanel` met marker becomes a neutral check; related transactions table (fetch by id); technical details collapsible.

### 7. Console: case workspace
- `cases/[id]/page.tsx` two-column; note composer on top; `CaseTimeline` newest first, compact rows, emphasis for notes/escalations/decisions; right rail with actions, `CustomerCard`, linked alerts, SLA.

### 8. Console: remaining screens
- Customers list risk band column; customer page header strip; transactions formatting, names, customer name filter (resolves via `listCustomers(q)`); login split layout; dashboard/risk/rules/audit/users verified inside the shell.

### 9. Docs and verification
- Browser walkthrough at 1440px and 1024px; update roadmap notes; commit docs.
