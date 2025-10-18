# Chapitre 3 : Problèmes classiques et pièges — Excel, encodage, nombres, dates

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Diagnostiquer et résoudre les pièges Excel courants
- Gérer les zéros manquants (leading zeros, scientific notation)
- Parser correctement les dates ambiguës
- Détecter et corriger l'encodage corrompu
- Éviter les anti-patterns répandus

---

## 📖 Table des matières

1. [Excel — Le problème universel](#excel--le-problème-universel)
2. [Zéros manquants (Leading Zeros)](#zéros-manquants-leading-zeros)
3. [Scientific Notation et Overflow](#scientific-notation-et-overflow)
4. [Dates : l'ambiguïté infernale](#dates-lambiguïté-infernale)
5. [Encodage corrompu](#encodage-corrompu)
6. [Cas complexes réels](#cas-complexes-réels)

---

## Excel — Le problème universel

### Le CSV "Excel-friendly" n'existe pas

**Réalité** : Quand vous exportez un fichier CSV depuis Excel, plusieurs transformations silencieuses s'opèrent.

#### Problème 1 : Séparateur implicite

Vous en **France** exportez en CSV depuis Excel (français) :

```
Fichier.xlsx → "Enregistrer sous" → Fichier.csv
```

**Résultat : délimiteur = `;` (par défaut régional)**

```csv
id;nom;email
1;Alice;alice@example.com
```

Mais votre script Python assume `,` :

```python
df = pd.read_csv('file.csv')  # Assume `,` par défaut
# Résultat: 1 colonne géante!
#    id;nom;email
# 0  1;Alice;alice@example.com
```

**Solution** :
```python
df = pd.read_csv('file.csv', sep=None, engine='python')  # Auto-detect
# Ou explicitement
df = pd.read_csv('file.csv', sep=';', encoding='iso-8859-1')
```

#### Problème 2 : Encodage Excel Windows

Excel Windows exporte par défaut en **Latin-1 (Windows-1252)**, pas UTF-8.

```
"Café" (UTF-8: C3 A9) → Excel → "Café" (Latin-1: E9) → CSV
```

Lors de l'import sous Linux :

```python
df = pd.read_csv('file.csv', encoding='utf-8')  # ❌ Crash ou "Caf?"
df = pd.read_csv('file.csv', encoding='iso-8859-1')  # ✅ OK
```

#### Problème 3 : Suppression de zéros manquants

**Voir section dédiée ci-dessous.** Excel considère `001` comme un nombre et le sauvegarde comme `1`.

#### Problème 4 : Conversion de dates en Serial Numbers

Excel stocke les dates comme **numéros série** (jours depuis 1900).

```
1er janvier 1900 = 1
2 janvier 1900 = 2
...
15 janvier 2025 = 45695
```

**Export CSV problématique** :
```csv
id,nom,embauche
1,Alice,45695
2,Bob,45600
```

Si vous importez cela ailleurs, `45695` = nombre, pas date !

```python
import openpyxl
from datetime import datetime, timedelta

# Conversion Excel serial → date
excel_date = 45695
base_date = datetime(1900, 1, 1)
actual_date = base_date + timedelta(days=excel_date - 2)  # -2 car Excel bug
print(actual_date)  # 2025-01-15
```

**Mieux : exporter Excel au format ISO 8601**

```python
# En Python, formater la date avant export
df['embauche'] = pd.to_datetime(df['embauche']).dt.strftime('%Y-%m-%d')
df.to_csv('output.csv', index=False)
```

#### Problème 5 : Formules non évaluées

Vous avez une formule Excel :
```
=SUM(B1:B10)
```

Export CSV :
```csv
id,total
1,=SUM(B1:B10)
```

À l'import, c'est une **chaîne de texte**, pas une valeur !

```python
df = pd.read_csv('file.csv')
df['total']  # [1, '=SUM(B1:B10)'] ← Chaîne, pas nombre!
```

**Solution** : Avant export, copiez + collez-spécial > valeurs uniquement.

---

## Zéros manquants (Leading Zeros)

### Le drame des codes avec zéros manquants

Vous avez un **code produit : `001234`** (toujours 6 chiffres).

#### Ce qui se passe avec Excel

```
Excel voit: 001234
Excel considère: c'est un nombre → 1234 (zéro manquant!)
Export CSV:
  001234 → "1234"
```

Lors de l'import :
```python
df = pd.read_csv('file.csv')
df['code_produit']
# [1234, 5670, 9012] ← Les zéros manquants!
```

#### Le piège classique

```csv
# Fichier original (bon)
code_produit,nom
001234,Produit A
005678,Produit B

# Après export Excel→CSV (mauvais)
code_produit,nom
1234,Produit A
5678,Produit B
```

### Solutions

#### 1️⃣ **Formater avant export**

En Excel, **avant** export CSV, appliquez le format `Texte` aux colonnes :

```
Sélectionner colonne → Format → Texte
```

Ensuite, Excel conserve les zéros.

#### 2️⃣ **Utiliser l'exportateur CSV d'Excel**

```
Fichier → Exporter → CSV UTF-8 (délimité par des virgules)
```

Au lieu de "Enregistrer sous".

#### 3️⃣ **Padder en Python après import**

```python
df = pd.read_csv('file.csv', dtype={'code_produit': str})
# dtype=str force à traiter comme texte, préserve les zéros

# Ou si c'est déjà corrompu :
df['code_produit'] = df['code_produit'].astype(str).str.zfill(6)
# .zfill(6) ajoute des zéros au début pour atteindre 6 chars
print(df['code_produit'])
# ['001234', '005678', '009012']
```

#### 4️⃣ **Google Sheets — pas mieux**

Google Sheets a le même problème!

```
Télécharger → CSV (Excel)  ← Même comportement
```

**Préférez** :
```
Télécharger → CSV (Feuilles de calcul)  ← Mieux
```

### Cas réel : NIR (Numéro d'inscription à la Rité)

En France, le **NIR** = 13 chiffres, souvent commençant par `1` ou `2`.

Exemple : `1 45 06 75 502 053 47`

Mais parfois : `045 06 75 502 053 47` (commence par `0`).

```python
# Danger : traiter comme nombre
df['NIR'].astype(int)  # ❌ Le zéro initial disparaît!

# Correct : traiter comme texte
df['NIR'].astype(str).str.zfill(13)  # ✅ Préserve les zéros
```

---

## Scientific Notation et Overflow

### Le piège de la notation scientifique

Excel affiche les grands nombres en **notation scientifique** pour gagner de la place.

```
Nombre: 12345678901234
Excel voit: 1.23E+13
Export CSV: "1.23E+13"  ← Texte maintenant!
```

À l'import Python :

```python
df = pd.read_csv('file.csv')
df['telephone']
# ['1.23E+13'] ← Chaîne, pas nombre!

# Conversion
df['telephone'].astype(float)
# 12300000000000.0 ← Précision perdue!
```

### Solution 1️⃣ : Format texte avant export

En Excel :
```
Format → Texte (avant export CSV)
```

### Solution 2️⃣ : Traiter comme string en import

```python
df = pd.read_csv('file.csv', dtype={'telephone': str})
# Préserve "12345678901234"
```

### Exemple réel : Numéro de carte bancaire

```csv
numero_carte
4532015112830366
```

Si Excel le traite comme nombre :
```
4532015112830366 → 4.53E+15 → CSV → "4.53E+15" → Chaîne corrompue!
```

**Correct** :
```python
df = pd.read_csv('file.csv', dtype={'numero_carte': str})
# Résultat: '4532015112830366' ← Exact
```

---

## Dates : l'ambiguïté infernale

### Les 3 formats principaux

Même données, 3 représentations :

```csv
# Format US (MM/DD/YYYY)
embauche: 01/15/2025

# Format EU (DD/MM/YYYY)
embauche: 15/01/2025

# ISO 8601 (YYYY-MM-DD)
embauche: 2025-01-15
```

**Quel est le bon ?** Sans contexte : IMPOSSIBLE de savoir !

### Interprétation ambiguë

```csv
01/02/2025
```

Est-ce :
- 1er février 2025 (US : MM/DD) ?
- 2 janvier 2025 (EU : DD/MM) ?

**Même code produit Excel accepte les DEUX**.

```python
df = pd.read_csv('file.csv')
df['embauche'] = pd.to_datetime(df['embauche'])  # ❌ Erreur ou faux résultat
```

### Solution : Être explicite

```python
# Spécifiez le format
df = pd.read_csv('file.csv', parse_dates=['embauche'], dayfirst=False)
# dayfirst=False → assume MM/DD (US)

# Ou
df = pd.read_csv('file.csv', parse_dates=['embauche'], dayfirst=True)
# dayfirst=True → assume DD/MM (EU)

# Ou format explicite
df['embauche'] = pd.to_datetime(df['embauche'], format='%d/%m/%Y')
```

### Cas réel : Export Salesforce

**Salesforce exporte les dates en ISO 8601 par défaut** ✅

```csv
CreatedDate,Name,Email
2025-01-15T14:23:45Z,Alice,alice@example.com
```

**Mais anciennes versions** : `01/15/2025` (US).

### ISO 8601 — Dites adieu à l'ambiguïté

**Format standard international** : `YYYY-MM-DD` ou avec heure `YYYY-MM-DDTHH:MM:SSZ`

Avantages :
- ✅ Univoque
- ✅ Triable lexicographiquement
- ✅ Supporté par tous les systèmes
- ✅ Timezone explicite (le `Z`)

```csv
embauche
2025-01-15
2025-01-16
```

```python
df = pd.to_datetime(df['embauche'])  # ✅ Pas d'ambiguïté
```

### Heure avec timezone

```csv
# Recommandé
2025-01-15T14:23:45Z           # UTC (Z = Zulu time)
2025-01-15T14:23:45+02:00      # UTC+2

# Pas recommandé en CSV (mais possible)
2025-01-15 14:23:45 CET
```

### Pièges date courants

#### 1. Dates en deux colonnes

```csv
date_jour,date_mois,date_annee
15,1,2025
```

```python
df['embauche'] = pd.to_datetime({
    'day': df['date_jour'],
    'month': df['date_mois'],
    'year': df['date_annee']
})
```

#### 2. Dates fragmentées

Jour de la semaine ajouté :
```csv
embauche,jour
15/01/2025,mercredi
```

À ignorer lors du parsing.

#### 3. Années à 2 chiffres

```csv
embauche
15/01/25
```

Pivot :
```python
df['embauche'] = pd.to_datetime(df['embauche'], format='%d/%m/%y')
# 25 → 2025 (Python assume 20XX par défaut)
```

Attention : `50` = 2050 ou 1950 ?

```python
df['embauche'] = pd.to_datetime(
    df['embauche'],
    format='%d/%m/%y'
).where(df['embauche'] >= '1949-01-01', df['embauche'].dt.year - 100)
```

---

## Encodage corrompu

### Diagnostic

```bash
# Vérifier l'encodage
file -i data.csv
chardet data.csv
```

### Symptômes courants

```python
# Accents deviennent ???
'Alice' → 'Alice'
'Café' → 'Caf?'
'José' → 'Jos?'

# Ou pire : crash à l'import
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9
```

### Récupération

```python
# Essayer plusieurs encodages
for encoding in ['utf-8', 'iso-8859-1', 'cp1252', 'utf-16']:
    try:
        df = pd.read_csv('file.csv', encoding=encoding)
        print(f"✅ Succès avec {encoding}")
        break
    except:
        print(f"❌ Échec avec {encoding}")
```

### Conversion définitive

```python
# Détecter
detected = chardet.detect(open('file.csv', 'rb').read())['encoding']

# Convertir
df = pd.read_csv('file.csv', encoding=detected)
df.to_csv('file_fixed.csv', index=False, encoding='utf-8')
```

---

## Cas complexes réels

### Cas 1 : Export SAP

SAP exporte souvent en **ANSI (Windows-1252)** avec délimiteur `;`.

```python
df = pd.read_csv('sap_export.txt', sep=';', encoding='cp1252')
```

### Cas 2 : Legacy Cobol

Vieilles données **EBCDIC** :

```python
df = pd.read_csv('legacy.dat', encoding='ibm500')
```

### Cas 3 : Données mixtes

Fichier partiellement corrompu :

```python
df = pd.read_csv('mixed.csv', encoding='utf-8', errors='replace')
# Remplace les caractères non-valides par '?'
```

### Cas 4 : Excel avec de vraies formules

```python
# Pour lire un Excel avec formules évaluées
df = pd.read_excel('file.xlsx', engine='openpyxl')
# Vs CSV qui garde les formules texte
```

---

## 🎓 Exercices pratiques

### Exercice 3.1 : Déboguer Excel
Créez un fichier CSV problématique (zéros manquants, dates ambiguës) et diagnostiquez les erreurs.

### Exercice 3.2 : Encodage detectif
Prenez 5 fichiers CSV et identifiez leur encodage avec `file` et `chardet`.

### Exercice 3.3 : Date parsing
Écrivez une fonction Python robuste qui parse les dates dans plusieurs formats.

---

## 📚 Références

- **Pandas read_csv options** : https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html
- **ISO 8601 Dates** : https://en.wikipedia.org/wiki/ISO_8601
- **Excel Date Serial Numbers** : https://support.microsoft.com/en-us/office/date-serial-numbers-3d9b7d9f-0e85-49c5-9d52-fdae22941656

---

**Fin de la Partie I — Prêt pour les outils ? → [Partie II : Outils CLI](../Partie_II_Outils_CLI/04_Outils_Essentiels.md)**
