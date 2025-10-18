# Chapitre 7 : Python — Écosystème complet CSV & JSON

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Utiliser le module **`csv`** standard (low-level)
- Dominer **`pandas`** pour manipulation avancée
- Exploiter **`polars`** et **`datatable`** pour performance
- Utiliser **`dask`** et **`modin`** pour données distribuées
- Convertir avec **`pyarrow`** et gérer **Parquet**

---

## 📖 Table des matières

1. [Module csv — Baseline standard](#module-csv--baseline-standard)
2. [Pandas — L'incontournable](#pandas--lincontournable)
3. [Polars — La révolution performance](#polars--la-révolution-performance)
4. [Dask & Modin — Parallélisation transparente](#dask--modin--parallélisation-transparente)
5. [PyArrow & Parquet — Optimisation I/O](#pyarrow--parquet--optimisation-io)
6. [Comparatif frameworks](#comparatif-frameworks)

---

## Module csv — Baseline standard

### Installation (inclus en standard)

```python
import csv
```

### Lecture simple

```python
import csv

# DictReader (colonnes avec noms)
with open('data.csv', 'r', encoding='utf-8') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row)  # {'id': '1', 'nom': 'Alice', ...}

# Reader simple (listes)
with open('data.csv', 'r', encoding='utf-8') as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)  # ['1', 'Alice', ...]
```

### Écriture

```python
import csv

data = [
    {'id': 1, 'nom': 'Alice', 'email': 'alice@example.com'},
    {'id': 2, 'nom': 'Bob', 'email': 'bob@example.com'},
]

with open('output.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['id', 'nom', 'email'])
    writer.writeheader()
    writer.writerows(data)
```

### Dialects (séparateurs)

```python
# Définir un dialecte personnalisé
csv.register_dialect('semicolon', delimiter=';', quoting=csv.QUOTE_ALL)

# Utiliser
with open('data.csv', 'r') as f:
    reader = csv.DictReader(f, dialect='semicolon')
    for row in reader:
        print(row)
```

### Limitations du csv standard

```python
# ❌ Tout est chaîne
df['age']  # ['25', '30', '35'] ← strings, pas int!

# ❌ Pas de performance sur gros fichiers
# ❌ Pas d'agrégation/groupement natif
# ❌ Manipulation complexe = boucles manuelles
```

---

## Pandas — L'incontournable

### Installation

```bash
pip install pandas
```

### Lecture

```python
import pandas as pd

# Basique
df = pd.read_csv('data.csv')

# Avec options
df = pd.read_csv(
    'data.csv',
    encoding='utf-8',
    delimiter=',',
    dtype={'id': int, 'age': int},
    parse_dates=['date_embauche'],
    na_values=['NULL', 'N/A', ''],
    skiprows=1,  # Sauter 1ère ligne
    nrows=1000  # Lire seulement 1000 lignes
)
```

### Écriture

```python
# Simple
df.to_csv('output.csv', index=False)

# Avec options
df.to_csv(
    'output.csv',
    index=False,
    encoding='utf-8',
    quotechar='"',
    quoting='minimal'
)
```

### Transformations courantes

```python
# Filtrer
df_filtered = df[df['age'] > 25]
df_filtered = df[df['email'].str.contains('gmail')]

# Sélectionner colonnes
df_subset = df[['id', 'nom', 'email']]
df_subset = df.drop(columns=['unwanted'])

# Ajouter colonne
df['full_name'] = df['first_name'] + ' ' + df['last_name']

# Grouper et agréger
df.groupby('department').agg({
    'salary': ['sum', 'mean'],
    'id': 'count'
})

# Jointure
result = pd.merge(users_df, orders_df, on='id', how='inner')

# Trier
df_sorted = df.sort_values('salary', ascending=False)

# Pivot
pivot_df = df.pivot_table(
    values='amount',
    index='category',
    columns='month',
    aggfunc='sum'
)
```

### Lecture par chunks (pour gros fichiers)

```python
# Important pour fichiers > RAM disponible
for chunk in pd.read_csv('huge.csv', chunksize=100_000):
    process_chunk(chunk)
    # Libère la mémoire à chaque itération
```

### JSON avec Pandas

```python
# Lire JSON
df = pd.read_json('data.json')

# JSON Lines (une ligne = un JSON)
df = pd.read_json('data.jsonl', lines=True)

# Écrire
df.to_json('output.json', orient='records')  # Array d'objets
df.to_json('output.jsonl', orient='records', lines=True)  # JSONL
```

---

## Polars — La révolution performance

### Installation

```bash
pip install polars
```

### Avantages

- ⚡ **100x plus rapide** que Pandas sur gros fichiers
- 🧠 **Consomme moins de mémoire**
- 🔄 **Lazy evaluation** (évalue seulement si nécessaire)
- 🔗 **API similaire** à Pandas (courbe d'apprentissage faible)

### Lecture

```python
import polars as pl

# Eager (charge tout)
df = pl.read_csv('data.csv')

# Lazy (optimise avant d'exécuter)
df = pl.scan_csv('huge.csv').collect()

# Avec types explicites
df = pl.read_csv(
    'data.csv',
    dtypes={'id': pl.Int32, 'salary': pl.Float64}
)
```

### Transformations

```python
# Filtrer
result = df.filter(pl.col('age') > 25)

# Sélectionner
result = df.select(['id', 'nom', 'email'])

# Ajouter colonne
result = df.with_columns(
    full_name=(pl.col('first_name') + ' ' + pl.col('last_name'))
)

# Grouper
result = df.groupby('department').agg(
    pl.col('salary').sum().alias('total'),
    pl.col('id').count().alias('count')
)

# Jointure
result = users_df.join(orders_df, on='id')
```

### Performance : Pandas vs Polars

```python
import timeit

# Pandas : 15 seconds
%timeit -n 1 -r 1 pd.read_csv('1GB.csv').groupby('category')['amount'].sum()

# Polars : 0.3 seconds
%timeit -n 1 -r 1 pl.read_csv('1GB.csv').groupby('category').agg(pl.col('amount').sum())

# Ratio : 50x plus rapide!
```

### Lazy evaluation

```python
# Polars optimise automatiquement
query = (
    pl.scan_csv('huge.csv')
    .filter(pl.col('age') > 25)
    .select(['id', 'nom', 'salary'])
    .groupby('department').agg(pl.col('salary').mean())
    .sort('salary', descending=True)
    .limit(10)
)

result = query.collect()  # Exécute une seule fois, optimisé
```

---

## Dask & Modin — Parallélisation transparente

### Dask

Installation :
```bash
pip install dask[dataframe]
```

Concept : Divise les données et parallélise automatiquement.

```python
import dask.dataframe as dd

# Lecture parallelisée
df = dd.read_csv('huge.csv')  # Automatiquement parallélisé

# Transformation (lazy)
result = df[df['age'] > 25]['salary'].mean()

# Compute (exécuter)
final_result = result.compute()
```

### Modin

Installation :
```bash
pip install modin
```

Drop-in replacement pour Pandas, utilise tous les CPU cores.

```python
# Remplacer simplement
import modin.pandas as pd  # Au lieu de pandas

# API identique, automatiquement parallélisé
df = pd.read_csv('huge.csv')  # Distribuée sur tous les cores
result = df.groupby('department')['salary'].mean()  # Ultra-rapide
```

---

## PyArrow & Parquet — Optimisation I/O

### Installation

```bash
pip install pyarrow
```

### Avantages du Parquet

- 🗜️ **Compression excellent** (10x moins volumineux que CSV)
- ⚡ **Lecture columnar** (lire seulement colonnes nécessaires)
- 🔒 **Schéma strict** (validation de type)
- 🌐 **Format standard** (utilisé par Spark, BigQuery, etc.)

### Conversion CSV → Parquet

```python
import pyarrow.csv as pcsv
import pyarrow.parquet as pq

# Lire CSV
table = pcsv.read_csv('data.csv')

# Écrire Parquet
pq.write_table(table, 'data.parquet')

# Lire Parquet
table = pq.read_table('data.parquet')

# Convertir vers Pandas
df = table.to_pandas()
```

### Streaming pour gros fichiers

```python
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq

# Lire CSV par chunks et écrire Parquet
chunks = pd.read_csv('huge.csv', chunksize=100_000)
writer = None

for i, chunk in enumerate(chunks):
    table = pa.Table.from_pandas(chunk)
    if writer is None:
        writer = pq.ParquetWriter('output.parquet', table.schema)
    writer.write_table(table)

if writer:
    writer.close()
```

### Compression

```python
# Sans compression (défaut)
pq.write_table(table, 'data.parquet')

# Avec compression (gzip, snappy, brotli)
pq.write_table(table, 'data.parquet', compression='snappy')
# snappy = bon compromis vitesse/compression
```

### Filtrage Parquet (sans charger tout)

```python
# Lire seulement certaines colonnes
df = pd.read_parquet('data.parquet', columns=['id', 'nom'])

# Lire avec filtre (pushdown)
# ← Polars et Spark peuvent filtrer avant de charger
```

---

## Comparatif frameworks

### Tableau comparatif

| Framework | Vitesse | Mémoire | Ease | Gros vol. | Distribué | Cas d'usage |
|-----------|---------|---------|------|-----------|-----------|------------|
| **csv** | ⚡ Rapide | 💾 Bas | ✅ Simple | ❌ | ❌ | Scripts simples |
| **pandas** | ⚠️ Moyen | ⚠️ Moyen | ✅ Simple | ⚠️ Chunks | ❌ | Data science, analytics |
| **polars** | ⚡⚡⚡ | 💾💾 Très bas | ✅ Simple | ✅ | ✅ Lazy | Gros fichiers, streaming |
| **dask** | ⚡⚡ | 💾💾 Bas | ✅ Similaire Pandas | ✅ | ✅ Cluster | Distributed computing |
| **modin** | ⚡⚡ | 💾💾 Bas | ✅ Identique Pandas | ✅ | ✅ Multi-core | Drop-in Pandas |
| **pyarrow** | ⚡⚡⚡ | 💾 Bas | ⚠️ Intermédiaire | ✅ | N/A | Format columnar |

### Matrice décision

**Fichier < 100MB, manipulations simples ?**
→ **Pandas** (rapide, simple)

**Fichier 1-100GB, transformations complexes ?**
→ **Polars** (ultra-rapide, lazy eval)

**Données distribuées sur cluster ?**
→ **Dask** (Spark-like)

**Remplacer Pandas avec parallélisation ?**
→ **Modin** (drop-in)

**Conversion pour data warehouse ?**
→ **PyArrow + Parquet** (standard)

---

## 🎓 Exercices pratiques

### Exercice 7.1 : CSV standard
Écrivez un script Python qui lit un CSV ligne par ligne et calcule des statistiques simples.

### Exercice 7.2 : Pandas vs Polars
Créez un CSV de 500MB, comparez les temps d'exécution avec Pandas et Polars.

### Exercice 7.3 : Conversion Parquet
Convertissez un gros CSV en Parquet, mesurez la taille et la vitesse de lecture.

---

## 📚 Références

- **Pandas Docs** : https://pandas.pydata.org/docs/
- **Polars Book** : https://docs.pola-rs.com/
- **Dask Docs** : https://docs.dask.org/
- **PyArrow Docs** : https://arrow.apache.org/docs/python/

---

**Prêt pour Node.js ? → [Chapitre 8 : JavaScript/Node.js](./08_JavaScript_NodeJS.md)**
