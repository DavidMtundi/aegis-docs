# Aegis

## Product Requirements & Technical Architecture Specification

**Document Status:** Engineering Build Specification  
**Version:** 1.0  
**Product:** Aegis AML Compliance Platform  
**Primary Repository:** `aegis`  
**Marketing Repository:** `aegis-web`  
**Target Market:** Banks, SACCOs, fintechs, payment providers, digital lenders and other regulated financial institutions, initially focused on Kenya/Africa.

---

# 1. Executive Summary

Aegis is a configurable financial-crime compliance platform designed to help financial institutions:

* onboard and manage customers,
* perform KYC/KYB,
* screen customers and entities,
* assess customer risk,
* monitor transactions,
* detect suspicious activity,
* generate explainable alerts,
* investigate alerts,
* manage compliance cases,
* maintain complete audit trails,
* configure compliance rules without code changes, and
* eventually support regulatory reporting and AI-assisted investigations.

Aegis must be designed as a **compliance operating platform**, not merely an AML rules engine.

The central product workflow is:

```text
Customer / Transaction
        ↓
Data ingestion
        ↓
Normalization
        ↓
Feature calculation
        ↓
Risk evaluation
        ↓
Configurable rule evaluation
        ↓
Alert
        ↓
Investigation
        ↓
Case
        ↓
Compliance decision
        ↓
Audit / Reporting
```

The MVP must prove this workflow end-to-end before expanding into the full product vision.

---

# 2. Product Vision

## 2.1 Vision

> Aegis should become the compliance operating system for modern African financial institutions.

The long-term platform should provide:

```text
                    AEGIS
                      │
       ┌──────────────┼──────────────┐
       │              │              │
      KYC            Risk        Screening
       │              │              │
       └──────────────┼──────────────┘
                      │
              Transaction Monitoring
                      │
                      ▼
                   Alerts
                      │
                      ▼
              Case Management
                      │
                      ▼
               Investigation
                      │
                      ▼
             Regulatory Reporting
                      │
                      ▼
                  Audit
```

---

# 3. Product Principles

The following principles are mandatory architectural constraints.

## 3.1 Configuration over hardcoding

Compliance policies must not be hardcoded into application logic.

Compliance users must eventually be able to configure:

* thresholds,
* amounts,
* transaction counts,
* time windows,
* transaction types,
* risk levels,
* jurisdictions,
* escalation rules,
* alert severity,
* SLAs,
* rule activation/deactivation,
* effective dates.

Changing a compliance rule should not require a software deployment.

---

## 3.2 Explainability over black-box detection

Every alert must be explainable.

The system must be able to answer:

> Why was this alert generated?

For example:

```text
Rule: STRUCTURING_001

Reason:
8 transactions were received within 24 hours.

Total:
KES 790,000

7 transactions were below the configured threshold
of KES 100,000.

Customer:
CUST-10293
```

Do not create unexplained risk decisions.

---

## 3.3 Auditability by default

Every significant compliance action must be auditable.

At minimum:

```text
WHO
WHAT
WHEN
BEFORE
AFTER
REASON
```

This includes:

* rule changes,
* risk model changes,
* case changes,
* alert disposition,
* customer changes,
* screening decisions,
* permissions,
* configuration changes.

---

## 3.4 Multi-tenancy from day one

Aegis is a SaaS platform.

Tenant isolation must therefore be implemented from the beginning.

Every tenant-owned domain object must have a `tenant_id`.

Do not build a single-tenant system and retrofit tenancy later.

---

## 3.5 Human-in-the-loop compliance

Aegis assists compliance officers.

It should not silently make consequential compliance decisions on behalf of the institution.

The platform must preserve:

* evidence,
* triggered rules,
* scores,
* analyst decisions,
* approvals,
* overrides,
* reasoning,
* audit history.

---

## 3.6 Deterministic core

The deterministic compliance engine is the source of truth.

Future AI functionality must operate alongside the evidence and rules rather than replacing them.

---

# 4. Product Scope

## 4.1 Core domains

Aegis will eventually contain:

1. Tenant Management
2. Identity & Access Management
3. Customer Management
4. KYC
5. KYB
6. Screening
7. Risk Management
8. Account Management
9. Transaction Management
10. Feature Engine
11. AML Rule Engine
12. Alert Management
13. Case Management
14. Investigation
15. Audit
16. Reporting
17. Integrations
18. Notifications
19. AI Investigation Assistant

