# Chapitre 17 : Optimisations de Performance — Benchmarks & Tuning

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Benchmarker** pipelines CSV/JSON
- **Identifier** goulots d'étranglement
- **Optimiser** I/O, mémoire, CPU
- **Paralléliser** efficacement
- **Mesurer** improvements

---

## 📖 Table des matières

1. [Outils de benchmarking](#outils-de-benchmarking)
2. [Optimisations I/O](#optimisations-io)
3. [Optimisations mémoire](#optimisations-mémoire)
4. [Parallélisation](#parallélisation)
5. [Comparaisons de performance](#comparaisons-de-performance)
6. [Profiling en production](#profiling-en-production)

---

## Outils de benchmarking

### Python: timeit et cProfile

```python
import timeit
import cProfile
import pstats
from io import StringIO
import pandas as pd

# ========== TIMEIT: Simple benchmarking ==========

def read_csv_pandas():
    df = pd.read_csv('data.csv')
    return len(df)

def read_csv_csv():
    import csv
    count = 0
    with open('data.csv') as f:
        reader = csv.reader(f)
        for row in reader:
            count += 1
    return count

# Benchmark
n_runs = 10
time_pandas = timeit.timeit(read_csv_pandas, number=n_runs)
time_csv = timeit.timeit(read_csv_csv, number=n_runs)

print(f"Pandas: {time_pandas/n_runs:.4f}s per run")
print(f"CSV:    {time_csv/n_runs:.4f}s per run")
print(f"Speedup: {time_csv/time_pandas:.2f}x")

# ========== CPROFILE: Detailed profiling ==========

def process_csv():
    df = pd.read_csv('data.csv')
    df = df.dropna()
    df['total'] = df['price'] * df['qty']
    df.to_json('output.json', orient='records')

# Profile
pr = cProfile.Profile()
pr.enable()
process_csv()
pr.disable()

# Print results
s = StringIO()
ps = pstats.Stats(pr, stream=s).sort_stats('cumulative')
ps.print_stats(10)  # Top 10 functions
print(s.getvalue())

# Output example:
# ncalls  tottime  percall  cumtime  percall filename:lineno(function)
# 1       0.050   0.050    0.245   0.245 <stdin>:1(process_csv)
# 1       0.120   0.120    0.180   0.180 {method 'read_csv' of pandas}
# 1       0.040   0.040    0.040   0.040 {method 'to_json' of pandas}
```

### Python: memory_profiler

```python
from memory_profiler import profile
import pandas as pd

@profile
def load_and_process():
    # Line-by-line memory tracking
    df = pd.read_csv('large.csv')  # Memory spike
    df = df.dropna()               # Slightly less
    df['col'] = df['a'] + df['b']  # Temp variables
    return df

if __name__ == '__main__':
    load_and_process()

# Run: python -m memory_profiler script.py
# Output:
# Line #      Mem usage    Increment   Line Contents
# 4      25.5 MiB      0.0 MiB   @profile
# 5      25.6 MiB      0.1 MiB   def load_and_process():
# 6     385.2 MiB    359.6 MiB       df = pd.read_csv('large.csv')
# 7     380.1 MiB     -5.1 MiB       df = df.dropna()
# 8     385.5 MiB      5.4 MiB       df['col'] = df['a'] + df['b']
```

### Rust: Criterion

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use csv::Reader;

fn csv_read_1k(c: &mut Criterion) {
    c.bench_function("read 1k", |b| {
        b.iter(|| {
            let file = std::fs::File::open("test_1k.csv").unwrap();
            let mut reader = Reader::from_reader(file);
            let count = reader.records().count();
            black_box(count)
        })
    });
}

fn csv_read_100k(c: &mut Criterion) {
    c.bench_function("read 100k", |b| {
        b.iter(|| {
            let file = std::fs::File::open("test_100k.csv").unwrap();
            let mut reader = Reader::from_reader(file);
            let count = reader.records().count();
            black_box(count)
        })
    });
}

criterion_group!(benches, csv_read_1k, csv_read_100k);
criterion_main!(benches);
```

Run: `cargo bench`

---

## Optimisations I/O

### Buffering strategy

```python
import pandas as pd

# ❌ SLOW: Line-by-line reading
slow_lines = 0
with open('huge.csv') as f:
    for line in f:  # Default 8KB buffer
        slow_lines += 1

# ✅ FASTER: Chunked reading
df = pd.read_csv('huge.csv', chunksize=50000)
for chunk in df:
    process(chunk)

# ✅ FASTEST: Memory-mapped I/O + Polars
import polars as pl
df = pl.read_csv('huge.csv', batch_size=100000)
```

### Compression trade-offs

```python
import pandas as pd
import time

# Benchmark: Read from different formats

# CSV (uncompressed)
start = time.time()
df = pd.read_csv('data.csv')
csv_time = time.time() - start

# CSV gzipped
start = time.time()
df = pd.read_csv('data.csv.gz')
gz_time = time.time() - start

# Parquet
start = time.time()
df = pd.read_parquet('data.parquet')
parquet_time = time.time() - start

print(f"CSV:      {csv_time:.3f}s")
print(f"CSV.gz:   {gz_time:.3f}s (slower due to decompression)")
print(f"Parquet:  {parquet_time:.3f}s (fastest + efficient)")

# File sizes
import os
csv_size = os.path.getsize('data.csv')
gz_size = os.path.getsize('data.csv.gz')
parquet_size = os.path.getsize('data.parquet')

print(f"\nCSV:      {csv_size/1e9:.2f}GB")
print(f"CSV.gz:   {gz_size/1e9:.2f}GB (60% compression)")
print(f"Parquet:  {parquet_size/1e9:.2f}GB (40% compression)")
```

### Streaming vs loading all

```python
import pandas as pd
import json

# ========== CSV STREAMING (memory efficient) ==========
def process_csv_streaming(filename: str):
    """Process huge CSV line by line"""
    total = 0
    errors = 0
    
    df = pd.read_csv(filename, chunksize=10000)
    for chunk in df:
        # Process chunk (10k rows at a time)
        chunk_clean = chunk.dropna()
        chunk_clean = chunk_clean[chunk_clean['age'] > 18]
        
        total += len(chunk_clean)
        errors += len(chunk) - len(chunk_clean)
        
        # Output immediately (don't accumulate)
        chunk_clean.to_csv('output.csv', mode='a', header=False)

# ========== MEMORY FOOTPRINT ==========
# Loading all: 8GB RAM for 100M rows
# Streaming:  100MB RAM (one chunk)
```

---

## Optimisations mémoire

### Copy-on-write patterns

```python
import pandas as pd

# ❌ INEFFICIENT: Multiple copies
df1 = pd.read_csv('data.csv')
df2 = df1.copy()           # Copy 1
df3 = df2[['col1', 'col2']]  # Copy 2 (new DataFrame)
df4 = df3.dropna()          # Copy 3

# ✅ EFFICIENT: Chain operations
df = (
    pd.read_csv('data.csv')
    [['col1', 'col2']]
    .dropna()
)

# ✅ ULTRA-EFFICIENT: Use Polars (lazy evaluation)
import polars as pl
df = (
    pl.scan_csv('data.csv')
    .select(['col1', 'col2'])
    .drop_nulls()
    .collect()
)
```

### Dtype optimization

```python
import pandas as pd

# ❌ Inefficient dtypes
df = pd.read_csv('data.csv')  # All columns default to object/int64
df.info()

# Output:
# RangeIndex: 100000 entries, 0 to 99999
# Data columns:
# user_id: int64 (8 bytes) → could be int32 or category
# name:    object (8 bytes per cell) → could be category
# active:  bool (1 byte) → already optimal
# Memory: ~10 GB

# ✅ Specify dtypes
dtypes = {
    'user_id': 'uint32',          # 4 bytes instead of 8
    'name': 'category',           # 1-2 bytes with dictionary
    'active': 'bool',             # 1 byte
}
df = pd.read_csv('data.csv', dtype=dtypes)
df.info()

# Output: Memory: ~2 GB (5x improvement)
```

### Garbage collection tuning

```python
import gc
import pandas as pd

# Disable automatic GC during heavy processing
gc.disable()

# Process large dataset
df = pd.read_csv('huge.csv')
result = expensive_processing(df)

# Manual GC at strategic points
gc.collect()

# Re-enable for normal operation
gc.enable()
```

---

## Parallélisation

### Pandas groupby parallelization

```python
import pandas as pd
from multiprocessing import Pool
import numpy as np

def process_group(group_data):
    """Process one group"""
    name, group = group_data
    group['total'] = group['qty'] * group['price']
    return group.sum()

def parallel_groupby(df, groupby_col, func):
    """Parallelize groupby"""
    groups = df.groupby(groupby_col)
    
    with Pool(processes=4) as pool:
        results = pool.map(process_group, groups)
    
    return pd.concat(results, axis=1).T

# Usage
df = pd.read_csv('sales.csv')
result = parallel_groupby(df, 'product_id', process_group)
```

### Dask for distributed computing

```python
import dask.dataframe as dd
import pandas as pd

# Read huge CSV with Dask (lazy)
df = dd.read_csv('huge.csv', blocksize='50MB')

# Operations are lazy
df_filtered = df[df['age'] > 18]
df_grouped = df_filtered.groupby('country')['salary'].mean()

# Compute when needed
result = df_grouped.compute()  # Actually execute

# Multiple files
df_multi = dd.read_csv('/data/2024-*.csv')  # Glob pattern
result = df_multi.salary.mean().compute()
```

### Parallel processing with GNU Parallel

```bash
# Process multiple CSV files in parallel
parallel gzip {} ::: *.csv

# Process each file with Python script
parallel python process.py {} ::: /data/*.csv

# With progress bar
parallel --progress python process.py {} ::: /data/*.csv

# Using job distribution
parallel -j 4 process.sh {} ::: input*.csv
```

---

## Comparaisons de performance

### Benchmark CSV parsing

```python
import pandas as pd
import polars as pl
import dask.dataframe as dd
import csv
import json
import time

def benchmark_csv_parsing(file_path: str, num_runs: int = 3):
    """Compare different CSV parsing methods"""
    
    times = {}
    
    # 1. Python csv module
    def read_csv():
        rows = []
        with open(file_path) as f:
            reader = csv.DictReader(f)
            for row in reader:
                rows.append(row)
        return rows
    
    # 2. Pandas
    def read_pandas():
        return pd.read_csv(file_path)
    
    # 3. Polars
    def read_polars():
        return pl.read_csv(file_path)
    
    # 4. Dask
    def read_dask():
        return dd.read_csv(file_path)
    
    # Benchmark each
    for name, func in [("csv", read_csv), ("pandas", read_pandas), 
                       ("polars", read_polars), ("dask", read_dask)]:
        start = time.time()
        for _ in range(num_runs):
            result = func()
        avg_time = (time.time() - start) / num_runs
        times[name] = avg_time
    
    # Results
    print("CSV Parsing Benchmarks (seconds):")
    for name in sorted(times, key=times.get):
        print(f"  {name:12} {times[name]:.4f}s")
    
    # Relative to slowest
    slowest = max(times.values())
    print("\nRelative speed (vs slowest):")
    for name in sorted(times, key=lambda x: times[x], reverse=True):
        speedup = slowest / times[name]
        print(f"  {name:12} {speedup:.1f}x")

# Usage
benchmark_csv_parsing('large_file.csv', num_runs=5)

# Output:
# CSV Parsing Benchmarks (seconds):
#   polars       2.1234s
#   pandas       3.4567s
#   dask         4.5678s
#   csv          8.9012s
# 
# Relative speed (vs slowest):
#   csv          1.0x
#   dask         1.9x
#   pandas       2.6x
#   polars       4.2x
```

---

## Profiling en production

### Structured logging with timing

```python
import logging
import time
from functools import wraps
import json

logger = logging.getLogger(__name__)

def timed_function(func):
    """Decorator to log function timing"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        try:
            result = func(*args, **kwargs)
            duration = time.time() - start
            
            logger.info(json.dumps({
                'event': 'function_completed',
                'function': func.__name__,
                'duration_ms': duration * 1000,
                'status': 'success'
            }))
            return result
        except Exception as e:
            duration = time.time() - start
            logger.error(json.dumps({
                'event': 'function_failed',
                'function': func.__name__,
                'duration_ms': duration * 1000,
                'error': str(e)
            }))
            raise
    
    return wrapper

@timed_function
def process_csv(filename: str):
    df = pd.read_csv(filename)
    df = df.dropna()
    return df

# Usage logs timing to JSON for analysis
process_csv('data.csv')
```

### Memory profiling decorator

```python
import tracemalloc
from functools import wraps

def profile_memory(func):
    """Decorator to track memory usage"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        tracemalloc.start()
        
        result = func(*args, **kwargs)
        
        current, peak = tracemalloc.get_traced_memory()
        tracemalloc.stop()
        
        print(f"{func.__name__}:")
        print(f"  Current: {current / 1024 / 1024:.1f} MB")
        print(f"  Peak:    {peak / 1024 / 1024:.1f} MB")
        
        return result
    
    return wrapper

@profile_memory
def process_large_csv():
    df = pd.read_csv('huge.csv')
    df = df.dropna()
    return df
```

---

## 🎓 Exercices pratiques

### Exercice 17.1 : Benchmark
Comparez vitesses: pandas vs polars vs csv module.

### Exercice 17.2 : Profiling
Profilez script Python, identifiez goulots.

### Exercice 17.3 : Memory optimization
Réduisez memory footprint d'un script de 50%.

### Exercice 17.4 : Parallelization
Traitez 10 fichiers CSV en parallèle.

### Exercice 17.5 : Production profiling
Loggez timings/memory en JSON, analyez.

---

## 📚 Références

- **Pandas perf tips** : https://pandas.pydata.org/docs/user_guide/enhancing.html
- **Polars docs** : https://docs.pola-rs.com/
- **Dask documentation** : https://docs.dask.org/
- **Python timeit** : https://docs.python.org/3/library/timeit.html
- **Memory profiler** : https://pypi.org/project/memory-profiler/

---

**Prêt pour la sécurité? → [Chapitre 18 : Sécurité & Conformité](./18_Securite_Conformite.md)**
