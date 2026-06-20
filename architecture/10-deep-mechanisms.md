# 10 · Глубокие механизмы — pricing, gossip/recovery, guardian

> Три подсистемы, каждая тянет на отдельный разбор: динамическое ценообразование (EIP-1559 по моделям), devshard gossip+recovery (живучесть канала), genesis guardian (анти-захват).
> Назад к [индексу](../ARCHITECTURE.md).

---

## A. Динамическое ценообразование (EIP-1559 по моделям)

Per-model, per-block, per-token контроллер цены. Код: `inference-chain/x/inference/keeper/dynamic_pricing.go`. Проект-док: `proposals/tokenomics-v2/dynamic-pricing.md`.

### Контур управления
**Вход:** `BeginBlocker` зовёт `UpdateDynamicPricing(ctx)` раз в блок (`module/module.go:214`); ошибки логируются, но глушатся (блоки продолжают идти).

**Утилизация** (`dynamic_pricing.go:73-93`):
```
numerator   = averageLoadPerBlock         # скользящее среднее токенов/блок для модели
denominator = capacityPerBlock = capacity(tokens/сек) × DynamicPricingEstimatedBlockSeconds(=5)
utilization = numerator / denominator      # decimal, может быть >1.0
```
Скользящее окно (`rolling_window_state.go`) — кольцо `UtilizationWindowToBlocks(60с)/5с = 12 блоков` по умолчанию, **кормится в EndBlock** завершёнными инференсами (`PromptTokenCount+CompletionTokenCount`). То есть цена в блоке N отражает нагрузку до N−1.

**Формула корректировки** (`CalculateModelDynamicPrice`, `:135-231`):
```
lower=0.40, upper=0.60, elasticity=0.05, minPrice=1
if  lower ≤ U ≤ upper:  newPrice = currentPrice            # зона стабильности — без изменений
elif U < lower:         factor = 1 − (lower−U)·elasticity  # клампится ≥ 0.98
else (U > upper):       factor = 1 + (U−upper)·elasticity  # клампится ≤ 1.02
newPrice = max(floor(currentPrice · factor), minPrice)
```
**Кламп ±2%/блок выведен, не зашит:** `maxDeviation = 1−upper = lower = 0.40` ⇒ `1 ± 0.40·0.05 = 1.02 / 0.98`. При U=100%→+2%, U=0%→−2%, U=80%→+1%, U=20%→−1%.

