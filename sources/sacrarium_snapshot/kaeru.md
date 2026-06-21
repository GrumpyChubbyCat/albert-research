# Kaeru

**Kind:** project
**Status:** active — pre-1.0, MVP shipped (2026-05-07)
**License:** MIT OR Apache-2.0

## Что это

Когнитивная память для LLM-агентов — типизированный property-граф, в котором агент мыслит, плюс recollection-слой для долгосрочных идей и итогов. Реализация абстрактной идеи из [[wiki_as_agent_memory]].

Два слоя по биологической аналогии:

- **cognitive** (hippocampus) — рабочий граф высокой скорости, где агент активно мыслит: episodes, scratch, drafts, hypotheses, experiments, audit events.
- **recollection** (cortex) — устоявшиеся идеи, итоги, summaries, references; преимущественно read-only.

Per-initiative scope через junction relations: один substrate, много инициатив, multi-membership. Агент, открывая проект A, спрашивает «что я тут делал в прошлый раз?» и получает ответ, отскопированный по A.

## Имя

**Kaeru** (蛙) — японское «лягушка». Омонимично двум глаголам сразу: 帰る («возвращаться») и 変える («менять»). Имя описывает функцию: агент, который возвращается, вспоминает и переоформляет.

Лягушачья метафора попадает в дизайн помимо имени: graph traversal как прыжки между узлами; амфибия живёт в двух средах — cognitive и recollection; метаморфоза головастика — consolidation; язык на расстояние — recall.

## Стек

- **CozoDB поверх RocksDB.** Bi-temporal `Validity` нативна для substrate'а — не пристроена сверху. Time-travel запросы из коробки, conflict resolution non-destructive (старая версия инвалидируется, не удаляется).
- **Cargo workspace из 3 крейтов:** `kaeru-core` (библиотека: substrate, schema, primitives), `kaeru-cli` (бинарь `kaeru`: CLI surface), `kaeru-mcp` (бинарь `kaeru-mcp`: Model Context Protocol HTTP server).
- **Single-binary embedded** для CLI: ни сети, ни сервера, vault на диске под платформенным дефолтом (`$XDG_DATA_HOME/kaeru` на Linux). Override через `KAERU_VAULT_PATH`.
- **MCP-сервер — long-lived HTTP daemon, один на машину.** RocksDB single-writer; stdio-MCP-подсетка с подпроцессом-на-сессию упёрлась бы в lock contention. Вместо этого сессии Claude Code / Cursor / других MCP-aware агентов конкурентно подключаются к одному демону.
- **Markdown-export** в Obsidian-friendly vault — для человека-читателя; substrate остаётся source of truth.

## Что внутри

**15 типов узлов:**

- 9 cognitive: `task`, `checklist`, `roadmap`, `experiment`, `hypothesis`, `scratch`, `draft`, `episode`, `audit_event`.
- 6 recollection: `idea`, `outcome`, `reference`, `concept`, `entity`, `summary` (последние три могут жить и в cognitive как provisional drafts; `tier`-колонка различает).

**12 типов рёбер с операционной семантикой** — curator API реагирует на каждый тип, а не просто хранит как ассоциацию: `derived_from`, `refers_to`, `supersedes`, `causal`, `temporal`, `contradicts`, `part_of`, `blocks`, `targets`, `verifies`, `falsifies`, `consolidated_to`.

**~36 примитивов curator API** в категориях: awareness (`use_initiative`, `awake`, `pin`, `unpin`, `active_window`), recall (`recall`, `walk`, `summary_view`, `recent_episodes`, `fuzzy_recall`), active mutation (`write_episode`, `synthesise`, `link`, `update_connections`, `reorganise`, `mark_resolved`, `mark_under_review`), hypothesis cycle (`formulate_hypothesis`, `run_experiment`, `update_hypothesis_status`), metabolism (`forget`, `improve`, `lint`), consolidation (`consolidate_out`, `consolidate_in`), recollection (`recollect_idea`, `recollect_outcome`, `recollect_summary`, `recollect_reference`, `recollect_provenance`), temporal (`at`, `history`).

Каждая мутация автоматически пишет `audit_event`-узел.

## Принципы

