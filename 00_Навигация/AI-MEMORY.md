# AI Assistant Memory - Monon00ke's Digital Garden

> Этот файл содержит всю необходимую информацию для продолжения работы над проектом в новой сессии.

## Проект

**Monon00ke's Digital Garden** — персональный сайт-заметки на базе [Quartz v4.5.2](https://quartz.jzhao.xyz/), развёрнутый на GitHub Pages.

- **Репозиторий**: https://github.com/Monon00ke/obsidian-vault
- **Сайт**: https://monon00ke.github.io/obsidian-vault/
- **Владелец**: Monon00ke (GitHub)

## Аутентификация

### GitHub CLI
```bash
gh auth login
```

### Token (если gh не работает)
Токен хранится у пользователя. Для push использовать:
```bash
git remote set-url origin https://Monon00ke:TOKEN@github.com/Monon00ke/obsidian-vault.git
```

### Git config
```bash
git config user.email "monon00ke@gmail.com"
git config user.name "Monon00ke"
```

## Начало работы в новой сессии

### 1. Клонировать репозиторий
```bash
git clone https://github.com/Monon00ke/obsidian-vault.git
cd obsidian-vault
```

### 2. Установить зависимости
```bash
npm install
```

### 3. Проверить сборку
```bash
npx quartz build
```

### 4. Запустить локальный сервер (опционально)
```bash
npx quartz serve
# Откроется на http://localhost:8080
```

## Структура проекта

```
obsidian-vault/
├── content/                    # Все заметки (Obsidian vault)
│   ├── index.md               # Главная страница
│   ├── English/               # Английский язык
│   │   ├── Grammar/
│   │   ├── IELTS/
│   │   │   └── Speaking Topics/
│   │   ├── Vocabulary/
│   │   └── Exam Prep/
│   ├── Math/                  # Математика
│   │   ├── Math Analysis/
│   │   └── Math Statistics/
│   ├── OS Labs/               # Лабораторные по ОС
│   ├── Functional Analysis/   # Функциональный анализ
│   ├── Scientific Work/       # Научная работа
│   └── Photos/                # Изображения
── quartz.config.ts           # Конфигурация сайта
├── quartz.layout.ts           # Раскладка страниц
├── .github/workflows/
│   └── deploy.yaml            # GitHub Actions для деплоя
├── 404.html                   # SPA fallback для GitHub Pages
├── SETUP.md                   # Инструкция для пользователя
└── AI-MEMORY.md               # Этот файл
```

## Конфигурация Quartz

### quartz.config.ts — ключевые настройки
```typescript
const config: QuartzConfig = {
  configuration: {
    pageTitle: "Monon00ke's Digital Garden",
    baseUrl: "monon00ke.github.io/obsidian-vault",  // БЕЗ https://
    locale: "ru-RU",
    markdownLinkResolution: "relative",  // Важно для вложенных папок
    enableSPA: true,
    enablePopovers: true,
    ignorePatterns: ["private", "templates", ".obsidian"],
    theme: {
      typography: {
        header: "Schibsted Grotesk",
        body: "Source Sans Pro",
        code: "IBM Plex Mono",
      },
      // ... цвета light/dark mode
    },
  },
  plugins: {
    transformers: [
      Plugin.FrontMatter(),
      Plugin.ObsidianFlavoredMarkdown({ enableInHtmlEmbed: false }),
      Plugin.CrawlLinks({ markdownLinkResolution: "relative" }),
      Plugin.Latex({ renderEngine: "katex" }),
      // ...
    ],
    emitters: [
      Plugin.ContentPage(),
      Plugin.FolderPage(),
      Plugin.Graph(),  // Граф связей
      // ...
    ],
  },
}
```

### quartz.layout.ts — компоненты
- **Left sidebar**: Explorer, Search, Dark mode, Reader mode
- **Right sidebar**: Graph (мини-граф + кнопка глобального графа), TableOfContents, Backlinks
- **SPA enabled**: навигация без перезагрузки страницы

## Формат файлов

### Front Matter (обязателен)
```markdown
---
title: "Название заметки"
---
```

### Ссылки
Использовать Obsidian wikilinks:
```markdown
[[File Name]]                    # Ссылка на файл
[[File Name|Display Text]]       # Ссылка с текстом
[[Folder/File Name]]             # Ссылка на файл в папке
```

**Важно**: `markdownLinkResolution: "relative"` — ссылки разрешаются относительно текущего файла.

### Изображения
```markdown
![[Pasted image 20260518192021.png]]
```
Хранить в `content/Photos/` или в той же папке что и заметка.

