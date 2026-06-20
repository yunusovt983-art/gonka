# 01 · Core Domain — Proof of Compute 2.0 (`x/inference`)

> Ограниченный контекст: **`inference-chain/x/inference`**. Это сердце Gonka.
> Назад к [индексу](../ARCHITECTURE.md).

## Назначение контекста (одно предложение)

`x/inference` задаёт часы эпох, проводит Proof-of-Compute для избрания валидаторов с весом = compute, оценивает и эскроует запросы инференса, статистически валидирует их корректность, минтит и распределяет награды по «биткоин»-кривой и дирижирует всеми остальными модулями.

> Сквозная тема: **`x/inference` владеет таймингом и политикой эпох; модули-спутники — исполнители**, которых он вызывает на границе эпохи внутри `EndBlock` (`AdvanceEpoch`, `Slash`, `AddVestedRewards`, `InitiateKeyGenerationForEpoch`, `RequestThresholdSignature`).

---

## 1. Главные часы — модель эпохи

Вся цепь крутится вокруг одного якоря — `PocStartBlockHeight` эпохи. **`EpochContext`** (`types/epoch_context.go`) превращает относительные смещения из `EpochParams` в абсолютные высоты блоков для каждой стадии. Поддерживаются три указателя (`types/epoch.md`):

- **Effective / Current** — активные сейчас валидаторы.
- **Upcoming** — готовится через PoC.
- **Previous** — прошлая.

Таймлайн стадий внутри эпохи (`epoch_context.go:120-192`):

```
StartOfPoC → PoCGenerationWindDown → EndOfPoCGeneration
   → StartOfPoCValidation → PoCValidationWindDown → EndOfPoCValidation
      → SetNewValidators → ClaimMoney → NextPoCStart
```

Эпоха 0 — особая (без PoC, всегда `InferencePhase`).

---

## 2. Стейт-машина `EndBlock` — доменный дирижёр

`module/module.go:368-568`. Каждый блок `EndBlock` выполняет рутину (обработка завершённых инференсов, экспирация с возвратом эскроу, прунинг, отслеживание апгрейдов), затем проверяет триггеры стадий в строгом порядке:

1. **`IsEndOfPoCValidationStage` → `onEndOfPoCValidationStage`** (`module.go:619-822`) — формирование эпохи целиком:
   - `AdvanceEpoch` для collateral и streamvesting;
   - **settlement** аккаунтов эффективной эпохи;
   - **`ComputeNewWeights`** для будущей эпохи;
   - назначение моделей участникам;
   - применение штрафов/залога/капа власти;
   - запись `ActiveParticipants`, наполнение группы будущей эпохи;
   - **запуск BLS DKG** для новой эпохи.
2. **`IsSetNewValidatorsStage` → `onSetNewValidatorsStage`** (`module.go:835`) — вычисление цены unit-of-compute (взвешенная медиана), перенос upcoming→effective, переключение индекса эффективной эпохи. **Новый набор валидаторов активируется на H+2** — сознательный буфер в 2 блока, чтобы dapi успел подгрузить модели.
3. **`IsStartOfPocStage`** — создание следующей upcoming-эпохи + её Cosmos `x/group` группы + подгрупп по моделям, выборка «сохраняемых» узлов, фиксация таймстемпов старта генерации.

Отдельный dirty-flag (`currentEpochGroup.IsChanged`) триггерит `Staking.SetComputeValidators` на блок позже наполнения группы.

---

## 3. Подпись механизма: PoC-вес → consensus power

Это и есть суть PoC 2.0. **Стейкинг токенов не участвует.** Трасса (`module/chainvalidation.go`, `epochgroup/`):

