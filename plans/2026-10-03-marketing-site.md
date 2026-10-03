# Marketing site implementation plan

**Goal:** implement `designs/2026-10-03-marketing-site.md`.

**Architecture:** keep the Next static export and the existing component set. Swap tokens and fonts centrally in `globals.css` and `layout.tsx`; restructure the homepage; move every claim into `lib/` content files that can be checked; roadmap items come from one module.

**Tech stack:** Next 16.3.5 (static export), React 19, Tailwind 4; `node --test` for `lib/` helpers.

## Global constraints
- No new npm packages. Fonts via `next/font/google`, as in the console.
- Only claims the product supports today (roadmap Phases 0–2 and the console workbench). Anything later is labelled "Planned" or removed.
- No invented customers, quotes or metrics.
- Each task ends green: `npm test && npm run build && npx eslint .`, then one commit.
- Branch `marketing-redesign` in `aegis-web`.

## Built today (source for copy)
Transaction ingest (single, JSON batch, CSV import) with idempotency; three seeded scenarios (structuring, rapid movement / pass-through, high-risk geography) evaluated over each version's lookback; rule versions with draft and activate; alerts with condition-level evidence, plain-language headline, assign, escalate, dismiss with reason; cases from alerts with linked alerts, notes, timeline, escalation, assignment and close with disposition; Customer 360; customer risk scoring with a versioned, weighted model and score history; dashboard with SLA tracking; four built-in roles; filterable append-only audit log; per-tenant isolation; multi-currency accounts.

Not built: screening, new rule authoring, simulation/backtest, approval workflow, webhooks, sandbox, ISO 20022 mapping, on-prem packaging, exports, network/UBO analysis, checklists.

## Tasks

### 1. Test harness and contact link
- `package.json`: `"test": "node --test lib/**/*.test.ts"` (Node 22+ strips types; same approach as the console). `tsconfig.json`: `allowImportingTsExtensions: true`.
- `lib/contact.ts`: `CONTACT_EMAIL = 'access@aegis.example'`, `INSTITUTION_TYPES`, `accessRequestMailto({ institutionType?, topics? }): string` → `mailto:` with encoded subject "Aegis access request" and a body with prompts (name, institution, institution type, topics).
- `lib/contact.test.ts`: encodes spaces and newlines; includes the chosen type; works with no input.

### 2. Roadmap module
- `lib/roadmap.ts`: `roadmapItems: { id: 'screening' | 'simulation'; title; summary; points[] }[]` and `isPlanned(href)` for nav/product pages (`/platform/screening`).
- `lib/roadmap.test.ts`: screening is planned; transaction monitoring is not.

### 3. Visual system
- `app/layout.tsx`: Source Serif 4 (`--font-display`), IBM Plex Sans (`--font-body`), IBM Plex Mono (`--font-mono`); drop the Fontshare link and `font-satoshi`.
- `app/globals.css`: map existing variables to console tokens (`--bg` #f3f1ec, `--surface`/`--bg-elevated` #fbfaf7, `--bg-soft` #ebe8e0, `--text` #1c2421, `--text-muted` #5c6b64, `--border` #d5ddd7, `--brand` #0f6b5c, `--brand-hover` #0a5246, `--brand-soft` #d8efe8), console radial background, `--font-sans: var(--font-body)`; update hard-coded blue rgba values; add `.chip-planned`.
- `components/icons/AegisMark.tsx`, `app/icon.svg`: teal.

### 4. Real screenshots
- Capture from the running console (smoke tenant, 1440 px): alert workspace, case workspace, Customer 360. Save to `public/screens/{alert,case,customer}.png` (≤ 300 KB each, `sips` resize if needed).
- `components/ScreenFrame.tsx`: framed `<img>` with browser-chrome bar, `alt`, explicit width/height (static export has no image optimiser).

### 5. Homepage restructure
- `app/page.tsx`: `Hero`, `ProblemSection`, `InvestigationFlow` (new), `GovernanceSection`, `ForYourInstitution`, `RoadmapSection` (new), `CtaSection`.
- `Hero`: static `ScreenFrame` with the alert screenshot; CTAs "Request access" and "How it works". Remove the carousel.
- `ProblemSection` (reuse the unused file): three problem points, no numbers.
- `InvestigationFlow`: three steps, each a `ScreenFrame` and caption.
- `GovernanceSection` (reuse): versioned rules with the real `RAPID_MOVEMENT_001` definition (fields `credit_sum_1h`, `debit_sum_1h`, `pass_through_ratio_1h`, lookback `1h`), audit log, roles, tenant isolation. Move `RulesPanel` here from `FeatureTabs`.
- `ForYourInstitution`: built-capability bullets only.
- `RoadmapSection`: renders `roadmapItems` with `.chip-planned`.

### 6. Request access
- `CtaSection`: institution-type select and topics textarea build an `accessRequestMailto` link; the button is an `<a href>`. Remove name/email fields, fake submitted state and the "specialist will follow up" message. Note under the button: "Opens your email app. Prefer to write directly? access@aegis.example".

### 7. Content truth pass
- `lib/product-pages.ts`: proof stats become capability facts (no "100%", "Minutes", "Faster"); remove or move to "Planned" every item in "Not built". Rule engine: built items (declarative definitions, versions, draft/activate, validation) plus a planned simulation/approval note. Screening page: future tense.
- `ProductPage`: "Planned" chip in the hero when `isPlanned`; screening CTA "Ask about the screening roadmap".
- `lib/site-nav.ts` and `Navbar`: "Planned" label on Screening.
- `HowItWorks`: "Enrich" step drops screening; "Decide" step wording matches built close dispositions.
- `FaqSection` content: same check.
- `lib/positioning.ts`: hero copy per design.

### 8. How-it-works and footer
- `/how-it-works`: add `SavingsCalculator` below the pipeline, keeping the illustrative note.
- `Footer`: remove Security, Privacy, Terms and Resources links; "Request access" → `/#contact`.

### 9. Remove dead code
- Delete `Capabilities`, `RecognitionSection`, `TrustStrip`, `ImpactSection`, `IntegrationsStrip`, `ProofStrip`, `TestimonialsSection`, `ResourcesSection`, `AudienceStrip`, `FeatureTabs` once nothing imports them. `FaqSection` stays for `/faq`.

### 10. Verification and docs
- Claims check: `rg -i "backtest|iso 20022|on-prem|webhook|sandbox|100%|testimonial|8 min read"` in `app components lib` returns only roadmap-labelled text.
- Browser at 1440 px and 390 px: home, how-it-works, transaction-monitoring, screening, banks, faq. No horizontal overflow.
- Roadmap: marketing-site section updated. Design status → implemented.
