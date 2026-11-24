# Архитектура и runtime BMAD

BMAD v6 построен вокруг стандарта **Agent Skills** (`SKILL.md`): каждый агент и каждый рабочий процесс (workflow) — это самостоятельный скилл (папка с `SKILL.md`), который агентная IDE (Cursor, Claude Code, Codex и др.) подхватывает нативно.

---

## 1. Модули в `src/`

Внутри репозитория встроены два базовых модуля:

- **`src/core-skills/`** (`code: core`) — общие инструменты для любого проекта:
  - `bmad-help` — маршрутизация и подсказки, что делать дальше.
  - `bmad-customize` — интерактивная настройка `_bmad/custom/`.
  - `bmad-brainstorming`, `bmad-forge-idea`, `bmad-advanced-elicitation`, `bmad-deep-recon`, `bmad-party-mode`, `bmad-review`.
- **`src/bmm-skills/`** (`code: bmm`, «BMad Method») — полный цикл разработки ПО:
  - `agents/` — 5 персон-агентов: `bmad-agent-analyst` (Mary), `bmad-agent-pm` (John), `bmad-agent-ux-designer` (Sally), `bmad-agent-architect` (Winston), `bmad-agent-dev` (Amelia).
  - `plan/` — фаза планирования (`bmad-product-brief`, `bmad-prfaq`, `bmad-prd`, `bmad-ux`, `bmad-architecture`, `bmad-spec`, `bmad-create-epics-and-stories`, `bmad-sprint-planning`, `bmad-project-context`, `bmad-generate-project-context`).
  - `ship/` — фаза реализации (`bmad-build`, `bmad-build-auto`, `bmad-code-review`, `bmad-qa-generate-e2e-tests`, `bmad-correct-course`, `bmad-walkthrough`, `bmad-retrospective`).

Каждый модуль описывается двумя файлами:
- **`module.yaml`** — метаданные модуля (`code`, `name`, `description`), вопросы конфигурации для установщика (`prompt`, `default`, `scope: user`), создаваемые директории и (для `bmm`) список `agents:` с краткой эссенцией персон (для `party-mode`, `bmad-help` и др.).
- **`module-help.csv`** — каталог доступных скиллов модуля (используется `bmad-help` для рекомендаций следующего шага).

Внешние модули (например, `bmad-builder`, `bmad-loop`, `tea`) регистрируются в корневом `bmad-modules.yaml` и при установке скачиваются из GitHub.

---

## 2. Анатомия скилла

Минимальный скилл — это папка `<skill-name>/` с файлом `SKILL.md`:

```text
bmad-my-skill/
├── SKILL.md           # Точка входа: YAML frontmatter (name, description) + инструкции
├── customize.toml     # Опционально: настраиваемые поля [workflow] или [agent]
├── references/        # Опционально: дополнительные гайды (подгружаются по мере необходимости)
├── assets/            # Опционально: шаблоны, CSV-таблицы, схемы
├── scripts/           # Опционально: вспомогательные скрипты самого скилла
└── steps/             # Опционально: пошаговый workflow (step-01-*.md, step-02-*.md)
```

Допустимы 3 раскладки (описаны в `tools/skill-validator.md`):
1. **Single-file** — вся логика в `SKILL.md` + вспомогательные файлы (`references/`, `templates/`).
2. **Flat step files** — `step-NN-name.md` рядом с `SKILL.md` в корне скилла.
3. **`steps/` subdirectory** — `steps/step-NN-name.md`.

### Разница между обычным скиллом и агент-скиллом

- **Workflow-скилл** (например `bmad-architecture`): при активации резолвит `[workflow]` из `customize.toml` и центральный конфиг, ведёт процесс по шагам, может вызывать другие скиллы через формулировку ``Invoke the `skill-name` skill``.
- **Агент-скилл** (например `bmad-agent-dev`): при активации резолвит `[agent]` из `customize.toml`, принимает персону (имя, стиль речи, `principles`), загружает `persistent_facts`, выполняет `activation_steps_prepend` / `activation_steps_append` и показывает интерактивное меню `[[agent.menu]]` (каждый пункт меню либо вызывает `skill = "..."`, либо исполняет `prompt = "..."`).

---

## 3. Runtime-движок (`src/scripts/`)

Скиллы — это markdown-промпты, которые в нужные моменты вызывают детерминированные Python-скрипты (через `uv run`, требуется Python 3.11+ ради встроенного `tomllib`). При установке эти скрипты копируются в `{project-root}/_bmad/scripts/`.

### 3.1. Центральный конфиг (`resolve_config.py` + `config_utils.py`)

Хранит проектные и пользовательские настройки (`user_name`, `communication_language`, `document_output_language`, `output_folder`, `planning_artifacts`, `implementation_artifacts`, `project_knowledge` и др.).

Скрипт мерджит **4 слоя TOML** (последующий переопределяет предыдущий):
1. `_bmad/config.toml` — командные настройки проекта (создаётся установщиком, обязательный).
2. `_bmad/config.user.toml` — локальные настройки пользователя (git-ignored).
3. `_bmad/custom/config.toml` — командные кастомные переопределения (переживают обновления BMAD).
4. `_bmad/custom/config.user.toml` — личные кастомные переопределения.

Вызов из скилла:
```bash
uv run {project-root}/_bmad/scripts/resolve_config.py --project-root {project-root}
```

### 3.2. Кастомизация скилла (`resolve_customization.py`)

Каждый настраиваемый скилл имеет свой `customize.toml` (дефолты, перезаписываются при обновлении BMAD). Пользователь может переопределять любые поля без правки самого скилла.

Скрипт мерджит **3 слоя TOML**:
1. `{skill-root}/customize.toml` — дефолты скилла.
2. `{project-root}/_bmad/custom/{skill-name}.toml` — командные оверрайды.
3. `{project-root}/_bmad/custom/{skill-name}.user.toml` — личные оверрайды.

**Правила структурного слияния (`structural_merge` в `config_utils.py`):**
- Скаляры — перезаписываются.
- Таблицы (объекты) — рекурсивный deep-merge.
- Массивы таблиц с ключом `code` или `id` (например `[[agent.menu]]`) — элементы с совпадающим `code`/`id` заменяются целиком, новые добавляются в конец.
- Остальные массивы (например `principles`, `persistent_facts`, `activation_steps_*`) — **дополняются (append)**, а не перезаписываются.

### 3.3. Рендеринг снапшотов (`render_skill.py`)

Используется сложными скиллами с жёсткой структурой (сейчас это `bmad-build` и `bmad-build-auto`). Скрипт собирает все `.md` файлы скилла (кроме `SKILL.md`), подставляет compile-time токены (`{{config.key}}`, `{{.key}}`, `{workflow.key}`, инклюды `[[bmad-snapshot:file.md]]`) и записывает неизменяемый снапшот в проект. Обычные скиллы `render_skill.py` не используют и подставляют переменные в runtime.

### 3.4. Рабочая память (`memlog.py`)

Append-only лог `.memlog.md` для длинных сессий (например, `bmad-architecture`, `bmad-spec`). Агенты дописывают туда факты по мере работы (`decision`, `constraint`, `version`, `assumption`, `question`, `direction`, `event`), а в конце дистиллируют итоговый документ из лога. Это позволяет безопасно прерывать и возобновлять длинные воркфлоу.
