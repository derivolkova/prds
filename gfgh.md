
<img width="831" height="606" alt="image" src="https://github.com/user-attachments/assets/e4c7f055-aa17-4620-a63c-05d6cc52811c" />

```python
import json

data_string = input().strip()

try:
    data = json.loads(data_string)
except json.JSONDecodeError:
    print("Ошибка данных")
else:
    print(f"Выручка: {data['revenue']}")
finally:
    print("Соединение закрыто")
```

<img width="857" height="626" alt="image" src="https://github.com/user-attachments/assets/69c97b2f-9c7e-49fd-891e-09fcadccf025" />

```python
def predict_users(current_users, months):
    if months == 0:
        return int(current_users)
    return predict_users(current_users * 1.1, months - 1)

users = int(input())
m = int(input())
print(predict_users(users, m))
```
<img width="834" height="754" alt="image" src="https://github.com/user-attachments/assets/03bcfa19-f85e-4200-b40a-d9da5dafcb8c" />

```python
sales_dict = {}
while True:
    line = input().strip()
    if line == "END":
        break
   
    shop_id, region, amount = line.split(',')
    key = (shop_id, region)
    sales_dict[key] = sales_dict.get(key, 0.0) + float(amount)
   
for key in sorted(sales_dict.keys()):
    print(f"Магазин: {key[0]}, Регион: {key[1]} - Выручка: {sales_dict[key]}")
```

<img width="820" height="951" alt="image" src="https://github.com/user-attachments/assets/ec887eb3-e6ca-41cc-9cd6-c6c806f68bc3" />

```python
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

    def get_payout(self):
        return self.salary

class Manager(Employee):
    def __init__(self, name, salary, kpi_bonus):
        super().__init__(name, salary)
        self.kpi_bonus = kpi_bonus

    def get_payout(self):
        return self.salary + self.kpi_bonus

name = input().strip()
salary = float(input())
bonus = float(input())

m = Manager(name, salary, bonus)
print(f"К выплате менеджеру {m.name}: {m.get_payout()}")
```

<img width="888" height="714" alt="image" src="https://github.com/user-attachments/assets/96695517-1f3d-4c50-adcc-7a9440bd5e6c" />

```python
class Report:
    def __init__(self, title, pages):
        self.title = title
        self.pages = pages

    def __str__(self):
        return f"Отчет: {self.title} ({self.pages} стр.)"

    def __repr__(self):
        return f"Report(title='{self.title}', pages={self.pages})"

t = input().strip()
p = int(input())
doc = Report(t, p)

print("STR:", str(doc))
print("REPR:", repr(doc))
```

<img width="817" height="740" alt="image" src="https://github.com/user-attachments/assets/f1bdb0b7-47d6-46cf-bce6-e4cbf13cfa1f" />

```python
campaigns = [
    ("Promo_A", 1000, 1.5),
    ("Promo_B", 500, 2.0),
    ("Promo_C", 1500, 1.5),
    ("Promo_D", 800, 2.0)
]

campaigns.sort(key=lambda x: (-x[2], x[1]))

for c in campaigns:
    print(c[0])
```

<img width="820" height="656" alt="image" src="https://github.com/user-attachments/assets/91b066fe-f938-4834-8bf4-ffcde54376b3" />

```python
def calculate_final_price(base_price, *args, **kwargs):
    price = base_price
    for discount in args:
        price = price * (1 - discount / 100)
    for deduction in kwargs.values():
        price = price - deduction
    return price

price = float(input())
percents = []
try:
    percents_input = input().strip()
    if percents_input:
        percents = [float(x) for x in percents_input.split(',')]
except:
    pass

print(f"Итог: {calculate_final_price(price, *percents, promo=500, cashback=200):.2f}")
```

<img width="829" height="603" alt="image" src="https://github.com/user-attachments/assets/3a667a59-29c1-4bb4-b774-0fd12359bb61" />

```python
blacklist_input = input().strip().split(',')
transactions_input = input().strip().split(',')

blacklist_set = set(blacklist_input)
blocked_count = 0

for transaction in transactions_input:
    if transaction in blacklist_set:
        blocked_count += 1

print(f"Заблокировано транзакций: {blocked_count}")
```

<img width="867" height="631" alt="image" src="https://github.com/user-attachments/assets/0840945c-ad28-4c57-9ded-2da60c272e7c" />

```python
def predict_users(current_users, months):
    if current_users < 0 or months < 0:
        raise ValueError("Некорректные данные")
    if months == 0:
        return int(current_users)
    return predict_users(current_users * 1.1, months - 1)

users = int(input())
m = int(input())

try:
    print(predict_users(users, m))
except ValueError as e:
    print(e)
```



















<img width="1095" height="918" alt="image" src="https://github.com/user-attachments/assets/830ab29c-8174-476d-8189-d64c233169b8" />



```sql
WITH rfm_raw AS (
    SELECT
        c.customer_id,
        EXTRACT(DAY FROM ('2019-12-31' - MAX(s.sales_transaction_date))) AS recency,
        COUNT(*) AS frequency,
        SUM(s.sales_amount) AS monetary
    FROM customers c
    JOIN sales s ON c.customer_id = s.customer_id
    GROUP BY c.customer_id
),
rfm_scores AS (
    SELECT
        customer_id,
        recency,
        frequency,
        monetary,
        NTILE(5) OVER (ORDER BY recency DESC) AS r_score,
        NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
    FROM rfm_raw
),
rfm_segments AS (
    SELECT
        customer_id,
        r_score,
        f_score,
        m_score,
        CONCAT('R', r_score, '_F', f_score, '_M', m_score) AS segment,
        monetary
    FROM rfm_scores
)
SELECT
    segment,
    COUNT(customer_id) AS customers_count,
    CAST(AVG(monetary) AS NUMERIC(10, 2)) AS avg_segment_monetary
FROM rfm_segments
GROUP BY segment
ORDER BY customers_count DESC
LIMIT 10;
```



<img width="1654" height="680" alt="image" src="https://github.com/user-attachments/assets/46ea4a1e-82a3-444b-8dd6-59839a5ea7a7" />


```sql
WITH monthly_sales AS (
    SELECT 
        s.product_id,
        DATE_TRUNC('month', s.sales_transaction_date) AS sale_month,
        AVG(s.sales_amount) AS avg_sale_price
    FROM sales s
    GROUP BY s.product_id, DATE_TRUNC('month', s.sales_transaction_date)
),
variance_data AS (
    SELECT 
        ms.product_id,
        p.model,
        ms.sale_month,
        ms.avg_sale_price,
        p.base_msrp,
        ((ms.avg_sale_price - p.base_msrp) / p.base_msrp * 100) AS variance_pct
    FROM monthly_sales ms
    JOIN products p ON ms.product_id = p.product_id
    WHERE p.base_msrp IS NOT NULL AND p.base_msrp != 0
),
classified AS (
    SELECT 
        *,
        CASE 
            WHEN variance_pct > 10 THEN 'Наценка'
            WHEN variance_pct < -10 THEN 'Скидка'
            ELSE 'В пределах нормы'
        END AS price_status
    FROM variance_data
)
SELECT 
    product_id,
    model,
    sale_month,
    avg_sale_price,
    base_msrp,
    variance_pct,
    price_status
FROM classified
ORDER BY ABS(variance_pct) DESC
LIMIT 10;
```