---

# 5. MVP Scope

The MVP should deliberately be smaller.

## MVP modules

### Required

* Tenant management
* Authentication
* RBAC
* Customer
* Accounts
* Transactions
* Transaction ingestion
* Feature calculation
* AML rules
* Alert generation
* Alert investigation
* Case management
* Risk score foundation
* Audit trail
* Basic compliance dashboard
* Rule configuration

### Initial AML scenarios

Implement three scenarios first:

1. Structuring
2. Rapid movement / pass-through
3. High-risk geography

The remaining scenarios should be designed as extensible rule definitions but do not need to be production-ready in MVP.

---

# 6. MVP Vertical Slice

The MVP must demonstrate:

```text
Tenant
 ↓
Customer
 ↓
Account
 ↓
Transaction
 ↓
Feature calculation
 ↓
Rule evaluation
 ↓
Alert
 ↓
Investigation
 ↓
Case
 ↓
Disposition
 ↓
Audit
```

Example:

A transaction is imported:

```text
Customer: CUST-001
Account: ACC-001
Amount: KES 95,000
Type: CREDIT
```

Additional transactions are imported.

The engine determines:

```text
7 transactions
within 24 hours
total = KES 680,000
```

A configured rule triggers.

Aegis creates:

```text
ALERT-001
```

The investigator opens the alert and sees:

* customer,
* transactions,
* triggered rule,
* rule version,
* evidence,
* risk,
* related accounts,
* timeline.

The investigator can then:

```text
Dismiss
Escalate
Create Case
```

The resulting decision is audited.

This is the minimum definition of a functioning AML product.

---

# 7. High-Level Architecture

Aegis should begin as a **modular monolith**.

Do not prematurely split every module into microservices.

Recommended structure:

```text
aegis/
│
├── src/
│   ├── identity/
│   ├── tenancy/
│   ├── customers/
│   ├── kyc/
│   ├── kyb/
│   ├── accounts/
│   ├── transactions/
│   ├── features/
│   ├── rules/
│   ├── risk/
│   ├── screening/
│   ├── alerts/
│   ├── cases/
│   ├── investigations/
│   ├── audit/
│   ├── reporting/
│   ├── integrations/
│   └── shared/
│
├── tests/
├── migrations/
├── docs/
└── infrastructure/
```

Each module must have clear boundaries.

Avoid:

```text
controllers → arbitrary database access
```

Prefer:

```text
Controller
   ↓
Application Service
   ↓
Domain
   ↓
Repository
   ↓
Database
```

---

# 8. Architectural Layers

Each module should follow:

```text
API
 │
 ├── Controllers
 ├── DTOs
 └── Request validation
       ↓
Application
 │
 ├── Use cases
 ├── Commands
 └── Queries
       ↓
Domain
 │
 ├── Entities
 ├── Value objects
 ├── Domain services
 └── Policies
       ↓
Infrastructure
 │
 ├── PostgreSQL
 ├── Redis
 ├── Event bus
 └── External APIs
```

The domain layer should not depend directly on HTTP, PostgreSQL, Redis, or external providers.

---

# 9. Core Domain Model

## 9.1 Tenant

```text
Tenant
- id
- name
- legal_name
- type
- country
- status
- created_at
- updated_at
```

All tenant-owned records must reference the tenant.

---

# 10. Users and RBAC

## User

```text
User
- id
- tenant_id
- email
- name
- status
- last_login_at
- created_at
```

## Role

```text
Role
- id
- tenant_id
- name
```

## Permission

```text
Permission
- id
- code
- description
```

Examples:

```text
customer.read
customer.write

transaction.read

alert.read
alert.assign
alert.dismiss

case.read
case.create
case.update
case.close

rule.read
rule.create
rule.update
rule.activate

audit.read

report.read
```

---

# 11. Customer Domain

A customer may be:

```text
INDIVIDUAL
BUSINESS
```

## Customer

```text
Customer
- id
- tenant_id
- type
- external_reference
- status
- risk_level
- risk_score
- country
- created_at
- updated_at
```

## Individual

```text
Individual
- customer_id
- first_name
- middle_name
- last_name
- date_of_birth
- nationality
- occupation
```

## Business

```text
Business
- customer_id
- legal_name
- registration_number
- incorporation_country
- business_type
```

---

# 12. Relationships

Aegis must eventually support connected-entity analysis.

