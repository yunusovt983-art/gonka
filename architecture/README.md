# 🖼️ Галерея архитектуры gonka

> Навигатор по всем диаграммам и документам. Каждая страница построена по принципу
> **суть → 🗺️ визуальный обзор → 💻 детали/код → 🔗 связи**.
> Стиль диаграмм единый (clean-architecture): зелёное **ядро**, тёмные **адаптеры**, бледная **аннотация-правило**, подписи на стрелках.

---

## 🚀 Быстрый старт — три уровня высоты

| Уровень | Документ | Формат | Когда смотреть |
|---|---|---|---|
| 🚁 **Птичий полёт** | [helicopter-view.md](helicopter-view.md) | ANSI (6 панелей) | первое знакомство — вся система на одном экране |
| 🗺️ **Карта системы** | [00-system-map.md](00-system-map.md) | mermaid (6 схем) | как компоненты связаны и какие потоки идут |
| 📐 **Стратегия (DDD)** | [../ARCHITECTURE.md](../ARCHITECTURE.md) | mermaid (2 схемы) | классификация поддоменов, Context Map, единый язык |

---

## 🗺️ Обзорные карты (высокий уровень)

| Диаграмма | Где | Что показывает |
|---|---|---|
| Слои и направление зависимостей | [ARCHITECTURE.md](../ARCHITECTURE.md) | off-chain → цепь (conformist), ядро дирижирует доменами |
| Context Map (5+ контекстов) | [ARCHITECTURE.md](../ARCHITECTURE.md) | отношения between-contexts (Customer–Supplier, Conformist, ACL) |
| Helicopter View · 30 000 футов | [helicopter-view.md](helicopter-view.md) | 4 плана: clients → off-chain → chain → EVM |
| Helicopter View · классификация поддоменов | [helicopter-view.md](helicopter-view.md) | Core / Supporting / Generic на оси преимущества |
| Helicopter View · правило зависимостей | [helicopter-view.md](helicopter-view.md) | всё направлено внутрь, к ядру |
| Helicopter View · такт эпохи | [helicopter-view.md](helicopter-view.md) | StartOfPoC → … → ClaimMoney; halt-правило |
| Helicopter View · поток власти | [helicopter-view.md](helicopter-view.md) | compute → вес → залог → CometBFT |

---

## 🗺️ System Map — 6 схем потоков

Все в одном файле → [00-system-map.md](00-system-map.md):

| # | Схема | Что показывает |
|---|---|---|
| 1 | Мастер-карта | все компоненты и связи (off-chain + 7 on-chain модулей + мост) |
| 2 | Поток власти | compute → consensus (PoC 2.0) |
| 3 | Поток инференса | transfer-agent → executor → SPRT-валидация |
| 4 | Жизненный цикл эпохи | стадии PoC-цикла |
| 5 | Devshard-эскроу | open → секвенсор → gossip → settle 2/3+1 |
| 6 | Мост | обе стороны с авто-возвратом |

---

## 📐 Документы по контекстам (01–11)

| Док | Обзорная диаграмма | Что показывает |
|---|---|---|
| [01 · Core PoC](01-core-proof-of-compute.md) | «Один часовой дирижирует исполнителями» | EndBlock → collateral/bls/vesting/staking |
| [02 · Supporting](02-supporting-contexts.md) | _(текстовый разбор)_ | bls · collateral · streamvesting · restrictions · … |
| [03 · dapi](03-orchestration-dapi.md) | «Слои dapi» | вход → broker-ядро → адаптеры |
| [04 · Devshard](04-devshard-payment-channel.md) | «Слои devshard» | Gateway → Proxy → Session → transport → цепь |
| [05 · Экономика](05-economics.md) | две монеты + фикс-пул | WorkCoins 1:1 + RewardCoins по весу; остаток → gov |
| [06 · Каталог идей](06-ideas-catalog.md) | 6 категорий приёмов | что переносимо в другие системы |
| [07 · ML-узел](07-mlnode-compute.md) | api-proxy → vLLM/pow/train | две реализации PoC (v2 в форке vLLM) |
| [08 · Мост + proto](08-bridge-and-protocol.md) | EVM ↔ релеер ↔ ядро ↔ BLS | порог BLS авторизует, вход >50% власти |
| [09 · Тесты + эволюция](09-testing-and-evolution.md) | dual-binary upgrade | одно gov-событие → cosmovisor/dapi/ml + Testermint |
| [10 · Глубокие механизмы](10-deep-mechanisms.md) | 3 механизма | pricing · gossip/recovery · guardian |
| [11 · Продвинутые подсистемы](11-advanced-subsystems.md) | 3 механизма | делегирование · анатомия апгрейда · лимитер |

> 🔍 Точность всех доков проверена против кода → [REVIEW.md](../REVIEW.md).

---

## 🧠 Wiki — второй мозг (31 атомарная заметка)

Точка входа → [Wiki/MOC — gonka](../Wiki/MOC%20—%20gonka.md).

**29 заметок** содержат `🗺️ Обзор` (стилизованная диаграмма) + `💻 Код` (реальный сниппет из `repo/` с `file:line`). Структура каждой: суть → обзор → код → детали → связи.

Кластеры: первопринципы · доменные понятия · off-chain/devshard · ML-узел · мост/эволюция · глубокие механизмы · синтез («25 переносимых идей»).

---

## 🎨 Палитра (как читать цвета)

```text
  🟩 зелёный  (:::core / :::coresub) — доменное ЯДРО и его части
  ⬛ тёмный   (:::adapter)           — адаптеры / периферия / исполнители
  ⬛ чёрный   (:::entry)             — точки входа / внешние системы
  ▫️ без рамки (:::note)            — аннотация-правило сверху
  ───►  поток / вызов        ◄── направление зависимости
```

Диаграммы рендерятся нативно и в **GitHub**, и в **Obsidian** (mermaid + ANSI в code-блоках).