### Capacity (важная деталь)
Кэшируется раз на эпоху (`CacheAllModelCapacities`, зовётся в `onSetNewValidatorsStage`). ⚠️ Проект-док обещает поле `total_throughput`, но **его нет** — код использует `TotalWeight` модельной подгруппы (сумму PoC-веса MLNode'ов) как прокси (эвристика «~1000 нонсов PoC ≈ 1000 ток/с»), с TODO добавить реальный throughput (`dynamic_pricing.go:352-365`). Capacity постоянна всю эпоху.

### Grace-период (бесплатный инференс)
`currentEpoch.Index ≤ GracePeriodEndEpoch (90)` → `handleGracePeriod`: до 90 все модели = `GracePeriodPerTokenPrice (0)`; ровно на 90 — сидируется `BasePerTokenPrice (100)`; после — нормальный алгоритм.

### Две цены и итоговая комиссия
| | **PerTokenPrice (динамическая)** | **UnitOfComputePrice (legacy)** |
|---|---|---|
| Каденция | каждый блок | каждую эпоху |
| Скоуп | per-model | сеть целиком |
| Источник | EIP-1559 алгоритм | взвешенная медиана предложений участников |
| **В комиссии?** | **Да** | **Нет (рудимент)** |

> **Поправка к [05](05-economics.md):** старая 3-факторная формула `Tokens × UnitsOfComputePerToken × UnitOfComputePrice` **заменена**. Реальная (`inference_state.go:200-230`):
> ```
> Cost   = (PromptTokens + CompletionTokens) × inference.PerTokenPrice
> Escrow = (MaxTokens   + promptTokens)      × inference.PerTokenPrice
> ```
> `UnitOfComputePrice` всё ещё считается каждую эпоху (взвешенная медиана, `epochgroup/unit_of_compute_price.go`), но **не имеет потребителя в пути комиссии** — вестигиальна.

**Фиксация цены:** `RecordInferencePrice` копирует `GetModelCurrentPrice(model)` в `inference.PerTokenPrice` при первом из Start/Finish (идемпотентно, `PerTokenPrice>0`). Эскроу резервируется по зафиксированной цене → изменение цены в полёте не влияет на текущий инференс.

### Рациональ и крайние случаи
- **Зона стабильности [0.40,0.60]** — dead-band против дёрганья от мелких флуктуаций.
- **Кламп ±2%** — ни один блок не может вздёрнуть цену; всплеск не сделает 10× мгновенно (анти-манипуляция).
- **Floor 1 ngonka** — против нулевой стоимости (деление/коллапс стимулов).
- **Нулевой трафик** → утилизация 0 → −2%/блок к полу 1. **Сатурация** → +2%/блок максимум.

---

## B. Devshard gossip + recovery — живучесть канала при обходе

Host-to-host координация, держащая off-chain платёжный канал живым (и подписываемым), когда пользователь-секвенсор обходит часть хостов. Код: `devshard/gossip/`, `devshard/host/`.

### Два канала gossip
- **Канал A — nonce-gossip K случайным пирам.** `GossipNonceRequest{Nonce, StateHash, StateSig, SlotID}` (SlotID = слот *подписавшего*). `K=10` (`gossip.go:60`): если пиров ≤K — всем, иначе `rand.Perm[:K]`.
- **Канал B — tx-broadcast ВСЕМ (с дедупом).** `BroadcastTxs` дедуп по `TxHash`; «устаревшие tx редки и критичны → всем».

### Детект эквивокации
`OnNonceReceived`: если `seen[nonce]` есть и хеш не совпал → **эквивокация** (HTTP 409). `checkStateConflict` пока только логирует (`TODO: slashing evidence`). Один нонс + тот же хеш → дубликат, подпись идёт в `AccumulateGossipSig` (сбор кворума).

### Recovery-цикл (~60с) — ключевой механизм
`tryRecovery` (`gossip.go:304-381`):
```
1. Gate 1 — я отстал?   if highestSeen ≤ lastAppliedNonce: return
2. Gate 2 — меня обходят? if time.Since(lastReq) < recoveryDelay(60с): return
                          # recovery ТОЛЬКО для хостов, которых юзер пропускает
3. fetch: GetDiffs(lastApplied+1, highestSeen)   # GET /diffs у пира
4. apply: ApplyRecoveredDiffs → ApplyDiff ПРОВЕРЯЕТ user-подпись над DiffContent,
          валидирует PostStateRoot; плохая подпись → abort
5. sign:  signIfAccepted → подписывает state root для всех своих слотов
6. gossip: рассылает СВОИ подписи K пирам
```
> Эффект: хост, с которым пользователь **никогда не говорит**, всё равно (a) узнаёт свежие user-подписанные diff'ы от пиров, (b) проверяет авторизацию пользователя, (c) ставит свою подпись, (d) распространяет её. «Молчащий обойдённый хост» → «вносящий со-подписант» за ~60с.

**Rebroadcast** устаревших SEEN-нонсов: **тик 30с**, порог устаревания `StaleTTL=120с`, один раз на запись (`gossip.go:61,256`).

### Два уровня здоровья хоста
- **ParticipantRequestLimiter** — жёсткий карантин (process-wide, по validator-адресу). 429/503 → **60 мин**; 404/403/transport-fail на инференсе / 3×EOF / 3×empty / stalled-winner → **30 мин**. ⚠️ Сбои **не-инференс RPC (gossip/verify) НЕ карантинят** (`participant_limiter.go:407`) — это и держит здоровый хост в ротации. Карантинный хост получает **ghost-пробу** (нонс «сжигается» локально, без HTTP, чтобы `nonce % size` не разъехался).
- **PerfTracker** — мягкий, per-escrow, per-slot; влияет только на спекулятивное решение, не блокирует.

### Безопасность gossip
- **Только члены группы** (пользователь исключён) — `isGroupMember`, иначе 403.
- **stateSig обязан восстанавливаться в заявленный слот** (`server.go:660-688`) перед попаданием в `seen` — защита от **отравления seen-map** (иначе можно было бы вбросить `(nonce, fakeHash)` и спровоцировать ложную эквивокацию).
- ⚠️ **Открытая дыра — mempool gossip DoS** (`ProposedAt=0`): злой член группы госсипит невалидную tx → она мгновенно «устаревает» → все хосты придерживают свои state-подписи → сессия может застопориться. Смягчено только ограничением членства.

### Как gossip+recovery гарантируют кворум 2/3+1 при частичной связности
1. **Аккумуляция подписей через gossip** — любой хост, услышавший подпись, складывает её в свой счётчик финализации (`AccumulateGossipSig`), проверив слот и хеш. Кворум может собраться на любом хорошо связанном хосте.
2. **Recovery даёт обойдённым хостам произвести свои подписи** (выше) — именно их не хватает near-quorum группе.
3. **Расчёт достигает карантинных хостов** — finalize/сбор подписей использует `WithoutAdmission()`-клиентов, обходя карантин, чтобы забрать уже произведённую подпись.

> Итог: пользователь как единственный секвенсор **не может подавить улику хоста** (она уходит к пирам через gossip) и **не может рассчитаться без 2/3+1 подписей** (честные хосты их придержат через grace/staleness, если юзер играет с включением).

---

## C. Genesis Guardian — вето без контроля на bootstrap

Анти-захват: пока сеть «незрелая» (мало совокупной power), назначенные bootstrap-валидаторы усиливаются так, чтобы держать **>1/3** (вето), но **<1/2 и <2/3** (не контроль), и авто-отключаются при зрелости.

> ⚠️ **Параметры эволюционировали через апгрейды.** Genesis-значения ≠ действующие на слое v0.2.13:

| Параметр | Genesis | Действует на v0.2.13 |
|---|---|---|
| Множитель `m` | 0.52 | **0.33334** (v0.2.13, `v0_2_13/upgrades.go:228`) |
| Доля guardian'ов `m/(1+m)` | ~34.2% (>1/3, вето) | **~25%** (≤1/3, уже не единоличное вето) |
| `NetworkMaturityThreshold` | 2 000 000 | **15 000 000** (v0.2.7, `v0_2_7/upgrades.go:3311`) |
| `NetworkMaturityMinHeight` | 0 | **3 000 000** (v0.2.7) |
| Кол-во guardian'ов | 3 | 3 |

### Три РАЗНЫХ механизма (не путать)
- **A. Усиление staking-power** (consensus VP) — `genesis_guardian_enhancement.go:168` (присваивание).
- **B. Резервирование слотов BLS DKG** — там же `:229`.
- **C. Тай-брейкер валидации PoC** («guardianProtection», `chainvalidation.go:350`) — не про power.

### A. Формула усиления власти
```
otherTotal = totalNetworkPower − totalGuardianPower         # только не-guardian power
totalEnhancement = otherTotal × m
if totalEnhancement < totalGuardianPower: NO-OP             # никогда не понижает
perGuardian = totalEnhancement / numGuardians              # поровну, округление вниз
guardian.Power = perGuardian                                # ПЕРЕЗАПИСЬ, не +=
```
Доля guardian'ов от нового тотала (все 3 на месте): `m·O / ((1+m)·O) = m/(1+m)`. При `m=0.52` → 34.2%; при `m=0.33334` (v0.2.13) → 25%.

**Зачем `m` чуть >0.5 (в genesis):** решая `m/(1+m) > 1/3` ⇒ `m > 0.5`. То есть 0.52 — почти минимальный множитель, дающий guardian'ам **вето (>1/3)** без контроля. v0.2.13 снизил до 25% и одновременно перенастроил кворум governance на 0.25/0.75 (`upgrades.go:223`).

### Гейт зрелости
`InNetworkMature = totalNetworkPower ≥ threshold && height ≥ minHeight` (множитель-дефолт — `types/params.go:151`). Проверяется **каждый переход эпохи** (`IsSetNewValidatorsStage`); т.к. height монотонен, при достижении зрелости усиление **выключается навсегда** → нет постоянной централизации.

### B. Резервирование слотов BLS DKG
> ⚠️ **Где это происходит (точность):** сам `x/bls` **веса-слепой** — `AssignSlots` (`dkg_initiation.go`) раздаёт слоты строго по percentage-весам всего набора участников, без понятия «guardian». Резервирование делает **inference-модуль**: `ApplyBLSGuardianSlotReservation` (`genesis_guardian_enhancement.go:221-229`) переформирует percentage-веса так, что guardian'ы получают долю `f = m/(1+m)`, и уже эти веса скармливаются `AssignSlots`.

Доля `f` слотов делится поровну между guardian'ами, остальное масштабируется не-guardian'ам по весу; раздача — `floor(ratio·ITotalSlots)` + largest-remainder. Порог реконструкции ключа `t = ITotalSlots − offset`. Гарантируя guardian'ам ≥f слотов, **групповой BLS-ключ нельзя сформировать/использовать без них** — захваченное большинство не прогонит DKG в одиночку.

### Кто guardian'ы и кто их меняет
3 genesis-адреса `gonkavaloper...`. v0.2.7 **мигрировал** адреса+порог+min-height в **governance-управляемые** `Params.GenesisGuardianParams` (меняются `MsgUpdateParams`). Но `GenesisGuardianMultiplier` и `...Enabled` остались в неизменяемых `GenesisOnlyParams` — множитель меняется только апгрейдом (как и сделал v0.2.13).

### Взаимодействие с капами власти
Усиление работает на `computeResults` **непосредственно перед** `SetComputeValidators` и **не прогоняется** через консенсусный кап `ApplyPowerCapping` (0.25, на `activeParticipants`). Поэтому guardian, поднятый до 34% (genesis), **превышал** кап 0.25 — другой путь, другая структура. Снижение v0.2.13 до 25% как раз выравнивает guardian-VP с капом 0.25.

---

## Поправки к прежним докам (этот проход)
- **05/Wiki guardian:** значения 0.52 / 2M / 34.2% — это **genesis**; на слое v0.2.13 действуют **0.33334 / 15M (+minHeight 3M) / ~25%**. Исправлено в [05](05-economics.md).
- **05 формула комиссии:** заменена на `Tokens × PerTokenPrice` (выше, §A).
- **Devshard rebroadcast:** порог staleness 120с, не 30с (30с — лишь тик).

## Главные файлы
Pricing: `keeper/{dynamic_pricing,rolling_window_state,query_dynamic_pricing}.go`, `calculations/inference_state.go:200-230`, `epochgroup/unit_of_compute_price.go`, `module.go:214,852` · Gossip: `devshard/gossip/{gossip,doc}.go`, `devshard/host/{host,staleness}.go`, `cmd/devshardctl/participant_limiter.go`, `devshard/transport/{client,server}.go` · Guardian: `module/genesis_guardian_enhancement.go`, `keeper/params.go:161`, `bls/keeper/dkg_initiation.go`, `app/upgrades/v0_2_7/upgrades.go:3303`, `app/upgrades/v0_2_13/upgrades.go:153-230`
