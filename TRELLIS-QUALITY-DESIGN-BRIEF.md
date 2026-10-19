# Design brief: value profile and quality checks in Resource Explorer

**For:** the Fable design session. **From:** the TestGen evaluation session, 2026-10-19.
**Status:** brief only. The owner has accepted the direction; the design is yours to make.
**Read first:** `TRELLIS-EVALUATION.md`, next to this file in the TestGen clone
(`~/localGit/egeria-v6/dataops-testgen`, branch `trellis/evaluation-note`). It holds the
facts behind every claim here.

## The job

Design a Resource Explorer (RE) capability that profiles database column *values* and runs
quality checks derived from that profile. It uses DataKitchen TestGen's test catalogue as
vendored data and runs that catalogue with RE's own machinery. The deliverable is a design
note in the trellis repo,
`packages/resource-explorer/docs/design-notes/DESIGN-VALUE-PROFILE-AND-QUALITY-CHECKS.md`
(name negotiable), detailed enough that a Sonnet or Opus session can implement slice 1
from it.

## Already decided: do not reopen

1. **Hybrid option C.** Vendor TestGen's test-type YAML and per-dialect SQL *as data*. Do
   not adopt TestGen's engine, metadata DB, scheduler, UI, auth or MCP. Do not run TestGen
   as a service.
2. **RE is not the coordinator.** Egeria coordinates (a Survey Definition with steps run by
   RE's engine host), or RE coordinates through `SurveyDefinitionExecutor` on Prefect or
   locally. The run is three linear steps: **value profile → propose checks → run checks**.
3. **Check definitions are proposed in RE and accepted into Egeria.** Proposals live in an
   RE registry table. An accepted check becomes one Egeria `DataQualityRule`, linked by
   `GovernedBy` to the column element. Acceptance goes through Curate, with `Ownership` set
   to the accepting user.
4. **Results are `QualityAnnotation`s** in a `SurveyReport`, one per check per run.
5. **Slice 1 is Postgres only, eight checks:** Missing_Pct, Required,
   Required_No_Placeholder, Unique, Future_Date, Min_Date, Recency, and Row_Ct_Pct. RE
   proposes Row_Ct_Pct itself; TestGen has it as manual-only.
6. **Follow Egeria's types.** Where a type is missing a field, use `additionalProperties` or
   `jsonProperties` and raise it with the Egeria team. Do not invent types.

## The four things to design

### 1. The value-profile step

Measure values instead of estimating them. Today `row_count_snapshot` reads `pg_stats`.

- **Where.** A built-in database step beside `postgres_column_profile`
  (`surveyors/database/column_profile_step.py`), declared in `DATABASE_STEP_REGISTRY`
  (`survey_definition_adapter.py:550`) and in `engine-host/steps/`.
  - It has to be built-in: library steps in `trellis-analytics` cannot open a connection.
  - It needs `requires_capability: read` and a fetch cost; follow `data_class_match`.
- **What it measures.** The `profile_results` columns that the eight slice-1 selection
  criteria and defaults read: `record_ct`, `value_ct`, `distinct_value_ct`,
  `zero_length_ct`, `filled_value_ct`, `general_type`, `min_date`, `max_date`,
  `future_date_ct`, and `sample_ratio`. Derive the full list from the YAML rather than
  from this sentence. The source is TestGen's
  `testgen/template/flavors/postgresql/profiling/project_profiling_query.sql`.
- **The naming rule.** Store the measures under **TestGen's column names**, so the YAML
  `selection_criteria` and `default_parm_values` fragments run verbatim against RE's table.
  Some of those fragments use helper functions such as `DATEDIFF` that TestGen's DB setup
  defines; vendor those helpers too. This rule is what keeps upstream upgrades a data
  diff.
- **Sampling.** Reuse RE's `surveyors/database/sampling.py` (strategies and budgets), not
  TestGen's fixed 30% BERNOULLI. Two things to decide:
  - which measures are *invalid* on a sample. TestGen's `Required` and
    `Required_No_Placeholder` criteria demand `sample_ratio IS NULL`;
  - how a sampled measure is labelled.
- **Absence.** Use RE's `result_status` vocabulary. A column the step could not profile is
  `not_established`, never zero.
- **Storage.** Append-only and stamped with `surveyed_at`, like every other analysis
  result. Optionally publish a `ResourceProfileAnnotation` (0620) per column; decide
  whether slice 1 does this.

### 2. The proposal table and the propose step

- **Content.** The propose step evaluates each vendored test type's `selection_criteria`
  over the latest value profile and writes one proposal per (column, test type). Each row
  carries:
  - the parameters: baselines, `threshold_value`, tolerances, severity;
  - the profile run that produced it;
  - a status: proposed, accepted, rejected, superseded.
- **Shape.** A registry table in RE's "analysis results" family, append-only. Decide:
  - the identity key;
  - how a re-run supersedes an open proposal;
  - that a *rejected* proposal is never re-proposed unless the profile changes materially.
- **Vendoring.** Put the eight YAMLs and their per-dialect SQL in a Trellis-owned location.
  Decide where: a `trellis-quality` package, or under RE.
  - Each vendored file needs a provenance header with the upstream commit (`3606859c`),
    the Apache-2.0 licence, and the DataKitchen NOTICE.
  - Do not name the engine "TestGen"; there is no trademark grant.
  - Specify a sync script that diffs the vendored copy against the fork and reports new or
    changed types and placeholders.
- **The analysis-catalogue entry.** Assessment tier, since it evaluates against criteria.
  Add `check_registry.yaml` checks, one per test type.

### 3. Accepting a proposal into a `DataQualityRule`

- **The Egeria type.** `DataQualityRule` (0450), whose supertype chain is
  `GovernanceRule → GovernanceControl → GovernanceDefinition`. It is in the 6.2 archive
  (`OpenMetadataTypesArchive6_2.java:1697`) and on the running platform. Attributes:
  - `qualityDimension`, `checkType`, `metric`, `severity`, `businessImpact`, `method`,
    `units`;
  - `schedule` and `scheduler`, `expression`, `qualityEngine`;
  - `comparisonOperator` (`mustBe`, `mustBeLessOrEqualTo`, `mustBeBetween`, …) and
    `thresholdValues` (a list of strings).
- **The proposed mapping** (refine it):
  - `checkType=library`;
  - `method` is the test-type code;
  - `qualityEngine` is Trellis's engine name;
  - `qualityDimension` comes from the YAML `dq_dimension`;
  - `metric`, `comparisonOperator` and `thresholdValues` come from the parameters;
  - the baseline and tolerance go in `additionalProperties`.
- **Reference model.** Egeria's Bitol (ODCS) connector maps one check to one rule.
  `bitol-integration-connectors/README.md:175` and `DataContractMapper.java` show the
  mapping; align with them wherever you can.
- **The `GovernedBy` target.** Choose between the physical `RelationalColumn` and the
  design-level `DataField` that ODCS mapping uses. A `RelationalColumn` exists only if the
  database was catalogued; say what happens when it was not.
- **The flow.** Follow `workflows/curate.py`'s `materialize_component_if_accepted` (line
  730): ownership rules, zones (configured-only; no default publish zone), and read-back
  before claiming success.
