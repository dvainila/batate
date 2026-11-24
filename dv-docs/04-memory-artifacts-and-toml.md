# Память, артефакты, TOML-конфиги и итоговая сборка у пользователя

Этот документ отвечает на 4 практических вопроса:
1. Как в BMAD устроена память?
2. Куда ложатся скиллы, артефакты и результаты работы агентов?
3. Как устроены и работают TOML-файлы?
4. Как всё это соединяется вместе на машине конечного пользователя?

---

## 1. Как устроена память в BMAD

В BMAD нет единой «базы данных памяти». Вместо этого используется **4-уровневая файловая память**, где у каждого слоя своя роль:

### 1.1. Статическая память (`persistent_facts` + конфиг)
- **Где живёт:** в `customize.toml` скилла/агента и в оверрайдах `_bmad/custom/<skill-name>.toml` (и `.user.toml`).
- **Как работает:** при каждой активации агент/скилл загружает список `persistent_facts`. Строки записываются в контекст как постоянные правила, а записи вида `file:{project-root}/docs/rules.md` (поддерживаются glob-паттерны) заставляют агента прочитать указанные файлы.
- **Для чего:** стандарты команды, архитектурные запреты, доменные константы.

### 1.2. Оперативная память прогона (`.memlog.md` через `memlog.py`)
- **Где живёт:** в рабочей папке конкретного запуска скилла — `{doc_workspace}/.memlog.md` (например, внутри папки спецификации, брейнсторма или архитектуры).
- **Как работает:** это **append-only лог** в Markdown с простым YAML-frontmatter (`topic`, `goal`, `updated`). Скилл пишет туда однострочные записи с тегами через детерминированный скрипт:
  ```bash
  uv run {project-root}/_bmad/scripts/memlog.py append --workspace {doc_workspace} --type decision --text "..."
  ```
  Скрипт делает атомарную запись (через temp-файл + `fsync` + `rename`) и не требует перечитывать файл в процессе работы.
- **Зачем нужен:**
  1. Переживает сжатие контекстного окна LLM и закрытие чата — при возобновлении (`resume`) скилл читает `.memlog.md` и восстанавливает состояние.
  2. Итоговый документ (`SPEC.md`, `ARCHITECTURE-SPINE.md`, `PRD` и т.д.) **дистиллируется из `.memlog.md` в самом конце**, а не пишется на лету.

### 1.3. Межсессионная память командных комнат (`bmad-party-mode`)
- **Где живёт:** `{output_folder}/party-mode/memories/<party-id>/.memlog.md` (путь задаётся параметром `memory_dir` в `bmad-party-mode/customize.toml`).
- **Как работает:** если у комнаты включено `memory = true` (или глобально `party_memory = true`), участники помнят ключевые итоги прошлых обсуждений между сессиями. Это тоже `.memlog.md`, а не полный транскрипт чата.

### 1.4. Долговременная память проекта (артефакты и `AGENTS.md`)
- **Инструкции репозитория:** скилл `bmad-project-context` поддерживает managed-блок в `AGENTS.md` в корне репозитория (проверенные команды сборки/тестов, конвенции, известные грабли).
- **Цепочка артефактов:** каждый следующий скилл опирается на файлы, сохранённые предыдущим:
  `Product Brief / PRFAQ` → `PRD` → `UX` → `Architecture` → `Spec` → `Epics & Stories` → `Sprint Status` → `Code + Tests + Retro`.

---

## 2. Куда ложатся скиллы, артефакты и результаты работы

### 2.1. Куда ложатся скиллы после установки
Установщик (`tools/installer/ide/platform-codes.yaml`) раскладывает скиллы **напрямую в директорию выбранной IDE** (в `_bmad/` копии скиллов не хранятся — после копирования в IDE установщик очищает временные папки):

| IDE / Платформа | Внутри проекта (`target_dir`) | Глобально (`global_target_dir`) |
| --- | --- | --- |
| **Cursor, Codex, Windsurf, GitHub Copilot, Amp, Auggie** | `.agents/skills/<skill-name>/` | `~/.agents/skills/` (для Codex: `~/.codex/skills/`) |
| **Claude Code** | `.claude/skills/<skill-name>/` | `~/.claude/skills/` |
| **Cline** | `.cline/skills/<skill-name>/` | `~/.cline/skills/` |
| **GitHub Copilot (дополнительно)** | `.github/agents/` (команды-обёртки) | — |

Внутри `.agents/skills/bmad-agent-dev/` лежит точная копия папки скилла (`SKILL.md`, `customize.toml`, `references/`, `assets/`).

### 2.2. Куда ложатся артефакты скиллов и результаты работы агентов
Все выходные пути настраиваются в `src/core-skills/module.yaml` и `src/bmm-skills/module.yaml` и записываются в `_bmad/config.toml`:

| Категория | Переменная конфига | Дефолтный путь | Что туда попадает |
| --- | --- | --- | --- |
| **Корень артефактов** | `output_folder` | `{project-root}/_bmad-output` | Базовая папка всех выходных артефактов |
| **Планирование** | `planning_artifacts` | `_bmad-output/planning-artifacts` | Product Brief, PRFAQ, PRD, UX, Architecture Spine, Epics & Stories, Change Proposals (`bmad-correct-course`) |
| **Реализация** | `implementation_artifacts` | `_bmad-output/implementation-artifacts` | `sprint-status.yaml` (`bmad-sprint-planning`), файлы историй, отчёты `bmad-build`, QA E2E тесты, ретроспективы |
| **Спецификации** | — | `_bmad-output/specs/spec-{slug}/` | `SPEC.md`, `.memlog.md` и companion-файлы (`bmad-spec`) |
| **Брейнсторминг** | — | `_bmad-output/brainstorming/` | Сессии и `.memlog.md` (`bmad-brainstorming`) |
| **Память party-mode** | `memory_dir` | `_bmad-output/party-mode/memories/` | Персональная память групп агентов |
| **Знания о проекте** | `project_knowledge` | `{project-root}/docs` | Долгоживущая документация, исследования (`bmad-deep-recon`), гайды |
| **Код и репо** | `{project-root}` | Корень проекта | Исходный код и тесты (`bmad-agent-dev` / `bmad-build`), `AGENTS.md` (`bmad-project-context`) |

> Папки `planning_artifacts`, `implementation_artifacts` и `project_knowledge` не создаются установщиком заранее — их лениво создаёт первый скилл, который туда пишет.

---

## 3. Как работают TOML-файлы

В BMAD используется строго типизированный TOML (парсится стандартной библиотекой Python 3.11+ `tomllib` в `src/scripts/config_utils.py`).

### Два независимых контура TOML

#### Контур A: Центральный конфиг (`resolve_config.py`)
Отвечает за настройки всего проекта и пользователя. Собирается из **4 слоёв** (каждый следующий переопределяет предыдущий):
1. `_bmad/config.toml` — командные настройки (обязательный, генерируется установщиком).
2. `_bmad/config.user.toml` — личные настройки разработчика (`user_name`, `communication_language`, `user_skill_level`).
3. `_bmad/custom/config.toml` — командные кастомные дополнения (переживают обновления BMAD).
4. `_bmad/custom/config.user.toml` — личные кастомные дополнения.

Пример структуры `_bmad/config.toml`:
```toml
[core]
project_name = "my-app"
output_folder = "{project-root}/_bmad-output"

[modules.bmm]
planning_artifacts = "{project-root}/_bmad-output/planning-artifacts"
implementation_artifacts = "{project-root}/_bmad-output/implementation-artifacts"
project_knowledge = "{project-root}/docs"
```

#### Контур B: Кастомизация скилла или агента (`resolve_customization.py`)
Отвечает за поведение конкретного скилла. Собирается из **3 слоёв**:
1. `{skill-root}/customize.toml` — дефолты из пакета (перезаписываются при обновлении BMAD).
2. `{project-root}/_bmad/custom/{skill-name}.toml` — командные оверрайды (не трогаются установщиком).
3. `{project-root}/_bmad/custom/{skill-name}.user.toml` — личные оверрайды.

Внутри используются таблицы `[agent]` (для агентов) или `[workflow]` (для скиллов), а также массивы таблиц `[[agent.menu]]`.

### Правила слияния (`structural_merge`)
1. **Скаляры** (строки, числа, булевы) — новое значение перезаписывает старое.
2. **Таблицы** (`[table]`) — рекурсивный deep-merge по ключам.
3. **Обычные массивы** (`principles`, `persistent_facts`, `activation_steps_prepend`, `activation_steps_append`) — **дополняются (append)** в конец, а не заменяются.
4. **Массивы таблиц с ключом `code` или `id`** (например `[[agent.menu]]`, `[[workflow.party_groups]]`) — если элемент с таким `code`/`id` уже есть, он **заменяется целиком**; если нет — **добавляется в конец**.

> Важно: кастомные файлы в `_bmad/custom/` должны быть разреженными (sparse) — в них нужно писать только те поля, которые вы добавляете или меняете, а не копировать весь `customize.toml`.

---

## 4. Как всё собирается вместе у конечного пользователя

Полный цикл от установки до результата выглядит так:

```text
1. Установка (один раз / при обновлении)
   bmad install
     ├── Спрашивает настройки из module.yaml -> пишет _bmad/config.toml и config.user.toml
     ├── Копирует src/scripts/*.py -> _bmad/scripts/
     ├── Пишет индексы -> _bmad/_config/manifest.yaml и skill-manifest.csv
     └── Раскладывает скиллы -> .agents/skills/ (Cursor) или .claude/skills/ (Claude Code)

2. Обнаружение в IDE
   IDE сканирует .agents/skills/*/SKILL.md (YAML frontmatter: name + description)
   Пользователь пишет: "Позови Amelia" или "Запусти bmad-architecture"

3. Активация скилла/агента (в момент вызова)
   LLM читает SKILL.md и выполняет первые шаги:
     ├── uv run _bmad/scripts/resolve_customization.py --skill ... --project-root ...
     │     └── Мерджит customize.toml + _bmad/custom/<skill>.toml + .user.toml -> JSON
     ├── uv run _bmad/scripts/resolve_config.py --project-root ...
     │     └── Мерджит 4 слоя центрального config*.toml -> JSON
     └── Подгружает persistent_facts (в т.ч. file:...) и применяет язык/персону

4. Работа и сохранение результата
   ├── (Если скилл сложный) render_skill.py собирает снапшот из .md и конфига
   ├── (Если сессия длинная) memlog.py дописывает факты в {doc_workspace}/.memlog.md
   ├── По ходу работы скилл может вызывать соседние скиллы ("Invoke the X skill")
   └── В финале записывает артефакт в _bmad-output/... (или меняет код / AGENTS.md)
```

Если в Python-среде у пользователя что-то пошло не так и `uv run` упал, во всех `SKILL.md` предусмотрен fallback: агент читает те же TOML-файлы напрямую и применяет правила `structural_merge` вручную.
