# Документация по устройству BMAD-METHOD (v6.12.4)

Эта папка (`dv-docs/`) описывает внутреннее устройство репозитория `BMAD-METHOD` с одной практической целью: **быстро понять архитектуру, чтобы расширять BMAD под свои задачи**.

## Карта документов

1. **[01-architecture-and-runtime.md](./01-architecture-and-runtime.md)** — ядро BMAD: модули (`core-skills`, `bmm-skills`), анатомия скиллов и агентов, многослойная система конфигов/кастомизации (TOML) и Python-скрипты в `src/scripts/`.
2. **[02-installer-and-ide-setup.md](./02-installer-and-ide-setup.md)** — как работает CLI-установщик (`tools/installer/`): куда попадают скиллы, конфиги и манифесты в целевом проекте, как подключаются IDE и внешние модули.
3. **[03-how-to-extend.md](./03-how-to-extend.md)** — практический гайд по расширению: кастомизация через `_bmad/custom/`, создание своих скиллов/агентов/модулей, правила валидации и локальное тестирование.
4. **[04-memory-artifacts-and-toml.md](./04-memory-artifacts-and-toml.md)** — устройство памяти (4 слоя), куда ложатся скиллы, артефакты и результаты работы агентов, детальный разбор TOML-файлов и полная схема сборки на машине конечного пользователя.
5. **[05-local-custom-install.md](./05-local-custom-install.md)** — полностью локальная (offline) установка из этого репозитория в отдельный проект, флаги CLI, подключение локальных `--custom-source` модулей и `quick-update`.

### Диаграммы

- **[diagram-01-modules-and-config.drawio](./diagram-01-modules-and-config.drawio)** — Уровни 1–2: структура `src/`, разбор `module.yaml` / `module-help.csv`, сборка `_bmad/config.toml`, вызов `bmad-party-mode`.
- **[diagram-01.1-source-architecture.drawio](./diagram-01.1-source-architecture.drawio)** — Архитектура до установки: карта `src/` + installer, YAML/CSV манифесты, два контура TOML, кастомизация `_bmad/custom/` и четыре файловых хранилища.
- **[diagram-02-skills-execution-and-memory.drawio](./diagram-02-skills-execution-and-memory.drawio)** — Уровень 3: четыре примера (`bmad-create-epics-and-stories`, `bmad-brainstorming`, `bmad-build`, `bmad-agent-architect`) от слэш-команды до результата, плюс верхний слой памяти, конфигов и артефактов.
- **[diagram-03-local-install-mapping.drawio](./diagram-03-local-install-mapping.drawio)** — Локальная установка в пустой проект (два дерева: `BMAD-METHOD-6.12.4/` слева ↔ `my-project/` справа по образцу `clean_bmad_v6` без `bmb`, 3 этапа переноса, слияния и генерации).

---

## На какие части репозитория НЕ нужно отвлекаться

Если задача — понять механику работы BMAD и расширять его, смело игнорируй:

| Путь | Что это | Почему можно пропустить |
| --- | --- | --- |
| `.github/` | GitHub Actions (CI, релизы, авто-проверки) | Инфраструктура CI самого open-source репо |
| `.husky/` | Git pre-commit/commit-msg хуки (`lint-staged`) | Локальные проверки перед коммитом в этот репо |
| `.vscode/` | Настройки редактора VS Code | Конфигурация IDE для контрибьюторов |
| `.coderabbit.yaml`, `greptile.json` | Конфиги AI-ревьюеров PR | Нужны только для PR в upstream-репозиторий |
| `docs/` | Пользовательская документация (гайды, концепты) | Контент для сайта; для понимания кода не нужен |
| `website/` | Astro + Starlight сайт документации | Сборка и хостинг `docs.bmad-method.org` |
| `test/` | Автотесты установщика, ссылок и рендерера | Полезно запускать как проверку, но не для чтения при старте |
| `.claude-plugin/` | `marketplace.json` для каталога плагинов Claude | Метаданные дистрибуции плагина |

---

## Разбор сомнительных папок: `shims` и `web-bundles`

### 1. `v6-shims` (`src/core-skills/v6-shims/` и `src/bmm-skills/v6-shims/`)

Отдельной корневой папки `shims/` в репозитории нет — это подпапки `v6-shims/` внутри `src/core-skills/` и `src/bmm-skills/` плюс логика в `tools/installer/core/shim-policy.js`.

- **Что это:** устаревшие скиллы со старыми ID из ранних версий v6 (например, `bmad-create-prd`, `bmad-quick-dev`, `bmad-dev-story`). Они нужны только для обратной совместимости: большинство просто перенаправляют вызов на новый скилл (`bmad-prd`, `bmad-build` и т.д.), передавая ему готовый `intent`. В v7 их удалят.
- **Нужно ли смотреть?** **Нет.** При создании новых скиллов на них ориентироваться не нужно. Единственное, о чём полезно знать: установщик сканирует модули рекурсивно и отличает шимы по `metadata.lifecycle: shim` в frontmatter `SKILL.md` (а также по префиксу `Deprecated` в `description`).

### 2. `web-bundles/`

- **Что это:** предсобранные самодостаточные пакеты (`SKILL.md`, `INSTRUCTIONS.md`, CSV-данные) для **Google Gemini Gems** и **ChatGPT Custom GPTs** (чтобы прогонять brainstorming, brief, PRD в веб-подписке, не тратя токены в IDE). Генерируются из обычных скиллов утилитой `bmad-os-skill-to-bundle`, упаковываются в ZIP скриптом `tools/bundle-web-bundles.js`.
- **Нужно ли смотреть?** **Нет, если ты работаешь в IDE** (Cursor, Claude Code, Codex и др.). Это отдельный канал дистрибуции для веб-чатов. Может пригодиться только в случае, если захочешь выгрузить свой кастомный скилл в Gemini Gem / Custom GPT.

---

## На что действительно стоит смотреть

1. **`src/`** — вся суть BMAD:
   - `src/core-skills/` — базовые утилиты (`bmad-help`, `bmad-customize`, `bmad-review`, `bmad-brainstorming` и др.).
   - `src/bmm-skills/` — основной модуль разработки: агенты (`agents/`), планирование (`plan/`), реализация (`ship/`).
   - `src/scripts/` — Python-движок резолвинга конфигов, кастомизации, рендеринга снапшотов и memlog.
2. **`bmad-modules.yaml`** — реестр официальных внешних модулей (`bmb`, `cis`, `tea`, `bmad-loop`, `gds`).
3. **`tools/installer/`** — CLI (`bmad-cli.js`), логика установки в `_bmad/` и IDE-папки (`platform-codes.yaml`).
4. **`tools/skill-validator.md` и `tools/validate_skills.py`** — спецификация и валидатор того, как должен быть написан любой скилл.
