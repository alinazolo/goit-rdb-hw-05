# goit-rdb-hw-05 — PostgreSQL Customer Feature Engineering

## Опис проєкту

Цей проєкт присвячений роботі з PostgreSQL на основі датасету Olist Brazilian E-Commerce. Мета — підготувати структуровані дані, побудувати SQL CTE-pipeline для розрахунку customer-level features, дослідити Recursive CTE, Materialized View та проаналізувати продуктивність запитів за допомогою `EXPLAIN ANALYZE`.

## 1. Джерело даних

**Датасет:** Brazilian E-Commerce Public Dataset by Olist

**Джерело:** [Kaggle — Olist Brazilian E-Commerce](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

Датасет містить інформацію про покупців, замовлення, товари, продавців та інші дані бразильської e-commerce платформи.

Для роботи використовуються п'ять CSV-файлів:

- `olist_customers_dataset.csv`
- `olist_orders_dataset.csv`
- `olist_order_items_dataset.csv`
- `olist_products_dataset.csv`
- `olist_sellers_dataset.csv`

## 2. Отримання даних

Файли можна завантажити автоматично через бібліотеку `kagglehub`:

```python
import kagglehub

dataset_dir = kagglehub.dataset_download(
    "olistbr/brazilian-ecommerce"
)

print(dataset_dir)
```

Альтернативний спосіб — завантажити архів із Kaggle вручну та розпакувати CSV-файли до папки `data/olist/`.

## 3. Структура бази даних

У проєкті використовуються два основні шари даних.

### Raw layer

Raw-шар містить дані, імпортовані з CSV-файлів за допомогою `pandas.to_sql()`.

Таблиці:

- `olist_customers_raw`
- `olist_orders_raw`
- `olist_order_items_raw`
- `olist_products_raw`
- `olist_sellers_raw`

Цей шар використовується як проміжне сховище перед очищенням і приведенням даних до потрібних типів.

### Typed layer

Typed-шар містить структуровані таблиці PostgreSQL із визначеними типами даних, первинними та зовнішніми ключами, а також обмеженнями цілісності.

Таблиці:

- `olist_customers`
- `olist_orders`
- `olist_order_items`
- `olist_products`
- `olist_sellers`

У typed-шарі використовуються типи `TEXT`, `INTEGER`, `NUMERIC`, `TIMESTAMP`, а також обмеження `PRIMARY KEY`, `FOREIGN KEY`, `NOT NULL` і `CHECK`.

## 4. Реалізовані завдання

**Завдання 1 — Setup і Typed Schema**

Налаштування PostgreSQL, імпорт CSV-файлів у raw-шар, створення typed-таблиць та завантаження даних.

**Завдання 2 — CTE Pipeline**

Побудова багатокрокового SQL-pipeline для розрахунку customer-level features:

- `recency_days`
- `frequency`
- `monetary`
- `cohort_quarter`
- `avg_order_value_30d_recent`
- `cancellation_rate`
- `preferred_category`

Для ідентифікації покупців використовується `customer_unique_id`, що дозволяє об'єднувати повторні замовлення одного покупця.

**Завдання 3 — Recursive CTE**

Створення синтетичної таблиці `hw5_referrals` та обчислення глибини referral chain і повного шляху залучення. Реалізовано захист від циклів через `path_ids` і обмеження `depth`.

**Завдання 4 — Materialized View**

Створення `mv_olist_customer_features`, унікального індексу на `customer_unique_id` та оновлення через `REFRESH MATERIALIZED VIEW CONCURRENTLY`.

**Завдання 5 — EXPLAIN ANALYZE**

Порівняння продуктивності повного CTE-pipeline та lookup із Materialized View.

Результати вимірювання:

| Сценарій | Execution Time |
|---|---:|
| Повний CTE-pipeline | 1353.054 ms |
| Lookup із Materialized View | 0.158 ms |

Ці сценарії виконують різний обсяг роботи, тому наведені значення не є прямим порівнянням швидкості однакових операцій.

## 5. Інструкція запуску

### Крок 1. Створення віртуального середовища

```bash
python3.12 -m venv .venv
source .venv/bin/activate
```

### Крок 2. Встановлення залежностей

```bash
pip install ipykernel jupyter ipython-sql sqlalchemy psycopg2-binary pgserver pandas kagglehub
```

### Крок 3. Налаштування Jupyter Kernel

```bash
python -m ipykernel install --user --name goit-rdb-hw-05 --display-name "Python 3.12 (goit-rdb-hw-05)"
```

### Крок 4. Запуск notebook

Відкрийте `homework.ipynb` у VS Code та виберіть kernel `Python 3.12 (goit-rdb-hw-05)`.

### Крок 5. Запуск PostgreSQL

```python
import pgserver
from sqlalchemy import create_engine, text

pg = pgserver.get_server(
    "/tmp/hw5_pg",
    cleanup_mode="stop"
)

engine = create_engine(pg.get_uri(), future=True)

with engine.connect() as conn:
    print(conn.execute(text("SELECT version()")).scalar_one())
```

### Крок 6. Виконання SQL-запитів

```python
%load_ext sql
%config SqlMagic.autopandas = True
%sql $engine.url
```

Після підключення виконайте комірки notebook послідовно: завантаження CSV, створення raw/typed-таблиць, CTE-pipeline, Recursive CTE, Materialized View та `EXPLAIN ANALYZE`.

## 6. Використані технології

- Python 3.12
- PostgreSQL
- SQLAlchemy
- Pandas
- pgserver
- KaggleHub
- Jupyter Notebook
- Visual Studio Code

## 7. Висновок

Проєкт демонструє використання PostgreSQL для підготовки даних та побудови customer-level ML-features.

CTE забезпечує структуроване обчислення ознак, Recursive CTE дозволяє аналізувати ієрархічні зв'язки, а Materialized View зменшує потребу в повторному виконанні складних SQL-запитів.

Результати `EXPLAIN ANALYZE` демонструють різницю між повним переобчисленням ознак і читанням попередньо підготовлених даних.
