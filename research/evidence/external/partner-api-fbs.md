# Evidence: Partner API — модель FBS

Источник: [Список методов FBS](https://yandex.ru/dev/market/partner-api/doc/ru/overview/fbs)  
Смежные: [Обработка FBS-заказов](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/fbs), [Получение заказов](https://yandex.ru/dev/market/partner-api/doc/ru/step-by-step/orders-receive)  
Снято: `2026-08-28`  
Тип: `external` · не код RULI

## Модель

Товар на складе продавца, доставку делает Маркет. Продавец собирает и упаковывает заказ, затем отвозит на склад Маркета или отдаёт в машину Маркета.

База API: `https://api.partner.market.yandex.ru`. Методы в обзоре — **v2**, кроме получения списка заказов (рекомендуется **POST v1/businesses/{businessId}/orders**). Auth: Api-Key со скоупом `inventory-and-order-processing` (заказы/ярлыки/короба) или `all-methods`.

Идентификатор магазина в API — **campaignId** (не путать с id магазина в кабинете). Кабинет — **businessId**.

## Каталог методов (обзор FBS)

Группы на странице overview:

| Группа | Зачем |
|--------|--------|
| Кабинеты и магазины | `GET …/campaigns/{id}`, `GET …/settings` |
| Товары (ассортимент магазина) | update / list / delete offers; hidden-offers |
| Остатки | `POST …/offers/stocks` (чтение), `PUT …/offers/stocks` (передача) |
| Цены | `POST …/offer-prices/updates`, просмотр, карантин цены, калькулятор тарифов |
| Заказы | короба, external-id, смена статуса (один/пачка), проверка КИЗ, документы юрлица |
| Отгрузки FBS | first-mile: список/одна поставка, confirm, акт, перенос заказов, паллеты, лист сборки, ТТН, акт расхождений |
| Ярлыки | PDF на заказ / на коробку / массовый отчёт; данные для своей печати; ярлыки паллет |
| Невыкупы и возвраты | список, карточка, заявление, фото, решение |
| Отчёты | аналитика, заказы, товары, реализация, платежи |
| Индекс качества | детали рейтинга |
| Справочники | службы доставки |

На этой странице **нет** методов заведения карточки (`offer-mappings` / `offer-cards`) — это другой контур (кабинет/бизнес-каталог), не «обработка FBS».

## Конвейер заказа (step-by-step)

```text
1. Получить заказ
   уведомление POST /notification  ИЛИ  опрос POST v1/businesses/{businessId}/orders
   FBS poll: не реже 1 раза в час (рекомендуют чаще)

2. (опц.) Лист сборки
   POST v2/reports/documents/shipment-list/generate → poll GET v2/reports/info/{reportId}

3. Короба + маркировка
   PUT v2/campaigns/{campaignId}/orders/{orderId}/boxes
   можно повторять до READY_TO_SHIP
   Честный знак для ФЛ необязателен; для юрлица и ювелирки — обязателен до READY_TO_SHIP

4. Ярлыки + «Готов к отгрузке»
   (опц.) POST …/orders/{orderId}/external-id   # id Shop на ярлыке
   GET  …/orders/{orderId}/delivery/labels      # PDF, default A7 (75×120 мм)
   PUT  …/orders/{orderId}/status
        status=PROCESSING, substatus=READY_TO_SHIP
   После этого заказ попадает в акт приёма-передачи.

5. Отгрузка (first-mile)
   Все заказы поставки должны быть READY_TO_SHIP, иначе акт не отдадут.
   GET …/shipments/reception-transfer-act  — акт + подтверждение отгрузки (шаг API, в кабинете его нет)
   Доверительная приёмка: PUT …/pallets + GET …/pallet/labels
```

## Статусы, которые продавец ставит сам (FBS)

Из `PUT …/orders/{orderId}/status`:

| Переход | Когда |
|---------|--------|
| `PROCESSING` + `STARTED` | подтвердить черновик (если кабинет так настроен) |
| `PROCESSING` + `READY_TO_SHIP` | собран, будет отгружен; только из `STARTED` |
| `CANCELLED` + `SHOP_FAILED` | не можем выполнить; можно из `STARTED` и из `READY_TO_SHIP` |

Дальше (`SHIPPED`, `DELIVERY`, …) двигает Маркет. `PACKAGING` в enum есть, в step-by-step продавец его не выставляет.

Отмена / вырезание позиции / перенос в следующую поставку **бьют индекс качества**.

## Короба (`PUT …/boxes`)

Три операции одним телом: раскладка, КИЗ, удаление позиции (`allowRemove: true`).  
Нельзя увеличить заказ. Нельзя выкинуть акционный / единственный / ~99% стоимости товар — тогда только полная отмена `SHOP_FAILED`.  
После смены раскладки уже наклеенные ярлыки нужно печатать заново.

## Ярлыки

`GET …/delivery/labels` → `application/pdf`. Query `format`: `A7` (по умолчанию, 75×120), `A9`, `A9_HORIZONTALLY` (58×40), `A4`.  
Массово: `POST v2/reports/documents/labels/generate` + `GET v2/reports/info/{id}`.  
Лимит одиночных ярлыков: 10 000 запросов/час. Скоуп: `inventory-and-order-processing`.

## Получение заказов (не в overview, но шаг 1 FBS)

Рекомендуют **уведомления**. Fallback: `POST v1/businesses/{businessId}/orders` (не `GET v2/campaigns/.../orders`).  
Фильтры: даты доставки, `updatedAtFrom`/`updatedAtTo`, `orderIds`. Пагинация `pageToken`, `limit` ≤ 50.
