# DataOps TestGen: an evaluation for Trellis

**Status:** draft for discussion with the owner. 2026-10-18.
**Subject:** DataKitchen DataOps TestGen 5.134.0 (`3606859c`), the owner's fork, Apache-2.0.
**Answers:** the open question in Resource Explorer's
`docs/design-notes/REQUIREMENTS-RE-DATA-CHANNELS.md` §6: "Is it usable as a library, or only
as an application with its own results store?"

## Short answer

1. **Worth integrating?** Yes, but only part of it. TestGen fills the exact gap that RE's
   data-channels note names in §5: measured value profiles, quality tests generated from the
   profile, and re-runs over time. Everything else in TestGen duplicates something Trellis
   already has, and it does so under different rules: UI, scheduler, users, its own store,
   scores, notifications and MCP.
2. **Docker or absorb?** Neither whole. **The recommendation is a hybrid.** Take TestGen's
   **test-type catalogue as data**: 89 YAML files, each with per-dialect SQL, plus the
   per-dialect profiling queries. Run them from a narrow RE step behind the
   `EngineCapabilities` seam, with Postgres first. Publish the results as Egeria
   `QualityAnnotation`s through RE's existing publisher and identity path. Do not adopt the
   engine, the metadata DB, the UI or the auth.
3. **A new tool?** No. This is a capability inside RE: an Assessment-tier analysis plus a
   value-profile step. It still needs a design pass before code. The main questions are
   where test *definitions* live and how they map to Egeria. That pass could go to Fable if
   the owner wants, but it is a slice design, not a new product.

## What TestGen is (facts)

| Area | Finding |
|---|---|
| Test types | **89** types, one YAML each (`testgen/template/dbsetup_test_types/`). 69 are CAT, batched into one aggregate SELECT per table; 16 are QUERY, one SQL statement each; 4 are METADATA, which query TestGen's own DB. Scopes: 67 column, 11 table, 8 referential, 2 custom, 1 table group. |
| Dialects | 11 SQL families across 17 connection codes: Postgres, Redshift (and Spectrum), MSSQL (with Azure, Synapse and OneLake), Snowflake, Databricks, BigQuery, Oracle, SAP HANA, Salesforce Data 360, and DuckDB (with files, S3 and Azure Blob). No flavour is gated as enterprise. |
| Parameters | These are fixed columns on `test_definitions`: baselines, `threshold_value`, lower and upper tolerance, subset, group-by and having conditions, match-side columns, `custom_query`, and severity (Fail, Warning or Log). Placeholders are `{BASELINE_CT}`, `{THRESHOLD_VALUE}` and similar, about 40 keys (`execute_tests_query.py::_get_params`). |
| Generation | Each test type carries a `selection_criteria` SQL fragment, which is applied to `profile_results` of the latest profile. `default_parm_values` copies profile measures into the baselines. Generation sets are Standard (34), Table Comparison (32), Monitor (3) and Contract (1); 20 types are manual only. |
| Profiling | Pushed-down SQL per column and per dialect. It measures counts, lengths, patterns, frequent values, min, max, mean, standard deviation, percentiles, date buckets, a PII flag, and the functional type. Sampling is `TABLESAMPLE BERNOULLI` at 30%, capped at 999k rows. Value hashes (multiset, epoch and keyed) were added 2026-10-13 and are off by default. |
| Hygiene issues | 33 anomaly types as SQL predicates over `profile_results`, including Potential_PII. |
| Monitors | Freshness, Volume and Schema Drift, with thresholds predicted by statsmodels SARIMAX. No prophet or sklearn. |
| Runtime | Jobs are rows in `job_executions`. A custom cron scheduler polls every 5 s and spawns `testgen exec-job` subprocesses. `run-app all` runs three processes: Streamlit UI, scheduler, and FastAPI with MCP mounted at `/mcp`. |
| Own store | **Postgres only**, schema `testgen` by default (configurable), about 50 tables, **136 migration scripts**. It expects its own database name and a schema-owning admin user. |
| API and MCP | REST `/api/v1` covers runs, results, profiling, schedules, notifications, and test-definition import and export. MCP has about 110 tools, with reads and writes, plus prompts. The MCP pin is `mcp==1.28.1`; Trellis requires `>=2.0`. |
| Auth | Its own `auth_users` (bcrypt) and OAuth 2.1 tokens and PATs. **The open-source RBAC grants every permission** (`utils/plugins.py:38`). It has no SSO or OIDC. Auth and RBAC can be replaced by a `testgen_*` plugin. |
| Target credentials | Stored per connection and encrypted. No per-user identity, the same as RE today. |
| Defaults to fix if run | **Mixpanel telemetry is on** (`TG_ANALYTICS`, `settings.py:535`). There is a release check. The API binds 0.0.0.0 (`settings.py:741`). Compose publishes `8501:8501` and the local DB `5432:5432` on all interfaces. |
| Cadence | 1,398 commits so far in 2026; 5.92.2 to 5.134.0 in the latest release. The schema migrates often. |
| Licence | Apache-2.0. NOTICE is three lines (DataKitchen, Inc., 2024). Bundles Highlight.js (BSD-3) and VanJS (MIT) in the UI only. There is no trademark grant, so do not present derived work as "TestGen". |

