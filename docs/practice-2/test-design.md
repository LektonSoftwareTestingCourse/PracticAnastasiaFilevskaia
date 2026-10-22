# Test-design ядра СМП: Authorization и Card-Management

---

## 0. Назначение и допущения

Документ описывает, какие тесты нужны для Authorization и Card-Management и почему их достаточно. Применены три техники: классы эквивалентности, граничные значения и попарное тестирование (PICT). Для периферии даны классы верхнего уровня, а проверки собраны в [`checklists.md`](checklists.md). Модели и наборы лежат в [`pict/`](pict/).

Допущения:

1. Ожидаемые результаты выведены из ТЗ и кода; прогон на запущенной системе не выполнялся. Если ТЗ и код расходятся, ожидаемым считается поведение кода.
2. Суммы в копейках, даты в UTC. Authorization берёт дату из `transmissionDateTime`, поэтому границы суток, месяца и срока карты проверяются подстановкой даты в запрос.
3. Тест на лимиты или баланс всегда использует новую карту.

Условные обозначения:

- REQ-AUTH-<номер_требования> - ID выделенного из ТЗ требования;
- TC - тест-кейс (test case);
- EC - класс эквивалентности (equivalence class);
- BV - граничное значение (boundary value);
- MC - критичное сочетание, добавленное вручную;
- PW - строка попарного набора (pairwise);
- B - баланс карты (available balance);
- D - дневной лимит карты;
- M - месячный лимит карты.


## 1. Требования

| ID | Требование |
|---|---|
| REQ-AUTH-02 | Карта не найдена → DECLINED, 14 |
| REQ-AUTH-03 | Статус не ACTIVE → DECLINED (INACTIVE, BLOCKED — 05; EXPIRED — 54) |
| REQ-AUTH-04 | Срок действия истёк → DECLINED, 54 |
| REQ-AUTH-05 | Дневной лимит превышен → DECLINED, 61 |
| REQ-AUTH-06 | Месячный лимит превышен → DECLINED, 61 |
| REQ-AUTH-07 | Сумма больше баланса → DECLINED, 51 |
| REQ-AUTH-08 | Успех → APPROVED, 00; резервирование; RRN — 12 цифр, authCode — 6 символов A–Z0–9 |
| REQ-AUTH-09 | Card-Management недоступен → DECLINED |
| REQ-AUTH-10 | Bin Lookup недоступен → решение принимается |
| REQ-AUTH-11 | Откат возвращает средства; повторный откат — отказ |
| REQ-CMS-02 | `POST /api/cards`: PAN = BIN + 9 цифр + цифра Луна, срок +3 года (MMYY), статус ACTIVE |
| REQ-CMS-03 | `GET /api/cards/{pan}`: 200 или 404 |
| REQ-CMS-04 | `GET /api/cards`: пагинация (`limit`, `offset`), `total` |
| REQ-CMS-05 | `PATCH /api/cards/{pan}`: частичное обновление статуса, лимитов, баланса |
| REQ-CMS-06 | `DELETE`: мягкое удаление |
| REQ-CMS-07 | `POST /api/cards/generate`: создаёт `count` карт по списку `bins` |
| REQ-CMS-08 | PAN проходит проверку Луна |
| REQ-CMS-09 | `POST /api/cards/{pan}/reserve`: `availableBalance -= amount` |

## 2. Классы эквивалентности

Для каждого класса выбран представитель. Валидные классы объединены в один позитивный тест, каждый невалидный класс проверяется отдельным тестом.

### 2.1. Authorization

| Условие | Валидные классы (представитель → тест) | Невалидные классы (представитель → тест) |
|---|---|---|
| Карта в CMS | существует (карта S) → EC-01 | не найдена: PAN `4000009999999991` → EC-02 |
| Card-Management | доступен → EC-01 | недоступен → EC-03 |
| Статус | ACTIVE → EC-01 | INACTIVE → EC-04; BLOCKED → EC-05; EXPIRED → EC-06 |
| Срок действия | в будущем (+3 года) → EC-01; в месяце транзакции → BV-10 | раньше месяца транзакции → EC-07 |
| Сумма и баланс | amount=10000, баланс=1000000 → EC-01 | amount > баланса: баланс=5000 → EC-08 |
| Дневной лимит | первая транзакция дня, лимит 500000 → EC-01 | amount > лимита: лимит=5000 → EC-09 |
| Месячный лимит | первая транзакция месяца → EC-01 | накопленная сумма > лимита → EC-10 |
| Канал и MCC | любой канал (POS/ATM/ECOM), любой 4-значный MCC → EC-01 | невалидных классов нет (в решении не участвуют) |
| Bin Lookup | доступен → EC-01 | недоступен (деградация, ожидается успех) → EC-11 |
| Откат | первый откат по RRN → EC-12 | повторный откат → EC-13; неизвестный RRN → EC-14 |

### 2.2. Card-Management

