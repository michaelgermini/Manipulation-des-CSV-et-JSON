# Chapitre 29 : Graph Data & NoSQL — MongoDB, Neo4j, Cassandra

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **MongoDB** pour données JSON
- **Neo4j** pour graphes relationnels
- **Cassandra** pour time-series
- **Queries** et agrégations
- **Performance tuning**

---

## MongoDB

### Connexion et opérations basiques

```python
from pymongo import MongoClient
import json

# Connexion
client = MongoClient('mongodb://localhost:27017/')
db = client['csv_data']
collection = db['customers']

# Insérer
customer = {
    'name': 'Alice',
    'email': 'alice@example.com',
    'age': 30,
    'tags': ['premium', 'active']
}
result = collection.insert_one(customer)
print(f"Inserted ID: {result.inserted_id}")

# Insérer plusieurs
customers = [
    {'name': 'Bob', 'age': 25},
    {'name': 'Charlie', 'age': 35}
]
collection.insert_many(customers)
```

### Querying

```python
# Find one
customer = collection.find_one({'name': 'Alice'})

# Find many
for customer in collection.find({'age': {'$gt': 25}}):
    print(customer)

# Aggregation pipeline
pipeline = [
    {'$match': {'age': {'$gt': 20}}},
    {'$group': {'_id': None, 'avg_age': {'$avg': '$age'}}},
    {'$sort': {'avg_age': -1}}
]
results = list(collection.aggregate(pipeline))
```

---

## Neo4j

### Graph queries

```python
from neo4j import GraphDatabase

# Connexion
driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

def create_nodes(tx):
    tx.run("""
        CREATE (alice:Person {name: 'Alice', age: 30})
        CREATE (bob:Person {name: 'Bob', age: 25})
        CREATE (alice)-[:KNOWS]->(bob)
    """)

with driver.session() as session:
    session.write_transaction(create_nodes)

def find_graph(tx):
    result = tx.run("""
        MATCH (p:Person)-[:KNOWS]->(friend)
        RETURN p.name, friend.name
    """)
    return [(record['p.name'], record['friend.name']) for record in result]

with driver.session() as session:
    relationships = session.read_transaction(find_graph)
    print(relationships)  # [('Alice', 'Bob')]
```

---

## Cassandra

### Time-series data

```python
from cassandra.cluster import Cluster

cluster = Cluster(['127.0.0.1'])
session = cluster.connect('analytics')

# Create table
session.execute("""
    CREATE TABLE IF NOT EXISTS metrics (
        metric_name TEXT,
        timestamp BIGINT,
        value FLOAT,
        PRIMARY KEY ((metric_name), timestamp)
    )
    WITH CLUSTERING ORDER BY (timestamp DESC)
""")

# Insert time-series
session.execute(
    "INSERT INTO metrics (metric_name, timestamp, value) VALUES (%s, %s, %s)",
    ('cpu_usage', 1634567890, 45.5)
)

# Query range
rows = session.execute(
    "SELECT * FROM metrics WHERE metric_name = %s AND timestamp > %s",
    ('cpu_usage', 1634567800)
)
```

---

## 🎓 Exercices pratiques

### Exercice 29.1 : MongoDB
Insérez CSV en MongoDB et queryez.

### Exercice 29.2 : Neo4j
Créez graphe relations et requêtez.

### Exercice 29.3 : Cassandra
Insérez time-series et requêtez ranges.

---

**Voir aussi: [Chapitre 30 : Data Governance](./30_Data_Governance.md)**
