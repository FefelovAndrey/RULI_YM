# YM_Ruli — Яндекс Маркет ↔ RULI

Аналитический репозиторий проекта подключения **Яндекс Маркета** к контуру **RULI**.

Здесь живут смысл и контракты для разработки: исследования AS IS, граф проекта, Feature PRD.  
Код систем (WA, 1С, PIM, MPS, …) и БД — **не** клонируются сюда; доступ через MCP и пути из `refs/`.

## С чего начать

1. [AGENTS.md](AGENTS.md) — карта для агентов и правила записи.
2. [WORKSPACE-INDEX.md](WORKSPACE-INDEX.md) — индекс папок.
3. [meta/FOLDER-LOGIC.md](meta/FOLDER-LOGIC.md) — **логика организации** (можно корректировать).
4. [context/00-charter.md](context/00-charter.md) — зачем проект.

## Три рабочих слоя

| Слой | Папка | Содержание |
|------|--------|------------|
| Исследования → AS IS | `research/` | evidence, briefs, as-is |
| Граф проекта | `graph/` | узлы, рёбра, blast radius |
| Feature PRD | `features/` | контракты по итерациям |

Порядок: **research → decisions → graph → features** (ключевые фичи сначала).

## Секреты

Значения — только в `secrets/` (не в git) или в Cursor secrets.  
Имена переменных — [`.env.example`](.env.example).

## Трекер

Яндекс Трекер: read-only по умолчанию. Запись — только по явной просьбе в сообщении.
