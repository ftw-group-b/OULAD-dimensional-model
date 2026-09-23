# OULAD dbt Mart: Complete Execution Guide

This guide explains how to configure, run, test, and verify the dbt part of the
OULAD pipeline.

The dbt step starts only after the Silver layer has been built and validated.
It reads the seven clean Silver tables and creates the five dimensions and two
fact tables in the Gold mart.

```text
OULAD CSV files
      ↓
Bronze: 01-raw
      ↓
Silver: 02-clean
      ↓
dbt build
      ↓
Gold mart: 03-mart
      ↓
Analytics: 04-analytics
      ↓
Databricks dashboards
```

## 1. Did we use SQL or Python?

The dbt transformations use **SQL with Jinja**, not Python models.

- SQL performs the transformations, joins, casts, hashing, and aggregations.
- Jinja is the `{{ ... }}` syntax used by dbt for functions such as `source()`
  and `ref()`.
- YAML configures the project, declares sources, documents models, and defines
  tests.
- One SQL/Jinja macro customizes the output schema name.
- Python is needed only for optional local execution. In our actual pipeline,
  Databricks runs the dbt task in its managed workflow environment.

The repository contains a separate PySpark source-profiling notebook, but that
notebook is not part of the dbt mart.

An easy way to remember the roles is:

> SQL builds the tables, YAML describes and tests them, Jinja connects the
> models, and Databricks stores and processes the data.

## 2. Files used by dbt

```text
dbt_project.yml
profiles.yml.example
requirements-dbt.txt

macros/
└── generate_schema_name.sql

models/
├── staging/
│   └── sources.yml
└── mart/
    ├── dim_student.sql
    ├── dim_course.sql
    ├── dim_module_presentation.sql
    ├── dim_date.sql
    ├── dim_demographics.sql
    ├── fact_assessments.sql
    ├── fact_vle_interactions.sql
    └── schema.yml

tests/dbt/
├── assert_assessment_measures.sql
├── assert_fact_course_presentation_consistency.sql
└── assert_vle_measures.sql
```

The active paths are defined in `dbt_project.yml`:

```yaml
model-paths: ["models"]
test-paths: ["tests/dbt"]
macro-paths: ["macros"]
```

Because of this configuration, the files under `src/mart/`, the numbered
Databricks validation files, and the pipeline notebooks are not dbt models.

For our actual Databricks Workflow run, the core files are `dbt_project.yml`,
`models/`, `macros/`, and `tests/dbt/`. `profiles.yml.example` and
`requirements-dbt.txt` support optional local execution; we did not manually
use them when Databricks ran the connected dbt task.

## 3. What dbt builds

| Model | Grain: one row represents | Main Silver input |
|---|---|---|
| `dim_student` | One student | `student_info_clean` |
| `dim_course` | One course or module | `courses_clean` |
| `dim_module_presentation` | One module offering | `courses_clean` |
| `dim_date` | One relative course day | Assessment, submission, and VLE dates |
| `dim_demographics` | One demographic profile | `student_info_clean` |
| `fact_assessments` | One student assessment submission | Student assessments, assessments, and students |
| `fact_vle_interactions` | One student, site, presentation, and relative day | Student VLE, VLE resources, and students |

All seven models are materialized as physical Delta tables in:

```text
ftw-week-07.03-mart
```

## 4. How we actually executed dbt

We used a **dbt task inside a Databricks Workflow**. Databricks was already
connected to the repository and SQL warehouse.

We did not need to:

- download dbt to a personal computer;
- open a local terminal;
- create a local Python virtual environment;
- install `dbt-databricks` manually;
- create a personal `profiles.yml`; or
- paste a Databricks token into the project.

Databricks checked out the project, supplied the dbt runtime and warehouse
connection, ran the dbt command, and displayed the model and test logs inside
the Workflow run.

## 5. Databricks Workflow prerequisites

Before the dbt task starts, confirm that:

1. The repository is connected to the Databricks Workflow.
2. The Workflow can locate the project root containing `dbt_project.yml`.
3. A Databricks SQL warehouse is selected for the dbt task.
4. Bronze and Silver completed successfully.
5. The seven Silver tables exist in `ftw-week-07.02-clean`.
6. The Workflow run identity can read `02-clean` and create or replace tables
   in `03-mart`.

Required Silver tables:

```text
courses_clean
assessments_clean
vle_clean
student_info_clean
student_registration_clean
student_assessment_clean
student_vle_clean
```

