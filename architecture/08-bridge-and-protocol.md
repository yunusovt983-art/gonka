# 08 · EVM-мост и протокольная поверхность (Published Language)

> Часть A — cross-chain мост `gonka ↔ EVM`. Часть B — каталог proto-сообщений всех модулей цепи.
> Назад к [индексу](../ARCHITECTURE.md).

---

# Часть A — EVM-мост

## Назначение
Двунаправленный мост между нативной цепью Gonka и EVM-цепями (Ethereum, Polygon, Arbitrum, Optimism). Две оси активов:
1. **Нативный Gonka ↔ WGNK** («Wrapped Gonka», ERC-20 от того же контракта; 9 знаков под нативный).
2. **EVM ERC-20/ETH ↔ обёрнутый CW20 на Gonka** — депонированный EVM-токен минтится как CosmWasm CW20 на Gonka, и наоборот.

Все cross-chain авторизации подписаны **пороговой BLS-подписью** набора валидаторов текущей эпохи (та же DKG-группа, что и on-chain — см. [[BLS-порог — слот-взвешенный Shamir]]).

## Контракт `BridgeContract` (Solidity)
Один контракт — **и мост, и сам ERC-20 WGNK**.
- Стейт-машина: `ADMIN_CONTROL` (старт/failsafe) → `NORMAL_OPERATION`. Владелец — мультисиг.
- Ключ BLS-группы на эпоху (точка G2) в `epochGroupKeys[epochId]`; окно 365 эпох с авто-очисткой.
- **Переходы эпох последовательны** (`epochId == latest+1`), каждый новый ключ подписан *прошлым* ключом (криптографическая цепочка доверия). Genesis-эпоха 1 задаётся админом.
- Операции: `withdraw(WithdrawalCommand)` (ERC-20/ETH наружу; `tokenContract==address(this)` = ETH), `mintWithSignature(MintCommand)` (минт WGNK). Обе требуют BLS-подпись над `keccak256(abi.encodePacked(...))`.
- **Авто-burn:** перевод WGNK на адрес самого контракта сжигает токены (`WGNKBurned`) — это UX «отправь на мост, чтобы увести обратно».
- BLS-проверка через нативные **прекомпайлы BLS12-381 (EIP-2537)**.

**Формат хеша-сообщения (доменно-разделён):**
- Mint: `keccak256(abi.encodePacked(epochId, GONKA_CHAIN_ID, requestId, ETHEREUM_CHAIN_ID, MINT_OPERATION, recipient, amount))`
- Withdraw: то же с `WITHDRAW_OPERATION` + `tokenContract`.

Цепь строит ту же раскладку байт: keeper даёт хвост сообщения, BLS-подсистема предваряет `epochId(8)+gonkaChainId(32)+requestId(32)` (`msg_server_request_bridge_mint.go` / `_withdrawal.go`).

## Четыре потока

### 1. EVM → Gonka (минт обёрнутого CW20) — путь `BridgeExchange`
1. Юзер депонирует ERC-20/ETH на контракт (обычный `Transfer`).
2. **Релеер** (форк geth+prysm) ловит депозит; каждый валидатор Gonka шлёт `MsgBridgeExchange{originChain, contractAddress, ownerAddress, amount, blockNumber, receiptIndex, receiptsRoot}`.
3. Первая отправка создаёт `BridgeTransaction` (`BRIDGE_PENDING`), привязанную к epoch-группе; записывает власть отправителя.
4. Каждое следующее подтверждение валидируется против epoch-группы транзакции, дедуплится (`HasBridgeTransactionValidator`), власть суммируется.
5. При `TotalValidationPower ≥ totalEpochPower/2 + 1` → `BRIDGE_COMPLETED` (ровно раз) → `handleCompletedBridgeTransaction`: либо `HandleNativeTokenRelease` (если адрес — зарегистрированный мост), либо `GetOrCreateWrappedTokenContract` + `MintTokens`.

