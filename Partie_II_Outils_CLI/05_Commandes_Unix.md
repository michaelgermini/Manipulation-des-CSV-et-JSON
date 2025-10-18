# Chapitre 5 : Commandes Unix essentielles — awk, sed, cut, sort, uniq, join

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Utiliser **sed** pour substitutions et transformations
- Maîtriser **awk** pour traitement texte complexe
- Sélectionner colonnes avec **cut**
- Trier et dédupliquer avec **sort** et **uniq**
- Joindre fichiers avec **join**
- Combiner ces outils en pipelines puissants

---

## 📖 Table des matières

1. [cut — Sélectionner colonnes](#cut--sélectionner-colonnes)
2. [sort & uniq — Trier et dédupliquer](#sort--uniq--trier-et-dédupliquer)
3. [sed — Stream editor (substitutions)](#sed--stream-editor-substitutions)
4. [awk — Text processing avancé](#awk--text-processing-avancé)
5. [join — Jointure de fichiers](#join--jointure-de-fichiers)
6. [Pipelines puissants](#pipelines-puissants)

---

## cut — Sélectionner colonnes

### Syntaxe basique

```bash
cut -d',' -f1,3,5 data.csv
# -d : délimiteur (défaut: TAB)
# -f : numéros de colonnes (1-indexed)
```

### Exemples pratiques

```bash
# Sélectionner colonnes 1 et 3
cut -d',' -f1,3 users.csv

# Colonnes 2 à 5
cut -d',' -f2-5 data.csv

# Toutes sauf colonne 2
cut -d',' -f1,3- data.csv

# Avec délimiteur personnalisé
cut -d'|' -f1,4 pipe_separated.txt

# TAB (défaut)
cut -f1,3 tab_separated.txt
```

### Cas pratique : Extraire emails et noms

```bash
# Entrée: id,name,email
# 1,Alice,alice@example.com
# 2,Bob,bob@example.com

cut -d',' -f2,3 data.csv > names_emails.csv
# Résultat: name,email
#          Alice,alice@example.com
#          Bob,bob@example.com
```

---

## sort & uniq — Trier et dédupliquer

### sort — Tri simple

```bash
# Tri par défaut (lexicographique)
sort data.csv

# Tri numérique (colonne 1)
sort -n -t',' -k1 data.csv
# -t : délimiteur
# -k : clé de tri (numéro colonne)

# Tri décroissant
sort -r data.csv

# Tri multi-colonnes
sort -t',' -k2,2 -k1,1 data.csv  # Trier par col2, puis col1

# Tri numérique avec délimiteur
sort -t',' -k3 -n data.csv  # Trier col 3 numériquement
```

### uniq — Dédupliquer

```bash
# Éliminer lignes dupliquées (nécessite tri avant!)
sort data.csv | uniq

# Compter occurrences
sort data.csv | uniq -c

# Afficher seulement doublons
sort data.csv | uniq -d

# Afficher seulement uniques
sort data.csv | uniq -u
```

### Cas pratique : Top 10 catégories

```bash
# Entrée: category,amount
# books,50
# books,60
# electronics,100
# books,40

cat data.csv | \
  cut -d',' -f1 | \
  sort | \
  uniq -c | \
  sort -rn | \
  head -10

# Résultat:
#      3 books
#      1 electronics
```

---

## sed — Stream editor (substitutions)

### Substitution simple

```bash
# Remplacer première occurrence par ligne
sed 's/old/new/' file.txt

# Remplacer toutes occurrences par ligne (g = global)
sed 's/old/new/g' file.txt

# Avec délimiteur personnalisé
sed 's|/path/old|/path/new|g' file.txt

# Case-insensitive (i = ignore case)
sed 's/OLD/new/gi' file.txt

# In-place edit (modifie le fichier)
sed -i 's/old/new/g' file.txt  # Linux
sed -i '' 's/old/new/g' file.txt  # Mac
```

### Suppression de lignes

```bash
# Supprimer ligne 5
sed '5d' file.txt

# Supprimer lignes 5-10
sed '5,10d' file.txt

# Supprimer lignes vides
sed '/^$/d' file.txt

# Supprimer lignes contenant "pattern"
sed '/pattern/d' file.csv
```

### Cas pratique : Nettoyer CSV

```bash
# Données brutes: "Name, Email, Age"
# (remarquez: espaces autour des virgules)

sed 's/, /,/g' dirty.csv > clean.csv
# Résultat: "Name,Email,Age"

# Supprimer guillemets
sed 's/"//g' data.csv > unquoted.csv

# Convertir séparateur ; → ,
sed 's/;/,/g' semicolon.csv > comma.csv
```

---

## awk — Text processing avancé

### Concepts clés

```bash
# Structure générale
awk '[condition] { action }' file.txt

# Variables spéciales
# NR : numéro de ligne (line number)
# NF : nombre de colonnes (number of fields)
# $0 : ligne entière
# $1, $2, ... : colonnes
# FS : field separator (délimiteur)
# OFS : output field separator
```

### Exemples simples

```bash
# Imprimer colonnes 1 et 3
awk -F',' '{print $1, $3}' data.csv

# Condition : afficher si colonne 1 > 100
awk -F',' '$1 > 100 {print}' data.csv

# Afficher numéro de ligne + contenu
awk '{print NR ": " $0}' file.txt

# Compter lignes
awk 'END {print NR}' file.txt

# Somme colonne (avec délimiteur ',')
awk -F',' '{sum += $2} END {print sum}' data.csv
```

### Cas pratique : Statistiques rapides

```bash
# Données: id,amount,category
# 1,100,books
# 2,200,electronics
# 3,150,books

# Somme par catégorie
awk -F',' '
  NR > 1 {category[$3] += $2}
  END {
    for (cat in category)
      print cat ": " category[cat]
  }
' data.csv

# Résultat:
# books: 250
# electronics: 200
```

### Transformation complexe

```bash
# Combiner deux colonnes
awk -F',' '{print $1 " -> " $2}' data.csv

# Formatage
awk -F',' '{printf "%-10s | %10.2f\n", $1, $2}' data.csv

# Filtrer et transformer
awk -F',' '$2 > 50 {print $1 "," $2/2}' data.csv
```

---

## join — Jointure de fichiers

### Syntaxe

```bash
# Join sur colonne 1 (défaut)
join -t',' file1.csv file2.csv

# Join sur colonne personnalisée
join -t',' -1 2 -2 1 file1.csv file2.csv
# -1 2 : colonne 2 de file1
# -2 1 : colonne 1 de file2

# Left join
join -t',' -a 1 file1.csv file2.csv

# Full outer join
join -t',' -a 1 -a 2 file1.csv file2.csv

# Inner join (défaut)
join -t',' file1.csv file2.csv
```

### ⚠️ Condition importante

**Les fichiers doivent être triés sur la colonne de jointure!**

```bash
# Trier avant de joindre
sort -t',' -k1 file1.csv > file1_sorted.csv
sort -t',' -k1 file2.csv > file2_sorted.csv
join -t',' file1_sorted.csv file2_sorted.csv
```

### Cas pratique : Enrichir données

```bash
# file1.csv (users)
# id,name
# 1,Alice
# 2,Bob

# file2.csv (orders)
# id,amount
# 1,100
# 2,200

# Enrichissement
join -t',' -1 1 -2 1 file1.csv file2.csv

# Résultat:
# 1 Alice 100
# 2 Bob 200
```

---

## Pipelines puissants

### Pipeline 1 : Transformer et nettoyer

```bash
# Entrée: sales.csv (sales_date,product,amount)
# 2025-01-15;iPhone;1000
# 2025-01-16;iPad;500

# Workflow:
# 1. Convertir séparateur (;→,)
# 2. Trier par date
# 3. Sélectionner colonnes
# 4. Calculer sommes

cat sales.csv | \
  sed 's/;/,/g' | \
  sort -t',' -k1 | \
  cut -d',' -f1,2 | \
  awk -F',' '{print $0 "," count++}'

# Résultat:
# 2025-01-15,iPhone,0
# 2025-01-16,iPad,1
```

### Pipeline 2 : Analyser logs

```bash
# Logs Apache (access.log)
# 192.168.1.1 - - [15/Jan/2025:10:30:45] "GET /api/users HTTP/1.1" 200

# Extraire IPs et compter requêtes
cat access.log | \
  awk '{print $1}' | \
  sort | \
  uniq -c | \
  sort -rn | \
  head -10

# Résultat (top 10 IPs):
#     150 192.168.1.1
#      45 192.168.1.2
```

### Pipeline 3 : Validation + Transformation

```bash
# users.csv (id,email,age)
# 1,alice@example.com,28
# 2,invalid-email,35
# 3,bob@example.com,42

# Valider emails et extraire
cat users.csv | \
  grep '@' | \
  awk -F',' '$3 > 25 {print $1 "," $2}' | \
  sort -u

# Résultat (utilisateurs > 25 ans avec email valide)
# 1,alice@example.com
# 3,bob@example.com
```

---

## 🎓 Exercices pratiques

### Exercice 5.1 : Cut
Créez un fichier CSV (id,name,department,salary), puis :
- Extrayez nom et département
- Comptez nombre de colonnes

### Exercice 5.2 : Sort & Uniq
Générez données dupliquées, puis :
- Triez et dédupliquez
- Comptez occurrences par catégorie

### Exercice 5.3 : Sed
Prenez un CSV avec séparateur incorrect, convertissez-le.

### Exercice 5.4 : Awk
Calculez sommes/moyennes par groupe

### Exercice 5.5 : Pipeline complet
Combinez ≥3 commandes pour transformer données brutes en rapport.

---

## 📚 Références

- **Sed Tutorial** : https://www.gnu.org/software/sed/manual/sed.html
- **Awk Tutorial** : https://www.gnu.org/software/gawk/manual/gawk.html
- **Linux Text Processing** : https://tldp.org/LDP/Bash-Beginners-Guide/html/

---

**Prêt pour les workflows? → [Chapitre 6 : Workflows](./06_Workflows_Ligne_Commande.md)**