The `03-mart` schema must already exist unless the Workflow identity has
permission to create it.

## 6. Run Bronze and Silver first

The first two Workflow tasks run:

```text
1. notebooks/01_bronze_oulad.sql
2. notebooks/02_silver_oulad.sql
```

The dbt task depends on the Silver task succeeding. If Bronze or Silver fails,
the Workflow must not start dbt.

You can confirm the Silver inputs in Databricks SQL:

```sql
SHOW TABLES IN `ftw-week-07`.`02-clean`;
```

## 7. dbt task configuration in Databricks

The Workflow contains a task similar to:

```text
Task name: dbt_mart
Task type: dbt
Project source: connected Git repository
Project directory: repository root containing dbt_project.yml
SQL warehouse: the warehouse connected to the pipeline
Command: dbt build --select path:models/mart
Dependency: Silver task succeeded
```

The exact screen labels can vary, but these are the important values. The
project directory must point to the folder containing `dbt_project.yml`, not to
`models/mart` itself.

Because the Databricks dbt task supplies the connection, the repository's
`profiles.yml.example` is not required for this managed run. It is only a
reference for someone who wants to reproduce the build outside Databricks.

## 8. Run the dbt task

Start the Workflow, or rerun the `dbt_mart` task after Silver succeeds.
Databricks executes:

```bash
dbt build --select path:models/mart
```

You do not run this command on your laptop when using the connected Workflow.
It is the command configured inside the Databricks dbt task.

The command:

1. Reads the Silver declarations from `models/staging/sources.yml`.
2. Compiles the seven SQL/Jinja models.
3. Creates or replaces seven Delta tables in `03-mart`.
4. Runs the generic tests from `models/mart/schema.yml`.
5. Runs the selected custom SQL tests under `tests/dbt/`.
6. Marks the task as failed if a model or test fails.

Do not manually add `CREATE OR REPLACE TABLE` to the model files. Every model
contains a `SELECT` query describing the desired result. Because
`dbt_project.yml` configures the models as `table`, dbt generates the required
DDL for Databricks.

## 9. Review the Databricks task logs

Open the `dbt_mart` task in the Workflow run and check the dbt output.

Confirm that:

- all seven models completed successfully;
- the generic YAML tests passed;
- the three custom SQL tests passed; and
- the task finished successfully before the next task started.

Do not continue to Analytics when the dbt task reports `ERROR` or `FAIL`.

## 10. Continue after dbt

After `dbt_mart` succeeds, the Workflow runs:

```text
notebooks/06_run_after_dbt.sql
```

That notebook validates the mart, builds Analytics, validates Analytics, and
refreshes the data-quality dashboard views.

The actual production order is:

```text
1. Bronze notebook and Bronze validation
2. Silver notebook and Silver validation
3. Databricks dbt task: dbt build --select path:models/mart
4. notebooks/06_run_after_dbt.sql
5. Databricks dashboard refresh
```

The dashboards were created and viewed directly in Databricks. We did not use
Metabase in the actual implementation. The repository's `metabase/` folder is
optional reference material and is not part of the executed pipeline.

## 11. Optional local execution only

The following setup is not part of our actual Databricks Workflow. It is useful
only when another developer wants to reproduce or troubleshoot the dbt project
from a personal computer.

From the downloaded project root:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements-dbt.txt
```

Copy the example connection profile:

```bash
mkdir -p ~/.dbt
cp profiles.yml.example ~/.dbt/profiles.yml
```

Set the three local connection variables:

```bash
export DATABRICKS_HOST="your-workspace-server-hostname"
export DATABRICKS_HTTP_PATH="/sql/1.0/warehouses/your-warehouse-id"
read -s DATABRICKS_TOKEN
export DATABRICKS_TOKEN
```

Never commit the real token.

## 12. Optional local checks

When running locally, use:

```bash
dbt debug
dbt ls --select path:models/mart
dbt parse
dbt compile --select path:models/mart
dbt build --select path:models/mart
```

`dbt debug` verifies the local profile and connection. `dbt compile` writes the
compiled SQL under `target/`. Do not manually edit generated files under
`target/`.

## 13. What was and was not manually provided

For the connected Databricks Workflow, we provided:

- the repository containing the dbt project;
- the dbt task command;
- the selected SQL warehouse;
- the dependency on a successful Silver task; and
- the Workflow identity and its Unity Catalog permissions.

Databricks provided the execution environment and connection. The
`requirements-dbt.txt` and `profiles.yml.example` files remain useful for local
reproduction, but they were not steps we personally executed for the managed
pipeline.

## 14. SQL queries used in the models

### `dim_student.sql`

Reads distinct student IDs and creates a stable SHA-256 key.

```sql
select distinct
  sha2(cast(id_student as string), 256) as student_key,
  cast(id_student as bigint) as id_student
