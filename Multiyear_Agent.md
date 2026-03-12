# AGENTS.md

## Project: General Multi-Year Deal Processing Framework

This document is a **general, reusable design guide** for building logic that processes **multi-year deals, subscriptions, contracts, quotes, renewals, billing schedules, and term-based commercial records**.

It is intentionally:

- **not vendor-specific**
- **not platform-specific**
- **not programming-language-specific**
- **not spreadsheet-specific**

Use this document whenever building a system, script, macro, service, pipeline, or application that must interpret and transform multi-year commercial deal data into normalized billing or output records.

---

## 1) Purpose

Many multi-year deals are simple on the surface but complicated in implementation because:

- the **contract term** may not match the **billing schedule**
- a deal may include **stub periods**, **partial periods**, **co-termed alignments**, or **renewal boundaries**
- some source documents explicitly define **billing periods**, while others require those periods to be derived
- pricing may need to be handled differently depending on whether the source provides **period totals**, **unit prices**, **list prices**, **discounted prices**, or only term totals

The purpose of this framework is to define a **general approach** for:

1. parsing multi-year deal data
2. deciding whether to use explicit billing periods or derived periods
3. allocating price across periods safely
4. producing consistent output rows for downstream systems

---

## 2) Core Principle

> **Use explicit period data when it exists and is usable. If explicit period data is missing, empty, or invalid, fall back to derived period logic.**

This is the most important implementation rule in this document.

In other words:

- If the source includes a real billing schedule with valid dates, treat that as authoritative.
- If the source includes a billing-period section but it is empty or unusable, do not trust it.
- In that case, generate periods using the fallback logic appropriate to the business context.

---

## 3) General Problem Model

A multi-year deal usually contains some combination of these concepts:

### 3.1 Commercial item
An item being sold, licensed, subscribed, supported, or renewed.

Typical fields:
- item identifier / product code / SKU
- description
- quantity
- start date
- end date
- pricing metrics
- billing behavior

### 3.2 Contract term
The overall active time range for the item.

Examples:
- `2026-03-31` to `2027-04-24`
- 36-month subscription
- 5-year support agreement

### 3.3 Billing periods
How the contract is actually invoiced, recognized, or reported over time.

Examples:
- annual billing
- quarterly billing
- first short stub period followed by regular yearly periods
- custom dates aligned to fiscal or renewal cycles

### 3.4 Pricing layers
A deal can include multiple price concepts. Common examples:
- list price
- regular price
- discounted price
- unit price
- term total
- billing-period total
- support price
- one-time fee vs recurring fee

A robust design must avoid assuming these are interchangeable.

---

## 4) General Design Goals

A good multi-year deal processor should:

1. work across many source layouts
2. support both explicit and derived period models
3. separate parsing from business decisions
4. separate pricing allocation from date segmentation
5. remain configurable
6. make fallback behavior explicit
7. produce auditable output
8. handle edge cases without silently corrupting numbers or dates

---

## 5) Canonical Decision Framework

Use the following general decision tree.

```text
1. Parse commercial items
2. Detect whether an explicit period schedule exists
3. Validate whether explicit period rows are actually usable

4. If explicit period schedule exists AND at least one valid period row exists:
       Use explicit period-driven processing
   Else if the deal is multi-period or multi-year and fallback derivation is needed:
       Use derived-period logic
   Else:
       Use single-term / single-line processing
```

### Important note

Do **not** switch into explicit-period mode merely because a section, table, or label exists.

You should switch only when the explicit period data contains **usable rows**.

---

## 6) What Counts as “Usable Explicit Period Data”

Explicit period data should be considered usable only when all required minimum conditions are met.

### Recommended validation rule
A source should be treated as having usable explicit periods only if:

1. a period schedule section or table is found
2. the expected period columns can be identified
3. at least one data row exists
4. at least one row contains a valid start date and valid end date
5. optional numeric fields are either valid or safely defaultable

