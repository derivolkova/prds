<img width="439" height="260" alt="image" src="https://github.com/user-attachments/assets/a4e5d861-b56f-4497-a9a7-841877105db3" />
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
        price *= 1-discount / 100
    for dedaction in kwargs.values():
        price -= dedaction 
    return price 
    
price = float(input()) 
percents_input = input().strip()
percents = [float(x) for x in percents_input.split(',')] if percents_input else [] 
print(f"Итог: {calculate_final_price(price, *percents, promo=500, cashback=200):.2f}")
```

<img width="829" height="603" alt="image" src="https://github.com/user-attachments/assets/3a667a59-29c1-4bb4-b774-0fd12359bb61" />

```python
blacklist_input = input().strip().split(',')
transactions_input = input().strip().split(',')

blacklist_set = set(blacklist_input)
blocked_count = 0

blocked_count = sum(1 for tx in transactions_input if tx in blacklist_input)

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
WITH rfm AS (
    SELECT
        customer_id,
        DATE '2019-12-31' - MAX(sales_transaction_date)::date AS recency,
        COUNT(*) AS frequency,
        SUM(sales_amount) AS monetary
    FROM sales
    GROUP BY customer_id
),
scores AS (
    SELECT
        monetary,
        NTILE(5) OVER (ORDER BY recency DESC) AS r_score,
        NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
    FROM rfm
)
SELECT
    CONCAT('R', r_score, '_F', f_score, '_M', m_score) AS segment,
    COUNT(*) AS customers_count,
    ROUND(AVG(monetary)::numeric, 2) AS avg_segment_monetary
FROM scores
GROUP BY r_score, f_score, m_score
ORDER BY customers_count DESC
LIMIT 10;
```



<img width="1654" height="680" alt="image" src="https://github.com/user-attachments/assets/46ea4a1e-82a3-444b-8dd6-59839a5ea7a7" />


```sql
SELECT
    p.product_id,
    p.model,
    DATE_TRUNC('month', s.sales_transaction_date) AS sale_month,
    AVG(s.sales_amount) AS avg_sale_price,
    p.base_msrp,
    ((AVG(s.sales_amount) - p.base_msrp) / p.base_msrp) * 100 AS variance_pct,
    'В пределах нормы' AS price_status
FROM sales s
JOIN products p USING(product_id)
GROUP BY p.product_id, p.model, p.base_msrp, sale_month
ORDER BY variance_pct
LIMIT 10;
```




<img width="1780" height="862" alt="image" src="https://github.com/user-attachments/assets/13840ed3-5197-447c-b0eb-68951bf42598" />

```sql
WITH base AS (
    SELECT
        s.sales_amount,
        p.product_type,
        COALESCE(d.state, c.state) AS state
    FROM sales s
    LEFT JOIN dealerships d ON s.dealership_id = d.dealership_id AND s.channel = 'dealership'
    LEFT JOIN customers c ON s.customer_id = c.customer_id AND s.channel = 'internet'
    JOIN products p ON s.product_id = p.product_id
    WHERE s.sales_transaction_date BETWEEN '2019-01-01' AND '2019-12-31'
),
stats AS (
    SELECT
        state,
        SUM(sales_amount) AS total_sales,
        COUNT(*) AS transactions_count,
        AVG(sales_amount) AS avg_check,
        RANK() OVER (ORDER BY SUM(sales_amount) DESC) AS sales_rank
    FROM base
    GROUP BY state
)
SELECT
    'State General' AS record_type,
    state,
    NULL AS product_type,
    total_sales,
    transactions_count,
    avg_check,
    sales_rank
FROM stats
WHERE sales_rank <= 3

UNION ALL

SELECT
    'Product Detail',
    state,
    product_type,
    SUM(sales_amount),
    NULL,
    NULL,
    NULL
FROM base
WHERE state IN (SELECT state FROM stats WHERE sales_rank <= 3)
GROUP BY state, product_type
ORDER BY state, record_type DESC, product_type
limit 10
```



<img width="1763" height="744" alt="image" src="https://github.com/user-attachments/assets/7b746a97-71ae-4944-9005-49d071fbe9a4" />

```sql
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    MAX(s.sales_transaction_date) AS last_purchase,
    MAX(e.opened_date) AS last_email_open
FROM customers c
JOIN sales s ON c.customer_id = s.customer_id
JOIN emails e ON c.customer_id = e.customer_id
WHERE s.sales_transaction_date <= '2019-12-31'
    AND e.opened_date IS NOT NULL
GROUP BY c.customer_id, c.first_name, c.last_name, c.email
HAVING
    ('2019-12-31'::date - MAX(s.sales_transaction_date)::date) > 180
    AND MAX(e.opened_date) > MAX(s.sales_transaction_date)
ORDER BY MAX(e.opened_date) DESC
LIMIT 10;
```
<img width="1529" height="856" alt="image" src="https://github.com/user-attachments/assets/19262e5b-3a61-476a-bf43-8209e3be83ec" />

```sql
SELECT
    CONCAT(sp.first_name, ' ', sp.last_name) AS full_name,
    SUM(s.sales_amount) AS total_sales,
    RANK() OVER (ORDER BY SUM(s.sales_amount) DESC) AS sales_rank,
    SUM(s.sales_amount) / COUNT(*) AS avg_check,
    COUNT(DISTINCT s.product_id) AS unique_products,
    EXTRACT(YEAR FROM AGE('2019-12-31', sp.hire_date)) * 12 + EXTRACT(MONTH FROM AGE('2019-12-31', sp.hire_date)) AS tenure_months
FROM salespeople sp
INNER JOIN sales s ON sp.dealership_id = s.dealership_id
    AND s.sales_transaction_date <= '2019-12-31'
WHERE sp.termination_date IS NULL
GROUP BY sp.salesperson_id, sp.first_name, sp.last_name, sp.hire_date
ORDER BY sales_rank
LIMIT 5;
```




