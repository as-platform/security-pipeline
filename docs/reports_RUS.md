# Отчёты

🇬🇧 [English version](reports.md)

Artifact `artifact-name` хранится 7 дней (пустой набор не ошибка) и содержит три файла: `gitleaks.json`, `kics/kics.json` и `checkov.json`.

Сырой вывод сканеров не загружается никогда. Шаг на хосте читает счётчики из сырого отчёта, пишет в artifact очищенную копию и удаляет сырые файлы. Копия строится по allowlist: попадают только перечисленные поля, всё остальное отбрасывается.

## `gitleaks.json`

Весь отчёт `gitleaks detect --redact` без `-v`, без изменений.

- Каждый `Secret` равен `REDACTED`.
- Значение внутри `Match` заменено на `REDACTED`.
- Поля со строкой исходника нет.

## `kics/kics.json`

Остаётся:

- сводные счётчики: `kics_version`, `files_scanned`, `lines_scanned`, `files_parsed`, `lines_parsed`, `lines_ignored`, `files_failed_to_scan`, `queries_total`, `queries_failed_to_execute`, `queries_failed_to_compute_similarity_id`, `scan_id`, `total_counter`, `total_bom_resources`, `start`, `end`, `severity_counters`
- по каждому запросу: `query_name`, `query_id`, `query_url`, `severity`, `platform`, `category`, `cloud_provider` (если есть)
- по каждому файлу: `file_name`, `similarity_id`, `line`, `resource_type`, `resource_name`, `issue_type`

Убирается: `search_key`, `search_value`, `expected_value`, `actual_value`, `remediation`, `remediation_type`, `description` запроса и `paths`. kics копирует в эти поля текст исходников: инструкцию `RUN` или `ENV` целиком, ссылку на образ, значение атрибута.

## `checkov.json`

Вывод: объект для одного framework и массив для нескольких.

Остаётся:

- по каждому framework: `check_type`, `summary` (`passed`, `failed`, `skipped`, `parsing_errors`, `resource_count`, `checkov_version`) и `results.failed_checks`
- по каждой проваленной проверке: `check_id`, `check_name`, `file_path`, `file_line_range`, `resource`, `guideline`, `severity`, `check_type`

Убирается: всё остальное.

- `code_block`
- `check_result`: его `results_configuration` и `entity` содержат всю конфигурацию ресурса или job, включая `env:` и `run:` workflow и инструкцию Dockerfile
- `evaluations`, `fixed_definition`, `entity_tags`, `breadcrumbs`
- пройденные и пропущенные проверки, `repo_file_path`, `file_abs_path`, ошибки разбора

checkov OSS отдаёт `severity` как `null`.

## Почему секреты не попадают в отчёты

- Поиск секретов выключен в kics (`--disable-secrets`) и в checkov (`--skip-framework secrets`): секреты ищет только gitleaks. Вывод kics в консоль уходит в файл, который удаляется и не загружается.
- Значения секретов убираются allowlist'ами, а не доверием к тому, что инструменты их спрячут. Поле, которое инструмент заполняет текстом исходника, отбрасывается, как и inline-токены, которые инструменты цитируют из просканированных файлов или из метаданных правил.

## Что всё же видно

- Оставшиеся поля несут имена из проверяемого репозитория (`file_name`, `resource_name`, `resource`, `check_name`). Секрет, который кто-то положил в имя файла или ресурса, остаётся виден.
- Artifact публичного репозитория может скачать любой залогиненный на GitHub пользователь, пока он хранится.
- Allowlist нужно пересмотреть при обновлении образа инструмента: поле, которое добавила новая версия, не загружается, пока allowlist его не назовёт.