### Examples of unusable explicit period data
Treat period data as unusable when:
- the section title exists but rows are blank
- all dates are missing
- dates are malformed and cannot be parsed
- the only rows are totals or summaries
- the schedule exists structurally but has no actionable periods

When that happens, **fall back to derived logic**.

---

## 7) Period Strategy Types

A general multi-year processor should support at least three strategy types.

### Strategy A — Explicit schedule strategy
Use the source’s period table as the authoritative billing or segmentation schedule.

Use when:
- explicit period rows exist
- dates are valid
- business intent is clear

### Strategy B — Derived recurring strategy
Generate periods from the contract term using a rule such as:
- yearly anniversaries
- monthly boundaries
- quarterly boundaries
- fiscal periods
- renewal alignment dates

Use when:
- no usable period table exists
- the business rules clearly define how periods should be generated

### Strategy C — Single-term strategy
Treat the item as one continuous term and emit one record.

Use when:
- the downstream model does not require periodization
- the item is not meant to be segmented
- no explicit schedule exists and no fallback segmentation is needed

---

## 8) Recommended Architecture

This framework should be implemented in **layers**.

### 8.1 Parsing layer
Responsible for reading source data and normalizing it.

Responsibilities:
- detect sections / tables / input ranges
- identify headers or field names
- parse dates safely
- parse numeric values safely
- extract rows into structured records

### 8.2 Validation layer
Responsible for deciding whether parsed data is usable.

Responsibilities:
- validate explicit period schedule
- validate item dates
- validate quantities and prices
- identify incomplete records
- detect malformed rows

### 8.3 Decision layer
Responsible for choosing the periodization strategy.

Responsibilities:
- explicit schedule vs derived schedule vs single-term
- choosing fallback behavior
- selecting allocation mode

### 8.4 Allocation layer
Responsible for distributing money across periods.

Responsibilities:
- period totals vs prorated unit values
- rounding rules
- quantity-aware allocation
- reconciliation checks

### 8.5 Rendering / output layer
Responsible for producing downstream output.

Responsibilities:
- normalized output rows
- export format
- output ordering
- audit fields and traceability metadata

---

## 9) General Data Model

Use a data model that is generic enough for any implementation language.

### 9.1 Contract item

```text
ItemId
ItemDescription
Quantity
ContractStartDate
ContractEndDate
BillingModel
PriceModel
ListPrice
RegularPrice
NetPrice
TermTotal
Currency
Attributes...
```

### 9.2 Explicit period row

```text
PeriodLabel
PeriodStartDate
PeriodEndDate
PeriodDuration
PeriodTotalAmount
PeriodMetadata...
```

### 9.3 Normalized output row

```text
ItemId
Quantity
OutputStartDate
OutputEndDate
AllocatedCost
ReferencePrice
DerivedFrom
AllocationMethod
SourceRowId
Warnings
```

This model can be represented as objects, structs, classes, dictionaries, tables, collections, or records depending on the technology stack.

---

## 10) General Pricing Principles

A robust system must distinguish among different pricing concepts.

### 10.1 Do not assume one price column means everything
Possible source fields may represent different meanings:
- public list price
- standard internal price
- net sell price
- unit price
- total deal value
- total for one billing period
- total across full term

### 10.2 Define a clear source-of-truth rule for each output field
For every output field, explicitly define:
- where it comes from
- what it means
- when it should be prorated or not prorated
- fallback behavior if the preferred field is missing

### 10.3 Keep “reference price” separate from “allocated billable price”
In many systems, one field represents a reference or MSRP-like value, while another represents the actual billable or net amount. These should be modeled separately.

---

## 11) Allocation Strategies

The processor should support multiple allocation strategies because different businesses interpret multi-period pricing differently.

### Strategy 1 — Use explicit period totals
Use the period total provided by the source as authoritative.

Best when:
- the source provides trustworthy per-period amounts
- exact invoice alignment matters

