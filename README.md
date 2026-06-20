# Gonka — архитектурный «второй мозг»

Разбор архитектуры [gonka](https://github.com/gonka-ai/gonka) (децентрализованная AI-инфраструктура, **Proof of Compute 2.0**) по принципам DDD + атомарные заметки в стиле «второго мозга» (distill-from-first-principles).

> Слой исходников: `8a35022` · тег `v0.2.13-devshard-v2` · 2026-06-19.

## Навигация

- **[ARCHITECTURE.md](ARCHITECTURE.md)** — точка входа: стратегический DDD (Context Map, классификация поддоменов, Ubiquitous Language).
- **[architecture/00-system-map.md](architecture/00-system-map.md)** — 🗺️ единая мастер-диаграмма всех компонентов и потоков.
- **[architecture/01..11](architecture/)** — тактические разборы: ядро PoC, спутниковые контексты, оркестратор, devshard, экономика, ML-узел, мост, эволюция, глубокие механизмы.
- **[REVIEW.md](REVIEW.md)** — состязательная верификация документации против кода (что подтверждено, что исправлено, что непроверяемо).
- **[Wiki/](Wiki/)** — 31 атомарная заметка (Obsidian-хранилище). Точка входа — [`Wiki/MOC — gonka.md`](Wiki/MOC%20—%20gonka.md).

## Что покрыто

Консенсус (PoC 2.0), экономика V2 (bitcoin-эмиссия, collateral, vesting, динамическое ценообразование), off-chain оркестрация (dapi/broker), devshard (эскроу-канал инференса), ML-узел (математика PoC, vLLM), EVM-мост (BLS-порог), эволюция (Testermint, апгрейды), продвинутые подсистемы (делегирование, bandwidth-лимитер).

---

*Аналитическая документация; исходный код gonka — в [оригинальном репозитории](https://github.com/gonka-ai/gonka).*

🤖 Generated with [Claude Code](https://claude.com/claude-code)