```text
Customer
 ├── Accounts
 ├── Devices
 ├── Addresses
 ├── Businesses
 ├── Beneficial Owners
 └── Related Customers
```

Relationships should be explicit:

```text
Customer A
    │
    ├── OWNS → Account 001
    │
    ├── OWNS → Account 002
    │
    └── USES → Device 003
```

This becomes important for multi-account and network-based AML detection.

---

# 13. Account Domain

```text
Account
- id
- tenant_id
- customer_id
- external_reference
- account_type
- currency
- status
- opened_at
- closed_at
```

Possible account types:

```text
BANK_ACCOUNT
MOBILE_WALLET
LOAN_ACCOUNT
SACCO_ACCOUNT
MERCHANT_ACCOUNT
OTHER
```

---

# 14. Transaction Domain

```text
Transaction
- id
- tenant_id
- external_reference
- account_id
- customer_id
- transaction_type
- direction
- amount
- currency
- timestamp
- counterparty_id
- counterparty_account
- country
- channel
- device_id
- location
- status
- metadata
- created_at
```

Do not store institution-specific transaction formats directly in the core domain.

Use adapters.

---

# 15. Transaction Ingestion

External systems should be translated into an Aegis canonical transaction model.

```text
External System
      ↓
Adapter
      ↓
Validation
      ↓
Normalization
      ↓
Transaction
      ↓
Event
```

Supported ingestion mechanisms should eventually include:

* REST API
* Webhooks
* CSV
* SFTP
* Kafka/event streams

MVP can start with:

1. REST API
2. CSV import

---

# 16. Idempotency

Transaction ingestion must be idempotent.

External systems may resend transactions.

Use:

```text
tenant_id + external_reference
```

or an equivalent institution-specific idempotency key.

The system must not create duplicate transactions.

---

# 17. Feature Engine

Rules should not directly query raw transaction tables for every evaluation.

Introduce a feature layer.

Examples:

```text
transaction_count_24h
transaction_sum_24h
credit_sum_7d
debit_sum_7d
average_transaction_30d
largest_transaction_30d
unique_counterparties_24h
countries_7d
days_since_account_opened
```

Features should support configurable windows.

Example:

```text
SUM(credit.amount)
WHERE customer_id = X
AND timestamp >= NOW() - 24 HOURS
```

---

# 18. Rule Engine

Rules are first-class entities.

## Rule

```text
Rule
- id
- tenant_id
- code
- name
- description
- scenario
- status
- severity
- version
- effective_from
- effective_to
- definition
- created_by
- approved_by
- created_at
- updated_at
```

---

# 19. Rule Lifecycle

Rules must follow:

```text
DRAFT
  ↓
VALIDATION
  ↓
SIMULATION
  ↓
PENDING_APPROVAL
  ↓
APPROVED
  ↓
ACTIVE
  ↓
SUSPENDED
  ↓
RETIRED
```

No direct modification of an active version.

Create a new version instead.

---

# 20. Rule Versioning

Example:

```text
STRUCTURING_001
    v1
    v2
    v3
    v4
```

An alert must store the exact rule version that triggered it.

This is critical for historical reproducibility.

If a rule changes tomorrow, an alert generated today must still be explainable using today's rule version.

---

# 21. Rule Definition

Use a structured JSON representation.

Example:

```json
{
  "scenario": "STRUCTURING",
  "conditions": [
    {
      "feature": "transaction_count",
      "window": "24h",
      "operator": ">=",
      "value": 5
    },
    {
      "feature": "transaction_sum",
      "window": "24h",
      "operator": ">=",
      "value": 450000
    }
  ],
  "actions": [
    {
      "type": "CREATE_ALERT",
      "severity": "HIGH"
    }
  ]
}
```

The rule schema must be versioned.

Validate definitions before activation.

---

# 22. Rule Engine Requirements

The engine must support:

* comparisons,
* AND / OR conditions,
* transaction counts,
* sums,
* averages,
* minimum/maximum,
* time windows,
* transaction direction,
* transaction type,
* customer attributes,
* account age,
* geography,
* risk levels,
* related entities,
* configurable thresholds.

Future versions may add:

* graph conditions,
* statistical anomalies,
* behavioural baselines,
* ML signals.

---

# 23. Rule Simulation

Compliance users should eventually be able to simulate a rule against historical data.

Example:

```text
Current:
Threshold = KES 100,000

Historical period:
30 days

Transactions:
4,200,000

Alerts:
1,842
```

