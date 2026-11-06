# Документация SNOVA

Исходники публичной документации **SNOVA** — платформы подписок для интернет-магазинов ([docs.snova-pay.ru](https://docs.snova-pay.ru)).

Источник правды о продукте — код монорепозитория `snova`: API-контракты (`packages/contracts`), маршруты (`apps/api/src/routes`), вебхуки (`packages/billing/src/outboundWebhooks.ts`), виджет (`apps/widget`) и модули интеграций (`integrations/`, `docs/*_INTEGRATION.md`). Меняя поведение продукта, обновляйте и соответствующую статью здесь.

---

## Структура репозитория

```
docs/
├── navigation.json            # Дерево навигации сайта (группы, пункты меню, порядок)
└── content/                   # Статьи документации (Markdown с директивами)
    ├── quickstart.md          # Начало работы: как устроена SNOVA, подключение магазина
    ├── widget-integration.md  # Виджет подписки: установка на любой сайт
    ├── api-reference.md       # REST API (/v1/api/*)
    ├── webhooks.md            # Вебхуки: события, подпись, повторы
    └── integrations/          # Готовые интеграции
        ├── tilda.md           # Tilda
        ├── insales.md         # inSales
        ├── bitrix.md          # 1С-Битрикс
        └── woocommerce.md     # WooCommerce
```

---

## Как добавить статью

1. Создайте Markdown-файл в папке `content/`, например `content/my-guide.md`.
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
3. Создайте Pull Request. После слияния в `main` сайт подхватит изменения без пересборки: он читает статьи прямо из GitHub. В браузере читателя статьи кэшируются на 10 минут, поэтому изменения появляются с задержкой до 10 минут.

Если GitHub недоступен, сайт показывает копию статей, встроенную в сборку (`apps/docs/src/content/fallback` в репозитории `snova`). После крупных изменений обновляйте и её.

---

## Поддерживаемые директивы

Директива начинается строкой `:::имя` и закрывается строкой `:::`. Внутри блоков кода не начинайте строки с `:::`, а внутри `:::steps` — с `###`: парсер воспримет их как разметку.

- **Выноски (callout)**:
```markdown
:::callout{type="tip" title="Совет"}
Полезная подсказка
:::
```
Типы: `tip`, `info`, `note`, `warning`, `danger`. `title` необязателен.

- **Вкладки с кодом (tabs)** — у каждого блока кода укажите язык и `tab="Название"`:
````markdown
:::tabs
```bash tab="cURL"
curl https://api.snova-pay.ru/v1/api/selling-plans -u "pk_…:sk_…"
```
```python tab="Python"
requests.get(url, auth=(public_key, secret_key))
```
:::
````

- **Пошаговые инструкции (steps)** — каждый шаг начинается с заголовка `###`:
```markdown
:::steps
### Создайте аккаунт
Инструкция…

### Получите ключи
Инструкция…
:::
```

- **Сетка карточек-ссылок (cardgrid)** — иконки: `Layers`, `ShieldCheck`, `Code`, `Blocks`, `Zap`:
```markdown
:::cardgrid
:::card{title="Вебхуки" href="/webhooks" icon="ShieldCheck"}
Описание карточки.
:::
:::
```