- **Editing.** An edit made in Egeria wins over regeneration, which is TestGen's
  `lock_refresh` meaning. Specify how RE notices it.
- **Blocker to plan around.** pyegeria has **no** `DataQualityRule` API and Dr.Egeria has no
  command. Draft a pyegeria spec in the style of
  `docs/automate-notification-manager-pyegeria-spec.md`. Do not specify RE code that calls
  a generic `create_governance_definition` body; CLAUDE.md rule 12 warns against it.

### 4. The annotation shape for results

- **The Egeria types.** `QualityAnnotation` (0640) extends `DataFieldAnnotation`.
  - Its own attributes are `qualityDimension`, `qualityScore` (an integer) and
    `qualityDescription`.
  - It inherits `matchingValues`, `nonMatchingValues`, `expression`, `explanation`,
    `jsonProperties`, `confidence`, `sampleSize`, `samplePercent` and `samplingMethod`.
  - It has no pass/fail, threshold or severity field.
- **Requirements.** One per check per run, in the run's `SurveyReport`, with
  `AssociatedAnnotation` to the column. The design must decide:
  - **Status.** Where the status goes: pass, warning, fail, error, training (TestGen's
    `result_code` -1), and not run.
  - **Measure.** Where the measure and the threshold it was compared against go.
  - **Rule link.** How the annotation names its rule. No relationship type links an
    annotation to a governance definition, so carry the rule's GUID and qualified name,
    and raise the question with the Egeria team.
  - **Registry entry.** What changes in RE's `ANNOTATION_TYPES_REGISTRY` and in
    `annotation_props.py`. Today `QualityScoreAnnotation` carries only a float dict.
  - **Failures.** When a failure raises an RFA. Follow `investigation-framing-design.md`
    (around lines 313-325): RFA only for the Assess, Maintain, Certify and Remediate
    purposes, never for Select.
  - **Certification.** Whether a fully passing run sets the `data-certified` guard
    (`SurveyActionGuard`) for the Egeria-coordinated path.
- **Absence.** Four states must stay distinguishable:
  - not governed (no accepted rule);
  - never run;
  - training;
  - passed.

  Flattening any two of them is this codebase's most common bug class; see the
  `find-absence-as-answer` skill.

## Also cover, briefly

- **Identity.** Interactive runs publish as the user through
  `egeria_identity.apply_identity`. Engine-host and queued runs run as `Daemon(...,
  requested_by)` with `Ownership` set to the requester. Credentials for the surveyed
  database are per registration (`credential_crypto.py`); slice 1 needs no change there.
- **The `EngineCapabilities` seam.** Specify the slice so it does not deepen the Postgres
  assumption: per-dialect SQL comes from the vendored templates, keyed by flavour (roadmap
  item 1 in `REQUIREMENTS-RE-DATA-CHANNELS.md`).
- **UI.** Where proposals are reviewed: most likely an Assessment card plus Curate accept.
  Wireframe-level only.
- **Slices 2 and later.** One paragraph each:
  - TestGen's functional-type and pattern inference, which unlocks Email_Format,
    Pattern_Match, LOV_Match and others;
  - monitors with SARIMAX;
  - user-authored checks;
  - other engines.

## Rules for the design session

- **Design only.** Write design notes and specs; implement nothing.
- **Shared writes.** Before writing in `~/localGit/egeria-v6/trellis`, follow the
  `coordinate-shared-writes` skill, since other sessions share that checkout.
- **Egeria.** Make no Egeria writes. A read to confirm a type on the live platform needs a
  peer round first. The quickstart was being recreated on a new image on 2026-10-19; check
  it is back up.
- **TestGen.** Do not run TestGen's containers.
- **Secrets.** Open no `.env` or credential files.
- **Open questions** go to the owner as a short list at the end of the design note, not as
  guesses inside it.
