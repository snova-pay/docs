# Вебхуки и уведомления

Вебхуки (Webhooks) — это механизм оповещения вашего сервера о событиях, происходящих в системе **Snova Pay** (успешная оплата, отмена счета, возврат средств).

:::callout{type="tip"}
Для настройки URL вебхуков перейдите в **Личный кабинет -> Настройки -> Вебхуки**.
:::

## Формат запроса

Когда статус счёта меняется, наш сервер отправляет `POST` запрос с телом в формате JSON на указанный вами URL:

```http
POST /api/snova-webhook HTTP/1.1
Host: your-shop.com
Content-Type: application/json
X-Snova-Signature: 8f9b3a...
X-Snova-Timestamp: 1730810400
```

### Пример полезной нагрузки (Payload)

```json
{
  "event": "invoice.paid",
  "data": {
    "id": "inv_9482049102",
    "orderId": "order-10492",
    "status": "PAID",
    "amount": 150000,
    "fee": 4500,
    "netAmount": 145500,
    "currency": "RUB",
    "paymentMethod": "SBP",
    "paidAt": "2026-11-05T15:20:00Z",
    "customer": {
      "email": "client@mail.ru",
      "phone": "+79991234567"
    }
  }
}
```

## Проверка подписи (HMAC-SHA256)

Каждый запрос подписывается вашим **Webhook Secret Key**. Это исключает подделку запросов третьими лицами.

:::tabs
```javascript tab="Node.js (TypeScript)"
import crypto from 'node:crypto';
import type { Request, Response } from 'express';

export function handleWebhook(req: Request, res: Response) {
  const signature = req.headers['x-snova-signature'] as string;
  const timestamp = req.headers['x-snova-timestamp'] as string;
  const secret = process.env.SNOVA_WEBHOOK_SECRET!;

  const payload = `${timestamp}.${JSON.stringify(req.body)}`;
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');

  if (signature !== expectedSignature) {
    return res.status(401).send('Invalid signature');
  }

  const { event, data } = req.body;

  if (event === 'invoice.paid') {
    console.log(`Заказ ${data.orderId} успешно оплачен!`);
    // Активируйте заказ в вашей базе данных
  }

  res.status(200).json({ received: true });
}
```

```python tab="Python (FastAPI)"
import hmac
import hashlib
from fastapi import FastAPI, Header, HTTPException, Request

app = FastAPI()
WEBHOOK_SECRET = "snova_wh_sample_secret"

@app.post("/webhook")
async def snova_webhook(
    request: Request,
    x_snova_signature: str = Header(...),
    x_snova_timestamp: str = Header(...)
):
    body = await request.body()
    data_to_sign = f"{x_snova_timestamp}.".encode() + body
    
    expected_sig = hmac.new(
        WEBHOOK_SECRET.encode(),
        data_to_sign,
        hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(x_snova_signature, expected_sig):
        raise HTTPException(status_code=401, detail="Invalid signature")

    payload = await request.json()
    if payload.get("event") == "invoice.paid":
        order_id = payload["data"]["orderId"]
        print(f"Order {order_id} paid successfully!")

    return {"received": True}
```
:::

## Ответ на вебхук

Ваш сервер должен ответить кодом **200 OK** в течение 5 секунд. Если ответ не получен или возвращен код ошибки 5xx, Snova Pay будет повторять отправку уведомления по экспоненциальной шкале (до 12 попыток в течение 24 часов).
