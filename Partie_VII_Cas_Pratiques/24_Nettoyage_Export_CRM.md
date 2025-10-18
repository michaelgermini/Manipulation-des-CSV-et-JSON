# Chapitre 24 : Nettoyage d'un export CRM — Du brut au tableau analytique

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Diagnostiquer** une source de données (audit)
- **Nettoyer** encodages, dates, formats
- **Valider** et dédupliquer
- **Enrichir** données (calculs, flags)
- **Exporter** vers data warehouse

---

## 📖 Table des matières

1. [Scénario réel](#scénario-réel)
2. [Étape 1 : Diagnostic](#étape-1--diagnostic)
3. [Étape 2 : Nettoyage](#étape-2--nettoyage)
4. [Étape 3 : Validation](#étape-3--validation)
5. [Étape 4 : Enrichissement](#étape-4--enrichissement)
6. [Étape 5 : Export analytique](#étape-5--export-analytique)

---

## Scénario réel

### Situation

Vous recevez un **export quotidien de Salesforce** :
- Format : CSV
- Taille : 500MB (50K contacts)
- Problèmes connus : Encodages mixtes, dates mal formatées, doublons

**Objectif** : Transformer en données nettoyées pour analytics (Tableau/Power BI)

### Fichier brut

```
id,Name,Email,Phone,Industry,Annual_Revenue,Created_Date,Last_Activity,LeadScore,Status
"001","Alice Smith"," alice@example.com ",,"Technology","$2,500,000.00","01/15/2025 10:30:45 AM",,,"Active"
"002","Bob Jones",,"+1-415-SALES-1","Technology","2.5M","2025-01-14T08:00:00Z","2025-01-15 14:30",75,"Active"
"003","Café Dupont","cafe.dupont@example.fr",,,"NULL","01/15/2025",,"SCORE:50","Prospect"
...  (50000 lignes)
```

**Problèmes visibles** :
- ❌ Encodages mixtes (Unicode, ASCII)
- ❌ Espaces superflus
- ❌ Dates dans formats différents
- ❌ Revenus en "$2,500,000" vs "2.5M"
- ❌ Téléphones mal formatés
- ❌ Valeurs manquantes ("NULL", vides, "")
- ❌ Scores bizarres ("SCORE:50")

---

## Étape 1 : Diagnostic

### Script de diagnostic

```python
import pandas as pd
import numpy as np

# Chargement
df_raw = pd.read_csv('salesforce_export.csv', encoding='latin-1')

print("=== DIAGNOSTIC ===")
print(f"Forme : {df_raw.shape}")
print(f"\nMissing values:\n{df_raw.isnull().sum()}")
print(f"\nTypes:\n{df_raw.dtypes}")

# Exemples
print("\n=== Exemples EMAIL ===")
print(df_raw['Email'].head(10))

print("\n=== Exemples REVENUE ===")
print(df_raw['Annual_Revenue'].head(10).unique())

print("\n=== Exemples DATES ===")
print(df_raw['Created_Date'].head(10).unique())

# Vérifier doublons
print(f"\nDoublons par Email: {df_raw.duplicated(subset=['Email']).sum()}")
print(f"Doublons par Id: {df_raw.duplicated(subset=['id']).sum()}")

# Stats par colonne
print("\n=== Stats ===")
for col in df_raw.columns:
    unique = df_raw[col].nunique()
    missing = df_raw[col].isnull().sum()
    print(f"{col}: {unique} unique, {missing} missing")
```

**Output diagnostic:**
```
Forme : (50000, 10)

Missing values:
id                  0
Name              125
Email             3420
Phone             8900
Industry         15600
Annual_Revenue   18000
Created_Date        0
Last_Activity   22100
LeadScore        41200
Status             50

Doublons par Email: 342
Doublons par Id: 0

Encoding issues: ~2% (Unicode chars)
```

---

## Étape 2 : Nettoyage

### Script complet de nettoyage

```python
import pandas as pd
import numpy as np
from datetime import datetime
import re

# Chargement
df = pd.read_csv('salesforce_export.csv', encoding='latin-1', low_memory=False)

print("[1] Suppression colonnes inutiles...")
# Garder seulement colonnes analytiques
keep_cols = ['id', 'Name', 'Email', 'Phone', 'Industry', 'Annual_Revenue', 'Created_Date', 'LeadScore', 'Status']
df = df[keep_cols]

print("[2] Nettoyage texte...")
# Trim espaces
for col in ['Name', 'Email', 'Phone', 'Industry', 'Status']:
    df[col] = df[col].astype(str).str.strip()

# Minuscules Email
df['Email'] = df['Email'].str.lower()

# Normaliser valeurs manquantes
df = df.replace(['NULL', 'null', 'NONE', '', 'N/A', '#N/A'], np.nan)

print("[3] Nettoyage Email...")
# Valider format email
def validate_email(email):
    if pd.isna(email) or email == 'nan':
        return np.nan
    if '@' not in email or '.' not in email:
        return np.nan
    return email

df['Email'] = df['Email'].apply(validate_email)

print("[4] Nettoyage Phone...")
# Normaliser téléphones (extract digits only)
def clean_phone(phone):
    if pd.isna(phone):
        return np.nan
    digits = re.sub(r'\D', '', str(phone))
    if len(digits) < 10:
        return np.nan
    return f"+{digits[-10:]}"  # Format: +XXXXXXXXXX

df['Phone'] = df['Phone'].apply(clean_phone)

print("[5] Nettoyage Revenue...")
# Convertir revenus en float
def parse_revenue(rev):
    if pd.isna(rev) or rev == 'nan':
        return np.nan
    rev = str(rev).strip()
    
    # "$2,500,000" → 2500000
    rev = rev.replace('$', '').replace(',', '')
    
    # "2.5M" → 2500000
    if 'M' in rev.upper():
        return float(rev.upper().replace('M', '')) * 1_000_000
    if 'K' in rev.upper():
        return float(rev.upper().replace('K', '')) * 1_000
    
    try:
        return float(rev)
    except:
        return np.nan

df['Annual_Revenue'] = df['Annual_Revenue'].apply(parse_revenue)

print("[6] Nettoyage LeadScore...")
# Extraire score numérique
def parse_score(score):
    if pd.isna(score):
        return np.nan
    score_str = str(score).strip()
    
    # "SCORE:50" → 50
    score_str = score_str.replace('SCORE:', '').strip()
    
    try:
        val = float(score_str)
        # Valider range 0-100
        return val if 0 <= val <= 100 else np.nan
    except:
        return np.nan

df['LeadScore'] = df['LeadScore'].apply(parse_score)

print("[7] Nettoyage Dates...")
# Standardiser dates en ISO 8601
def parse_date(date_str):
    if pd.isna(date_str):
        return np.nan
    
    formats = [
        '%m/%d/%Y %I:%M:%S %p',  # 01/15/2025 10:30:45 AM
        '%Y-%m-%dT%H:%M:%SZ',     # 2025-01-14T08:00:00Z
        '%m/%d/%Y',               # 01/15/2025
        '%Y-%m-%d %H:%M:%S',      # 2025-01-15 14:30:00
    ]
    
    for fmt in formats:
        try:
            return pd.to_datetime(date_str, format=fmt).strftime('%Y-%m-%d')
        except:
            continue
    
    return np.nan

df['Created_Date'] = df['Created_Date'].apply(parse_date)
df['Last_Activity'] = df['Last_Activity'].apply(parse_date)

print("[8] Standardisation Status...")
# Mapper statuts courants
status_map = {
    'Active': 'Active',
    'Prospect': 'Prospect',
    'Lead': 'Prospect',
    'Qualified': 'Qualified',
    'Customer': 'Active',
    'Closed-Lost': 'Closed',
    'Closed-Won': 'Closed',
    'Inactive': 'Inactive',
}

df['Status'] = df['Status'].map(status_map).fillna('Unknown')

print("[9] Validation types...")
# Convertir types
df['id'] = df['id'].astype(str)
df['Annual_Revenue'] = df['Annual_Revenue'].astype('float64')
df['LeadScore'] = df['LeadScore'].astype('float64')

print("[10] Suppression doublons...")
# Garder premier par email
df = df.drop_duplicates(subset=['Email'], keep='first')

print(f"Lignes après nettoyage: {len(df)}")

# Sauvegarder intermédiaire
df.to_csv('crm_cleaned.csv', index=False, encoding='utf-8')
print("✓ Fichier nettoyé: crm_cleaned.csv")
```

**Résultat après nettoyage:**
```
Ligne brute :
"003","Café Dupont","cafe.dupont@example.fr",,,"NULL","01/15/2025",,"SCORE:50","Prospect"

Ligne nettoyée :
003,Café Dupont,cafe.dupont@example.fr,NaN,NaN,NaN,2025-01-15,NaN,50.0,Prospect
```

---

## Étape 3 : Validation

```python
import pandas as pd
from datetime import datetime

df = pd.read_csv('crm_cleaned.csv')

print("=== VALIDATION ===")

# Rule 1: Tous contacts ont soit Email soit Phone
invalid_contact = df[(df['Email'].isna()) & (df['Phone'].isna())]
print(f"Contacts sans Email ET Phone: {len(invalid_contact)}")

# Rule 2: Email valide (pattern)
import re
def is_valid_email(email):
    if pd.isna(email):
        return True  # OK if empty
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

invalid_emails = df[~df['Email'].isna() & ~df['Email'].apply(is_valid_email)]
print(f"Emails invalides: {len(invalid_emails)}")

# Rule 3: Revenue dans range acceptable
invalid_revenue = df[~df['Annual_Revenue'].isna() & ((df['Annual_Revenue'] < 0) | (df['Annual_Revenue'] > 1_000_000_000))]
print(f"Revenus hors range: {len(invalid_revenue)}")

# Rule 4: LeadScore 0-100
invalid_score = df[~df['LeadScore'].isna() & ((df['LeadScore'] < 0) | (df['LeadScore'] > 100))]
print(f"Scores hors range: {len(invalid_score)}")

# Rule 5: Dates raisonnables (après 2020, avant aujourd'hui)
today = datetime.now().strftime('%Y-%m-%d')
invalid_dates = df[~df['Created_Date'].isna() & ((df['Created_Date'] < '2020-01-01') | (df['Created_Date'] > today))]
print(f"Dates invalides: {len(invalid_dates)}")

# Rule 6: Status valide
valid_statuses = ['Active', 'Prospect', 'Qualified', 'Closed', 'Inactive', 'Unknown']
invalid_status = df[~df['Status'].isin(valid_statuses)]
print(f"Statuts invalides: {len(invalid_status)}")

print(f"\nTotal issues: {len(invalid_contact) + len(invalid_emails) + len(invalid_revenue) + len(invalid_score) + len(invalid_dates) + len(invalid_status)}")

# Summary stats
print("\n=== STATS NETTOYÉES ===")
print(f"Total records: {len(df)}")
print(f"Avec Email: {df['Email'].notna().sum()}")
print(f"Avec Phone: {df['Phone'].notna().sum()}")
print(f"Revenue moyen: ${df['Annual_Revenue'].mean():,.0f}")
print(f"LeadScore moyen: {df['LeadScore'].mean():.1f}")
```

---

## Étape 4 : Enrichissement

```python
import pandas as pd
from datetime import datetime, timedelta

df = pd.read_csv('crm_cleaned.csv')

print("[1] Ajout flags...")

# Is prospect vs customer
df['is_customer'] = df['Status'].isin(['Active', 'Closed'])

# Has contact info
df['contact_quality'] = (df['Email'].notna().astype(int) + 
                          df['Phone'].notna().astype(int))

# Revenue category
def revenue_category(rev):
    if pd.isna(rev):
        return 'Unknown'
    if rev < 100_000:
        return 'Small'
    if rev < 1_000_000:
        return 'Medium'
    return 'Enterprise'

df['revenue_category'] = df['Annual_Revenue'].apply(revenue_category)

print("[2] Ajout calculs temps...")

# Days since creation
df['Created_Date'] = pd.to_datetime(df['Created_Date'])
today = pd.to_datetime(datetime.now().strftime('%Y-%m-%d'))
df['days_since_creation'] = (today - df['Created_Date']).dt.days

# Days since last activity
df['Last_Activity'] = pd.to_datetime(df['Last_Activity'], errors='coerce')
df['days_since_activity'] = (today - df['Last_Activity']).dt.days

# Is recently active (last 30 days)
df['is_active_recently'] = df['days_since_activity'] <= 30

print("[3] Ajout scoring...")

# Engagement score (0-100)
df['engagement_score'] = 0

# +20 si email
df.loc[df['Email'].notna(), 'engagement_score'] += 20

# +20 si phone
df.loc[df['Phone'].notna(), 'engagement_score'] += 20

# +20 si revenue known
df.loc[df['Annual_Revenue'].notna(), 'engagement_score'] += 20

# +20 si actif récemment
df.loc[df['is_active_recently'], 'engagement_score'] += 20

# +20 si lead score high
df.loc[df['LeadScore'] >= 50, 'engagement_score'] += 20

print(f"✓ Records enrichis")

# Sauvegarder
df.to_csv('crm_enriched.csv', index=False, encoding='utf-8')
print("✓ Fichier enrichi: crm_enriched.csv")

# Preview
print("\nExemples enrichis:")
print(df[['Name', 'Email', 'revenue_category', 'engagement_score', 'is_active_recently']].head())
```

---

## Étape 5 : Export analytique

```python
import pandas as pd

df = pd.read_csv('crm_enriched.csv')

print("=== EXPORT ANALYTICS ===")

# 1. Export pour BI (CSV optimisé)
export_cols = [
    'id', 'Name', 'Email', 'Phone', 
    'Industry', 'Annual_Revenue', 'revenue_category',
    'Status', 'LeadScore', 'engagement_score',
    'Created_Date', 'days_since_creation', 'is_active_recently',
    'contact_quality', 'is_customer'
]

df_export = df[export_cols].copy()

df_export.to_csv('crm_analytics_ready.csv', index=False, encoding='utf-8')
print("✓ Export BI: crm_analytics_ready.csv")

# 2. Fichier parquet (pour data warehouse)
df_export.to_parquet('crm_analytics_ready.parquet', compression='snappy')
print("✓ Export Warehouse: crm_analytics_ready.parquet")

# 3. Summary report
print("\n=== RAPPORT FINAL ===")
print(f"Total contacts: {len(df)}")
print(f"\nPar status:")
print(df['Status'].value_counts())

print(f"\nPar revenue_category:")
print(df['revenue_category'].value_counts())

print(f"\nEngagement score distribution:")
print(df['engagement_score'].describe())

print(f"\nActivité:")
print(f"  Active dernières 30j: {df['is_active_recently'].sum()} ({df['is_active_recently'].sum()/len(df)*100:.1f}%)")
print(f"  Clients: {df['is_customer'].sum()} ({df['is_customer'].sum()/len(df)*100:.1f}%)")

print(f"\n✓ Nettoyage terminé!")
```

---

## 🎓 Exercices pratiques

### Exercice 24.1 : Audit
Exécutez le script diagnostic sur un export réel.

### Exercice 24.2 : Nettoyage
Appliquez les règles de nettoyage et comparez avant/après.

### Exercice 24.3 : Validation
Identifiez et corrigez anomalies résiduelles.

### Exercice 24.4 : Enrichissement
Ajoutez vos propres flags métier.

### Exercice 24.5 : Visualisation
Importez dans Tableau/Power BI et créez dashboard.

---

**Prêt pour logs JSON? → [Chapitre 25 : Agrégation Logs](./25_Agregation_Logs_JSON.md)**
