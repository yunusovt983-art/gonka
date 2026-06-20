# 00 · Единая карта системы (все взаимосвязи)

> Одна мастер-диаграмма всех компонентов и потоков gonka + ключевые flow-схемы. Слой `v0.2.13`.
> Назад к [индексу](../ARCHITECTURE.md). Это визуальный «оглавитель» — каждый блок ссылается на свой документ.

---

## 1. Мастер-карта: компоненты и связи

```mermaid
flowchart TB
    USER["👤 Пользователь / Разработчик<br/>OpenAI-совместимый HTTP"]

    subgraph OFF["OFF-CHAIN"]
      direction TB
      subgraph DAPI["decentralized-api (dapi) — оркестратор · док 03"]
        BROKER["broker<br/>декларативный реконсилятор"]
        PHASE["chainphase<br/>фазовый трекер"]
        DISPATCH["event_listener<br/>new_block_dispatcher"]
        POCPIPE["PoC-конвейер<br/>MMR-коммит · off-chain валидатор"]
        TXMGR["tx_manager (NATS)<br/>батчинг · ретраи"]
        BWLIM["bandwidth limiter<br/>честная доля · док 11C"]
        STORE["payload / stats storage<br/>off-chain данные"]
      end
      subgraph ML["ml node (Python/GPU) · док 07"]
        APIPROXY["api: reverse-proxy<br/>least-connections"]
        VLLM["форк vLLM<br/>инференс + PoC v2 (k_dim=12)"]
        POWENG["pow: движок PoC v1<br/>расстояние на сфере"]
        TRAIN["train: DiLoCo<br/>⚠️ отвязан от цепи · док 09"]
      end
      subgraph DS["devshard — платёжный канал · док 04"]
        DCTL["devshardctl<br/>спекулятивный шлюз"]
        DHOST["devshardd (хосты)<br/>co-sign state root"]
        GOSSIP["gossip + recovery<br/>живучесть · док 10B"]
      end
      RELAYER["релеер<br/>форк Geth + Prysm · док 08"]
    end

    subgraph CHAIN["inference-chain — форк Cosmos SDK + CometBFT"]
      direction TB
      INF["x/inference — ЯДРО · док 01<br/>эпохи · PoC · settlement<br/>dynamic pricing · делегирование<br/>bridge · devshard settlement"]
      COL["x/collateral<br/>залог · slash · док 02"]
      BLS["x/bls<br/>DKG · threshold-подпись · док 02"]
      VEST["x/streamvesting<br/>вестинг · док 02"]
      REST["x/restrictions · док 02"]
      STK["x/staking (форк)<br/>SetComputeValidators"]
      GRP["x/group<br/>EpochGroup-веса"]
      GOV["x/gov · x/feegrant · x/bank"]
    end

    EVM["⛓️ EVM-мост (WGNK)<br/>BLS12-381 EIP-2537 · док 08"]
    CBFT["CometBFT<br/>консенсус блоков"]

    %% --- инференс ---
    USER -->|"chat/completions"| BROKER
    USER -.->|"стриминг (devshard)"| DCTL
    BROKER -->|"lock node"| APIPROXY
    APIPROXY --> VLLM
    BWLIM -. "throttle TA-пути" .-> BROKER

    %% --- dapi <-> chain ---
    DISPATCH <-->|"block events / Params"| INF
    PHASE --> DISPATCH
    POCPIPE -->|"MsgPoCV2StoreCommit<br/>MsgSubmitPocValidationsV2"| TXMGR
    BROKER -->|"StartPoC / InitValidate / InferenceUp"| ML
    ML -->|"артефакты PoC · инференс"| POCPIPE
    TXMGR -->|"Cosmos tx"| INF
    STORE -. "хеши on-chain, данные off-chain" .-> INF

    %% --- devshard ---
    DCTL -->|"Diff (user-sig)"| DHOST
    DHOST <-->|"nonce/tx gossip · recovery"| GOSSIP
    DHOST -->|"MsgSettleDevshardEscrow<br/>(кворум 2/3+1)"| INF
    DHOST -. "long-poll RuntimeConfig" .-> DAPI

    %% --- ядро дирижирует ---
    INF -->|"AdvanceEpoch · slash-policy"| COL
    INF -->|"InitKeyGen · RequestThresholdSig"| BLS
    INF -->|"AddVestedRewards"| VEST
    INF -->|"вес → ComputeResult"| GRP
    GRP --> STK
    INF -->|"SetComputeValidators"| STK
    STK --> CBFT
    REST -. "hook на bank" .-> GOV
    INF --> GOV

    %% --- мост ---
    RELAYER -->|"депозит EVM → MsgBridgeExchange"| INF
    BLS -->|"threshold sig (выход)"| EVM
    INF -->|"запрос подписи моста"| BLS
    RELAYER <--> EVM
```

**Как читать:** сплошная стрелка — основной поток (вызов/сообщение); пунктир — конфигурация/обязательства/побочный канал. Подписи на рёбрах — конкретные сообщения/действия. Номера доков ведут к разбору.

---

## 2. Поток власти: compute → consensus (суть PoC 2.0)

