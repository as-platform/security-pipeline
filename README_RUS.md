# security-pipeline

🇬🇧 [English version](README.md)

Reusable workflow, который проверяет репозиторий на секреты и ошибки конфигурации IaC через gitleaks, kics и checkov. Сканеры запускаются из подписанных образов в реестре Zot домашней платформы [as-platform](https://github.com/as-platform) на эфемерном раннере `["self-hosted","homelab"]`, без docker: образы проверяются `cosign`, скачиваются `buildah from` и запускаются `buildah run`.

Сканеры всегда работают в режиме отчёта. Один шаг на bash и `jq` читает их результаты и решает, падает ли job, поэтому скан не ломает pipeline, пока вызывающий сам этого не попросит.

## Что запускается

| Инструмент | Практика       | Режим                                                         | `gated` это                                  |
| ---------- | -------------- | ------------------------------------------------------------- | -------------------------------------------- |
| `gitleaks` | `SAST/SECRETS` | `detect --no-git --redact`, только рабочее дерево             | каждая находка                               |
| `kics`     | `SAST/IAC`     | `scan --ignore-on-exit results --disable-secrets`             | находки не ниже `severity-threshold`         |
| `checkov`  | `SAST/IAC`     | `--soft-fail --skip-framework secrets --skip-download`        | каждая проваленная проверка                  |

Секреты ищет только gitleaks: в kics и checkov поиск секретов выключен, поэтому найденный секрет не печатают три инструмента.

## Быстрый старт

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

Вызывающему репозиторию нужен доступ к раннеру (runner group организации) и секреты `ZOT_USER` и `ZOT_PASSWORD`. Секреты передаются явно, `secrets: inherit` не используется. Workflow подключается по `@main`, как и все остальные шаблоны платформы.

## Входы

| Вход                     | По умолчанию                | Смысл                                                                                      |
| ------------------------ | --------------------------- | ------------------------------------------------------------------------------------------ |
| `runner`                 | `["self-hosted","homelab"]` | JSON-массив меток `runs-on`                                                                |
| `timeout-minutes`        | `30`                        | таймаут job, держи его меньше `activeDeadlineSeconds` раннера                              |
| `repository`             | `""`                        | `owner/name` публичного репозитория для скана, пусто значит репозиторий вызывающего        |
| `ref`                    | `""`                        | ветка, тег или SHA, пусто значит ref события (ветка по умолчанию для другого репозитория)  |
| `artifact-name`          | `security-pipeline-reports` | имя artifact с отчётами                                                                    |
| `severity-threshold`     | `HIGH`                      | одно из `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO`; действует только на kics              |
| `image-gitleaks`         | пин, см. ниже               | пин образа                                                                                 |
| `image-kics`             | пин, см. ниже               | пин образа                                                                                 |
| `image-checkov`          | пин, см. ниже               | пин образа                                                                                 |
| `disable-gitleaks`       | `false`                     | пропустить инструмент                                                                      |
| `disable-kics`           | `false`                     | пропустить инструмент                                                                      |
| `disable-checkov`        | `false`                     | пропустить инструмент                                                                      |
| `raise-failure`          | `false`                     | уронить job, если любой включённый инструмент в ошибке или имеет gated-находки             |
| `gitleaks-raise-failure` | `false`                     | то же только для gitleaks                                                                  |
| `kics-raise-failure`     | `false`                     | то же только для kics                                                                      |
| `checkov-raise-failure`  | `false`                     | то же только для checkov                                                                   |

Групповых переключателей вроде `disable-sast` или `disable-iac` нет, только поинструментные. Реестр `zot.homelab` зашит, входа `registry` нет.

## Секреты

| Секрет         | Смысл                                  |
| -------------- | -------------------------------------- |
| `zot-user`     | пользователь Zot с чтением `golden/**` |
| `zot-password` | пароль этого пользователя              |

Оба обязательны. Пустое значение валит шаг `prepare` сообщением про fork pull request: GitHub не передаёт им секреты, и pipeline падает, а не пропускает скан молча (fail-closed).

## Выходы

| Выход     | Смысл                                                                                                                  |
| --------- | ---------------------------------------------------------------------------------------------------------------------- |
| `results` | компактный JSON-объект по инструментам: `{"gitleaks":{"status":"ok","findings":4,"gated":4,"incomplete":0,"raise":false},"kics":{...},...}` |
| `verdict` | `fail`, если инструмент сработал на гейте или job упал до гейта, иначе `pass`                                          |

`status` — `ok`, `error` или `skipped`. Выходы читаются в следующем job через `needs.<job>.outputs.results` и `fromJSON`, см. `examples/lvl_5_blocking.yml`.

`incomplete` — число: сколько частей скана инструмент не смог завершить. У checkov это сумма `summary.parsing_errors`, у kics `files_failed_to_scan + queries_failed_to_execute`, у gitleaks всегда `0`; инструмент в `error` или `skipped` отдаёт `0`. Значение больше 0 никогда не меняет `status`, `gated` и вердикт: инструмент печатает `::warning::<tool> scan incomplete: N` и остаётся `ok`, поэтому чистый результат при `incomplete > 0` означает «в прочитанном ничего не найдено». Замерено на `bridgecrewio/terragoat@729f8da62c6a85ce4af5ad3d123de97776d954c4` и на этом репозитории: `incomplete` равен 0 у всех трёх инструментов.

## Семантика гейта

- Инструмент `ok`, когда сканер завершился с кодом 0 и его отчёт — валидный JSON ожидаемой формы. Ненулевой код, отсутствующий или невалидный отчёт, либо сканер, который не запускался, — это `error`, а не «0 находок». Выключенный инструмент — `skipped`.
- Ожидаемая форма checkov — JSON, где у каждого элемента (объект для одного framework, массив для нескольких) есть `summary` с числовыми `failed` и `parsing_errors`; дерево без IaC печатает этот summary как весь объект, и это допустимо. `{}`, `[]`, элемент без summary и любая другая форма — `error`.
- Гейт срабатывает на инструменте, когда `raise-failure` или `<tool>-raise-failure` равен `true`, а инструмент в `error` или имеет `gated > 0`.
- По умолчанию не срабатывает ничего: job зелёный, `error` виден аннотацией `::error::`, а таблица в job summary показывает `tool | practice | status | findings | gated (threshold) | raise`.
- Job падает независимо от переключателей, если упал `prepare` (включая проверку события, см. «Заметки по безопасности»), проверка пина, `cosign verify`, pull или logout. Сбой целостности или инфраструктуры никогда не бывает мягким.
- Вердикт `fail` и тогда, когда статус job на гейте не `success`, например упал или аварийно завершился шаг до гейта: гейт печатает `the job failed before the gate`.
- `gated` у kics считает severity от `severity-threshold` и выше. checkov OSS не отдаёт severity, поэтому порог на него не действует и gated каждая проваленная проверка: он срабатывает по принципу «всё или ничего». При включении блокировки начни с `kics-raise-failure: true`.
- Шаг `Quality gate` сначала пишет summary и выходы, а падает job на следующем шаге `Enforce the gate`, поэтому выходы заданы и тогда, когда гейт сработал.

Каждый инструмент пишет `$RUNNER_TEMP/security-pipeline/results/<tool>.json`:

```json
{"tool": "kics", "practice": "SAST/IAC", "status": "ok", "rc": 0, "findings": 290, "gated": 56, "incomplete": 0, "report": "kics/kics.json"}
```

`rc` равен `-1`, если сканер не запускался. Отсутствующий файл результата или файл без числового `incomplete` считается `error`. Файл пишет шаг на хосте, вне монтирований любого контейнера.

## Отчёты

Artifact `artifact-name` хранится 7 дней (пустой набор не ошибка) и содержит три файла: `gitleaks.json`, `kics/kics.json` и `checkov.json`.

Сырой вывод сканеров не загружается никогда. Шаг на хосте читает счётчики из сырого отчёта, пишет в artifact очищенную копию и удаляет сырые файлы. Копия строится по allowlist: попадают только перечисленные поля, всё остальное отбрасывается.

### `gitleaks.json`

Весь отчёт `gitleaks detect --redact` без `-v`, без изменений.

- Каждый `Secret` равен `REDACTED`.
- Значение внутри `Match` заменено на `REDACTED`.
- Поля со строкой исходника нет.

### `kics/kics.json`

Остаётся:

- сводные счётчики: `kics_version`, `files_scanned`, `lines_scanned`, `files_parsed`, `lines_parsed`, `lines_ignored`, `files_failed_to_scan`, `queries_total`, `queries_failed_to_execute`, `queries_failed_to_compute_similarity_id`, `scan_id`, `total_counter`, `total_bom_resources`, `start`, `end`, `severity_counters`
- по каждому запросу: `query_name`, `query_id`, `query_url`, `severity`, `platform`, `category`, `cloud_provider` (если есть)
- по каждому файлу: `file_name`, `similarity_id`, `line`, `resource_type`, `resource_name`, `issue_type`

Убирается: `search_key`, `search_value`, `expected_value`, `actual_value`, `remediation`, `remediation_type`, `description` запроса и `paths`. kics копирует в эти поля текст исходников: инструкцию `RUN` или `ENV` целиком, ссылку на образ, значение атрибута.

### `checkov.json`

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

### Почему секреты не попадают в отчёты

- Поиск секретов выключен в kics (`--disable-secrets`) и в checkov (`--skip-framework secrets`): секреты ищет только gitleaks. Вывод kics в консоль уходит в файл, который удаляется и не загружается.
- Значения секретов убираются allowlist'ами, а не доверием к тому, что инструменты их спрячут. Поле, которое инструмент заполняет текстом исходника, отбрасывается, как и inline-токены, которые инструменты цитируют из просканированных файлов или из метаданных правил.

### Что всё же видно

- Оставшиеся поля несут имена из проверяемого репозитория (`file_name`, `resource_name`, `resource`, `check_name`). Секрет, который кто-то положил в имя файла или ресурса, остаётся виден.
- Artifact публичного репозитория может скачать любой залогиненный на GitHub пользователь, пока он хранится.
- Allowlist нужно пересмотреть при обновлении образа инструмента: поле, которое добавила новая версия, не загружается, пока allowlist его не назовёт.

## Примеры

| Файл                              | Что показывает                                                                         |
| --------------------------------- | -------------------------------------------------------------------------------------- |
| `examples/lvl_2_base.yml`         | мягкий режим на push, pull request и ручной запуск, с защитой от форков                |
| `examples/lvl_5_blocking.yml`     | `raise-failure: true` и deploy-job, который ждёт гейт и читает `results`               |
| `examples/scan_external_repo.yml` | ручной скан другого публичного репозитория через `repository` и `ref`                  |

Примеры лежат вне `.github/workflows/`, чтобы не запускаться здесь. `examples/scan_external_repo.yml` сканирует репозиторий, который ты не контролируешь: сначала прочитай «Скан репозитория, который ты не контролируешь» в заметках по безопасности.

## Структура

| Путь                                      | Назначение                                                        |
| ----------------------------------------- | ----------------------------------------------------------------- |
| `.github/workflows/security-pipeline.yml` | reusable workflow, один job `security-pipeline`                   |
| `common/prepare`                          | проверяет входы, инструменты хоста и секреты, создаёт рабочий каталог |
| `common/image`                            | проверка пина, `cosign verify` по digest, `buildah from`          |
| `common/image/cosign.pub`                 | публичный ключ, которым проверяются образы `golden-images`        |
| `common/logout`                           | убирает все креды Zot до сканов                                   |
| `common/quality_gate`                     | читает результаты, пишет summary и выходы                         |
| `tools/SAST/SECRETS/gitleaks`             | скан gitleaks                                                     |
| `tools/SAST/IAC/{kics,checkov}`           | сканы kics и checkov                                              |
| `examples`                                | вызывающие workflow по уровням зрелости                           |

### Порядок шагов

1. `checkout`
2. `prepare`
3. `zot-login`
4. `image` для каждого включённого инструмента
5. `logout`
6. сканы по очереди
7. `quality_gate`
8. `Enforce the gate`
9. загрузка отчётов
10. удаление контейнеров

- Сканы идут с `if: !cancelled()` и требуют успешного `logout`, поэтому упавший инструмент не скрывает остальные.
- Загрузка тоже требует успешного `logout` (`if: always() && steps.logout.outcome == 'success'`), поэтому сторонний action загрузки никогда не работает, пока на раннере могут оставаться креды Zot.

### Песочница сканера

Каждый сканер запускается как `buildah run --user 0:0` ровно с двумя монтированиями:

| Что монтируется                                      | Режим       |
| ---------------------------------------------------- | ----------- |
| workspace (checkout)                                 | на чтение   |
| `$RUNNER_TEMP/security-pipeline/out/<tool>`          | на запись   |

Не монтируются никогда: `$RUNNER_TEMP/security-pipeline/results` (вход гейта), `$RUNNER_TEMP/security-pipeline/reports` (загружаемые файлы), `$RUNNER_TEMP` целиком (там `GITHUB_ENV` и `GITHUB_PATH`), `$HOME` и authfile. Поэтому сканер не может переписать ни свой результат, ни результат другого инструмента, ни отчёт.

После скана шаг на хосте читает сырой вывод из `out/<tool>`, считает счётчики, пишет `results/<tool>.json` и очищенный отчёт в `reports/` и удаляет `out/<tool>`.

- checkov печатает JSON в stdout, а шаг на хосте перенаправляет его в файл, невидимый ни одному контейнеру. Для checkov `out/checkov` служит только рабочим каталогом.
- `--user 0:0` — это root rootless user namespace, то есть непривилегированный пользователь раннера на хосте.
- Если окажется, что инструменту нужно писать в дерево, на запись становится доступным только его mount, и этот раздел перечислит такие случаи.

## Почему один job

Каждый job — отдельный под раннера через KEDA, а ёмкость раннера мала. Один job означает один под, один checkout и один `zot-login`. Если нужен параллелизм, вызови workflow дважды с дополняющими друг друга `disable-*` и разными `artifact-name`.

## Пины образов

По умолчанию заданы пины вида `zot.homelab/golden/<tool>:<YYYYMMDD-sha>@sha256:<digest>` из прогона `golden-images`, который их собрал. Значение проверяется регулярным выражением `^zot\.homelab/golden/(gitleaks|trivy|kics|checkov):[0-9]{8}-[0-9a-f]{7,}@sha256:[0-9a-f]{64}$` до любого pull. В `cosign` и `buildah` уходит только `repo@digest`: skopeo 1.13.3 на раннере отвергает `repo:tag@digest`. `stable` во время запуска не резолвится.

| Вход             | Пин                                                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `image-gitleaks` | `zot.homelab/golden/gitleaks:20261118-0dcd8e4@sha256:4b2786189367f5bfdc65de189a1586d8060c3d7d7a315ae7fceb5c94d4a8237f`  |
| `image-kics`     | `zot.homelab/golden/kics:20261118-0dcd8e4@sha256:2ad5e4a1b087142af161df599d307da2ef2f18a88c0b3d501c260a8493bf68b1`      |
| `image-checkov`  | `zot.homelab/golden/checkov:20261118-0dcd8e4@sha256:75f39563ee79f00b6a583dcbf2c089020a7ee25c01af11ad606ac313a86d2933`   |

Renovate не видит Zot, поэтому пины обновляются вручную:

1. На хосте с логином в Zot прочитай digest нового датированного тега: `skopeo inspect --format '{{.Digest}}' docker://zot.homelab/golden/<tool>:<YYYYMMDD-sha>`.
2. Проверь его: `cosign verify --key common/image/cosign.pub --insecure-ignore-tlog=true zot.homelab/golden/<tool>@<digest>`.
3. Замени значение по умолчанию `image-<tool>` в `.github/workflows/security-pipeline.yml` и открой pull request.

Пин удалённого тега перестаёт работать, поэтому будущий ретеншен `golden/**` должен сохранять закреплённые теги. Ключ `common/image/cosign.pub` ротируется заменой этого единственного файла.

## Граница с ci-templates

[`ci-templates/security.yml`](https://github.com/as-platform/ci-templates) проверяет историю git через gitleaks и запускает `trivy fs`, hadolint и semgrep бинарями хоста раннера. Этот workflow запускает подписанные образы, проверяет только рабочее дерево и принимает решение по своим результатам. В репозитории вызывай один из двух, не оба. `trivy config` в этот workflow не входит.

## Ограничения

- gitleaks работает с `--no-git` по checkout глубиной 1: секрет, удалённый из дерева, но оставшийся в истории, не находится. Такой секрет нужно ротировать, а историю, где это важно, проверять через `ci-templates/security.yml`.
- Сетевой изоляции нет. Сканеры работают офлайн только благодаря флагам (`--skip-download` у checkov, запросы kics внутри образа); раннер их трафик не блокирует.
- Через `repository` можно сканировать только публичные репозитории: токена для checkout нет.
- `concurrency` не задан, его решает вызывающий. Пул раннеров мал (`maxReplicaCount 3`), поэтому pipeline на каждый push занимает один под на запуск.
- Подкаталог монорепозитория отдельно сканировать нельзя, проверяется весь checkout.
- Пользователь Zot, который используется сейчас, умеет и писать в Zot. Подключай только репозитории, которым ты это доверяешь, пока нет пользователя только на чтение `golden/**`.

## Подавления

Подавления лежат в проверяемом репозитории, поэтому тот, кто может менять код, может заглушить скан. Проверяй их как код и закрой их файлы через `CODEOWNERS`:

- checkov: `#checkov:skip=<id>:<причина>` у ресурса. `.checkov.yaml` или `.checkov.yml` проверяемого репозитория не загружается, см. «Заметки по безопасности»
- kics: комментарии `# kics-scan ignore-line` и `# kics-scan ignore-block`. `kics.config` проверяемого репозитория не загружается, см. «Заметки по безопасности»
- gitleaks: `.gitleaks.toml`, `.gitleaksignore` и комментарий `gitleaks:allow`

## Заметки по безопасности

- Workflow не работает на `pull_request_target` и `workflow_run`: `common/prepare` падает fail-closed с `::error::`, когда `GITHUB_EVENT_NAME` равен одному из них, потому что у job в первой фазе есть креды Zot, а сканируется дерево, которое может прийти из форка. Вызывай его из `push`, `pull_request` или `workflow_dispatch`.
- Fork pull request не получают секреты, поэтому защити job условием `github.event.pull_request.head.repo.fork != true`, как в примерах.
- Креды Zot живут только в фазе fetch. `common/logout` выполняет `buildah logout --all`, удаляет `~/.docker/config.json` и проверяет, что оба исчезли, до запуска первого сканера. Сторонние actions работают до логина или после успешного logout: загрузка отчётов не запускается, если logout упал.
- У workflow `permissions: contents: read`, checkout не сохраняет свой токен.
- Сканер читает только workspace и пишет только в свой `out/<tool>`; монтирования см. в разделе «Структура».

### Скан репозитория, который ты не контролируешь

Вход `repository` и `examples/scan_external_repo.yml` ставят перед сканерами недоверенное дерево. Что проверяемый репозиторий может настроить, найдено в образах и доказано локально с `--network none` и workspace `:ro` (checkov 3.4.5, kics 2.2.1, gitleaks 8.30.1):

- checkov загружает `.checkov.yaml` и `.checkov.yml` из аргумента `--directory`, из рабочего каталога и из домашнего каталога. Конфиг может задать `external-checks-dir` и `external-checks-git`, которые импортируют Python из проверяемого репозитория: проверка репозитория, печатавшая маркер, выполнилась на старой командной строке (маркер был в выводе), а `external-checks-git` забирает код по URL из репозитория. Обезврежено: `--config-file` не помогает (файлы конфигурации по умолчанию всё равно читаются), поэтому pipeline передаёт каталог как `--directory=<path>`, а такую запись код поиска (`get_default_config_paths`) не ищет, и запускает checkov с рабочим каталогом в его собственном пустом `out/checkov`. Конфиг репозитория не загружается; тот же fixture теперь не печатает маркер и отдаёт находки встроенных проверок. `.checkov.yaml` или `.checkov.yml` в корне workspace называется в `::warning::` как игнорируемый.
- kics загружает `kics.config` из сканируемого пути, и конфиг может заменить набор запросов (`queries-path`, `libraries-path`) на Rego из репозитория. kics запрещает `http.send` в запросе (`unsafe built-in function calls`), поэтому это способ подделать результат, а не выполнить код: репозиторий мог бы убрать все запросы и получить чистый скан. Обезврежено: kics запускается с `--config`, указывающим на пустой конфиг в `out/kics`, поэтому работают только встроенные запросы.
- gitleaks читает `.gitleaks.toml` (и `.gitleaksignore`) из сканируемого дерева. Эти файлы могут добавить правила и allowlist, но формат не умеет ни запускать код, ни перенаправлять отчёт, а сами файлы — поддерживаемые файлы подавлений (см. «Подавления»), поэтому они учитываются.
- Не обезврежено: inline-подавления (`#checkov:skip`, `# kics-scan ignore-line`, `gitleaks:allow`) и allowlist gitleaks, поэтому результат для такого репозитория показывает то, что его владелец оставил на виду; отсутствие сетевой изоляции (см. «Ограничения»); парсеры сканеров, читающие враждебные файлы.

Доказательство зависит от версий закреплённых образов. При обновлении пина повтори его: fixture-репозиторий с `.checkov.yaml`, где `external-checks-dir` указывает на Python-проверку, печатающую маркер, не должен давать маркера, а `kics.config` со своим `queries-path` не должен менять `queries_total`.

## Как проверяется репозиторий

Собственного CI у репозитория нет. Локально: `actionlint -config-file .github/actionlint.yaml` по workflow и примерам, `shellcheck -s bash` по каждому `run:` каждого action, grep-ворота на запрещённые конструкции (`${{` внутри `run:`, `ubuntu-latest`, `docker`, `container:`, Python, `secrets: inherit`, внешние actions без SHA) и `npx --yes --package renovate renovate-config-validator --strict renovate.json`. Поведение проверяет сквозной workflow в `runner-smoke`: он вызывает `security-pipeline@main` на раннере против закреплённой копии `bridgecrewio/terragoat` и против самого `runner-smoke`.

## Связанные репозитории

- [`ci-templates`](https://github.com/as-platform/ci-templates) — `actions/zot-login` и workflow, рядом с которыми стоит этот pipeline
- [`golden-images`](https://github.com/as-platform/golden-images) — собирает и подписывает образы сканеров
- [`runner-smoke`](https://github.com/as-platform/runner-smoke) — гоняет сквозные проверки pipeline на раннере
- [`k3s`](https://github.com/as-platform/k3s) — GitOps-репозиторий с раннером и Zot

## Лицензия

MIT — см. [`LICENSE`](LICENSE).