- **Facilitator, not enforcer.** Примитивы — доступные инструменты. Агент и пользователь сами решают, когда что вызывать. CLI подсказывает, не блокирует. Никаких mandated operation sequences.
- **`audit_event` — first-class node.** Изменения памяти сами становятся reasoning surface для агента. Substrate-уровневая история (`Validity`) и operational audit (`audit_event`-узлы) разделены: substrate отвечает за «что было», audit — за «кто и зачем сделал».
- **Структурный recall первичен.** Точное имя → typed graph traversal → summary view. FTS как fuzzy fallback, когда точное имя забыто. Vector embeddings — не основной режим.
- **Multi-session continuity** — главное use-case framing. Агент, открывая инициативу, имеет полный контекст того, о чём думал прошлой сессии, может ходить по provenance-цепочкам, и может явно консолидировать итоги в стабильный долгосрочный слой.
- **Two-tier с явным `consolidate_out`.** Operational drafts продвигаются в archival как deliberate, logged operation. Provenance (`derived_from`) переживает границу.

## Что вдохновило

- **Karpathy LLM-wiki gist** (`442a6bf555914893e9891c11519de94f`) — исходный паттерн: типизированные страницы, page index, append-only журнал, LLM как куратор.
- **Graphiti / Zep** — bi-temporal knowledge graph approach, прямой источник идеи нативной `Validity`.
- **Cognee** — curator-driven knowledge engine.
- **PageIndex** — reasoning-based hierarchical-summary navigation.
- **Гиппокамп + кортекс** в нейронауке — обоснование two-tier и явной consolidation между слоями.

## Решения, которые крышковали open questions

Ряд вопросов из абстрактной идеи [[wiki_as_agent_memory]] получил конкретный ответ в реализации:

- **Substrate**: CozoDB + RocksDB, не indradb. Главный аргумент — bi-temporal `Validity` нативна, не пристроена через audit-log как outbox.
- **Один граф vs два**: один graph + junction relations (`node_initiative`, `edge_initiative`) для per-initiative scope. Multi-membership: один и тот же узел может принадлежать нескольким инициативам одновременно.
- **Versioning / time-travel**: substrate-уровневые `Validity`-ассертации; не отдельный snapshot-механизм.
- **Conflict resolution**: `contradicts` запускает non-destructive `under_review`-флоу; `supersedes` ретрактует через bi-temporal substrate.
- **Retrieval-приоритет**: structural-first (имя → traversal → summary view), FTS как fallback, vector — не primary.

## Где живёт

- **Code:** `~/code/personal/kaeru/` — Rust workspace + initiative-side `research/` (live-design vault), `ROADMAP.md`, `SPEC.md`.
- **Surface:**
  - `kaeru-cli` — CLI binary (`kaeru initiatives`, `kaeru awake`, `kaeru jot`, `kaeru drill`, `kaeru claim`, `kaeru at`, `kaeru export`, …)
  - `kaeru-mcp` — long-lived HTTP MCP daemon
  - `kaeru-skill` — portable agent skill для рантаймов без MCP
- **Тесты:** 43, все зелёные на момент шипа.
- **License:** dual MIT OR Apache-2.0.

## Open

Из объявленного на шипе:

- Concurrency story для MCP-сервера при batch-fired calls (read-may-race-ahead-of-pending-writes).
- Audit events не привязаны к initiative junction; фильтрация в export через `affected_refs` intersection — workaround.
- Нет адаптеров для LangChain / Rig.
- Whole-second `Validity` resolution: `link` + immediate `unlink` в одну секунду резолвится неоднозначно (test code спит между операциями; интерактивное использование fine).

## Дальше

Pre-1.0 → friend-pilot phase → cloud (sync, gRPC, auth, multi-user collaboration). Если успех подтвердится pilot'ом, кortor становится фундаментом для open memory platform.

## Connections

- **Implements:** [[wiki_as_agent_memory]] — the abstract idea Kaeru realises.
- **Memory model:** [[memory_model_design]] — four-layer taxonomy Kaeru's design serves; cognitive tier covers working+episodic, recollection covers semantic.
- **Sibling perspective:** [[embodied_agent_memory]] — Frozen Backbone + Plastic Memory architectural view; Kaeru is a candidate substrate for the Plastic Memory side.
- **Consumer (planned):** [[octo_runtime]] / [[octo_architecture]] — Octo's Memory Layer is intended to integrate Kaeru as the two-tier component.
- **Albert layer:** Memory — [[albert_system_map]].
- **Program:** [[llm_research_roadmap]].
- **Values:** [[lamantin_manifesto]], [[values_mapping]] — explainability через provenance, transparent edit history через bi-temporal substrate, и agent-as-owner соответствуют Veracity и Intellectual Honesty.
