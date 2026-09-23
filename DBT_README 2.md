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
Dashboards
```

## 1. Did we use SQL or Python?

The dbt transformations use **SQL with Jinja**, not Python models.

- SQL performs the transformations, joins, casts, hashing, and aggregations.
- Jinja is the `{{ ... }}` syntax used by dbt for functions such as `source()`
  and `ref()`.
- YAML configures the project, declares sources, documents models, and defines
  tests.
- One SQL/Jinja macro customizes the output schema name.
- Python is only used to create the local environment and install the
  `dbt-databricks` package.

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

## 4. Prerequisites

Before running dbt, confirm that you have:

1. Python 3 installed on the machine that will run dbt.
2. Access to a running Databricks SQL warehouse.
3. Access to the `ftw-week-07` Unity Catalog catalog.
4. The complete repository downloaded or cloned.
5. A successful Bronze and Silver pipeline run.
6. These seven Silver tables in `ftw-week-07.02-clean`:

```text
courses_clean
assessments_clean
vle_clean
student_info_clean
student_registration_clean
student_assessment_clean
student_vle_clean
```

The user or service principal running dbt needs permission to:

- use the `ftw-week-07` catalog;
- read the `02-clean` schema and its tables; and
- use, create, replace, modify, and read tables in `03-mart`.

The `03-mart` schema must already exist unless the principal has permission to
create it.

## 5. Run Bronze and Silver first

Run these Databricks notebooks in order:

```text
1. notebooks/01_bronze_oulad.sql
2. notebooks/02_silver_oulad.sql
```

Do not start dbt if the Bronze or Silver quality gate fails.

You can confirm the Silver tables in Databricks SQL:

```sql
SHOW TABLES IN `ftw-week-07`.`02-clean`;
```

Optional row-count check:

```sql
SELECT 'courses_clean' AS table_name, COUNT(*) AS row_count
FROM `ftw-week-07`.`02-clean`.courses_clean

UNION ALL

SELECT 'student_info_clean', COUNT(*)
FROM `ftw-week-07`.`02-clean`.student_info_clean

UNION ALL

SELECT 'student_assessment_clean', COUNT(*)
FROM `ftw-week-07`.`02-clean`.student_assessment_clean

UNION ALL

SELECT 'student_vle_clean', COUNT(*)
FROM `ftw-week-07`.`02-clean`.student_vle_clean;
```

## 6. Open a terminal in the project root

The project root is the folder containing `dbt_project.yml`.

Example:

```bash
cd /path/to/OULAD-dimensional-model-main
```

Confirm the file is present:

```bash
ls dbt_project.yml
```

All dbt commands in this guide should be run from this directory.

## 7. Create a Python virtual environment

Using a virtual environment prevents the project dependencies from mixing with
other Python projects.

On macOS or Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

On Windows PowerShell:

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

When the environment is active, the terminal usually displays `(.venv)`.

## 8. Install dbt for Databricks

The repository uses `requirements-dbt.txt`:

```text
dbt-databricks>=1.10,<2.0
```

Install it:

```bash
python -m pip install -r requirements-dbt.txt
```

Confirm the installation:

```bash
dbt --version
```

The output should list `databricks` as an installed adapter.

## 9. Create the dbt connection profile

dbt normally reads its connection profile from:

```text
~/.dbt/profiles.yml
```

Create the folder and copy the provided example:

```bash
mkdir -p ~/.dbt
cp profiles.yml.example ~/.dbt/profiles.yml
```

The profile should contain:

```yaml
oulad_analytics:
  target: dev
  outputs:
    dev:
      type: databricks
      catalog: ftw-week-07
      schema: 03-mart
      host: "{{ env_var('DATABRICKS_HOST') }}"
      http_path: "{{ env_var('DATABRICKS_HTTP_PATH') }}"
      token: "{{ env_var('DATABRICKS_TOKEN') }}"
      threads: 4
