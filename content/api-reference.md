# REST API

REST API SNOVA предназначен для **серверной** интеграции: модуля CMS, собственного бэкенда магазина или учётной системы. Через него вы синхронизируете каталог, управляете планами, оформляете корзину с подписками, получаете заказы, делаете возвраты и управляете подписками.

:::callout{type="info"}
Для обычной установки виджета API не нужен: виджет работает по публичному ключу. API нужен, если вы хотите создавать заказы в своей системе или строить оформление подписки на своей стороне.
:::

## Основы

### Базовый адрес

```text
https://api.snova-pay.ru
```

Все методы находятся под префиксом `/v1/api/`. Запросы и ответы — в JSON (`Content-Type: application/json`), только по HTTPS.

### Аутентификация

API использует **HTTP Basic**: логин — публичный ключ (`pk_…`), пароль — секретный ключ (`sk_…`). Ключ выпускается в [личном кабинете](https://dashboard.snova-pay.ru/) в разделе **API → API-ключи** с назначением **«Интеграция с CMS»**.

:::tabs
```bash tab="cURL"
curl https://api.snova-pay.ru/v1/api/selling-plans \
  -u "pk_your_public_key:sk_your_secret_key"
```

```javascript tab="Node.js"
const auth = Buffer.from(
  `${process.env.SNOVA_PUBLIC_KEY}:${process.env.SNOVA_SECRET_KEY}`,
).toString('base64');

const response = await fetch('https://api.snova-pay.ru/v1/api/selling-plans', {
  headers: { Authorization: `Basic ${auth}` },
});
const { plans } = await response.json();
```

```python tab="Python"
import os
import requests

response = requests.get(
    "https://api.snova-pay.ru/v1/api/selling-plans",
    auth=(os.environ["SNOVA_PUBLIC_KEY"], os.environ["SNOVA_SECRET_KEY"]),
)
plans = response.json()["plans"]
```
:::

:::callout{type="warning"}
Секретный ключ используется только на сервере. Не передавайте его в браузер или мобильное приложение.
:::

### Суммы и идентификаторы

* **Суммы — в рублях**, числом с не более чем двумя знаками после запятой: `1290`, `1290.5`, `1290.50`. Поля с суммами оканчиваются на `Rub`: `priceRub`, `totalRub`, `amountRub`. Валюта — только RUB.
* **Идентификаторы SNOVA** — UUID. Товары, варианты и планы можно адресовать **внешними ID** из вашей CMS (`externalId`, `externalVariantId`) — теми, что вы передали при синхронизации каталога.
* **Даты** — в формате ISO 8601: `2026-11-05T18:00:00.000Z`.

### Ошибки

Ошибка возвращается с HTTP-кодом и телом вида `{"error": "<код>"}`.

| HTTP | `error` | Что значит |
| :--- | :--- | :--- |
| 400 | — | Тело или параметры запроса не прошли проверку |
| 401 | `missing_credentials` | Нет заголовка `Authorization: Basic …` |
| 401 | `invalid_credentials` | Неверная пара ключей или ключ отозван |
| 403 | `insufficient_scope` | У ключа нет назначения «Интеграция с CMS» |
| 403 | `merchant_suspended` | Магазин приостановлен платформой |
| 404 | `not_found`, `order_not_found` | Объект не найден или принадлежит другому магазину |
| 422 | зависит от метода | Запрос корректен, но действие невозможно (см. описание метода) |
| 429 | — | Превышен лимит запросов |

### Лимиты

Общий лимит — 200 запросов в минуту. У методов, которые меняют данные, лимит строже: он указан в описании метода.

## Каталог

### Синхронизировать каталог

`POST /v1/api/catalog/sync` · лимит 10 запросов/мин

Создаёт или обновляет товары и варианты по внешнему ID. Повторная отправка того же товара обновляет его, а не создаёт дубль. До 500 товаров за запрос, до 200 вариантов у товара.

```json
{
  "products": [
    {
      "externalId": "coffee-ethiopia",
      "title": "Кофе Эфиопия Иргачефф",
      "variants": [
        { "externalId": "coffee-ethiopia-250", "sku": "ETH-250", "priceRub": 890, "inStock": true },
        { "externalId": "coffee-ethiopia-1000", "sku": "ETH-1000", "priceRub": 2990 }
      ]
    }
  ]
}
```

| Поле варианта | Тип | Обязательное | Описание |
| :--- | :--- | :--- | :--- |
| `externalId` | `string` | Да | ID варианта в вашей системе, до 200 символов |
| `sku` | `string` | Нет | Артикул |
| `priceRub` | `number` | Да | Цена разовой покупки, больше нуля |
| `inStock` | `boolean` | Нет | Есть ли в наличии, по умолчанию `true` |

Ответ `200`:

```json
{ "productsUpserted": 1, "variantsUpserted": 2 }
```

## Планы подписки

### Список планов

`GET /v1/api/selling-plans`

Возвращает планы магазина и порядок вариантов, заданный в кабинете в разделе «Установка виджета».

```json
{
  "plans": [
    {
      "id": "5f0c…",
      "externalId": "monthly",
      "name": "Каждый месяц",
      "intervalUnit": "month",
      "intervalCount": 1,
      "anchorDay": null,
      "anchorWeekday": null,
      "discountType": "percentage",
      "discountValue": 10,
      "minCycles": 1,
      "maxCycles": null,
      "allowedActions": ["pause", "skip", "swap", "reschedule"],
      "allProducts": true,
      "productExternalIds": [],
      "tiers": [],
      "archived": false,
      "firstChargeAt": "2026-12-06T00:00:00.000Z"
    }
  ],
  "optionOrder": ["one_time", "5f0c…"]
}
```

`firstChargeAt` — когда прошло бы следующее списание по подписке, оформленной сегодня. `optionOrder` — порядок вариантов в виджете: ID планов и `one_time` (разовая покупка).

### Создать или заменить план

`PUT /v1/api/selling-plans/:externalId` · лимит 60 запросов/мин

Полностью заменяет план с этим внешним ID или создаёт новый. Ответ: `201` — план создан, `200` — обновлён.

| Поле | Тип | Обязательное | Описание |
| :--- | :--- | :--- | :--- |
| `name` | `string` | Да | Название плана, до 200 символов |
| `intervalUnit` | `string` | Да | `day`, `week` или `month` |
| `intervalCount` | `integer` | Да | Каждые N единиц, от 1 до 365 |
| `discountType` | `string` | Да | `percentage` или `fixed` (рубли) |
| `discountValue` | `number` | Да | Размер скидки; `0` — без скидки, процент не больше 100 |
| `anchorDay` | `integer` | Нет | Число месяца для списания, 1–28 или `-1` (последний день). Только для `month` |
| `anchorWeekday` | `integer` | Нет | День недели, 1–7 (понедельник — 1). Только для `week` |
| `minCycles` | `integer` | Нет | Минимальное число циклов, по умолчанию 1. Не мешает покупателю отменить подписку: отмена доступна всегда |
| `maxCycles` | `integer` | Нет | Всего циклов; `null` — до отмены |
| `allowedActions` | `string[]` | Нет | Что покупатель может сам: `pause`, `skip`, `swap`, `reschedule`. По умолчанию все |
| `allProducts` | `boolean` | Нет | `true` (по умолчанию) — план для всех товаров |
| `productExternalIds` | `string[]` | Нет | Внешние ID товаров, если `allProducts: false` |
| `tiers` | `array` | Нет | Скидки по количеству: `{ "minQuantity": 3, "discountType": "percentage", "discountValue": 15 }` |

Ошибка `422 unknown_products` — в `productExternalIds` есть товары, которых нет в каталоге; их список приходит в поле `externalIds`. Сначала синхронизируйте каталог.

### Архивировать план

`DELETE /v1/api/selling-plans/:externalId`

План перестаёт предлагаться при новых оформлениях. Уже оформленные подписки продолжают списываться по нему.

## Оформление корзины

Эти методы нужны, если оформление заказа происходит на вашем сайте (так работает плагин WooCommerce), а не в виджете SNOVA.

### Создать сессию оплаты

`POST /v1/api/checkout/sessions` · лимит 60 запросов/мин

Оплачивает всю корзину одним платежом: разовые позиции и позиции на любом числе планов. Каждый план становится отдельной подпиской. В ответе приходит ссылка на платёжную страницу провайдера, куда нужно перенаправить покупателя.

```json
{
  "externalOrderId": "10492",
  "customerEmail": "customer@example.com",
  "customerName": "Анна Смирнова",
  "customerPhone": "+79991234567",
  "shippingAddress": { "line1": "ул. Ленина, 1, кв. 5", "city": "Москва", "postalCode": "101000", "country": "RU" },
  "lines": [
    { "externalVariantId": "coffee-ethiopia-250", "quantity": 2, "unitPriceRub": 801, "sellingPlanId": "5f0c…" },
    { "externalVariantId": "mug-white", "quantity": 1, "unitPriceRub": 590, "sellingPlanId": null }
  ],
  "shippingRub": 300,
  "recurringShipping": [{ "sellingPlanId": "5f0c…", "amountRub": 300 }],
  "consents": [
    { "type": "recurring_payments", "textVersion": "2026-11-01" },
    { "type": "personal_data", "textVersion": "2026-11-01" }
  ],
  "customerIp": "203.0.113.10",
  "successUrl": "https://shop.example.com/checkout/success/10492",
  "failUrl": "https://shop.example.com/checkout/failed/10492"
}
```

| Поле | Обязательное | Описание |
| :--- | :--- | :--- |
| `externalOrderId` | Да | Номер заказа в вашей системе |
| `customerEmail` | Да | Email покупателя: на него приходят письма о подписке и вход в портал |
| `customerName`, `customerPhone`, `shippingAddress` | Нет | Данные для доставки. `country` — двухбуквенный код (`RU`) |
| `lines[]` | Да | Позиции корзины, от 1 до 200 |
| `lines[].unitPriceRub` | Да | Итоговая цена за единицу с учётом ваших скидок и купонов. Списывается как есть |
| `lines[].recurringUnitPriceRub` | Нет | Цена за единицу в продлениях, если она отличается от сегодняшней (например, купон только на первый заказ) |
| `lines[].sellingPlanId` | Да | ID плана SNOVA или `null` для разовой позиции |
| `shippingRub` | Нет | Доставка в этом платеже |
| `recurringShipping[]` | Нет | Доставка в каждом продлении, отдельно для каждого плана |
| `consents[]` | Да* | Согласия покупателя с версией текста. \*Если в корзине есть подписка, нужны оба: `recurring_payments` и `personal_data` |
| `customerIp` | Нет | IP покупателя; сохраняется вместе с согласиями |
| `successUrl`, `failUrl` | Да | Куда вернуть покупателя после оплаты |

Ответ `201`:

```json
{
  "checkoutSessionId": "9b1e…",
  "confirmationUrl": "https://yoomoney.ru/checkout/payments/v2/contract?orderId=…",
  "subscriptionIds": ["c41a…"]
}
```

| Ошибка | HTTP | Что значит |
| :--- | :--- | :--- |
| `consents_required` | 422 | В корзине есть подписка, но нет обоих согласий |
| `invalid_plan` | 422 | План не найден или в архиве (`detail` — ID плана) |
| `invalid_variant` | 422 | Вариант не найден в каталоге (`detail` — его ID) |
| `plan_not_applicable` | 422 | План не применяется к этому товару |
| `nothing_to_pay` | 422 | Нечего оплачивать |
| `payment_account_not_connected` | 422 | В кабинете не подключён эквайринг |
| `checkout_failed` | 502 | Провайдер не создал платёж, повторите позже |

Если сегодня платить нечего (например, купон на 100%), а в корзине есть подписка, карта проверяется списанием 1 ₽ с немедленным возвратом.

Каждый вызов создаёт новую сессию. Оплатить можно только одну сессию заказа с данным `externalOrderId`: если покупатель оплатит вторую открытую платёжную страницу того же заказа, эта сессия завершится ошибкой `order_already_paid`.

:::callout{type="warning"}
Возврат покупателя на `successUrl` не доказывает оплату. Подтверждайте заказ по вебхуку `checkout.completed` или запросом статуса сессии.
:::

### Статус сессии оплаты

`GET /v1/api/checkout/sessions/:id`

```json
{
  "id": "9b1e…",
  "status": "succeeded",
  "errorCode": null,
  "externalOrderId": "10492",
  "orderId": "71d2…",
  "subscriptionIds": ["c41a…"],
  "verificationOnly": false
}
```

`status`: `pending`, `succeeded` или `failed`. `verificationOnly: true` — платёж был только проверкой карты на 1 ₽.

## Заказы

Заказ SNOVA создаётся при каждом успешном списании: при оформлении и при каждом продлении подписки.

Статусы заказа:

| Статус | Что значит |
| :--- | :--- |
| `pending` | Оплата ещё не прошла |
| `paid` | Оплачен, но ваша система ещё не подтвердила, что создала у себя заказ |
| `pushed` | Ваша система подтвердила заказ методом `POST /v1/api/orders/:id/external` |
| `cancelled` | Отменён |

### Список заказов

`GET /v1/api/orders`

| Параметр | Описание |
| :--- | :--- |
| `status` | Фильтр по статусу |
| `createdSince` | Заказы, созданные начиная с этого момента |
| `createdBefore` | …и строго раньше этого момента |
| `limit` | До 500, по умолчанию 100 |

Заказы возвращаются от старых к новым. Запрос `?status=paid&createdBefore=<10 минут назад>` — страховка от потерянных вебхуков: он показывает оплаченные заказы, которые ваша система так и не подтвердила.

```json
{
  "orders": [
    {
      "id": "71d2…",
      "status": "paid",
      "externalOrderId": null,
      "subscriptionId": "c41a…",
      "checkoutSessionId": null,
      "totalRub": 1902,
      "createdAt": "2026-12-06T06:00:12.000Z"
    }
  ]
}
```

### Детали заказа

`GET /v1/api/orders/:id`

Всё, что нужно для сборки и отправки: позиции, покупатель, адрес. Используйте этот метод, а не тело вебхука, как источник данных о заказе.

```json
{
  "id": "71d2…",
  "externalOrderId": null,
  "status": "paid",
  "totalRub": 1902,
  "currency": "RUB",
  "createdAt": "2026-12-06T06:00:12.000Z",
  "customerEmail": "customer@example.com",
  "customerName": "Анна Смирнова",
  "customerPhone": "+79991234567",
  "shippingAddress": { "line1": "ул. Ленина, 1, кв. 5", "line2": null, "city": "Москва", "postalCode": "101000", "country": "RU" },
  "items": [
    { "kind": "item", "externalVariantId": "coffee-ethiopia-250", "sku": "ETH-250", "title": "Кофе Эфиопия Иргачефф, 250 г", "quantity": 2, "unitPriceRub": 801, "subscriptionId": "c41a…" },
    { "kind": "shipping", "externalVariantId": null, "sku": null, "title": "Доставка", "quantity": 1, "unitPriceRub": 300, "subscriptionId": null }
  ],
  "subscriptionId": "c41a…",
  "checkoutSessionId": null,
  "shippingRub": 300
}
```

### Подтвердить заказ

`POST /v1/api/orders/:id/external` · лимит 30 запросов/мин

Сохраняет номер заказа в вашей системе и переводит заказ в статус `pushed`.

```json
{ "externalOrderId": "B-20431" }
```

Ответ `200`: `{ "ok": true }`.

### Возврат

`POST /v1/api/orders/:id/refunds` · лимит 20 запросов/мин

Полный или частичный возврат по заказу. Деньги возвращает ваш провайдер.

```json
{ "amountRub": 500, "reason": "Повреждена упаковка" }
```

| Поле | Обязательное | Описание |
| :--- | :--- | :--- |
| `amountRub` | Нет | Сумма возврата. Если не указана — возвращается весь остаток |
| `reason` | Да | Причина возврата |

Ответ `200`:

```json
{ "status": "succeeded", "refundId": "0e3f…" }
```

`status: "pending"` — провайдер принял возврат, но ещё не завершил его. Ошибки: `404 order_not_found`, `422 nothing_left_to_refund` (всё уже возвращено), `422 refund_failed` (провайдер отказал, код — в поле `code`).

## Подписки

### Список подписок

`GET /v1/api/subscriptions`

| Параметр | Описание |
| :--- | :--- |
| `status` | `active`, `paused`, `past_due`, `cancelled`, `expired` или `draft` |
| `customerEmail` | Подписки одного покупателя |
| `updatedSince` | Подписки, изменённые начиная с этого момента, от старых к новым. Удобно для периодической сверки |
| `limit` | До 500, по умолчанию 100 |

```json
{
  "subscriptions": [
    {
      "id": "c41a…",
      "customerEmail": "customer@example.com",
      "status": "active",
      "nextChargeAt": "2027-01-06T06:00:00.000Z",
      "cyclesCompleted": 1,
      "totalRub": 1902,
      "items": [{ "externalVariantId": "coffee-ethiopia-250", "quantity": 2 }],
      "sellingPlanId": "5f0c…",
      "sellingPlanExternalId": "monthly",
      "checkoutSessionId": null,
      "updatedAt": "2026-12-06T06:00:12.000Z"
    }
  ]
}
```

### Подписка

`GET /v1/api/subscriptions/:id`

Полные данные подписки: план, доступные покупателю действия, позиции, сумма следующего списания (`recurringTotalRub`), маска карты, дата паузы, история заказов и адрес доставки.

### Действия с подпиской

Лимит — 60 запросов/мин на действие. Успешный ответ — `200 { "ok": true }`.

| Метод | Тело | Действие |
| :--- | :--- | :--- |
| `POST /v1/api/subscriptions/:id/pause` | `{ "pausedUntil": "2027-02-01T00:00:00Z" }` | Пауза; `pausedUntil: null` — до ручного возобновления |
| `POST /v1/api/subscriptions/:id/resume` | `{}` | Возобновить |
| `POST /v1/api/subscriptions/:id/skip` | `{}` | Пропустить ближайшую поставку |
| `POST /v1/api/subscriptions/:id/reschedule` | `{ "nextChargeAt": "2026-12-20T06:00:00Z" }` | Перенести дату следующего списания |
| `POST /v1/api/subscriptions/:id/swap` | `{ "items": [{ "externalVariantId": "coffee-kenya-250", "quantity": 2 }] }` | Заменить состав подписки |
| `POST /v1/api/subscriptions/:id/cancel` | `{ "reason": "Переезд" }` | Отменить подписку |

Во всех действиях можно передать `initiatedBy`:

* `merchant` (по умолчанию) — действие администратора магазина, ограничения плана не применяются;
* `customer` — действие от имени покупателя, например из раздела «Мои подписки» на вашем сайте. Тогда действуют правила плана: покупатель может только те действия, которые разрешены в плане (`allowedActions`). Отмена покупателю доступна всегда.

В `swap` с `initiatedBy: "merchant"` можно задать цену позиции (`unitPriceRub`). Иначе цена считается по каталогу со скидкой плана.

Ошибки: `404 not_found`; `422 not_allowed` (действие запрещено планом или недоступно в текущем статусе), `422 invalid_date`, `422 invalid_variant`. Для `cancel` запрет возвращается как `409 not_allowed`.

### Смена карты

`POST /v1/api/subscriptions/:id/card-update` · лимит 30 запросов/мин

```json
{ "redirectUrl": "https://shop.example.com/account/subscriptions" }
```

Ответ `201`: `{ "confirmationUrl": "https://…" }`. Перенаправьте покупателя по этой ссылке: он привяжет новую карту на странице провайдера и вернётся на `redirectUrl`.

## Портал покупателя

### Ссылка для входа в портал

`POST /v1/api/portal/sessions` · лимит 60 запросов/мин

Если покупатель уже вошёл на вашем сайте, выдайте ему одноразовую ссылку в портал SNOVA, чтобы ему не пришлось входить ещё раз.

```json
{ "customerEmail": "customer@example.com" }
```

Ответ `201`:

```json
{ "url": "https://…/login/callback?token=…&from=shop", "expiresAt": "2026-12-06T06:15:00.000Z" }
```

Сразу перенаправьте покупателя по `url`. Ошибка `404 customer_not_found` — у покупателя с этим email ещё нет подписок в вашем магазине.

:::callout{type="warning"}
Вызывайте этот метод только для покупателя, чей email вы проверили сами (он вошёл в свой аккаунт на вашем сайте). Ссылка даёт полный доступ к его подпискам.
:::
