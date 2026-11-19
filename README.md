# security-pipeline

🇷🇺 [Русская версия](README_RUS.md)

A reusable workflow that scans a repository for secrets and IaC misconfigurations with gitleaks, kics and checkov. The scanners run from signed images in the Zot registry of the [as-platform](https://github.com/as-platform) homelab platform, on the ephemeral runner `["self-hosted","homelab"]`, without docker: the images are verified with `cosign`, pulled with `buildah from` and started with `buildah run`.

The scanners always run in report mode. One bash and `jq` step reads their results and decides whether the job fails, so a scan never breaks a pipeline until the caller asks for it.

## What it runs

| Tool       | Practice       | Mode                                                          | `gated` is                              |
| ---------- | -------------- | ------------------------------------------------------------- | --------------------------------------- |
| `gitleaks` | `SAST/SECRETS` | `detect --no-git --redact`, working tree only                 | every finding                           |
| `kics`     | `SAST/IAC`     | `scan --ignore-on-exit results --disable-secrets`            | findings at or above `severity-threshold` |
| `checkov`  | `SAST/IAC`     | `--soft-fail --skip-framework secrets --skip-download`        | every failed check                      |

Secrets are the job of gitleaks only: kics and checkov have their secret scanning turned off, so a found secret is never printed by three tools.

## Quick start

```yaml
name: security
on:
  push: { branches: [main] }
  pull_request:

permissions:
  contents: read

jobs:
  security:
    if: github.event.pull_request.head.repo.fork != true
    uses: as-platform/security-pipeline/.github/workflows/security-pipeline.yml@main
    secrets:
      zot-user: ${{ secrets.ZOT_USER }}
      zot-password: ${{ secrets.ZOT_PASSWORD }}
```

The calling repository needs access to the runner (the runner group of the organization) and the secrets `ZOT_USER` and `ZOT_PASSWORD`. Pass the secrets explicitly, never with `secrets: inherit`. The workflow is referenced by `@main`, like every other template of the platform.

## Inputs

| Input                    | Default                    | Meaning                                                                                 |
| ------------------------ | -------------------------- | --------------------------------------------------------------------------------------- |
| `runner`                 | `["self-hosted","homelab"]` | JSON array of `runs-on` labels                                                          |
| `timeout-minutes`        | `30`                       | job timeout, keep it below the `activeDeadlineSeconds` of the runner                     |
| `repository`             | `""`                       | `owner/name` of a public repository to scan, empty means the caller repository          |
| `ref`                    | `""`                       | branch, tag or SHA, empty means the ref of the event (the default branch for another repository) |
| `artifact-name`          | `security-pipeline-reports` | name of the artifact with the reports                                                   |
| `severity-threshold`     | `HIGH`                     | one of `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO`; applies to kics only                |
| `image-gitleaks`         | pinned, see below          | image pin                                                                               |
| `image-kics`             | pinned, see below          | image pin                                                                               |
| `image-checkov`          | pinned, see below          | image pin                                                                               |
| `disable-gitleaks`       | `false`                    | skip the tool                                                                           |
| `disable-kics`           | `false`                    | skip the tool                                                                           |
| `disable-checkov`        | `false`                    | skip the tool                                                                           |
| `raise-failure`          | `false`                    | fail the job when any enabled tool errors or has gated findings                         |
| `gitleaks-raise-failure` | `false`                    | the same for gitleaks only                                                              |
| `kics-raise-failure`     | `false`                    | the same for kics only                                                                  |
| `checkov-raise-failure`  | `false`                    | the same for checkov only                                                               |

There are no group switches such as `disable-sast` or `disable-iac`, only per tool ones. The registry `zot.homelab` is fixed, there is no `registry` input.

## Secrets

| Secret         | Meaning                          |
| -------------- | -------------------------------- |
| `zot-user`     | Zot user that can read `golden/**` |
| `zot-password` | password of that user            |

Both are required. An empty value fails the `prepare` step with a message about fork pull requests: GitHub does not pass secrets to them, and the pipeline fails closed instead of skipping the scan silently.

## Outputs

| Output    | Meaning                                                                                                              |
| --------- | -------------------------------------------------------------------------------------------------------------------- |
| `results` | compact JSON object per tool: `{"gitleaks":{"status":"ok","findings":4,"gated":4,"incomplete":0,"raise":false},"kics":{...},...}` |
| `verdict` | `fail` when a tool tripped the gate or the job failed before the gate, otherwise `pass`                               |

`status` is `ok`, `error` or `skipped`. Read the outputs in a later job through `needs.<job>.outputs.results` with `fromJSON`, see `examples/lvl_5_blocking.yml`.

`incomplete` is a number: how many parts of the scan the tool could not finish. For checkov it is the sum of `summary.parsing_errors`, for kics `files_failed_to_scan + queries_failed_to_execute`, for gitleaks it is always `0`; a tool that is `error` or `skipped` reports `0`. A value above 0 never changes `status`, `gated` or the verdict: the tool prints `::warning::<tool> scan incomplete: N` and stays `ok`, so a clean result with `incomplete > 0` means "nothing found in what could be read". Measured on `bridgecrewio/terragoat@729f8da62c6a85ce4af5ad3d123de97776d954c4` and on this repository: `incomplete` is 0 for all three tools.

## Gate semantics

- A tool is `ok` when the scanner exits with 0 and its report is valid JSON of the expected shape. A non-zero exit code, a missing or invalid report, or a scanner that did not run is `error`, never `0 findings`. A disabled tool is `skipped`.
- The expected shape of checkov is JSON where every element (an object for one framework, an array for several) has a `summary` with numeric `failed` and `parsing_errors`; a tree without IaC prints that summary as the whole object, which counts. `{}`, `[]`, an element without a summary and any other shape are `error`.
- The gate trips for a tool when `raise-failure` or `<tool>-raise-failure` is `true` and the tool is in `error` or has `gated > 0`.
- By default nothing trips: the job stays green, an `error` is shown as an `::error::` annotation, and the table in the job summary lists `tool | practice | status | findings | gated (threshold) | raise`.
- The job fails regardless of the switches when `prepare` (including the event check, see Security notes), the pin check, `cosign verify`, the pull or the logout fails. An integrity or infrastructure failure is never soft.
- The verdict is `fail` also when the job status is not `success` at the gate, for example when a step before the gate failed or crashed: the gate prints `the job failed before the gate`.
- `gated` for kics counts the severities from `severity-threshold` up. checkov OSS reports no severity, so the threshold does not apply and every failed check is gated: it trips all or nothing. When introducing blocking mode, start with `kics-raise-failure: true`.
- The step `Quality gate` writes the summary and the outputs first, a later step `Enforce the gate` fails the job, so the outputs are set when the gate trips.

Every tool writes `$RUNNER_TEMP/security-pipeline/results/<tool>.json`:

```json
{"tool": "kics", "practice": "SAST/IAC", "status": "ok", "rc": 0, "findings": 290, "gated": 56, "incomplete": 0, "report": "kics/kics.json"}
```

`rc` is `-1` when the scanner did not run. A missing result file, or one without a numeric `incomplete`, counts as `error`. The file is written by the host step, outside every container mount.

## Reports

The artifact `artifact-name` (retention 7 days, empty is not an error) holds `gitleaks.json`, `kics/kics.json` and `checkov.json`. The raw output of the scanners is never uploaded: the host step reads the counters from the raw report, writes the uploaded file through an allowlist (checkov, kics) or as is (gitleaks), and deletes the raw files. This is what stays and what goes:

- `gitleaks.json`: the report of `gitleaks detect --redact` without `-v`, kept with its content unchanged. Every `Secret` is `REDACTED` and the value inside `Match` is replaced by `REDACTED`; the report has no field with the source line.
- `kics/kics.json`, kept: the summary counters (`kics_version`, `files_scanned`, `lines_scanned`, `files_parsed`, `lines_parsed`, `lines_ignored`, `files_failed_to_scan`, `queries_total`, `queries_failed_to_execute`, `queries_failed_to_compute_similarity_id`, `scan_id`, `total_counter`, `total_bom_resources`, `start`, `end`, `severity_counters`), per query `query_name`, `query_id`, `query_url`, `severity`, `platform`, `category`, `cloud_provider` (when present), and per file `file_name`, `similarity_id`, `line`, `resource_type`, `resource_name`, `issue_type`. Dropped: `search_key`, `search_value`, `expected_value`, `actual_value`, `remediation`, `remediation_type`, the query `description` and the scanned `paths`. kics copies source text into these fields (the whole `RUN` or `ENV` instruction, an image reference, an attribute value). Secrets scanning of kics is off (`--disable-secrets`) and its console output goes to a file that is deleted, not uploaded.
- `checkov.json`, kept: the shape of the checkov output (an object for one framework, an array for several) with, per framework, `check_type`, `summary` (`passed`, `failed`, `skipped`, `parsing_errors`, `resource_count`, `checkov_version`) and `results.failed_checks`, where each failed check keeps `check_id`, `check_name`, `file_path`, `file_line_range`, `resource`, `guideline`, `severity` and `check_type`. Dropped: everything else, in particular `code_block`, `check_result` (its `results_configuration` and `entity` hold the whole configuration of the resource or of the job, including the `env:` and `run:` of a workflow and the instruction of a Dockerfile), `evaluations`, `fixed_definition`, `entity_tags`, `breadcrumbs`, the passed and skipped checks, `repo_file_path`, `file_abs_path` and the list of parsing errors. The `secrets` framework of checkov is skipped (`--skip-framework secrets`). checkov OSS reports `severity` as `null`.

Secret values are removed by these allowlists, not by trusting the tools to hide them: a field that a tool fills with source text is dropped, and so are inline tokens that the tools quote from the scanned files or from rule metadata. The kept fields still carry names taken from the scanned repository (`file_name`, `resource_name`, `resource`, `check_name`), so a secret that someone puts into a file or resource name stays visible. The artifact of a public repository can be downloaded by any signed-in GitHub user while it is retained. Review the allowlists again when a tool image is bumped: a field that a new version adds is not uploaded until the allowlist names it.

## Examples

| File                           | What it shows                                                                          |
| ------------------------------ | -------------------------------------------------------------------------------------- |
| `examples/lvl_2_base.yml`      | soft mode on push, pull request and manual run, with the fork guard                    |
| `examples/lvl_5_blocking.yml`  | `raise-failure: true` and a deploy job that waits for the gate and reads `results`     |
| `examples/scan_external_repo.yml` | manual scan of another public repository through `repository` and `ref`             |

The examples live outside `.github/workflows/` so they do not run here. `examples/scan_external_repo.yml` scans a repository you do not control: read "Scanning a repository you do not control" in Security notes first.

## Layout

| Path                                  | Purpose                                                          |
| ------------------------------------- | ---------------------------------------------------------------- |
| `.github/workflows/security-pipeline.yml` | the reusable workflow, one job `security-pipeline`           |
| `common/prepare`                      | checks inputs, host tools and secrets, creates the work directory |
| `common/image`                        | pin check, `cosign verify` by digest, `buildah from`             |
| `common/image/cosign.pub`             | public key that verifies the images of `golden-images`           |
| `common/logout`                       | removes every Zot credential before the scans                    |
| `common/quality_gate`                 | reads the results, writes the summary and the outputs            |
| `tools/SAST/SECRETS/gitleaks`         | gitleaks scan                                                    |
| `tools/SAST/IAC/{kics,checkov}`       | kics and checkov scans                                           |
| `examples`                            | callers by maturity level                                        |

The steps of the job run in this order: `checkout`, `prepare`, `zot-login`, `image` for every enabled tool, `logout`, the scans one after another, `quality_gate`, `Enforce the gate`, upload of the reports, removal of the containers. The scans run with `if: !cancelled()` and need a successful `logout`, so one failed tool does not hide the others. The upload also needs a successful `logout` (`if: always() && steps.logout.outcome == 'success'`), so the third party upload action never runs while Zot credentials may still be on the runner.

The scanner runs as `buildah run --user 0:0` with exactly two mounts: the workspace read-only (`:ro`) and its own output directory `$RUNNER_TEMP/security-pipeline/out/<tool>` read-write. `$RUNNER_TEMP/security-pipeline/results` (the input of the gate) and `reports` (the uploaded files) are never mounted, so a scanner cannot rewrite its own result, the result of another tool or a report. The host step reads the raw output from `out/<tool>`, computes the counters, writes `results/<tool>.json` and the sanitized report into `reports/`, and deletes `out/<tool>`. checkov prints its JSON to stdout, which the host step redirects into a file that no container sees; for checkov `out/checkov` is only the working directory. `--user 0:0` is root of the rootless user namespace, that is the unprivileged runner user on the host. `$RUNNER_TEMP` as a whole (it holds `GITHUB_ENV` and `GITHUB_PATH`), `$HOME` and the auth file are not mounted. If a tool turns out to need a writable tree, only its mount becomes read-write and this paragraph lists it.

## Why one job

Every job is a separate runner pod started through KEDA, and the runner capacity is small. One job means one pod, one checkout and one `zot-login`. If you need parallelism, call the workflow twice with complementary `disable-*` inputs and different `artifact-name` values.

## Image pins

The defaults are pins of the form `zot.homelab/golden/<tool>:<YYYYMMDD-sha>@sha256:<digest>`, taken from the run of `golden-images` that built them. The value is checked with the regular expression `^zot\.homelab/golden/(gitleaks|trivy|kics|checkov):[0-9]{8}-[0-9a-f]{7,}@sha256:[0-9a-f]{64}$` before anything is pulled. Only `repo@digest` goes to `cosign` and `buildah`: skopeo 1.13.3 on the runner rejects `repo:tag@digest`. `stable` is never resolved at run time.

| Input            | Pin                                                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `image-gitleaks` | `zot.homelab/golden/gitleaks:20261118-0dcd8e4@sha256:4b2786189367f5bfdc65de189a1586d8060c3d7d7a315ae7fceb5c94d4a8237f`  |
| `image-kics`     | `zot.homelab/golden/kics:20261118-0dcd8e4@sha256:2ad5e4a1b087142af161df599d307da2ef2f18a88c0b3d501c260a8493bf68b1`      |
| `image-checkov`  | `zot.homelab/golden/checkov:20261118-0dcd8e4@sha256:75f39563ee79f00b6a583dcbf2c089020a7ee25c01af11ad606ac313a86d2933`   |

Renovate cannot reach Zot, so the pins are bumped by hand:

1. On a host that is logged in to Zot, read the digest of the new dated tag: `skopeo inspect --format '{{.Digest}}' docker://zot.homelab/golden/<tool>:<YYYYMMDD-sha>`.
2. Verify it: `cosign verify --key common/image/cosign.pub --insecure-ignore-tlog=true zot.homelab/golden/<tool>@<digest>`.
3. Replace the default of `image-<tool>` in `.github/workflows/security-pipeline.yml` and open a pull request.

The pin of a removed tag stops working, so a future retention policy for `golden/**` must keep the pinned tags. The key in `common/image/cosign.pub` is rotated by replacing that single file.

## Boundary with ci-templates

[`ci-templates/security.yml`](https://github.com/as-platform/ci-templates) scans the git history with gitleaks and runs `trivy fs`, hadolint and semgrep with the binaries of the runner host. This workflow runs signed images, scans the working tree only and gates on its own results. Call one of the two in a repository, not both. `trivy config` is not part of this workflow.

## Limitations

- gitleaks runs with `--no-git` over a checkout of depth 1: a secret that is removed from the tree but still in the history is not found. Rotate such a secret, and scan the history with `ci-templates/security.yml` where it matters.
- There is no network isolation. The scanners work offline only because of their flags (`--skip-download` for checkov, queries inside the kics image); the runner does not block their traffic.
- Only public repositories can be scanned through `repository`; there is no checkout token.
- `concurrency` is not set, the caller decides. The runner pool is small (`maxReplicaCount 3`), so a pipeline on every push takes one pod per run.
- A sub directory of a monorepo cannot be scanned on its own, the whole checkout is scanned.
- The Zot user in use today can also write to Zot. Connect only repositories you trust with it until a read-only user for `golden/**` exists.

## Suppressions

Suppressions live in the scanned repository, so whoever can change the code can silence the scan. Review them like code and protect their files with `CODEOWNERS`:

- checkov: `#checkov:skip=<id>:<reason>` in the resource. A `.checkov.yaml` or `.checkov.yml` of the scanned repository is not loaded, see Security notes
- kics: the comments `# kics-scan ignore-line` and `# kics-scan ignore-block`. A `kics.config` of the scanned repository is not loaded, see Security notes
- gitleaks: `.gitleaks.toml`, `.gitleaksignore` and the `gitleaks:allow` comment

## Security notes

- The workflow does not run on `pull_request_target` and `workflow_run`: `common/prepare` fails closed with an `::error::` when `GITHUB_EVENT_NAME` is one of them, because the job holds Zot credentials in its first phase and scans a tree that can come from a fork. Call it from `push`, `pull_request` or `workflow_dispatch`.
- Fork pull requests get no secrets, so guard the job with `github.event.pull_request.head.repo.fork != true`, as the examples do.
- Zot credentials exist only in the fetch phase. `common/logout` runs `buildah logout --all`, deletes `~/.docker/config.json` and checks that both are gone before the first scanner starts. Third party actions run before the login or after a successful logout: the upload of the reports does not run when the logout failed.
- The workflow has `permissions: contents: read`, the checkout does not persist its token.
- A scanner can read only the workspace and write only its own `out/<tool>`; see the mounts under Layout.

### Scanning a repository you do not control

The `repository` input and `examples/scan_external_repo.yml` put an untrusted tree in front of the scanners. What a scanned repository can configure, found in the images and proven locally with `--network none` and the workspace mounted `:ro` (checkov 3.4.5, kics 2.2.1, gitleaks 8.30.1):

- checkov loads `.checkov.yaml` and `.checkov.yml` from the `--directory` argument, from the working directory and from the home directory. The config can set `external-checks-dir` and `external-checks-git`, which import Python from the scanned repository: a check of the repository that printed a marker ran on the old command line (the marker was in the output), and `external-checks-git` fetches code from a URL of the repository. Neutralised: `--config-file` does not help (default config files are still read), so the pipeline passes the directory as `--directory=<path>`, a spelling the discovery code (`get_default_config_paths`) does not look for, and starts checkov with the working directory in its own empty `out/checkov`. No config of the repository is loaded; the same fixture now prints no marker and reports the findings of the built-in checks. A `.checkov.yaml` or `.checkov.yml` in the workspace root is named in a `::warning::` as ignored.
- kics loads `kics.config` from the scanned path, and the config can replace the query set (`queries-path`, `libraries-path`) with Rego from the repository. kics refuses `http.send` in a query (`unsafe built-in function calls`), so this is a way to forge a result, not to run code: a repository could drop every query and report a clean scan. Neutralised: kics runs with `--config` pointing at an empty config in `out/kics`, so only the built-in queries run.
- gitleaks reads `.gitleaks.toml` (and `.gitleaksignore`) from the scanned tree. These files can add rules and allowlists, but the format has no way to run code or to redirect the report, and they are the supported suppression files (see Suppressions), so they are honoured.
- Not neutralised: inline suppressions (`#checkov:skip`, `# kics-scan ignore-line`, `gitleaks:allow`) and the gitleaks allowlists, so the result for such a repository shows what its owner left visible; the lack of network isolation (see Limitations); the parsers of the scanners reading hostile files.

The proof depends on the pinned image versions. When a pin is bumped, repeat it: a fixture repository with a `.checkov.yaml` that sets `external-checks-dir` to a Python check printing a marker must give no marker, and a `kics.config` with its own `queries-path` must not change `queries_total`.

## How the repository is checked

The repository has no CI of its own. Locally: `actionlint -config-file .github/actionlint.yaml` on the workflow and the examples, `shellcheck -s bash` on every `run:` body of every action, grep gates for forbidden constructs (`${{` inside `run:`, `ubuntu-latest`, `docker`, `container:`, Python, `secrets: inherit`, unpinned third party actions), and `npx --yes --package renovate renovate-config-validator --strict renovate.json`. The behaviour is checked end to end by a workflow in `runner-smoke` that calls `security-pipeline@main` on the runner against a pinned copy of `bridgecrewio/terragoat` and against `runner-smoke` itself.

## Related repositories

- [`ci-templates`](https://github.com/as-platform/ci-templates) — `actions/zot-login` and the workflows this pipeline sits next to
- [`golden-images`](https://github.com/as-platform/golden-images) — builds and signs the scanner images
- [`runner-smoke`](https://github.com/as-platform/runner-smoke) — runs the end to end checks of the pipeline on the runner
- [`k3s`](https://github.com/as-platform/k3s) — the GitOps repository with the runner and Zot

## License

MIT — see [`LICENSE`](LICENSE).