**Is the logic separable?** As shipped, no. The engine reads its templates from its own DB
copies, writes `test_results`, and computes severity and status in app-DB SQL. Monitors and
hash tests need history and baselines from that DB. **The SQL itself is mostly
self-contained**: parameterised templates plus a dict of placeholders and a few dialect
helpers. The separable asset is the catalogue, not the engine.

## Overlap with RE

| Concern | RE today | TestGen |
|---|---|---|
| Null and distinct | `row_count_snapshot` reads `pg_stats`, an estimate as fresh as the last `ANALYZE` | Measured, sampled or full |
| Min, max, lengths, patterns, distributions | **None** | Yes |
| PII and data class | `data_class_match` against Egeria DataClasses, sampled, with absence handled honestly | `pii_flag` and anomaly 1100, from heuristics |
| Schema drift | `schema_diff` | `Schema_Drift` (a METADATA type, needs its own DB) |
| "Staleness" | `db_statistics_freshness` measures *planner statistics* staleness | `Table_Freshness` and `Freshness_Trend` measure *data* freshness. **These are different things that share a word.** |
| Quality tests and thresholds | **None** | 89 types |
| Engines | Postgres only | 17 connection codes |
| Results | RE registry, then a `SurveyReport` with annotations in Egeria | Its own store only (plus a DataKitchen Observability export) |

RE's sampling machinery (head, random, systematic, stratified; budgets; "not sampled" is
never "no match") is already sound. TestGen adds *what to measure and test*, not *how to
sample*.

## Options

### A. Run TestGen whole in Docker and call its API or MCP

- **For.** You get 17 engines, monitors and SARIMAX on day one. Upgrading means pulling an
  image.
- **Against: identity.** It runs a second user system. RE would call it with a PAT, a
  service identity for user actions, which breaks RE's rule. The only fix is a `testgen_*`
  auth plugin that validates Egeria bearer tokens, which is custom code inside their plugin
  surface.
- **Against: catalog of record.** Results live in TestGen's store. Egeria sees them only
  through a sync that RE would write: poll runs, map them to annotations, and reconcile.
  You would have two sources of truth and RE would own the reconciliation.
- **Against: duplicated state.** Connection credentials would live in two encrypted
  stores, and two schedulers would run, RE's and TestGen's.
- **Against: operational load.** Another stack: an engine container with three processes
  and Postgres 14. Every one of its defaults has to be overridden: telemetry off, release
  check off, ports bound to `127.0.0.1`, and the external `datakitchen-network`.
- **Against: coupling.** The API surface is young and moves fast. The MCP pin conflicts
  with Trellis, which is harmless across a process boundary but shows the drift. Authoring
  tests through the REST API is thin (import and export); most authoring is in MCP or the
  UI.
- **Against: engine host.** It does not fit. An Egeria-coordinated step would become
  "ask TestGen, wait, poll, translate", a long-running remote job inside a step.

### B. Absorb the engine into Trellis code

- **For.** One store, one identity, one scheduler.
- **Against.** The engine is welded to its metadata DB: CAT batching, generation from
  `profile_results`, severity mapping in SQL, and 136 migrations. Porting it means
  rewriting it, and from then on you own a fork that drifts from 1,400 commits a year.

### C. Hybrid: the catalogue as data, executed by RE (recommended)

- **Vendor the data, not the engine.** Copy `dbsetup_test_types/*.yaml`,
  `dbsetup_anomaly_types/*.yaml` and the per-dialect profiling SQL into a Trellis package
  (for example `trellis-quality` or a directory under RE). Keep NOTICE and the licence, and
  add a provenance header naming the upstream commit.
- **Write a thin executor.** It loads a test type, fills the placeholders from RE's own
  value profile, runs the SQL through RE's connection and capability layer, and maps
  `result_code` to RE's result vocabulary: pass, fail, warn, error, `not_established` and
  training. CAT batching is an optimisation; one statement per test is fine for a first
  slice.
