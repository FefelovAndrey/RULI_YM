# Граф проекта — вид для человека

Канон: [graph.yaml](graph.yaml). Схема: [schema.md](schema.md).

## Позвоночник (черновик)

```text
Каталог/оффер → Цены → Остатки → Заказы → (Возвраты)
         └────────── C-YM-API ──────────┘
```

Уточняется после `research/briefs/reality-brief.md`.

## Волны

| ID | Название | Статус |
|----|----------|--------|
| W1-FOUNDATION | Foundation | planned |
| W2-CORE | Core | planned |
| W3-EXCEPTIONS | Exceptions | planned |

## Фичи (слой 1)

| ID | Title | Wave | depends_on |
|----|-------|------|------------|
| F-CATALOG | Каталог / маппинг | W1 | C-YM-API |
| F-PRICE-SYNC | Цены | W2 | C-YM-API, F-CATALOG |
| F-STOCK-SYNC | Остатки | W2 | C-YM-API, F-CATALOG |
| F-ORDERS | Заказы | W2 | C-YM-API, F-STOCK-SYNC |
| F-RETURNS | Возвраты | W3 | F-ORDERS |

## Mermaid

```mermaid
flowchart LR
  C[C-YM-API] --> F1[F-CATALOG]
  F1 --> F2[F-PRICE-SYNC]
  F1 --> F3[F-STOCK-SYNC]
  F3 --> F4[F-ORDERS]
  F4 --> F5[F-RETURNS]
  C --> F2
  C --> F3
  C --> F4
```

## Blast radius — как считать

1. Меняете `R-*` / `D-*` / `shares_*` у фичи в `graph.yaml`.
2. Собираете все узлы с ребром на этот id.
3. Патчите только их Feature PRD в `features/`.

## Views

Подграфы по волнам — в [views/](views/) (по мере появления).
