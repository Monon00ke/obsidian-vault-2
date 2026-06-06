---
title: "Памятка для Grok — что сделано и что дальше"
---

## Что сделано в этой сессии (OpenCode / DeepSeek)

### Репозиторий obsidian-vault-2 (чистый)
- **URL:** https://github.com/Monon00ke/obsidian-vault-2
- **Сайт (Quartz):** https://monon00ke.github.io/obsidian-vault-2/ (Pages ещё не настроен, нужно включить Settings → Pages → GitHub Actions)

### Структура проекта — Quartz (v4)
- **Тип сайта:** Quartz static site generator (Obsidian → холст знаний)
- **Папка с заметками:** `content/`
- **Внутри:** 00_Навигация, English, Math, OS Labs, Physics, Presentations, Functional Analysis, Engineering (удалена), Canvases, Photos

### Что сделали:
1. **ОС — лабы:** Удалили дубликаты (оставили только английские версии). MOC файл ссылается на все лабы.
2. **Удалили папку Engineering** (был битый YAML — `Инженерная подготовка — экзамен.md`)
3. **Починили YAML frontmatter** в двух файлах (Engineering exam, DE задачи)
4. **Структура:** Все vault папки внутри `content/` (Quartz ожидает именно так)
5. **Дубликаты в OS Labs:** 14 уникальных лаб (3–23), русские копии удалены
6. **Physics:** Английский файл `Diffraction Grating.md` переименован в русский

### Проблемы / что дальше:
1. **GitHub Pages не включён** — нужно зайти в Settings → Pages → Source: GitHub Actions → Save
2. **Битый файл Engineering** уже удалён из репозитория, но workflow может упасть если есть ещё битые YAML (проверить все файлы на правильность frontmatter)
3. **Модель:** Продолжать на Grok Build с OAuth (подписка SuperGrok)
4. **Quartz требует правильный YAML frontmatter:** все `.md` файлы должны начинаться с `---\ntitle: "..."\n---`
5. В репозитории есть старые Quartz файлы в корне (Dockerfile, package.json и т.д.) — они нужны для сборки, не трогать

### Адреса:
- **Основной репозиторий:** https://github.com/Monon00ke/obsidian-vault (там же старый сайт https://monon00ke.github.io/obsidian-vault/)
- **Новый репозиторий:** https://github.com/Monon00ke/obsidian-vault-2