- **Store results the RE way.** They go in the registry, append-only and stamped with
  `surveyed_at`. They are published as Egeria `QualityAnnotation` (0640) on the
  `SurveyReport`: `qualityDimension` is the test type, `qualityScore` is the measure,
  `qualityDescription` is the message. The investigation-framing design already plans this
  mapping. The existing `QualityScoreAnnotation` carries only a float dict, so it needs a
  richer variant or a new registry entry that follows Egeria's type.
- **Identity comes free.** Publishing goes through `egeria_identity.apply_identity`: user
  actions run as the user, and the worker or engine host runs as a daemon with
  `requested_by`.
- **It fits the engine host as a built-in database step.** The `trellis-analytics` library
  path cannot open a connection or emit anything but `ResourceMeasureAnnotation`, so this
  has to be built-in, beside `postgres_column_profile`.
- **Upgrades become a data diff.** A script compares the fork's YAML against the vendored
  copy and reports new or changed types and placeholders. You adopt deliberately, not by
  pulling an image.
- **Defer** monitors and SARIMAX, Table Comparison and hashes, and the METADATA types. RE
  already does schema drift its own way.
- **Keep one door open.** The data-channels note's "external service as a channel" column
  stays valid. If a non-Postgres engine is needed before RE has an adapter for it, a
  TestGen container could serve as that channel *behind the same seam*. That is a later
  decision, not part of this one.

**Cost of C.** A slice of roughly: a value-profile step that measures, not estimates; an
executor for about 15 to 20 column and table types; a mapping to annotations and a registry
table; an analysis-catalogue entry; and the YAML sync script. **Risk:** the YAML schema or
placeholder names change upstream. The diff script exists to catch that.

## Owner decisions (2026-10-19)

**Option C is accepted.** The design questions below follow from it.

### Who coordinates a run

TestGen's runner is not used. The run is three linear steps: **value profile → propose
checks → run checks**. The coordinator is whichever of RE's two coordinators applies:

- **Egeria coordinates.** A Survey Definition (`GovernanceActionProcess`) hands each step to
  RE's engine host, which runs it as `Daemon(ENGINE_HOST, requested_by)`. Built-in
  connection-opening steps are declarable through `engine-host/steps/`
  (`engine_host.py:177`). The precedent is `processes/postgres-survey-then-derived.md`.
- **RE coordinates.** `SurveyDefinitionExecutor` runs the steps locally or on Prefect, and
  RE's scheduler handles re-runs. Automate subscriptions then come free.

"Propose" is its own step, so "no checks proposed" can never read as "all passed". "Run
checks" uses the last *accepted* baseline and records which one.

### Where check definitions live: proposed in RE, accepted in Egeria (recommended)

Egeria already has the type. **`DataQualityRule`** (model 0450, supertype
`GovernanceRule`) is in the 6.2 types archive, added by `571034df11` (2026-09-06, ODCS
support). That commit is an ancestor of the quickstart's `85d6c48a`, so the running platform
has it. Its attributes:

- `qualityDimension`, `checkType` (text, library, sql or custom), `metric`, `severity`,
  `method`, `units`, `expression`;
- `comparisonOperator` (`mustBe`, `mustBeLessOrEqualTo`, `mustBeBetween`, and so on);
- `thresholdValues` (a list of strings);
- `schedule` and `scheduler`, and `qualityEngine`.

A rule links to the field or structure it checks through `GovernedBy`. Egeria's Bitol
connector maps one ODCS check to one `DataQualityRule` in exactly this way.

| Option | Verdict |
|---|---|
| Regenerated from the profile every run, never stored | **No.** The baseline moves with the data, so drift can never fail. TestGen's own `lock_refresh` exists because of this. |
| RE registry only | Works, but a governance definition would live outside the catalog of record. |
| Every generated check written to Egeria | Floods Egeria: 200 columns × about 5 checks is 1,000 unreviewed rules. It also writes machine guesses as governance. |
| **Proposed in RE, accepted into Egeria** | **Recommended.** It is RE's existing pattern: DRAFT proposals, as `grain_determination` and `data_class_match` already make, then Curate-accept materialises them, as `workflows/curate.py` does with `Ownership` set to the accepting user. |

How it works:

- **Proposals** go in an RE registry table that is append-only and records the profile run
  that produced each one. They use TestGen's parameters: baselines, threshold, tolerances,
  severity.
- **Accepted checks** become one `DataQualityRule` per check, linked by `GovernedBy` to the
  column element. The mapping:
  - `checkType=library`;
  - `method` = the test-type code, for example `Missing_Pct`;
  - `qualityEngine` = Trellis's name for the engine, not "TestGen";
  - `metric`, `comparisonOperator` and `thresholdValues` from the parameters;
  - `qualityDimension` from the YAML's `dq_dimension`.

  `thresholdValues` holds bounds only, so the baseline and tolerance go in
  `additionalProperties`. A human edit in Egeria wins over regeneration, which is the same
  meaning as `lock_refresh`.