```mermaid
flowchart LR
    GPU["GPU считает PoC<br/>нонсы + расстояния"] -->|"Count + MMR-корень"| CALC["PoCWeightCalculator"]
    CALC -->|"Potential Weight"| COLL{"активация залогом<br/>BaseRatio 0.2 + collateral"}
    COLL -->|"Effective Weight"| GROUPW["вес в x/group<br/>(Metadata = pubkey)"]
    GROUPW -->|"делегирование N→1<br/>док 11A"| VP["per-model voting power"]
    GROUPW -->|"ComputeResult.Power"| SCV["SetComputeValidators<br/>(форк, PowerReduction=1)"]
    SCV -->|"voting power"| CMT["CometBFT + x/gov"]
    GUARD["Genesis Guardian<br/>усиление до зрелости · док 10C"] -. "перезапись power перед SCV" .-> SCV
    CAP["кап консенсуса 0.25"] -.-> SCV
```

---

## 3. Поток инференса (transfer-agent → executor → валидация)

```mermaid
sequenceDiagram
    autonumber
    participant U as Пользователь
    participant TA as dapi (Transfer Agent)
    participant EX as dapi (Executor)
    participant ML as ml node (vLLM)
    participant CH as chain

    U->>TA: chat/completions
    TA->>TA: bandwidth limiter (429 если перегруз)
    TA->>CH: MsgStartInference (async, escrow по PerTokenPrice)
    TA->>EX: forward + X-TA-Signature + PromptHash
    EX->>ML: inference (locked node, least-busy)
    ML-->>EX: SSE-стрим
    EX-->>U: проксирование (InferenceId в каждом event)
    EX->>CH: MsgFinishInference (async, ResponseHash, payload off-chain)
    Note over CH: позже — детерминированная выборка валидаций
    CH->>EX: другие участники ре-исполняют (logprobs > 0.99) → MsgValidation / SPRT
```

---

## 4. Жизненный цикл эпохи (PoC-цикл, сжато)

```mermaid
flowchart LR
    SP["StartOfPoC<br/>seed · группы"] --> GEN["генерация<br/>MMR-коммит"]
    GEN --> VAL["валидация PoC<br/>2/3 + guardian tie-break"]
    VAL --> END["EndOfPoCValidation<br/>ComputeNewWeights · settlement<br/>slash · BLS DKG · делегирование"]
    END --> SET["SetNewValidators<br/>цена unit · SetComputeValidators (H+2)"]
    SET --> CLAIM["ClaimMoney<br/>recovery валидаций → награды"]
    CLAIM --> SP
```

---

## 5. Devshard: жизненный цикл эскроу

```mermaid
flowchart LR
    OPEN["MsgCreateDevshardEscrow<br/>лочит amount · группа 16 слотов"] --> SEQ["пользователь-секвенсор<br/>Diff{nonce, txs, user-sig}"]
    SEQ -->|"nonce % 16 → хост"| EXEC["хост исполняет<br/>+ co-sign state root"]
    EXEC -->|"gossip + recovery"| PEERS["пиры собирают подписи<br/>док 10B"]
    PEERS --> SETTLE["MsgSettleDevshardEscrow<br/>пересчёт root + кворум 2/3+1"]
    SETTLE --> PAY["выплата Cost хостам<br/>остаток — пользователю"]
```

---

## 6. Поток моста (обе стороны)

```mermaid
flowchart TB
    subgraph IN["EVM → Gonka"]
      D1["депозит на контракт"] --> R1["релеер ловит"]
      R1 --> V1["валидаторы: MsgBridgeExchange"]
      V1 -->|">50% власти"| M1["минт CW20 / релиз нативного"]
    end
    subgraph OUT["Gonka → EVM"]
      Q2["MsgRequestBridgeMint/Withdrawal"] --> E2["эскроу в bridge_escrow"]
      E2 --> S2["BLS threshold sig"]
      S2 --> X2["mintWithSignature / withdraw на EVM"]
      S2 -. "провал → AfterThresholdSigningFailed" .-> RF["авто-возврат эскроу"]
    end
```

---

## 7. Навигатор: блок → документ

| Область | Документ |
|---|---|
| Ядро PoC, эпохи, settlement | [01](01-core-proof-of-compute.md) |
| collateral / bls / vesting / restrictions / genesistransfer / bookkeeper | [02](02-supporting-contexts.md) |
| dapi: broker, фазы, PoC-конвейер, инференс | [03](03-orchestration-dapi.md) |
| devshard: эскроу, спекулятивный прокси | [04](04-devshard-payment-channel.md) |
| Экономика V2 | [05](05-economics.md) |
| Каталог идей | [06](06-ideas-catalog.md) |
| ML-узел: математика PoC, vLLM, обучение | [07](07-mlnode-compute.md) |
| EVM-мост + proto-каталог | [08](08-bridge-and-protocol.md) |
| Testermint, апгрейды, судьба обучения | [09](09-testing-and-evolution.md) |
| Ценообразование, gossip/recovery, guardian | [10](10-deep-mechanisms.md) |
| Делегирование, анатомия апгрейда, лимитер | [11](11-advanced-subsystems.md) |
| Верификация точности | [REVIEW](../REVIEW.md) |
| Атомарные заметки | [Wiki/MOC — gonka](../Wiki/MOC%20—%20gonka.md) |
