# Chapitre 23 : Secrets Management — API Keys, Tokens, Credentials

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Gérer** secrets sécurisé (jamais hardcoder!)
- **Utiliser** HashiCorp Vault
- **Configurer** AWS Secrets Manager
- **Implémenter** secret rotation
- **Auditer** accès aux secrets

---

## ❌ Ne JAMAIS faire

```python
# ❌ NEVER - Secrets in code
API_KEY = "sk-proj-abcd1234"
DB_PASSWORD = "MyPassword123"
AWS_SECRET_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# ❌ NEVER - In config files
config = {
    "api_key": "secret_key_here",
    "db_password": "password123"
}

# ❌ NEVER - In environment variables at deploy time
export DB_PASSWORD=password123
```

---

## ✅ Meilleures pratiques

### 1. Environment Variables (Simple)

```bash
# .env (NEVER commit!)
DATABASE_URL=postgresql://user:pass@localhost/db
API_KEY=sk-proj-abc123
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE

# Load in Python
from dotenv import load_dotenv
import os

load_dotenv()
api_key = os.getenv('API_KEY')
db_url = os.getenv('DATABASE_URL')
```

```gitignore
# .gitignore - Prevent accidentally committing secrets
.env
.env.local
*.pem
secrets/
```

### 2. AWS Secrets Manager

```python
import boto3
import json

class SecretsManager:
    def __init__(self, region='us-east-1'):
        self.client = boto3.client('secretsmanager', region_name=region)
    
    def get_secret(self, secret_name):
        """Récupérer secret d'AWS"""
        try:
            response = self.client.get_secret_value(SecretId=secret_name)
            
            if 'SecretString' in response:
                return json.loads(response['SecretString'])
            else:
                return response['SecretBinary']
        
        except Exception as e:
            print(f"Error retrieving secret: {e}")
            return None
    
    def store_secret(self, secret_name, secret_value):
        """Stocker nouveau secret"""
        self.client.create_secret(
            Name=secret_name,
            SecretString=json.dumps(secret_value)
        )
        print(f"✓ Secret '{secret_name}' stored")
    
    def rotate_secret(self, secret_name):
        """Rotation de secret"""
        # Générer nouveau secret
        new_secret = generate_api_key()
        
        # Update
        self.client.update_secret(
            SecretId=secret_name,
            SecretString=json.dumps(new_secret)
        )
        
        print(f"✓ Secret '{secret_name}' rotated")

# Usage
sm = SecretsManager()
db_creds = sm.get_secret('prod/database')
api_key = db_creds['password']
```

### 3. HashiCorp Vault

```python
import hvac

class VaultManager:
    def __init__(self, url='http://localhost:8200', token=None):
        self.client = hvac.Client(url=url, token=token)
    
    def get_secret(self, path):
        """Récupérer secret de Vault"""
        response = self.client.secrets.kv.read_secret_version(path=path)
        return response['data']['data']
    
    def store_secret(self, path, secret):
        """Stocker secret dans Vault"""
        self.client.secrets.kv.create_or_update_secret(
            path=path,
            secret_data=secret
        )
    
    def enable_secret_rotation(self, secret_name, rotation_period='30d'):
        """Activer rotation automatique"""
        # Vault gère la rotation automatiquement
        pass

# Usage
vault = VaultManager(token='your-token')
db_creds = vault.get_secret('secret/prod/database')
```

### 4. Azure Key Vault

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

class AzureSecretsManager:
    def __init__(self, vault_name):
        vault_url = f"https://{vault_name}.vault.azure.net/"
        credential = DefaultAzureCredential()
        self.client = SecretClient(vault_url=vault_url, credential=credential)
    
    def get_secret(self, secret_name):
        """Récupérer secret d'Azure"""
        return self.client.get_secret(secret_name).value
    
    def store_secret(self, secret_name, secret_value):
        """Stocker secret"""
        self.client.set_secret(secret_name, secret_value)

