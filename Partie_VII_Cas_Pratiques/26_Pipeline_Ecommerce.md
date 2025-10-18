# Chapitre 26 : Pipeline E-commerce — Temps réel & Analytics

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Ingérer** données de ventes temps réel
- **Transformer** pour analytics
- **Enrichir** avec données externes
- **Détecter** anomalies
- **Exporter** pour BI/dashboards

---

## 📖 Table des matières

1. [Architecture pipeline](#architecture-pipeline)
2. [Ingestion données](#ingestion-données)
3. [Transformations](#transformations)
4. [Détection anomalies](#détection-anomalies)
5. [Enrichissement](#enrichissement)
6. [Export analytique](#export-analytique)

---

## Architecture pipeline

```
┌─────────────┐       ┌──────────────┐       ┌──────────────┐
│  Sources:   │       │   Ingestion  │       │  Processing  │
│ - Shopify   ├──────→│   Streaming  ├──────→│  Transform   │
│ - WooComm   │       │   (Kafka)    │       │  Aggregate   │
│ - Custom    │       │   CSV/JSON   │       │  Deduplicate │
└─────────────┘       └──────────────┘       └──────────────┘
                                                     │
                                                     ↓
                                              ┌──────────────┐
                                              │  Validation  │
                                              │  Enrichment  │
                                              └──────────────┘
                                                     │
                                    ┌────────────────┼────────────────┐
                                    ↓                ↓                ↓
                            ┌─────────────┐  ┌────────────┐  ┌────────────┐
                            │ Data Lake   │  │ BI Tool    │  │ Dashboards │
                            │ (Parquet)   │  │ (Tableau)  │  │ (Grafana)  │
                            └─────────────┘  └────────────┘  └────────────┘
```

---

## Ingestion données

### Streaming Kafka

```python
from kafka import KafkaConsumer, KafkaProducer
import json
from datetime import datetime

class OrderConsumer:
    """Consommer orders de Kafka"""
    
    def __init__(self, bootstrap_servers=['localhost:9092']):
        self.consumer = KafkaConsumer(
            'orders',
            bootstrap_servers=bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
            group_id='ecommerce-group'
        )
    
    def consume_orders(self):
        """Stream d'orders"""
        for message in self.consumer:
            order = message.value
            order['ingested_at'] = datetime.utcnow().isoformat()
            yield order

# Usage
consumer = OrderConsumer()
for order in consumer.consume_orders():
    print(f"Order {order['id']}: {order['total']} USD")
```

### CSV batch import

```python
import pandas as pd
from datetime import datetime
import os

class CSVImporter:
    """Importer fichiers CSV exportés de Shopify, WooCommerce, etc."""
    
    @staticmethod
    def import_shopify_export(filepath: str) -> pd.DataFrame:
        """Importer export Shopify"""
        df = pd.read_csv(filepath)
        
        # Map colonnes Shopify
        mapping = {
            'Order #': 'order_id',
            'Email': 'customer_email',
            'Financial Status': 'payment_status',
            'Total': 'order_total',
            'Created at': 'order_date'
        }
        df = df.rename(columns=mapping)
        
        # Parse dates
        df['order_date'] = pd.to_datetime(df['order_date'])
        
        # Add metadata
        df['source'] = 'shopify'
        df['imported_at'] = datetime.utcnow()
        
        return df
    
    @staticmethod
    def import_woocommerce_export(filepath: str) -> pd.DataFrame:
        """Importer export WooCommerce"""
        df = pd.read_csv(filepath)
        
        mapping = {
            'Order ID': 'order_id',
            'Order Status': 'order_status',
            'Order Total': 'order_total',
            'Order Date': 'order_date',
            'Customer Email': 'customer_email'
        }
        df = df.rename(columns=mapping)
        
        df['order_date'] = pd.to_datetime(df['order_date'])
        df['source'] = 'woocommerce'
        df['imported_at'] = datetime.utcnow()
        
        return df

# Usage
shopify_df = CSVImporter.import_shopify_export('shopify_orders_2024.csv')
woocomm_df = CSVImporter.import_woocommerce_export('woocommerce_orders_2024.csv')

# Combiner
all_orders = pd.concat([shopify_df, woocomm_df], ignore_index=True)
```

---

## Transformations

### Nettoyage & normalisation

```python
def clean_orders(df: pd.DataFrame) -> pd.DataFrame:
    """Nettoyer données orders"""
    df = df.copy()
    
    # Standardiser colonnes
    df.columns = df.columns.str.lower().str.replace(' ', '_')
    
    # Nettoyage prix
    df['order_total'] = df['order_total'].str.replace('$', '').str.replace(',', '')
    df['order_total'] = pd.to_numeric(df['order_total'], errors='coerce')
    
    # Nettoyage emails
    df['customer_email'] = df['customer_email'].str.lower().str.strip()
    
    # Standardiser statuts
    status_mapping = {
        'completed': 'completed',
        'processing': 'processing',
        'pending': 'pending',
        'canceled': 'cancelled',
        'refunded': 'refunded'
    }
    df['order_status'] = df['order_status'].str.lower().map(
        lambda x: status_mapping.get(x, x)
    )
    
    # Supprimer doublons
    df = df.drop_duplicates(subset=['order_id', 'source'])
    
    # Supprimer lignes complètement vides
    df = df.dropna(how='all')
    
    return df
```

### Enrichissement temps réel

```python
def calculate_derived_metrics(df: pd.DataFrame) -> pd.DataFrame:
    """Calculer métriques dérivées"""
    df = df.copy()
    
    # Order metrics
    df['is_completed'] = df['order_status'] == 'completed'
    df['is_refunded'] = df['order_status'] == 'refunded'
    df['days_since_order'] = (datetime.utcnow() - df['order_date']).dt.days
    
    # Customer metrics
    df['email_domain'] = df['customer_email'].str.split('@').str[1]
    df['is_corporate'] = df['email_domain'].isin(['company.com', 'acme.com'])
    
    # Value metrics
    df['order_value_category'] = pd.cut(
        df['order_total'],
        bins=[0, 50, 200, 1000, float('inf')],
        labels=['small', 'medium', 'large', 'xlarge']
    )
    
    # Timestamp extraction
    df['order_date'] = pd.to_datetime(df['order_date'])
    df['order_year'] = df['order_date'].dt.year
    df['order_month'] = df['order_date'].dt.month
    df['order_day_of_week'] = df['order_date'].dt.day_name()
    df['order_hour'] = df['order_date'].dt.hour
    
    return df
```

---

## Détection anomalies

### Statistical outliers

```python
import numpy as np
from scipy import stats

def detect_price_anomalies(df: pd.DataFrame, zscore_threshold: float = 3.0) -> pd.DataFrame:
    """Détecter anomalies de prix"""
    df = df.copy()
    
    # Calculer z-score par catégorie
    if 'product_category' in df.columns:
        df['price_zscore'] = df.groupby('product_category')['order_total'].transform(
            lambda x: np.abs(stats.zscore(x, nan_policy='omit'))
        )
    else:
        df['price_zscore'] = np.abs(stats.zscore(df['order_total'], nan_policy='omit'))
    
    # Marquer anomalies
    df['is_price_anomaly'] = df['price_zscore'] > zscore_threshold
    
    return df

def detect_customer_anomalies(df: pd.DataFrame) -> pd.DataFrame:
    """Détecter comportements clients suspects"""
    df = df.copy()
    
    # Orders multiples en temps court
    customer_orders = df.groupby('customer_email').size()
    df['is_bulk_buyer'] = df['customer_email'].map(
        lambda x: customer_orders[x] > 10
    )
    
    # Valeur totale haute
    customer_total = df.groupby('customer_email')['order_total'].sum()
    df['customer_lifetime_value'] = df['customer_email'].map(customer_total)
    
    # High-value first purchase
    df['is_high_value_first_purchase'] = (
        (df.groupby('customer_email').cumcount() == 0) &
        (df['order_total'] > df['order_total'].quantile(0.95))
    )
    
    return df

# Usage
df = detect_price_anomalies(df)
df = detect_customer_anomalies(df)

anomalies = df[df['is_price_anomaly'] | df['is_high_value_first_purchase']]
print(f"Detected {len(anomalies)} anomalies")
```

---

## Enrichissement

### Join avec données externes

```python
def enrich_with_geo(df: pd.DataFrame) -> pd.DataFrame:
    """Enrichir avec données géographiques"""
    
    # Charger mapping pays/région
    geo_data = pd.read_csv('country_regions.csv')
    
    df = df.merge(
        geo_data,
        left_on='country',
        right_on='country_code',
        how='left'
    )
    
    return df

def enrich_with_product_data(df: pd.DataFrame) -> pd.DataFrame:
    """Enrichir avec données produits"""
    
    products = pd.read_csv('product_catalog.csv')
    
    df = df.merge(
        products[['product_id', 'product_category', 'supplier_id', 'cost']],
        on='product_id',
        how='left'
    )
    
    # Calculer profit
    df['margin'] = df['order_total'] - df['cost']
    df['margin_pct'] = (df['margin'] / df['order_total'] * 100).round(2)
    
    return df

def enrich_with_customer_history(df: pd.DataFrame, historical_df: pd.DataFrame) -> pd.DataFrame:
    """Enrichir avec historique client"""
    
    customer_stats = historical_df.groupby('customer_email').agg({
        'order_id': 'count',
        'order_total': ['sum', 'mean'],
        'order_date': 'max'
    }).reset_index()
    
    customer_stats.columns = ['customer_email', 'order_count', 'lifetime_value', 'avg_order_value', 'last_order_date']
    
    df = df.merge(customer_stats, on='customer_email', how='left')
    df['is_repeat_customer'] = df['order_count'] > 1
    
    return df
```

---

## Export analytique

### Analytics data mart

```python
def prepare_analytics_export(df: pd.DataFrame) -> Dict[str, pd.DataFrame]:
    """Préparer données pour BI/analytics"""
    
    datasets = {}
    
    # 1. Fact table - Orders
    datasets['fact_orders'] = df[[
        'order_id', 'customer_id', 'product_id',
        'order_total', 'order_status', 'order_date',
        'margin', 'margin_pct'
    ]].copy()
    
    # 2. Customers dimension
    datasets['dim_customers'] = df[[
        'customer_id', 'customer_email', 'customer_name',
        'email_domain', 'is_corporate', 'order_count',
        'lifetime_value', 'is_repeat_customer'
    ]].drop_duplicates()
    
    # 3. Products dimension
    datasets['dim_products'] = df[[
        'product_id', 'product_name', 'product_category',
        'supplier_id', 'cost'
    ]].drop_duplicates()
    
    # 4. Daily summary
    datasets['daily_summary'] = df.groupby('order_date').agg({
        'order_id': 'count',
        'order_total': 'sum',
        'is_completed': 'sum',
        'is_refunded': 'sum',
        'margin': 'sum'
    }).reset_index()
    datasets['daily_summary'].columns = ['date', 'orders', 'revenue', 'completed', 'refunded', 'total_margin']
    
    # 5. Anomalies log
    anomalies = df[df['is_price_anomaly'] | df['is_high_value_first_purchase']]
    datasets['anomalies'] = anomalies[[
        'order_id', 'customer_email', 'order_total',
        'is_price_anomaly', 'is_high_value_first_purchase'
    ]]
    
    return datasets

# Export
datasets = prepare_analytics_export(df)

for name, data in datasets.items():
    # CSV
    data.to_csv(f'export/{name}.csv', index=False)
    # Parquet (compressed, efficient)
    data.to_parquet(f'export/{name}.parquet', compression='snappy')

print("✓ Exported to /export/")
```

---

## 🎓 Exercices pratiques

### Exercice 26.1 : Ingestion
Consommer orders de Kafka ou CSV import.

### Exercice 26.2 : Transformation
Nettoyer et normaliser données e-commerce.

### Exercice 26.3 : Anomalies
Détecter prix/clients anormaux.

### Exercice 26.4 : Enrichissement
Ajouter données géo et produits.

### Exercice 26.5 : Export
Créer données mart pour Tableau/PowerBI.

---

## 📚 Références

- **Apache Kafka** : https://kafka.apache.org/
- **pandas** : https://pandas.pydata.org/
- **Polars** : https://www.pola-rs.com/

---

**Prêt pour intégrations API? → [Chapitre 27 : Intégration API](./27_Integration_API.md)**