> **Что мешает поддельному минту:** нет единого доверенного минтера — нужно >50% власти валидаторов, каждый независимо ре-выводит транзакцию из данных EVM-receipt (`receiptsRoot/blockNumber/receiptIndex`). Повтор невозможен — tx ключуется содержимым `originChain_blockNumber_receiptIndex`; вторая отправка с расхождением полей отвергается («content mismatch — potential attack»).

### 2. Gonka → EVM (минт WGNK) — `RequestBridgeMint`
`MsgRequestBridgeMint` → проверка баланса/цепи/адреса → **атомарный перевод нативных в модуль `bridge_escrow`** → сборка BLS `SigningData` → `BlsKeeper.RequestThresholdSignature` → запись pending-refund → событие. Любой сбой **откатывает** эскроу. После готовой пороговой подписи кто угодно зовёт `mintWithSignature` на EVM.

### 3. Gonka → EVM (вывод ERC-20/ETH) — `RequestBridgeWithdrawal`
**Только для контрактов** (подписант — зарегистрированный CW20). Юзер сжигает обёрнутый токен на Gonka → CW20 зовёт `MsgRequestBridgeWithdrawal` → BLS `WITHDRAW_OPERATION` → пороговая подпись → `withdraw` на EVM.

### 4. WGNK → нативный (обратно)
Авто-burn WGNK на EVM → релеер ловит `WGNKBurned` → валидаторы шлют `MsgBridgeExchange` с адресом моста → при большинстве `HandleNativeTokenRelease` освобождает нативный Gonka из `bridge_escrow`.

## Авто-возврат при провале подписания
BLS-хуки (`module/bls_hooks.go`): `AfterThresholdSigningCompleted` чистит pending-refund;
`AfterThresholdSigningFailed` → `ProcessAutoRefundForFailedBridgeOperation`
(`bridge_pending_refund.go`): для mint — вернуть эскроу юзеру; для withdrawal — **ре-минт**
обёрнутых токенов юзеру. Поддержка авто-ретраев (`attempt`) до FAILED/EXPIRED. Юзер может
сам отменить (`MsgCancelBridgeOperation`); governance — форс-отмена/редирект.

## Модель безопасности (сводка)
- **Вход (EVM→Gonka):** порог >50% власти; дедуп по содержимому; привязка к receipt-root/block/index; детект «content mismatch»; завершение ровно раз.
- **Выход (Gonka→EVM):** BLS-порог живой эпохи; контракт — последовательная цепочка ключей, per-epoch `processedRequests[epoch][reqId]` (анти-replay), двойная привязка chain-id + домен операции, гейт `NORMAL_OPERATION`, 30-дневный таймаут → `ADMIN_CONTROL`.
- **Эскроу:** нативные заперты в `bridge_escrow`, движет только модуль inference.

> ⚠️ Доки контракта противоречат себе по размерам BLS-подписи/ключа (128/256 байт uncompressed в README vs 48/96 compressed в спеке vs 48-байт compressed G1 в `threshold_signing.proto`). Интегратору сверяться с реальным `BridgeContract.sol`, не с markdown.

---

# Часть B — Протокольная поверхность цепи

Каждый модуль публикует `Msg` (запись) и `Query` (чтение) через proto. Это его **Published Language** — контракт с внешним миром.

