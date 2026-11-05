# Snova Pay Documentation & Knowledge Base

Репозиторий базы знаний и публичной документации сервиса **Snova Pay** ([docs.snova-pay.ru](https://docs.snova-pay.ru)).

---

## Структура репозитория

```
docs/
├── navigation.json         # Дерево навигации сайта (группы, пункты меню, порядок)
└── content/                # Markdown / MDX статьи документации
    ├── quickstart.md       # Начало работы
    ├── widget-integration.md # Интеграция платежного виджета
    ├── webhooks.md         # Настройка и проверка подписи вебхуков
    ├── api-reference.md    # Спецификация REST API
    └── integrations/       # Модули для CMS
        ├── tilda.md        # Tilda Publishing
        ├── insales.md      # InSales
        └── bitrix.md       # 1С-Битрикс
```

---

## Как добавить новую статью

1. Создайте Markdown-файл в папке `content/` (например, `content/my-guide.md`).
2. Добавьте его в `navigation.json` в нужную группу:
```json
{
  "id": "my-guide",
  "title": "Моё руководство",
  "path": "my-guide",
  "file": "my-guide.md",
  "description": "Краткое описание для поиска и шапки статьи"
}
```
3. Создайте Pull Request. После слияния в `main` сайт [docs.snova-pay.ru](https://docs.snova-pay.ru) обновится автоматически без необходимости передеплоя Docker-образа.

---

## Поддерживаемые MDX-компоненты

- **Коллауты (Callouts)**:
```markdown
:::callout{type="tip" title="Совет"}
Полезная подсказка
:::
```
Типы: `tip`, `info`, `note`, `warning`, `danger`.

- **Табы кода (Tabs)**:
````markdown
:::tabs
```bash tab="cURL"
curl https://api.snova-pay.ru/v1/invoices ...
```
```javascript tab="Node.js"
const snova = new SnovaClient(...);
```
:::
````

- **Пошаговые руководства (Steps)**:
```markdown
:::steps
### Шаг 1: Создайте аккаунт
Инструкция...

### Шаг 2: Получите ключи
Инструкция...
:::
```