| Условие | Валидные классы (представитель → тест) | Невалидные классы (представитель → тест) |
|---|---|---|
| `bin` | 6 цифр: `400000` → TC-CMS-PW-01 | менее 6: `40000` → TC-CMS-PW-16; более 6: `4000000` → TC-CMS-PW-15; с буквой: `40000A` → TC-CMS-PW-09 |
| `cardholderName` | заглавные латинские, пробел, `-`, `.`, `'`: `IVAN IVANOV` → TC-CMS-PW-01 | строчные → TC-CMS-PW-17; цифры → TC-CMS-PW-13; пустая → EC-02 |
| `currencyCode` | 3 цифры: `840` → TC-CMS-PW-01 | 2 символа → TC-CMS-PW-11; 4 символа → TC-CMS-PW-14; буквы `RUB` → EC-03 |
| `dailyLimit`, `monthlyLimit` | ≥ 0: `15000000`, `300000000` → EC-01 | `dailyLimit=-1` → TC-CMS-PW-12; `monthlyLimit=-1` → TC-CMS-PW-10 |
| PAN в пути | 16 цифр, карта есть → EC-04 | 16 цифр, карты нет → EC-05; не 16 цифр или нецифровой → EC-06 |
| `limit` (список) | 1…10000: `5` → EC-07 | ≤ 0: `-1` → EC-08; > 10000: `10001` → BV-07 |
| `offset` | ≥ 0: `0`, `5` → EC-07 | < 0: `-1` → EC-09 |
| PATCH | допустимый статус → EC-10 | недопустимый статус → EC-11; отрицательный лимит → EC-12 |
| DELETE | существующая карта → EC-13 | несуществующая → EC-14 |
| generate: `count` | ≥ 1: `20` → EC-15 | ≤ 0: `0` → EC-16 |
| generate: `bins` | непустой список валидных BIN → EC-15 | `[]` → EC-17; BIN неверного формата `["40"]` → EC-18 |
| reserve: `amount` | 0 < amount ≤ баланса → EC-19 | amount > баланса → EC-20; amount ≤ 0 → EC-21 |
| reserve: `rrn` | 12 цифр, новый → EC-19 | не 12 цифр → EC-22; повтор → EC-23 |

## 3. Граничные значения

На границе берутся три точки: **ON** — значение на границе, **OFF** — значение за ней (`OFF−` и `OFF+` по разные стороны).

### 3.1. Authorization

| Граница | OFF− | ON | OFF+ | Тесты |
|---|---|---|---|---|
| Сумма и баланс | B−1 → APPROVED | B → APPROVED | B+1 → 51 | BV-01…03 |
| Сумма и дневной лимит | D−1 → APPROVED | D → APPROVED | D+1 → 61 | BV-04…06 |
| Накопленный месячный лимит | M−1 → APPROVED | M → APPROVED | M+1 → 61 | BV-07…09 |
| Срок карты (MMYY) | предпоследний день месяца → APPROVED | последний день месяца 23:59:59Z → APPROVED | 1-е число следующего месяца 00:00:00Z → 54 | BV-10…12 |
| Смена суток | 23:59:59Z, D исчерпан → 61 | — | 00:00:00Z следующих суток → APPROVED | BV-13, 14 |
| Смена месяца | последний день месяца, M исчерпан → 61 | — | 1-е число следующего месяца → APPROVED | BV-15, 16 |

### 3.2. Card-Management

| Граница | OFF− | ON | OFF+ | Тесты |
|---|---|---|---|---|
| Длина PAN в пути | 15 цифр → 400 | 16 цифр → 200 | 17 цифр → 400 | BV-01…03 |
| `limit` | 0 → 400 | 1 → 200 | — | BV-04, 05 |
| `limit`, максимум | — | 10000 → 200 | 10001 → 413 | BV-06, 07 |
| `offset` | −1 → 400 (EC-09) | 0 → 200 | — | BV-08 |
| `count` (генерация) | 0 → 400 (EC-16) | 1 → 1 карта | 2 → 2 карты | BV-09, 10 |
| reserve: `amount` и баланс | — | B → 200, баланс 0 | B+1 → 402 (EC-20) | BV-11 |
| reserve: нижняя граница `amount` | 0 → 400 (EC-21) | 1 → 200 | — | BV-12 |
| Формат `expiryDate` (MMYY), месяц | — | `01` (ведущий ноль), `12` | — | BV-13, 14 |
| `bin`, `currencyCode`, лимиты | длины 5 / 2, значение −1 | длины 6 / 3, значение 0 | длины 7 / 4, значение 1 | строки pairwise |

## 4. Попарное тестирование (pairwise)

Наборы сгенерированы инструментом Microsoft PICT из моделей в [`pict/`](pict/).

| Сервис | Модель → набор | Параметров | Ограничений | Строк | Полный перебор |
|---|---|:-:|:-:|:-:|:-:|
| Authorization | `model.txt` → `cases.txt` | 7 | 4 | 29 | 3888 |
| Card-Management | `model-card-management.txt` → `cases-card-management.txt` | 6 | 2 | 38 | 2048 |

**Authorization.** Параметры: статус карты (4); соотношение суммы с дневным лимитом, месячным лимитом и балансом (по 3: ниже / равно / выше); срок (3); тип терминала (3); MCC (4). Тип терминала и MCC в решении не участвуют, они подтверждают, что результат от них не зависит. Ограничения исключают строки, где у отказа несколько причин: (1) если карта не ACTIVE, остальные условия выполнены; (2) если срок истёк, остальные условия выполнены; (3) не более одного соотношения суммы может быть «выше». Так по результату всегда ясно, какая проверка сработала.

**Card-Management.** Значения с `~` невалидны, PICT ставит каждое в отдельную строку, остальные значения в ней валидны. Ограничение: месячный лимит не меньше дневного.

**Ограничения техники.** Тройки и более сложные сочетания попарный набор покрывает не полностью. Исключённые ограничениями пары и порядок проверок проверяются вручную тестами MC-01 и MC-02.

## 5. Тест-кейсы
### 5.1. Authorization: классы эквивалентности

