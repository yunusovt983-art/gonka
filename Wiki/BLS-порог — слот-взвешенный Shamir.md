---
title: BLS-порог — слот-взвешенный Shamir
type: concept
tags: [gonka, bls, threshold, dkg, cryptography, bridge]
source: inference-chain/x/bls/keeper/
updated: 2026-06-20
---

# BLS-порог — слот-взвешенный Shamir

> **Суть:** мосту в EVM нужна подпись, доказывающая согласие >50% веса валидаторов,
> но без единого приватного ключа (его кража = катастрофа). Решение: per-epoch DKG
> создаёт общий BLS-ключ, *секрета которого не держит никто*, и `t-of-n` пороговое
> подписание. Хитрость — **слот-взвешивание** сворачивает «взвешенный порог» в простой
> счётный.

## 🗺️ Обзор
```mermaid
flowchart TB
    NOTE["Вес дискретизируется в равные слоты — взвешенный порог сводится к счётному t-of-n"]:::note
    WEIGHT["PoC-вес<br/>валидаторов"]:::entry
    SLOTS["I слот-долей<br/>диапазон ∝ весу"]:::coresub
    DKG["Slot-weighted DKG<br/>общий BLS-ключ"]:::core
    GATE["Гейт фазы<br/>slots > I/2"]:::coresub
    BRIDGE["EVM-мост<br/>threshold-подпись"]:::adapter
    WEIGHT -->|"AssignSlots"| SLOTS
    SLOTS --> DKG
    DKG -->|"каждый переход"| GATE
    DKG -->|"t = I − offset"| BRIDGE
    classDef core fill:#2e7d46,stroke:#86efac,color:#ffffff
    classDef coresub fill:#3a8d56,stroke:#bbf7d0,color:#ffffff
    classDef adapter fill:#1e293b,stroke:#475569,color:#e2e8f0
    classDef entry fill:#0f172a,stroke:#334155,color:#e2e8f0
    classDef note fill:none,stroke:none,color:#94a3b8
```

## 💻 Код (`inference-chain/x/bls/keeper/phase_transitions.go:79`)
```go
// Check if we have sufficient participation (more than half the slots)
if slotsWithDealerParts > epochBLSData.ITotalSlots/2 {
    // Sufficient participation - transition to VERIFYING
    params, err := k.GetParams(ctx)
    // ...
    currentBlockHeight := ctx.BlockHeight()

    epochBLSData.DkgPhase = types.DKGPhase_DKG_PHASE_VERIFYING
    epochBLSData.VerifyingPhaseDeadlineBlock = currentBlockHeight + params.VerificationPhaseDurationBlocks
    // ...
}
```

## Слот-взвешенный VSS (главная идея)
Секрет шарится не «по валидаторам», а на `I` равных **слот-долей** (живой genesis:
`i_total_slots = 100`, offset `50`; в коде есть комментарий «прод 1000», но это
нереализованная цель — фактически 100). Валидатор владеет непрерывным диапазоном
слотов ∝ своему весу.
```
порог t = I − offset = 50  →  восстановление требует ≥51 слот-доли = >50% веса
```
> «t+1 слотов» автоматически означает «>50% веса» — **не нужен отдельный слой весов**.
> Дискретизируй вес в равные слоты, и взвешенные пороговые схемы становятся простыми.

## Для чего подпись (не beacon случайности)
1. **Межэпоховая цепочка доверия** — валидаторы *прошлой* эпохи заверяют групповой
   pubkey *новой* (→ `DKG_PHASE_SIGNED`).
2. **Мост** — `RequestThresholdSignature` подписывает Ethereum-совместимый payload
   (`abi.encodePacked → keccak256 → hashToG1`), релеер отправляет в EVM-контракт.

## Стейт-машина DKG (по высоте блока)
`DEALING → VERIFYING → DISPUTING → COMPLETED → SIGNED` (или `FAILED`). Каждый гейт
требует `slots > I/2`. Агрегат — `EpochBLSData` по эпохе.

## Три неочевидных решения
- **Анти-O(N²) газ:** dealer-части лежат под суб-ключами по отправителю → N-й дилер
  платит константный газ (чинит баг, где поздние дилеры «дорожали»).
- **Exclusion вместо slashing:** единственная санкция за нечестность — исключение из
  набора валидных дилеров, а не штраф. Дешевле, когда «плохой» просто не нужен.
- **Детерминированная адъюдикация спора on-chain:** обвинённый дилер публикует
  открытую долю + ECIES-seed; цепь *повторно шифрует* и побайтово сравнивает.
  Воспроизводимый арбитраж без доверенной стороны.

## Связи
- Откуда вес для слотов: [[Proof of Compute 2.0 — власть есть вычисление]].
- Кто инициирует DKG: [[Эпоха — главные часы сети]].
- Принцип воспроизводимости: [[Детерминизм — дисциплина консенсуса]].
