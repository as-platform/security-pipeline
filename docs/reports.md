# Reports

🇷🇺 [Русская версия](reports_RUS.md)

The artifact `artifact-name` is kept for 7 days (an empty one is not an error) and holds three files: `gitleaks.json`, `kics/kics.json` and `checkov.json`.

The raw output of the scanners is never uploaded. A host step reads the counters from the raw report, writes a sanitized copy into the artifact and deletes the raw files. The copy is built from an allowlist: only the listed fields get in, everything else is dropped.

## `gitleaks.json`

The whole report of `gitleaks detect --redact` without `-v`, unchanged.

- Every `Secret` is `REDACTED`.
- The value inside `Match` is replaced by `REDACTED`.
- There is no field with the source line.

## `kics/kics.json`

Kept:

- summary counters: `kics_version`, `files_scanned`, `lines_scanned`, `files_parsed`, `lines_parsed`, `lines_ignored`, `files_failed_to_scan`, `queries_total`, `queries_failed_to_execute`, `queries_failed_to_compute_similarity_id`, `scan_id`, `total_counter`, `total_bom_resources`, `start`, `end`, `severity_counters`
- per query: `query_name`, `query_id`, `query_url`, `severity`, `platform`, `category`, `cloud_provider` (when present)
- per file: `file_name`, `similarity_id`, `line`, `resource_type`, `resource_name`, `issue_type`

Dropped: `search_key`, `search_value`, `expected_value`, `actual_value`, `remediation`, `remediation_type`, the query `description` and `paths`. kics copies source text into these fields: a whole `RUN` or `ENV` instruction, an image reference, an attribute value.

## `checkov.json`

The output is an object for one framework and an array for several.

Kept:

- per framework: `check_type`, `summary` (`passed`, `failed`, `skipped`, `parsing_errors`, `resource_count`, `checkov_version`) and `results.failed_checks`
- per failed check: `check_id`, `check_name`, `file_path`, `file_line_range`, `resource`, `guideline`, `severity`, `check_type`

Dropped: everything else.

- `code_block`
- `check_result`: its `results_configuration` and `entity` hold the whole configuration of a resource or a job, including `env:` and `run:` of a workflow and a Dockerfile instruction
- `evaluations`, `fixed_definition`, `entity_tags`, `breadcrumbs`
- passed and skipped checks, `repo_file_path`, `file_abs_path`, parsing errors

checkov OSS reports `severity` as `null`.

## Why secrets stay out

- Secrets scanning is off in kics (`--disable-secrets`) and in checkov (`--skip-framework secrets`): gitleaks is the only secrets scanner. The console output of kics goes to a file that is deleted and never uploaded.
- Secret values are removed by the allowlists, not by trusting the tools to hide them. A field that a tool fills with source text is dropped, and so are inline tokens that the tools quote from the scanned files or from rule metadata.

## What can still be seen

- The kept fields carry names taken from the scanned repository (`file_name`, `resource_name`, `resource`, `check_name`). A secret that someone puts into a file or resource name stays visible.
- The artifact of a public repository can be downloaded by any signed-in GitHub user while it is retained.
- Review the allowlists when a tool image is bumped: a field that a new version adds is not uploaded until the allowlist names it.