1. Участники доказывают GPU-compute → `PoCWeightCalculator.Calculate()` → `ActiveParticipant.Weight`.
2. Вес кладётся в Cosmos **`x/group`** как вес члена; `Metadata` члена = **ed25519 pubkey валидатора**.
3. `GetComputeResults` декодирует членов группы в `[]ComputeResult{Power, ValidatorPubKey, OperatorAddress}` (`epoch_group.go:390-422`).
4. **`Staking.SetComputeValidators`** (в форке `gonka-ai/cosmos-sdk`) бондит валидатора на каждый результат с **voting power CometBFT = PoC Power напрямую**, минуя `MaxValidators` и бондинг токенов.

### EpochGroup — переиспользование `x/group` (`epochgroup/`)

`EpochGroup` оборачивает группу+политику `x/group` на эпоху. **Два уровня:** root-группа (`ModelId==""`, все участники — источник валидаторов) и подгруппы по моделям (члены, поддерживающие модель). Политика группы — фиксированный `PercentageDecisionPolicy("0.50", 4m)`: порог >50% по весу для on-chain голосований об инвалидации/ревалидации инференса.

`EpochGroupData` (ключ `(EpochIndex, ModelId)`) — сериализованное состояние: `ValidationWeights[]`, `TotalWeight/TotalThroughput`, `UnitOfComputePrice`, `ModelSnapshot`, `MemberSeedSignatures`. Членство дублируется: и в `x/group`, и в `ValidationWeights`. Хак: поле `Metadata` группы используется как dirty-флаг `"changed"/"unchanged"` для триггера ре-бондинга.

---

## 4. Механизм Proof of Compute

- **Сид/нонс:** участник единожды за эпоху шлёт `MsgSubmitSeed` — secp256k1-**подпись** (on-chain «сид» — это сама подпись, `message_submit_seed.go`). GPU-узлы off-chain выводят из неё детерминированный поток нонсов. Нет сида → нет веса.
- **V1 (legacy):** нонсы + распределение выходов грузились on-chain в `PoCBatch`.
- **V2 (актуально):** on-chain коммитятся только `Count` + 32-байтный корень Merkle `RootHash` на модель (`PoCV2StoreCommit`); артефакты — off-chain. Разбивка по узлам — `MLNodeWeightDistribution` (сумма = `Count`). Газ метрируется (`BaseValidationGas + GasPerPocCount·Δcount`), счётчик монотонный, один коммит на блок.
- **Валидация:** другие участники тянут артефакты и голосуют (`PoCValidationV2`, `ValidatedWeight>0` = валидно). Принятие требует **супербольшинства 2/3** (по слотам или по весу), с **единогласием guardian'ов как тай-брейком** (`chainvalidation.go:268-400`).
- **Вес:** `claimedWeight = Count × timeNormalizationFactor` — нормализация по реальному времени генерации, чтобы ранний старт не накручивал вес.
- **Confirmation PoC** (`module/confirmation_poc.go`): *случайно запускаемое* среди эпохи перепрувание (вероятность `ExpectedConfirmationsPerEpoch / windowLength` против `DeterministicFloat(prevBlockHash)`), чтобы поймать тех, кто выиграл вес и перестал обслуживать. Может **только понизить** confirmation-вес участника (min-take) и кормит `ConfirmationPoCRatio` (коэф. отклонения 0.909).

---

## 5. Валидация инференса — SPRT (статистическое ядро)

Корректность AI-выходов проверяется **повторным исполнением + последовательным критерием Вальда (SPRT)** на потоке исходов валидаций для каждого участника (`calculations/sprt.go`, `status.go`):

