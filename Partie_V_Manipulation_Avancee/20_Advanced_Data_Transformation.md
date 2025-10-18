# Chapitre 20 : Advanced Data Transformation — Pivoting, Reshaping, Window Functions

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Pivoter** données (melting & pivoting)
- **Remodeler** DataFrames complexes
- **Window functions** pour analytics
- **Jointures avancées** (fuzzy, asof)
- **Transformations conditionnelles** (case when)

---

## 📖 Table des matières

1. [Pivoting & Unpivoting](#pivoting--unpivoting)
2. [Window Functions](#window-functions)
3. [Jointures Avancées](#jointures-avancées)
4. [Transformations Conditionnelles](#transformations-conditionnelles)
5. [Polars pour Performance](#polars-pour-performance)

---

## Pivoting & Unpivoting

### Pivot (Wide format)

```python
import pandas as pd
import polars as pl

# Data en format long
sales_long = pd.DataFrame({
    'date': ['2025-01', '2025-01', '2025-01', '2025-02', '2025-02', '2025-02'],
    'product': ['A', 'B', 'C', 'A', 'B', 'C'],
    'revenue': [100, 200, 150, 120, 180, 160]
})

# Pivot: produit en colonnes
sales_wide = sales_long.pivot(index='date', columns='product', values='revenue')
print(sales_wide)
#         A    B    C
# date
# 2025-01  100  200  150
# 2025-02  120  180  160

# Polars (plus rapide)
sales_pl = pl.DataFrame(sales_long).pivot(
    on='product',
    index='date',
    values='revenue'
)
```

### Unpivot (Melt)

```python
# Wide format → Long format
sales_wide = pd.DataFrame({
    'date': ['2025-01', '2025-02'],
    'A': [100, 120],
    'B': [200, 180],
    'C': [150, 160]
})

# Melt
sales_long = sales_wide.melt(
    id_vars=['date'],
    var_name='product',
    value_name='revenue'
)
print(sales_long)
#       date product  revenue
# 0  2025-01       A      100
# 1  2025-02       A      120
# 2  2025-01       B      200
```

### Pivot multi-level

```python
# Données complexes
sales = pd.DataFrame({
    'date': ['2025-01', '2025-01', '2025-02', '2025-02'],
    'region': ['North', 'South', 'North', 'South'],
    'product': ['A', 'A', 'B', 'B'],
    'revenue': [100, 150, 120, 180]
})

# Pivot multi-niveaux
pivot_multi = sales.pivot_table(
    index='date',
    columns=['region', 'product'],
    values='revenue',
    aggfunc='sum'
)
```

---

## Window Functions

### Row numbering & Ranking

```python
import pandas as pd

sales = pd.DataFrame({
    'date': ['2025-01', '2025-01', '2025-01', '2025-02', '2025-02'],
    'product': ['A', 'B', 'C', 'A', 'B'],
    'revenue': [100, 200, 150, 120, 180]
})

# Row number
sales['row_num'] = sales.groupby('date').cumcount() + 1

# Rank (ties handled)
sales['rank'] = sales.groupby('date')['revenue'].rank(method='dense')

# Percent rank
sales['pct_rank'] = sales.groupby('date')['revenue'].rank(pct=True)
```

### Running totals & Moving averages

```python
# Running sum
sales['running_sum'] = sales.groupby('product')['revenue'].cumsum()

# Moving average (3-period)
sales['ma_3'] = sales.groupby('product')['revenue'].rolling(
    window=3,
    min_periods=1
).mean().reset_index(drop=True)

# Lead/Lag
sales['prev_revenue'] = sales.groupby('product')['revenue'].shift(1)
sales['next_revenue'] = sales.groupby('product')['revenue'].shift(-1)
```

### Polars window functions (fastest)

```python
import polars as pl

sales_pl = pl.DataFrame(sales)

result = sales_pl.with_columns([
    pl.row_number().over('date').alias('row_num'),
    pl.col('revenue').rank().over('date').alias('rank'),
    pl.col('revenue').cum_sum().over('product').alias('running_sum'),
    pl.col('revenue').rolling_mean(window_size=3).over('product').alias('ma_3'),
    pl.col('revenue').shift(1).over('product').alias('prev_revenue'),
])
```

---

## Jointures Avancées

### Fuzzy matching

```python
from fuzzywuzzy import fuzz
import pandas as pd

customers_a = pd.DataFrame({
    'id': [1, 2, 3],
    'name': ['John Smith', 'Jane Doe', 'Bob Johnson']
})

customers_b = pd.DataFrame({
    'id': [10, 20, 30],
    'name': ['Jon Smith', 'Jane D.', 'Robert Johnson']
})

# Fuzzy match
matches = []
for idx_a, name_a in enumerate(customers_a['name']):
    best_match_idx = None
    best_score = 0
    
    for idx_b, name_b in enumerate(customers_b['name']):
        score = fuzz.token_set_ratio(name_a, name_b)
        if score > best_score:
            best_score = score
            best_match_idx = idx_b
    
    if best_score > 80:
        matches.append({
            'id_a': customers_a.iloc[idx_a]['id'],
            'id_b': customers_b.iloc[best_match_idx]['id'],
            'score': best_score
        })

matches_df = pd.DataFrame(matches)
```

### asof join (time-series)

```python
# Merge sur la dernière date <= target
trades = pd.DataFrame({
    'time': pd.to_datetime(['10:00', '10:05', '10:10', '10:15']),
    'price': [100, 101, 99, 102]
})

quotes = pd.DataFrame({
    'time': pd.to_datetime(['10:01', '10:06', '10:11']),
    'ask': [100.5, 101.5, 99.5],
    'bid': [99.5, 100.5, 98.5]
})

# Join sur la dernière quote <= trade time
result = pd.merge_asof(
    trades.sort_values('time'),
    quotes.sort_values('time'),
    on='time',
    direction='backward'
)
```

### Range join

```python
# Join pour plages qui se chevauchent
events_a = pd.DataFrame({
    'id': [1, 2],
    'start': [pd.Timestamp('2025-01-01'), pd.Timestamp('2025-01-05')],
    'end': [pd.Timestamp('2025-01-10'), pd.Timestamp('2025-01-15')]
})

events_b = pd.DataFrame({
    'id': [10, 20],
    'timestamp': [pd.Timestamp('2025-01-05'), pd.Timestamp('2025-01-08')]
})

# Trouver quelle plage contient chaque timestamp
result = []
for _, row_b in events_b.iterrows():
    for _, row_a in events_a.iterrows():
        if row_a['start'] <= row_b['timestamp'] <= row_a['end']:
            result.append({'event_id': row_a['id'], 'timestamp_id': row_b['id']})

range_join = pd.DataFrame(result)
```

---

## Transformations Conditionnelles

### Case-when

```python
import pandas as pd

sales = pd.DataFrame({
    'date': ['2025-01', '2025-01', '2025-02'],
    'revenue': [100, 250, 150]
})

# Simple condition
sales['category'] = pd.cut(
    sales['revenue'],
    bins=[0, 100, 200, float('inf')],
    labels=['Low', 'Medium', 'High']
)

# Multiple conditions
sales['performance'] = sales['revenue'].apply(
    lambda x: 'Excellent' if x > 200
              else 'Good' if x > 150
              else 'Fair' if x > 100
              else 'Poor'
)

# Polars (more efficient)
import polars as pl

sales_pl = pl.DataFrame(sales).with_columns([
    pl.when(pl.col('revenue') > 200)
        .then(pl.lit('Excellent'))
        .when(pl.col('revenue') > 150)
        .then(pl.lit('Good'))
        .otherwise(pl.lit('Fair'))
        .alias('performance')
])
```

### String transformations

```python
# Extract, replace, split
df = pd.DataFrame({
    'email': ['john@company.com', 'jane@company.com'],
    'phone': ['(555) 123-4567', '(555) 987-6543']
})

# Extract domain
df['domain'] = df['email'].str.extract(r'@([^.]+)')[0]

# Remove non-digits from phone
df['phone_clean'] = df['phone'].str.replace(r'\D', '', regex=True)

# Split into first/last
df[['first', 'last']] = df['email'].str.split('@', expand=True, n=1)[0].str.split(
    '.', expand=True, n=1
)
```

---

## Polars pour Performance

### Lazy evaluation

```python
import polars as pl

# Lazy (optimized query plan)
result = (
    pl.scan_csv('huge.csv')
    .filter(pl.col('revenue') > 100)
    .select(['date', 'product', 'revenue'])
    .groupby('product')
    .agg(pl.col('revenue').sum())
    .collect()  # Execute only here
)

# Vs Pandas (eager)
import pandas as pd
df = pd.read_csv('huge.csv')  # Load all
df_filtered = df[df['revenue'] > 100]  # Filter
result = df_filtered.groupby('product')['revenue'].sum()  # Aggregate
```

### Expressions for composability

```python
import polars as pl

sales = pl.DataFrame({
    'date': ['2025-01', '2025-01', '2025-02'],
    'product': ['A', 'B', 'A'],
    'revenue': [100, 200, 150]
})

# Composable expressions
result = (
    sales
    .groupby('product')
    .agg([
        pl.col('revenue').sum().alias('total'),
        pl.col('revenue').mean().alias('avg'),
        pl.col('revenue').max().alias('max'),
    ])
    .sort('total', descending=True)
)
```

---

## 🎓 Exercices pratiques

### Exercice 20.1 : Pivoting
Pivot long format → wide format et vice-versa.

### Exercice 20.2 : Window functions
Calculez running totals et rankings.

### Exercice 20.3 : Fuzzy matching
Matchez clients avec scores de similarité.

### Exercice 20.4 : Complex transformations
Case-when + string operations.

### Exercice 20.5 : Polars performance
Compare Pandas vs Polars sur 1GB CSV.

---

## 📚 Références

- **Pandas Reshaping** : https://pandas.pydata.org/docs/user_guide/reshape.html
- **Polars** : https://docs.pola-rs.com/
- **Window Functions** : https://en.wikipedia.org/wiki/Window_function

---

**Prêt pour streaming real-time? → [Chapitre 21 : Stream Processing](./21_Stream_Processing.md)**
