# Aegis Docs

Product requirements and technical architecture for the **Aegis AML Compliance Platform**.

| Repository | Purpose |
|---|---|
| [`aegis`](https://github.com/DavidMtundi/aegis) | Core platform (backend) |
| [`aegis-web`](https://github.com/DavidMtundi/aegis-web) | Marketing website |
| **`aegis-docs`** (this repo) | PRD & architecture specification |

## Documents

- [`product-requirements-and-architecture.md`](./product-requirements-and-architecture.md) — Engineering build specification (v1.0)
- [`designs/2026-09-23-vertical-slice-ingest-structuring-alert-audit.md`](./designs/2026-09-23-vertical-slice-ingest-structuring-alert-audit.md) — **APPROVED** first vertical-slice design baseline
- [`plans/2026-09-23-vertical-slice-pr1-pr5.md`](./plans/2026-09-23-vertical-slice-pr1-pr5.md) — Implementation plan (PR1–PR5)
- [`plans/2026-10-01-aegis-build-roadmap.md`](./plans/2026-10-01-aegis-build-roadmap.md) — **Current roadmap**: audit of all repos and phased build plan to MVP and beyond
- [`plans/2026-10-01-phase-0-hardening.md`](./plans/2026-10-01-phase-0-hardening.md) — Phase 0: toolchain and security hardening (detailed tasks)
- [`research/2026-10-01-sumsub-kyc-in-2026.md`](./research/2026-10-01-sumsub-kyc-in-2026.md) — Industry research: Sumsub *KYC in 2026* report (credited), plus how it maps to Aegis

## How to use this repo

- Product, compliance, and engineering align here before changing scope in `aegis`.
- Architecture decisions that affect code should still get ADRs in `aegis/docs/decisions/`.
- Marketing copy lives in `aegis-web`; do not treat this repo as the public site.
