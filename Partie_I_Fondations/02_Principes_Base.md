# Chapitre 2 : Principes de base — Encodages, séparateurs, types, sérialisation

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous comprendrez :
- Les encodages de texte et leurs pièges (UTF-8, Latin-1, etc.)
- Les délimiteurs et variantes CSV (`;`, `\t`, `|`)
- Les types de données natifs dans JSON vs CSV
- La sérialisation et désérialisation
- Les best practices pour éviter les erreurs

---

## 📖 Table des matières

1. [Encodages de texte](#encodages-de-texte)
2. [Séparateurs et délimiteurs](#séparateurs-et-délimiteurs)
3. [Types de données et représentation](#types-de-données-et-représentation)
4. [Sérialisation et désérialisation](#sérialisation-et-désérialisation)
5. [Schémas et métadonnées](#schémas-et-métadonnées)
6. [Best practices](#best-practices)

---

## Encodages de texte

### Comprendre les encodages

**Encodage** = Système de conversion entre caractères (texte humain) et bytes (données informatiques).

#### Hiérarchie des encodages

```
ASCII (7-bit, 1970s)
  ↓
Latin-1 / ISO-8859-1 (8-bit, 1980s) — Europe occidentale
  ↓
UTF-8 (variable-width, 1990s) — Multi-langue, universel ✅
  ↓
UTF-16, UTF-32 (moins courants, plus volumineux)
```

### ASCII — Le fondateur

```
A = 65 (0x41)
a = 97 (0x61)
0 = 48 (0x30)
```

**Limitations** :
- Seulement 128 caractères
- Pas de lettres accentuées
- ❌ Inadéquat pour le français, espagnol, arabe, etc.

### Latin-1 (ISO-8859-1) — Le classique européen

Encode 256 caractères (0-255) en 1 byte.

```
é = 233 (0xE9)
ñ = 241 (0xF1)
€ = ❌ NON SUPPORTÉ (introduit UTF-8)
```

**Problème** :
- Limité à Europe occidentale
- Incompatible avec caractères autres

**Où vous le rencontrez** :
- ✅ Vieux exports de bases de données SQL Server (Windows)
- ✅ Fichiers Excel anciens (avant 2010)
- ✅ Legacy systems (mainframes, systèmes propriétaires)

### UTF-8 — L'universel 🌍

**UTF-8** = Unicode Transformation Format, 8-bit.

```
A     = 0x41 (1 byte)
é     = 0xC3 0xA9 (2 bytes)
你    = 0xE4 0xBD 0xA0 (3 bytes)
😀    = 0xF0 0x9F 0x98 0x80 (4 bytes)
```

**Avantages** :
- ✅ Support complet Unicode (tous les langues du monde)
- ✅ Backward compatible avec ASCII
- ✅ Autosynchronisation (pas d'ambiguïté sur limites de caractères)
- ✅ Standard moderne, web natif

**Inconvénients** :
- Taille variable (1-4 bytes par caractère)
- Fichiers UTF-8 avec caractères spéciaux = plus volumineux

### Détection et conversion d'encodage

#### Détecter l'encodage d'un fichier

```bash
# Linux/Mac
file -i data.csv
# Output: data.csv: text/plain; charset=iso-8859-1

# Avec chardet (Python)
chardet data.csv
# Output: {'encoding': 'ISO-8859-1', 'confidence': 0.99}
```

#### Convertir l'encodage

```bash
# Latin-1 → UTF-8
iconv -f ISO-8859-1 -t UTF-8 old.csv > new.csv

# Vérifier
file -i new.csv
# Output: new.csv: text/plain; charset=utf-8
```

#### Python : Gestion robuste

```python
import chardet

# Détecter
with open('data.csv', 'rb') as f:
    result = chardet.detect(f.read())
    detected_encoding = result['encoding']

# Convertir
with open('data.csv', 'r', encoding=detected_encoding) as f:
    data = f.read()

# Réécrire en UTF-8
with open('data_utf8.csv', 'w', encoding='utf-8') as f:
    f.write(data)
```

### Pièges d'encodage courants

#### 1. BOM (Byte Order Mark)

**BOM** = séquence spéciale au début du fichier (`EF BB BF` pour UTF-8).

```
UTF-8 BOM = 0xEF 0xBB 0xBF
```

**Problème** :
```csv
﻿id,nom,email
1,Alice,alice@example.com
```

Le `﻿` est invisible, mais cause des problèmes lors du parsing !

**Solution** :
```python
# Pandas gère automatiquement
df = pd.read_csv('file.csv', encoding='utf-8-sig')  # ← "-sig" = ignore BOM
```

#### 2. Mélange d'encodages

Rare mais possible : fichier partiellement en Latin-1, partiellement en UTF-8.

**Symptôme** : Parse OK jusqu'à un certain point, puis crash ou caractères garnis.

```
Ligne 1-1000: Latin-1 OK
Ligne 1001: "Name ← UTF-8 é" → CRASH
```

**Solution** :
```python
# Lecture avec gestion d'erreurs
df = pd.read_csv('file.csv', encoding='latin-1', errors='replace')
# ↑ Remplace caractères non-décodables par "?"
```

#### 3. Windows vs Unix

**Windows** (anciens): Par défaut Latin-1 (Windows-1252)  
**Unix/Linux** : Par défaut UTF-8  
**Mac** : Historiquement MacRoman, moderne UTF-8

**Réalité** :
```
Excel (Windows) export → fichier.csv [encoding Latin-1]
Importer sous Linux → "Café" devient "Caf?"
```

---

## Séparateurs et délimiteurs

### CSV — Plus qu'une virgule !

**CSV = Comma-Separated Values**, mais les variantes abondent :

| Délimiteur | Nom | Encodage | Exemple | Usage |
|-----------|------|----------|---------|-------|
| `,` | CSV classique | UTF-8 | `id,nom,email` | 🌍 Universel |
| `;` | CSV continental | ISO-8859-1 | `id;nom;email` | 🇫🇷 France, Allemagne |
| `\t` | TSV (Tab) | UTF-8 | `id↹nom↹email` | Bioinformatique, logs |
| `\|` | PSV (Pipe) | UTF-8 | `id\|nom\|email` | Scripts Unix |
| Autre | Custom | ? | `id::nom::email` | Legacy/propriétaire |

### Détection du délimiteur

```python
import csv

# Pandas détection auto
df = pd.read_csv('mystery.csv', sep=None, engine='python')
# Pandas essaye automatiquement différents séparateurs

# Ou sniffer manuel
with open('file.csv', 'r') as f:
    sample = f.read(1024)
    dialect = csv.Sniffer().sniff(sample)
    delimiter = dialect.delimiter
    print(f"Détecté: {repr(delimiter)}")
```

### Guillemets et échappement

**Problème** : Que faire si la valeur elle-même contient le délimiteur ?

```csv
id,nom,ville
1,"Dupont, Jean",Paris          ← Guillemets car contient une virgule
2,"Saint-Jean-de-Luz",Pays-Basque
```

**Règles CSV (RFC 4180)** :
1. Les valeurs contenant `,` doivent être entre guillemets (`"`)
2. Les guillemets littéraux doivent être doublés (`""`)

```csv
1,"Dupont, Jean",Paris           ← Virgule dans valeur
2,"L""équipe d""Alice"",Service  ← Guillemets littéraux
3,"Multi
ligne",Description              ← Retour à la ligne dans valeur
```

**Gestion en Python** :

```python
import csv

# Lecture avec gestion guillemets
with open('data.csv', 'r', encoding='utf-8') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row)  # Guillemets gérés automatiquement

# Écriture
with open('output.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['id', 'nom', 'ville'])
    writer.writeheader()
    writer.writerow({'id': 1, 'nom': 'Dupont, Jean', 'ville': 'Paris'})
    # ↑ Guillemets ajoutés automatiquement
```

### En-têtes (headers)

**Question** : La première ligne est-elle un en-tête ?

```csv
id,nom,email          ← En-tête
1,Alice,alice@example.com
2,Bob,bob@example.com
```

vs

```csv
1,Alice,alice@example.com
2,Bob,bob@example.com
```

**Problème** :
```python
# Faux : traite 1ère ligne comme en-tête
df = pd.read_csv('no_header.csv')
# Résultat: colonne "1" au lieu de "id"

# Correct
df = pd.read_csv('no_header.csv', header=None)
df.columns = ['id', 'nom', 'email']
```

---

## Types de données et représentation

### CSV — Tout est texte

**En CSV, chaque valeur est une chaîne de caractères.**

```csv
id,nom,age,salaire,actif,embauche
1,Alice,28,50000.50,true,2025-01-15
2,Bob,35,60000.75,false,2024-12-01
```

Lors de la lecture :
```python
df = pd.read_csv('data.csv')
print(df.dtypes)
# id           object  (← chaîne, pas entier!)
# nom          object
# age          object  (← chaîne, pas entier!)
# salaire      object  (← chaîne, pas float!)
# actif        object  (← chaîne, pas booléen!)
# embauche     object  (← chaîne, pas date!)
```

**Problème** : Le CSV ne dit pas quels types attendre !

### Conversion de types en CSV

```python
# Approche explicite
df = pd.read_csv('data.csv', dtype={
    'id': int,
    'age': int,
    'salaire': float,
    'actif': bool,
    'embauche': 'datetime64'
})

# Approche infer (plus lente mais souvent OK)
df = pd.read_csv('data.csv')
df = df.infer_objects()  # Essaye de déduire les types
```

### JSON — Types natifs

**JSON a 6 types primitifs :**

```json
{
  "id": 1,                              // number (entier)
  "salaire": 50000.50,                  // number (float)
  "actif": true,                        // boolean
  "surnom": null,                       // null (absence de valeur)
  "nom": "Alice",                       // string
  "compétences": ["Python", "SQL"],    // array
  "adresse": {                          // object
    "rue": "123 rue de la Paix",
    "code_postal": "75001"
  }
}
```

**Avantages** :
- ✅ Types explicites
- ✅ Pas d'ambiguïté (1 vs "1")
- ✅ JSON Schema peut enforcer types

### Sérialisation des types problématiques

#### Dates

```csv
# CSV : comment représenter ?
2025-01-15       # ISO 8601 ← Recommandé
01/15/2025       # US format
15/01/2025       # Format européen
2025-01-15T14:23:45Z  # Avec heure
```

```json
// JSON : ISO 8601 standard
{
  "date": "2025-01-15",
  "timestamp": "2025-01-15T14:23:45Z"
}
```

```python
# Parsing en Python
df = pd.read_csv('data.csv', parse_dates=['embauche'])
# Convertit automatiquement string → datetime

# Ou explicitement
df['embauche'] = pd.to_datetime(df['embauche'], format='%Y-%m-%d')
```

#### Nombres avec séparateurs

```csv
1234567.89     # US/international standard
1.234.567,89   # Europe continentale
1,234,567.89   # US avec séparateur de milliers
```

```python
# Problème
df = pd.read_csv('data.csv', thousands=',')  # Indique le séparateur de milliers

# Ou nettoyer manuellement
df['salaire'] = df['salaire'].str.replace(',', '').astype(float)
```

#### Booléens

```csv
true / false          ← Standard JSON
TRUE / FALSE          ← Variante
T / F                 ← Abrégé
1 / 0                 ← Numérique
yes / no              ← Métier
```

```python
df = pd.read_csv('data.csv', dtype={'actif': bool})
# Pandas accepte: true, false, TRUE, FALSE, 1, 0
```

#### Null/Manquant

```csv
id,nom,email,notes
1,Alice,alice@example.com,
2,Bob,,Important
3,Charlie,charlie@example.com,NULL
,Dave,dave@example.com,
```

**Représentations du manquant** :
- Vide (aucun caractère)
- `NULL` (chaîne)
- `N/A`
- `#N/A`
- `-`

```python
# Pandas configuration
df = pd.read_csv('data.csv', na_values=['', 'NULL', 'N/A', '-'])
print(df)
#    id  nom              email        notes
# 0   1  Alice  alice@example.com        NaN
# 1   2  Bob                NaN    Important
# 2   3  Charlie charlie@example.com     NaN
# 3 NaN  Dave   dave@example.com        NaN
```

```json
// JSON : représentation standard du manquant
{
  "id": 1,
  "email": "alice@example.com",
  "notes": null
}
```

---

## Sérialisation et désérialisation

### Concepts fondamentaux

**Sérialisation** = Convertir objet en-mémoire → format texte (CSV/JSON)  
**Désérialisation** = Convertir texte (CSV/JSON) → objet en-mémoire

```
Objet Python              Sérialiser            Texte (fichier)
┌─────────────────┐      ──────────→           ┌─────────────┐
│ {               │       json.dumps()        │ {"id": 1... │
│  "id": 1,       │                          │             │
│  "nom": "Alice" │                          │             │
│ }               │      ←──────────          │             │
└─────────────────┘      Désérialiser         └─────────────┘
                         json.loads()
```

### Cas : Objet Python → CSV

```python
import csv

# Données en mémoire
users = [
    {'id': 1, 'nom': 'Alice', 'email': 'alice@example.com'},
    {'id': 2, 'nom': 'Bob', 'email': 'bob@example.com'},
]

# Sérialisation en CSV
with open('users.csv', 'w', newline='', encoding='utf-8') as f:
    fieldnames = ['id', 'nom', 'email']
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(users)

# Résultat fichier :
# id,nom,email
# 1,Alice,alice@example.com
# 2,Bob,bob@example.com
```

### Cas : CSV → Objet Python

```python
import csv

# Désérialisation depuis CSV
with open('users.csv', 'r', encoding='utf-8') as f:
    reader = csv.DictReader(f)
    users = list(reader)

print(users)
# [
#   {'id': '1', 'nom': 'Alice', 'email': 'alice@example.com'},
#   {'id': '2', 'nom': 'Bob', 'email': 'bob@example.com'},
# ]

# Attention: id est string! Il faut convertir
for user in users:
    user['id'] = int(user['id'])
```

### JSON : Sérialisation/Désérialisation plus claire

```python
import json

# Objet Python
user = {'id': 1, 'nom': 'Alice', 'age': 28, 'actif': True}

# Sérialisation
json_str = json.dumps(user)
print(json_str)
# {"id": 1, "nom": "Alice", "age": 28, "actif": true}

# Désérialisation
user_restored = json.loads(json_str)
print(user_restored)
# {'id': 1, 'nom': 'Alice', 'age': 28, 'actif': True}

# Types préservés !
print(type(user_restored['id']))  # <class 'int'>
print(type(user_restored['actif']))  # <class 'bool'>
```

### Streaming (pour gros fichiers)

**Problème** : Charger tout en mémoire = risqué si fichier > RAM.

**Solution** : Traiter ligne par ligne.

```python
# CSV : Naturellement streaming
import csv

with open('huge.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:  # Une ligne à la fois
        process(row)

# JSON : Problématique (DOM parsing charge tout)
import json

# ❌ Mauvais : charge entire file
with open('huge.json', 'r') as f:
    data = json.load(f)

# ✅ Meilleur : JSON Lines (un objet par ligne)
# Fichier: huge.jsonl
# {"id": 1, "nom": "Alice"}
# {"id": 2, "nom": "Bob"}

with open('huge.jsonl', 'r') as f:
    for line in f:
        obj = json.loads(line)
        process(obj)
```

---

## Schémas et métadonnées

### CSV : Schéma implicite

CSV n'a pas de schéma formel. Le schéma est...

1. **Les en-têtes** (1ère ligne)
2. **Convention du contexte** (vous le savez)
3. **Documentation externe** (README, wiki)

```csv
# Supposé :
# - id : entier
# - nom : string < 100 chars
# - email : string valide (format email)
# - age : entier entre 18 et 120
id,nom,email,age
1,Alice,alice@example.com,28
2,Bob,bob@example.com,35
```

**Problème** : Rien de formel. Un nouveau parseur pourrait mal interpréter.

### JSON : Schéma explicite avec JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": { "type": "integer" },
    "nom": { "type": "string", "maxLength": 100 },
    "email": { "type": "string", "format": "email" },
    "age": { "type": "integer", "minimum": 18, "maximum": 120 }
  },
  "required": ["id", "nom", "email"]
}
```

**Avantages** :
- ✅ Validation formelle
- ✅ Documentation exécutable
- ✅ Erreurs détectées tôt

```python
# Validation avec jsonschema
import json
from jsonschema import validate, ValidationError

schema = {...}  # Voir ci-dessus
data = {"id": "not_an_int", "nom": "Alice"}  # Invalide

try:
    validate(instance=data, schema=schema)
except ValidationError as e:
    print(f"Erreur: {e.message}")
    # "Erreur: 'not_an_int' is not of type 'integer'"
```

### Métadonnées communes

```csv
# CSV : via comments ou fichier séparé
# Source: Salesforce export
# Date: 2025-01-15
# Encoding: UTF-8
# Delimiter: ,
id,nom,email
1,Alice,alice@example.com
```

```json
// JSON : métadonnées intégrées
{
  "_metadata": {
    "source": "Salesforce export",
    "date": "2025-01-15",
    "encoding": "UTF-8"
  },
  "data": [
    {"id": 1, "nom": "Alice", "email": "alice@example.com"}
  ]
}
```

---

## Best practices

### ✅ DO (À faire)

1. **Toujours UTF-8**
   ```python
   df = pd.read_csv('file.csv', encoding='utf-8')
   ```

2. **Détectez/testez l'encodage**
   ```bash
   file -i data.csv
   chardet data.csv
   ```

3. **Utilisez ISO 8601 pour dates**
   ```csv
   2025-01-15T14:23:45Z
   ```

4. **Explicit typing en CSV**
   ```python
   df = pd.read_csv('data.csv', dtype={
       'id': int,
       'age': int,
       'date': 'datetime64'
   })
   ```

5. **JSON Lines pour streaming**
   ```
   {"id": 1, "nom": "Alice"}
   {"id": 2, "nom": "Bob"}
   ```

6. **Validation de schéma**
   ```python
   validate(data, schema)
   ```

### ❌ DON'T (À éviter)

1. **Assumez l'encodage**
   ```python
   # ❌ Mauvais
   df = pd.read_csv('file.csv')  # Assume système défaut
   
   # ✅ Bon
   df = pd.read_csv('file.csv', encoding='utf-8')
   ```

2. **Dates mal formatées**
   ```csv
   ❌ 01/02/2025  (Ambigu: US vs EU)
   ✅ 2025-01-02  (ISO 8601)
   ```

3. **Mélanger types**
   ```csv
   ❌ id = "1", "2", 3  (Inconsistent)
   ✅ id = 1, 2, 3      (tous entiers)
   ```

4. **Ignorer la casse et espaces**
   ```csv
   ❌ email, Email, EMAIL  (3 colonnes différentes?)
   ✅ email, name, age     (casse cohérente)
   ```

---

## 🎓 Exercices pratiques

### Exercice 2.1 : Audit d'encodage
Trouvez 3 fichiers CSV en local. Pour chacun :
```bash
file -i file.csv
chardet file.csv
```

Documentez les résultats. Tous en UTF-8 ?

### Exercice 2.2 : Conversion
Prenez un fichier non-UTF-8 et convertissez-le :
```bash
iconv -f ISO-8859-1 -t UTF-8 old.csv > new.csv
```

Vérifiez le résultat.

### Exercice 2.3 : Schéma JSON
Créez un JSON Schema pour les users (id, nom, email, age).

---

## 📚 Références

- **UTF-8 Spec** : https://tools.ietf.org/html/rfc3629
- **CSV RFC 4180** : https://tools.ietf.org/html/rfc4180
- **JSON Schema** : https://json-schema.org
- **Pandas dtype** : https://pandas.pydata.org/docs/user_guide/basics.html#dtypes

---

**Prêt pour le pièges ? → [Chapitre 3 : Problèmes classiques](./03_Problemes_Classiques.md)**
