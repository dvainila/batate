# Локальная и кастомная установка BMAD из этого репозитория

Установщик можно запускать **напрямую из этого репозитория полностью локально (offline)** — без публикации в npm и без скачивания с GitHub.

Это работает за счёт `tools/installer/project-root.js`: CLI определяет корень репозитория, в котором запущен, и берёт встроенные модули напрямую с диска:
- `core` → `src/core-skills/`
- `bmm` → `src/bmm-skills/`
- runtime-скрипты → `src/scripts/`

Любые твои правки в `src/` сразу попадут в целевой проект.

---

## 1. Базовый запуск в отдельную папку

### Интерактивный режим
```bash
node tools/installer/bmad-cli.js install --directory /path/to/target-project
```

### Неинтерактивный режим (одной командой)
Пример установки для Cursor без шимов и с русским языком общения:
```bash
node tools/installer/bmad-cli.js install \
  --directory /path/to/target-project \
  --modules bmm \
  --tools cursor \
  --communication-language Russian \
  --document-output-language Russian \
  --no-shims \
  --yes
```
*(Модуль `core` устанавливается автоматически вместе с `bmm`.)*

---

## 2. Полезные флаги CLI (`tools/installer/commands/install.js`)

| Флаг | Назначение |
| --- | --- |
| `--directory <path>` | Целевая папка проекта (по умолчанию текущая директория) |
| `--modules <list>` | Список модулей через запятую (например `bmm`) |
| `--tools <list>` | Целевые IDE через запятую (`cursor`, `claude-code`, `codex`, `windsurf`, `github-copilot` и др.) |
| `--list-tools` | Показать все поддерживаемые ID IDE и их папки установки |
| `--set <module>.<key>=<value>` | Переопределить параметр из `module.yaml` без промптов (например `bmm.user_skill_level=expert`) |
| `--list-options [module]` | Показать доступные ключи для `--set` |
| `--user-name <name>` | Имя пользователя для агентов |
| `--communication-language <lang>` | Язык общения агентов |
| `--document-output-language <lang>` | Язык генерируемых документов |
| `--output-folder <path>` | Папка для артефактов (по умолчанию `_bmad-output`) |
| `--shims` / `--no-shims` | Ставить или не ставить deprecated `v6-shims` |
| `--action <type>` | Режим для существующей установки: `install`, `update`, `quick-update` |
| `-y, --yes` | Принять дефолты и пропустить интерактивные вопросы |

---

## 3. Установка собственного локального модуля (`--custom-source`)

Чтобы не смешивать свои скиллы с `src/bmm-skills/`, можно вынести их в отдельную папку-модуль рядом и подключать при установке без Git/GitHub:

```bash
node tools/installer/bmad-cli.js install \
  --directory /path/to/target-project \
  --modules bmm \
  --custom-source /path/to/my-custom-module \
  --tools cursor \
  --yes
```
*(Можно передать несколько путей или Git URL через запятую.)*

Минимальная структура локального кастомного модуля:
```text
my-custom-module/
├── module.yaml          # code, name, description и опциональные вопросы (prompt:)
├── module-help.csv      # опционально: для интеграции с bmad-help
└── my-skills/
    └── bmad-my-skill/
        └── SKILL.md
```

---

## 4. Итеративный цикл разработки и обновления

1. Вносишь правки в `src/` (или в свой локальный модуль).
2. Проверяешь валидность скилла перед установкой:
   ```bash
   uv run --python 3.11 tools/validate_skills.py --strict src/bmm-skills/plan/bmad-my-skill
   ```
3. Накатываешь обновления в целевой проект:
   ```bash
   node tools/installer/bmad-cli.js install \
     --directory /path/to/target-project \
     --action quick-update \
     --yes
   ```
   Установщик обновит скиллы в `.agents/skills/` (или `.claude/skills/`) и скрипты в `_bmad/scripts/`, **сохранив** твои проектные/личные оверрайды в `_bmad/custom/`.

---

## 5. Когда установщик всё-таки полезет в сеть

Сеть нужна **только** в двух случаях:
- если выбрать внешние официальные модули из `bmad-modules.yaml` (`bmb`, `cis`, `tea`, `bmad-loop`, `gds`) — они клонируются с GitHub в `~/.bmad/cache/external-modules/`;
- если передать Git URL в `--custom-source`.

Если ставить только `core`, `bmm` и локальные папки через `--custom-source`, установка проходит на 100% offline.