Change:

```text
Threshold = KES 150,000
```

Simulation:

```text
Projected alerts:
734
```

Simulation must not modify live rules.

---

# 24. Alert Domain

## Alert

```text
Alert
- id
- tenant_id
- customer_id
- account_id
- rule_id
- rule_version_id
- severity
- status
- title
- description
- risk_score
- triggered_at
- assigned_to
- evidence
- created_at
- updated_at
```

---

# 25. Alert Status

```text
NEW
 ↓
ASSIGNED
 ↓
IN_REVIEW
 ↓
ESCALATED / DISMISSED / CASE_CREATED
```

Do not allow arbitrary state transitions.

Define valid transitions explicitly.

---

# 26. Explainability

Every alert must contain an evidence structure.

Example:

```json
{
  "rule": "STRUCTURING_001",
  "rule_version": 3,
  "facts": [
    {
      "feature": "transaction_count_24h",
      "value": 8,
      "threshold": 5
    },
    {
      "feature": "transaction_sum_24h",
      "value": 790000,
      "threshold": 450000
    }
  ]
}
```

The UI can turn this into human-readable evidence.

---

# 27. Case Management

An alert is not automatically a case.

Multiple alerts may belong to one case.

```text
Alert 101
Alert 109
Alert 117
   ↓
Case 501
```

## Case

```text
Case
- id
- tenant_id
- customer_id
- title
- status
- priority
- assigned_to
- opened_at
- due_at
- closed_at
- disposition
- conclusion
```

---

# 28. Case Status

```text
OPEN
INVESTIGATING
PENDING_REVIEW
ESCALATED
CLOSED
```

Possible dispositions:

```text
FALSE_POSITIVE
NO_SUSPICIOUS_ACTIVITY
SUSPICIOUS_ACTIVITY
ESCALATED
REPORTED
OTHER
```

Final disposition must require an explanation.

---

# 29. Investigation Workspace

Investigators need one consolidated view.

The case screen should expose:

```text
Customer profile
KYC information
Accounts
Transactions
Alerts
Risk score
Screening results
Related entities
Devices
Timeline
Documents
Notes
Tasks
Audit history
```

This is the primary operational screen of the product.

---

# 30. Risk Engine

Separate risk scoring from transaction rules.

Risk should be composable from:

```text
Customer attributes
+
KYC
+
Geography
+
Screening
+
Transaction behaviour
+
Historical alerts
+
External risk signals
```

Example:

```text
Customer Risk Score

Geography        +20
Customer type    +10
PEP              +40
Transaction      +15
Behaviour        +15

Total             100
```

Risk models must be configurable.

---

# 31. Screening

Screening is a separate domain.

The architecture should support:

```text
Customer
 ↓
Screening Request
 ↓
Provider
 ↓
Match Results
 ↓
Match Review
 ↓
Decision
```

Future providers may include:

* sanctions lists,
* PEP databases,
* watchlists,
* adverse media providers.

Do not hardcode a single screening provider into the domain.

---

# 32. Audit Architecture

Create a central immutable audit mechanism.

Example:

```text
AuditEvent
- id
- tenant_id
- actor_id
- action
- entity_type
- entity_id
- timestamp
- before
- after
- metadata
- correlation_id
```

Examples:

```text
RULE_CREATED
RULE_APPROVED
RULE_ACTIVATED

ALERT_ASSIGNED
ALERT_DISMISSED

CASE_CREATED
CASE_ESCALATED
CASE_CLOSED

CUSTOMER_UPDATED
RISK_CHANGED
```

Audit records must not be casually editable or deleted.

---

# 33. Event Architecture

The system should be event-driven internally even while remaining a modular monolith.

Example:

```text
TransactionReceived
        ↓
TransactionNormalized
        ↓
TransactionCreated
        ↓
FeaturesCalculated
        ↓
RulesEvaluated
        ↓
AlertCreated
```

Additional events:

```text
CustomerCreated
CustomerUpdated

RuleActivated
RuleRetired

AlertCreated
AlertAssigned
AlertDismissed

CaseCreated
CaseClosed

RiskScoreChanged
```

Use an outbox pattern for reliable event publication.

---

# 34. Real-Time vs Batch

Do not design the system exclusively around T+1.

## Real-time pipeline

```text
Transaction
 ↓
Event
 ↓
Feature evaluation
 ↓
Rules
 ↓
Alert
```

