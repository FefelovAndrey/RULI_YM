# AS IS: FBS-исполнение в Webasyst (контур RULI)

Канон: **Webasyst Shop + 1С + Partner API**. Не YMWB.

Снято прогоном **2026-09-01** на sandbox-заказе Маркета `61089913793` / Shop `2534310`.  
Бриф теста и чеклист доработок: [fbs-test-2026-09-01.md](../briefs/fbs-test-2026-09-01.md).  
Якоря кода: [wa-ym-fbs-pack.md](../evidence/code/wa-ym-fbs-pack.md).

---

## Цепочка (факт стенда)

```text
Маркет (sandbox, fake:true)
  → POST /ym_api/{profile}/?action=/order/accept   # сегодня: Postman, тело fake:false
  → Shop заказ (processing / in_stock)
  → действие «На упаковку» (только из in_stock)
  → clabels Сборка: сканер номера Shop
  → PUT …/orders/{id}/status.json  PROCESSING + READY_TO_SHIP
  → Shop «Отправлен» (shipped)
  → GET …/delivery/labels.json     PDF ярлыка
  → PUT …/orders/{id}/status.json  SHIPPED
  → [не пройдено] rulimpsupplies first-mile (акт / слот)
```

На боевом контуре приёмка должна идти callback'ом Маркета, не Postman. Тело `"fake": false` на sandbox **не** переводит заказ в боевой кабинет.

---

## Где что нажимается

| Шаг | UI | Технический смысл |
|-----|----|-------------------|
| Список заказов | Shop, статус `in_stock` | Единственный переход на `na-upakovku` |
| Сборка | `?plugin=clabels&action=check` | Скан **Shop**-номера; камера = складское рабочее место |
| READY_TO_SHIP | уходит из clabels при сборке | `shopClabelsPluginBackendMarketOrderShipping` |
| Ярлык PDF | карточка YM: «Скачать ярлыки» | `?plugin=ym&action=getOrderLabels&order_id=` **id Маркета** |
| Ярлык QZ | clabels «Получить наклейку» | только если гейт `is_label` открыт |
| SHIPPED | карточка YM: «Заказ передан в доставку» | `ymChangeStatus` → PUT `SHIPPED` |
| Shop «Отправлен» | workflow `ship` | не то же самое, что PUT SHIPPED; доступно повторно — не жать дважды |
| First-mile | `?plugin=rulimpsupplies` | список отгрузок Маркета; fake-заказ туда не садится |

Вкладка «Подбор» в боковом меню Shop **закомментирована** (`BackendNav.html`).

---

## Состояния

| Слой | После сборки (09:52) | После отгрузки (12:35) |
|------|----------------------|-------------------------|
| Маркет | `PROCESSING` / `READY_TO_SHIP` | `PROCESSING` / `SHIPPED` |
| Shop | `na-upakovku` → затем `shipped` | `shipped` |
| `shop_ym_order_state` | **дыры**: при accept строка не создаётся | апдейты через `updateByField` могут не попасть |

Идентификаторы: Маркет `61089913793` ≠ Shop `2534310`. Сканер Сборки ждёт Shop. Скачивание ярлыка — id Маркета.

---

## Ограничения контура (не баги процесса)

- Sandbox `fake: true` — заказ не в живых first-mile отгрузках.
- Дефолт дат/статусов rulimpsupplies режет список («Отгрузки не найдены» при актах в `OUTBOUND_CREATED`).
- 1С-гейт на Сборке молча пропускается без `id_1c`.
- Лог `query.17.log`: `NULL` после URL — пустой второй dump, не fail API.

---

## Связь с остальным as-is

Каталог/остатки/цены — [systems-map.md](systems-map.md), [data-readiness.md](data-readiness.md).  
YMWB-сборка в Sheets ([process.md](process.md)) **не** этот контур.
