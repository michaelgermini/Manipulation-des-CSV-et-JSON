# Chapitre 4 : Outils essentiels — csvkit, xsv, miller (mlr), jq, jshon, jo

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Installer et utiliser **csvkit** pour manipulation CSV avancée
- Maîtriser **xsv** pour performance extrême
- Explorer **miller (mlr)** pour transformations complexes
- Dominer **jq** pour JSON queries et transformations
- Utiliser **jshon** et **jo** pour cas spécifiques

---

## 📖 Table des matières

1. [Csvkit — L'outil multi-usage](#csvkit--loutil-multi-usage)
2. [xsv — La vitesse extrême](#xsv--la-vitesse-extrême)
3. [Miller (mlr) — Le couteau suisse](#miller-mlr--le-couteau-suisse)
4. [jq — Le processeur JSON](#jq--le-processeur-json)
5. [jshon — Lightweight JSON](#jshon--lightweight-json)
6. [jo — Créer JSON en CLI](#jo--créer-json-en-cli)
7. [Comparatif et cas d'usage](#comparatif-et-cas-dusage)

---

## csvkit — L'outil multi-usage

### Installation

```bash
pip install csvkit
```

### Outils principaux

#### 1. `in2csv` — Convertir vers CSV

```bash
# Excel → CSV
in2csv data.xlsx > data.csv

# JSON → CSV
in2csv data.json > data.csv

# Base de données → CSV
in2csv postgresql://user:pass@localhost/db -t "SELECT * FROM users" > users.csv
```

#### 2. `csvstat` — Statistiques rapides

```bash
csvstat data.csv
# Résultat: min, max, moyenne, quantiles pour chaque colonne
```

#### 3. `csvcut` — Sélectionner/réordonner colonnes

```bash
# Sélectionner colonnes
csvcut -c id,nom,email data.csv

# Réordonner
csvcut -c email,nom,id data.csv

# Renommer (colonne 1→"ID", colonne 2→"Name")
csvcut -c 1,2,3 data.csv | csvcut -n
```

#### 4. `csvgrep` — Filtrer lignes

```bash
# Lignes où email contient "gmail"
csvgrep -c email -r "gmail" data.csv

# Lignes où age > 25 (approximatif)
csvgrep -c age -r "^[3-9][0-9]$" data.csv
```

#### 5. `csvformat` — Reformater

```bash
# Latin-1 → UTF-8
csvformat -e iso-8859-1 data.csv > data_utf8.csv

# ; → ,
csvformat -d ";" -D "," data.csv > data_comma.csv
```

#### 6. `csvjoin` — Jointure deux fichiers

```bash
# INNER JOIN sur 'id'
csvjoin -c id users.csv orders.csv > joined.csv

# LEFT JOIN
csvjoin -c id --left-join users.csv orders.csv > joined.csv
```

### Exemple pratique

```bash
# Workflow csvkit complet
# 1. Convertir Excel → CSV
in2csv sales.xlsx > sales.csv

# 2. Stats rapides
csvstat sales.csv

# 3. Filtrer (montant > 1000)
csvgrep -c amount -r "^[1-9][0-9]{3}" sales.csv > sales_1000plus.csv

# 4. Sélectionner colonnes
csvcut -c date,customer,amount sales_1000plus.csv > summary.csv

# 5. Compter lignes
wc -l summary.csv
```

---

## xsv — La vitesse extrême

### Installation

```bash
cargo install xsv  # Rust required
```

### Avantages

- ⚡ **Ultra-rapide** (Rust native)
- 💾 **Bas niveau de mémoire**
- 🔧 Parfait pour gros fichiers

### Commandes principales

#### 1. `xsv count` — Compter rapidement

```bash
xsv count huge.csv
# Résultat: 1000000
```

#### 2. `xsv select` — Sélectionner colonnes

```bash
xsv select id,nom huge.csv
xsv select 1,3,5 huge.csv  # Par index
```

#### 3. `xsv search` — Rechercher

```bash
xsv search "pattern" data.csv
xsv search -i "pattern" data.csv  # Case-insensitive
```

#### 4. `xsv stats` — Statistiques avancées

```bash
xsv stats data.csv
# Plus rapide que csvstat sur gros fichiers
```

#### 5. `xsv behead` — Retirer en-têtes

```bash
xsv behead data.csv
```

#### 6. `xsv slice` — Extraire sous-ensemble

```bash
xsv slice -l 100 data.csv  # Premiers 100 lignes
xsv slice -s 1000 -e 2000 data.csv  # Lignes 1000-2000
```

### Exemple pratique

```bash
# Workflow xsv pour gros fichier
xsv count huge.csv  # 50 millions de lignes
xsv stats huge.csv  # Stats en quelques secondes
xsv search "value" huge.csv | wc -l  # Compter matches
xsv select id,amount huge.csv | xsv sort -s amount > sorted.csv
```

---

## Miller (mlr) — Le couteau suisse

### Installation

```bash
brew install miller  # macOS
apt-get install miller  # Linux
```

### Philosophie

Miller = **AWK** pour CSV/JSON/TSV/etc.

### Commandes principales

#### 1. Conversion format

```bash
# CSV → JSON
mlr --csv --ojson cat data.csv

# JSON → CSV
mlr --json --ocsv cat data.json

# TSV → CSV
mlr --ifs tab --ocsv cat data.tsv
```

#### 2. Sélectionner colonnes

```bash
mlr --csv cut -f id,nom data.csv
mlr --csv cut -o -f id,nom data.csv  # Reorder
```

#### 3. Filtrer

```bash
mlr --csv filter '$age > 25' data.csv
mlr --csv filter '$email =~ "gmail"' data.csv
```

#### 4. Mapper (transformer)

```bash
mlr --csv put '$uppercase_name = toupper($name)' data.csv
mlr --csv put '$year_hired = substr($hire_date, 0, 4)' data.csv
```

#### 5. Agrégation

```bash
mlr --csv stats1 -a sum,mean -f amount -g category data.csv
# Somme et moyenne du montant, groupé par catégorie
```

#### 6. Jointure

```bash
mlr --csv join -l id -r id -f users.csv orders.csv
```

### Exemple pratique

```bash
# Workflow miller complet
# 1. Charger un CSV
mlr --csv --ojson cat data.csv > intermediate.json

# 2. Transformer
mlr --json put '
  $full_name = $first_name . " " . $last_name;
  $age_group = $age < 25 ? "young" : ($age < 65 ? "adult" : "senior")
' intermediate.json

# 3. Filtrer
mlr --json filter '$age_group == "adult"' intermediate.json

# 4. Exporter
mlr --json --ocsv cat intermediate.json > output.csv
```

---

## jq — Le processeur JSON

### Installation

```bash
apt-get install jq  # Linux
brew install jq     # macOS
```

### Concepts clés

- **Pipes** : `|` combine opérations
- **Selectors** : `.field`, `.[]`, `.[0]`
- **Filters** : `select()`, `map()`, `group_by()`
- **Outputs** : JSON, texte, CSV

### Commandes essentielles

#### 1. Affichage formaté

```bash
jq . input.json  # Pretty-print
jq -r . input.json  # Raw (sans guillemets JSON)
```

#### 2. Sélectionner champs

```bash
# Un objet
jq '.id, .name' input.json
jq '.user.email' input.json  # Imbriqué

# Array
jq '.[]' input.json  # Tous les éléments
jq '.items[0]' input.json  # Premier
jq '.items[-1]' input.json  # Dernier
jq '.items[1:3]' input.json  # Slice
```

#### 3. Filtrer (`select`)

```bash
# Éléments où id > 10
jq '.[] | select(.id > 10)' data.json

# Chaînes contenant "alice"
jq '.[] | select(.name | contains("alice"))' data.json

# Clés existantes
jq '.[] | select(.email)' data.json  # email exists
```

#### 4. Transformer (`map`)

```bash
# Appliquer fonction à chaque élément
jq 'map(.name | ascii_upcase)' data.json
jq 'map({id, name})' data.json  # Projection
jq 'map(.id * 2)' data.json  # Calcul
```

#### 5. Grouper et agréger

```bash
# Grouper par catégorie
jq 'group_by(.category)' data.json

# Compter par catégorie
jq 'group_by(.category) | map({category: .[0].category, count: length})'

# Somme par groupe
jq 'group_by(.category) | map({category: .[0].category, total: (map(.amount) | add)})'
```

#### 6. Réduire

```bash
# Somme totale
jq '[.[] | .amount] | add' data.json

# Réduire custom
jq 'reduce .[] as $item (0; . + $item.amount)' data.json
```

#### 7. Construire (construire nouveau JSON)

```bash
# Créer objet
jq '{id, name, email}' input.json

# Créer array
jq '[.id, .name, .email]' input.json

# Renommer clés
jq '{user_id: .id, full_name: .name}' input.json
```

### Exemple pratique

```bash
# Données brutes
curl https://api.example.com/users | jq .

# Filtrer utilisateurs actifs
curl https://api.example.com/users | jq '.[] | select(.status == "active")'

# Extrait noms et emails
curl https://api.example.com/users | jq '.[] | {name, email}'

# Grouper par département
curl https://api.example.com/users | jq 'group_by(.department)'

# Compter par département
curl https://api.example.com/users | jq \
  'group_by(.department) | 
   map({dept: .[0].department, count: length}) |
   sort_by(.count) |
   reverse'
```

---

## jshon — Lightweight JSON

### Installation

```bash
apt-get install jshon
brew install jshon
```

### Pour quand jq est trop lourd

```bash
# Extraire une clé
jshon -e 'id' file.json

# Naviguer imbriqué
jshon -e 'user' -e 'email' file.json

# Array iteration
jshon -a < items.json  # Tous les items
```

---

## jo — Créer JSON en CLI

### Installation

```bash
apt-get install jo
brew install jo
```

### Créer JSON simplement

```bash
# Objet
jo name="Alice" age=28 email="alice@example.com"
# Output: {"name":"Alice","age":28,"email":"alice@example.com"}

# Array d'objets
jo -a $(jo name="Alice" age=28) $(jo name="Bob" age=35)
# Output: [{"name":"Alice","age":28},{"name":"Bob","age":35}]

# De variables
name="Charlie"
email="charlie@example.com"
jo name email
# Output: {"name":"Charlie","email":"charlie@example.com"}
```

---

## Comparatif et cas d'usage

### Tableau comparatif

| Outil | Format | Vitesse | Complexité | Meilleur pour |
|-------|--------|---------|-----------|---------------|
| **csvkit** | CSV, JSON, DB | ⚠️ Lent | 🟡 Moyenne | DB imports, jointures |
| **xsv** | CSV | ⚡⚡⚡ Rapide | 🟢 Simple | Gros fichiers, stats |
| **miller** | CSV, JSON, TSV | ⚡ Rapide | 🟡 Moyenne | Transformations complexes |
| **jq** | JSON | ⚡ Rapide | 🔴 Complexe | Queries JSON avancées |
| **jshon** | JSON | ⚡ Rapide | 🟢 Simple | JSON léger et rapide |
| **jo** | JSON | ⚡ Rapide | 🟢 Simple | Créer JSON en CLI |

### Matrice de décision

**Vous avez un CSV de 1GB ?**
→ **xsv**

**Vous devez joindre deux fichiers CSV ?**
→ **csvkit** (csvjoin) ou **miller**

**Vous devez transformer un gros JSON ?**
→ **jq**

**Vous devez créer du JSON en shell ?**
→ **jo**

**Vous manipulez CSV et JSON ensemble ?**
→ **miller**

---

## 🎓 Exercices pratiques

### Exercice 4.1 : Csvkit
Prenez un fichier Excel, convertissez-le en CSV avec `in2csv`, stat rapides avec `csvstat`.

### Exercice 4.2 : xsv
Générez un CSV de 10M lignes, utilisez `xsv` pour stats et slices.

### Exercice 4.3 : jq avancé
Récupérez des données via API, filtrez et groupez avec jq.

---

## 📚 Références

- **csvkit docs** : https://csvkit.readthedocs.io/
- **xsv repo** : https://github.com/BurntSushi/xsv
- **miller repo** : https://github.com/johnkerl/miller
- **jq manual** : https://stedolan.github.io/jq/manual/

---

**Prêt pour les commandes Unix ? → [Chapitre 5 : Commandes Unix](./05_Commandes_Unix.md)**
