# Начало работы со Snova Pay

Добро пожаловать в документацию **Snova Pay** — современной платежной инфраструктуры для бизнеса, e-commerce и онлайн-сервисов.

:::callout{type="tip"}
Для начала работы вам понадобится учетная запись в личном кабинете [Snova Portal](https://portal.snova-pay.ru). Если у вас еще нет аккаунта, зарегистрируйтесь за 2 минуты.
:::

## Быстрый старт за 3 шага

:::steps
### Создайте проект и получите API-ключи

Перейдите в личный кабинет мерчанта в раздел **Настройки -> API Ключи**. Создайте ключ с нужными правами доступа.

* **Публичный ключ (`pk_...`)** — используется на фронтенде для инициализации платежного виджета.
* **Секретный ключ (`sk_...`)** — используется исключительно на вашем бэкенде для подписания запросов к REST API.

:::callout{type="warning"}
Никогда не публикуйте и не передавайте секретный ключ на сторону клиента или в публичные репозитории.
:::

### Выберите способ интеграции

В зависимости от архитектуры вашего проекта выберите наиболее удобный вариант подключения:

* **Платежный виджет**: Готовое всплывающее или встраиваемое окно оплаты без перенаправления пользователя со страницы.
* **REST API**: Полный контроль над процессом формирования счетов, рекуррентных списаний и выплат.
* **Готовые модули CMS**: Плагины для Tilda, InSales, 1C-Битрикс и других популярных платформ.

### Настройте вебхуки

Для автоматического обновления статусов заказов укажите URL обработчика в личном кабинете. Snova Pay будет отправлять HTTP POST запросы с криптографической подписью при каждом изменении статуса платежа.
:::

## Пример создания счета через API

:::tabs
```bash tab="cURL"
curl -X POST https://api.snova-pay.ru/v1/invoices \
  -H "Authorization: Bearer snova_sec_sample_key" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 150000,
    "currency": "RUB",
    "orderId": "order-10492",
    "description": "Оплата заказа #10492",
    "customer": {
      "email": "customer@example.com"
    }
  }'
```

```javascript tab="Node.js"
import { SnovaClient } from '@snova/sdk';

const snova = new SnovaClient({
  apiKey: process.env.SNOVA_SECRET_KEY,
});

const invoice = await snova.invoices.create({
  amount: 150000, // 1500.00 RUB в копейках
  currency: 'RUB',
  orderId: 'order-10492',
  description: 'Оплата заказа #10492',
  customer: {
    email: 'customer@example.com',
  },
});

console.log('Invoice payment URL:', invoice.paymentUrl);
```

```python tab="Python"
import requests

response = requests.post(
    "https://api.snova-pay.ru/v1/invoices",
    headers={
        "Authorization": "Bearer snova_sec_sample_key",
        "Content-Type": "application/json",
    },
    json={
        "amount": 150000,
        "currency": "RUB",
        "orderId": "order-10492",
        "description": "Оплата заказа #10492",
        "customer": {"email": "customer@example.com"},
    },
)

invoice = response.json()
print("Payment URL:", invoice["paymentUrl"])
```
:::

## Полезные ссылки

:::cardgrid
:::card{title="Платежный виджет" href="/widget-integration" icon="Layers"}
Инструкция по подключению JS-виджета на ваш сайт.
:::

:::card{title="Вебхуки и безопасность" href="/webhooks" icon="ShieldCheck"}
Проверка цифровой подписи HMAC SHA-256 и обработка уведомлений.
:::
:::
