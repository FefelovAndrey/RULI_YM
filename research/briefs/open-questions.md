# Open Questions Log

| ID | Вопрос | Владелец | Срок | Статус | Связь (F-/D-/R-) |
|----|--------|----------|------|--------|------------------|
| Q-001 | Карточки ЯМ заводятся только в кабинете? Кто гарантирует `Sklad` = `offerId`? | Операции | | open | D-YM-CARD-GEN |
| Q-002 | Реальная схема — только FBS? Есть FBY/DBS/экспресс? | Операции | | open | D-YM-FULFILLMENT |
| Q-003 | Зачем `discountBase = Цена × 1.18`? Это требование витрины или локальное правило? | Коммерция | | open | R-PRICE-RULE |
| Q-004 | Откуда в `!YMWB.db` попадают Invask/Okno/United — отдельный импорт? | ИТ | | open | D-STOCK-SOURCE |
| Q-005 | Нужно ли RULI закрывать приёмку/ярлык/отгрузку по API или достаточно сигнала в Трекер/Sheets? | Sponsor | 2026-09-01 | **answered** | D-YM-FULFILLMENT F-ORDERS |
| Q-006 | Где живёт `clients/http_client` и какие retry/timeout у Partner API? | ИТ | | open (YMWB; в WA — `shopYmApi`) | |
| Q-007 | Мультикампания / несколько складов ЯМ планируются? | ИТ | | open | |
| Q-008 | `offerId` ЯМ = `Sklad` (YMWB) или `sku_id` (как Ozon/WB)? | ИТ / операции | | open | D-YM-CARD-GEN R-OFFER-ID |
| Q-009 | Какой `shop_set` будет списком выгрузки на ЯМ (аналог `ozon-24`)? | Операции | | open | F-CATALOG |
| Q-010 | Кто ведёт маппинг part_type → категория ЯМ и обязательные parameterId? | Контент | | open | F-CATALOG |
| Q-011 | Авто-accept живого FBS без Postman на профиле 17 стабилен? | ИТ | | open | F-ORDERS |
| Q-012 | First-mile (акт/слот) на живом FBS: свой документ, без чужих отгрузок? | Операции | | open | F-ORDERS |

Q-005: на sandbox 2026-09-01 RULI **уже** закрывает accept / READY_TO_SHIP / labels / SHIPPED в WA. Не закрыто: запись `ym_order_state` на accept, кнопка ярлыка на Сборке, first-mile. См. [fbs-test-2026-09-01.md](fbs-test-2026-09-01.md).
