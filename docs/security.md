# Security

🇷🇺 [Русская версия](security_RUS.md)

## Suppressions

Suppressions live in the scanned repository, so whoever can change the code can silence the scan. Review them like code and protect their files with `CODEOWNERS`:

- checkov: `#checkov:skip=<id>:<reason>` in the resource. A `.checkov.yaml` or `.checkov.yml` of the scanned repository is not loaded, see [Security notes](#security-notes)
- kics: the comments `# kics-scan ignore-line` and `# kics-scan ignore-block`. A `kics.config` of the scanned repository is not loaded, see [Security notes](#security-notes)
- gitleaks: `.gitleaks.toml`, `.gitleaksignore` and the `gitleaks:allow` comment

## Security notes

- The workflow does not run on `pull_request_target` and `workflow_run`: `common/prepare` fails closed with an `::error::` when `GITHUB_EVENT_NAME` is one of them, because the job holds Zot credentials in its first phase and scans a tree that can come from a fork. Call it from `push`, `pull_request` or `workflow_dispatch`.
- Fork pull requests get no secrets, so guard the job with `github.event.pull_request.head.repo.fork != true`, as the examples do.
- Zot credentials exist only in the fetch phase. `common/logout` runs `buildah logout --all`, deletes `~/.docker/config.json` and checks that both are gone before the first scanner starts. Third party actions run before the login or after a successful logout: the upload of the reports does not run when the logout failed.
- The workflow has `permissions: contents: read`, the checkout does not persist its token.
- A scanner can read only the workspace and write only its own `out/<tool>`; see [Scanner sandbox](architecture.md#scanner-sandbox).

### Scanning a repository you do not control

The `repository` input and `examples/scan_external_repo.yml` put an untrusted tree in front of the scanners. What a scanned repository can configure, found in the images and proven locally with `--network none` and the workspace mounted `:ro` (checkov 3.4.5, kics 2.2.1, gitleaks 8.30.1):

- checkov loads `.checkov.yaml` and `.checkov.yml` from the `--directory` argument, from the working directory and from the home directory. The config can set `external-checks-dir` and `external-checks-git`, which import Python from the scanned repository: a check of the repository that printed a marker ran on the old command line (the marker was in the output), and `external-checks-git` fetches code from a URL of the repository. Neutralised: `--config-file` does not help (default config files are still read), so the pipeline passes the directory as `--directory=<path>`, a spelling the discovery code (`get_default_config_paths`) does not look for, and starts checkov with the working directory in its own empty `out/checkov`. No config of the repository is loaded; the same fixture now prints no marker and reports the findings of the built-in checks. A `.checkov.yaml` or `.checkov.yml` in the workspace root is named in a `::warning::` as ignored.
- kics loads `kics.config` from the scanned path, and the config can replace the query set (`queries-path`, `libraries-path`) with Rego from the repository. kics refuses `http.send` in a query (`unsafe built-in function calls`), so this is a way to forge a result, not to run code: a repository could drop every query and report a clean scan. Neutralised: kics runs with `--config` pointing at an empty config in `out/kics`, so only the built-in queries run.
- gitleaks reads `.gitleaks.toml` (and `.gitleaksignore`) from the scanned tree. These files can add rules and allowlists, but the format has no way to run code or to redirect the report, and they are the supported suppression files (see Suppressions), so they are honoured.
- Not neutralised: inline suppressions (`#checkov:skip`, `# kics-scan ignore-line`, `gitleaks:allow`) and the gitleaks allowlists, so the result for such a repository shows what its owner left visible; the lack of network isolation (see [Limitations](../README.md#limitations)); the parsers of the scanners reading hostile files.

The proof depends on the pinned image versions. When a pin is bumped, repeat it: a fixture repository with a `.checkov.yaml` that sets `external-checks-dir` to a Python check printing a marker must give no marker, and a `kics.config` with its own `queries-path` must not change `queries_total`.