| ID | Вид | Треб. | Предусловие | Шаги | Ожидаемый результат | Источник |
|---|---|---|---|---|---|---|
| TC-AUTH-EC-01 | позитивный | REQ-AUTH-08 | Карта S, статус ACTIVE; Card-Management и Bin Lookup запущены | Авторизация, amount=10000 | APPROVED, 00; `rrn` — 12 цифр; `authCode` — 6 символов; баланс −10000 | классы валидные |
| TC-AUTH-EC-02 | негативный | REQ-AUTH-02 | Карты `4000009999999991` нет | Авторизация по этому PAN | DECLINED, 14 | карта не найдена |
| TC-AUTH-EC-03 | негативный | REQ-AUTH-09 | Карта S; Card-Management остановлен | Авторизация, amount=10000 | DECLINED, 96 ⚠ OQ-2 | CMS недоступен |
| TC-AUTH-EC-04 | негативный | REQ-AUTH-03 | Карта S, `status=INACTIVE` | Авторизация | DECLINED, 05 | статус INACTIVE |
| TC-AUTH-EC-05 | негативный | REQ-AUTH-03 | Карта S, `status=BLOCKED` | Авторизация | DECLINED, 05 | статус BLOCKED |
| TC-AUTH-EC-06 | негативный | REQ-AUTH-03 | Карта S, `status=EXPIRED` | Авторизация | DECLINED, 54 | статус EXPIRED |
| TC-AUTH-EC-07 | негативный | REQ-AUTH-04 | Карта S (ACTIVE), срок = прошлый месяц | Авторизация | DECLINED, 54 | срок истёк |
| TC-AUTH-EC-08 | негативный | REQ-AUTH-07 | Карта S, `initialBalance=5000` | Авторизация, amount=10000 | DECLINED, 51 | amount > баланса |
| TC-AUTH-EC-09 | негативный | REQ-AUTH-05 | Карта S, `dailyLimit=5000` | Авторизация, amount=10000 | DECLINED, 61 | amount > дневного лимита |
| TC-AUTH-EC-10 | негативный | REQ-AUTH-06 | Карта: D=500000, M=800000, B=2000000; день 1: amount=500000 (APPROVED) | День 2: amount=400000 | DECLINED, 61 | месячный лимит накоплен |
| TC-AUTH-EC-11 | позитивный | REQ-AUTH-10 | Карта S; Bin Lookup остановлен | Авторизация, amount=10000 | APPROVED, 00 | деградация |
| TC-AUTH-EC-12 | позитивный | REQ-AUTH-11 | Одобренная транзакция, RRN известен | `POST /api/internal/rollback` | 200, 00; баланс восстановлен | первый откат |
| TC-AUTH-EC-13 | негативный | REQ-AUTH-11 | Откат по RRN уже выполнен | Повторный rollback | Отказ ALREADY_ROLLED_BACK | повторный откат |
| TC-AUTH-EC-14 | негативный | REQ-AUTH-11 | — | rollback с `rrn=000000000000` | Отказ TRANSACTION_NOT_FOUND | неизвестный RRN |

### 5.2. Authorization: граничные значения

Базовая карта: B=1000000, D=500000, M=2000000, если не указано иное; на каждый тест — новая карта.

| ID | Вид | Треб. | Предусловие | Шаги | Ожидаемый результат | Источник |
|---|---|---|---|---|---|---|
| TC-AUTH-BV-01 | позитивный | REQ-AUTH-07 | Карта: B=1000000, D=M=2000000 | amount=999999 | APPROVED, 00 | баланс, OFF− |
| TC-AUTH-BV-02 | позитивный | REQ-AUTH-07 | то же | amount=1000000 | APPROVED, 00; баланс 0 | баланс, ON |
| TC-AUTH-BV-03 | негативный | REQ-AUTH-07 | то же | amount=1000001 | DECLINED, 51 | баланс, OFF+ |
| TC-AUTH-BV-04 | позитивный | REQ-AUTH-05 | Карта S | amount=499999 | APPROVED, 00 | дневной лимит, OFF− |
| TC-AUTH-BV-05 | позитивный | REQ-AUTH-05 | Карта S | amount=500000 | APPROVED, 00 | дневной лимит, ON |
| TC-AUTH-BV-06 | негативный | REQ-AUTH-05 | Карта S | amount=500001 | DECLINED, 61 | дневной лимит, OFF+ |
| TC-AUTH-BV-07 | позитивный | REQ-AUTH-06 | Карта: D=500000, M=800000, B=2000000; день 1: amount=500000 | День 2: amount=299999 | APPROVED, 00 | месячный лимит, OFF− |
| TC-AUTH-BV-08 | позитивный | REQ-AUTH-06 | то же | День 2: amount=300000 (итого M) | APPROVED, 00 | месячный лимит, ON |
| TC-AUTH-BV-09 | негативный | REQ-AUTH-06 | то же | День 2: amount=300001 | DECLINED, 61 | месячный лимит, OFF+ |
| TC-AUTH-BV-10 | позитивный | REQ-AUTH-04 | Карта S, срок = месяц MM | `transmissionDateTime` — предпоследний день MM | APPROVED, 00 | срок, OFF− |
| TC-AUTH-BV-11 | позитивный | REQ-AUTH-04 | то же | Последний день MM, 23:59:59Z | APPROVED, 00 | срок, ON |
| TC-AUTH-BV-12 | негативный | REQ-AUTH-04 | то же | 1-е число MM+1, 00:00:00Z | DECLINED, 54 | срок, OFF+ |
| TC-AUTH-BV-13 | негативный | REQ-AUTH-05 | Карта: D=10000; день X, 23:59:59Z: amount=10000 (APPROVED) | День X, 23:59:59Z: amount=1 | DECLINED, 61 | смена суток, до границы |
| TC-AUTH-BV-14 | позитивный | REQ-AUTH-05 | то же состояние | День X+1, 00:00:00Z: amount=10000 | APPROVED, 00 | смена суток, после границы |
| TC-AUTH-BV-15 | негативный | REQ-AUTH-06 | Карта: D=M=800000, B=2000000; последний день месяца: amount=800000 | Тот же месяц, amount=1 (выбрать месяц без смены между транзакциями) | DECLINED, 61 | смена месяца, до границы |
| TC-AUTH-BV-16 | позитивный | REQ-AUTH-06 | то же состояние | 1-е число следующего месяца, 00:00:00Z: amount=800000 | APPROVED, 00 | смена месяца, после границы |

### 5.3. Authorization: критичные сочетания (вручную)