- Два SPRT на участника: **Invalidation SPRT** (H0 = «хорошая» доля `p0=FalsePositiveRate`, H1 = «плохая» `p1=BadParticipantInvalidationRate`) → INVALID; **Inactivity SPRT** (доли простоя) → INACTIVE.
- `LLR += failures·ln(p1/p0) + passes·ln((1-p1)/(1-p0))`; решение: `LLR ≥ H` → нарушитель, `LLR ≤ -H` → оправдан, иначе — продолжаем выборку. Порог `H = 4`. Плюс «триппроволока» по подряд идущим отказам (`FalsePositiveRate^N < QuickFailureThreshold = 1e-6`).
- Вся математика детерминирована (ряды Тейлора для `exp`/`tanh`, без `math.*`) ради консенсуса между узлами.
- **Выборка** (`should_validate.go`): каждый претендент детерминированно (`DeterministicFloat(seed, inferenceId)`) решает, какие инференсы перевалидировать — с весом по своей доле власти и **обратно** репутации исполнителя (доверенных проверяют реже). Целевая доля валидаций растёт с трафиком (`min_validation_average.go`).
- **Флоу инвалидации** использует парные `x/group`-предложения (`MsgInvalidateInference`/`MsgRevalidateInference`) с согласием группы-политики, а не единолично. `maximum_invalidations.go` ограничивает число одновременных открытых инвалидаций: `floor(max(min, M·W·(R/100)·tanh(I/C)))`.

> **Проверяемость выборки:** валидатор выбирает инференсы приватным секретным сидом; при claim сид раскрывается и цепь *повторяет* `ShouldValidate`, проверяя, что валидатор сделал ровно положенное. Пропуск → `ErrValidationsMissed`, claim отклонён. Это убивает «вишенкинг» лёгких валидаций (`docs/specs/inference-validation-flow.md`).

---

## 6. Стейт-машина статуса участника

Enum: `ACTIVE / INACTIVE / INVALID / UNCONFIRMED` (RAMPING устарел). INVALID/INACTIVE «липкие» в пределах эпохи (оживляет только сброс на старте эпохи). `UpdateParticipantStatus` — *единственный* мутатор статуса; при `→INVALID` слэшит залог (`SlashFractionInvalid=0.20`), пишет исключение, ужимает репутацию (`EpochsCompleted *= InvalidReputationPreserve`) и убирает из всех epoch-групп. При `→INACTIVE` — аналогично с долей простоя (0.10).

---

## 7. Settlement и «биткоин»-награды

`accountsettle.go`, `bitcoin_rewards.go`. **Модель монет:** **WorkCoins** = плата пользователя за инференс (из эскроу, не минтится); **RewardCoins** = свежеминтированная субсидия по весу PoC. Метки суб-аккаунтов: `owed`/`settled`/`balance`.

- **Эмиссия:** непрерывный экспоненциальный спад (без дискретных халвингов): `reward = InitialEpochReward · exp(DecayRate)^epochsSinceGenesis`. Дефолты кода: `InitialEpochReward=285 000 gonka`, `DecayRate=-0.000475`. Интеграл точно = **600M gonka** (`StandardRewardAmount`) — жёсткий потолок, у предела включается пропорциональный scale-down. *(Живой genesis поднял до 323K/680M — см. [05](05-economics.md).)*
- **Распределение:** `RewardCoins[p] = floor(weight[p] · epochReward / totalFullWeight)`. **Критический инвариант — нет перераспределения:** знаменатель *не* перенормируется после капа/инвалидации/простоя; любая «потерянная» доля (и остаток от целочисленного деления) уходит в **gov-аккаунт**, не соседям.
- **Кап власти 30%** на reward-веса (послаблено для малых сетей: 1→100%, 2→50%, 3→40%).
- **Атомарность** через `CacheContext`: минт + обновление токеномики + перевод в gov + per-participant записи коммитятся вместе.
- Одна живая `SettleAmount` на участника, клеймится только для `currentEpoch-1`; просроченные сметаются в gov.

---

## 8. Динамическое ценообразование (`dynamic_pricing.go`)

Две цены:

- **PerTokenPrice** — то, что реально платит пользователь. Per-block контроллер обратной связи по утилизации с зоной стабильности `[0.40, 0.60]`, эластичностью 0.05, клампом ±~2%/блок. Бесплатно в grace-период (эпоха ≤ 90).
- **UnitOfComputePrice** — **взвешенная медиана** (по PoC-весу) предложений участников `MsgSubmitUnitOfComputePriceProposal`, пересчёт на границе эпохи.

