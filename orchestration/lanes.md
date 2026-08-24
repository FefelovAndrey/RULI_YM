# Lanes — типы работ в проекте ЯМ

Уточнять по мере Reality Brief. Стартовый набор:

| Lane | Когда | Артефакт | Глубина |
|------|-------|----------|---------|
| Fast Change | Локальный UI/поле, 1 система | Change Spec / короткий PRD | Узкий code locator |
| Rule / Formula | Формула цены, комиссии, правило маппинга | Rule Spec + примеры | Code + data samples |
| Sync | Цены/остатки/статусы | Sync Spec / Feature PRD + NFR | Integration + data lag |
| Process | Сквозной процесс (заказ/возврат) | PRD-M | Process + code + data |
| Integration | API ЯМ, webhooks, polling | Options → Integration PRD | Scout + Critic + Options |
| Program | Весь запуск канала | Charter → волны → Feature PRD | Полный конвейер |

Program lane — режим по умолчанию для этого репозитория; внутри — нарезать на Sync/Process/… фичи.
