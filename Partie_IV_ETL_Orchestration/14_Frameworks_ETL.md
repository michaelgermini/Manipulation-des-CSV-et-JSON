# Chapitre 14 : Frameworks ETL/Orchestration — Airflow, Prefect, Dagster

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Construire DAGs avec **Apache Airflow**
- Orchestrer pipelines CSV/JSON
- Monitorer et debugger pipelines
- Gérer dépendances et retries
- Choisir entre Airflow, Prefect, Dagster

---

## 📖 Table des matières

1. [Apache Airflow — Le standard](#apache-airflow--le-standard)
2. [Concepts clés](#concepts-clés)
3. [Premier DAG CSV/JSON](#premier-dag-csvjson)
4. [Patterns avancés](#patterns-avancés)
5. [Prefect et Dagster](#prefect-et-dagster)
6. [Comparatif](#comparatif)

---

## Apache Airflow — Le standard

### Installation

```bash
# Installation simple
pip install apache-airflow

# Avec extras
pip install 'apache-airflow[google,postgres,ssh]'

# Initialize database
airflow db init

# Créer user
airflow users create --username admin --password admin --firstname Admin --lastname Admin --role Admin --email admin@example.com

# Start webserver
airflow webserver -p 8080

# Start scheduler (autre terminal)
airflow scheduler
```

Accédez à http://localhost:8080 avec login: admin / password: admin

### Structure projet

```
my_airflow_project/
├─ dags/
│  ├─ csv_etl_dag.py
│  ├─ json_pipeline_dag.py
│  └─ shared_tasks.py
├─ plugins/
│  └─ custom_operators.py
├─ config/
│  └─ airflow.cfg
└─ requirements.txt
```

---

## Concepts clés

### DAG (Directed Acyclic Graph)

**DAG** = Graphe de tâches avec dépendances ordonnées.

```python
from airflow import DAG
from datetime import datetime

# Créer un DAG
with DAG(
    'my_pipeline',
    start_date=datetime(2025, 1, 1),
    schedule_interval='@daily',  # Quotidien
    catchup=False  # Pas d'exécution rétroactive
) as dag:
    # Les tâches vont ici
    pass
```

### Tasks (Tâches)

```python
from airflow.operators.python import PythonOperator

def extract_csv():
    """Extraire données CSV"""
    import pandas as pd
    df = pd.read_csv('data.csv')
    df.to_json('extracted.json', orient='records')

extract_task = PythonOperator(
    task_id='extract_csv',
    python_callable=extract_csv
)
```

### Dépendances

```python
# Linear
task1 >> task2 >> task3

# Parallèle
task1 >> [task2, task3] >> task4

# Complex
task1 >> task2
task1 >> task3
task2 >> task4
task3 >> task4
```

---

## Premier DAG CSV/JSON

### DAG simple : CSV → Nettoyage → JSON

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import pandas as pd
import json
import os

default_args = {
    'owner': 'data-team',
    'start_date': datetime(2025, 1, 1),
    'retries': 3,
    'retry_delay': timedelta(minutes=5)
}

with DAG(
    'csv_to_json_pipeline',
    default_args=default_args,
    schedule_interval='@daily',
    catchup=False
) as dag:

    def extract_csv():
        """Lire CSV source"""
        df = pd.read_csv('/data/input/users.csv')
        df.to_csv('/tmp/extracted.csv', index=False)
        print(f"Extracted {len(df)} rows")

    def clean_data():
        """Nettoyer données"""
        df = pd.read_csv('/tmp/extracted.csv')
        
        # Nettoyage
        df['email'] = df['email'].str.lower().str.strip()
        df = df.dropna(subset=['email'])
        df = df.drop_duplicates(subset=['email'])
        
        df.to_csv('/tmp/cleaned.csv', index=False)
        print(f"Cleaned to {len(df)} rows")

    def convert_to_json():
        """Convertir en JSON"""
        df = pd.read_csv('/tmp/cleaned.csv')
        
        # Convertir types
        df['age'] = df['age'].astype(int)
        df['created'] = pd.to_datetime(df['created'])
        
        # Exporter JSON
        df.to_json(
            '/data/output/users.json',
            orient='records',
            date_format='iso',
            indent=2
        )
        print("Exported to JSON")

    def validate():
        """Valider résultat"""
        with open('/data/output/users.json') as f:
            data = json.load(f)
        assert isinstance(data, list), "Invalid JSON structure"
        assert len(data) > 0, "No records"
        print(f"✓ Validation OK: {len(data)} records")

    # Créer tâches
    extract = PythonOperator(
        task_id='extract_csv',
        python_callable=extract_csv
    )

    clean = PythonOperator(
        task_id='clean_data',
        python_callable=clean_data
    )

    convert = PythonOperator(
        task_id='convert_to_json',
        python_callable=convert_to_json
    )

    validate_task = PythonOperator(
        task_id='validate',
        python_callable=validate
    )

    # Définir dépendances
    extract >> clean >> convert >> validate_task
```

**Visualisation dans Airflow UI:**
```
extract_csv → clean_data → convert_to_json → validate
```

---

## Patterns avancés

### Branching conditionnel

```python
def check_data_quality():
    """Retourner 'good' ou 'bad' branch"""
    df = pd.read_csv('/tmp/cleaned.csv')
    if len(df) > 100:
        return 'good_data'
    else:
        return 'bad_data'

from airflow.operators.python import BranchPythonOperator

branch = BranchPythonOperator(
    task_id='quality_check',
    python_callable=check_data_quality
)

# Branch tasks
good_path = PythonOperator(
    task_id='good_data',
    python_callable=lambda: print("Processing good data")
)

bad_path = PythonOperator(
    task_id='bad_data',
    python_callable=lambda: print("Alert: Low quality")
)

branch >> [good_path, bad_path]
```

### Parallélisation dynamique

```python
from airflow.models import Variable
from airflow.operators.python import PythonOperator

def process_file(file_path: str):
    """Traiter un fichier"""
    df = pd.read_csv(file_path)
    df_clean = df.dropna()
    return len(df_clean)

# Créer tâches dynamiquement
files = ['/data/file1.csv', '/data/file2.csv', '/data/file3.csv']

tasks = []
for i, file in enumerate(files):
    task = PythonOperator(
        task_id=f'process_file_{i}',
        python_callable=process_file,
        op_kwargs={'file_path': file}
    )
    tasks.append(task)

# Exécuter en parallèle
extract >> tasks >> convert
```

### Transfert de données entre tâches

```python
def push_data():
    """Pousser données dans XCom"""
    data = pd.read_csv('/tmp/data.csv')
    return data.to_json()  # Sérialiser

def pull_data(ti):
    """Récupérer données de XCom"""
    json_str = ti.xcom_pull(task_ids='push_data')
    df = pd.read_json(json_str)
    print(df)

push_task = PythonOperator(
    task_id='push_data',
    python_callable=push_data
)

pull_task = PythonOperator(
    task_id='pull_data',
    python_callable=pull_data
)

push_task >> pull_task
```

### Alertes et notifications

```python
from airflow.utils.email import send_email

def on_success_callback(context):
    """Notification succès"""
    send_email(
        to='team@example.com',
        subject=f"Pipeline {context['task'].task_id} succeeded",
        html_content=f"Pipeline ran successfully on {context['execution_date']}"
    )

def on_failure_callback(context):
    """Notification erreur"""
    send_email(
        to='team@example.com',
        subject=f"Pipeline {context['task'].task_id} FAILED",
        html_content=str(context['exception'])
    )

task = PythonOperator(
    task_id='critical_task',
    python_callable=my_function,
    on_success_callback=on_success_callback,
    on_failure_callback=on_failure_callback,
    retries=3,
    retry_delay=timedelta(minutes=5)
)
```

---

## Prefect et Dagster

### Prefect — Plus moderne que Airflow

```python
from prefect import flow, task
import pandas as pd

@task(retries=3, retry_delay_seconds=60)
def extract_csv(path: str):
    """Extraire CSV"""
    return pd.read_csv(path)

@task
def clean_data(df: pd.DataFrame):
    """Nettoyer"""
    return df.dropna().drop_duplicates()

@task
def export_json(df: pd.DataFrame, output_path: str):
    """Exporter JSON"""
    df.to_json(output_path, orient='records', indent=2)

@flow(name="CSV to JSON Pipeline")
def pipeline():
    df = extract_csv('/data/users.csv')
    df_clean = clean_data(df)
    export_json(df_clean, '/data/output.json')

# Lancer
if __name__ == "__main__":
    pipeline()
```

**Avantages Prefect:**
- ✅ Plus simple que Airflow
- ✅ API plus pythonic
- ✅ Cloud UI moderne
- ✅ Meilleure gestion d'erreurs

### Dagster — Orienté assets

```python
from dagster import asset, define_asset_job
import pandas as pd

@asset
def users_raw():
    """Asset brut"""
    return pd.read_csv('/data/users.csv')

@asset
def users_cleaned(users_raw: pd.DataFrame):
    """Asset nettoyé"""
    return users_raw.dropna().drop_duplicates()

@asset
def users_json(users_cleaned: pd.DataFrame):
    """Asset JSON"""
    users_cleaned.to_json('/data/output.json')
    return '/data/output.json'

# Define job
users_job = define_asset_job("users_pipeline")

# Lazy definition
defs = Definitions(
    assets=[users_raw, users_cleaned, users_json],
    jobs=[users_job]
)
```

**Avantages Dagster:**
- ✅ Asset-oriented (clearer lineage)
- ✅ Excellent for data platforms
- ✅ Type hints support
- ✅ Test-friendly

---

## Comparatif

| Aspect | Airflow | Prefect | Dagster |
|--------|---------|---------|---------|
| **Maturité** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Communauté** | Très grande | Croissante | Croissante |
| **Ease of use** | Modéré | Facile | Facile |
| **Scalabilité** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **UI/UX** | Fonctionnel | Moderne | Moderne |
| **Learning curve** | Steep | Moderate | Moderate |
| **Cloud support** | GCP, AWS | Cloud Prefect | Cloud Dagster |
| **Meilleur pour** | Large enterprises | Mid-size teams | Data platforms |

---

## 🎓 Exercices pratiques

### Exercice 14.1 : Premier DAG
Créez DAG simple : CSV → read → write JSON

### Exercice 14.2 : Dépendances
Ajoutez validation et alertes au DAG.

### Exercice 14.3 : Retries
Testez logique de retry et failure handling.

### Exercice 14.4 : Monitoring
Utilisez Airflow UI pour monitorer exécutions.

### Exercice 14.5 : Comparaison
Implémentez même pipeline en Prefect et Dagster.

---

## 📚 Références

- **Apache Airflow** : https://airflow.apache.org/
- **Prefect** : https://www.prefect.io/
- **Dagster** : https://dagster.io/
- **DAGs Best Practices** : https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html

---

**Prêt pour CI/CD? → [Chapitre 15 : CI/CD & GitOps](./15_CI_CD_GitOps.md)**
