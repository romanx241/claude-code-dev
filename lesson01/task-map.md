# Карта задачи: дневной лимит списаний

Задача — модуль дневного лимита по спецификации `app/backend/DAILY_LIMIT.md`.
Ниже: что в коде уже есть, чего нет и где будут жить модуль и тесты.
У каждого утверждения указан файл, из которого это видно.

## 1. Что уже есть

### Округление денег

- `money(value)` — приведение к `Decimal` с двумя знаками, округление
  `ROUND_HALF_UP`; константа `TWO_PLACES = Decimal("0.01")` —
  `app/backend/wallet/money.py`.
- `parse_amount(raw)` — разбор суммы из тела запроса, возвращает `Decimal`
  или `None` — `app/backend/wallet/money.py`.
- Спецификация прямо ссылается на это округление: «округление — как у
  остальных денег продукта (`wallet/money.py`)» — `app/backend/DAILY_LIMIT.md`.
- Числовые константы продукта (`MAX_OPERATION_AMOUNT`, `MIN_OPERATION_AMOUNT`,
  `FEE_PERCENT`, `MIN_FEE`) лежат в `app/backend/wallet/config.py`.

### Формат элемента истории

- `OperationView` — dataclass с полями `operation_id`, `wallet_id`, `type`,
  `amount`, `comment`, `created_at` (ISO-8601 UTC, `"2026-09-08T12:04:11Z"`) —
  `app/backend/wallet/projection.py`.
- `OperationView.to_item()` — словарь элемента `items` в ответе API:
  `operation_id`, `type`, `amount`, `comment`, `created_at` —
  `app/backend/wallet/projection.py`.
- История отдаётся клиенту именно через `to_item()` —
  `app/backend/wallet/routers/wallet.py` (`GET /api/wallet/operations`).
- Спецификация лимита требует на входе «те же словари, что уходят клиенту
  в истории (`OperationView.to_item`): поля `type`, `amount`, `created_at`» —
  `app/backend/DAILY_LIMIT.md`.
- Тариф кошелька — поле `tariff` у `WalletRow` (`BASIC` / `FREE`) —
  `app/backend/wallet/auth.py`; в ответе API он отдаётся как `"tariff"` —
  `app/backend/wallet/routers/wallet.py`.

### Похожий модуль — образец устройства

- `app/backend/wallet/cashback.py` — чистый модуль без базы и окружения:
  на вход список операций, на выход числа; импортирует `money` из
  `wallet.money`; константы продукта объявлены в самом модуле.
- В нём уже есть разбор метки времени по префиксу: `operation_month(created_at)`
  возвращает `str(created_at)[:7]` — `app/backend/wallet/cashback.py`.
  Это прямой аналог `operation_day(created_at)` из спецификации.
- Суммирование по типу операции и периоду — `monthly_turnover(operations, month)`
  — `app/backend/wallet/cashback.py`.
- Модуль подключён к обработчику `GET /api/wallet/cashback` —
  `app/backend/wallet/routers/wallet.py`.
- Рядом лежит документ-спецификация `CASHBACK.md`, первичный по отношению
  к коду, — `app/backend/CASHBACK.md` (так же устроена пара
  `DAILY_LIMIT.md` ↔ `wallet/daily_limit.py`).

### Как устроены существующие тесты и откуда они запускаются

- Тесты лежат в `app/backend/tests/` — `app/backend/tests/`,
  `app/backend/README.md` (раздел «Тесты»), `CONTRIBUTING.md` (раздел «Тесты»).
- `app/backend/tests/cashback_test.py` — тесты модуля без базы: список
  операций задан константой в памяти, проверки через `assert` и
  `pytest.mark.parametrize`; импорт `from wallet.cashback import ...`.
- `app/backend/tests/conftest.py` добавляет `app/backend` в `sys.path`,
  поэтому `import wallet.*` работает из корня бэкенда.
- Разрушительные тесты API — `app/backend/tests/contract_test.py`; фикстуры
  базы (`guard`, `credentials`, `client`, `db`) пропускаются без
  `WALLET_ALLOW_DESTRUCTIVE_TESTS=1` — `app/backend/tests/conftest.py`.
- Запуск: обычный `pytest` из `app/backend` для тестов без базы; для
  контрактных — с флагом и `DATABASE_URL` тестовой базы —
  `app/backend/tests/conftest.py`, `CONTRIBUTING.md`.
- `pytest==8.3.4` в зависимостях — `app/backend/requirements.txt`.
- Правило проекта: тест на дефект должен краснеть на старом коде и
  зеленеть на новом — `CONTRIBUTING.md`.

## 2. Чего нет

- Файла `app/backend/wallet/daily_limit.py` нет: в `app/backend/wallet/`
  его нет, а `DAILY_LIMIT.md` говорит о нём как о ещё не написанном коде —
  `app/backend/wallet/`, `app/backend/DAILY_LIMIT.md`.
- Нет ни одной из пяти функций спецификации — `operation_day`,
  `daily_spent`, `daily_limit`, `remaining_limit`, `withdraw_allowed`:
  поиск по коду находит их только в тексте `app/backend/DAILY_LIMIT.md`.
- Нет констант лимитов по тарифам (`BASIC` — 150 000,00 ₽, `FREE` —
  30 000,00 ₽): в `app/backend/wallet/config.py` их нет, в
  `app/backend/wallet/cashback.py` есть только `TARIFF_WITH_CASHBACK`.
- Нет тестов лимита: в `app/backend/tests/` только `cashback_test.py`,
  `contract_test.py` и `conftest.py`.
- Нет подключения проверки к обработчику списания и ответа API с остатком —
  это явно вынесено за рамки спецификации — `app/backend/DAILY_LIMIT.md`
  (раздел 5), `app/backend/wallet/routers/wallet.py` (обработчик
  `POST /api/wallet/withdraw` лимит не проверяет).

## 3. Где будут жить модуль и тесты

- Модуль: `app/backend/wallet/daily_limit.py` — путь назван в самой
  спецификации (`app/backend/DAILY_LIMIT.md`, раздел 4 и вступление),
  по аналогии с `app/backend/wallet/cashback.py`: чистый модуль, импорт
  `money` из `wallet.money`, константы лимитов внутри модуля.
- Тесты: `app/backend/tests/daily_limit_test.py` — по аналогии с
  `app/backend/tests/cashback_test.py`: список операций в памяти, без базы,
  запуск обычным `pytest` из `app/backend` (`CONTRIBUTING.md`,
  `app/backend/tests/conftest.py`).
- Спецификация уже лежит рядом с кодом — `app/backend/DAILY_LIMIT.md`,
  как `app/backend/CASHBACK.md` рядом с `wallet/cashback.py`.