- **Results** are a `QualityAnnotation` per check per run, carried in a `SurveyReport`.
  `AssociatedAnnotation` links it to the column. The status and the measure go in
  `jsonProperties`, because `QualityAnnotation` has only an integer `qualityScore`.
- **Absence** has three distinct states:
  - a column with no accepted rule is *not governed*;
  - a rule that did not run is *never run*;
  - a TestGen `result_code` of -1 is *training*.

  None of these is "passing".

**Open items:**

- **No pyegeria convenience API for `DataQualityRule`.** The only reference is in
  `md_processing_utils/element_status_fields.py`, and there is no Dr.Egeria command. A
  generic governance-definition create is exactly what CLAUDE.md rule 12 warns about. This
  needs a pyegeria spec, like `automate-notification-manager-pyegeria-spec.md`.
- **No relationship links an annotation to the rule it evaluated.** For now, carry the
  rule's GUID and qualified name in `jsonProperties`. Ask the Egeria team whether a link
  type is intended.
- **What does `GovernedBy` point at?** The physical column is a `RelationalColumn` and
  exists only if the database was catalogued. The design-level `DataField` is the target
  Egeria's ODCS mapping uses. To be settled in the design pass.
- **Possible later route.** TestGen exports ODCS (`testgen/common/odcs/`) and Egeria imports
  ODCS into `DataQualityRule`s. That route is not needed for C.

### First slice: eight checks, Postgres only

The selection rule:

- generated from the profile (TestGen's Standard set);
- a CAT type, so a single aggregate SQL expression;
- no TestGen app-DB history, no prediction, and **no dependency on TestGen's
  `functional_data_type` or pattern inference**;
- no overlap with an analysis RE already has.

| Test type | Dimension | Selection (from profile) | Default |
|---|---|---|---|
| Missing_Pct | Completeness | `record_ct <> value_ct` | Cohen's H ≥ 2, warning |
| Required | Completeness | no nulls and more than 10 rows | nulls > 0, fail |
| Required_No_Placeholder | Completeness | text, fully filled | placeholder values (n/a, tbd, …) > 0, fail |
| Unique | Uniqueness | more than 500 rows, all distinct | duplicates > 0, fail |
| Future_Date | Timeliness | date column with no future dates | future dates > 0, fail |
| Min_Date | Validity | date column with a minimum date | dates before the baseline > 0, fail |
| Recency | Timeliness | date column, maximum within 62 days | stepped day limit, warning |
| Row_Ct_Pct | Volume | table (manual-only in TestGen; RE proposes it from its row-count baseline) | change > threshold %, fail |

**Slice 2** ports TestGen's functional-type and pattern inference, which opens Email_Format,
Pattern_Match, LOV_Match, Min_Val, Distinct_Value_Ct, Unique_Pct and Valid_US_Zip. **Later:**
the shift and outlier tests, monitors with SARIMAX, Table Comparison and hashes, and
user-authored Dupe_Rows, Text_Length_Range, Max_Val and Max_Date.

**A choice that keeps upgrades cheap.** Store RE's value profile under **TestGen's
`profile_results` column names**: `record_ct`, `value_ct`, `general_type`, `future_date_ct`,
and so on. The YAML `selection_criteria` and `default_parm_values` fragments can then run
verbatim against RE's table, instead of being re-coded. They use helpers such as `DATEDIFF`
that TestGen's setup defines, and those come along as vendored SQL.

## Decisions for the owner (original list)

1. Accept C, or prefer A for breadth of engines now?
2. Where do test *definitions* live? Options: RE's registry, Egeria governance definitions,
   or generated on the fly from the profile and never stored. This is the main design
   question, and the one most worth a Fable pass.
3. The annotation shape: follow Egeria's `QualityAnnotation` as is (the owner's stated
   preference), or ask the Egeria team for a richer data-quality result type?
4. A first-slice test list. *Superseded by "First slice" above.* The first draft listed
   several types that are manual-only or that need TestGen's functional-type inference.
5. An optional spike: run TestGen once in Docker, bound to `127.0.0.1` with telemetry off,
   against a throwaway Postgres, and compare its profile with RE's on the same tables. This
   needs the owner's go-ahead. Note that RE's API-reference registration on 2026-10-16
   fetched `:8530` from a running instance, so some instance already ran on this machine.
   Check how it was bound.
