# Вебхуки

Вебхуки сообщают вашему серверу о событиях в SNOVA: оплатах, возвратах и изменениях подписок. По ним удобно создавать заказы на сборку в своей системе, отражать возвраты и следить за статусами подписок.

## Настройка

:::steps
### Укажите адрес обработчика

В [личном кабинете](https://dashboard.snova-pay.ru/) откройте **API → Уведомления на ваш сервер** и введите адрес, начинающийся с `https://`. Адрес должен быть доступен из интернета: на `http://`, внутренние IP-адреса и адреса с перенаправлением (3xx) вебхуки не доставляются.

### Сгенерируйте секрет

Нажмите **«Сгенерировать секрет»** и сохраните его в настройках своего сервера. Этим секретом подписан каждый вебхук. Без секрета события не отправляются.

### Проверьте доставку

В блоке **Журнал доставок** видно каждое событие, число попыток и последнюю ошибку. Любое событие можно отправить ещё раз кнопкой в строке журнала.
:::

:::callout{type="info"}
Если вы перегенерируете секрет, старый сразу перестаёт действовать. Сначала обновите секрет на своём сервере.
:::

## Формат запроса

SNOVA отправляет `POST` с JSON-телом:

```http
POST /snova/webhook HTTP/1.1
Host: shop.example.com
Content-Type: application/json
X-Snova-Signature: 5d41402abc4b2a76b9719d911017c592...
X-Snova-Timestamp: 1765000812
X-Snova-Event-Id: 3f2b8c1e-7a4d-4e3b-9c1a-2d5e6f7a8b9c

{"type":"order.paid","data":{"orderId":"71d2…","subscriptionId":"c41a…","checkoutSessionId":null,"externalOrderId":null,"total":"1902.00"}}
```

| Заголовок | Описание |
| :--- | :--- |
| `X-Snova-Signature` | Подпись HMAC-SHA256 в hex (см. ниже) |
| `X-Snova-Timestamp` | Время отправки, Unix-время в секундах |
| `X-Snova-Event-Id` | ID события. Одинаков во всех повторных попытках, по нему отбрасывайте дубли |

Тело всегда имеет вид `{"type": "<событие>", "data": {...}}`.

## События

### Заказы и возвраты

| Событие | Когда | Поля `data` |
| :--- | :--- | :--- |
| `order.created` | Создан заказ: при оформлении или продлении подписки | `orderId`, `subscriptionId`, `checkoutSessionId`, `externalOrderId` |
| `order.paid` | Заказ оплачен | те же + `total` |
| `order.refunded` | Проведён полный или частичный возврат | те же + `amount` (сумма этого возврата) |

Суммы `total` и `amount` передаются строкой в рублях: `"1902.00"`.

`subscriptionId` равен `null` у первого заказа оформления корзины, если в нём несколько подписок; тогда заказ связан с корзиной через `checkoutSessionId`. `externalOrderId` — номер заказа в вашей системе, если он известен SNOVA.

### Подписки

Во всех событиях есть `subscriptionId`.

| Событие | Когда | Дополнительные поля |
| :--- | :--- | :--- |
| `subscription.created` | Подписка оформлена и первая поставка оплачена | `checkoutSessionId` — при оформлении корзины через API |
| `subscription.paused` | Поставлена на паузу | `reason: "out_of_stock"` и `item` — если паузу поставила SNOVA из-за отсутствия товара |
| `subscription.resumed` | Возобновлена | — |
| `subscription.skipped` | Пропущена поставка | `nextChargeAt`; `reason: "out_of_stock"` и `item` — если товара не было в наличии |
| `subscription.rescheduled` | Перенесена дата списания | `nextChargeAt` |
| `subscription.items_changed` | Изменён состав подписки | — |
| `subscription.payment_method_updated` | Покупатель привязал новую карту | — |
| `subscription.past_due` | Не прошло списание, начались повторные попытки | — |
| `subscription.cancelled` | Подписка отменена покупателем, магазином или после неудачных повторных попыток списания | — |
| `subscription.expired` | Пройдено максимальное число циклов плана | — |

### Оформление корзины через API

Приходят, если вы оформляете оплату методом [`POST /v1/api/checkout/sessions`](/api-reference).

| Событие | Поля `data` |
| :--- | :--- |
| `checkout.completed` | `checkoutSessionId`, `externalOrderId`, `orderId`, `subscriptionIds` |
| `checkout.failed` | `checkoutSessionId`, `externalOrderId`, `code` — причина ошибки |

:::callout{type="tip"}
Тело вебхука содержит только идентификаторы. Чтобы получить позиции, покупателя и адрес доставки, запросите заказ методом [`GET /v1/api/orders/:id`](/api-reference). Затем подтвердите его методом `POST /v1/api/orders/:id/external`: тогда заказ не попадёт в список неподтверждённых.
:::

## Проверка подписи

Подпись — это HMAC-SHA256 в hex от строки `<X-Snova-Timestamp>.<тело запроса>`, где ключ — ваш секрет вебхуков.

Как проверять:

1. Считайте тело запроса **как есть, байт в байт**. Не разбирайте JSON и не сериализуйте его заново: подпись перестанет совпадать.
2. Посчитайте HMAC и сравните его с `X-Snova-Signature` сравнением с постоянным временем.
3. Отклоняйте запросы, у которых `X-Snova-Timestamp` старше 5 минут: это защищает от повторной отправки перехваченного запроса.
4. Отбрасывайте события с уже обработанным `X-Snova-Event-Id`.

:::tabs
```javascript tab="Node.js (Express)"
import crypto from 'node:crypto';
import express from 'express';

const app = express();
const SECRET = process.env.SNOVA_WEBHOOK_SECRET;

// Тело нужно в исходном виде: express.raw, а не express.json
app.post('/snova/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.get('X-Snova-Signature') ?? '';
  const timestamp = req.get('X-Snova-Timestamp') ?? '';

  const expected = crypto
    .createHmac('sha256', SECRET)
    .update(`${timestamp}.`)
    .update(req.body)
    .digest('hex');

  const valid =
    signature.length === expected.length &&
    crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
  const fresh = Math.abs(Date.now() / 1000 - Number(timestamp)) < 300;

  if (!valid || !fresh) {
    return res.status(401).send('Invalid signature');
  }

  const { type, data } = JSON.parse(req.body.toString('utf8'));

  if (type === 'order.paid') {
    // Поставьте в очередь: GET /v1/api/orders/:id, создание заказа у себя,
    // затем POST /v1/api/orders/:id/external
    console.log(`Оплачен заказ ${data.orderId} на ${data.total} ₽`);
  }

  res.sendStatus(200);
});
```

```python tab="Python (FastAPI)"
import hashlib
import hmac
import json
import os
import time

from fastapi import FastAPI, Header, HTTPException, Request

app = FastAPI()
SECRET = os.environ["SNOVA_WEBHOOK_SECRET"].encode()


@app.post("/snova/webhook")
async def snova_webhook(
    request: Request,
    x_snova_signature: str = Header(...),
    x_snova_timestamp: str = Header(...),
):
    body = await request.body()
    expected = hmac.new(
        SECRET, f"{x_snova_timestamp}.".encode() + body, hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(x_snova_signature, expected):
        raise HTTPException(status_code=401, detail="Invalid signature")
    if abs(time.time() - int(x_snova_timestamp)) > 300:
        raise HTTPException(status_code=401, detail="Stale timestamp")

    event = json.loads(body)
    if event["type"] == "order.paid":
        print(f"Оплачен заказ {event['data']['orderId']}")

    return {"ok": True}
```

```php tab="PHP"
<?php
$secret = getenv('SNOVA_WEBHOOK_SECRET');
$body = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_SNOVA_SIGNATURE'] ?? '';
$timestamp = $_SERVER['HTTP_X_SNOVA_TIMESTAMP'] ?? '';

$expected = hash_hmac('sha256', $timestamp . '.' . $body, $secret);

if (!hash_equals($expected, $signature) || abs(time() - (int) $timestamp) > 300) {
    http_response_code(401);
    exit('Invalid signature');
}

$event = json_decode($body, true);
if ($event['type'] === 'order.paid') {
    // Создайте заказ по данным GET /v1/api/orders/{orderId}
}

http_response_code(200);
echo '{"ok":true}';
```
:::

## Ответ и повторные попытки

Ответьте любым кодом **2xx** в течение **10 секунд**. Тяжёлую обработку (создание заказа, запросы к API) делайте асинхронно, после ответа.

Если сервер не ответил за 10 секунд, был недоступен или вернул любой другой код (3xx, 4xx, 5xx), SNOVA повторит отправку:

| Попытка | Через сколько после предыдущей |
| :--- | :--- |
| 1 | сразу после события |
| 2 | 1 минута |
| 3 | 5 минут |
| 4 | 15 минут |
| 5 | 1 час |
| 6 | 4 часа |

После 6 неудачных попыток (примерно через 5,5 часа) событие получает статус «Не доставлен» в журнале доставок, и его можно отправить ещё раз вручную. После третьей неудачной попытки SNOVA присылает пользователям личного кабинета письмо о том, что вебхуки не доставляются.

:::callout{type="warning"}
Порядок доставки не гарантирован: повторная попытка старого события может прийти позже нового. Не полагайтесь на порядок событий. Чтобы узнать актуальное состояние подписки, запросите её через [API](/api-reference).
:::

Если ваш сервер был недоступен дольше, чем длятся повторы, используйте запрос `GET /v1/api/orders?status=paid`: он покажет оплаченные заказы, которые вы ещё не подтвердили.