```

Meaning of each setting:

| Setting | What to put |
|---|---|
| `type` | `databricks` |
| `catalog` | `ftw-week-07` |
| `schema` | `03-mart` |
| `host` | Your Databricks workspace server hostname |
| `http_path` | The HTTP path of the SQL warehouse |
| `token` | Your Databricks access token |
| `threads` | Number of models/tests dbt may run at the same time |

Get the host and HTTP path from the connection details of the SQL warehouse.
Use a token belonging to the user or service principal authorized to run the
pipeline.

Do not paste a real token into the repository, `profiles.yml.example`, or a
GitHub commit.

## 10. Set the connection values

For a macOS or Linux terminal:

```bash
export DATABRICKS_HOST="your-workspace-server-hostname"
export DATABRICKS_HTTP_PATH="/sql/1.0/warehouses/your-warehouse-id"
read -s DATABRICKS_TOKEN
export DATABRICKS_TOKEN
```

After the `read -s` command, paste the token and press Enter. The token is not
displayed while you type.

Confirm that the variables exist without printing the secret:

```bash
test -n "$DATABRICKS_HOST" && echo "Host is set"
test -n "$DATABRICKS_HTTP_PATH" && echo "HTTP path is set"
test -n "$DATABRICKS_TOKEN" && echo "Token is set"
```

These values last only for the current terminal session. A Databricks Job or CI
environment should provide the same variables through its secret-management
feature.

## 11. Test the connection

Run:

```bash
dbt debug
```

Continue only when the final connection test succeeds.

`dbt debug` checks:

- whether `dbt_project.yml` is valid;
- whether the profile exists;
- whether the three environment variables exist; and
- whether dbt can connect to Databricks.

## 12. Check what dbt will run

List the selected mart resources:

```bash
dbt ls --select path:models/mart
```

Parse the project without building tables:

```bash
dbt parse
```

Compile the SQL and Jinja into the SQL that will be submitted to Databricks:

```bash
dbt compile --select path:models/mart
```

Compiled files are written under `target/`. This is useful when you want to
show what SQL dbt generated. Do not manually edit files inside `target/`.

## 13. Build and test the Gold mart

Run the assessed dbt command:

```bash
dbt build --select path:models/mart
```

This single command:

1. Reads the Silver source declarations from `models/staging/sources.yml`.
2. Compiles the seven SQL/Jinja models.
3. Creates or replaces seven Delta tables in `03-mart`.
4. Runs the generic tests from `models/mart/schema.yml`.
5. Runs the selected custom SQL tests under `tests/dbt/`.
6. Returns a non-zero exit code if a model or test fails.

Do not manually add `CREATE OR REPLACE TABLE` to the model files. Each dbt
model contains a `SELECT` query describing the desired table. Because
`dbt_project.yml` configures the models as `table`, dbt generates the required
DDL for Databricks.

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

## 18. Continue the pipeline after dbt

When dbt finishes successfully, run this Databricks notebook:

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
5. Refresh the Metabase dashboards
```

Do not use `notebooks/00_run_full_pipeline.sql` as the assessed dbt route. It is
a Databricks-only demonstration runner that builds a SQL mirror of the mart.
The assessed implementation is the `dbt build` command followed by
`notebooks/06_run_after_dbt.sql`.

## 19. Generate dbt documentation

Generate the dbt documentation site:

```bash
dbt docs generate
```

Open it locally:

```bash
dbt docs serve
```

The generated site shows model descriptions, columns, tests, sources, and the
dependency graph.

Stop the local documentation server with `Control+C`.

## 20. Useful commands

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

The detailed run artifacts are saved under `target/`, including:

- `manifest.json`: project graph and metadata;
- `run_results.json`: model and test results; and
- `compiled/`: compiled SQL.

## 22. Troubleshooting

### `profiles.yml` was not found

Confirm the file exists:

```bash
ls ~/.dbt/profiles.yml
```

The profile name inside it must be `oulad_analytics`, matching:

```yaml
profile: oulad_analytics
```

in `dbt_project.yml`.

### An environment variable is missing

Set `DATABRICKS_HOST`, `DATABRICKS_HTTP_PATH`, and `DATABRICKS_TOKEN` again in
the same terminal where you run dbt.

### Connection refused or timed out

Confirm that:

- the SQL warehouse is running;
- the host and HTTP path came from the same warehouse connection details;
- the token has not expired; and
- your network can reach the Databricks workspace.

Then run `dbt debug` again.

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
5. Run `dbt parse`.
6. Run `dbt compile --select path:models/mart`.
7. Run `dbt build --select path:models/mart`.
8. Run `notebooks/06_run_after_dbt.sql`.
9. Review downstream Analytics and dashboard results.

Do not edit generated files under `target/` and do not commit real credentials.

## 24. Short presentation explanation

> We used SQL for the dbt transformations. Each SQL model contains a SELECT
> query describing one dimension or fact table. We used YAML to declare the
> Silver sources, document the models, and define tests. We used one macro to
> make dbt write directly to the required 03-mart schema. Python was only used
> to install and run dbt-databricks; we did not create Python dbt models. After
> Silver passed validation, we ran dbt build. dbt generated the required table
> DDL, created the five dimensions and two facts as Delta tables, and ran the
> generic and custom tests. We continued to Analytics only after every dbt
> model and test passed.

## 25. Completion checklist

- [ ] Bronze notebook succeeded.
- [ ] Silver notebook and Silver validation succeeded.
- [ ] Python virtual environment is active.
- [ ] `dbt-databricks` is installed.
- [ ] `profiles.yml` exists outside the repository.
- [ ] Host, HTTP path, and token are set securely.
- [ ] `dbt debug` passed.
- [ ] `dbt build --select path:models/mart` passed.
- [ ] Seven tables exist in `ftw-week-07.03-mart`.
- [ ] `notebooks/06_run_after_dbt.sql` succeeded.
- [ ] Analytics and data-quality views were refreshed.
- [ ] Metabase can read the latest outputs.
