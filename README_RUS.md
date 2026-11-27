# security-pipeline

🇬🇧 [English version](README.md)

Reusable workflow GitHub Actions, который проверяет репозиторий на секреты, ошибки конфигурации IaC и уязвимые зависимости через gitleaks, kics, checkov и trivy.

Сканеры запускаются из подписанных образов в Zot, на раннере `["self-hosted","homelab"]`, без docker. По умолчанию они только сообщают о находках: job остаётся зелёным, пока ты сам не попросишь его падать.

## Что запускается

| Инструмент | Что ищет                                          | Что считает гейт                                      |
| ---------- | ------------------------------------------------- | ----------------------------------------------------- |
| `gitleaks` | секреты в рабочем дереве (не в истории)           | каждую находку                                        |
| `kics`     | ошибки конфигурации IaC                           | находки не ниже `severity-threshold`                  |
| `checkov`  | ошибки конфигурации IaC                           | каждую проваленную проверку                           |
| `trivy fs` | уязвимые зависимости в lockfile (SCA)             | находки с исправлением не ниже `severity-threshold`   |

Секреты ищет только gitleaks: у kics и checkov поиск секретов выключен. `trivy fs` выключен по умолчанию: в репозитории с lockfile поставь `disable-trivy-fs: false`.

## Быстрый старт

