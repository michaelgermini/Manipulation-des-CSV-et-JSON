# Chapitre 18 : Sécurité & Conformité — Chiffrement, PII, Auditing

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Protéger** données sensibles (PII)
- **Chiffrer** en transit et au repos
- **Valider** intégrité (checksum, signatures)
- **Auditer** accès et modifications
- **Conformer** RGPD, CCPA, SOC 2

---

## 📖 Table des matières

1. [Données sensibles (PII)](#données-sensibles-pii)
2. [Chiffrement](#chiffrement)
3. [Intégrité et signatures](#intégrité-et-signatures)
4. [Auditing et logging](#auditing-et-logging)
5. [Conformité RGPD/CCPA](#conformité-rgpdccpa)
6. [Best practices](#best-practices)

---

## Données sensibles (PII)

### Détecter PII

```python
import re
import pandas as pd
from typing import Dict, List

class PIIDetector:
    """Détecter données personnelles sensibles"""
    
    PATTERNS = {
        'email': r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
        'phone': r'(\+\d{1,3})?[\s.-]?\d{3}[\s.-]?\d{3}[\s.-]?\d{4}',
        'ssn': r'\d{3}-\d{2}-\d{4}',  # US Social Security Number
        'credit_card': r'\b\d{4}[\s.-]?\d{4}[\s.-]?\d{4}[\s.-]?\d{4}\b',
        'ipv4': r'\b(?:\d{1,3}\.){3}\d{1,3}\b',
    }
    
    def scan_dataframe(self, df: pd.DataFrame) -> Dict:
        """Scanner DataFrame pour PII"""
        findings = {}
        
        for col in df.select_dtypes(include='object').columns:
            for pattern_name, pattern in self.PATTERNS.items():
                matches = df[col].astype(str).str.contains(pattern, na=False).sum()
                
                if matches > 0:
                    if col not in findings:
                        findings[col] = {}
                    findings[col][pattern_name] = matches
        
        return findings
    
    def scan_file(self, filepath: str) -> Dict:
        """Scanner fichier CSV"""
        df = pd.read_csv(filepath)
        return self.scan_dataframe(df)

# Usage
detector = PIIDetector()
findings = detector.scan_file('customer_data.csv')

for column, patterns in findings.items():
    print(f"Column '{column}' contains:")
    for pattern_type, count in patterns.items():
        print(f"  - {count} {pattern_type} values")
```

### Masquer/Anonymiser PII

```python
import pandas as pd
import hashlib

class PIIAnonymizer:
    """Anonymiser données sensibles"""
    
    @staticmethod
    def mask_email(email: str) -> str:
        """Email → a***@example.com"""
        if pd.isna(email):
            return None
        local, domain = str(email).split('@')
        return f"{local[0]}***@{domain}"
    
    @staticmethod
    def mask_phone(phone: str) -> str:
        """Phone → ***-***-1234"""
        if pd.isna(phone):
            return None
        phone_str = str(phone).replace('-', '').replace(' ', '')
        return f"***-***-{phone_str[-4:]}"
    
    @staticmethod
    def mask_ssn(ssn: str) -> str:
        """SSN → ***-**-1234"""
        if pd.isna(ssn):
            return None
        ssn_str = str(ssn).replace('-', '')
        return f"***-**-{ssn_str[-4:]}"
    
    @staticmethod
    def hash_pii(value: str, salt: str = "secret") -> str:
        """Hash for irreversible anonymization"""
        if pd.isna(value):
            return None
        return hashlib.sha256(f"{value}{salt}".encode()).hexdigest()[:16]

def anonymize_dataframe(df: pd.DataFrame, config: Dict) -> pd.DataFrame:
    """Anonymiser colonnes sensibles"""
    df_anon = df.copy()
    anonymizer = PIIAnonymizer()
    
    operations = {
        'mask_email': anonymizer.mask_email,
        'mask_phone': anonymizer.mask_phone,
        'mask_ssn': anonymizer.mask_ssn,
        'hash': anonymizer.hash_pii,
    }
    
    for column, operation in config.items():
        if column in df_anon.columns:
            df_anon[column] = df_anon[column].apply(operations[operation])
    
    return df_anon

# Usage
config = {
    'email': 'mask_email',
    'phone': 'mask_phone',
    'ssn': 'hash',
    'name': 'hash'
}

df = pd.read_csv('sensitive_data.csv')
df_anon = anonymize_dataframe(df, config)
df_anon.to_csv('anonymized_data.csv', index=False)
```

---

## Chiffrement

### AES-256 chiffrement fichiers

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
import os
import base64

class FileEncryption:
    """Chiffrer/déchiffrer fichiers CSV/JSON"""
    
    @staticmethod
    def derive_key_from_password(password: str, salt: bytes = None) -> tuple:
        """Dériver clé AES de mot de passe"""
        if salt is None:
            salt = os.urandom(16)
        
        kdf = PBKDF2(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
        )
        
        key = base64.urlsafe_b64encode(kdf.derive(password.encode()))
        return key, salt
    
    @staticmethod
    def encrypt_file(input_path: str, output_path: str, password: str):
        """Chiffrer fichier"""
        # Dériver clé
        key, salt = FileEncryption.derive_key_from_password(password)
        cipher = Fernet(key)
        
        # Lire fichier
        with open(input_path, 'rb') as f:
            data = f.read()
        
        # Chiffrer
        encrypted_data = cipher.encrypt(data)
        
        # Écrire avec salt
        with open(output_path, 'wb') as f:
            f.write(salt)  # Salt in first 16 bytes
            f.write(encrypted_data)
        
        print(f"✓ File encrypted: {output_path}")
    
    @staticmethod
    def decrypt_file(input_path: str, output_path: str, password: str):
        """Déchiffrer fichier"""
        # Lire fichier
        with open(input_path, 'rb') as f:
            salt = f.read(16)
            encrypted_data = f.read()
        
        # Dériver clé avec même salt
        key, _ = FileEncryption.derive_key_from_password(password, salt)
        cipher = Fernet(key)
        
        # Déchiffrer
        try:
            data = cipher.decrypt(encrypted_data)
            with open(output_path, 'wb') as f:
                f.write(data)
            print(f"✓ File decrypted: {output_path}")
        except Exception as e:
            print(f"❌ Decryption failed: {e}")

# Usage
FileEncryption.encrypt_file(
    'sensitive_data.csv',
    'sensitive_data.csv.enc',
    password='MySecurePassword123!'
)

FileEncryption.decrypt_file(
    'sensitive_data.csv.enc',
    'sensitive_data_decrypted.csv',
    password='MySecurePassword123!'
)
```

### TLS pour HTTPS

```python
import ssl
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.ssl_ import create_urllib3_context

class SecureHTTPAdapter(HTTPAdapter):
    """HTTP adapter with strong TLS settings"""
    
    def init_poolmanager(self, *args, **kwargs):
        context = create_urllib3_context()
        context.minimum_version = ssl.TLSVersion.TLSv1_2
        context.maximum_version = ssl.TLSVersion.TLSv1_3
        context.options |= ssl.OP_CIPHER_SERVER_PREFERENCE
        
        kwargs['ssl_context'] = context
        return super().init_poolmanager(*args, **kwargs)

# Usage
session = requests.Session()
session.mount('https://', SecureHTTPAdapter())

response = session.get(
    'https://api.example.com/data',
    verify=True,  # Verify SSL certificate
    timeout=30
)
```

---

## Intégrité et signatures

### Checksums (MD5, SHA256)

```python
import hashlib
import os

class FileIntegrity:
    """Vérifier intégrité fichiers"""
    
    @staticmethod
    def compute_hash(filepath: str, algorithm: str = 'sha256') -> str:
        """Calculer hash fichier"""
        hash_obj = hashlib.new(algorithm)
        
        with open(filepath, 'rb') as f:
            # Lire par chunks (important pour gros fichiers)
            for chunk in iter(lambda: f.read(8192), b''):
                hash_obj.update(chunk)
        
        return hash_obj.hexdigest()
    
    @staticmethod
    def verify_hash(filepath: str, expected_hash: str, algorithm: str = 'sha256') -> bool:
        """Vérifier hash"""
        computed = FileIntegrity.compute_hash(filepath, algorithm)
        matches = computed == expected_hash
        
        print(f"File: {filepath}")
        print(f"Expected: {expected_hash}")
        print(f"Computed: {computed}")
        print(f"Status:   {'✓ VALID' if matches else '❌ INVALID'}")
        
        return matches

# Usage
# Calculer hash lors de l'export
hash_sha256 = FileIntegrity.compute_hash('export.csv', 'sha256')
print(f"SHA256: {hash_sha256}")

# Vérifier intégrité à l'import
FileIntegrity.verify_hash(
    'export.csv',
    'a1b2c3d4e5f6...',
    'sha256'
)
```

### Signatures numériques

```python
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.backends import default_backend
import base64

class DigitalSignature:
    """Signer et vérifier fichiers"""
    
    @staticmethod
    def generate_keypair():
        """Générer paire RSA"""
        private_key = rsa.generate_private_key(
            public_exponent=65537,
            key_size=2048,
            backend=default_backend()
        )
        public_key = private_key.public_key()
        return private_key, public_key
    
    @staticmethod
    def sign_file(filepath: str, private_key) -> str:
        """Signer fichier avec clé privée"""
        with open(filepath, 'rb') as f:
            data = f.read()
        
        signature = private_key.sign(
            data,
            padding.PSS(
                mgf=padding.MGF1(hashes.SHA256()),
                salt_length=padding.PSS.MAX_LENGTH
            ),
            hashes.SHA256()
        )
        
        return base64.b64encode(signature).decode()
    
    @staticmethod
    def verify_signature(filepath: str, signature_b64: str, public_key) -> bool:
        """Vérifier signature"""
        with open(filepath, 'rb') as f:
            data = f.read()
        
        signature = base64.b64decode(signature_b64)
        
        try:
            public_key.verify(
                signature,
                data,
                padding.PSS(
                    mgf=padding.MGF1(hashes.SHA256()),
                    salt_length=padding.PSS.MAX_LENGTH
                ),
                hashes.SHA256()
            )
            print("✓ Signature valid")
            return True
        except Exception as e:
            print(f"❌ Signature invalid: {e}")
            return False

# Usage
private_key, public_key = DigitalSignature.generate_keypair()

# Sign
signature = DigitalSignature.sign_file('data.csv', private_key)
print(f"Signature: {signature}")

# Verify
DigitalSignature.verify_signature('data.csv', signature, public_key)
```

---

## Auditing et logging

### Structured audit logs

```python
import logging
import json
from datetime import datetime
import hashlib

class AuditLogger:
    """Logger structuré pour audit trail"""
    
    def __init__(self, log_file: str = 'audit.log'):
        self.logger = logging.getLogger('audit')
        handler = logging.FileHandler(log_file)
        handler.setFormatter(logging.Formatter('%(message)s'))
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)
    
    def log_access(self, user: str, resource: str, action: str, status: str):
        """Logger accès à ressource"""
        event = {
            'timestamp': datetime.utcnow().isoformat(),
            'user': user,
            'resource': resource,
            'action': action,
            'status': status
        }
        self.logger.info(json.dumps(event))
    
    def log_modification(self, user: str, file_path: str, old_hash: str, new_hash: str):
        """Logger modification de fichier"""
        event = {
            'timestamp': datetime.utcnow().isoformat(),
            'user': user,
            'type': 'file_modification',
            'file': file_path,
            'old_hash': old_hash,
            'new_hash': new_hash,
            'changed': old_hash != new_hash
        }
        self.logger.info(json.dumps(event))
    
    def log_export(self, user: str, filename: str, rows: int, pii_fields: List[str]):
        """Logger export de données"""
        event = {
            'timestamp': datetime.utcnow().isoformat(),
            'user': user,
            'type': 'data_export',
            'filename': filename,
            'rows': rows,
            'pii_fields': pii_fields
        }
        self.logger.info(json.dumps(event))

# Usage
auditor = AuditLogger('audit.log')

# Log access
auditor.log_access('alice@example.com', '/data/customers.csv', 'READ', 'SUCCESS')

# Log modification
auditor.log_modification(
    'bob@example.com',
    '/data/customers.csv',
    old_hash='abc123...',
    new_hash='def456...'
)

# Log export
auditor.log_export(
    'charlie@example.com',
    'export_2024.csv',
    rows=10000,
    pii_fields=['email', 'phone']
)
```

---

## Conformité RGPD/CCPA

### Droit à l'oubli (Right to be forgotten)

```python
import pandas as pd
from sqlalchemy import create_engine

class DataDeletion:
    """Implémenter droit à l'oubli"""
    
    @staticmethod
    def delete_user_data(user_email: str, database_url: str) -> bool:
        """Supprimer toutes données utilisateur"""
        engine = create_engine(database_url)
        
        try:
            # Delete from all tables
            with engine.begin() as conn:
                # Users table
                conn.execute(
                    f"DELETE FROM users WHERE email = '{user_email}'"
                )
                
                # Orders table
                conn.execute(
                    f"DELETE FROM orders WHERE user_email = '{user_email}'"
                )
                
                # Activities table
                conn.execute(
                    f"DELETE FROM activities WHERE user_email = '{user_email}'"
                )
            
            print(f"✓ All data for {user_email} deleted")
            return True
        except Exception as e:
            print(f"❌ Deletion failed: {e}")
            return False

# Usage
DataDeletion.delete_user_data(
    'user@example.com',
    'postgresql://user:pass@localhost/mydb'
)
```

### Data portability (Export)

```python
def export_user_data(user_email: str, export_format: str = 'json') -> str:
    """Exporter toutes données utilisateur (RGPD data portability)"""
    
    df = pd.read_csv('users.csv')
    user_data = df[df['email'] == user_email]
    
    if export_format == 'json':
        return user_data.to_json(orient='records', indent=2)
    elif export_format == 'csv':
        return user_data.to_csv(index=False)
    else:
        raise ValueError(f"Unsupported format: {export_format}")

# Usage
json_export = export_user_data('user@example.com', 'json')
print(json_export)
```

### Consent management

```python
class ConsentManager:
    """Gérer consentements RGPD"""
    
    CONSENT_TYPES = {
        'marketing': 'Marketing communications',
        'analytics': 'Analytics tracking',
        'third_party': 'Third-party sharing',
        'profiling': 'Automated profiling'
    }
    
    @staticmethod
    def check_consent(user_id: str, consent_type: str) -> bool:
        """Vérifier consentement utilisateur"""
        df = pd.read_csv('consents.csv')
        user_consent = df[df['user_id'] == user_id]
        
        if user_consent.empty:
            return False
        
        return user_consent[consent_type].values[0]
    
    @staticmethod
    def record_consent(user_id: str, consents: Dict[str, bool]):
        """Enregistrer consentements"""
        record = {
            'user_id': user_id,
            'timestamp': datetime.utcnow().isoformat(),
            **consents
        }
        
        df = pd.DataFrame([record])
        df.to_csv('consents.csv', mode='a', header=False, index=False)

# Usage
if ConsentManager.check_consent('user123', 'marketing'):
    send_marketing_email('user@example.com')
```

---

## Best practices

### Security checklist

```python
def security_audit_checklist():
    """Checklist de sécurité"""
    
    checklist = {
        'Encryption': [
            '✓ TLS 1.2+ for HTTPS',
            '✓ AES-256 at rest',
            '✓ Secrets in environment variables',
            '✗ No hardcoded passwords'
        ],
        'Data Protection': [
            '✓ PII detection implemented',
            '✓ Masking for sensitive fields',
            '✓ Data deletion process',
            '✓ Backup encryption'
        ],
        'Auditing': [
            '✓ Audit logs for all access',
            '✓ File integrity checks',
            '✓ Change tracking',
            '✓ User activity logging'
        ],
        'Compliance': [
            '✓ RGPD data portability',
            '✓ Right to be forgotten implemented',
            '✓ Consent management',
            '✓ Data retention policy'
        ]
    }
    
    for category, items in checklist.items():
        print(f"\n{category}:")
        for item in items:
            status = '✓' if item.startswith('✓') else '✗'
            print(f"  {item}")
```

---

## 🎓 Exercices pratiques

### Exercice 18.1 : Détection PII
Scanner CSV pour données sensibles.

### Exercice 18.2 : Chiffrement
Chiffrer/déchiffrer fichier CSV.

### Exercice 18.3 : Intégrité
Vérifier signature fichier.

### Exercice 18.4 : Audit logs
Logger accès et modifications en JSON.

### Exercice 18.5 : RGPD
Implémenter droit à l'oubli + export.

---

## 📚 Références

- **RGPD** : https://gdpr-info.eu/
- **CCPA** : https://cpra.ca.gov/
- **cryptography** : https://cryptography.io/
- **OWASP** : https://owasp.org/

---

**Prêt pour les cas pratiques? → [Chapitre 24 : Nettoyage CRM](./24_Nettoyage_Export_CRM.md)**
