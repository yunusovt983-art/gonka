# Review исследования — верификация против кода

> Состязательная проверка ранее написанной архитектуры ([ARCHITECTURE.md](ARCHITECTURE.md), [`architecture/01..06`](architecture/)) против исходников `repo/`. Слой `8a35022` (v0.2.13).
> Цель проверки — **найти ошибки**, а не подтвердить. Все ссылки — относительно `repo/`.

## Итог одной строкой

Из ~30 ключевых утверждений **подтверждены точь-в-точь ~24**, **исправлены 4**, **2 в принципе непроверяемы** (форк вне репозитория). Числовые параметры (награды, залог, ценообразование, SPRT, devshard) оказались точными до цифры. Найдены 4 реальных бага в доках — все исправлены в этом проходе.

---

## ✅ Подтверждено (точь-в-точь)

| Утверждение | Доказательство |
|---|---|
| SPRT: `H=4`, `BadParticipantInvalidationRate=0.20`, `QuickFailureThreshold=1e-6`, `FPR=0.05` | `x/inference/types/params.go:226,240-247` |
| Награды: `InitialEpochReward` 285K код / 323K genesis; `DecayRate −0.000475`; cap 600M/680M | `params.go:336-337,147`; `genesis.json:5100,5154` — интеграл `285000/0.000475 = 600M` точен |
| Залог: `BaseWeightRatio 0.2`, `CPWU` 1/4.2, `SlashInvalid 0.20`, `SlashDowntime 0.10`, grace 180 | `params.go:324-329`; `genesis.json:5076-5093` |
| Ценообразование: зона `[0.40,0.60]`, эластичность `0.05`, free-grace эпоха 90 | `genesis.json:5113-5135` |
| Devshard: `GroupSize=16`, кворум `2·gs/3+1`, `nonce==inference_id`, `executor=group[id%len]`, `phase_byte=0x02` зашит, `MaxActiveNonce=maxNonce−(gs+1)` | `params.go:100`; `devshard_settlement.go:27,47,122,187,208`; `machine.go:378,697`; `max_nonce.go:11,25` |
| BLS-порог = `> ITotalSlots/2`; 4 фазы DKG с `DISPUTING` | `phase_transitions.go:80,155,176,251`; `types.pb.go:34-46` |
| H+2 буфер активации валидаторов | `module.go:460-481` |
| `PercentageDecisionPolicy("0.50",4m)`, confirmation-coeff `0.909`, EpochLength 40 | `epoch_group.go:109`; `confirmation_poc.go:18`; `params.go:205` |

**Отдельно отмечено как «неочевидно, но верно»:** точность интеграла спада (до цифры); `inference_id == nonce` действительно *принуждается* (`machine.go:378`), а не просто конвенция; свойство `phase_byte=0x02` (любое нефинализированное состояние хешируется иначе и отвергается) — ровно как описано.

---

## 🔧 Исправлено (реальные баги доков)

| # | Что было не так | Как теперь |
|---|---|---|
| 1 | **Один кап власти 0.25** на всё | Два РАЗНЫХ капа: консенсус = `MaxIndividualPowerPercentage` 0.25 (`power_capping.go:40`); награды = жёстко зашитые **0.30** (`bitcoin_rewards.go:277`). Исправлено в [05](architecture/05-economics.md) и [01](architecture/01-core-proof-of-compute.md). |
| 2 | **BLS `I=1000` в проде** | Живой genesis: `i_total_slots = 100`, offset `50`. «1000» — лишь комментарий-цель в коде. Исправлено в [02](architecture/02-supporting-contexts.md), ARCHITECTURE, Wiki. |
| 3 | Форк `product-science/cosmos-sdk` | Реальная зависимость `go.mod:7`: `gonka-ai/cosmos-sdk v0.53.3-ps17-observability`. `cosmos_changes.md` сам устарел. Исправлено в [05](architecture/05-economics.md). |
| 4 | **«Ошибки не валят цепь»** (слишком абсолютно) | Глушатся только ошибки **соседних** модулей (`module.go:633,662,743`); сбой **ядра эпохи** всё же `return err` и валит цепь (`module.go:385,474,491,500,507`). Исправлено в ARCHITECTURE §3 и [01](architecture/01-core-proof-of-compute.md). |

