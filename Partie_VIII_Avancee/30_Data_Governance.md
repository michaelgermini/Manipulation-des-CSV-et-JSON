# Chapitre 30 : Data Governance & Lineage — Apache Atlas, Metadata Management

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Data governance** policies
- **Data lineage** tracking
- **Metadata management**
- **Data quality** monitoring
- **Compliance** frameworks

---

## 📖 Table des matières

1. [Data Governance Framework](#data-governance-framework)
2. [Metadata Management](#metadata-management)
3. [Data Lineage](#data-lineage)
4. [Data Quality](#data-quality)
5. [Compliance & Auditing](#compliance--auditing)

---

## Data Governance Framework

### Data catalog

```python
# Metadata store
data_assets = {
    'customers_csv': {
        'type': 'file',
        'location': 's3://data/customers.csv',
        'owner': 'alice@company.com',
        'created': '2025-01-01',
        'tags': ['pii', 'critical'],
        'quality_score': 0.95,
        'sla': {
            'availability': 0.999,
            'update_frequency': 'daily'
        }
    },
    'orders_json': {
        'type': 'api',
        'endpoint': 'https://api.example.com/orders',
        'owner': 'bob@company.com',
        'retention': '7 years',
        'encryption': 'AES-256'
    }
}

# Data classification
def classify_data(asset):
    if 'pii' in asset.get('tags', []):
        return 'CONFIDENTIAL'
    elif 'critical' in asset.get('tags', []):
        return 'INTERNAL'
    else:
        return 'PUBLIC'
```

### Access policies

```python
access_policies = {
    'CONFIDENTIAL': {
        'users': ['alice@company.com', 'bob@company.com'],
        'roles': ['data_scientist', 'data_engineer'],
        'encryption': 'required',
        'audit_logs': 'required'
    },
    'INTERNAL': {
        'users': ['*@company.com'],
        'roles': ['employee'],
        'encryption': 'optional',
        'audit_logs': 'required'
    },
    'PUBLIC': {
        'users': ['*'],
        'encryption': 'optional',
        'audit_logs': 'optional'
    }
}

def check_access(user, asset, classification):
    policy = access_policies[classification]
    
    # Check user access
    if user not in policy.get('users', []) and '*' not in policy['users']:
        return False
    
    return True
```

---

## Metadata Management

### Apache Atlas integration

```python
from pyatlasclient import AtlasClient

# Connect to Atlas
atlas_client = AtlasClient('http://localhost:21000', ('admin', 'admin'))

# Create entity
entity_def = {
    'typeName': 'DataSet',
    'attributes': {
        'name': 'customers_csv',
        'description': 'Customer master data',
        'owner': 'alice@company.com',
        'path': 's3://data/customers.csv',
        'format': 'CSV'
    }
}

response = atlas_client.entity_post(entity_def)
entity_id = response['entity']['guid']

# Search entities
results = atlas_client.search('customers', limit=10)
for entity in results['entities']:
    print(f"{entity['attributes']['name']} (owner: {entity['attributes']['owner']})")
```

### Schema registry

```python
from confluent_kafka.schema_registry import SchemaRegistryClient

registry = SchemaRegistryClient({'url': 'http://localhost:8081'})

# Register schema
schema = """{
    "type": "record",
    "name": "Customer",
    "fields": [
        {"name": "id", "type": "int"},
        {"name": "name", "type": "string"},
        {"name": "email", "type": "string"},
        {"name": "created", "type": "long", "logicalType": "timestamp-millis"}
    ]
}"""

schema_id = registry.register_schema(
    subject='customers-value',
    schema_str=schema
)

# Retrieve schema
retrieved = registry.get_schema(schema_id)
```

---

## Data Lineage

### Track transformations

```python
import json
from datetime import datetime

class LineageTracker:
    def __init__(self):
        self.lineage = {
            'datasets': {},
            'transformations': [],
            'jobs': {}
        }
    
    def register_dataset(self, dataset_id, metadata):
        self.lineage['datasets'][dataset_id] = {
            'metadata': metadata,
            'created': datetime.now().isoformat(),
            'sources': [],
            'sinks': []
        }
    
    def register_transformation(self, job_id, sources, sinks, transformation_code):
        self.lineage['transformations'].append({
            'job_id': job_id,
            'sources': sources,
            'sinks': sinks,
            'code': transformation_code,
            'timestamp': datetime.now().isoformat()
        })
        
        # Update lineage
        for source in sources:
            if source in self.lineage['datasets']:
                self.lineage['datasets'][source]['sinks'].append(job_id)
        
        for sink in sinks:
            if sink in self.lineage['datasets']:
                self.lineage['datasets'][sink]['sources'].append(job_id)
    
    def trace_lineage(self, dataset_id, direction='upstream'):
        """Trace upstream or downstream lineage"""
        lineage_path = []
        visited = set()
        
        def traverse(current, is_upstream):
            if current in visited:
                return
            visited.add(current)
            lineage_path.append(current)
            
            if current in self.lineage['datasets']:
                related = (
                    self.lineage['datasets'][current]['sources'] if is_upstream
                    else self.lineage['datasets'][current]['sinks']
                )
                for rel in related:
                    traverse(rel, is_upstream)
        
        is_upstream = direction == 'upstream'
        traverse(dataset_id, is_upstream)
        
        return lineage_path
    
    def export_lineage(self, filepath):
        with open(filepath, 'w') as f:
            json.dump(self.lineage, f, indent=2)

# Usage
tracker = LineageTracker()
tracker.register_dataset('raw_customers', {'source': 'salesforce', 'format': 'csv'})
tracker.register_dataset('cleaned_customers', {'format': 'parquet'})
tracker.register_transformation(
    'clean_job_1',
    sources=['raw_customers'],
    sinks=['cleaned_customers'],
    transformation_code='SELECT id, name, email FROM raw WHERE email IS NOT NULL'
)

# Trace lineage
upstream = tracker.trace_lineage('cleaned_customers', direction='upstream')
print(f"Upstream sources: {upstream}")
```

---

## Data Quality

### Quality rules

```python
class DataQualityFramework:
    def __init__(self):
        self.rules = {}
        self.results = []
    
    def add_rule(self, rule_name, rule_func, description):
        self.rules[rule_name] = {
            'func': rule_func,
            'description': description
        }
    
    def evaluate(self, dataframe, rules_to_check):
        results = {
            'timestamp': datetime.now().isoformat(),
            'total_rows': len(dataframe),
            'passed_rules': [],
            'failed_rules': [],
            'violations': {}
        }
        
        for rule_name in rules_to_check:
            if rule_name not in self.rules:
                continue
            
            rule = self.rules[rule_name]
            try:
                violations = rule['func'](dataframe)
                
                if len(violations) == 0:
                    results['passed_rules'].append(rule_name)
                else:
                    results['failed_rules'].append(rule_name)
                    results['violations'][rule_name] = {
                        'count': len(violations),
                        'percentage': (len(violations) / len(dataframe)) * 100
                    }
            except Exception as e:
                results['violations'][rule_name] = {'error': str(e)}
        
        self.results.append(results)
        return results

# Setup rules
dq = DataQualityFramework()

# Rule 1: No null emails
dq.add_rule(
    'no_null_emails',
    lambda df: df[df['email'].isna()],
    'Email field should not be null'
)

# Rule 2: Valid email format
import re
email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
dq.add_rule(
    'valid_email_format',
    lambda df: df[~df['email'].str.match(email_pattern)],
    'Email should match standard format'
)

# Rule 3: Positive IDs
dq.add_rule(
    'positive_ids',
    lambda df: df[df['id'] <= 0],
    'ID should be positive'
)

# Evaluate
import pandas as pd
df = pd.read_csv('customers.csv')
quality_results = dq.evaluate(df, ['no_null_emails', 'valid_email_format', 'positive_ids'])
print(json.dumps(quality_results, indent=2))
```

---

## Compliance & Auditing

### Audit logging

```python
import logging
from datetime import datetime

class AuditLogger:
    def __init__(self, log_file):
        self.logger = logging.getLogger('audit')
        handler = logging.FileHandler(log_file)
        formatter = logging.Formatter(
            '%(asctime)s | %(user)s | %(action)s | %(resource)s | %(result)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)
    
    def log_access(self, user, resource, access_type):
        extra = {'user': user, 'action': f'ACCESS_{access_type}', 'resource': resource}
        self.logger.info('', extra=extra)
    
    def log_modification(self, user, resource, change):
        extra = {'user': user, 'action': 'MODIFY', 'resource': resource}
        self.logger.info(change, extra=extra)
    
    def log_deletion(self, user, resource, reason):
        extra = {'user': user, 'action': 'DELETE', 'resource': resource}
        self.logger.info(reason, extra=extra)

# Usage
auditor = AuditLogger('/var/log/data_governance.log')
auditor.log_access('alice@company.com', 'customers.csv', 'READ')
auditor.log_modification('bob@company.com', 'orders.json', 'Added 100 records')
```

---

## 🎓 Exercices pratiques

### Exercice 30.1 : Data catalog
Créez catalog de data assets.

### Exercice 30.2 : Lineage tracking
Tracez lineage CSV→Transform→JSON.

### Exercice 30.3 : Quality rules
Implémentez 5 data quality rules.

### Exercice 30.4 : Audit logging
Loggez toutes les modifications données.

### Exercice 30.5 : Compliance
Documentez RGPD compliance.

---

## 📚 Références

- **Apache Atlas** : https://atlas.apache.org/
- **Data Governance** : https://www.gartner.com/en/glossary/data-governance
- **Data Lineage** : https://www.talend.com/resources/ebook/data-lineage/

---

**✅ GUIDE COMPLET TERMINÉ! Toutes les 30 chapitres sont maintenant complètement développées.**