from {{ source('oulad_clean', 'student_info_clean') }}
```

### `dim_course.sql`

Reads distinct module codes and creates a course key.

```sql
select distinct
  sha2(code_module, 256) as course_key,
  code_module
from {{ source('oulad_clean', 'courses_clean') }}
```

### `dim_module_presentation.sql`

Creates one row for each module and presentation combination. It keeps the
presentation length and creates both course and presentation keys.

Main transformation:

```sql
sha2(concat_ws('||', code_module, code_presentation), 256)
```

### `dim_date.sql`

Combines assessment dates, submission dates, and VLE activity dates. It finds
the minimum and maximum relative day, generates every day between them, derives
the relative week, and assigns a course phase.

Main query pattern:

```sql
with source_dates as (
  select assessment_date as relative_day from {{ source('oulad_clean', 'assessments_clean') }}
  union all
  select date_submitted from {{ source('oulad_clean', 'student_assessment_clean') }}
  union all
  select activity_date from {{ source('oulad_clean', 'student_vle_clean') }}
)
```

OULAD contains relative day numbers instead of complete calendar dates, so
this dimension represents course-relative days.

### `dim_demographics.sql`

Creates one row for each distinct demographic profile. Missing demographic
attributes use `UNKNOWN` only when the key is created, preventing a null key.
The original descriptive value remains null when the source value is unknown.

Main key pattern:

```sql
sha2(
  concat_ws(
    '||',
    coalesce(gender, 'UNKNOWN'),
    coalesce(region, 'UNKNOWN'),
    coalesce(highest_education, 'UNKNOWN'),
    coalesce(imd_band, 'UNKNOWN'),
    coalesce(age_band, 'UNKNOWN'),
    coalesce(disability, 'UNKNOWN')
  ),
  256
) as demographics_key
```

### `fact_assessments.sql`

Joins assessment submissions to assessment definitions and student
enrollments. It creates foreign keys and keeps the assessment type, weight,
banked indicator, and score.

Main joins:

```sql
from {{ source('oulad_clean', 'student_assessment_clean') }} as submission
inner join {{ source('oulad_clean', 'assessments_clean') }} as assessment
  on submission.id_assessment = assessment.id_assessment
inner join {{ source('oulad_clean', 'student_info_clean') }} as student
  on assessment.code_module = student.code_module
  and assessment.code_presentation = student.code_presentation
  and submission.id_student = student.id_student
```

Missing scores remain `NULL`. They are not changed to zero.

### `fact_vle_interactions.sql`

Joins daily student VLE interactions to VLE resources and student
enrollments. It creates the fact key and foreign keys and keeps activity type
and click count.

Main joins:

```sql
from {{ source('oulad_clean', 'student_vle_clean') }} as interaction
inner join {{ source('oulad_clean', 'vle_clean') }} as activity
  on interaction.code_module = activity.code_module
  and interaction.code_presentation = activity.code_presentation
  and interaction.id_site = activity.id_site
inner join {{ source('oulad_clean', 'student_info_clean') }} as student
  on interaction.code_module = student.code_module
  and interaction.code_presentation = student.code_presentation
  and interaction.id_student = student.id_student
```

The daily VLE aggregation happens in Silver. dbt turns the clean daily rows
into the final fact structure.

## 15. How `source()` and `ref()` are used

### `source()`

`source()` points to an existing table that dbt does not build.

Example:

```sql
{{ source('oulad_clean', 'student_info_clean') }}
```

dbt resolves this using `models/staging/sources.yml`:

```text
ftw-week-07.02-clean.student_info_clean
```

All seven mart model SQL files read Silver using `source()`.

### `ref()`

`ref()` points to a dbt model and creates a dependency.

In this project, `ref()` is mainly used by the relationship tests in
`schema.yml` and by the custom tests under `tests/dbt/`.

Example:

```yaml
relationships:
  arguments:
    to: ref('dim_student')
    field: student_key