| ID | Вид | Треб. | Предусловие | Шаги | Ожидаемый результат | Источник |
|---|---|---|---|---|---|---|
| TC-AUTH-MC-01 | негативный | REQ-AUTH-03 | Карта S, `status=BLOCKED`, B=5000 | amount=600000 (больше баланса и дневного лимита) | DECLINED, 05 (приоритет статуса) | сочетание вне pairwise |
| TC-AUTH-MC-02 | негативный | REQ-AUTH-07 | Карта: B=5000, D=5000 | amount=10000 | DECLINED, 51; в ТЗ ожидалось бы 61 ⚠ OQ-1 | сочетание вне pairwise |
| TC-AUTH-MC-03 | позитивный | REQ-AUTH-08 | Карта S | amount=B+1 (отказ 51), затем amount=B | Отказ не списал баланс и не израсходовал лимит: вторая авторизация APPROVED | побочные эффекты |
| TC-AUTH-MC-04 | позитивный | REQ-AUTH-08 | Карта: B=15000, лимиты достаточны | Одновременно два запроса amount=10000 | Не более одного APPROVED; баланс ≥ 0 (зафиксировать фактическое поведение) | параллельность |
| TC-AUTH-MC-05 | позитивный | REQ-AUTH-08 | Карта с большим балансом и лимитами | 100 одобрений подряд | `rrn` — 12 цифр и уникальны; `authCode` — 6 символов A–Z0–9 ⚠ OQ-3 | форматы |

### 5.4. Authorization: попарный набор (PICT)

Источник — `pict/cases.txt`, 29 тестов.

**Шаги (для всех строк):** (1) `POST /api/cards` (`bin=400000`, `cardholderName=IVAN IVANOV`, `currencyCode=643`) с лимитами и балансом из колонки «Предусловие»; (2) если статус не ACTIVE — PATCH статуса; (3) если срок не «по умолчанию» — SQL-замена `expiry_date`; (4) стандартный запрос с `amount=10000`, `terminalType` и `mcc` из строки; (5) сверить `status`, `responseCode`, `declineReason`, при APPROVED — формат `rrn`, `authCode` и уменьшение баланса. Соотношения: «ниже» — лимит или баланс на 1 больше суммы, «равно» — равен, «выше» — на 1 меньше; значения в колонке уже рассчитаны.

