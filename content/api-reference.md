# REST API Reference

Интерфейс **Snova Pay REST API** спроектирован по стандарту HTTP REST с авторизацией через Bearer Tokens и ответами в формате JSON.

## Базовый URL

:::callout{type="info"}
Все запросы должны выполняться исключительно по протоколу HTTPS:
`https://api.snova-pay.ru/v1`
:::

## Аутентификация

Для аутентификации передавайте ваш секретный ключ API в заголовке `Authorization`:

```http
Authorization: Bearer snova_sec_...
```

## Эндпоинты

### Создание счёта (Create Invoice)

`POST /v1/invoices`

Создает новый платежный счёт и возвращает ссылку на оплату (`paymentUrl`) и уникальный `id`.

#### Тело запроса

| Поле | Тип | Обязательное | Описание |
| :--- | :--- | :--- | :--- |
| `amount` | `integer` | Да | Сумма в минимальных денежных единицах (копейках). Например, `10000` = 100.00 ₽ |
| `currency` | `string` | Да | Трехбуквенный код валюты ISO 4217 (`RUB`) |
| `orderId` | `string` | Да | Внутренний номер заказа в вашей системе (до 64 символов) |
| `description` | `string` | Нет | Назначение платежа, отображаемое плательщику |
| `customer` | `object` | Нет | Данные покупателя (`email`, `phone`, `name`) |
| `metadata` | `object` | Нет | Произвольный JSON-объект с вашими метаданными |
| `successUrl` | `string` | Нет | URL для возврата клиента при успешной оплате |
| `failUrl` | `string` | Нет | URL для возврата клиента при отмене оплаты |

#### Пример ответа

```json
{
  "id": "inv_01HY5T9A8X8R2W3Z",
  "status": "PENDING",
  "amount": 10000,
  "currency": "RUB",
  "orderId": "order_78912",
  "paymentUrl": "https://pay.snova-pay.ru/pay/inv_01HY5T9A8X8R2W3Z",
  "createdAt": "2026-11-05T18:00:00.000Z",
  "expiresAt": "2026-11-05T18:30:00.000Z"
}
```

---

### Получение информации о счёте (Get Invoice)

`GET /v1/invoices/:id`

Возвращает текущее состояние счета и историю транзакций.

#### Пример ответа

```json
{
  "id": "inv_01HY5T9A8X8R2W3Z",
  "status": "PAID",
  "amount": 10000,
  "currency": "RUB",
  "orderId": "order_78912",
  "paidAt": "2026-11-05T18:05:12.000Z",
  "paymentMethod": "SBP"
}
```

---

### Оформление возврата (Refund)

`POST /v1/invoices/:id/refund`

Инициирует полный или частичный возврат средств по оплаченному счёту.

#### Тело запроса

```json
{
  "amount": 5000,
  "reason": "Возврат по заявке клиента"
}
```