---

## 9. Жизненный цикл сущности Inference

`Inference` (`types/inference.pb.go`), статусы: `STARTED → FINISHED → VALIDATED / INVALIDATED / VOTING / EXPIRED`.

- `StartInference` эскроует `(maxTokens+promptTokens)·PerTokenPrice` и ставит таймаут.
- `FinishInference` пишет реальную стоимость, перераспределяет работу через `ShareWork` (исполнитель + валидаторы), ставит в очередь на валидацию.
- Цепочка подписей нескольких сторон (Developer / TransferAgent / ExecutorAgent) проверяется в `signature_validate.go`.
- Просроченные инференсы авто-возвращают эскроу в Begin/EndBlock.

---

## 10. Прочие подсистемы ядра

- **PoC-делегирование:** держатель веса прошлой эпохи, не запускающий модель, может делегировать consensus-вес прямому члену модельной группы (per-model, N→1). Штрафы за отказ/неучастие копятся аддитивно (кап 1.0).
- **Genesis guardian:** пока сеть незрелая (`totalNetworkPower < NetworkMaturityThreshold = 2 000 000`), назначенные bootstrap-валидаторы получают усиление власти (`otherTotal × multiplier 0.52`, поровну) и резервируют долю слотов BLS-DKG (~34%) — анти-коллапс на старте. Авто-отключается при зрелости.
- **Назначение моделей:** governance-регистрируемые `Model` (с `validation_threshold`, `throughput_per_nonce`, `units_of_compute_per_token`); MLNode жадно назначается первой поддерживаемой модели; детерминированная выборка резервирует ~50% веса под PoC-слот.
- **Bridge:** cross-chain минт/вывод на EVM-цепи, авторизуется BLS-порогом (см. [02](02-supporting-contexts.md)); авто-возврат при провале подписания.

---

## Агрегаты, инварианты, доменные сервисы (тактический срез)

| Тип DDD | Элемент | Инвариант |
|---|---|---|
| **Aggregate root** | `EpochGroupData` (ключ `EpochIndex,ModelId`) | Сумма `ValidationWeights` = `TotalWeight`; членство синхронно с `x/group`. |
| **Aggregate root** | `Inference` | Монотонный переход статусов; эскроу ≥ итоговой стоимости; возврат остатка. |
| **Entity** | `ActiveParticipant` | Статус меняется только через `UpdateParticipantStatus`. |
| **Value object** | `ComputeResult{Power, PubKey, Operator}` | Power из PoC, не из токенов. |
| **Domain service** | `EpochContext` | Чистое отображение высоты блока → стадия. |
| **Domain service** | `PoCWeightCalculator`, SPRT-калькуляторы | Детерминированы, без float. |
| **Policy / Saga** | `EndBlock` stage-machine | Сбой **соседнего** модуля глушится (`epoch_error`, `module.go:633,662,743`); сбой **ядра эпохи** валит цепь (`return err`, `module.go:385,474,491,500,507`). |

## Расхождения «доки ↔ код» (на этом слое)

- `calculations/status.md` описывает старую z-score модель — в коде уже SPRT.
- Часть терминологии наград в proposals говорит «burned», фактически средства идут в **gov-аккаунт**.
- **`AlphaThreshold` дефолт = 0.0** (`params.go:306`) несмотря на комментарий «70% minimum ratio» — флип статуса по confirmation-PoC-ratio через этот порог по умолчанию **выключен**.
- Два разных капа власти: консенсусный = `MaxIndividualPowerPercentage` 0.25 (`power_capping.go`), наградный = жёстко зашитые 0.30 (`bitcoin_rewards.go:277`). Здесь корректно «30%» для наград; в [05](05-economics.md) исправлено разделение.