| ID | Вид | Треб. | Предусловие (карта) | Канал | MCC | Ожидаемый результат | Источник |
|---|---|---|---|---|---|---|---|
| TC-AUTH-PW-01 | негативный | REQ-AUTH-03 | статус INACTIVE; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | ECOM | 5411 (grocery) | DECLINED, 05, CARD_INACTIVE (причина: статус INACTIVE) | строка 1 `cases.txt` |
| TC-AUTH-PW-02 | негативный | REQ-AUTH-03 | статус BLOCKED; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | POS | 5812 (restaurant) | DECLINED, 05, CARD_BLOCKED (причина: статус BLOCKED) | строка 2 `cases.txt` |
| TC-AUTH-PW-03 | позитивный | REQ-AUTH-08 | статус ACTIVE; dailyLimit=10000, monthlyLimit=10000, initialBalance=10000; срок = месяц транзакции (SQL) | ATM | 5411 (grocery) | APPROVED, 00 (все проверки пройдены) | строка 3 `cases.txt` |
| TC-AUTH-PW-04 | негативный | REQ-AUTH-03 | статус INACTIVE; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | ATM | 4722 (travel) | DECLINED, 05, CARD_INACTIVE (причина: статус INACTIVE) | строка 4 `cases.txt` |
| TC-AUTH-PW-05 | негативный | REQ-AUTH-03 | статус EXPIRED; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | POS | 4722 (travel) | DECLINED, 54, CARD_EXPIRED (причина: статус EXPIRED) | строка 5 `cases.txt` |
| TC-AUTH-PW-06 | негативный | REQ-AUTH-03 | статус BLOCKED; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | ECOM | 4722 (travel) | DECLINED, 05, CARD_BLOCKED (причина: статус BLOCKED) | строка 6 `cases.txt` |
| TC-AUTH-PW-07 | негативный | REQ-AUTH-03 | статус EXPIRED; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | ECOM | 5812 (restaurant) | DECLINED, 54, CARD_EXPIRED (причина: статус EXPIRED) | строка 7 `cases.txt` |
| TC-AUTH-PW-08 | негативный | REQ-AUTH-03 | статус BLOCKED; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | ATM | 5732 (electronics) | DECLINED, 05, CARD_BLOCKED (причина: статус BLOCKED) | строка 8 `cases.txt` |
| TC-AUTH-PW-09 | негативный | REQ-AUTH-03 | статус INACTIVE; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | POS | 5732 (electronics) | DECLINED, 05, CARD_INACTIVE (причина: статус INACTIVE) | строка 9 `cases.txt` |
| TC-AUTH-PW-10 | негативный | REQ-AUTH-07 | статус ACTIVE; dailyLimit=10001, monthlyLimit=10000, initialBalance=9999; срок = месяц транзакции (SQL) | ECOM | 5732 (electronics) | DECLINED, 51, INSUFFICIENT_FUNDS (причина: сумма > баланса) | строка 10 `cases.txt` |
| TC-AUTH-PW-11 | негативный | REQ-AUTH-05 | статус ACTIVE; dailyLimit=9999, monthlyLimit=10001, initialBalance=10000; срок = месяц транзакции (SQL) | POS | 5732 (electronics) | DECLINED, 61, EXCEEDS_AMOUNT_LIMIT (причина: сумма > дневного лимита) | строка 11 `cases.txt` |
| TC-AUTH-PW-12 | негативный | REQ-AUTH-04 | статус ACTIVE; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок = прошлый месяц (SQL) | ECOM | 4722 (travel) | DECLINED, 54, CARD_EXPIRED (причина: срок истёк) | строка 12 `cases.txt` |
| TC-AUTH-PW-13 | негативный | REQ-AUTH-07 | статус ACTIVE; dailyLimit=10000, monthlyLimit=10000, initialBalance=9999; срок по умолчанию (+3 года) | POS | 4722 (travel) | DECLINED, 51, INSUFFICIENT_FUNDS (причина: сумма > баланса) | строка 13 `cases.txt` |
| TC-AUTH-PW-14 | негативный | REQ-AUTH-03 | статус INACTIVE; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | ATM | 5812 (restaurant) | DECLINED, 05, CARD_INACTIVE (причина: статус INACTIVE) | строка 14 `cases.txt` |
| TC-AUTH-PW-15 | негативный | REQ-AUTH-06 | статус ACTIVE; dailyLimit=10000, monthlyLimit=9999, initialBalance=10000; срок = месяц транзакции (SQL) | ECOM | 4722 (travel) | DECLINED, 61, EXCEEDS_AMOUNT_LIMIT (причина: сумма > месячного лимита) | строка 15 `cases.txt` |
| TC-AUTH-PW-16 | негативный | REQ-AUTH-06 | статус ACTIVE; dailyLimit=10000, monthlyLimit=9999, initialBalance=10001; срок по умолчанию (+3 года) | ATM | 5732 (electronics) | DECLINED, 61, EXCEEDS_AMOUNT_LIMIT (причина: сумма > месячного лимита) | строка 16 `cases.txt` |
| TC-AUTH-PW-17 | негативный | REQ-AUTH-04 | статус ACTIVE; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок = прошлый месяц (SQL) | ATM | 5812 (restaurant) | DECLINED, 54, CARD_EXPIRED (причина: срок истёк) | строка 17 `cases.txt` |
| TC-AUTH-PW-18 | негативный | REQ-AUTH-05 | статус ACTIVE; dailyLimit=9999, monthlyLimit=10000, initialBalance=10001; срок по умолчанию (+3 года) | POS | 5411 (grocery) | DECLINED, 61, EXCEEDS_AMOUNT_LIMIT (причина: сумма > дневного лимита) | строка 18 `cases.txt` |
| TC-AUTH-PW-19 | негативный | REQ-AUTH-07 | статус ACTIVE; dailyLimit=10000, monthlyLimit=10001, initialBalance=9999; срок = месяц транзакции (SQL) | ATM | 5812 (restaurant) | DECLINED, 51, INSUFFICIENT_FUNDS (причина: сумма > баланса) | строка 19 `cases.txt` |
| TC-AUTH-PW-20 | негативный | REQ-AUTH-06 | статус ACTIVE; dailyLimit=10001, monthlyLimit=9999, initialBalance=10000; срок по умолчанию (+3 года) | POS | 5411 (grocery) | DECLINED, 61, EXCEEDS_AMOUNT_LIMIT (причина: сумма > месячного лимита) | строка 20 `cases.txt` |
| TC-AUTH-PW-21 | негативный | REQ-AUTH-03 | статус EXPIRED; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | ATM | 5732 (electronics) | DECLINED, 54, CARD_EXPIRED (причина: статус EXPIRED) | строка 21 `cases.txt` |
| TC-AUTH-PW-22 | негативный | REQ-AUTH-06 | статус ACTIVE; dailyLimit=10001, monthlyLimit=9999, initialBalance=10000; срок = месяц транзакции (SQL) | ECOM | 5812 (restaurant) | DECLINED, 61, EXCEEDS_AMOUNT_LIMIT (причина: сумма > месячного лимита) | строка 22 `cases.txt` |
| TC-AUTH-PW-23 | негативный | REQ-AUTH-05 | статус ACTIVE; dailyLimit=9999, monthlyLimit=10000, initialBalance=10001; срок = месяц транзакции (SQL) | ATM | 5812 (restaurant) | DECLINED, 61, EXCEEDS_AMOUNT_LIMIT (причина: сумма > дневного лимита) | строка 23 `cases.txt` |
| TC-AUTH-PW-24 | негативный | REQ-AUTH-03 | статус BLOCKED; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | POS | 5411 (grocery) | DECLINED, 05, CARD_BLOCKED (причина: статус BLOCKED) | строка 24 `cases.txt` |
| TC-AUTH-PW-25 | негативный | REQ-AUTH-04 | статус ACTIVE; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок = прошлый месяц (SQL) | POS | 5732 (electronics) | DECLINED, 54, CARD_EXPIRED (причина: срок истёк) | строка 25 `cases.txt` |
| TC-AUTH-PW-26 | негативный | REQ-AUTH-04 | статус ACTIVE; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок = прошлый месяц (SQL) | ECOM | 5411 (grocery) | DECLINED, 54, CARD_EXPIRED (причина: срок истёк) | строка 26 `cases.txt` |
| TC-AUTH-PW-27 | негативный | REQ-AUTH-05 | статус ACTIVE; dailyLimit=9999, monthlyLimit=10000, initialBalance=10001; срок по умолчанию (+3 года) | ECOM | 4722 (travel) | DECLINED, 61, EXCEEDS_AMOUNT_LIMIT (причина: сумма > дневного лимита) | строка 27 `cases.txt` |
| TC-AUTH-PW-28 | негативный | REQ-AUTH-03 | статус EXPIRED; dailyLimit=10001, monthlyLimit=10001, initialBalance=10001; срок по умолчанию (+3 года) | ECOM | 5411 (grocery) | DECLINED, 54, CARD_EXPIRED (причина: статус EXPIRED) | строка 28 `cases.txt` |
| TC-AUTH-PW-29 | негативный | REQ-AUTH-07 | статус ACTIVE; dailyLimit=10001, monthlyLimit=10001, initialBalance=9999; срок = месяц транзакции (SQL) | ATM | 5411 (grocery) | DECLINED, 51, INSUFFICIENT_FUNDS (причина: сумма > баланса) | строка 29 `cases.txt` |

