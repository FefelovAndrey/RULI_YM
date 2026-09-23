# Системы и границы

Статус: `draft`  
Обновлено: `2026-09-01` (sandbox FBS: accept → SHIPPED; first-mile не закрыт)

Контур Яндекс Маркета в этом репозитории — **RULI**: Webasyst Shop, 1С, PIM, кабинет/Partner API. Сторонние сервисы витрины в модель не входят.

## Контур

| Система | Роль в ЯМ-проекте | Source of truth (черновик) |
|---------|-------------------|----------------------------|
| WA / Shop | Заказы, сборка, ярлыки, профиль кампании ЯМ | `shop_order` + params `ym_order_id`; плагины `ym`, `clabels`, `rulimpsupplies` |
| 1С / WMS | Отбор, РО, передача в упаковку | Статусы отбора/РО по `id_1c` заказа |
| PIM | Маппинг WA↔1С | Не выгрузка карточек на МП |
| Яндекс Маркет | Витрина, офферы, статус FBS, ярлыки, first-mile | Кабинет + Partner API |
| Ozon / WB в WA | Образец карточек и сборки | Не ЯМ |

## Потоки (черновик)

```text
Partner API → plugins/ym → shop_order (ym_order_id)
  → 1С (НомерЗаказаСайта)
  → WMS отбор
  → clabels сборка / ярлык / READY_TO_SHIP
  → rulimpsupplies first-mile
```

Карточки, остатки и цены на ЯМ — отдельные контуры WA/`ym` (и кабинет); не смешивать со сторонними сервисами.

## Интеграции

| Связь | Синхрон/асинхрон | Владелец | Заметки |
|-------|------------------|----------|---------|
| WA `ym` ↔ Partner API | HTTP | `shopYmApi` | Заказы, статусы, ярлыки, остатки/цены в профиле плагина |
| WA → 1С | обмен odata1c | `odata1c` | `НомерЗаказаСайта` = id заказа Shop |
| clabels → Partner API | HTTP через `shopYmApi` | `clabels` | Печать ярлыка, `READY_TO_SHIP` |
| rulimpsupplies → Partner API | HTTP | first-mile | Поставки / акт |

Детали и evidence — в `research/` (якоря кода WA). Пути репозиториев — в `refs/codebases.md`.

Прогон 2026-09-01 + чеклист доработок: `research/briefs/fbs-test-2026-09-01.md`. AS IS исполнения: `research/as-is/fbs-fulfillment-wa.md`.