Use for scenarios where immediate detection is useful.

## Batch pipeline

```text
Transactions
 ↓
Aggregation
 ↓
Historical features
 ↓
Rules
 ↓
Alerts
```

Use for:

* behavioural change,
* long-window aggregation,
* dormant-account analysis,
* periodic risk recalculation.

The rule engine should be usable by both.

---

# 35. Background Jobs

The system will require scheduled jobs for:

* T+1 transaction monitoring,
* feature aggregation,
* customer risk recalculation,
* periodic screening,
* SLA monitoring,
* alert escalation,
* report generation,
* data retention,
* reconciliation.

Jobs must be:

* idempotent,
* observable,
* retryable,
* auditable.

---

# 36. API Design

All APIs should be tenant-aware.

## Authentication

```http
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
```

---

## Customers

```http
POST   /api/v1/customers
GET    /api/v1/customers
GET    /api/v1/customers/{id}
PATCH  /api/v1/customers/{id}
```

---

## Accounts

```http
POST   /api/v1/accounts
GET    /api/v1/accounts
GET    /api/v1/accounts/{id}
```

---

## Transactions

```http
POST /api/v1/transactions
POST /api/v1/transactions/bulk
GET  /api/v1/transactions
GET  /api/v1/transactions/{id}
```

---

## Rules

```http
POST /api/v1/rules
GET  /api/v1/rules
GET  /api/v1/rules/{id}

POST /api/v1/rules/{id}/versions
POST /api/v1/rules/{id}/validate
POST /api/v1/rules/{id}/simulate
POST /api/v1/rules/{id}/submit
POST /api/v1/rules/{id}/approve
POST /api/v1/rules/{id}/activate
POST /api/v1/rules/{id}/suspend
```

---

## Alerts

```http
GET   /api/v1/alerts
GET   /api/v1/alerts/{id}
POST  /api/v1/alerts/{id}/assign
POST  /api/v1/alerts/{id}/dismiss
POST  /api/v1/alerts/{id}/create-case
```

---

## Cases

```http
POST  /api/v1/cases
GET   /api/v1/cases
GET   /api/v1/cases/{id}
PATCH /api/v1/cases/{id}

POST /api/v1/cases/{id}/assign
POST /api/v1/cases/{id}/escalate
POST /api/v1/cases/{id}/close
POST /api/v1/cases/{id}/notes
```

---

## Risk

```http
GET /api/v1/customers/{id}/risk
POST /api/v1/customers/{id}/risk/recalculate
```

---

## Audit

```http
GET /api/v1/audit-events
GET /api/v1/audit-events/{id}
```

---

# 37. API Standards

All APIs must implement:

* versioning,
* pagination,
* filtering,
* sorting,
* request validation,
* consistent error responses,
* correlation IDs,
* idempotency where required,
* authorization,
* audit logging.

Example error:

```json
{
  "code": "RULE_NOT_ACTIVE",
  "message": "The requested rule is not active.",
  "correlationId": "..."
}
```

---

# 38. Database Requirements

PostgreSQL is the primary database.

Every tenant-owned table must contain:

```text
tenant_id
created_at
updated_at
```

Where appropriate:

```text
created_by
updated_by
```

Use foreign keys and database constraints.

Do not rely exclusively on application-level validation.

---

# 39. Data Isolation

Tenant isolation must be enforced at multiple levels:

1. Authentication context
2. Application service
3. Repository
4. Database constraints
5. Automated tests

Repositories should require tenant context.

Avoid APIs such as:

```text
repository.findById(id)
```

where tenant context can accidentally be omitted.

Prefer:

```text
repository.findByTenantAndId(tenantId, id)
```

---

# 40. Security Requirements

Aegis handles highly sensitive financial data.

Required:

* encrypted transport,
* encrypted secrets,
* secure password hashing,
* short-lived access tokens,
* refresh token rotation,
* RBAC,
* tenant isolation,
* audit logs,
* rate limiting,
* input validation,
* secure headers,
* secrets outside source control,
* database encryption where supported,
* secure backups,
* dependency scanning,
* vulnerability scanning.

Never store:

* plaintext passwords,
* API secrets in code,
* customer credentials,
* unnecessary sensitive information.

---

# 41. Data Retention

Retention must be configurable by tenant and aligned with applicable legal/regulatory requirements.

Do not implement destructive deletion workflows without considering:

* audit requirements,
* legal holds,
* investigation status,
* retention policies.

---

# 42. Observability

Production services must expose:

### Logs

Structured logs containing:

```text
timestamp
service
tenant_id
correlation_id
request_id
event
severity
```

Do not log sensitive financial data unnecessarily.

### Metrics

At minimum:

```text
transactions_processed
transactions_failed
rules_evaluated
alerts_created
alerts_failed
cases_open
cases_closed
job_duration
job_failures
api_latency
```

### Health

```http
GET /health/live
GET /health/ready
```

---

# 43. Performance Requirements

The architecture must support eventual high-volume transaction processing.

Do not optimize prematurely, but avoid designs that require:

```text
for every transaction:
    query entire transaction history
```

Feature aggregation, indexes, caching and precomputed data should be used where appropriate.

All expensive operations should be measurable.

---

# 44. Reliability

Transaction processing must be resilient to:

* duplicate events,
* retries,
* partial failures,
* external provider failures,
* database failures,
* worker restarts.

Critical processing must be idempotent.

---

# 45. Testing Requirements

Testing is mandatory at multiple levels.

## Unit tests

Rules, domain logic, feature calculations, state transitions.

## Integration tests

Database repositories, event publishing, ingestion.

## Contract tests

External integrations.

## End-to-end tests

At minimum:

```text
Create tenant
 ↓
Create customer
 ↓
Create account
 ↓
Submit transactions
 ↓
Trigger rule
 ↓
Create alert
 ↓
Create case
 ↓
Close case
 ↓
Verify audit trail
```

---

# 46. AML Rule Testing

Every rule must have:

### Positive test

Input should trigger.

### Negative test

Input should not trigger.

### Boundary tests

Test:

```text
threshold - 1
threshold
threshold + 1
```

### Time-window tests

Test:

```text
window - 1
window boundary
window + 1
```

### Regression tests

Changing the rule engine must not silently alter existing rule behaviour.

---

# 47. Compliance Configuration

The compliance team must eventually control:

```text
Rules
Thresholds
Risk weights
Severity
SLA
Escalation
Effective dates
Jurisdictions
Transaction types
```

Engineering should provide the configuration mechanism.

Compliance owns the policy values.

Engineering should not silently decide regulatory thresholds.

---

# 48. Configuration Approval

Sensitive configuration changes should support:

```text
Draft
 ↓
Review
 ↓
Approval
 ↓
Activation
```

For high-risk changes, require separation of duties:

```text
Creator ≠ Approver
```

where configured by the institution.

---

# 49. Dashboard

MVP dashboard should provide:

```text
Transactions processed
Alerts today
Open alerts
High-risk alerts
Open cases
Overdue cases
Risk distribution
Top triggered rules
```

Example:

```text
┌──────────────────────────────────────────┐
│ AML OPERATIONS                           │
├────────────┬────────────┬───────────────┤
│ 2.4M       │ 1,284      │ 93            │
│ Transactions│ Alerts     │ Open Cases    │
├────────────┴────────────┴───────────────┤
│ Alert trend                              │
│                                          │
├──────────────────────────────────────────┤
│ Top Rules                                │
│ Structuring          421                 │
│ Rapid movement       213                 │
│ High-risk geography  104                 │
└──────────────────────────────────────────┘
```

---

# 50. Initial AML Scenarios

## 50.1 Structuring

Detect multiple transactions designed to avoid a configurable threshold.

Parameters:

```text
transaction type
minimum count
maximum individual amount
aggregate amount
time window
customer/account scope
```

---

## 50.2 Rapid movement / pass-through

Detect funds entering and leaving an account within a configurable period.

Parameters:

```text
incoming amount
outgoing amount
time window
minimum percentage moved
transaction types
```

---

## 50.3 High-risk geography

Detect transactions involving configured high-risk jurisdictions.

Parameters:

```text
country
transaction direction
amount
customer risk
counterparty
```

Country lists must be configurable.

Do not hardcode geopolitical lists into application code.

---

# 51. Future AML Scenarios

Architecture must accommodate:

* behavioural change,
* new-account flight,
* dormant-account reactivation,
* round-number activity,
* multi-account/device activity,
* crypto-related activity,
* velocity anomalies,
* unusual counterparties,
* geographic anomalies.

---

# 52. Regulatory Reporting

Not part of the first MVP implementation, but the architecture must leave room for:

```text
Case
 ↓
Reporting eligibility
 ↓
Compliance review
 ↓
Report preparation
 ↓
Approval
 ↓
Submission
 ↓
Submission record
```

The exact regulatory workflow should be defined with the compliance/legal team before implementation.

---

# 53. Integration Architecture

External integrations must use adapters.

```text
                    Aegis
                      │
              Integration Layer
                      │
       ┌──────────────┼──────────────┐
       │              │              │
 Core Banking    Mobile Money    Screening
 Adapter          Adapter        Provider
```

Never let external provider models leak into the domain.

---

# 54. AI Roadmap

AI is a later capability.

Potential capabilities:

### Investigation summary

```text
Summarize this case.
```

### Evidence explanation

```text
Explain why this customer triggered an alert.
```

### Timeline generation

```text
Summarize suspicious activity over the last 90 days.
```

### Investigator assistance

```text
What evidence should I review next?
```

### Report drafting

```text
Draft an investigation summary based only on the available evidence.
```

AI output must always identify its source evidence.

AI must not silently modify:

* rules,
* risk scores,
* alerts,
* case dispositions,
* regulatory decisions.

---

# 55. Repository Standards

The repository should maintain:

```text
/docs
  architecture.md
  domain-model.md
  rule-engine.md
  security.md
  api.md
  decisions/

/src
  /modules
  /shared

/tests
  /unit
  /integration
  /e2e
```

Architecture decisions should use ADRs.

Example:

```text
ADR-001 Modular Monolith
ADR-002 PostgreSQL
ADR-003 Rule Versioning
ADR-004 Event Outbox
ADR-005 Tenant Isolation
```

---

# 56. Engineering Rules

The senior engineer must follow these principles:

### Do

* keep domain modules isolated,
* use explicit interfaces,
* write tests before complex rule changes,
* version rules,
* preserve audit history,
* validate all external data,
* enforce tenant boundaries,
* make processing idempotent,
* document architectural decisions,
* use migrations for schema changes.

### Do not

* hardcode AML thresholds,
* directly modify active rule versions,
* bypass tenant filtering,
* put business logic in controllers,
* allow arbitrary state transitions,
* expose database models directly through APIs,
* couple the domain to one external provider,
* introduce microservices without a demonstrated need,
* allow AI to become an authoritative compliance decision-maker.

---

# 57. MVP Milestones

## Milestone 1 — Platform Core

Deliver:

* PostgreSQL schema
* migrations
* tenant model
* authentication
* RBAC
* repository pattern
* audit foundation
* API foundation

### Exit criteria

A user can authenticate and operate inside a tenant with authorized permissions.

---

# 58. Milestone 2 — Customer & Transaction Core

Deliver:

* customer
* individual/business
* accounts
* transactions
* transaction ingestion
* CSV import
* normalization
* idempotency

### Exit criteria

A tenant can ingest and query realistic transaction data.

---

# 59. Milestone 3 — Rule Engine

Deliver:

* rule model
* rule schema
* rule validation
* rule versions
* rule evaluation
* three MVP scenarios
* rule tests

### Exit criteria

A transaction dataset can produce deterministic, explainable rule results.

---

# 60. Milestone 4 — Alert Management

Deliver:

* alert creation
* severity
* assignment
* alert lifecycle
* evidence
* alert search/filtering

### Exit criteria

Investigators can work alerts without engineering intervention.

---

# 61. Milestone 5 — Cases & Investigation

Deliver:

* case creation
* alert-to-case linking
* investigator assignment
* notes
* timeline
* customer 360
* disposition
* escalation

### Exit criteria

An investigator can take an alert from detection to documented decision.

---

# 62. Milestone 6 — Risk & Dashboard

Deliver:

* initial risk model
* configurable risk factors
* customer risk score
* dashboard
* rule performance metrics

### Exit criteria

Compliance can understand the institution's current AML risk workload.

---

# 63. Milestone 7 — Screening

Deliver:

* screening abstraction
* provider adapter
* screening result
* match review
* screening audit

---

# 64. Milestone 8 — Production Hardening

Deliver:

* security review
* tenant isolation testing
* performance testing
* observability
* backup/restore
* failure recovery
* rate limiting
* dependency scanning
* penetration/security assessment
* deployment automation

---

# 65. MVP Definition of Done

Aegis MVP is considered complete only when:

### Data

* customers can be created,
* accounts can be created,
* transactions can be ingested,
* duplicates are prevented.