### 5.5. Card-Management: классы эквивалентности

Классы `bin`, `cardholderName` (кроме пустой), `currencyCode` (кроме букв) и отрицательных лимитов проверяются строками попарного набора (5.7).

| ID | Вид | Треб. | Предусловие | Шаги | Ожидаемый результат | Источник |
|---|---|---|---|---|---|---|
| TC-CMS-EC-01 | позитивный | REQ-CMS-02, 08 | — | Создать 100 карт (`POST /api/cards`) | PAN из 16 цифр, уникальны, проходят Луна; срок +3 года; ACTIVE | классы валидные |
| TC-CMS-EC-02 | негативный | REQ-CMS-02 | — | `POST /api/cards`, `cardholderName=""` | 400 | имя пустое |
| TC-CMS-EC-03 | негативный | REQ-CMS-02 | — | `POST /api/cards`, `currencyCode=RUB` | 400 | валюта — буквы |
| TC-CMS-EC-04 | позитивный | REQ-CMS-03 | Карта создана | `GET /api/cards/{pan}` | 200, поля совпадают | PAN существует |
| TC-CMS-EC-05 | негативный | REQ-CMS-03 | — | `GET /api/cards/4000009999999991` | 404 | PAN не найден |
| TC-CMS-EC-06 | негативный | REQ-CMS-03 | — | `GET /api/cards/40000012345` | 400 | PAN неверного формата |
| TC-CMS-EC-07 | позитивный | REQ-CMS-04 | В базе ≥ 10 карт | `GET /api/cards?limit=5&offset=0`, затем `offset=5` | 200; по 5 карт, без пересечений; `total` — общее число ⚠ OQ-4 | `limit`, `offset` валидные |
| TC-CMS-EC-08 | негативный | REQ-CMS-04 | — | `GET /api/cards?limit=-1` | 400 | `limit` ≤ 0 |
| TC-CMS-EC-09 | негативный | REQ-CMS-04 | — | `GET /api/cards?offset=-1` | 400 | `offset` < 0 |
| TC-CMS-EC-10 | позитивный | REQ-CMS-05 | Карта ACTIVE | PATCH `{"status":"BLOCKED"}`, затем GET | 200; изменён только статус | PATCH валидный |
| TC-CMS-EC-11 | негативный | REQ-CMS-05 | Карта создана | PATCH `{"status":"FOO"}` | 400 | статус недопустим |
| TC-CMS-EC-12 | негативный | REQ-CMS-05 | Карта создана | PATCH `{"dailyLimit":-1}` | 400 | лимит < 0 |
| TC-CMS-EC-13 | позитивный | REQ-CMS-06 | Карта создана | DELETE, затем GET | 204; затем 404 | DELETE валидный |
| TC-CMS-EC-14 | негативный | REQ-CMS-06 | — | DELETE `4000009999999991` | 404 | карты нет |
| TC-CMS-EC-15 | позитивный | REQ-CMS-07 | — | generate `{"count":20,"bins":["400000","400001"]}` | 201; 20 карт, поровну по двум BIN | generate валидный |
| TC-CMS-EC-16 | негативный | REQ-CMS-07 | — | generate, `count=0` | 400 | `count` ≤ 0 |
| TC-CMS-EC-17 | негативный | REQ-CMS-07 | — | generate, `bins=[]` | 400 | пустой `bins` |
| TC-CMS-EC-18 | негативный | REQ-CMS-07 | — | generate, `bins=["40"]` | 400 | BIN неверного формата |
| TC-CMS-EC-19 | позитивный | REQ-CMS-09 | Карта, баланс 1000000 | reserve `{"amount":150000,"rrn":"012345678901"}` | 200; баланс 850000 | reserve валидный |
| TC-CMS-EC-20 | негативный | REQ-CMS-09 | Карта, баланс 1000000 | reserve, `amount=1000001` | 402; баланс не изменился | amount > баланса |
| TC-CMS-EC-21 | негативный | REQ-CMS-09 | Карта создана | reserve, `amount=0` | 400 | amount ≤ 0 |
| TC-CMS-EC-22 | негативный | REQ-CMS-09 | Карта создана | reserve, `rrn="123"` | 400 | `rrn` не из 12 цифр |
| TC-CMS-EC-23 | негативный | REQ-CMS-09 | Выполнен EC-19 | reserve с тем же `rrn` | 409; баланс не списан повторно | повтор `rrn` |

### 5.6. Card-Management: граничные значения

