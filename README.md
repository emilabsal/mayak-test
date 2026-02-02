# Backend тестовое задание  
**Webhook платежа и подписка (TypeScript, псевдокод)**

## Общее понимание задачи

Webhook от платежного провайдера — недоверенный источник. Он может приходить несколько раз, в неправильном порядке, с задержкой или с частично пустыми полями. Поэтому вся логика строится вокруг идемпотентности и возможности безопасно повторить обработку.

Ключевая идея решения:  
**сначала сохраняем webhook как факт**, затем **в транзакции** пытаемся применить бизнес-логику. Если что-то упало — провайдер пришлет webhook еще раз, и система корректно продолжит.

---

## Часть 1. Схема данных

### users

```ts
users {
  id: UUID
  email: string | null
  external_customer_id: string | null
  created_at: timestamp
}
```

Email и `external_customer_id` уникальны, но только если они есть.

---

### subscriptions

```ts
subscriptions {
  id: UUID
  user_id: UUID
  plan_id: string
  status: 'inactive' | 'active' | 'expired' | 'canceled'
  current_period_start: timestamp
  current_period_end: timestamp
  created_at: timestamp
  updated_at: timestamp
}
```

---

### payments

```ts
payments {
  id: UUID
  external_payment_id: string
  user_id: UUID | null
  subscription_id: UUID | null
  amount: number
  currency: string
  status: 'pending' | 'succeeded' | 'failed'
  paid_at: timestamp | null
  created_at: timestamp
}
```

---

### webhook_events

```ts
webhook_events {
  id: UUID
  provider: string
  external_event_id: string
  event_type: string
  payload: json
  status: 'received' | 'processed' | 'failed'
  error: string | null
  created_at: timestamp
  processed_at: timestamp | null
}
```

---

## Часть 2. Логика обработчика webhook

```ts
async function handleWebhook(req) {
  if (!verifySignature(req.rawBody, req.headers)) {
    return 401
  }

  const event = parse(req.rawBody)

  if (!event.externalEventId || !event.externalPaymentId) {
    return 400
  }

  const webhook = await saveOrGetWebhook(event)

  if (webhook.status === 'processed') {
    return 200
  }

  try {
    await db.transaction(async () => {
      const payment = await findOrCreatePayment(
        event.externalPaymentId,
        event.amount,
        event.currency
      )

      if (payment.status === 'succeeded') return

      const user = await findUser(event.externalCustomerId)
      if (!user) throw new RetryableError('user not found')

      if (!isValidAmount(event.amount, event.planId)) {
        await markPaymentFailed(payment)
        throw new NonRetryableError('invalid amount')
      }

      const subscription = await upsertSubscription(user, event.planId)

      await markPaymentSucceeded(payment, user, subscription)
      await markWebhookProcessed(webhook)
    })

    return 200
  } catch (e) {
    return e instanceof NonRetryableError ? 400 : 500
  }
}
```

---

## Часть 3. Edge cases

- Повторный webhook не создает дублей  
- Webhook до создания пользователя → retry  
- Нет email → используем external id  
- Неверная сумма → payment failed  
- Задержанный webhook → продление от current_period_end  
- Падение сервера → восстановление по pending payment  

---

## Часть 4. Логи и наблюдаемость

Логируются:
- external_event_id
- external_payment_id
- payment_id
- user_id
- subscription_id
- статус обработки

Payload webhook хранится полностью.

---

## Итог

Решение идемпотентно, устойчиво к падениям и подходит для продакшена.