### Detection

* rules can be configured,
* rules can be versioned,
* rules can be activated,
* rules evaluate correctly,
* three AML scenarios work.

### Alerts

* alerts are generated,
* alerts contain evidence,
* alerts reference rule versions,
* alerts can be assigned,
* alerts can be dismissed/escalated.

### Cases

* alerts can become cases,
* cases can be assigned,
* investigators can add notes,
* cases can be closed,
* dispositions are recorded.

### Audit

Every important operation produces an audit event.

### Security

Tenant A cannot access Tenant B data.

### Demonstration

The following must work from a clean environment:

```text
Create tenant
      ↓
Create user
      ↓
Create customer
      ↓
Create account
      ↓
Import transactions
      ↓
Run rule engine
      ↓
Generate alert
      ↓
Investigate
      ↓
Create case
      ↓
Close case
      ↓
Inspect audit trail
```

---

# 66. Production Readiness Criteria

Before onboarding a real institution, the following must be completed:

* security assessment,
* tenant isolation verification,
* backup and restoration testing,
* disaster recovery plan,
* monitoring,
* alerting,
* database migration strategy,
* secrets management,
* access-control review,
* audit integrity review,
* data-retention policy,
* incident response process,
* compliance/legal review,
* performance testing,
* integration failure testing.

---

# 67. Open Product Decisions

These decisions must be resolved by Product/Compliance rather than guessed by Engineering:

1. Initial supported institution types.
2. Initial customer risk categories.
3. Risk scoring methodology.
4. AML thresholds.
5. Scenario-specific windows.
6. Alert severity definitions.
7. Investigation SLAs.
8. Case escalation rules.
9. Rule approval workflow.
10. Required segregation of duties.
11. Data retention requirements.
12. Required regulatory reports.
13. Screening providers.
14. Supported currencies.
15. Supported jurisdictions.
16. Required transaction channels.
17. Initial external integrations.

Engineering should make the platform capable of configuring these values without embedding assumptions into the codebase.

---

# 68. Recommended Team Structure

For a serious MVP:

### Senior/Principal Engineer

Own:

* architecture,
* domain boundaries,
* rule engine,
* technical standards,
* security architecture,
* code quality.

### Backend Engineer

Own:

* APIs,
* persistence,
* transaction ingestion,
* background processing.

### Frontend Engineer

Own:

* compliance dashboard,
* alerts,
* investigation workspace,
* case management,
* rule configuration.

### Compliance SME

Own:

* AML scenarios,
* thresholds,
* risk methodology,
* investigation workflow,
* regulatory requirements.

### DevOps/Platform support

Own:

* infrastructure,
* CI/CD,
* monitoring,
* backups,
* deployment.

One strong senior engineer can initially cover several of these responsibilities, but **compliance expertise cannot be replaced by engineering assumptions**.

---

# 69. First Engineering Sprint

The team should not start by building all modules.

The first sprint should establish:

```text
1. PostgreSQL schema foundation
2. Tenant model
3. User/RBAC
4. Customer model
5. Account model
6. Transaction model
7. Repository interfaces
8. Audit mechanism
9. Transaction ingestion endpoint
10. One end-to-end structuring rule
```

Then demonstrate:

```text
POST transaction
      ↓
Persist
      ↓
Feature calculation
      ↓
Rule evaluation
      ↓
Alert
      ↓
Audit
```

If that works, expand the slice.

---

# 70. Final Engineering Principle

The team should continuously ask:

> "Can a compliance officer use this without asking an engineer to change code?"

If the answer is no for something that is a **policy decision**, the implementation is probably too hardcoded.

The goal is not merely to build software that detects suspicious transactions.

The goal is to build a platform where a financial institution can:

```text
CONFIGURE
    ↓
MONITOR
    ↓
DETECT
    ↓
INVESTIGATE
    ↓
DECIDE
    ↓
AUDIT
    ↓
REPORT
```

That is Aegis.

---

# 71. Immediate Engineering Objective

The immediate objective is therefore:

> **Build one production-quality vertical slice from transaction ingestion through explainable AML detection, alert investigation, case disposition and audit — inside a properly isolated multi-tenant architecture.**

Do not expand the scope until this workflow is working reliably.

Once this foundation works, additional AML scenarios, KYC/KYB, screening, advanced risk models, regulatory reporting, integrations and AI can be added without redesigning the core platform.

**End of Specification**