# Usage
azure_sm = AzureSecretsManager(vault_name='my-keyvault')
api_key = azure_sm.get_secret('api-key')
```

---

## Secret Rotation

### Automated rotation script

```python
import schedule
import time
from datetime import datetime, timedelta

class SecretRotationManager:
    def __init__(self, secrets_manager):
        self.sm = secrets_manager
        self.rotation_config = {
            'api_keys': {'interval': 'monthly', 'days': 30},
            'db_password': {'interval': 'quarterly', 'days': 90},
            'jwt_secret': {'interval': 'monthly', 'days': 30}
        }
    
    def rotate_all_secrets(self):
        """Rotate tous les secrets si nécessaire"""
        
        for secret_name, config in self.rotation_config.items():
            if self._needs_rotation(secret_name):
                self._rotate_secret(secret_name)
    
    def _needs_rotation(self, secret_name):
        """Vérifier si rotation nécessaire"""
        
        # Get rotation timestamp from metadata
        last_rotation = self.sm.get_metadata(secret_name).get('last_rotated')
        
        if not last_rotation:
            return True
        
        interval_days = self.rotation_config[secret_name]['days']
        rotation_due = datetime.now() > last_rotation + timedelta(days=interval_days)
        
        return rotation_due
    
    def _rotate_secret(self, secret_name):
        """Rotate secret"""
        
        # Generate new secret
        new_secret = self._generate_secret(secret_name)
        
        # Update in vault
        self.sm.store_secret(secret_name, new_secret)
        
        # Log rotation
        print(f"✓ Rotated {secret_name} at {datetime.now()}")
    
    def _generate_secret(self, secret_name):
        """Generate new secret based on type"""
        
        if 'api' in secret_name:
            return generate_api_key()
        elif 'password' in secret_name:
            return generate_password()
        elif 'jwt' in secret_name:
            return generate_jwt_secret()
        
        return generate_random_token()

# Schedule rotation
def setup_rotation_schedule():
    schedule.every().day.at("02:00").do(rotation_manager.rotate_all_secrets)
    
    while True:
        schedule.run_pending()
        time.sleep(60)
```

---

## Auditing & Monitoring

### Audit logging

```python
import logging
from datetime import datetime

class SecretAuditLog:
    def __init__(self, log_file):
        self.logger = logging.getLogger('secrets_audit')
        handler = logging.FileHandler(log_file)
        formatter = logging.Formatter(
            '%(asctime)s - %(user)s - %(action)s - %(secret_name)s - %(status)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
    
    def log_access(self, user, secret_name, success=True):
        """Log access to secret"""
        extra = {
            'user': user,
            'action': 'accessed',
            'secret_name': secret_name,
            'status': 'success' if success else 'failed'
        }
        self.logger.info(f"Secret accessed", extra=extra)
    
    def log_rotation(self, secret_name):
        """Log secret rotation"""
        extra = {
            'user': 'system',
            'action': 'rotated',
            'secret_name': secret_name,
            'status': 'success'
        }
        self.logger.info(f"Secret rotated", extra=extra)

# Usage
audit_log = SecretAuditLog('/var/log/secrets_audit.log')
audit_log.log_access(user='alice@company.com', secret_name='db_password')
```

---

## Container Secrets

### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: dXNlcm5hbWU=  # base64 encoded
  password: cGFzc3dvcmQ=  # base64 encoded

---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

### Docker secrets

```bash
# Create secret
docker secret create db_password -
# paste password, then Ctrl-D

# Use in service
docker service create \
  --name myapp \
  --secret db_password \
  myapp:latest
```

---

## 🎓 Exercices pratiques

### Exercice 23.1 : .env file
Créez .env + chargez en Python.

### Exercice 23.2 : AWS Secrets
Stockez & retrievez secret AWS.

### Exercice 23.3 : Rotation
Programmez rotation automatique.

### Exercice 23.4 : Audit
Loggez tous les accès secrets.

---

**✓ Fin des chapitres - Annexes suivantes →**
