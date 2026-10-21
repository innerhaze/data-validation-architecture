# Data validation architecture

Design notes on how a multi-layer validation stack is organised for a client-facing data delivery pipeline. Written up from field experience building and maintaining such a stack across several code migrations.

The focus is the architecture: which layers run, in which order, what each one is responsible for, and the reasoning behind decisions that were made along the way.

## The layers in order

A delivery that reaches a client has moved through at least five validation layers after the data was built and before it was shipped. Each layer runs on the output of the previous one, so a failure late in the stack does not require re-running earlier layers.

### 1. Field-level invariants

The first layer runs per field on each row of the output. It is a library of deterministic rules — type validity, null-pattern checks, format regexes, chain or category assignment preservation, cross-field consistency on the few fields where values must agree. The rules are expressed in a JSON configuration so that adding or removing a check does not require touching the engine that runs it.

**Catches:** malformed field values, type regressions, violations of a documented preservation rule (for instance, a chain assignment that should never be stripped by cleanup logic).

### 2. Schema conformance

The second layer runs once over the output as a whole, not per row. It checks that every column defined in the delivery schema actually ships, that nothing undocumented has leaked in, and that column ordering is what the contract says it is.

**Catches:** missing columns, extra columns from an upstream renaming or a merge of two code paths, column-order drift that would silently misalign a downstream consumer that reads by position rather than by name.

### 3. Value-level distribution profiling

The third layer looks at each column's distribution independently of the row-level rules. For each column it records null rate, unique rate, modal value, and the shape of the pattern distribution. A language-model pass then consolidates all findings across all columns into a single review object, rather than reviewing each column in isolation.

**Catches:** a field whose null rate has jumped between runs, a column whose modal value is suddenly the empty string, a pattern distribution that has flipped from mostly phone numbers to mostly nulls.

### 4. Post-cleanup pattern scan

The fourth layer runs over the final file, after the earlier layers have had a chance to repair what they could. It is a scan for anomaly patterns at the record level that only become visible once the row is finished. The patterns include text that is nominally a business name but is really a floor label or a unit label, text where the business name is literally the street address or the city name repeated, and container-plus-tenant collisions where a shopping-centre record has merged with its tenants because an upstream index tagged the tenants with the container's name.

**Catches:** garbage-text that would reach a client-facing field, merge collisions that are below the de-duplication threshold and therefore not caught by the de-duplication report, residual duplicates that slipped through earlier layers.

### 5. Three-way congruence

The last layer runs after the file has been delivered and compares three artefacts against each other: what the database contains, what the delivery export contains, and what the post-delivery scan produced. Row counts have to match, identifiers have to be present in all three, and no artefact can contain a row that points at a merged-child identifier from either of the others.

**Catches:** samples still pointing at the losing side of a merge after the merge ran (which would ship duplicated rows), identifiers that exist in the export but not in the database (which usually means an upstream ingestion wrote directly to the file), identifiers that exist in the database but not in the export (which means a filter removed something it should not have).

## Why the order matters

The layers are deliberately ordered from cheapest and most local to most expensive and most global. Field-level rules can run in milliseconds per row during the build. Schema conformance is a single scan of the header. Value-level profiling is more expensive but is still intra-file. The pattern scan reads the whole file but does not join against anything external. The three-way congruence is the only layer that touches the database again after the build, and is the only one that produces findings that cannot be fixed in-place without re-running an upstream step.

Running the layers out of order would waste work. A column that fails schema conformance should not go through value-level profiling, because the profile would describe a column that is not going to ship. A row with a chain-id regression should be caught before the pattern scan looks at its name.

The order also means that each layer only sees defects the earlier layers could not fix. A pattern-scan finding is by definition one the field-level and schema layers accepted. That is useful for maintaining the earlier layers: the patterns that keep reaching the scan layer are candidates for new field-level rules.

## Design decisions along the way

**Deterministic rules before the language model.** Any check that can be expressed as a rule runs as a rule, before the model is called. This cuts model budget on the common cases and removes variance on the ones that have a correct answer by construction. The model only runs when a rule cannot decide.

**Default-on, not opt-in.** The schema conformance layer was originally behind a flag. It was flipped to default-on because an opt-in safety net is a safety net that routinely gets forgotten in the heat of a delivery. If a specific delivery really has to disable a layer, the disablement is explicit in the configuration and visible in the audit trail.

**Separate pattern expansion from model verdict.** The pattern-scan layer internally separates its rules into two modules. One module widens the pattern coverage — regexes and heuristics that expand what counts as "this looks like a floor label". The other module narrows: it hands a candidate to a language model and asks for a verdict on this specific record. Keeping the two modules apart forces the author to pick the right tool for each new check, and makes it obvious to a reader which rules are deterministic and which are probabilistic.

**Consolidated findings pass.** Rather than ask the model to review each column independently (which produces N independent calls and inconsistent reasoning across them), the value-level layer runs a single model pass that sees all column findings together and returns one coherent summary. One call instead of many, and the reviewer can see cross-column patterns that per-column calls would miss.

**Port fixes bidirectionally when the system migrates.** During a migration between code repositories, bug fixes landed in the pre-migration code shortly before cutover were explicitly ported to the post-migration code within the layers I owned, with each pair of commits referencing the other. The alternative — "fix it in the new place, the old place is going away" — leaves the old code wrong for the window between fix and cutover.

**A decision taken against: wiring the congruence layer as a hard gate.** The three-way congruence layer runs as an auditor-run tool, not as a build-time hard gate. The reason is that a congruence failure usually means "re-check the state of the database" rather than "block this delivery" — the delivery is often correct and the database state has drifted. A hard gate here would mostly block correct deliveries.

## Defect classes caught that previously escaped

The classes below are the ones this architecture is designed to catch. Each is a real failure mode in location-data delivery, not a hypothetical:

- A chain-assignment field being silently stripped on records that had both a null address field and a known duplicate on the same site. Caught by the field-level preservation rule.
- Business names that are really unit labels, floor labels, or the city name repeated as the business name. Caught by the pattern-scan layer.
- A shopping-centre record merging with its tenants, below the de-duplication report's score threshold. Caught by the container-plus-tenant collision rule.
- Join-table rows pointing at merged-child identifiers after a merge ran, which would duplicate rows in the client-facing output. Caught by three-way congruence.
- A record whose business name is literally its street address, usually a scraping bug.

## What this is not

This is not a schema registry. It does not provide a general-purpose validation DSL. It describes how to organise validation work, not how to build the validator library.

This is also not a monitoring layer. The checks are gates, not observability. A monitoring layer is a reasonable thing to add on top, but it answers a different question ("has the data drifted over time?") than the one the gate layers answer ("is this specific delivery correct?").

## Scope of my contribution

This write-up describes a validation stack I helped operate and expand over
roughly fifteen months at a location-data company. Not every layer is my design.

The field-level invariants layer (1) and the schema conformance layer (2)
were authored by a colleague on the pre-migration codebase, before I began
working on the stack. My contributions to those two layers were targeted
rule additions, a default-on flip on the conformance check, and ports
across the code migration.

The value-level distribution profiling layer (3), the post-cleanup pattern
scan (4), and the three-way congruence check (5) are my design and
implementation. The pattern-scan layer was later extracted into a
standalone methodology package I own, which the architecture notes describe
in its post-cleanup role.

The ordering, the deterministic-rules-before-model principle, the
default-on decision, the bidirectional-port rule during migrations, and
the decision against hard-gating the congruence layer are all choices I
made across the stack regardless of who authored the layer.
