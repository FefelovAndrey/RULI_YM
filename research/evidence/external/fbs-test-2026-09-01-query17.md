# Лог прогона FBS — заказ 61089913793

Источник: `shopYmApi` dump, профиль 17 (`query.17.log` с `test.ruli.ru`).  
Снято: копия `Downloads/query.17.log`. В git — **только этот заказ**, без PDF-байтов, без адреса/GPS/buyer, без синтетики `334683501`.

Прогон: [fbs-test-2026-09-01.md](../../briefs/fbs-test-2026-09-01.md).  
Время в заголовке дампа — стенд (UTC+5). `updatedAt` в теле Маркета — UTC+3.

---

## Что не клали

| Исключено | Почему |
|-----------|--------|
| GET `…/orders/334683501.json` + `NOT_FOUND` | другой заказ, не этот прогон |
| Полный dump GET order (`delivery.region`, `street`, `gps`, `buyer`) | PII |
| Тело PDF после `labels.json` | бинарь; факт: ответ начинается с `%PDF-1.5` |
| onesignal / debug / atol / kafka / fast_stock чужих заказов | не Partner API этого теста |
| Токены OAuth / auth-token | секреты |

Повторные GET того же статуса при открытии карточки сжаты.

---

## Ключевые вызовы (этот заказ)

| Время стенда | Метод | Результат |
|--------------|-------|-----------|
| 2026-08-31 16:04:13 | GET `…/orders/61089913793.json` | `NULL`, затем `PROCESSING` / `STARTED`, `fake: true` |
| 2026-09-01 09:10–09:52 | GET тот же URL | без сдвига статуса (карточка) |
| **09:52:51** | PUT `…/orders/61089913793/status.json` | echo `READY_TO_SHIP`; GET `updatedAt` 01-09-2026 07:52:51 |
| 09:52:51–12:35:26 | GET order | держится `READY_TO_SHIP` |
| **12:07:36** | GET `…/delivery/labels.json` | `NULL`, затем `%PDF-1.5` |
| **12:30:51** | GET labels | то же |
| **12:33:48** | GET labels | то же |
| **12:35:57** | PUT `…/status.json` | echo `SHIPPED`; GET `updatedAt` 01-09-2026 10:35:57 |
| 12:36:00–12:40:08 | GET order | `SHIPPED` |
| **12:39:23** | PUT `…/status.json` | снова `SHIPPED` (идемпотентно) |

Каждый URL в логе сразу сопровождается вторым dump `NULL` — пустое тело, не ошибка HTTP. Третий dump — ответ Маркета.

---

## Фрагменты (как в логе)

Dump из `shopYmApi.class.php` line 39. IP клиента в заголовке строк опущен.

### GET после accept — 2026-08-31 16:04:13

```text
'https://api.partner.market.yandex.ru/v2/campaigns/137514772/orders/61089913793.json'

NULL

[
  'order' => [
    'id' => 61089913793,
    'externalOrderId' => '61089913793',
    'status' => 'PROCESSING',
    'substatus' => 'STARTED',
    'creationDate' => '31-08-2026 13:39:07',
    'updatedAt' => '31-08-2026 13:39:52',
    'currency' => 'RUR',
    'itemsTotal' => 833.0,
    'paymentType' => 'PREPAID',
    'paymentMethod' => 'YANDEX',
    'fake' => true,
    'items' => [
      [
        'id' => 1185030360,
        'offerId' => '1107932',
        'shopSku' => '1107932',
        'count' => 1,
        'vat' => 'VAT_22',
      ],
    ],
    // delivery.region / street / gps / buyer — вырезано
  ],
]
```

### PUT READY_TO_SHIP — 2026-09-01 09:52:51

```text
'https://api.partner.market.yandex.ru/v2/campaigns/137514772/orders/61089913793/status.json'

[
  'order' => [
    'status' => 'PROCESSING',
    'substatus' => 'READY_TO_SHIP',
  ],
]
```

Следом полный GET того же id: `substatus => READY_TO_SHIP`, `updatedAt => 01-09-2026 07:52:51`, `fake => true`.

### GET labels — 2026-09-01 12:07:36 (и 12:30:51, 12:33:48)

```text
'https://api.partner.market.yandex.ru/v2/campaigns/137514772/orders/61089913793/delivery/labels.json'

NULL

'%PDF-1.5
… <binary omitted>
```

### PUT SHIPPED — 2026-09-01 12:35:57

```text
'https://api.partner.market.yandex.ru/v2/campaigns/137514772/orders/61089913793/status.json'

[
  'order' => [
    'status' => 'PROCESSING',
    'substatus' => 'SHIPPED',
  ],
]
```

Следом GET: `substatus => SHIPPED`, `updatedAt => 01-09-2026 10:35:57`.

### Повтор PUT SHIPPED — 2026-09-01 12:39:23

Тот же URL и тот же echo `SHIPPED`. Последующие GET до 12:40:08 без смены `updatedAt`.