```

This test confirms that a fact-table `student_key` exists in `dim_student`.

## 16. Tests executed by dbt

### Generic YAML tests

`models/mart/schema.yml` defines:

| Test | Meaning |
|---|---|
| `not_null` | A required field cannot be null |
| `unique` | A key cannot appear more than once |
| `relationships` | A fact foreign key must exist in its dimension |
| `accepted_values` | A value must belong to the approved list |

Examples include:

- primary keys must be unique and non-null;
- fact keys must match their dimensions;
- `assessment_type` must be `CMA`, `TMA`, or `Exam`.

### Custom SQL tests

A custom dbt test passes when its query returns zero rows. Every returned row is
an invalid record.

`assert_assessment_measures.sql` checks that observed scores and assessment
weights are between 0 and 100.

`assert_vle_measures.sql` checks that VLE click totals are positive.

`assert_fact_course_presentation_consistency.sql` checks that the direct
course key in each fact agrees with the course key of its referenced module
presentation.

Run tests without rebuilding the models:

```bash
dbt test --select path:models/mart
```

Normally, use `dbt build` because it builds and tests in dependency order.

## 17. Verify the dbt output in Databricks

After `dbt build` succeeds, run:

```sql
SHOW TABLES IN `ftw-week-07`.`03-mart`;
```

Expected tables:

```text
dim_student
dim_course
dim_module_presentation
dim_date
dim_demographics
fact_assessments
fact_vle_interactions
```

Check their row counts:

```sql
SELECT 'dim_student' AS table_name, COUNT(*) AS row_count
FROM `ftw-week-07`.`03-mart`.dim_student

UNION ALL

SELECT 'dim_course', COUNT(*)
FROM `ftw-week-07`.`03-mart`.dim_course

UNION ALL

SELECT 'dim_module_presentation', COUNT(*)
FROM `ftw-week-07`.`03-mart`.dim_module_presentation

UNION ALL

SELECT 'dim_date', COUNT(*)
FROM `ftw-week-07`.`03-mart`.dim_date

UNION ALL

SELECT 'dim_demographics', COUNT(*)
FROM `ftw-week-07`.`03-mart`.dim_demographics

UNION ALL

SELECT 'fact_assessments', COUNT(*)
FROM `ftw-week-07`.`03-mart`.fact_assessments

UNION ALL

