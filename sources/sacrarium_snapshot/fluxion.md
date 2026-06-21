# Fluxion

**Kind:** project / cross-hardware media AI SDK (LamantinAI ecosystem)
**Status:** active; early, but with a usable core
**Language:** Rust
**Repository layout:** workspace из трёх слоёв — `flxcore`, `flxgst`, `flxpython`
**Albert layer (primary focus):** Perception
**Albert layer (adjacent):** feeds Octo (Reaction-layer runtime) как сенсорный источник

## Что это

Fluxion — **Neuromorphic Streaming SDK**, cross-hardware фреймворк для интеллектуального анализа медиапотоков. Не монолитный рантайм, а набор GStreamer-плагинов плюс SDK-фундамент, на которых собираются осмысленные пайплайны.

Главная идея: **один pipeline-граф приложения должен оставаться стабильным, пока под ним меняются runtime'ы и железо.** Не «универсальный модельный сервер», а «один pipeline-контракт с заменяемыми intelligent-элементами».

## Архитектура

Workspace из трёх слоёв:

- **`flxcore`** — SDK-фундамент: метадата, логирование, inference-контракты, backend-библиотеки. Внутри: `flxlog`, `flxmeta`, `flxnncore` (ядро NN-инференса), `flxort` (ONNX Runtime), `flxrknpu` + `flxrknpu-sys` (Rockchip RKNN).
- **`flxgst`** — GStreamer-плагины, экспонирующие capabilities внутрь streaming-графов: `flxneuro` (inference), `flxtracker`, `flxdraw`, `flxmuxer`, `flxdemuxer`, `flxframerate`.
- **`flxpython`** — Python-биндинги к metadata, построенные вокруг нативных `flxn` классов, а не сырых GObject-обёрток.

Под капотом — GStreamer как pipeline-инфраструктура, Rust для плагинов и SDK.

## Текущие возможности (апрель 2026)

Готовые building blocks для structured media AI пайплайнов:

- **Inference** (через GStreamer-plugin `flxneuro`):
  - object detection
  - pose estimation
  - instance segmentation
  - все три на YOLOv8 поверх ONNX Runtime (CPU) или Rockchip RKNN (RK3588 NPU)
  - **compile-time backend selection** для `flxneuro`
- **Tracking:** online object tracking со стабильными per-object ID (на базе `jamtrack-rs`, ByteTrack)
- **Streams:** мультиплексирование и демультиплексирование (request-pad, live-friendly warning-and-drop при отсутствующих ветках)
- **Rendering:** RGBA-overlay для bbox, labels, confidence, track ID, pose keypoints
- **Metadata:** primitives для frames, detections, downstream аналитики; propagation между muxing/inference/tracking/rendering
- **Configuration:** TOML для intelligent-элементов; JSON-обогащение лейблов для YOLO задач
- **Lifecycle:** timeout-based tracker management на multiplexed-потоках
- **Negotiation:** caps-safe plugin negotiation между custom-элементами

Боевая характеристика на сегодня: **4 параллельных потока YOLOv8 + ByteTrack + draw на 18 FPS на RK3588.**

## Направление развития

Идёт по двум осям, которые стоит держать раздельно.

### Near-term (явная roadmap из README)

1. Hardening inference / tracking / rendering / demuxing для production.
2. Усиление metadata-контрактов между intelligent-элементами.
3. Расширение backend-покрытия за пределы ONNX Runtime + RKNN — в первую очередь Raspberry Pi 5-class.
4. Эргономика пайплайнов поверх существующей архитектуры.

Целевая инвариантность: **«один application-level граф, разные backend'ы и hardware-цели»** — одна и та же detection-цепочка работает на разных runtime'ах, одна metadata-модель — на разных таргетах.

Также в near-term: speech-to-text пайплайны и stateful multimodal media intelligence — Fluxion не ограничивается видением.

### Long-term (стратегическая трактовка)

Параллельно классический CV-стек (YOLOv8 + ByteTrack) постепенно перестаёт быть production-целью и становится **teacher / pseudo-label / dataset machinery** для собственной нейросети, разрабатываемой пользователем — streaming scene model для online-анализа видеопотока с прямой генерацией структурированных ивентов.

Это та же линия, что описана в [[streaming_scene_model_project_context]] и [[realtime_vlm_context]]: переход от per-frame inference к stateful real-time scene understanding с эмиссией типизированных событий вместо free-form captioning. Fluxion — инженерный носитель этого перехода.

## Стратегическая роль внутри Octo

Octo — multisensor agent runtime; Fluxion будет одним из самых мощных сенсорных источников, подключённых к нему. Fluxion-коннектор эмитит высокоуровневые сцен-ивенты (incident detected, entity entered zone, abnormal interaction), на которые reflex-слой Octo реагирует детерминированно или эскалирует в cognition-слой.

Это точка схождения двух линий: **Fluxion даёт Octo глаза, которые уже думают.**

## Non-goals (явно из README)

На текущей стадии Fluxion **не пытается** быть:

- универсальным оркестратором всех AI-workflow
- коллекцией спекулятивных backend'ов до того, как pipeline-примитивы доделаны

Цель — крепкий usable foundation с ясным направлением, не широкая абстракция «обо всём».

## Built on

- **GStreamer / `gstreamer-rs`** — pipeline-инфраструктура
- **`jamtrack-rs`** — Rust-tracking, основа ByteTrack-пути
- **ONNX Runtime / `ort`** — ONNX-бэкенд
- **Rockchip RKNN** — NPU-бэкенд для RK3588

## Открытые вопросы

- Когда и как переходить от YOLOv8+ByteTrack к собственной streaming scene model — постепенная гибридная стадия или один прыжок?
- Какой контракт ивентов между Fluxion-коннектором и Octo (envelope, capability declaration, batching)?
- Как distillation использует текущий классический стек как teacher для будущей streaming-модели (формат датасета, целевые задачи)?
- Какой набор edge-акселераторов поддерживать в первой production-версии (Pi 5, Jetson, мобильные NPU)?
- Когда из Fluxion появится явный отдельный «event emission» слой над metadata propagation — для clean интеграции с Octo connector contract?

## Connections

- **Albert layer:** Perception — [[albert_system_map]]
- **Hosts research direction:** [[streaming_scene_model_project_context]], [[realtime_vlm_context]] — модель, которая постепенно заменит текущий классический стек.
- **Adjacent dataset work:** [[synthetic_anomaly_pipeline]] — pipeline деградации видеопотока для обучающих/eval данных.
- **Feeds into:** [[octo_runtime]] — высокоуровневые perception-ивенты как connector source для агентского рантайма.
- **Program:** [[llm_research_roadmap]] § Infrastructure Projects.
- **Values:** [[lamantin_manifesto]], [[values_mapping]] — state-first / event-first / structured-metadata-first соответствует Intellectual Honesty и Channel Purity.