### Backlinks
Добавлять в конец каждой заметки:
```markdown
---
## Backlinks

- [[00 OS Labs MOC]]
- [[Related Note]]
```

## Деплой

### Автоматический (через GitHub Actions)
```bash
git add .
git commit -m "Описание изменений"
git push
```
GitHub Actions автоматически соберёт и задеплоит (~2-3 минуты).

### Проверка статуса деплоя
```bash
gh api repos/Monon00ke/obsidian-vault/actions/runs --jq '.workflow_runs[0] | .status, .conclusion'
```

### Ручной запуск деплоя
```bash
gh workflow run "Deploy Quartz site to GitHub Pages" --repo Monon00ke/obsidian-vault
```

## Известные проблемы и решения

### 404 при переходе по ссылкам
**Причина**: GitHub Pages не поддерживает SPA роутинг по умолчанию.
**Решение**: Файл `404.html` копирует `index.html` и Quartz обрабатывает роутинг на клиенте.

### Ссылки не работают на мобильных
**Причина**: Кэш браузера.
**Решение**: Очистить кэш или открыть в инкогнито.

### baseUrl с https:// ломает ссылки
**Причина**: Quartz добавляет `https://` к baseUrl, получается `https://https://...`
**Решение**: В `quartz.config.ts` указывать `baseUrl: "monon00ke.github.io/obsidian-vault"` (без протокола).

### Файлы из вложенных папок не открываются
**Причина**: `markdownLinkResolution: "shortest"` не работает с подпапками.
**Решение**: Использовать `"relative"`.

## Полезные команды

### Проверить сборку локально
```bash
npx quartz build --verbose
```

### Проверить конкретную страницу
```bash
curl -s -o /dev/null -w "%{http_code}" "https://monon00ke.github.io/obsidian-vault/PATH/TO/PAGE"
```

### Посмотреть contentIndex (все страницы)
```bash
curl -s "https://monon00ke.github.io/obsidian-vault/static/contentIndex.json" | python3 -m json.tool
```

### Проверить граф
```bash
curl -s "https://monon00ke.github.io/obsidian-vault/static/contentIndex.json" | python3 -c "
import json, sys
data = json.load(sys.stdin)
print(f'Pages: {len(data)}')
for slug in list(data.keys())[:10]:
    print(f'  - {slug}')
"
```

## Текущее состояние (на 2026-05-22)

### Работает:
- ✅ Главная страница с навигацией
- ✅ Все заметки рендерятся корректно
- ✅ Граф связей (мини-граф + глобальный)
- ✅ Поиск (Ctrl+K)
- ✅ Тёмная тема
- ✅ Backlinks
- ✅ Оглавление
- ✅ Вложенные папки (Speaking Topics и др.)
- ✅ Изображения
- ✅ LaTeX/KaTeX формулы
- ✅ SPA навигация без перезагрузки
- ✅ Мобильная версия

### Структура контента:
- **English**: Grammar (12 тем), IELTS (Writing, Speaking Topics 3-18), Vocabulary, Exam Prep
- **Math**: Math Analysis, Math Statistics (10+ домашних работ)
- **OS Labs**: 6 лабораторных (File Signature, sed/awk, Users, Filesystem, Script Validation, Processes)
- **Functional Analysis**: Probability Theory Notes
- **Scientific Work**: 4 лабораторные
- **Photos**: 15 изображений

## Что может понадобиться сделать

### Добавить новую заметку
1. Создать `.md` файл в нужной папке `content/`
2. Добавить front matter с `title`
3. Добавить ссылку в соответствующий MOC (Map of Content)
4. Добавить backlinks в связанные заметки
5. Закоммитить и запушить

### Изменить дизайн
- Цвета: `quartz.config.ts` → `theme.colors`
- Шрифты: `quartz.config.ts` → `theme.typography`
- Компоненты: `quartz.layout.ts`

### Добавить плагин
1. Найти плагин в `quartz/plugins/`
2. Добавить в `quartz.config.ts` → `plugins.transformers` или `plugins.emitters`
3. Пересобрать

### Исправить битые ссылки
```bash
# Найти все wikilinks
grep -r '\[\[' content/ --include='*.md'

# Проверить что файлы существуют
```

## Контакты и ссылки

- **Quartz docs**: https://quartz.jzhao.xyz/
- **Quartz GitHub**: https://github.com/jackyzha0/quartz
- **Quartz Discord**: https://discord.gg/cRFFHYye7t
- **GitHub Actions логи**: https://github.com/Monon00ke/obsidian-vault/actions

---
*Последнее обновление: 2026-05-22*
*Создано для AI-ассистента opencode*
