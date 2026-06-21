# Gonka — архитектурный «второй мозг»

Архитектурный анализ проекта [gonka](https://github.com/gonka-ai/gonka) (децентрализованная AI-инфраструктура, **Proof of Compute 2.0**) по принципам DDD + атомарные заметки в стиле «второго мозга» (distill-from-first-principles).

> Слой исходников: `8a35022` · тег `v0.2.13-devshard-v2` · 2026-06-19.

## Архитектура (HLD)

```text
                          👤 Пользователь / Разработчик
                                     │  OpenAI-совместимый HTTP
                                     ▼
  ┌──────────────────────────── OFF-CHAIN · адаптеры ───────────────────────═────┐
  │   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐               │
  │   │  dapi        │      │  devshard    │      │  ml node     │               │
  │   │ оркестратор  │      │ платёж-канал │      │ vLLM · PoC   │               │
  │   │ broker·фазы  │      │ эскроу 2/3+1 │      │ GPU compute  │               │
  │   └──────┬───────┘      └──────┬───────┘      └──────┬───────┘               │
  │  Cosmos  │ tx/events    settle │ state-root   команды│ артефакты             │
  └──────────┼─────────────────────┼─────────────────────┘                       │
             │                     │            conformist к цепи ───────────────┘
             ▼                     ▼
  ╔══════════════════ INFERENCE-CHAIN (форк Cosmos SDK + CometBFT) ═══════════════╗
  ║   ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓     ║
  ║   ┃  ▓▓▓ CORE ▓▓▓   x/inference                                         ┃     ║
  ║   ┃  эпохи · PoC 2.0 · settlement · pricing · делегирование · bridge    ┃     ║
  ║   ┗━━━━━┯━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┯━━━━━━━━━━━━━━━┛     ║
  ║  дирижирует│ AdvanceEpoch·Slash·RequestSig             │ compute → power      ║
  ║            ▼                                           ▼                      ║
  ║   ┌──────────── SUPPORTING ───────────┐   ┌─────────── GENERIC ───────────┐   ║
  ║   │ collateral · bls · streamvesting  │   │ staking(fork) · group · gov   │   ║
  ║   │ restrictions · genesistransfer    │   │ bank · feegrant · CometBFT    │   ║
  ║   └───────────────────────────────────┘   └───────────────────────────────┘   ║
  ╚═══════════════════════════════════════════┯═══════════════════════════════════╝
                                              │ BLS threshold signature
                                              ▼
                              ┌──────────────────────────────┐
                              │ ⛓  EVM BRIDGE  (WGNK token)  │
                              └──────────────────────────────┘

  Легенда:  ┏━┓ ядро (компетентное преимущество) · ┌─┐ модули/адаптеры · ║═ граница цепи
            ───►  поток/вызов · ▓▓▓ «горячая» GPU-работа
```

> Детальнее: [🚁 Helicopter View](architecture/helicopter-view.md) (6 ANSI-панелей) · [🗺️ System Map](architecture/00-system-map.md) (mermaid).

## Навигация

- **[🖼️ Галерея диаграмм](architecture/README.md)** — навигатор по всем визуализациям проекта.
- **[🚁 Helicopter View](architecture/helicopter-view.md)** — вид с высоты птичьего полёта (ANSI): вся система на одном экране в философии DDD.
- **[ARCHITECTURE.md](ARCHITECTURE.md)** — точка входа: стратегический DDD (Context Map, классификация поддоменов, Ubiquitous Language).
- **[architecture/00-system-map.md](architecture/00-system-map.md)** — 🗺️ единая мастер-диаграмма всех компонентов и потоков.
- **[architecture/01..11](architecture/)** — тактические разборы: ядро PoC, спутниковые контексты, оркестратор, devshard, экономика, ML-узел, мост, эволюция, глубокие механизмы.
- **[REVIEW.md](REVIEW.md)** — состязательная верификация документации против кода (что подтверждено, что исправлено, что непроверяемо).
- **[Wiki/](Wiki/)** — 31 атомарная заметка (Obsidian-хранилище). Точка входа — [`Wiki/MOC — gonka.md`](Wiki/MOC%20—%20gonka.md).

## Что покрыто

Консенсус (PoC 2.0), экономика V2 (bitcoin-эмиссия, collateral, vesting, динамическое ценообразование), off-chain оркестрация (dapi/broker), devshard (эскроу-канал инференса), ML-узел (математика PoC, vLLM), EVM-мост (BLS-порог), эволюция (Testermint, апгрейды), продвинутые подсистемы (делегирование, bandwidth-лимитер).

---

## Лицензия

- **Исходный код gonka** (каталог `repo/`) — [Gonka License](LICENSE.md): как у исходного проекта gonka — модифицированная Apache 2.0 с ограничением на форк (© Product Science, Inc.).
- **Архитектурный обзор** (документация: `ARCHITECTURE.md`, `architecture/`, `Wiki/`, `REVIEW.md`) — выполнен на основе лицензии [AGPL-3.0](LICENSE-AGPL.md).
