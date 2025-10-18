# Chapitre 21 : Stream Processing Real-time — Kafka, Spark, Flink

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Kafka** pour event streaming
- **Spark Structured Streaming** pour analytics
- **Apache Flink** pour complex events
- **Windowing** (tumbling, sliding, session)
- **State management** pour stateful operations

---

## 📖 Table des matières

1. [Kafka Streaming](#kafka-streaming)
2. [Spark Structured Streaming](#spark-structured-streaming)
3. [Windowing Strategies](#windowing-strategies)
4. [State Management](#state-management)
5. [Apache Flink](#apache-flink)

---

## Kafka Streaming

### Architecture Kafka

```
┌──────────────┐
│   Producers  │  (envoient events)
└──────────────┘
       │
       ↓
┌──────────────────────────────────────┐
│  Kafka Cluster (Topics & Partitions) │
│  ┌──────────┐  ┌──────────┐         │
│  │Partition │  │Partition │  ...    │
│  └──────────┘  └──────────┘         │
└──────────────────────────────────────┘
       ↑
       │
┌──────────────┐
│  Consumers   │  (lisent events)
└──────────────┘
```

### Producer (envoyer events)

```python
from kafka import KafkaProducer
import json
import time
from datetime import datetime

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    acks='all',  # Wait for all replicas
    retries=3
)

# Envoyer events
for i in range(1000):
    event = {
        'timestamp': datetime.now().isoformat(),
        'user_id': f'user_{i % 100}',
        'amount': 100 + (i % 500),
        'action': 'purchase',
        'region': ['US', 'EU', 'ASIA'][i % 3]
    }
    
    # Send asynchronously
    future = producer.send('transactions', value=event)
    record_metadata = future.get(timeout=10)
    
    print(f"Event sent to partition {record_metadata.partition} at offset {record_metadata.offset}")
    time.sleep(0.1)

producer.close()
```

### Consumer (recevoir events)

```python
from kafka import KafkaConsumer
import json
from datetime import datetime

consumer = KafkaConsumer(
    'transactions',
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',  # Start from beginning
    group_id='analytics-group',
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    max_poll_records=100  # Batch size
)

# Traiter events en streaming
print("Listening for events...")
for message in consumer:
    event = message.value
    
    # Process event
    print(f"[{event['timestamp']}] {event['action']}: ${event['amount']} from {event['user_id']}")
    
    # Save to DB, send to analytics, etc.
    # save_to_database(event)
```

### Consumer groups

```python
# Multiple consumers en parallèle
consumer1 = KafkaConsumer(
    'transactions',
    bootstrap_servers=['localhost:9092'],
    group_id='analytics-group',
    auto_offset_reset='earliest'
)

consumer2 = KafkaConsumer(
    'transactions',
    bootstrap_servers=['localhost:9092'],
    group_id='analytics-group',  # Même groupe
    auto_offset_reset='earliest'
)

# Chaque partition est assignée à un consumer
# Permet parallélisation automatique
```

---

## Spark Structured Streaming

### Setup Spark Session

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col, window, sum as spark_sum, avg, count
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DoubleType

spark = SparkSession.builder \
    .appName("StructuredStreaming") \
    .config("spark.sql.streaming.schemaInference", "true") \
    .getOrCreate()

spark.sparkContext.setLogLevel("WARN")
```

### Reading from Kafka

```python
# Read from Kafka topic
df = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "transactions") \
    .option("startingOffsets", "earliest") \
    .load()

# Define JSON schema
schema = StructType([
    StructField("timestamp", StringType()),
    StructField("user_id", StringType()),
    StructField("amount", IntegerType()),
    StructField("action", StringType()),
    StructField("region", StringType())
])

# Parse JSON
parsed_df = df.select(
    from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")

parsed_df.printSchema()
```

### Window aggregation

```python
from pyspark.sql.functions import to_timestamp, window

# Convert timestamp string to timestamp type
events_df = parsed_df.withColumn(
    "event_time",
    to_timestamp(col("timestamp"), "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
)

# 5-minute tumbling window aggregation
windowed = events_df \
    .groupby(
        window(col("event_time"), "5 minutes"),
        col("region")
    ) \
    .agg(
        spark_sum("amount").alias("total_revenue"),
        count("user_id").alias("num_transactions"),
        avg("amount").alias("avg_transaction")
    )

# Output to console
query = windowed \
    .writeStream \
    .format("console") \
    .option("truncate", False) \
    .option("checkpointLocation", "/tmp/checkpoint") \
    .start()

query.awaitTermination()
```

### Output modes

```python
# Mode 1: Append (only new results since last trigger)
query = windowed \
    .writeStream \
    .outputMode("append") \
    .format("console") \
    .start()

# Mode 2: Update (only changed rows)
query = windowed \
    .writeStream \
    .outputMode("update") \
    .format("console") \
    .start()

# Mode 3: Complete (all rows every time)
query = windowed \
    .writeStream \
    .outputMode("complete") \
    .format("console") \
    .start()

# Write to Parquet
query = windowed \
    .writeStream \
    .outputMode("append") \
    .format("parquet") \
    .option("path", "/data/output") \
    .option("checkpointLocation", "/tmp/checkpoint") \
    .start()
```

---

## Windowing Strategies

### Tumbling (non-overlapping)

```python
# 5-minute windows: [00:00-05:00], [05:00-10:00], ...
windowed = events_df \
    .groupby(window(col("event_time"), "5 minutes")) \
    .agg(spark_sum("amount").alias("total"))
```

### Sliding (overlapping)

```python
# 10-minute windows, advanced by 5 minutes
# [00:00-10:00], [05:00-15:00], [10:00-20:00], ...
windowed = events_df \
    .groupby(window(
        col("event_time"), 
        "10 minutes",      # window duration
        "5 minutes"        # slide interval
    )) \
    .agg(spark_sum("amount").alias("total"))
```

### Session (event-based)

```python
from pyspark.sql.functions import session_window

# Close session after 10 minutes of inactivity
windowed = events_df \
    .groupby(
        session_window(col("event_time"), "10 minutes"),
        col("user_id")
    ) \
    .agg(spark_sum("amount").alias("session_total"))
```

---

## State Management

### Stateful operations

```python
from pyspark.sql.functions import col, max as spark_max

# Track cumulative stats per user
user_stats = events_df \
    .groupby("user_id") \
    .agg(
        spark_sum("amount").alias("total_spent"),
        count("*").alias("num_transactions"),
        spark_max("event_time").alias("last_seen")
    )

# Write to persistent store
query = user_stats \
    .writeStream \
    .outputMode("update") \
    .format("parquet") \
    .option("path", "/data/user_stats") \
    .option("checkpointLocation", "/tmp/user_checkpoint") \
    .start()

# Or write to database
query = user_stats \
    .writeStream \
    .outputMode("update") \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://localhost:5432/analytics") \
    .option("dbtable", "user_stats") \
    .option("user", "postgres") \
    .option("password", "password") \
    .option("checkpointLocation", "/tmp/db_checkpoint") \
    .start()
```

---

## Apache Flink

### Setup Flink

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.functions import MapFunction, FilterFunction
from pyflink.common import SimpleStringSchema

# Create execution environment
env = StreamExecutionEnvironment.get_execution_environment()
env.set_parallelism(4)

# Add checkpoint (for fault tolerance)
env.enable_checkpointing(60000)  # Every 60 seconds
```

### Event processing

```python
from pyflink.datastream.functions import MapFunction
import json
from datetime import datetime

class AugmentEventFunction(MapFunction):
    def map(self, event_str):
        event = json.loads(event_str)
        event['processed_at'] = datetime.now().isoformat()
        event['processed'] = True
        return json.dumps(event)

# Read from Kafka
kafka_stream = env.add_source(...)

# Apply transformation
augmented = kafka_stream.map(AugmentEventFunction())

# Sink to output
augmented.add_sink(...)

# Execute job
env.execute("Event Processing Job")
```

### Keyed state

```python
from pyflink.datastream.state import ValueStateDescriptor

class StatefulMap(KeyedProcessFunction):
    def open(self, runtime_context):
        self.state = runtime_context.get_state(
            ValueStateDescriptor("user_total", int)
        )
    
    def process_element(self, element, ctx):
        current = self.state.value() or 0
        current += element['amount']
        self.state.update(current)
        
        yield (element['user_id'], current)
```

---

## 🎓 Exercices pratiques

### Exercice 21.1 : Kafka basic
- Créez producer et consumer
- Envoyez 100 events
- Lisez et traitez les events

### Exercice 21.2 : Structured Streaming
- Lisez topic Kafka
- Calculez agrégations 5-min
- Exportez en Parquet

### Exercice 21.3 : Windowing
- Implémentez tumbling windows
- Implémentez sliding windows
- Comparez résultats

### Exercice 21.4 : Complex state
- Track utilisateurs actifs
- Calculez session totals
- Détectez anomalies

### Exercice 21.5 : Flink job
- Créez Flink job simple
- Lisez Kafka source
- Écrivez résultats

---

## 📊 Benchmarks

```
Architecture | Latency | Throughput | Scalability
Kafka        | 5-10ms  | 1M+/sec    | ⭐⭐⭐⭐⭐
Spark Stream | 100ms+  | 100K+/sec  | ⭐⭐⭐⭐⭐
Flink        | 1-10ms  | 1M+/sec    | ⭐⭐⭐⭐⭐
```

---

## 📚 Références

- **Apache Kafka** : https://kafka.apache.org/
- **Spark Streaming** : https://spark.apache.org/streaming/
- **Apache Flink** : https://flink.apache.org/
- **Streaming Concepts** : https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491901632/

---

**Prêt pour ML pipelines? → [Chapitre 28 : ML Pipelines & Feature Engineering](../Partie_VIII_Avancee/28_ML_Pipelines.md)**
