# Gate semantics

🇷🇺 [Русская версия](gate_RUS.md)

- A tool is `ok` when the scanner exits with 0 and its report is valid JSON of the expected shape. A non-zero exit code, a missing or invalid report, or a scanner that did not run is `error`, never `0 findings`. A disabled tool is `skipped`.
- The expected shape of checkov is JSON where every element (an object for one framework, an array for several) has a `summary` with numeric `failed` and `parsing_errors`; a tree without IaC prints that summary as the whole object, which counts. `{}`, `[]`, an element without a summary and any other shape are `error`.
- The gate trips for a tool when `raise-failure` or `<tool>-raise-failure` is `true` and the tool is in `error` or has `gated > 0`.
- By default nothing trips: the job stays green, an `error` is shown as an `::error::` annotation, and the table in the job summary lists `tool | practice | status | findings | gated (threshold) | raise`.
- The job fails regardless of the switches when `prepare` (including the event check, see [Security notes](security.md#security-notes)), the pin check, `cosign verify`, the pull or the logout fails. An integrity or infrastructure failure is never soft.
- The verdict is `fail` also when the job status is not `success` at the gate, for example when a step before the gate failed or crashed: the gate prints `the job failed before the gate`.
- `gated` for kics counts the severities from `severity-threshold` up. checkov OSS reports no severity, so the threshold does not apply and every failed check is gated: it trips all or nothing. When introducing blocking mode, start with `kics-raise-failure: true`.
- The step `Quality gate` writes the summary and the outputs first, a later step `Enforce the gate` fails the job, so the outputs are set when the gate trips.

Every tool writes `$RUNNER_TEMP/security-pipeline/results/<tool>.json`:

```json
{"tool": "kics", "practice": "SAST/IAC", "status": "ok", "rc": 0, "findings": 290, "gated": 56, "incomplete": 0, "report": "kics/kics.json"}
```

`rc` is `-1` when the scanner did not run. A missing result file, or one without a numeric `incomplete`, counts as `error`. The file is written by the host step, outside every container mount.

## The `incomplete` counter

`incomplete` is a number: how many parts of the scan the tool could not finish. For checkov it is the sum of `summary.parsing_errors`, for kics `files_failed_to_scan + queries_failed_to_execute`, for gitleaks it is always `0`; a tool that is `error` or `skipped` reports `0`. A value above 0 never changes `status`, `gated` or the verdict: the tool prints `::warning::<tool> scan incomplete: N` and stays `ok`, so a clean result with `incomplete > 0` means "nothing found in what could be read". Measured on `bridgecrewio/terragoat@729f8da62c6a85ce4af5ad3d123de97776d954c4` and on this repository: `incomplete` is 0 for all three tools.