| ID | Вид | Треб. | Предусловие | Шаги | Ожидаемый результат | Источник |
|---|---|---|---|---|---|---|
| TC-CMS-BV-01 | негативный | REQ-CMS-03 | — | `GET /api/cards/` + 15 цифр | 400 | длина PAN, OFF− |
| TC-CMS-BV-02 | позитивный | REQ-CMS-03 | Карта создана | `GET /api/cards/{pan}` (16 цифр) | 200 | длина PAN, ON |
| TC-CMS-BV-03 | негативный | REQ-CMS-03 | — | `GET /api/cards/` + 17 цифр | 400 | длина PAN, OFF+ |
| TC-CMS-BV-04 | негативный | REQ-CMS-04 | — | `GET /api/cards?limit=0` | 400 | `limit`, OFF− |
| TC-CMS-BV-05 | позитивный | REQ-CMS-04 | Карты есть | `GET /api/cards?limit=1` | 200; одна карта | `limit`, ON |
| TC-CMS-BV-06 | позитивный | REQ-CMS-04 | Карты есть | `GET /api/cards?limit=10000` | 200 | `limit` максимум, ON |
| TC-CMS-BV-07 | негативный | REQ-CMS-04 | — | `GET /api/cards?limit=10001` | 413 | `limit` максимум, OFF+ |
| TC-CMS-BV-08 | позитивный | REQ-CMS-04 | Карты есть | `GET /api/cards?offset=0` | 200; первая страница | `offset`, ON |
| TC-CMS-BV-09 | позитивный | REQ-CMS-07 | — | generate, `count=1` | 201; ровно 1 карта | `count`, ON |
| TC-CMS-BV-10 | позитивный | REQ-CMS-07 | — | generate, `count=2` | 201; ровно 2 карты | `count`, OFF+ |
| TC-CMS-BV-11 | позитивный | REQ-CMS-09 | Карта, баланс B | reserve, `amount=B` | 200; баланс 0 | reserve, ON |
| TC-CMS-BV-12 | позитивный | REQ-CMS-09 | Карта, баланс > 0 | reserve, `amount=1` | 200; баланс −1 | `amount`, ON |
| TC-CMS-BV-13 | позитивный | REQ-CMS-02 | Системная дата — январь | Создать карту | `expiryDate` начинается с `01`, год +3 | MMYY, месяц `01` |
| TC-CMS-BV-14 | позитивный | REQ-CMS-02 | Системная дата — декабрь | Создать карту | `expiryDate` начинается с `12`, год +3 | MMYY, месяц `12` |

BV-13 и BV-14 выполняются, когда есть возможность задать дату окружения; иначе — в соответствующий месяц.

### 5.7. Card-Management: попарный набор (PICT)

Источник — `pict/cases-card-management.txt`, 38 тестов. **Предусловие:** Card-Management запущен. **Шаги:** `POST /api/cards` со значениями из строки; сверить код ответа. Для позитивных строк проверить также: PAN из 16 цифр, начинается с `bin`, проходит Луну, `expiryDate` = текущая дата + 3 года, `status=ACTIVE`. Значение с `~` невалидно и единственное в строке.

