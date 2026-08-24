# AGENTS.md — карта для агентов

Репозиторий: **смысловая модель** проекта «Яндекс Маркет ↔ RULI», не монорепо кода.

Перед любой записью прочитай [meta/FOLDER-LOGIC.md](meta/FOLDER-LOGIC.md).  
Если логика папок меняется — править сначала `meta/FOLDER-LOGIC.md`, потом структуру.

## Куда писать

| Задача | Папка | Не делать |
|--------|--------|-----------|
| Разведка, AS IS, сырьё evidence | `research/` | не писать сразу Feature PRD |
| Выбор варианта решения | `decisions/` | не прятать decision в prose PRD |
| Узлы/рёбра влияния | `graph/graph.yaml` (+ вид в `graph.md`) | не дублировать бэклог Трекера |
| Контракт для разработки | `features/F-*/PRD.md` | не писать PRD до закрытия нужных `D-*` |
| Указатели на код/БД/Трекер | `refs/` | не клонировать чужие репо сюда |
| Playbook оркестрации | `orchestration/` | не смешивать с контентом проекта |
| Секреты | `secrets/` (local) | никогда не коммитить значения |

## Жизненный цикл

```text
сигнал (INI / хотелка)
  → research/ (evidence + briefs)
  → decisions/ (вариант выбран)
  → graph/ (F-* / R-* / D-*)
  → features/F-.../PRD.md   # ключевые сначала
  → изменение R-* / D-* → blast radius → патч соседних PRD
```

## Гейты

1. **Problem** — outcome проверяем.
2. **Reality** — есть Reality Brief / as-is с evidence.
3. **Options** — для зависимых фич закрыты нужные `D-*`.
4. **Acceptance** — в Feature PRD есть AC (happy + ключевые fail).
5. **Tracker write** — только по явной просьбе пользователя.

## MCP и внешние системы

- **Yandex Tracker** — read-only по умолчанию; шаблоны в `refs/tracker.md`.
- **БД** — через MCP; выжимки в `research/evidence/db/` без PII в git.
- **Код** — пути в `refs/codebases.md`; якоря в evidence/code, не форки.

## Секреты

См. `secrets/README.md` и `.env.example`. Не цитировать токены/DSN в чате и в md.

## Навигация

- Индекс папок: [WORKSPACE-INDEX.md](WORKSPACE-INDEX.md)
- Логика структуры: [meta/FOLDER-LOGIC.md](meta/FOLDER-LOGIC.md)
- Статус фич: [features/STATUS.md](features/STATUS.md)
