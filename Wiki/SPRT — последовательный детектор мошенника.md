---
title: SPRT — последовательный детектор мошенника
type: concept
tags: [gonka, sprt, validation, statistics, anti-fraud]
source: inference-chain/x/inference/calculations/sprt.go, status.go
updated: 2026-06-20
---

# SPRT — последовательный детектор мошенника

> **Суть:** корректность AI-выходов нельзя проверить «равенством» — модели
> недетерминированны. Поэтому каждый инференс выборочно переисполняется, а решение
> «честный / мошенник» по потоку исходов принимает **последовательный критерий
> Вальда (SPRT)** — с контролируемыми ошибками и минимумом наблюдений.

## 🗺️ Обзор
```mermaid
flowchart TB
    NOTE["последовательный критерий Вальда: решение по потоку исходов<br/>контролируемые ошибки, минимум наблюдений"]:::note
    OBS["Исходы проверок<br/>failures / passes"]:::adapter
    LLR["LLR<br/>копит log-likelihood ratio"]:::core
    DEC["Decision<br/>пороги ±H"]:::coresub
    INV["INVALID<br/>LLR ≥ +H"]:::entry
    PASS["оправдан<br/>LLR ≤ −H"]:::entry
    NOTE -.-> OBS
    OBS -->|"UpdateCounts"| LLR
    LLR -->|"сравнение"| DEC
    DEC -->|"нарушитель"| INV
    DEC -->|"чист"| PASS
    classDef core fill:#2e7d46,stroke:#86efac,color:#ffffff
    classDef coresub fill:#3a8d56,stroke:#bbf7d0,color:#ffffff
    classDef adapter fill:#1e293b,stroke:#475569,color:#e2e8f0
    classDef entry fill:#0f172a,stroke:#334155,color:#e2e8f0
    classDef note fill:none,stroke:none,color:#94a3b8
```

## 💻 Код (`inference-chain/x/inference/calculations/sprt.go:25`)
```go
// UpdateCounts applies a batch: `failures` and `passes` since last call.
// LLR += failures*logFail + passes*logPass
func (s SPRT) UpdateCounts(failures, passes int64) SPRT {
    if failures != 0 {
        s.LLR = s.LLR.Add(s.logFail.Mul(decimal.NewFromInt(failures)))
    }
    if passes != 0 {
        s.LLR = s.LLR.Add(s.logPass.Mul(decimal.NewFromInt(passes)))
    }
    return s
}

// Decision uses symmetric thresholds ±H
func (s SPRT) Decision() Decision {
    if s.LLR.GreaterThanOrEqual(s.H) {
        return Fail // favor H1 (reject H0)
    }
    if s.LLR.LessThanOrEqual(s.H.Neg()) {
        return Pass // favor H0
    }
    return Undetermined
}
```

## Как работает (`calculations/sprt.go`)
Копится log-likelihood ratio:
```
LLR += failures · ln(p1/p0) + passes · ln((1-p1)/(1-p0))
решение:  LLR ≥ +H → нарушитель ;  LLR ≤ −H → оправдан ;  иначе — ещё выборка
```
Два независимых SPRT на участника:
| SPRT | H0 (хороший) | H1 (плохой) | Вердикт |
|---|---|---|---|
| **Invalidation** | `FalsePositiveRate` | `BadParticipantInvalidationRate=0.20` | INVALID |
| **Inactivity** | низкий простой | высокий простой | INACTIVE |

Порог `H = 4`. Плюс «триппроволока»: подряд идущие отказы при
`FalsePositiveRate^N < 1e-6` → мгновенный INVALID.

## Две оптимизации горячего пути
1. **Биномиальный тест по таблицам.** Критические значения предрассчитаны (ключи ‰:
   50/100/200/300/400/500) → O(log n), zero-alloc вместо точного `decimal`
   (ускорение ~10⁴–10⁵×). `criticalK = floor(n·p0 + z·√(n·p0·(1−p0)))`, z=1.6448.
   `stats_table.go`, `docs/binom-stattest.md`.
2. **Детерминизм без float** — ряды Тейлора для `exp` (см.
   [[Детерминизм — дисциплина консенсуса]]).

## Авто-выключатель против ложных срабатываний
Порог простоя динамический: базовый сетевой miss-rate + маржа, с кэпом (500‰). При
сетевом outage наказание **само отключается** — не карает всех за общую беду.

## Что теряет пойманный
INVALID/INACTIVE «липкие» в эпохе → слэш залога 10–20% + обнуление наград + падение
веса. См. [[Гибридный вес — база плюс залог]].

## Связи
- Как выбираются инференсы на проверку: [[Сид — подпись как источник нонсов]].
- Финансовые последствия: [[Гибридный вес — база плюс залог]].
- Почему всё детерминировано: [[Детерминизм — дисциплина консенсуса]].
