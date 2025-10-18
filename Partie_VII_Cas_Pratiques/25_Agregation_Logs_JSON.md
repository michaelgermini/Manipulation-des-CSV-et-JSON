# Chapitre 25 : Agrégation de logs JSON pour analytics — ELK & Observabilité

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Parser** logs JSON structurés
- **Agréger** par service, niveau, erreur
- **Intégrer** avec ELK Stack
- **Exporter** pour analytics
- **Monitorer** et alerter en temps réel

---

## 📖 Table des matières

1. [Format des logs JSON](#format-des-logs-json)
2. [Parsing et nettoyage](#parsing-et-nettoyage)
3. [Agrégations courantes](#agrégations-courantes)
4. [ELK Stack et Elasticsearch](#elk-stack-et-elasticsearch)
5. [Pipeline complet](#pipeline-complet)
6. [Alertes et dashboards](#alertes-et-dashboards)

---

## Format des logs JSON

### Standard moderne (JSON Lines)

```json
{"timestamp":"2025-01-15T14:23:45.123Z","service":"payment-api","level":"INFO","message":"Payment processed","user_id":12345,"amount":99.99,"currency":"EUR","duration_ms":245}
{"timestamp":"2025-01-15T14:23:46.456Z","service":"payment-api","level":"ERROR","message":"Payment failed","user_id":12346,"error":{"code":"TIMEOUT","details":"3000ms exceeded"},"stack_trace":"...","severity":"critical"}
{"timestamp":"2025-01-15T14:23:47.789Z","service":"auth-service","level":"WARN","message":"Suspicious login attempt","user_id":12347,"ip":"192.168.1.1","attempts":5}
```

### Avantages des logs JSON
- ✅ Structure uniforme et parsable
- ✅ Contexte riche (user_id, request_id, etc)
- ✅ Facilite agrégation et alertes
- ✅ Compatible avec ELK, Datadog, Splunk
- ✅ Moins de faux positifs dans recherche texte

---

## Parsing et nettoyage

### Lecture streaming

```python
import json
from pathlib import Path
from datetime import datetime
import gzip

def read_log_lines(log_file: str):
    """Lire logs JSON ligne par ligne (streaming)"""
    opener = gzip.open if log_file.endswith('.gz') else open
    
    with opener(log_file, 'rt', encoding='utf-8') as f:
        line_num = 0
        for line in f:
            line_num += 1
            line = line.strip()
            
            # Skip empty lines
            if not line:
                continue
            
            try:
                log = json.loads(line)
                yield log
            except json.JSONDecodeError as e:
                print(f"Line {line_num}: Invalid JSON: {e}")
                continue

# Usage
for log in read_log_lines('app.log.jsonl'):
    print(f"{log['timestamp']} | {log['service']} | {log['level']} | {log['message']}")
```

### Validation et enrichissement

```python
import pandas as pd
from datetime import datetime

def enrich_logs(log_file: str) -> pd.DataFrame:
    """Enrichir et valider les logs"""
    logs = []
    
    for log in read_log_lines(log_file):
        # Valider champs obligatoires
        if not all(k in log for k in ['timestamp', 'service', 'level', 'message']):
            continue
        
        # Convertir timestamp
        try:
            log['timestamp'] = pd.to_datetime(log['timestamp'])
        except:
            continue
        
        # Normaliser level
        log['level'] = log['level'].upper()
        if log['level'] not in ['DEBUG', 'INFO', 'WARN', 'ERROR', 'CRITICAL']:
            log['level'] = 'UNKNOWN'
        
        # Extraire error_code si présent
        if 'error' in log and isinstance(log['error'], dict):
            log['error_code'] = log['error'].get('code')
        else:
            log['error_code'] = None
        
        logs.append(log)
    
    df = pd.DataFrame(logs)
    df['date'] = df['timestamp'].dt.date
    df['hour'] = df['timestamp'].dt.hour
    
    return df

# Usage
df = enrich_logs('app.log.jsonl')
print(f"Loaded {len(df)} logs")
print(df.head())
```

---

## Agrégations courantes

### Par service et level

```python
def analyze_logs_by_service_level(df: pd.DataFrame):
    """Compter logs par service et level"""
    agg = df.groupby(['service', 'level']).agg({
        'message': 'count',
        'timestamp': ['min', 'max']
    }).rename(columns={'message': 'count'})
    
    print("Logs par service et niveau:")
    print(agg)
    
    # Output:
    #                      count       timestamp
    #                                       min                      max
    # service       level
    # auth-service  ERROR      42  2025-01-15 00:00:00 2025-01-15 23:59:59
    # auth-service  INFO      1234 2025-01-15 00:00:00 2025-01-15 23:59:59
    # payment-api   ERROR      8   2025-01-15 12:30:45 2025-01-15 14:23:45
```

### Erreurs les plus fréquentes

```python
def top_errors(df: pd.DataFrame, n: int = 10):
    """Top N erreurs"""
    error_logs = df[df['level'] == 'ERROR']
    
    if error_logs.empty:
        print("Aucune erreur")
        return
    
    errors = error_logs.groupby('error_code').agg({
        'message': 'count',
        'service': lambda x: x.mode()[0] if len(x.mode()) > 0 else 'N/A'
    }).rename(columns={'message': 'count'}).sort_values('count', ascending=False)
    
    print(f"Top {n} erreurs:")
    print(errors.head(n))
    
    # Output:
    #             count         service
    # error_code
    # TIMEOUT       234       payment-api
    # UNAUTHORIZED  89        auth-service
    # DB_CONN_FAIL  45        user-service
```

### Latence par service

```python
def analyze_latency(df: pd.DataFrame):
    """Analyser latence (duration_ms)"""
    if 'duration_ms' not in df.columns:
        print("Colonne 'duration_ms' non trouvée")
        return
    
    latency = df.groupby('service')['duration_ms'].agg([
        'count', 'mean', 'median', 'min', 'max',
        ('p95', lambda x: x.quantile(0.95)),
        ('p99', lambda x: x.quantile(0.99))
    ]).round(2)
    
    print("Latence par service (ms):")
    print(latency)
    
    # Alerter si p99 > threshold
    threshold = 1000
    slow_services = latency[latency['p99'] > threshold]
    if not slow_services.empty:
        print(f"⚠️ Services lents (p99 > {threshold}ms):")
        print(slow_services)
```

### Tendance temporelle

```python
def hourly_trend(df: pd.DataFrame):
    """Tendance horaire"""
    trend = df.groupby(['date', 'hour', 'level']).agg({
        'message': 'count'
    }).rename(columns={'message': 'count'}).reset_index()
    
    # Pivot pour visualisation
    pivot = trend.pivot_table(
        index=['date', 'hour'],
        columns='level',
        values='count',
        fill_value=0
    )
    
    print("Tendance horaire:")
    print(pivot)
    
    # Export pour Grafana
    pivot.to_csv('hourly_trend.csv')
```

---

## ELK Stack et Elasticsearch

### Envoyer logs à Elasticsearch

```python
from elasticsearch import Elasticsearch
import json
from datetime import datetime

def send_to_elasticsearch(log_file: str, es_host: str = 'localhost:9200'):
    """Envoyer logs JSON à Elasticsearch"""
    
    es = Elasticsearch([es_host])
    
    count = 0
    for log in read_log_lines(log_file):
        # Valider timestamp ISO format
        if isinstance(log.get('timestamp'), str):
            log['timestamp'] = pd.to_datetime(log['timestamp']).isoformat()
        
        # Index name : logs-{service}-{date}
        service = log.get('service', 'unknown')
        date = log.get('timestamp', '')[:10]
        index_name = f"logs-{service}-{date}"
        
        try:
            es.index(index=index_name, body=log)
            count += 1
        except Exception as e:
            print(f"Error indexing: {e}")
    
    print(f"✓ Indexed {count} logs to Elasticsearch")

# Usage
send_to_elasticsearch('app.log.jsonl')
```

### Logstash configuration

```logstash
# logstash.conf
input {
  file {
    path => "/var/log/app/*.log.jsonl"
    codec => json
    start_position => "beginning"
  }
}

filter {
  # Parse timestamp
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }
  
  # Extract error code
  if [error] {
    mutate {
      add_field => { "error_code" => "%{[error][code]}" }
    }
  }
  
  # GeoIP lookup for IPs
  if [ip] {
    geoip {
      source => "ip"
    }
  }
}

output {
  elasticsearch {
    hosts => [ "localhost:9200" ]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

---

## Pipeline complet

### Script ETL logs JSON → Analytics

```python
import pandas as pd
import json
from datetime import datetime, timedelta
import os

def process_logs_for_analytics(log_dir: str, output_dir: str):
    """Pipeline complet : lecture → aggrégation → export"""
    
    print("[1] Lecture logs...")
    all_logs = []
    for log_file in os.listdir(log_dir):
        if log_file.endswith(('.jsonl', '.jsonl.gz')):
            for log in read_log_lines(os.path.join(log_dir, log_file)):
                all_logs.append(log)
    
    df = pd.DataFrame(all_logs)
    df['timestamp'] = pd.to_datetime(df['timestamp'])
    print(f"   → {len(df)} logs chargés")
    
    print("[2] Nettoyage et enrichissement...")
    df = df.dropna(subset=['service', 'level'])
    df['date'] = df['timestamp'].dt.date
    df['hour'] = df['timestamp'].dt.hour
    df['level'] = df['level'].str.upper()
    print(f"   → {len(df)} logs après nettoyage")
    
    print("[3] Agrégations...")
    
    # Par service/level
    by_service_level = df.groupby(['service', 'level']).agg({
        'message': 'count'
    }).reset_index().rename(columns={'message': 'count'})
    by_service_level.to_csv(f"{output_dir}/by_service_level.csv", index=False)
    
    # Erreurs
    errors = df[df['level'] == 'ERROR'].groupby('message').agg({
        'timestamp': 'count'
    }).reset_index().rename(columns={
        'message': 'error_msg',
        'timestamp': 'count'
    }).sort_values('count', ascending=False).head(20)
    errors.to_csv(f"{output_dir}/top_errors.csv", index=False)
    
    # Latence
    if 'duration_ms' in df.columns:
        latency = df.groupby('service')['duration_ms'].agg([
            'count', 'mean', 'median', ('p95', lambda x: x.quantile(0.95))
        ]).round(2).reset_index()
        latency.to_csv(f"{output_dir}/latency.csv", index=False)
    
    print(f"   → 3 fichiers exportés")
    
    print("[4] Export JSON pour BI...")
    bi_data = {
        'summary': {
            'total_logs': len(df),
            'date_range': {
                'start': df['timestamp'].min().isoformat(),
                'end': df['timestamp'].max().isoformat()
            },
            'services': df['service'].nunique(),
            'errors': len(df[df['level'] == 'ERROR'])
        },
        'by_level': df['level'].value_counts().to_dict(),
        'by_service': df['service'].value_counts().to_dict()
    }
    
    with open(f"{output_dir}/summary.json", 'w') as f:
        json.dump(bi_data, f, indent=2, default=str)
    
    print(f"   → summary.json créé")
    
    print("✓ Pipeline complété!")
    return df

# Usage
df = process_logs_for_analytics('/var/log/app/', './analytics_output/')
```

---

## Alertes et dashboards

### Alertes basées sur seuils

```python
def check_alerts(df: pd.DataFrame):
    """Vérifier conditions d'alerte"""
    
    alerts = []
    
    # Alert 1: Trop d'erreurs
    error_count = len(df[df['level'] == 'ERROR'])
    if error_count > 100:
        alerts.append({
            'severity': 'HIGH',
            'message': f'Trop d\'erreurs détectées: {error_count}',
            'timestamp': datetime.now()
        })
    
    # Alert 2: Service DOWN
    last_hour = datetime.now() - timedelta(hours=1)
    recent_logs = df[df['timestamp'] > last_hour]
    for service in df['service'].unique():
        service_logs = recent_logs[recent_logs['service'] == service]
        if len(service_logs) == 0:
            alerts.append({
                'severity': 'CRITICAL',
                'message': f'Service {service} n\'a pas loggé depuis 1h',
                'timestamp': datetime.now()
            })
    
    # Alert 3: Latence élevée
    if 'duration_ms' in df.columns:
        p99_latency = df['duration_ms'].quantile(0.99)
        if p99_latency > 5000:
            alerts.append({
                'severity': 'MEDIUM',
                'message': f'Latence P99 très élevée: {p99_latency:.0f}ms',
                'timestamp': datetime.now()
            })
    
    # Alert 4: Erreur spécifique
    db_errors = len(df[df['message'].str.contains('DATABASE', case=False, na=False)])
    if db_errors > 10:
        alerts.append({
            'severity': 'HIGH',
            'message': f'Erreurs database: {db_errors}',
            'timestamp': datetime.now()
        })
    
    return alerts

# Usage
alerts = check_alerts(df)
for alert in alerts:
    print(f"🚨 [{alert['severity']}] {alert['message']}")
```

### Dashboard Grafana (JSON Model)

```json
{
  "dashboard": {
    "title": "Logs Analytics",
    "panels": [
      {
        "title": "Logs par service",
        "type": "piechart",
        "targets": [
          {
            "expr": "count by (service) (logs)",
            "legendFormat": "{{service}}"
          }
        ]
      },
      {
        "title": "Erreurs dans dernière heure",
        "type": "stat",
        "targets": [
          {
            "expr": "count(logs{level=\"ERROR\"}[1h])"
          }
        ]
      },
      {
        "title": "Latence P99",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, duration_ms)"
          }
        ]
      }
    ]
  }
}
```

---

## 🎓 Exercices pratiques

### Exercice 25.1 : Parsing logs
Lisez 1000 lignes de logs JSON, validez et affichez stats.

### Exercice 25.2 : Agrégations
Calculez top 5 erreurs, latence moyenne par service.

### Exercice 25.3 : Exportexport
Générez fichiers CSV pour Tableau/Power BI.

### Exercice 25.4 : Elasticsearch
Envoyez logs à Elasticsearch local (Docker).

### Exercice 25.5 : Alertes
Implémentez système d'alerte pour anomalies.

---

## 📚 Références

- **Elasticsearch** : https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
- **Logstash** : https://www.elastic.co/guide/en/logstash/current/index.html
- **Kibana** : https://www.elastic.co/guide/en/kibana/current/index.html
- **JSON Lines format** : http://jsonlines.org/

---

**Prêt pour pipelines complets? → [Chapitre 26 : Pipeline Complet](./26_Pipeline_Complet.md)**
