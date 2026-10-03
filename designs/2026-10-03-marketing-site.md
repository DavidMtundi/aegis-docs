# Marketing site redesign (aegis-web)

Status: implemented 2026-10-03 on branch `marketing-redesign` in `aegis-web`. Plan: `plans/2026-10-03-marketing-site.md`.

## Why

A review of the marketing site (after the console workbench) found that it is visually polished but makes claims the product cannot back, and loses every lead.

1. Unbuilt features are sold as live. Screening has its own product page and an integration tile, but it is Phase 3 and the module is empty. "Backtest before go-live" and "analyst → manager approval" are Phase 4. ISO 20022 ingestion, on-prem deployment, webhooks and a sandbox appear in solution cards with nothing behind them.
2. Testimonials are attributed to anonymous organisations ("Financial Crime Team Lead, Digital bank (Kenya)"), and the same quote appears twice on the homepage. Stats such as "100% explainable alerts" and "Minutes to tune a rule" read as measured results; there are no customers yet.
3. The request-access form discards submissions (`onSubmit` only sets local state) and then tells the visitor a specialist will follow up.
4. The homepage stacks 14 sections (about ten screens at 1920 px). Trust strip, proof strip, impact stats, "How it works" and the pipeline repeat the same points.
5. The site uses a blue brand with Satoshi; the console uses a warm paper palette with Source Serif 4, IBM Plex Sans and a teal accent. Site and product look like different companies.
6. The hero carousel crops its own table (the amount column reads "AMOU…"). Resource cards advertise an "8 min read" guide that links to a product page. Footer Security, Privacy and Terms all link to `/#contact`.
7. Four components are unused: `Capabilities`, `GovernanceSection`, `ProblemSection`, `RecognitionSection`.

## Direction

One brand with the console, and only claims the product supports. Built features are presented as available; screening and rule simulation sit in a labelled roadmap section. Real console screenshots replace invented quotes and stats. The homepage has one job: get a compliance lead to request a pilot conversation.

## Design

### Visual system
- Tokens match `aegis-console/src/app/globals.css`: paper background `#f3f1ec`, ink `#1c2421`, muted `#5c6b64`, line `#d5ddd7`, panel `#fbfaf7`, accent `#0f6b5c` / strong `#0a5246` / soft `#d8efe8`, and the same soft radial background.
- Fonts via `next/font/google`, as in the console: Source Serif 4 (display), IBM Plex Sans (body), IBM Plex Mono (code). The Fontshare Satoshi stylesheet is removed.
- Existing utility classes in `globals.css` (`btn-primary`, `heading-lg`, `field-label`, etc.) keep their names and switch to the new tokens, so components change only where layout changes.
- A "Planned" chip style for roadmap items, using the muted palette rather than the accent.

### Homepage (14 sections → 7)
1. **Hero.** Headline, one-line value statement, "Request access" (primary) and "How it works" (secondary). A static framed screenshot of the real alert workspace replaces the carousel.
2. **The problem.** Three short points: false-positive load, alerts no one can explain, decisions without a trail. No numbers.
3. **Investigation flow.** Three real screenshots with one caption each: alert evidence and headline → case timeline → Customer 360.
4. **Governance.** Compliance-owned versioned rules (keeps the rule JSON panel from `FeatureTabs`), append-only audit trail, tenant isolation, role-based access. Built features only.
5. **Who it's for.** Banks, SACCOs, fintechs as compact cards linking to `/solutions/*`.
6. **On the roadmap.** Sanctions and PEP screening; rule simulation with analyst → manager approval. Each carries a "Planned" chip.
7. **Request access.** Short "what we'll cover" list and a button that opens a pre-filled email.

Removed from the homepage: testimonials, impact stats, proof strip, trust strip, integrations strip, resources, homepage FAQ. `/faq` stays in the nav.

### Other pages
- `/how-it-works` gains the savings calculator, unchanged in behaviour and still labelled "Illustrative — not a guarantee".
- `/platform/screening` stays reachable but is reframed as planned: "Planned" chip in the hero, copy in future tense, CTA "Ask about the screening roadmap".
- `/platform/rule-engine`: simulation, backtest and approval move to a "Planned" list; built items (declarative rules, versions, draft/activate, validation) stay.
- `/solutions/*`: drop ISO 20022, on-prem, webhooks and sandbox bullets unless the product supports them; replace with built capabilities.
- Nav: Screening in the Platform menu gets a "Planned" label.
- Footer: Security, Privacy and Terms links are removed until those pages exist. "Resources" link removed.

### Request access
- The form is replaced by a `mailto:` link to `access@aegis.example` (placeholder, to be swapped before launch). Subject "Aegis access request"; body pre-filled with prompts for name, institution, institution type and topics.
- A small select for institution type stays, so the email body includes it. No data leaves the browser except through the visitor's own mail client.
- The link builder lives in `lib/contact.ts` with a unit test.

### Screenshots
- Captured from the running console with the seeded smoke tenant at 1440 px and saved as optimised PNGs under `public/screens/` (alert workspace, case workspace, Customer 360).
- Seed data must look plausible and contain no real personal data.

### Code structure
- Delete unused components (`Capabilities`, `GovernanceSection`, `ProblemSection`, `RecognitionSection`) and sections removed from the homepage once nothing imports them.
- Content stays in `lib/` (`product-pages.ts`, `positioning.ts`, `site-nav.ts`). Roadmap items live in `lib/roadmap.ts` so the homepage, nav and product pages share one source.
- Add a `test` script using `node --test` for `lib/` helpers, matching the console.

### Verification
- `npm test`, `npm run build`, `npx eslint`.
- Screenshots in the Cursor browser at 1440 px and 390 px for home, how-it-works, one platform page, screening and one solution page.
- A claims check: grep for "backtest", "ISO 20022", "on-prem", "webhook", "sandbox", "100%" and testimonial markup returns only roadmap-labelled or removed content.

## Out of scope
A real lead-capture backend, blog or resources content, legal pages, analytics, dark mode, a shared token package between console and site.
