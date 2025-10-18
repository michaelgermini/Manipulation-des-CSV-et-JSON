# Chapitre 15 : Monitoring & Observabilité — Métriques, Logs, Traces

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Instrumenter** pipelines CSV/JSON
- **Collecter** métriques (Prometheus, Datadog)
- **Corréler** logs et traces (ELK, Jaeger)
- **Alerter** sur anomalies
- **Debugger** problèmes production

---

## 📖 Table des matières

1. [Concepts de base](#concepts-de-base)
2. [Métriques avec Prometheus](#métriques-avec-prometheus)
3. [Logs centralisés (ELK Stack)](#logs-centralisés-elk-stack)
4. [Distributed tracing (Jaeger)](#distributed-tracing-jaeger)
5. [Dashboards (Grafana)](#dashboards-grafana)
6. [Alertes et SLOs](#alertes-et-slos)

---

## Concepts de base

### Les 3 piliers de l'observabilité

```
┌─────────────────────────────────────────────┐
│         OBSERVABILITÉ                       │
├─────────────────────────────────────────────┤
│                                             │
│  1. MÉTRIQUES (Prometheus)                 │
│     → Nombres + timestamps                 │
│     → CPU, Memory, Throughput              │
│                                             │
│  2. LOGS (ELK/Splunk)                      │
│     → Événements structurés                │
│     → Diagnostics détaillés                │
│                                             │
│  3. TRACES (Jaeger/Zipkin)                 │
│     → Requêtes end-to-end                  │
│     → Latency breakdown                    │
│                                             │
└─────────────────────────────────────────────┘
```

### Métriques utiles pour CSV/JSON

| Métrique | Type | Seuil d'alerte |
|----------|------|--------|
| **Throughput** (records/sec) | Gauge | < 1000 |
| **Processing latency** (p99 ms) | Histogram | > 1000 |
| **Error rate** (%) | Counter | > 1% |
| **File size** (GB) | Gauge | > 100 |
| **Memory usage** (%) | Gauge | > 80 |
| **Processing duration** (sec) | Histogram | > 300 |

---

## Métriques avec Prometheus

### Instrumenter Python

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Définir métriques
csv_records_processed = Counter(
    'csv_records_processed_total',
    'Total CSV records processed',
    ['source', 'status']  # Labels
)

processing_duration = Histogram(
    'csv_processing_duration_seconds',
    'Time spent processing CSV',
    buckets=(0.1, 0.5, 1.0, 5.0, 10.0)
)

error_rate = Gauge(
    'csv_error_rate',
    'Current error rate (%)'
)

memory_usage = Gauge(
    'process_memory_mb',
    'Process memory in MB'
)

# Démarrer serveur Prometheus
start_http_server(8000)

def process_csv(filename: str):
    """Process CSV avec instrumentation"""
    start_time = time.time()
    errors = 0
    total = 0
    
    try:
        for line in open(filename):
            total += 1
            try:
                # Processing logic
                data = line.strip().split(',')
                csv_records_processed.labels(
                    source=filename,
                    status='success'
                ).inc()
            except Exception as e:
                errors += 1
                csv_records_processed.labels(
                    source=filename,
                    status='error'
                ).inc()
    
    finally:
        duration = time.time() - start_time
        processing_duration.observe(duration)
        
        # Calculer error rate
        if total > 0:
            error_rate.set((errors / total) * 100)
        
        print(f"Processed {total} records in {duration:.2f}s ({errors} errors)")

# Usage
process_csv('large_data.csv')

# Les métriques sont disponibles sur http://localhost:8000/metrics
```

### Prometheus configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'csv-pipeline'
    static_configs:
      - targets: ['localhost:8000']

  - job_name: 'elasticsearch'
    static_configs:
      - targets: ['localhost:9200']

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
```

### Lancer Prometheus

```bash
# Télécharger et démarrer
./prometheus --config.file=prometheus.yml

# Accéder à http://localhost:9090
```

---

## Logs centralisés (ELK Stack)

### Logstash pipeline

```logstash
# logstash.conf pour CSV pipelines

input {
  # Lire logs structurés
  file {
    path => "/var/log/csv-pipeline/*.log"
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
  
  # Extract metrics
  if [metric_name] {
    mutate {
      convert => { "metric_value" => "float" }
    }
  }
  
  # Add environment tags
  mutate {
    add_field => { "environment" => "production" }
    add_field => { "team" => "data-engineering" }
  }
  
  # Geolocate IPs
  if [client_ip] {
    geoip {
      source => "client_ip"
    }
  }
}

output {
  # Envoyer à Elasticsearch
  elasticsearch {
    hosts => [ "localhost:9200" ]
    index => "csv-pipeline-%{+YYYY.MM.dd}"
  }
  
  # Debug output
  stdout { codec => rubydebug }
}
```

### Python → Elasticsearch

```python
from elasticsearch import Elasticsearch
import logging
import json
from datetime import datetime

class ElasticsearchHandler(logging.Handler):
    """Custom logging handler pour Elasticsearch"""
    
    def __init__(self, hosts=['localhost:9200']):
        super().__init__()
        self.es = Elasticsearch(hosts)
    
    def emit(self, record):
        """Envoyer log à Elasticsearch"""
        try:
            log_entry = {
                '@timestamp': datetime.utcnow().isoformat(),
                'level': record.levelname,
                'message': record.getMessage(),
                'logger': record.name,
                'module': record.module,
            }
            
            # Ajouter contextes custom
            if hasattr(record, 'csv_file'):
                log_entry['csv_file'] = record.csv_file
            if hasattr(record, 'records_processed'):
                log_entry['records_processed'] = record.records_processed
            
            index = f"logs-{datetime.utcnow():%Y.%m.%d}"
            self.es.index(index=index, body=log_entry)
        except Exception:
            self.handleError(record)

# Setup logging
logger = logging.getLogger('csv-pipeline')
logger.addHandler(ElasticsearchHandler())

def process_with_logging(filename: str):
    """Traiter CSV avec logs → Elasticsearch"""
    for i, line in enumerate(open(filename)):
        if i % 10000 == 0:
            extra = {'csv_file': filename, 'records_processed': i}
            logger.info(f"Processing {i} records", extra=extra)

# Usage
process_with_logging('large_data.csv')

# Rechercher dans Kibana:
# GET /logs-*/_search?q=csv_file:large_data.csv
```

---

## Distributed tracing (Jaeger)

### Instrumenter avec OpenTelemetry

```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Setup Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831,
)

trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

tracer = trace.get_tracer(__name__)

def extract_csv(filename: str):
    """Extraire CSV avec tracing"""
    with tracer.start_as_current_span("extract_csv") as span:
        span.set_attribute("csv.filename", filename)
        span.set_attribute("csv.source", "local_disk")
        
        data = []
        with open(filename) as f:
            for i, line in enumerate(f):
                if i == 0:
                    continue  # Skip header
                data.append(line.strip().split(','))
        
        span.set_attribute("csv.record_count", len(data))
        return data

def transform_csv(data: list):
    """Transformer données avec tracing"""
    with tracer.start_as_current_span("transform_csv") as span:
        span.set_attribute("csv.input_records", len(data))
        
        # Processing
        transformed = []
        errors = 0
        for record in data:
            try:
                # Transform logic
                transformed.append(transform_record(record))
            except Exception as e:
                errors += 1
                span.record_exception(e)
        
        span.set_attribute("csv.output_records", len(transformed))
        span.set_attribute("csv.errors", errors)
        return transformed

def export_json(data: list, output_file: str):
    """Exporter JSON avec tracing"""
    with tracer.start_as_current_span("export_json") as span:
        span.set_attribute("json.output_file", output_file)
        span.set_attribute("json.record_count", len(data))
        
        import json
        with open(output_file, 'w') as f:
            json.dump(data, f, indent=2)

# Pipeline complet avec traces
def main():
    with tracer.start_as_current_span("csv_json_pipeline") as root_span:
        root_span.set_attribute("pipeline.name", "main")
        
        data = extract_csv('input.csv')
        transformed = transform_csv(data)
        export_json(transformed, 'output.json')

if __name__ == "__main__":
    main()
    # Accédez à Jaeger UI sur http://localhost:16686
```

---

## Dashboards (Grafana)

### Créer dashboard JSON

```json
{
  "dashboard": {
    "title": "CSV Pipeline Monitoring",
    "panels": [
      {
        "id": 1,
        "title": "Records Processed Per Second",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(csv_records_processed_total[1m])",
            "legendFormat": "{{source}}"
          }
        ],
        "yaxes": [{ "label": "records/sec" }]
      },
      {
        "id": 2,
        "title": "Processing Duration P99",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, csv_processing_duration_seconds_bucket)"
          }
        ],
        "yaxes": [{ "label": "seconds" }]
      },
      {
        "id": 3,
        "title": "Error Rate",
        "type": "gauge",
        "targets": [
          {
            "expr": "csv_error_rate"
          }
        ],
        "thresholds": "1,5"
      },
      {
        "id": 4,
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "process_memory_mb"
          }
        ]
      }
    ]
  }
}
```

---

## Alertes et SLOs

### Alertes Prometheus

```yaml
# prometheus_alerts.yml
groups:
  - name: csv_pipeline
    rules:
      # Alert 1: High error rate
      - alert: HighCSVErrorRate
        expr: csv_error_rate > 5
        for: 5m
        annotations:
          summary: "High CSV error rate detected"
          description: "Error rate is {{ $value }}%"
      
      # Alert 2: Slow processing
      - alert: SlowCSVProcessing
        expr: histogram_quantile(0.99, csv_processing_duration_seconds) > 10
        for: 10m
        annotations:
          summary: "CSV processing is slow"
          description: "P99 latency is {{ $value }}s"
      
      # Alert 3: High memory
      - alert: HighMemoryUsage
        expr: process_memory_mb > 8000
        for: 5m
        annotations:
          summary: "CSV process using high memory"
          description: "Memory usage: {{ $value }}MB"
      
      # Alert 4: Pipeline failed
      - alert: PipelineFailed
        expr: increase(csv_records_processed_total{status="error"}[5m]) > 1000
        for: 1m
        annotations:
          summary: "CSV pipeline failed"
```

### SLO (Service Level Objectives)

```python
def calculate_slos(metrics_data):
    """Calculer SLOs pour pipeline CSV"""
    
    # SLO 1: Availability
    total_runs = metrics_data['total_pipeline_runs']
    failed_runs = metrics_data['failed_pipeline_runs']
    availability = ((total_runs - failed_runs) / total_runs) * 100
    slo_availability = 99.9
    
    print(f"Availability: {availability:.2f}% (SLO: {slo_availability}%)")
    if availability < slo_availability:
        print("❌ SLO violated!")
    
    # SLO 2: Latency (P99 < 5 seconds)
    p99_latency = metrics_data['p99_processing_duration']
    slo_latency = 5.0
    
    print(f"P99 Latency: {p99_latency:.2f}s (SLO: {slo_latency}s)")
    if p99_latency > slo_latency:
        print("❌ SLO violated!")
    
    # SLO 3: Error rate < 0.5%
    error_rate = metrics_data['error_rate']
    slo_error = 0.5
    
    print(f"Error rate: {error_rate:.2f}% (SLO: {slo_error}%)")
    if error_rate > slo_error:
        print("❌ SLO violated!")
```

---

## 🎓 Exercices pratiques

### Exercice 15.1 : Métriques
Instrumentez un pipeline CSV avec Prometheus.

### Exercice 15.2 : Logs
Envoyez logs JSON à ELK Stack.

### Exercice 15.3 : Traces
Tracez pipeline end-to-end avec Jaeger.

### Exercice 15.4 : Dashboard
Créez dashboard Grafana pour métriques CSV.

### Exercice 15.5 : Alertes
Configurez alertes pour anomalies pipeline.

---

## 📚 Références

- **Prometheus** : https://prometheus.io/
- **Grafana** : https://grafana.com/
- **Elastic Stack** : https://www.elastic.co/
- **Jaeger** : https://www.jaegertracing.io/
- **OpenTelemetry** : https://opentelemetry.io/

---

**Prêt pour CI/CD? → [Chapitre 16 : CI/CD & GitOps](./16_CI_CD_GitOps.md)**
