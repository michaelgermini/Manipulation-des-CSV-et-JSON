# Chapitre 27 : Intégration API — REST, GraphQL, Webhooks

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Consommer** APIs REST avec rate limiting
- **Paginater** grandes réponses
- **Utiliser** GraphQL pour requêtes efficaces
- **Implémenter** webhooks pour temps réel
- **Cacher** et valider réponses

---

## 📖 Table des matières

1. [REST APIs](#rest-apis)
2. [GraphQL](#graphql)
3. [Webhooks](#webhooks)
4. [Caching & Retry](#caching--retry)
5. [Cas d'usage réels](#cas-dusage-réels)

---

## REST APIs

### Requêtes basiques

```python
import requests
import json
from typing import Dict, List, Optional

class APIClient:
    """Client pour APIs REST"""
    
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.headers = {
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json'
        }
        self.session = requests.Session()
        self.session.headers.update(self.headers)
    
    def get(self, endpoint: str, params: Dict = None) -> Dict:
        """GET request"""
        url = f"{self.base_url}/{endpoint}"
        response = self.session.get(url, params=params)
        response.raise_for_status()
        return response.json()
    
    def post(self, endpoint: str, data: Dict) -> Dict:
        """POST request"""
        url = f"{self.base_url}/{endpoint}"
        response = self.session.post(url, json=data)
        response.raise_for_status()
        return response.json()

# Usage
client = APIClient(
    base_url='https://api.example.com/v1',
    api_key='your-api-key'
)

users = client.get('users', params={'limit': 10})
```

### Pagination

```python
def get_all_items(client: APIClient, endpoint: str, limit: int = 100) -> List[Dict]:
    """Paginer automatiquement via API"""
    items = []
    page = 1
    
    while True:
        response = client.get(endpoint, params={
            'page': page,
            'limit': limit
        })
        
        if not response.get('data'):
            break
        
        items.extend(response['data'])
        
        # Check if more pages
        if response.get('pagination', {}).get('has_next'):
            page += 1
        else:
            break
    
    return items

def get_all_items_cursor(client: APIClient, endpoint: str) -> List[Dict]:
    """Paginater avec curseur (plus efficace)"""
    items = []
    cursor = None
    
    while True:
        params = {'limit': 100}
        if cursor:
            params['cursor'] = cursor
        
        response = client.get(endpoint, params=params)
        
        if not response.get('data'):
            break
        
        items.extend(response['data'])
        
        cursor = response.get('pagination', {}).get('next_cursor')
        if not cursor:
            break
    
    return items

# Usage
all_users = get_all_items_cursor(client, 'users')
print(f"Retrieved {len(all_users)} users")
```

### Streaming large datasets

```python
import pandas as pd
from io import StringIO

def stream_csv_from_api(client: APIClient, endpoint: str, chunk_size: int = 1000):
    """Stream données CSV depuis API"""
    offset = 0
    
    while True:
        response = client.get(endpoint, params={
            'offset': offset,
            'limit': chunk_size,
            'format': 'csv'
        })
        
        if response is None or len(response) == 0:
            break
        
        # Parse CSV chunk
        df = pd.read_csv(StringIO(response))
        yield df
        
        offset += chunk_size

# Usage
for chunk in stream_csv_from_api(client, 'export/users', chunk_size=5000):
    print(f"Processing {len(chunk)} rows")
    # Process chunk
```

---

## GraphQL

### Requêtes GraphQL

```python
import requests

class GraphQLClient:
    """Client pour APIs GraphQL"""
    
    def __init__(self, url: str, api_key: str = None):
        self.url = url
        self.headers = {
            'Content-Type': 'application/json'
        }
        if api_key:
            self.headers['Authorization'] = f'Bearer {api_key}'
    
    def execute(self, query: str, variables: Dict = None) -> Dict:
        """Execute GraphQL query"""
        payload = {
            'query': query,
            'variables': variables or {}
        }
        
        response = requests.post(
            self.url,
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()
        
        result = response.json()
        
        # Check for GraphQL errors
        if 'errors' in result:
            raise Exception(f"GraphQL error: {result['errors']}")
        
        return result['data']

# Usage
client = GraphQLClient('https://api.example.com/graphql')

query = """
query GetUsers($limit: Int!) {
  users(limit: $limit) {
    id
    name
    email
    orders {
      id
      total
      status
    }
  }
}
"""

data = client.execute(query, variables={'limit': 10})
print(data)
```

### Batch requests

```python
def batch_graphql_requests(client: GraphQLClient, queries: List[str]) -> List[Dict]:
    """Execute multiple GraphQL queries efficacement"""
    
    # Combine into batch query
    batch_query = f"""
    query {{
      {' '.join([f'q{i}: {q}' for i, q in enumerate(queries)])}
    }}
    """
    
    response = client.execute(batch_query)
    results = [response[f'q{i}'] for i in range(len(queries))]
    
    return results
```

---

## Webhooks

### Receiver Flask

```python
from flask import Flask, request, jsonify
import hmac
import hashlib
import json

app = Flask(__name__)

WEBHOOK_SECRET = 'your-webhook-secret'

def verify_webhook_signature(payload: bytes, signature: str) -> bool:
    """Vérifier signature webhook"""
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(expected, signature)

@app.route('/webhooks/orders', methods=['POST'])
def handle_order_webhook():
    """Recevoir webhook orders"""
    
    # Vérifier signature
    signature = request.headers.get('X-Webhook-Signature')
    if not verify_webhook_signature(request.data, signature):
        return jsonify({'error': 'Invalid signature'}), 401
    
    # Parser payload
    event = request.get_json()
    
    # Process based on type
    if event['type'] == 'order.created':
        handle_order_created(event['data'])
    elif event['type'] == 'order.shipped':
        handle_order_shipped(event['data'])
    
    # Return success
    return jsonify({'status': 'received'}), 200

def handle_order_created(data):
    """Traiter commande créée"""
    print(f"New order: {data['id']} from {data['customer_email']}")
    # Store in DB, notify, etc.

def handle_order_shipped(data):
    """Traiter commande expédiée"""
    print(f"Order {data['id']} shipped!")

if __name__ == '__main__':
    app.run(port=5000)
```

### Webhook sender

```python
import hmac
import hashlib
import json

def send_webhook(url: str, data: Dict, secret: str):
    """Envoyer webhook avec signature"""
    
    payload = json.dumps(data).encode()
    
    signature = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    headers = {
        'Content-Type': 'application/json',
        'X-Webhook-Signature': signature
    }
    
    response = requests.post(url, data=payload, headers=headers)
    return response.status_code == 200
```

---

## Caching & Retry

### Retry avec backoff exponentiel

```python
import time
from functools import wraps

def retry_with_backoff(max_retries: int = 3, base_delay: float = 1.0):
    """Decorator pour retry avec backoff exponentiel"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except requests.exceptions.RequestException as e:
                    if attempt == max_retries - 1:
                        raise
                    
                    delay = base_delay * (2 ** attempt)
                    print(f"Attempt {attempt + 1} failed, retrying in {delay}s...")
                    time.sleep(delay)
        
        return wrapper
    return decorator

@retry_with_backoff(max_retries=3, base_delay=1.0)
def fetch_data_reliable(url: str) -> Dict:
    """Fetch data with automatic retry"""
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
```

### Response caching

```python
import pickle
import os
from datetime import datetime, timedelta

class CachedAPIClient:
    """API client with caching"""
    
    def __init__(self, cache_dir: str = '.cache', ttl_hours: int = 24):
        self.cache_dir = cache_dir
        self.ttl = timedelta(hours=ttl_hours)
        os.makedirs(cache_dir, exist_ok=True)
    
    def _get_cache_path(self, key: str) -> str:
        """Générer path cache"""
        return os.path.join(self.cache_dir, f"{key}.pickle")
    
    def _is_cache_valid(self, cache_path: str) -> bool:
        """Vérifier si cache est valide"""
        if not os.path.exists(cache_path):
            return False
        
        modified = datetime.fromtimestamp(os.path.getmtime(cache_path))
        return datetime.now() - modified < self.ttl
    
    def get_with_cache(self, key: str, fetch_func, *args, **kwargs) -> Dict:
        """Get avec caching automatique"""
        cache_path = self._get_cache_path(key)
        
        # Check cache
        if self._is_cache_valid(cache_path):
            with open(cache_path, 'rb') as f:
                return pickle.load(f)
        
        # Fetch new data
        data = fetch_func(*args, **kwargs)
        
        # Save cache
        with open(cache_path, 'wb') as f:
            pickle.dump(data, f)
        
        return data

# Usage
client = CachedAPIClient(ttl_hours=24)

def fetch_users():
    return requests.get('https://api.example.com/users').json()

users = client.get_with_cache('users', fetch_users)
```

---

## Cas d'usage réels

### Multi-source data pipeline

```python
def fetch_and_combine_data():
    """Combiner données de multiples sources API"""
    
    # Fetch de source 1
    rest_client = APIClient(
        base_url='https://api.rest.example.com/v1',
        api_key='rest-key'
    )
    users_rest = get_all_items_cursor(rest_client, 'users')
    
    # Fetch de source 2 (GraphQL)
    graphql_client = GraphQLClient(
        url='https://api.graphql.example.com',
        api_key='graphql-key'
    )
    orders_query = """
    query {
      orders {
        id
        userId
        total
        status
      }
    }
    """
    orders_data = graphql_client.execute(orders_query)
    
    # Combine
    df_users = pd.DataFrame(users_rest)
    df_orders = pd.DataFrame(orders_data['orders'])
    
    # Join
    combined = df_users.merge(
        df_orders,
        left_on='id',
        right_on='userId',
        how='left'
    )
    
    return combined
```

---

## 🎓 Exercices pratiques

### Exercice 27.1 : REST API
Consommer API REST avec pagination.

### Exercice 27.2 : GraphQL
Requête GraphQL pour récupérer données liées.

### Exercice 27.3 : Webhook
Implementer receiver webhook en Flask.

### Exercice 27.4 : Retry
Ajouter logique retry & backoff.

### Exercice 27.5 : Caching
Implémenter caching pour réduire appels API.

---

## 📚 Références

- **Requests** : https://requests.readthedocs.io/
- **GraphQL** : https://graphql.org/
- **Flask** : https://flask.palletsprojects.com/
- **Webhook best practices** : https://zapier.com/engineering/webhook/

---

**Prêt pour les annexes? → [Cheatsheets & Resources](../Annexes/)**