### Strategy 2 — Prorate by days
Allocate a total or unit amount according to the number of days in each overlapping period.

Typical formula:

```text
allocatedAmount = baseAmount * (overlapDays / totalEligibleDays)
```

Best when:
- the source has dates but not reliable per-period amounts
- day-weighted proration is business-approved

### Strategy 3 — Split evenly across periods
Allocate evenly across the number of segments.

Typical formula:

```text
allocatedAmount = baseAmount / segmentCount
```

Best when:
- periods are intended to be equal conceptually
- exact date weighting is not required

### Strategy 4 — Custom business allocation
Some businesses require custom rules such as:
- full first year, prorated final stub
- minimum charge thresholds
- front-loaded revenue recognition
- separate recurring and non-recurring components

The system should allow pluggable allocation rules.

---

## 12) Overlap-Based Processing

If items and periods are both date-based, overlap logic is often the safest general-purpose model.

### Overlap rule
An item overlaps a period when:

```text
itemStart <= periodEnd AND itemEnd >= periodStart
```

### Derived output window
For each overlap:

```text
outputStart = max(itemStart, periodStart)
outputEnd   = min(itemEnd, periodEnd)
```

### Why this matters
Overlap logic supports:
- stub periods
- co-termed items
- partial renewals
- items starting after the first schedule row
- items ending before the final schedule row
- multi-item contracts with different date ranges

---

## 13) Fallback Logic

Fallback logic should be intentional, visible, and configurable.

### Recommended fallback order

```text
If usable explicit periods exist:
    use explicit periods
Else if derived multi-period logic is enabled:
    generate periods using configured derivation rules
Else:
    use single-term output
```

### Never hide fallback silently
The system should record which path was used, for example:
- `EXPLICIT_PERIODS`
- `DERIVED_ANNUAL`
- `DERIVED_MONTHLY`
- `SINGLE_TERM`

This makes debugging and auditing much easier.

---

## 14) Validation and Safety Rules

A good multi-year deal processor should validate all critical elements.

### Validate dates
- start date present when required
- end date present when required
- end date not before start date
- parsable formats only

### Validate quantities
- blank quantities get explicit default behavior
- zero or negative quantities require policy handling

### Validate money
- parse safely from text or numeric values
- strip formatting characters when needed
- define rounding policy

### Validate reconciliation
After allocation, compare the sum of output amounts to the expected total.

Possible checks:
- sum of period totals equals overall total
- sum of allocated net amounts equals source net total within tolerance
- duration totals are consistent with dates

---

## 15) Configuration Model

This framework should be configurable rather than hard-coded.

### Recommended configuration categories

#### 15.1 Period strategy settings
- prefer explicit periods: yes/no
- allow derived annual periods: yes/no
- allow derived monthly periods: yes/no
- fallback if explicit schedule invalid

#### 15.2 Allocation settings
- use explicit period totals
- prorate by days
- split evenly
- custom allocation handler

#### 15.3 Price source settings
- preferred reference price field
- fallback reference price field
- preferred cost field
- whether reference price should remain constant across periods

#### 15.4 Validation settings
- strict date parsing vs tolerant parsing
- rounding tolerance
- required fields for processing

---

## 16) Logging and Auditability

Every implementation should make its decisions visible.

### Record at least:
- which strategy was chosen
- whether explicit periods were found
- whether explicit periods were rejected as unusable
- which allocation method was used
- any rows skipped and why
- any fields defaulted and why

This is essential for debugging multi-year deals because many failures come from assumptions that were never documented in output.

---

## 17) Testing Framework

A multi-year deal processor should be tested with a matrix of scenarios.

### Required test categories

#### Test 1 — Valid explicit schedule
- explicit periods present
- valid dates
- processor should use explicit schedule

#### Test 2 — Empty explicit schedule section
- section exists but rows empty
- processor should reject explicit schedule
- fallback should be used

