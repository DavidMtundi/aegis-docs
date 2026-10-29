# Console UX polish (full pass)

**Status:** Implemented  
**Date:** 2026-10-29  
**Approach:** Thin vertical slices: Rules → Dashboard/queues → shared system → remaining screens

## Direction
Keep warm paper / teal / Source Serif + IBM Plex. Improve hierarchy, readable statuses, calmer density, obvious primary actions, light motion. No brand/palette reset.

## Slice 1 — Rules
- Status/severity chips (human labels, not raw enums)
- Approvals queue as a work list with maker ≠ checker clarity
- Rule detail: clearer lifecycle actions, draft editor framed as a form section, simulation results as a summary card
- Rules list: filters/tabs by status if cheap; whole-row affordance

## Slice 2 — Dashboard + queues
- Dashboard: “start here” strip (open past SLA, unassigned alerts, pending screening) + existing stats
- Alert/case queues: consistent chips, denser headers, clearer empty states

## Slice 3 — Design system
- Shared `StatusChip` expansions, table row hover, empty/loading polish, button focus rings, subtle enter animation utility

## Slice 4 — Remaining
- Customers, transactions, screening, admin, login micro-polish for consistency
