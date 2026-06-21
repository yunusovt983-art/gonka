---
title: Devshard gossip — живучесть при обходе
type: concept
tags: [gonka, devshard, gossip, recovery, liveness, p2p]
source: devshard/gossip/gossip.go, devshard/host/host.go
updated: 2026-06-20
---

# Devshard gossip — живучесть при обходе

> **Суть:** пользователь — единственный секвенсор канала, и он мог бы «обойти» хост,
> чтобы лишить его доли. Gossip + recovery между хостами этого не дают: обойдённый хост
> сам узнаёт diff'ы от пиров, проверяет авторизацию юзера, подписывает и распространяет
> подпись — оставаясь годным для кворума 2/3+1.

## 🗺️ Обзор
```mermaid
flowchart TB
    NOTE["Обойдённый хост сам узнаёт diff'ы от пиров и остаётся годным для кворума"]:::note
    USER["Пользователь-секвенсор<br/>мог бы обойти хост"]:::entry
    REC["Recovery-цикл ~60с<br/>ядро живучести"]:::core
    G1["Gate 1: я отстал?<br/>highestSeen > lastApplied"]:::coresub
    G2["Gate 2: меня обходят?<br/>тишина юзера 60с"]:::coresub
    PEERS["Пиры группы<br/>fetch · verify · gossip K=10"]:::adapter
    NOTE -.-> REC
    USER -.->|"молчит со мной"| G2
    REC --> G1
    REC --> G2
    G1 -->|"оба да"| PEERS
    G2 -->|"оба да"| PEERS
    PEERS -->|"со-подпись в кворум"| REC
    classDef core fill:#2e7d46,stroke:#86efac,color:#ffffff
    classDef coresub fill:#3a8d56,stroke:#bbf7d0,color:#ffffff
    classDef adapter fill:#1e293b,stroke:#475569,color:#e2e8f0
    classDef entry fill:#0f172a,stroke:#334155,color:#e2e8f0
    classDef note fill:none,stroke:none,color:#94a3b8
```

## 💻 Код (`devshard/gossip/gossip.go:318`)
```go
if highestSeen <= lastAppliedNonce {
	return
}

// Only trigger recovery if we haven't received a user request recently.
if !lastReq.IsZero() && time.Since(lastReq) < recoveryDelay {
	return
}
// ...
diffs, err := fetcher.GetDiffs(ctx, lastAppliedNonce+1, highestSeen)
```

## Два канала
- **nonce-gossip** → `K=10` случайным пирам: `{Nonce, StateHash, StateSig, SlotID}`.
- **tx-broadcast** → всем (с дедупом): «устаревшие tx редки и критичны».

## Recovery-цикл (~60с) — ядро живучести
```
Gate 1: я отстал?    highestSeen > lastApplied ?
Gate 2: меня обходят? юзер НЕ говорил со мной последние 60с ?
если оба да:
  fetch  GetDiffs(lastApplied+1 .. highestSeen) у пира
  apply  ПРОВЕРИВ user-подпись над каждым diff (плохая → abort)
  sign   подписать state root своими слотами
  gossip разослать СВОИ подписи K пирам
```
> «Молчащий обойдённый хост» → «вносящий со-подписант» за ~60с. Пользователь **не может
> подавить улику** (она уходит к пирам) и **не может рассчитаться без 2/3+1**.

## Безопасность (две защиты + одна дыра)
- **Только члены группы** госсипят (юзер исключён) → 403 иначе.
- **stateSig обязан восстанавливаться в заявленный слот** до попадания в `seen` —
  защита от **отравления seen-map** (иначе вброс `(nonce, fakeHash)` спровоцировал бы
  ложную эквивокацию — детект «один нонс, разный хеш» → HTTP 409).
- ⚠️ **Открыто:** mempool gossip DoS (`ProposedAt=0`) — злой член вбрасывает невалидную
  tx → она мгновенно «устаревает» → все придерживают подписи → стоп сессии.

## Здоровье хоста — почему gossip-сбои не карают
- **Карантин** (`ParticipantRequestLimiter`) бьёт только по **инференс-пути** (429/503→60мин,
  прочее→30мин). Сбои **gossip/verify RPC игнорируются** — здоровый хост остаётся в
  ротации. Карантинному шлют **ghost-пробу** (нонс сжигается локально), чтобы
  маршрутизация `nonce % size` не разъехалась.
- Расчёт достигает даже карантинных хостов через `WithoutAdmission()`-клиент.

## Связи
- Что подписывают: [[State root и кворум — расчёт за одну транзакцию]].
- Зачем нонс маршрутизирует: [[Нонс — тройной идентификатор]].
- Контекст: [[Devshard — платёжный канал инференса]]. Разбор: `architecture/10-deep-mechanisms.md` §B.