```yaml
name: security
on:
  push: { branches: [main] }
  pull_request:
  workflow_dispatch:

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

Репозиторию нужен доступ к runner group и секреты `ZOT_USER` и `ZOT_PASSWORD`. Передавай их явно, не через `secrets: inherit`. Чтобы запускать только вручную, оставь `workflow_dispatch` единственным триггером.

## Входы

| Вход                                                     | По умолчанию                 | Смысл                                                            |
| -------------------------------------------------------- | ---------------------------- | ---------------------------------------------------------------- |
| `runner`                                                 | `["self-hosted","homelab"]`  | JSON-массив меток `runs-on`                                      |
| `timeout-minutes`                                        | `30`                         | таймаут job                                                      |
| `repository`, `ref`                                      | вызывающий репозиторий       | проверить другой публичный репозиторий                           |
| `artifact-name`                                          | `security-pipeline-reports`  | имя артефакта с отчётами                                         |
| `severity-threshold`                                     | `HIGH`                       | `CRITICAL`, `HIGH`, `MEDIUM`, `LOW` или `INFO`; kics и trivy fs  |
| `disable-gitleaks`, `disable-kics`, `disable-checkov`    | `false`                      | пропустить инструмент                                            |
| `disable-trivy-fs`                                       | `true`                       | `false` включает trivy fs                                        |
| `raise-failure`                                          | `false`                      | уронить job, если инструмент в `error` или есть находки гейта    |
| `gitleaks-raise-failure`, `kics-raise-failure`, `checkov-raise-failure`, `trivy-fs-raise-failure` | `false` | то же для одного инструмента                               |
| `image-gitleaks`, `image-kics`, `image-checkov`, `image-trivy` | пин                    | пин образа, см. [Пины образов](#пины-образов)                    |

Секреты `zot-user` и `zot-password` обязательны: пользователь Zot, который читает `golden/**` и `mirror/**`.

Выходы: `results` (JSON по инструментам, ключи `gitleaks`, `kics`, `checkov`, `trivy_fs`: `status` `ok`, `error` или `skipped`, `findings`, `gated`, `incomplete`, `raise`) и `verdict` (`pass` или `fail`). В следующем job читаются через `fromJSON(needs.<job>.outputs.results)`.

## Как работает гейт

- Инструмент `ok`, если сканер завершился с кодом 0 и отчёт валиден. Всё остальное — `error`, а не «0 находок».
- Job падает, только если включён `raise-failure` (или переключатель этого инструмента) и у инструмента `error` или есть находки гейта.
- Сломанный пин, упавший `cosign verify`, pull или logout роняют job всегда.
- У checkov OSS нет severity, поэтому считается каждая проваленная проверка. Блокирующий режим начинай с `kics-raise-failure: true`.
- `incomplete` — сколько частей скана сканер не смог прочитать. Это только предупреждение.
- `raise-failure` включает `trivy_fs`, только когда тот включён. Без lockfile `trivy_fs` даёт `ok` с 0 находок и предупреждением: `ok` не значит «чисто».
- trivy относит находку к гейту, только если у неё есть исправление: без исправления она попадает в `findings` и в отчёт, но не в `gated`.
- Возраст Trivy DB в целых часах: больше 72 — предупреждение, больше 168 — `status: error`, DB с датой больше чем на 1 час в будущем тоже даёт `status: error`.

## Отчёты

Артефакт хранится 7 дней и содержит `gitleaks.json`, `kics/kics.json`, `checkov.json` и `trivy_fs.json`. Отчёты собираются заново из списка разрешённых полей, поэтому значения секретов и исходный текст отбрасываются. Имена файлов и ресурсов остаются видны, а артефакт публичного репозитория может скачать любой залогиненный пользователь GitHub.

## Безопасность

- Workflow запускается только на `push`, `pull_request`, `workflow_dispatch`, `schedule` и `merge_group`, на любом другом событии он падает. Форк-PR не получают секреты, поэтому закрывай job условием, как в примере.
- Креды Zot живут только пока тянутся образы и удаляются до сканов.
- Сканер видит workspace только на чтение и свою выходную папку, больше ничего.
- `.checkov.yaml` и `kics.config` проверяемого репозитория игнорируются: они могут запустить код или подменить запросы. `.gitleaks.toml` учитывается.
- Inline-подавления (`#checkov:skip`, `# kics-scan ignore-line`, `gitleaks:allow`) работают, поэтому проверяй их как код.
- `trivy fs` сканирует офлайн: DB качается на хосте до logout из зеркала в Zot, а если зеркало недоступно, из публичного реестра. Подпись DB не проверяется. `trivy.yaml` проверяемого репозитория игнорируется, его `.trivyignore` учитывается и молча скрывает находки, поэтому закрой его CODEOWNERS.
- Скан чужого репозитория через `repository` — по мере сил: сканеры всё равно разбирают его файлы.

## Пины образов

Значения по умолчанию — `zot.homelab/golden/<tool>:<дата>-<sha>@sha256:<digest>` в `.github/workflows/security-pipeline.yml`. Каждый пин проверяется регулярным выражением и подписью `common/image/cosign.pub` до pull. Renovate не видит Zot, поэтому обновляй вручную: `skopeo inspect` для digest, `cosign verify --key common/image/cosign.pub --insecure-ignore-tlog=true <repo>@<digest>`, затем pull request.

## Ограничения

- gitleaks сканирует только рабочее дерево. Для истории есть [`ci-templates/security.yml`](https://github.com/as-platform/ci-templates); в репозитории вызывай один из двух workflow, не оба.
- Сетевой изоляции нет, офлайн держится только на флагах сканеров; `trivy fs` работает без сети.
- `trivy fs` не читает JAR, WAR и EAR и читает `pom.xml` без parent и BOM, поэтому такие части попадают в `incomplete`.
- `ci-templates/security.yml` тоже запускает `trivy fs` (уязвимости, секреты, конфигурации): вызывай один из двух workflow, не оба.
- Через `repository` проверяются только публичные репозитории. Подкаталог монорепо отдельно не проверить.
- Пользователь Zot умеет и писать в Zot: подключай только репозитории, которым доверяешь.

## Примеры

- `examples/lvl_2_base.yml`: мягкий режим на push, pull request и ручной запуск
- `examples/lvl_5_blocking.yml`: `raise-failure: true` и deploy-job, который ждёт гейта
- `examples/scan_external_repo.yml`: ручной скан другого публичного репозитория

## Связанные репозитории

- [`ci-templates`](https://github.com/as-platform/ci-templates): `zot-login` и сервисные workflow
- [`golden-images`](https://github.com/as-platform/golden-images): собирает и подписывает образы сканеров
- [`runner-smoke`](https://github.com/as-platform/runner-smoke): сквозные проверки этого pipeline
- [`k3s`](https://github.com/as-platform/k3s): GitOps-репозиторий с раннером и Zot

## Лицензия

MIT, см. [`LICENSE`](LICENSE).