## `inference` (самый большой)
**Жизненный цикл инференса:** `StartInference`, `FinishInference`, `Validation`, `InvalidateInference`/`RevalidateInference`, `ClaimRewards`.
**Участники/права:** `SubmitNewParticipant`, `SubmitNewUnfundedParticipant`, `Add/RemoveParticipantsFromAllowList`.
**Proof of Compute:** `SubmitPocBatch` (v1), `SubmitPocValidationsV2`, `PoCV2StoreCommit` (коммит MMR-корня), `MLNodeWeightDistribution`, `SubmitSeed`; **делегирование:** `SetPoCDelegation`, `RefusePoCDelegation`, `DeclarePoCIntent`.
**Экономика/модели:** `SubmitUnitOfComputePriceProposal`, `RegisterModel`/`DeleteGovernanceModel` (on-chain governance моделей!), `CreatePartialUpgrade` (rolling-апгрейд без halt), `SubmitHardwareDiff`.
**Мост:** `BridgeExchange`, `RegisterBridgeAddresses`, `RequestBridgeMint`, `RequestBridgeWithdrawal`, `CancelBridgeOperation`, `GovernanceCancelBridgeOperation`, `RegisterWrappedTokenContract`, `RegisterLiquidityPool`, …
**Devshard:** `CreateDevshardEscrow`, `SettleDevshardEscrow` (state_root + host-stats + мульти-слотовые BLS-подписи), `SetDevshardRequestsEnabled`.
**Query (~80 RPC):** `Inference(All)`, `Participant(All)`, `GetRandomExecutor`, `CurrentEpochGroupData`, `GetCurrentEpoch`, `EpochPerformanceSummary*`, PoC-запросы (`PocBatchesForStage`, `AllPoCV2StoreCommitsForStage`, `PoCValidationSnapshot`, `PoCDelegation`), `TokenomicsData`, `SettleAmount(All)`, `GetModelPerTokenPrice`, статистика разработчика, `HardwareNodes`, `PartialUpgrade(All)`, мост/devshard-запросы.

## `bls`
**Запись:** `SubmitDealerPart`, `SubmitVerificationVector`, `RespondDealerComplaints`, `SubmitGroupKeyValidationSignature` (цепочка доверия эпох для EVM-контракта), `SubmitPartialSignature`, `RequestThresholdSignature`.
**Чтение:** `EpochBLSData` (полное состояние DKG), `SigningStatus`, `SigningHistory`. *Богатейший крипто-модуль: полный DKG + конвейер пороговой подписи, авторизующий все мостовые операции.*

## `collateral`
**Запись:** `DepositCollateral`, `WithdrawCollateral` (возвращает `completion_epoch`). **Чтение:** `Collateral(All)`, `UnbondingCollateral(All)`. *Публичного `MsgSlash` нет — слэшинг внутренний.*

## `streamvesting`
**Запись:** `TransferWithVesting`, `BatchTransferWithVesting` (вестинг по эпохам, дефолт 180). **Чтение:** `VestingSchedule`, `TotalVestingAmount`.

## `restrictions`
**Запись:** `ExecuteEmergencyTransfer`. **Чтение:** `TransferRestrictionStatus`, `TransferExemptions`, `ExemptionUsage`.

## `genesistransfer`
**Запись:** `TransferOwnership` (одноразово, всё-или-ничего). **Чтение:** `TransferStatus`, `TransferHistory`, `AllowedAccounts`, `TransferEligibility` (dry-run).

## `bookkeeper`
Только `UpdateParams` / `Params`. Вся учётная логика — внутренняя (keeper-уровень).

## Возможности, вскрытые proto (сверх очевидной документации)
- **On-chain governance моделей** (`RegisterModel`/`ModelsAll`) — набор обслуживаемых моделей управляется голосованием.
- **Частичные/rolling апгрейды** (`CreatePartialUpgrade` + `MLNodeVersion`) — апгрейд версии узла/API без halt цепи.
- **PoC-делегирование** — аккаунт может делегировать свой PoC по модели другому.
- **Devshard** — солидная, слабо документированная подсистема verifiable-compute расчёта.
- **Обучение:** в tx-сервисе **нет** training-сообщений (вырезаны в v0.2.12) — см. [09](09-testing-and-evolution.md).
- **IBC + DEX:** liquidity pools, торговые approve обёрнутых/IBC-токенов, миграция CW20 — first-class.

## Главные файлы
`proposals/ethereum-bridge-contact/{ethereum-bridge-contract.md,contracts/,bls.js,mint-wgnk.js}` · `inference-chain/x/inference/keeper/{msg_server_bridge_exchange,msg_server_request_bridge_mint,msg_server_request_bridge_withdrawal,bridge_pending_refund}.go`, `module/bls_hooks.go` · `bridge/script.sh` · proto: `inference-chain/proto/inference/*/{tx,query}.proto`, `.../bls/threshold_signing.proto`, `.../inference/bridge.proto`
