# WORKSPACE-INDEX

Краткий индекс. Подробная логика папок — [meta/FOLDER-LOGIC.md](meta/FOLDER-LOGIC.md).

## Как начинать чат

- «Смотри `AGENTS.md` и `graph/`»
- «Дополни AS IS в `research/`»
- «Смотри прогон FBS: `research/briefs/fbs-test-2026-09-01.md`»
- «Набросай скелет графа в `graph/graph.yaml`»
- «Feature PRD для F-… по шаблону»
- «Поправь логику папок в `meta/FOLDER-LOGIC.md`»

## Папки

| Папка | О чём | Теги | Главные файлы |
|-------|--------|------|----------------|
| [meta/](meta/) | Логика организации repo (корректируемая) | `structure` `agents` | [FOLDER-LOGIC.md](meta/FOLDER-LOGIC.md) |
| [context/](context/) | Context Pack: charter, системы, ограничения | `charter` `systems` | [00-charter.md](context/00-charter.md), [README](context/README.md) |
| [research/](research/) | Исследования → AS IS, evidence, briefs | `as-is` `evidence` | [README](research/README.md), [reality-brief](research/briefs/reality-brief.md) |
| [decisions/](decisions/) | Options / ADR-лайт | `options` `ADR` | [README](decisions/README.md), [_template](decisions/_template.md) |
| [graph/](graph/) | Граф проекта: узлы, рёбра, blast radius | `graph` `impact` | [graph.yaml](graph/graph.yaml), [graph.md](graph/graph.md) |
| [features/](features/) | Feature PRD по итерациям | `PRD` `AC` | [STATUS.md](features/STATUS.md), [_template](features/_template/PRD.md) |
| [orchestration/](orchestration/) | Playbook оркестратора и handoff | `agents` `gates` | [README](orchestration/README.md), [router.md](orchestration/router.md) |
| [refs/](refs/) | Указатели на код, БД, Трекер | `MCP` `paths` | [README](refs/README.md) |
| [secrets/](secrets/) | Локальные креды (не в git) | `secrets` | [README](secrets/README.md) |

## Связанный код

См. [refs/codebases.md](refs/codebases.md) — клоны не входят в этот репозиторий.
