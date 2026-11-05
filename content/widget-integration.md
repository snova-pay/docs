# Интеграция платежного виджета

Платежный виджет **Snova Pay** позволяет принимать оплату банковскими картами, СБП и электронными кошельками прямо на вашем сайте без редиректа на внешние платежные шлюзы.

:::callout{type="info"}
Виджет оптимизирован для мобильных устройств, поддерживает Apple Pay / Google Pay / Tinkoff Pay и автоматически адаптируется под размер экрана покупателя.
:::

## Подключение скрипта

Добавьте скрипт виджета перед закрывающим тегом `</body>` на страницах вашего интернет-магазина:

```html
<script src="https://widget.snova-pay.ru/v1/widget.js" async></script>
```

## Инициализация и открытие формы

Для открытия виджета вызовите метод `SnovaPay.open()` с параметрами сформированного счёта:

:::tabs
```javascript tab="JavaScript (ES6)"
const snova = new window.SnovaPay({
  publicKey: 'pk_live_your_public_key',
});

document.getElementById('pay-button').addEventListener('click', () => {
  snova.open({
    invoiceId: 'inv_849204829',
    amount: 250000,
    currency: 'RUB',
    theme: 'dark', // 'light' | 'dark' | 'auto'
    onSuccess: (payment) => {
      console.log('Платеж успешно завершен!', payment);
      window.location.href = '/checkout/success';
    },
    onClose: () => {
      console.log('Пользователь закрыл виджет оплаты');
    },
    onError: (error) => {
      console.error('Ошибка при проведении платежа:', error);
      alert('Произошла ошибка при оплате. Попробуйте снова.');
    }
  });
});
```

```typescript tab="React"
import React from 'react';

export function CheckoutButton({ invoiceId, amount }: { invoiceId: string; amount: number }) {
  const handlePay = () => {
    if (!window.SnovaPay) return;

    window.SnovaPay.open({
      publicKey: 'pk_live_your_public_key',
      invoiceId,
      amount,
      currency: 'RUB',
      onSuccess: () => {
        window.location.href = '/success';
      },
    });
  };

  return (
    <button 
      onClick={handlePay}
      className="px-6 py-3 bg-violet-600 text-white font-medium rounded-lg hover:bg-violet-700 transition"
    >
      Оплатить {amount / 100} ₽
    </button>
  );
}
```
:::

## Параметры конфигурации

| Параметр | Тип | Обязательный | Описание |
| :--- | :--- | :--- | :--- |
| `publicKey` | `string` | Да | Публичный ключ вашего проекта |
| `invoiceId` | `string` | Да | Идентификатор счёта, созданного через backend API |
| `amount` | `number` | Да | Сумма к оплате в минимальных единицах валюты (копейки) |
| `currency` | `string` | Да | Валюта (`RUB`) |
| `theme` | `string` | Нет | Тема оформления: `light`, `dark` или `auto` (по умолчанию `auto`) |
| `onSuccess` | `function` | Нет | Callback-функция при успешной транзакции |
| `onClose` | `function` | Нет | Callback-функция при закрытии модального окна пользователем |
| `onError` | `function` | Нет | Callback-функция при ошибке проведения платежа |

:::callout{type="warning"}
Не полагайтесь исключительно на callback `onSuccess` на клиенте для выдачи цифровых товаров или активации подписок. Всегда подтверждайте оплату на бэкенде через [Вебхуки](/webhooks).
:::