SELECT 'fact_vle_interactions', COUNT(*)
FROM `ftw-week-07`.`03-mart`.fact_vle_interactions;
```

## 18. Post-dbt notebook details

When the dbt task finishes successfully, the Workflow starts this Databricks
notebook:

```text
notebooks/06_run_after_dbt.sql
```

It performs the remaining steps:

1. Validates the Gold mart.
2. Registers the mart relationships.
3. Builds the four Analytics tables.
4. Runs Analytics and cross-layer validation.
5. Refreshes the data-quality dashboard views.

The final production order is:

```text
1. notebooks/01_bronze_oulad.sql
2. notebooks/02_silver_oulad.sql
3. dbt build --select path:models/mart
4. notebooks/06_run_after_dbt.sql
5. Refresh the Databricks dashboards
```

Do not use `notebooks/00_run_full_pipeline.sql` as the assessed dbt route. It is
a Databricks-only demonstration runner that builds a SQL mirror of the mart.
The assessed implementation is the `dbt build` command followed by
`notebooks/06_run_after_dbt.sql`.

## 19. Generate dbt documentation

This section is optional. Generate the dbt documentation site in a local dbt
environment or a separate configured dbt task:

```bash
dbt docs generate
```

Open it from a local environment:

```bash
dbt docs serve
```

The generated site shows model descriptions, columns, tests, sources, and the
dependency graph.

Stop the local documentation server with `Control+C`.

## 20. Useful commands

The actual Workflow task uses `dbt build --select path:models/mart`. The other
commands are optional development or troubleshooting commands.

| Command | Purpose |
|---|---|
| `dbt debug` | Check project, profile, credentials, and connection |
| `dbt ls --select path:models/mart` | List the selected dbt resources |
| `dbt parse` | Validate project and Jinja structure |
| `dbt compile --select path:models/mart` | Generate executable SQL without running it |
| `dbt run --select path:models/mart` | Build models without running tests |
| `dbt test --select path:models/mart` | Test already-built models |
| `dbt build --select path:models/mart` | Build and test in dependency order |
| `dbt docs generate` | Generate the dbt documentation files |
| `dbt docs serve` | Open the generated documentation locally |
| `dbt clean` | Remove generated dbt folders such as `target/` and logs |

`dbt clean` removes local generated files. It does not drop the Databricks mart
tables.

## 21. Reading the dbt result

A successful run ends with model and test statuses such as:

```text
PASS
```

or:

```text
Completed successfully
```

Do not continue to Analytics if dbt reports `ERROR` or `FAIL`.

dbt generates run artifacts under `target/` in its execution environment,
including:

- `manifest.json`: project graph and metadata;
- `run_results.json`: model and test results; and
- `compiled/`: compiled SQL.

## 22. Troubleshooting

### Local-only: `profiles.yml` was not found

This error applies to optional local execution, not the connected Databricks
Workflow task.

Confirm the file exists:

```bash
ls ~/.dbt/profiles.yml
```

The profile name inside it must be `oulad_analytics`, matching:

```yaml
profile: oulad_analytics
```

in `dbt_project.yml`.

### Local-only: an environment variable is missing

Set `DATABRICKS_HOST`, `DATABRICKS_HTTP_PATH`, and `DATABRICKS_TOKEN` again in
the same terminal where you run dbt.

### The Workflow dbt task cannot connect

Confirm that:

- the SQL warehouse is running;
- the dbt task is connected to the correct SQL warehouse;
- the Workflow identity can use the warehouse and catalog; and
- the repository and project directory are available to the task.

For optional local execution, also confirm the host, HTTP path, token, and
network connection, then run `dbt debug` again.

### Source table was not found

Check that Silver completed successfully:

```sql
SHOW TABLES IN `ftw-week-07`.`02-clean`;
```

Also verify the catalog and schema in `models/staging/sources.yml`.

### Permission denied

Ask the catalog administrator to give the dbt principal permission to read
`02-clean` and create or replace tables in `03-mart`. Do not change the target
schema merely to hide a permission problem.

### dbt created the wrong schema name

Confirm that `macros/generate_schema_name.sql` exists and that
`dbt_project.yml` contains:

```yaml
+schema: 03-mart
```

The macro prevents dbt from combining the target schema with the custom schema
name.

### A generic test fails

Read the failing model, column, and test name in the dbt output. Examples:

- `unique` failure: investigate duplicate keys;
- `not_null` failure: investigate missing required values;
- `relationships` failure: investigate orphan foreign keys;
- `accepted_values` failure: investigate unexpected categories.

Do not disable the test just to make the run green. Fix the upstream data or
the transformation logic, then rerun `dbt build`.

### A custom test fails

Run the SQL from the named file in `tests/dbt/`. The returned rows are the
records that caused the failure. Investigate the matching Silver rows and the
model transformation before changing the test.

## 23. Updating a dbt model safely

When you need to change a Gold table:

1. Edit the matching file under `models/mart/`.
2. Keep the model as a `SELECT` statement.
3. Update `models/mart/schema.yml` if a column or test changes.
4. Add or update a custom SQL test for a new business rule.
5. Commit or sync the update to the repository connected to Databricks.
6. Rerun the Databricks `dbt_mart` task.
7. Confirm `dbt build --select path:models/mart` and every test passed in the
   task logs.
8. Run `notebooks/06_run_after_dbt.sql`.
9. Review downstream Analytics and dashboard results.

For optional local development, `dbt parse` and `dbt compile` can be used before
the Workflow run. Do not edit generated files under `target/` and do not commit
real credentials.

## 24. Short presentation explanation

> We used SQL for the dbt transformations. Each SQL model contains a SELECT
> query describing one dimension or fact table. We used YAML to declare the
> Silver sources, document the models, and define tests. We used one macro to
> make dbt write directly to the required 03-mart schema. We did not download
> dbt or run Python locally because dbt was already connected as a Databricks
> Workflow task. After Silver passed validation, the Workflow ran dbt build.
> dbt generated the required table DDL, created the five dimensions and two
> facts as Delta tables, and ran the generic and custom tests. We continued to
> Analytics only after every dbt model and test passed.

## 25. Completion checklist

- [ ] Bronze notebook succeeded.
- [ ] Silver notebook and Silver validation succeeded.
- [ ] The Databricks Workflow is connected to the correct repository.
- [ ] The `dbt_mart` task points to the project root and SQL warehouse.
- [ ] The Workflow identity can read `02-clean` and write to `03-mart`.
- [ ] `dbt build --select path:models/mart` passed in the task logs.
- [ ] All generic and custom dbt tests passed.
- [ ] Seven tables exist in `ftw-week-07.03-mart`.
- [ ] `notebooks/06_run_after_dbt.sql` succeeded.
- [ ] Analytics and data-quality views were refreshed.
- [ ] The Databricks business and data-quality dashboards show the latest
  `04-analytics` and `05-data-quality` outputs.
