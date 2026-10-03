# Console workbench redesign

Status: approved 2026-10-03. Plan: `plans/2026-10-03-console-workbench.md`.

## Why

A screen-by-screen review of the console (after Phase 2) found that the visual base is sound (warm paper palette, Source Serif headings, IBM Plex body, teal accent) but the information design works against analysts:

1. Customers, linked alerts and related transactions appear as raw UUIDs on the alert queue, alert page, case page and transactions list.
2. The alert page leads with system metadata (deduplication key, ids). Satisfied conditions are marked "Met" in red, which reads as an error. Related transactions are a bare id list. There is no plain-language summary of why the alert fired.
3. The queue has no customer name, age, SLA state or assignee; statuses are raw enums; filters need an Apply click; the transactions customer filter only takes a UUID.
4. The case page is one column of timeline cards, oldest first, with actions and context out of view.
5. The top nav overflows to two rows at laptop width; a full-screen "Checking session…" flashes on every page load; content is held to a narrow column.
6. Amounts render as `95000.00 KES`; date formats are inconsistent.

## Direction

An analyst workbench. Keep the palette and fonts; change layout and information hierarchy.

## Design

### Shell
- Left sidebar, grouped by job: Overview (Dashboard), Work (Alerts, Cases with open counts), Records (Customers, Transactions), Govern (Rules, Audit), Admin (Users, Risk model). Links follow the existing permission mirror. Tenant and signed-in user sit at the bottom.
- Below 1024px the sidebar collapses to a top bar with a menu button.
- Top bar: customer jump box (submits to `/customers?q=`) and a user menu with role and Log out.
- Session check renders a shell skeleton instead of a blank page.
- Content width grows to 1440px. Pages keep their `PageHeader` back links; there is no separate breadcrumb bar.

### Alert queue
- Columns: severity, rule, customer (name and country, linked), age (turns red past the alert SLA), status chip with readable label, assignee name, triggered time.
- Quick views as tabs: Open, Mine, Unassigned, Past SLA, All. Severity filter applies on change. View and severity live in the URL.
- The whole row is a link target.

### Alert workspace
- Left: headline summary built from the evidence (for example "5 transactions totalling KES 475,000 in 24 h"), then the condition table with a neutral check for met conditions, then related transactions with time, direction and formatted amount.
- Right rail (sticky on wide screens): primary action (Create case), secondary actions; customer card (name, country, type, risk score and band, open alert and case counts, link); assignment and SLA state.
- Technical details (alert id, rule version, deduplication key, raw JSON, all evaluated features) collapse under one toggle.

### Case workspace
- Left: note composer first, then the timeline newest first as compact rows. Notes, escalations and decisions stand out; system events are muted.
- Right rail: status, priority, assignee and actions; customer card; linked alerts by rule name, severity and status; case age against the case SLA.

### Other screens
- Customers list: risk band column. Customer page: header strip with risk band.
- Transactions: formatted amounts, customer names, customer filter by name.
- Login: split layout with product context on one side.
- Dashboard, risk model, rules, audit, users: move into the new shell and shared components without layout changes.

### API additions (additive only)
- Alert responses gain `customerName`, `customerCountry`, `assigneeName`.
- `GET /alerts` gains `view=open|mine|unassigned|pastSla` and `customerId`.
- `GET /settings/sla` returns `{ alertSlaDays, caseSlaDays }` (from the existing dashboard options) for any signed-in user.
- Case responses gain `customerName`, `assigneeName`, `linkedAlerts: [{ id, ruleName, severity, status }]`.
- Transaction list and detail responses gain `customerName`.
- Customer list responses gain `riskBand` (latest score, null when unscored).

### Console code structure
- Pure, tested helpers in `lib/format/` (money, relative age, SLA state, status labels) and `lib/alerts/headline.ts`.
- New components: `components/shell/{Sidebar,TopBar,ShellSkeleton}`, `components/customers/CustomerCard`, `components/ui/{StatusChip,Tabs}`.

## Out of scope
Real-time updates, saved views, dark mode, keyboard shortcuts, the marketing site (`aegis-web`, reviewed after the console).