---

## ⚠️ Непроверяемо в этом checkout

- **Перепрошивка staking/slashing форка** (`PowerReduction=1`, обход бондинга в `Delegate`, несжигающий `Slash`, ручной `TotalBondedTokens`). Форк — внешний модуль, исходников нет в `repo/`. Утверждения держатся **только на прозе `docs/cosmos_changes.md`**. В репо виден лишь *вызов* `SetComputeValidators` (`module.go:560`) и тест бондинга 101-го валидатора сверх `MaxValidators=100` (`app/tally_integration_test.go`). Помечено в [05](architecture/05-economics.md).

---

## 🕳️ Пробелы, которые покрыты в этом проходе (новые доки)

Review выявил под-охваченные зоны → дописаны три новых документа:

- [07 · ML-узел (Python)](architecture/07-mlnode-compute.md) — реальная математика PoC (расстояние на сфере), **две реализации PoC (v1/v2)**, валидация, vLLM-сервинг.
- [08 · Мост и протокол](architecture/08-bridge-and-protocol.md) — EVM-мост end-to-end, BLS-авторизация, полный каталог proto-сообщений (Published Language).
- [09 · Тесты и эволюция](architecture/09-testing-and-evolution.md) — Testermint, upgrade-каденция (13 апгрейдов), **обучение: построено и удалено в v0.2.12**.

### Самое важное открытие
**Обучение (DiLoCo) — не roadmap и не live, а УДАЛ�ённая фича.** README репозитория всё ещё рекламирует geo-distributed training, но on-chain координация обучения была **жёстко вырезана в v0.2.12** (`proposals/training-removal-v0.2.12/`, `keeper/training_state_cleanup.go`). ML-движок DiLoCo (`mlnode/packages/train`) реален и работает автономно, но не имеет связи с цепью. Это прямое противоречие README → зафиксировано в [09](architecture/09-testing-and-evolution.md).

---

## Второй проход — review документов 07–11

Состязательная верификация новых доков. Большинство высокоставочных утверждений (две реализации PoC, distance-on-sphere, EIP-2537 мост, кворум >50%, динамическое ценообразование, gossip K=10/recovery 60s, guardian m 0.52→0.33334, bandwidth-лимитер) — **подтверждены**. Найдены и **исправлены 3 существенные ошибки**:

| # | Было | Стало | Доказательство |
|---|---|---|---|
| 1 | DelegationParams после v0.2.12 = 0 («слой дремлет») | **Активны:** 0.1/0.15/0.05/0.75/0.3 | `v0_2_12/upgrades.go:651-661` |
| 2 | `RegisterMigration` no-op «с версии 7» | **no-op с версии 8**; v7 делает реальную работу | `app/upgrades.go:88-96` |
| 3 | Баунти $34 400 | **$35 200** (сумма 13 позиций) | `v0_2_12/upgrades.go:61-97` |

Плюс мелкие: убран неверный выпад про `doc.go` (StaleTTL), правки cite-drift (`types/params.go:151`, `:168`), смягчена формулировка «pow=v1», уточнено что `x/bls` веса-слепой (резервирование слотов — в inference-модуле).

### ⚠️ Где сам review ошибся (перепроверено по коду)
Два «correction» агента-ревьюера оказались неверны — я подтвердил исходные доки прямой проверкой:
- **Slot-reservation guardian'ов СУЩЕСТВУЕТ** — `ApplyBLSGuardianSlotReservation` (`genesis_guardian_enhancement.go:221`). Ревьюер грепнул только `x/bls` (где guardian-концепта правда нет) и сделал ложный вывод. Доку уточнил, не убрал.
- **ProposedAt=0 mempool DoS ЕСТЬ** в `devshard/docs/attacks.md:56-60` под `[TODO]`. Ревьюер его не нашёл. Дока корректна.

Урок: даже состязательный агент может выдать ложный «refute» из-за узкой области поиска — финальное слово за прямой проверкой кода.