#### Test 3 — Malformed explicit schedule
- one or more rows invalid
- no usable period rows
- fallback should be used

#### Test 4 — No explicit schedule
- derived logic should be used if configured

#### Test 5 — Single-term item
- one output row only

#### Test 6 — Stub first period
- short first period followed by regular periods
- overlap logic should handle it correctly

#### Test 7 — Partial item overlap
- item starts mid-period or ends before final period end
- output dates should reflect overlap windows

#### Test 8 — Pricing reconciliation
- allocated results should reconcile to source totals within policy tolerance

#### Test 9 — Mixed items
- multiple items with different terms and quantities
- all should be handled correctly in one run

#### Test 10 — Missing optional fields
- fallback/default behavior should be deterministic

---

## 18) Anti-Patterns to Avoid

Avoid these common mistakes:

1. assuming the existence of a period section means it is usable
2. hard-coding one vendor’s layout into the core logic
3. mixing parsing, validation, pricing, and rendering in one giant procedure
4. assuming all annual deals should be split by anniversary
5. assuming all period totals should be divided evenly
6. silently falling back without recording that a fallback occurred
7. using the same price field for both reference price and net billable amount without a defined rule
8. ignoring reconciliation checks after allocations

---

## 19) Reference Pseudocode

Use this language-agnostic pseudocode as a blueprint.

```text
items = parseItems(source)
periodSchedule = parseExplicitPeriods(source)
usableExplicitPeriods = validateExplicitPeriods(periodSchedule)

if usableExplicitPeriods:
    strategy = EXPLICIT_PERIODS
    outputRows = processUsingExplicitPeriods(items, periodSchedule, config)
else if shouldDerivePeriods(items, config):
    strategy = DERIVED_PERIODS
    derivedPeriods = derivePeriods(items, config)
    outputRows = processUsingDerivedPeriods(items, derivedPeriods, config)
else:
    strategy = SINGLE_TERM
    outputRows = processAsSingleTerm(items, config)

validateReconciliation(outputRows, source, config)
writeOutput(outputRows, strategy)
logDecisions(strategy, usableExplicitPeriods, config)
```

---

## 20) Recommended Helper Concepts

Regardless of implementation language, create helpers or modules conceptually equivalent to these:

- `parseItems`
- `parseExplicitPeriods`
- `validateExplicitPeriods`
- `derivePeriods`
- `computeOverlap`
- `allocateAmounts`
- `chooseReferencePrice`
- `normalizeOutput`
- `validateReconciliation`
- `logDecisions`

These names are conceptual and should be adapted to the chosen stack.

---

## 21) Definition of Done

A general multi-year deal processor is considered complete when:

1. it can distinguish between contract term and billing schedule
2. it prefers explicit period schedules only when they are usable
3. it falls back safely when explicit schedules are empty or invalid
4. it supports at least one derived-period model
5. it can allocate amounts using an explicit and documented strategy
6. it produces normalized outputs suitable for downstream use
7. it records which decision path it used
8. it passes reconciliation and edge-case tests
9. it is not tightly coupled to a single vendor, file layout, or implementation tool

---

## 22) Short Build Brief for an Engineering Agent

If you are building multi-year deal logic from this document:

1. Start by separating parsing, validation, decision, allocation, and output responsibilities.
2. Support both explicit period schedules and derived period logic.
3. Treat explicit period data as authoritative only when at least one period row is actually usable.
4. If explicit period data is empty or invalid, fall back to the configured derived logic.
5. Keep price-source decisions explicit and configurable.
6. Use overlap-based processing when item terms and billing periods can differ.
7. Add audit logging so strategy choice and fallback behavior are visible.
8. Make the system generic enough to survive changes in vendor, format, or technology stack.

---

## 23) Final Rule to Remember

> **Explicit periods win only when they are real and usable. Otherwise, derive periods or process the item as a single term according to configuration.**

This rule should guide any future multi-year deal implementation.
