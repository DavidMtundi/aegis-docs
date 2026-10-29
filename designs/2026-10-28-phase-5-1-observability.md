# Phase 5.1 — OpenTelemetry + correlation IDs

**Status:** Implemented  
**Date:** 2026-10-28

## Goal

Wire the already-referenced OpenTelemetry packages and propagate a stable correlation ID into logs, response headers, and audit events (roadmap 5.1).

## Design

1. **Correlation middleware** (early pipeline)
   - Read `X-Correlation-ID` (or `X-Request-ID`); else use `HttpContext.TraceIdentifier`.
   - Cap length to 100 (audit column).
   - Set `HttpContext.TraceIdentifier`, response header `X-Correlation-ID`, Serilog `CorrelationId` property, and `Activity.Current` tag `aegis.correlation_id`.

2. **OpenTelemetry** (API host)
   - `AddOpenTelemetry().WithTracing().WithMetrics()`
   - Instrument ASP.NET Core + HttpClient
   - Console exporter when `OpenTelemetry:ConsoleExporter` is true (default in Development)
   - OTLP exporter deferred (1.10.0 had a known advisory); add when bumping the OTel stack
   - Service name `aegis-api`

3. **Audit**
   - Controllers already pass `TraceIdentifier` — middleware makes that the correlation id, so existing audit writes pick it up with no mass controller edits.

4. **Out of scope**
   - Full APM vendor dashboards, custom business meters beyond ASP.NET defaults, changing every audit call site.
