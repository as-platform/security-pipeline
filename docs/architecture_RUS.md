# Архитектура

🇬🇧 [English version](architecture.md)

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

