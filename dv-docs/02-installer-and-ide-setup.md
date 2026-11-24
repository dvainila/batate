# Установщик и интеграция с IDE (`tools/installer/`)

Точка входа CLI — `tools/installer/bmad-cli.js` (в `package.json` зарегистрирован как `bmad` / `bmad-method`, скрипты `npm run bmad:install` и `npm run bmad:uninstall`).

---

## 1. Что происходит при `bmad install`

Основной оркестратор — `tools/installer/core/installer.js` (`class Installer`). Пайплайн установки:

1. **Определение путей (`core/install-paths.js`)**:
   - `{project-root}` — корень целевого проекта.
   - `{project-root}/_bmad/` — служебная папка BMAD в проекте.
2. **Сбор конфига (`core/config.js`)**:
   - Читает `module.yaml` каждого выбранного модуля (включая `core` и `bmm`).
   - Интерактивно задаёт вопросы с `prompt:` и записывает ответы в `_bmad/config.toml` (scope: team) и `_bmad/config.user.toml` (scope: user).
3. **Установка runtime-скриптов и служебных файлов**:
   - Копирует `src/scripts/*.py` в `_bmad/scripts/`.
   - Генерирует `_bmad/_config/manifest.yaml`, `_bmad/_config/skill-manifest.csv` и общий `module-help.csv`.
   - Сохраняет/восстанавливает пользовательские файлы из `_bmad/custom/` при обновлении.
4. **Деплой скиллов в IDE (`ide/manager.js`, `ide/_config-driven.js`)**:
   - Все скиллы модулей копируются в директорию выбранной IDE.
   - После копирования в IDE временные копии скиллов из `_bmad/` удаляются (`_cleanupSkillDirs`), чтобы не плодить дубли.
5. **Политика шимов (`core/shim-policy.js`)**:
   - Решает, ставить или удалять `v6-shims` на основе флага `--shims`, предыдущего манифеста или наличия уже установленных deprecated-скиллов.

---

## 2. Как поддерживаются разные IDE (`platform-codes.yaml`)

Вся настройка IDE декларативна и описана в `tools/installer/ide/platform-codes.yaml`. Для каждой платформы заданы:
- `target_dir` — папка скиллов внутри проекта (например, `.claude/skills` для Claude Code, `.cursor/skills` для Cursor, `.agents/skills` для Codex / Amp / Auggie и др.);
- `global_target_dir` — папка для глобальной установки в домашней директории пользователя (например, `~/.claude/skills`).

Установщик не переписывает формат скиллов под каждую IDE (раньше были отдельные адаптеры, сейчас всё сведено к единому стандарту Agent Skills и `_config-driven.js`).

---

## 3. Внешние и кастомные модули (`tools/installer/modules/`)

- **`official-modules.js` + `external-manager.js`** — загружают `bmad-modules.yaml`, клонируют/скачивают официальные внешние модули (поддерживаются каналы `stable`, `next`, `pin` на конкретный git-тег, а также `marketplace-plugin: true` для модулей, где скиллы разложены через `.claude-plugin/marketplace.json`).
- **`custom-module-manager.js`** — позволяет устанавливать локальные или сторонние кастомные модули (директории со своим `module.yaml` и набором скиллов).

---

## 4. Что появляется в целевом проекте после установки

```text
target-project/
├── _bmad/
│   ├── config.toml              # Общий конфиг команды
│   ├── config.user.toml         # Личный конфиг (в .gitignore)
│   ├── custom/                  # Твои оверрайды (не затираются при update!)
│   │   ├── config.toml
│   │   └── <skill-name>.toml
│   ├── scripts/                 # Копия src/scripts/*.py
│   └── _config/
│       ├── manifest.yaml        # Версии установленных модулей и IDE
│       └── skill-manifest.csv   # Индекс всех установленных скиллов
├── _bmad-output/                # Создаётся для артефактов (planning / implementation)
└── .cursor/skills/              # (или .claude/skills/, .agents/skills/ и т.д.)
    ├── bmad-agent-dev/
    ├── bmad-architecture/
    └── ...
```
