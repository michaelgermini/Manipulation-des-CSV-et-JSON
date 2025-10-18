# Chapitre 13 : Multi-langages — Choisir le bon outil

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Comparer** 6 langages pour CSV/JSON
- **Choisir** le meilleur selon context
- **Intégrer** multi-langages efficacement
- **Mesurer** trade-offs (vitesse, dev, maintenance)
- **Architecturer** solutions hybrides

---

## 📖 Table des matières

1. [Comparatif complet](#comparatif-complet)
2. [Performance benchmarks](#performance-benchmarks)
3. [Choix par use case](#choix-par-use-case)
4. [Intégration multi-langages](#intégration-multi-langages)
5. [Migration entre langages](#migration-entre-langages)

---

## Comparatif complet

### Tableau récapitulatif

| Aspect | Python | Go | Rust | Node.js | Java/Scala | R |
|--------|--------|-----|------|---------|-----------|---|
| **Vitesse** | 🐢🐢 | 🐇🐇 | 🐇🐇🐇 | 🐇 | 🐇🐇 | 🐢🐢 |
| **Memory** | 🟡 Modéré | ✅ Faible | ✅✅ Très faible | 🟡 Modéré | 🔴 Élevé | 🔴 Élevé |
| **Setup time** | ⚡ Instant | ⚠️ 5-10s | ⚠️ 30-60s | ⚡ Instant | ⚠️ 2-5s | ⚡ Instant |
| **Dev speed** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Stdlib** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Frameworks** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Learning curve** | Facile | Modéré | Difficile | Facile | Modéré | Facile |
| **Binaire** | ❌ Dépend runtime | ✅ Single 10MB | ✅ Single 5MB | ❌ Node requis | ❌ JVM requis | ❌ R requis |
| **Concurrence** | ⚠️ GIL | ✅✅ Goroutines | ✅✅ Async | ✅ Event-loop | ✅ Threads | ⚠️ Limitée |
| **Debugging** | ✅✅ Simple | ✅ Facile | ⚠️ Complexe | ✅ Simple | ⭐⭐ IDE good | ✅ Simple |

---

## Performance benchmarks

### CSV parsing (100MB)

```
Langage      Temps (sec)  Memory (MB)  Code lines
─────────────────────────────────────────────────
Rust         0.8          50          ~50
Go           1.2          80          ~30
Java         1.5          200         ~80
C#           1.6          150         ~60
Node.js      2.1          120         ~40
Python       5.3          300         ~20
R            8.9          400         ~15

* Benchmark sur CSV 100MB, 1M rows
```

### JSON processing (50MB)

```
Langage      Parse (ms)  Transform (ms)  Total (ms)
─────────────────────────────────────────────────
Rust         45         120             165
Go           60         140             200
Java         80         180             260
Node.js      90         200             290
Python       220        450             670
R            350        600             950

* Benchmark transform simples
```

---

## Choix par use case

### 1. CSV Processing - Batch simple (< 1GB)

**✅ Meilleur choix: Python**
```python
import pandas as pd
df = pd.read_csv('data.csv')
df['new_col'] = df['a'] + df['b']
df.to_csv('output.csv')
```
**Raison**: Rapidité de développement, 1 ligne pour 80% des cas

**Alternative**: R si analytics nécessaires
```r
library(readr)
df <- read_csv('data.csv')
df <- mutate(df, new_col = a + b)
write_csv(df, 'output.csv')
```

---

### 2. CSV Processing - Streaming (1GB+)

**✅ Meilleur choix: Go**
```go
file, _ := os.Open("huge.csv")
reader := csv.NewReader(file)
for {
    record, err := reader.Read()
    if err != nil { break }
    // Process line
}
```
**Raison**: Goroutines, performance, binaire standalone

**Alternative**: Rust pour ultra-performance
```rust
let reader = csv::Reader::from_path("huge.csv")?;
for result in reader.into_records() {
    let record = result?;
    // Process
}
```

---

### 3. Real-time Streaming (Kafka/WebSocket)

**✅ Meilleur choix: Node.js**
```javascript
kafka.on('message', async (msg) => {
  const data = JSON.parse(msg);
  await process(data);
  await emit('result', output);
});
```
**Raison**: Event-driven, async/await, JSON native

**Alternative**: Go pour performance
```go
for {
    msg := <-kafkaChan
    processAndEmit(msg)
}
```

---

### 4. Data Science & Analytics

**✅ Meilleur choix: R ou Python**

**R** (si visualisation+stat)
```r
library(tidyverse)
df %>%
  filter(age > 18) %>%
  group_by(country) %>%
  summarise(avg = mean(salary)) %>%
  ggplot(aes(country, avg)) + geom_col()
```

**Python** (si ML+engineering mix)
```python
import pandas as pd
from sklearn.preprocessing import StandardScaler
df = pd.read_csv('data.csv')
scaler = StandardScaler()
df_scaled = scaler.fit_transform(df)
```

---

### 5. Microservice/API

**✅ Meilleur choix: Go**
```go
func main() {
    http.HandleFunc("/csv", handleCSV)
    http.ListenAndServe(":8080", nil)
}
```
**Raison**: Startup 10ms, memory 50MB, single binary

**Alternative**: Node.js pour rapidité dev
```javascript
app.post('/csv', async (req, res) => {
  const data = await processCSV(req.file);
  res.json(data);
});
```

---

### 6. Big Data ETL (100GB+, Spark)

**✅ Meilleur choix: Scala ou Python**

**Scala** (performance)
```scala
val df = spark.read.csv("huge.csv")
df.filter($"age" > 18)
  .groupBy("country")
  .agg(sum("salary"))
  .show()
```

**Python** (+ usage étendu)
```python
df = spark.read.csv("huge.csv")
df.filter(col("age") > 18) \
  .groupBy("country") \
  .agg(sum("salary")) \
  .show()
```

---

## Intégration multi-langages

### Architecture polyglotte

```
┌──────────────┐
│  Python      │  Data Processing
│  (pandas)    │  Transformations
└──────────────┘
       │
       ↓ (CSV/JSON)
┌──────────────┐
│  Node.js     │  API/Streaming
│  (Express)   │  Real-time
└──────────────┘
       │
       ↓ (HTTP)
┌──────────────┐
│  Go          │  Cache/Queue
│  (Redis)     │  High throughput
└──────────────┘
       │
       ↓ (gRPC)
┌──────────────┐
│  Rust        │  Heavy lifting
│  (CLI tool)  │  Performance-critical
└──────────────┘
```

### Docker compose multi-langages

```yaml
version: '3.8'
services:
  python-processor:
    build:
      context: ./python
      dockerfile: Dockerfile
    volumes:
      - ./data:/data
    environment:
      - OUTPUT_QUEUE=amqp://rabbitmq:5672

  nodejs-api:
    build:
      context: ./nodejs
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    depends_on:
      - python-processor
    environment:
      - QUEUE_URL=amqp://rabbitmq:5672

  go-worker:
    build:
      context: ./go
      dockerfile: Dockerfile
    depends_on:
      - rabbitmq
    environment:
      - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672

  rust-cli:
    build:
      context: ./rust
      dockerfile: Dockerfile
    volumes:
      - ./data:/data
    command: /app/processor /data/input.csv /data/output.parquet

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
```

### Communication inter-processus

```python
# Python: Send to Node.js via queue
import pika
connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()
channel.basic_publish(
    exchange='',
    routing_key='node_queue',
    body=json.dumps({'csv': 'data.csv', 'action': 'transform'})
)

# Node.js: Receive from Python
amqp.connect('amqp://rabbitmq', async (err, conn) => {
  const ch = await conn.createChannel();
  ch.assertQueue('node_queue');
  ch.consume('node_queue', async (msg) => {
    const job = JSON.parse(msg.content.toString());
    const result = await processCSV(job.csv);
    // Send to Go worker
    ch.sendToQueue('go_queue', Buffer.from(JSON.stringify(result)));
  });
});

# Go: Process from Node.js
func main() {
    conn, _ := amqp.Dial("amqp://guest:guest@rabbitmq:5672/")
    ch, _ := conn.Channel()
    
    msgs, _ := ch.Consume("go_queue", "", false, false, false, false)
    
    for msg := range msgs {
        var data map[string]interface{}
        json.Unmarshal(msg.Body, &data)
        result := process(data)
        msg.Ack(false)
    }
}
```

---

## Migration entre langages

### Pattern: Rewrite progressif

```
Phase 1: Dual-write
┌─────────────┐
│ Python      │  Continues production
│ (old)       │
└─────────────┘
       │ Write
       ↓
  ┌─────────────┐
  │ Log file    │  Shadow data
  └─────────────┘
       │
       ↓ Read
┌─────────────┐
│ Go          │  New implementation
│ (new)       │  Parallel processing
└─────────────┘

Phase 2: Verify & Migrate
┌─────────────────────────────┐
│ Compare outputs             │
│ Python vs Go                │
│ 99.99% match = GO LIVE      │
└─────────────────────────────┘

Phase 3: Cutover
Old:  Python ─┐
              ├──→ API
New:  Go ─────┘
```

### CSV to Parquet migration example

**Ancien (Python)**
```python
# Slow, memory-heavy
df = pd.read_csv('huge.csv')  # 8GB RAM
df.to_csv('output.csv')
```

**Nouveau (Rust)**
```rust
// Fast, streaming
let reader = csv::ReaderBuilder::new()
    .from_path("huge.csv")?;
let writer = parquet::Writer::new(&schema);

for record in reader.into_records() {
    writer.write_record(&record?)?;
}
```

---

## 🎓 Exercices pratiques

### Exercice 13.1 : Benchmark
Comparez Python vs Go pour même CSV.

### Exercice 13.2 : Multi-language
Créez pipeline Python → Node.js → Go.

### Exercice 13.3 : Migration
Réécrivez script Python en Go progressivement.

### Exercice 13.4 : Choice
Recommandez langage pour 5 use cases.

### Exercice 13.5 : Docker
Créez docker-compose multi-langage.

---

## 📚 Références

- **Go vs Rust** : https://www.rust-lang.org/what/wg-cli/
- **Python performance** : https://pythonspeed.com/
- **Node.js benchmarks** : https://benchmarks.codalab.org/

---

**Prêt pour le cloud? → [Chapitre 19 : Cloud Deploy](./19_Cloud_Deploy.md)**
