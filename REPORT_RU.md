# Da Vinci DKG: аудит `PartialDecrypt` и `DecryptCombine`

Публичный технический кейс по проверке того, что zero-knowledge circuits реально гарантируют, как их statements связаны с кодом узла и контрактом и какие constraints можно удалить без изменения функциональной relation.

**Ревизия:** `vocdoni/davinci-dkg` commit [`9ec5ff3`](https://github.com/vocdoni/davinci-dkg/tree/9ec5ff39be536871584e1b0a13b76d83c51078c0), 20 октября 2026 года.  
**Дата проверки:** 7 ноября 2026 года.  
**Итог:** подтверждённой эксплуатируемой уязвимости в рассмотренном пути не обнаружено. Найдены и проверены существенные возможности упростить `PartialDecrypt`; `DecryptCombine` проверен, включая контрактные привязки.

## Executive summary

| Результат | Классификация | Подтверждение |
|---|---|---|
| Внешний Groth16 уже напрямую доказывает `D_i=[d_i]G` и `σ_i=[d_i]C₁`; вложенный Chaum–Pedersen/DLEQ повторяет ту же relation | подтверждённая оптимизация | блок удалён в экспериментальной копии; настоящий replay одного proof с изменёнными public inputs отвергается |
| Облегчённый `PartialDecrypt`, одновременно удаляющий DLEQ, избыточные on-curve checks результатов и условно избыточный range-check `Secret`, уменьшился с `26 194` до `5 208` constraints | подтверждённый совокупный результат эксперимента | package tests проходят; node компилируется; локальный benchmark ускорился примерно `4.4×` |
| Range-check `Secret≤ℓ−1` существует в исходном circuit, но функционально избыточен, если `C₁` обязательно проверен на принадлежность рабочей подгруппе | условная оптимизация, не уязвимость | `d_i` и `d_i+ℓ` дают те же `D_i` и `σ_i` на точках порядка `ℓ` |
| Отдельные on-curve checks для результатов уже проверяемого scalar multiplication дублируют более сильные constraints | подтверждённая малая оптимизация | отдельный эксперимент: `26 194→26 178`; честный proof принят, подстановка `(0,0)` отвергнута в Groth16 и PLONK |
| `DecryptCombine` корректно фиксирует интерполяционные веса, итоговую формулу и привязку к сохранённым ciphertext/partial decryptions | проверенный инвариант | чтение node/circuit/contract и успешные package tests |
| `LagrangeCoefficients` не range-checked и допускают несколько представителей одного класса modulo `ℓ` | informational; сейчас функционально безопасно | веса после маски `0/1` сразу действуют на точки подгруппы; арифметики двух закодированных весов в `𝔽_r` до группового действия нет |

Числа `26 194→5 208` и benchmark относятся ко всей облегчённой экспериментальной relation, а не измеряют вклад одного DLEQ-блока изолированно.

## Scope и модель угроз

В scope вошли:

- `circuits/partialdecrypt`: assignment, witness builder, circuit и replay-binding;
- `circuits/decryptcombine`: нативная интерполяция, witness builder, circuit и transcript;
- путь узла от сохранённых данных до создания proof;
- локальный runtime Groth16;
- контрактные привязки ciphertext и partial decryptions и вызов on-chain verifier.

Основные кодовые точки:

- [`partialdecrypt/circuit.go:34–99`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/circuits/partialdecrypt/circuit.go#L34-L99);
- [`partialdecrypt/witness.go:27–137`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/circuits/partialdecrypt/witness.go#L27-L137);
- [`decryptcombine/circuit.go:76–210`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/circuits/decryptcombine/circuit.go#L76-L210);
- [`decryptcombine/witness.go:34–167`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/circuits/decryptcombine/witness.go#L34-L167);
- [`node/decrypt.go`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/node/decrypt.go);
- [`circuits/artifacts.go:552–671`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/circuits/artifacts.go#L552-L671);
- [`DKGManager.sol`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/solidity/src/DKGManager.sol).

Рассматривался злонамеренный prover/combiner, способный выбирать private witness, public calldata и пытаться переиспользовать proof в другом контексте. Доверенными предпосылками считались корректность gnark/Groth16, proving/verifying keys и on-chain verifier, а также явно отмеченные системные инварианты.

Вне scope: полный аудит DKG contribution/finalize, реализация кривой и Groth16, trusted setup, весь Solidity-контракт, сетевой consensus/liveness, gas/DoS и сторонние реализации node.

## Минимальная математика протокола

BabyJubJub имеет координаты в поле `𝔽_r`. Рабочая криптографическая подгруппа `⟨G⟩` имеет порядок `ℓ`, поэтому её скаляры живут в другом поле `𝔽_ℓ`. Запись элемента `𝔽_ℓ` как целого внутри circuit-поля `𝔽_r` является кодированием представителя, а не морфизмом полей.

Голос `m` кодируется точкой `M=[m]G`. Для общего публичного ключа комитета `P=[s]G` и случайности `k` шифротекст имеет вид

`C₁=[k]G`, `C₂=M+[k]P`.

Участник с долей `d_i` публикует

`D_i=[d_i]G`, `σ_i=[d_i]C₁`.

Для номеров участников `x_i` коэффициенты `λ_i` восстанавливают свободный коэффициент секретного многочлена:

`s=Σλ_i d_i`, поэтому `Σ[λ_i]σ_i=[s]C₁`.

После добавления независимой маски владельца приложения итоговая relation `DecryptCombine` равна

`C₂=[m]G+Σ[λ_i]σ_i+Δ_org`, где `Δ_org=[d_org]C₁`.

## Реальный путь данных и proof

Здесь участвуют разные слои; смешивание их приводит к неверным выводам о том, что именно было проверено.

1. `Node.combine` обычным Go-кодом получает сохранённые `C₁,C₂`, принятые `σ_i` и номера участников, считает `Σ[λ_i]σ_i`, получает кандидата `[m]G` и находит малый дискретный лог `m` ([`node/decrypt.go:1240–1296`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/node/decrypt.go#L1240-L1296)).
2. `Assignment` ещё не является witness. `BuildWitness(Assignment)` повторно считает необходимые derived values, формирует полный private/public witness и отдельно public inputs.
3. `Define` не запускается заново на заполненном объекте. Он заранее скомпилирован в constraint system `ccs`; runtime содержит `ccs`, proving key `pk` и verifying key `vk`.
4. `CircuitRuntime.ProveAndVerify(witness)` создаёт Groth16 proof из `ccs+pk+witness` и немедленно проверяет его локально через `vk`. Этот метод ничего не отправляет наружу ([`artifacts.go:552–671`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/circuits/artifacts.go#L552-L671)).
5. Только после успешной локальной проверки node сериализует proof, public inputs и transcript и вызывает `DKGManager` ([`node/decrypt.go:1295–1345`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/node/decrypt.go#L1295-L1345)).
6. `DKGManager` отдельно сверяет calldata с on-chain state и затем вызывает отдельный on-chain verifier ([`DKGManager.sol:1147–1230`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/solidity/src/DKGManager.sol#L1147-L1230)).

Локальный `ProveAndVerify`, контракт `DKGManager` и on-chain verifier — три последовательных, но разных слоя.

## Finding 1 — вложенный DLEQ дублирует relation внешнего SNARK

**Классификация:** подтверждённая оптимизация; не уязвимость.

Исходный `PartialDecryptCircuit` напрямую получает private `Secret=d_i` и проверяет

`D_i=[d_i]G`, `σ_i=[d_i]C₁`.

После этого тот же circuit создаёт private `Nonce=w`, публичные `A₁=[w]G`, `A₂=[w]C₁`, challenge `e`, response `z=w+ed_i mod ℓ` и ещё раз проверяет

`[z]G=A₁+[e]D_i`, `[z]C₁=A₂+[e]σ_i`.

Вторая пара является самостоятельным Chaum–Pedersen/DLEQ proof, вложенным внутрь Groth16, хотя внешний SNARK уже доказывает существование одного `d_i`, одновременно объясняющего `D_i` и `σ_i`. Public context `eid,aid,ctIdx,participantIndex,C₁,D_i,σ_i` остаётся public input внешнего proof; replay-binding не требует второго Fiat–Shamir transcript внутри circuit.

В экспериментальной копии удалены `Nonce,A₁,A₂,e,Response` и обе response-equations. Replay-тест использует один уже созданный Groth16 proof и меняет по одному `eid`, `aid`, `ctIdx` и `participantIndex`; все варианты отвергаются verifier.

Экспериментальная облегчённая relation также удаляет два других класса избыточных constraints, описанных ниже. Поэтому совокупное уменьшение `26 194→5 208` нельзя приписывать одному DLEQ-блоку без отдельного измерения.

## Finding 2 — range-check `Secret` условно избыточен

**Классификация:** подтверждённая условная оптимизация; не уязвимость.

Исходный circuit действительно содержит `Secret≤ℓ−1` перед двумя scalar multiplications ([`partialdecrypt/circuit.go:55–60`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/circuits/partialdecrypt/circuit.go#L55-L60)). Reference node до создания partial-decryption proof вызывает `ValidateCiphertext` и отказывается использовать долю для точки вне рабочей подгруппы ([`node/decrypt.go:835–843`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/node/decrypt.go#L835-L843)).

Если `G,C₁∈⟨G⟩` и их порядок равен `ℓ`, то

`[d_i+ℓ]G=[d_i]G`, `[d_i+ℓ]C₁=[d_i]C₁`.

Следовательно, range-check выбирает каноническую запись witness, но не меняет `D_i` или `σ_i`. Эксперимент с `Secret+ℓ` после удаления range-check удовлетворяет оставшимся constraints на корректном `C₁`.

Граница утверждения принципиальна: контракт и `PartialDecrypt` circuit сами не доказывают `C₁∈⟨G⟩`; это off-chain инвариант reference node. Удалять range-check безопасно только при обязательности этой subgroup-проверки для всех допустимых proof generators либо после переноса инварианта в более сильную доверенную границу. Отдельная экономия одного range-check не измерена.

## Finding 3 — отдельные on-curve checks результатов scalar multiplication

**Классификация:** подтверждённая малая оптимизация; не уязвимость.

При корректных исходных точках и текущих Edwards gadgets равенства `D_i=[d_i]G`, `σ_i=[d_i]C₁`, `A₁=[w]G`, `A₂=[w]C₁` уже заставляют результаты быть точками кривой. Четыре отдельные проверки результатов дублируют эти более сильные constraints; проверка внешнего `C₁` остаётся необходимой.

В отдельном изолированном варианте удаление четырёх checks дало `26 194→26 178` constraints. Честный proof проходил, а подстановка заведомо внекривой пары `(0,0)` вместо каждого результата отвергалась тестовыми путями Groth16 и PLONK.

После удаления всего DLEQ-блока `A₁,A₂` исчезают вместе с ним; в облегчённой relation отдельно остаётся вопрос только о `D_i,σ_i`. Результаты разных экспериментов не суммировались повторно.

## Проверка `DecryptCombine`

### От проводки к весам

`Assignment` не содержит готовых весов. `BuildWitness` по номерам участников вычисляет `λ_i` в `𝔽_ℓ`, выбирает канонические целые представители и кладёт их в private `LagrangeCoefficients` ([`witness.go:116–167`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/circuits/decryptcombine/witness.go#L116-L167)).

Математически веса являются верхней строкой обратной проводки: прямая проводка переводит коэффициенты многочлена в его значения по номерам участников. Для `x=(1,2,3)` базисные коэффициенты переходят так:

`(1,0,0)→(1,1,1)`, `(0,1,0)→(1,2,3)`, `(0,0,1)→(1,4,9)`.

Если `(λ₁,λ₂,λ₃)` — верхняя строка обратной проводки, применение её к этим трём правым векторам возвращает свободные коэффициенты `1,0,0`:

`Σλ_i=1`, `Σλ_i x_i=0`, `Σλ_i x_i²=0` в `𝔽_ℓ`.

Circuit кодирует эти проверки групповыми действиями: для каждой степени `j` точка `Σ[λ_i x_i^j]G` должна быть `G` при `j=0` и identity при `j>0`. Для различных активных `x_i` все проверки вместе фиксируют нужные классы весов в `𝔽_ℓ` ([`circuit.go:151–195`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/circuits/decryptcombine/circuit.go#L151-L195)).

Затем circuit строит `combined=Σ[λ_i]σ_i`, проверяет `D_org=[d_org]G`, вычисляет `Δ_org=[d_org]C₁` и требует

`C₂=[Plaintext]G+combined+Δ_org`.

### Интеграционные привязки

Корректной circuit relation недостаточно, если combiner может подменить её входы. Здесь контракт закрывает этот разрыв:

- `C₁,C₂` сверяются с hash ранее сохранённого ciphertext ([`DKGManager.sol:1161–1219`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/solidity/src/DKGManager.sol#L1161-L1219));
- для каждого активного participant index требуется ранее установленный bitmap bit;
- каждая `σ_i` повторно хешируется и сравнивается с hash принятой partial decryption ([`DKGManager.sol:869–927`](https://github.com/vocdoni/davinci-dkg/blob/9ec5ff39be536871584e1b0a13b76d83c51078c0/solidity/src/DKGManager.sol#L869-L927));
- `OrganizerPK` сверяется с ключом текущего приложения;
- transcript commitment пересчитывается circuit и контрактом из фактических данных.

В рассмотренном контуре произвольную `σ_i`, ciphertext или ключ организатора подставить вместо сохранённых данных нельзя. Новой ошибки в `DecryptCombine` не обнаружено.

### Неканонические представители весов

**Классификация:** informational; сейчас не уязвимость и не требуемый range-check.

Circuit работает над `𝔽_r`, тогда как веса являются скалярами `𝔽_ℓ`. `BuildWitness` выбирает `0≤a<ℓ`, но constraints не требуют этого диапазона и могут принять `a+ℓ`, если значение помещается в `𝔽_r`.

В текущем `DecryptCombine` вес после умножения только на маску `0/1` сразу применяется к точкам порядка `ℓ`, поэтому `[a+ℓ]P=[a]P`. `combined` и plaintext не меняются; уникален класс веса в `𝔽_ℓ`, но не его целочисленная запись в `𝔽_r`. Комментарий source code о “unique witness” сильнее фактической гарантии.

Если будущая версия начнёт складывать, умножать или делить два таких представителя средствами `𝔽_r` до группового действия, аргумент перестанет работать. Например, при `ℓ=5,r=11` две записи `7` дают `7+7=3` в `𝔽_r`, хотя соответствующие элементы `2+2` дают `4` в `𝔽_ℓ`. Тогда потребуется constraint `0≤a<ℓ` либо доказанное редуцирование modulo `ℓ` внутри circuit. В нынешнем коде range-check весов для корректности расшифровки не нужен.

## Экспериментальные доказательства

| Проверка | Результат |
|---|---|
| Baseline: `go test ./circuits/partialdecrypt ./circuits/decryptcombine` | успешно |
| Облегчённая копия: те же package tests | успешно |
| Облегчённая копия: `go test ./node -run '^$'` | node компилируется |
| Один Groth16 proof, изменённые `eid/aid/ctIdx/participantIndex` | все replay-варианты отвергнуты |
| Совокупный размер облегчённого `PartialDecrypt` | `26 194→5 208` constraints, `−20 986` (`≈80%`) |
| Apple M4, `BenchmarkProve`, `5x`, три прогона | примерно `76.6→17.6 ms/proof`, `≈4.4×` быстрее |
| Удаление четырёх output on-curve checks отдельно | `−16` constraints; honest proof принят, `(0,0)` отвергнут Groth16 и PLONK |
| Контрольная серия `DecryptCombine` через `ProveAndVerify` | `100/100` proof прошли локальную проверку с первой попытки |

`ProveAndVerify` содержит до четырёх повторных попыток, если свежесозданный proof не проходит локальную проверку. Комментарий в коде предполагает вероятность такого события около 5%, но в серии `100/100` отказ не воспроизведён. Без воспроизводимого сбоя это только операционная заметка, не finding и не ослабление on-chain проверки.

## Ограничения и необходимые шаги перед merge

Экспериментальная облегчённая копия содержит локальный patch поверх указанного commit; patch не опубликован upstream и не является готовым deployment change. До merge необходимо:

1. зафиксировать окончательную trust boundary для subgroup-проверки `C₁`;
2. регенерировать circuit, proving/verifying artifacts и Solidity verifier;
3. изменить Solidity public-input layout с исходных 15 слов на новый layout;
4. прогнать полные integration и contract suites, включая реальные calldata/transcript paths;
5. повторить benchmark в контролируемой среде и отдельно измерить вклад каждого удалённого класса constraints;
6. исправить комментарий `DecryptCombine` о единственности witness;
7. исследовать retry только при появлении воспроизводимого invalid proof.

Этот документ не утверждает отсутствие ошибок вне описанного scope и не является полным аудитом всего Da Vinci DKG.

## Заключение

Главный результат — не найденный exploit, а уточнение реальной relation. `PartialDecrypt` достаточно доказать одним внешним Groth16 statement: один скрытый `d_i` объясняет и `D_i`, и `σ_i`. Вложенный DLEQ повторяет это утверждение и образует крупный избыточный блок; его изолированный вклад отдельно не измерен. Совокупный облегчённый вариант сохранил replay-binding в локальных тестах и дал крупное сокращение constraints, но требует регенерации артефактов и полной интеграционной проверки.

`DecryptCombine` прослежен от нативной интерполяции через witness и constraints до сохранённого on-chain state. Его веса корректно фиксируются как классы в `𝔽_ℓ`; отсутствие их range-check сейчас не меняет расшифровку. Контрактные hash-привязки не позволяют combiner произвольно заменить ciphertext или принятую partial decryption.