| ID | Вид | Треб. | bin | cardholderName | currencyCode | dailyLimit | monthlyLimit | initialBalance | Ожидаемый результат | Источник |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-CMS-PW-01 | позитивный | REQ-CMS-02 | 400000 | IVAN IVANOV | 840 | 0 | 1 | 0 | 201; PAN: 16 цифр, начинается с bin, проходит проверку Луна; status=ACTIVE | строка 1 `cases-card-management.txt` |
| TC-CMS-PW-02 | позитивный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | 643 | 1 | 1 | 100000000 | 201; PAN: 16 цифр, начинается с bin, проходит проверку Луна; status=ACTIVE | строка 2 `cases-card-management.txt` |
| TC-CMS-PW-03 | позитивный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | 643 | 15000000 | 300000000 | 0 | 201; PAN: 16 цифр, начинается с bin, проходит проверку Луна; status=ACTIVE | строка 3 `cases-card-management.txt` |
| TC-CMS-PW-04 | позитивный | REQ-CMS-02 | 400000 | IVAN IVANOV | 840 | 0 | 300000000 | 100000000 | 201; PAN: 16 цифр, начинается с bin, проходит проверку Луна; status=ACTIVE | строка 4 `cases-card-management.txt` |
| TC-CMS-PW-05 | позитивный | REQ-CMS-02 | 400000 | IVAN IVANOV | 840 | 1 | 300000000 | 0 | 201; PAN: 16 цифр, начинается с bin, проходит проверку Луна; status=ACTIVE | строка 5 `cases-card-management.txt` |
| TC-CMS-PW-06 | позитивный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | 840 | 0 | 0 | 100000000 | 201; PAN: 16 цифр, начинается с bin, проходит проверку Луна; status=ACTIVE | строка 6 `cases-card-management.txt` |
| TC-CMS-PW-07 | позитивный | REQ-CMS-02 | 400000 | IVAN IVANOV | 643 | 0 | 0 | 0 | 201; PAN: 16 цифр, начинается с bin, проходит проверку Луна; status=ACTIVE | строка 7 `cases-card-management.txt` |
| TC-CMS-PW-08 | позитивный | REQ-CMS-02 | 400000 | IVAN IVANOV | 840 | 15000000 | 300000000 | 100000000 | 201; PAN: 16 цифр, начинается с bin, проходит проверку Луна; status=ACTIVE | строка 8 `cases-card-management.txt` |
| TC-CMS-PW-09 | негативный | REQ-CMS-02 | ~40000A | ANNA-MARIA O'NEIL | 840 | 15000000 | 300000000 | 100000000 | 400, ошибка валидации поля `Bin` | строка 9 `cases-card-management.txt` |
| TC-CMS-PW-10 | негативный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | 840 | 1 | ~-1 | 0 | 400, ошибка валидации поля `MonthlyLimit` | строка 10 `cases-card-management.txt` |
| TC-CMS-PW-11 | негативный | REQ-CMS-02 | 400000 | IVAN IVANOV | ~64 | 1 | 300000000 | 0 | 400, ошибка валидации поля `CurrencyCode` | строка 11 `cases-card-management.txt` |
| TC-CMS-PW-12 | негативный | REQ-CMS-02 | 400000 | IVAN IVANOV | 643 | ~-1 | 300000000 | 100000000 | 400, ошибка валидации поля `DailyLimit` | строка 12 `cases-card-management.txt` |
| TC-CMS-PW-13 | негативный | REQ-CMS-02 | 400000 | ~IVAN 123 | 643 | 15000000 | 300000000 | 0 | 400, ошибка валидации поля `CardholderName` | строка 13 `cases-card-management.txt` |
| TC-CMS-PW-14 | негативный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | ~6430 | 1 | 1 | 100000000 | 400, ошибка валидации поля `CurrencyCode` | строка 14 `cases-card-management.txt` |
| TC-CMS-PW-15 | негативный | REQ-CMS-02 | ~4000000 | ANNA-MARIA O'NEIL | 840 | 0 | 300000000 | 100000000 | 400, ошибка валидации поля `Bin` | строка 15 `cases-card-management.txt` |
| TC-CMS-PW-16 | негативный | REQ-CMS-02 | ~40000 | IVAN IVANOV | 840 | 0 | 1 | 0 | 400, ошибка валидации поля `Bin` | строка 16 `cases-card-management.txt` |
| TC-CMS-PW-17 | негативный | REQ-CMS-02 | 400000 | ~ivan ivanov | 643 | 1 | 1 | 0 | 400, ошибка валидации поля `CardholderName` | строка 17 `cases-card-management.txt` |
| TC-CMS-PW-18 | негативный | REQ-CMS-02 | ~4000000 | IVAN IVANOV | 643 | 1 | 1 | 0 | 400, ошибка валидации поля `Bin` | строка 18 `cases-card-management.txt` |
| TC-CMS-PW-19 | негативный | REQ-CMS-02 | 400000 | IVAN IVANOV | 643 | 15000000 | ~-1 | 100000000 | 400, ошибка валидации поля `MonthlyLimit` | строка 19 `cases-card-management.txt` |
| TC-CMS-PW-20 | негативный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | ~64 | 0 | 1 | 100000000 | 400, ошибка валидации поля `CurrencyCode` | строка 20 `cases-card-management.txt` |
| TC-CMS-PW-21 | негативный | REQ-CMS-02 | 400000 | IVAN IVANOV | ~6430 | 0 | 0 | 100000000 | 400, ошибка валидации поля `CurrencyCode` | строка 21 `cases-card-management.txt` |
| TC-CMS-PW-22 | негативный | REQ-CMS-02 | ~4000000 | ANNA-MARIA O'NEIL | 840 | 0 | 0 | 0 | 400, ошибка валидации поля `Bin` | строка 22 `cases-card-management.txt` |
| TC-CMS-PW-23 | негативный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | 643 | ~-1 | 0 | 0 | 400, ошибка валидации поля `DailyLimit` | строка 23 `cases-card-management.txt` |
| TC-CMS-PW-24 | негативный | REQ-CMS-02 | ~40000A | IVAN IVANOV | 643 | 1 | 1 | 0 | 400, ошибка валидации поля `Bin` | строка 24 `cases-card-management.txt` |
| TC-CMS-PW-25 | негативный | REQ-CMS-02 | ~40000 | ANNA-MARIA O'NEIL | 643 | 15000000 | 300000000 | 100000000 | 400, ошибка валидации поля `Bin` | строка 25 `cases-card-management.txt` |
| TC-CMS-PW-26 | негативный | REQ-CMS-02 | 400000 | ~ivan ivanov | 840 | 15000000 | 300000000 | 100000000 | 400, ошибка валидации поля `CardholderName` | строка 26 `cases-card-management.txt` |
| TC-CMS-PW-27 | негативный | REQ-CMS-02 | 400000 | ~ivan ivanov | 840 | 0 | 0 | 0 | 400, ошибка валидации поля `CardholderName` | строка 27 `cases-card-management.txt` |
| TC-CMS-PW-28 | негативный | REQ-CMS-02 | 400000 | IVAN IVANOV | ~64 | 15000000 | 300000000 | 100000000 | 400, ошибка валидации поля `CurrencyCode` | строка 28 `cases-card-management.txt` |
| TC-CMS-PW-29 | негативный | REQ-CMS-02 | ~40000 | IVAN IVANOV | 840 | 1 | 300000000 | 0 | 400, ошибка валидации поля `Bin` | строка 29 `cases-card-management.txt` |
| TC-CMS-PW-30 | негативный | REQ-CMS-02 | ~40000 | ANNA-MARIA O'NEIL | 840 | 0 | 0 | 0 | 400, ошибка валидации поля `Bin` | строка 30 `cases-card-management.txt` |
| TC-CMS-PW-31 | негативный | REQ-CMS-02 | 400000 | ~IVAN 123 | 840 | 0 | 0 | 100000000 | 400, ошибка валидации поля `CardholderName` | строка 31 `cases-card-management.txt` |
| TC-CMS-PW-32 | негативный | REQ-CMS-02 | ~4000000 | ANNA-MARIA O'NEIL | 840 | 15000000 | 300000000 | 0 | 400, ошибка валидации поля `Bin` | строка 32 `cases-card-management.txt` |
| TC-CMS-PW-33 | негативный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | ~6430 | 15000000 | 300000000 | 0 | 400, ошибка валидации поля `CurrencyCode` | строка 33 `cases-card-management.txt` |
| TC-CMS-PW-34 | негативный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | 643 | 0 | ~-1 | 0 | 400, ошибка валидации поля `MonthlyLimit` | строка 34 `cases-card-management.txt` |
| TC-CMS-PW-35 | негативный | REQ-CMS-02 | 400000 | ~IVAN 123 | 840 | 1 | 1 | 0 | 400, ошибка валидации поля `CardholderName` | строка 35 `cases-card-management.txt` |
| TC-CMS-PW-36 | негативный | REQ-CMS-02 | 400000 | IVAN IVANOV | ~64 | 0 | 0 | 0 | 400, ошибка валидации поля `CurrencyCode` | строка 36 `cases-card-management.txt` |
| TC-CMS-PW-37 | негативный | REQ-CMS-02 | ~40000A | ANNA-MARIA O'NEIL | 840 | 0 | 0 | 0 | 400, ошибка валидации поля `Bin` | строка 37 `cases-card-management.txt` |
| TC-CMS-PW-38 | негативный | REQ-CMS-02 | 400000 | ANNA-MARIA O'NEIL | 840 | ~-1 | 1 | 100000000 | 400, ошибка валидации поля `DailyLimit` | строка 38 `cases-card-management.txt` |
