# Chapitre 6 : Workflows en ligne de commande — Pipelines, streaming, compression

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Construire **pipelines robustes** multi-outils
- Gérer **gros fichiers** avec streaming
- **Compresser** efficacement CSV/JSON
- **Vérifier** intégrité avec checksums
- **Paralléliser** traitement avec GNU parallel

---

## 📖 Table des matières

1. [Pipelines composés](#pipelines-composés)
2. [Gestion des gros fichiers](#gestion-des-gros-fichiers)
3. [Compression et décompression](#compression-et-décompression)
4. [Vérification d'intégrité](#vérification-d-intégrité)
5. [Parallélisation et performance](#parallélisation-et-performance)
6. [Recettes complètes](#recettes-complètes)

---

## Pipelines composés

### Concept : Le tuyau Unix

**Philosophie Unix** : Chaque outil fait UNE chose bien.  
**Composition** : Combinez avec `|` (pipe)

```bash
# Pattern
command1 input | command2 | command3 | command4 > output

# Flux de données
Input → cmd1 → cmd2 → cmd3 → cmd4 → Output
```

### Exemple 1 : Nettoyer → Filtrer → Compter

```bash
# Entrée: données CRM brutes
# Nettoyage → Filtrage → Analyse

cat crm_export.csv | \
  sed 's/, /,/g' | \  # Nettoyer espaces
  grep -v '^$' | \  # Éliminer lignes vides
  cut -d',' -f1,2,3 | \  # Sélectionner colonnes
  sort | uniq -c | \  # Compter
  sort -rn  # Tri décroissant

# ⚠️ Bonnes pratiques:
# - Indenter pour clarté
# - Commenter chaque étape
# - Tester chaque étape isolément
```

### Debugging pipelines

```bash
# Technique 1: Tester étape par étape
cat file.csv | head
cat file.csv | sed 's/;/,/g' | head
cat file.csv | sed 's/;/,/g' | cut -d',' -f1,2 | head

# Technique 2: Utiliser tee pour inspecter
cat file.csv | \
  sed 's/;/,/g' | tee step1.txt | \
  cut -d',' -f1,2 | tee step2.txt | \
  sort | uniq -c

# Technique 3: Redirection stderr
command1 2>&1 | command2  # Capture erreurs aussi
```

---

## Gestion des gros fichiers

### Streaming : Ne pas charger tout en mémoire

#### Problème
```bash
# ❌ Mauvais : charge tout en mémoire
cat huge.csv | sort | uniq
# Si fichier > RAM = crash ou très lent
```

#### Solution 1 : External sort
```bash
# ✅ Sort natif gère gros fichiers
# Crée fichiers temporaires si nécessaire
sort huge.csv > sorted.csv

# Avec limite mémoire
sort -S 1G huge.csv > sorted.csv  # Utilise max 1GB
```

#### Solution 2 : Line-by-line processing
```bash
# ✅ Traiter ligne par ligne (streaming)
while IFS=, read id name email; do
  echo "Processing: $name"
done < huge.csv
```

#### Solution 3 : Split et traitement parallèle
```bash
# Diviser en chunks
split -l 100000 huge.csv chunk_

# Traiter chaque chunk
for chunk in chunk_*; do
  cat "$chunk" | process_data >> output.csv &
done
wait  # Attendre tous les jobs
```

### Exemple pratique

```bash
# Données: logs de 10GB
# Filtrer et agréger sans charger tout

# ✅ Efficient (streaming)
cat huge.log | \
  grep "ERROR" | \
  awk '{print $5}' | \
  sort | uniq -c | \
  sort -rn > error_summary.txt

# VS

# ❌ Inefficient (charge mémoire)
grep "ERROR" huge.log > temp.log
awk '{print $5}' temp.log > temp2.log
sort temp2.log > temp3.log
uniq -c temp3.log > error_summary.txt
```

---

## Compression et décompression

### Formats courants

| Format | Ratio | Vitesse | Usage |
|--------|-------|---------|-------|
| **gzip** | ⭐⭐ | ⚡ Rapide | Standard web |
| **bzip2** | ⭐⭐⭐ | ⚠️ Lent | Archivage |
| **xz** | ⭐⭐⭐⭐ | 🔴 Très lent | Compression max |
| **zstd** | ⭐⭐⭐ | ⚡⚡ Très rapide | Modern (2020+) |

### Commandes

```bash
# Gzip
gzip data.csv  # Crée data.csv.gz
gunzip data.csv.gz
gzip -d data.csv.gz

# Dans un pipeline
cat data.csv | gzip > data.csv.gz
gunzip -c data.csv.gz | head

# Bzip2
bzip2 data.csv
bunzip2 data.csv.bz2

# Voir contenu sans décompresser
zcat data.csv.gz | head
bzcat data.csv.bz2 | head
```

### Comparaison ratios

```bash
# Créer test file (100MB CSV)
python3 -c "
import csv
with open('test.csv', 'w') as f:
  w = csv.writer(f)
  for i in range(1000000):
    w.writerow([i, f'data_{i}', f'value_{i}'])
"

# Tester compressions
ls -lh test.csv*
# test.csv        : 100M
# test.csv.gz     : 8M   (92% reduction)
# test.csv.bz2    : 5M   (95% reduction)
# test.csv.xz     : 2M   (98% reduction!)
```

### Streaming avec compression

```bash
# Compresser pendant traitement
cat huge.csv | \
  sed 's/;/,/g' | \
  gzip > processed.csv.gz

# Lire fichier compressé
zcat processed.csv.gz | head

# Traiter → Compresser → Décompresser → Traiter
zcat data.csv.gz | \
  awk -F',' '{print $1, $2}' | \
  gzip > output.csv.gz
```

---

## Vérification d'intégrité

### Checksums MD5 / SHA256

```bash
# Générer checksum
md5sum data.csv > data.csv.md5
sha256sum data.csv > data.csv.sha256

# Vérifier
md5sum -c data.csv.md5
# data.csv: OK

# Vérifier (détection modification)
echo "corrupted" >> data.csv
md5sum -c data.csv.md5
# data.csv: FAILED
```

### Workflow : Transfert sécurisé

```bash
# Côté source
cat large.csv | gzip | tee >(sha256sum > large.csv.gz.sha256) > large.csv.gz

# Côté destination
sha256sum -c large.csv.gz.sha256
# large.csv.gz: OK

gunzip large.csv.gz
```

### Validation complète

```bash
#!/bin/bash
# workflow_validate.sh

FILE=$1

# 1. Vérifier format
echo "[1] Vérification format CSV..."
head -1 "$FILE" | grep -q ','
[ $? -eq 0 ] && echo "✓ Format OK" || echo "✗ Format invalide"

# 2. Compter lignes
echo "[2] Comptage lignes..."
lines=$(wc -l < "$FILE")
echo "✓ $lines lignes"

# 3. Vérifier délimiteur cohérent
echo "[3] Vérification délimiteur..."
invalid=$(awk -F',' 'NR>1 && NF!=NF {print NR}' "$FILE" | wc -l)
[ "$invalid" -eq 0 ] && echo "✓ Délimiteur OK" || echo "✗ $invalid lignes invalides"

# 4. Générer checksum
echo "[4] Génération checksum..."
sha256sum "$FILE" > "$FILE.sha256"
echo "✓ Checksum: $(cat $FILE.sha256)"
```

---

## Parallélisation et performance

### GNU Parallel

```bash
# Installation
apt-get install parallel  # Linux
brew install parallel     # Mac

# Syntaxe basique
parallel [options] [command] ::: [inputs]

# Exemple: Traiter chunks en parallèle
parallel 'cat {} | wc -l' ::: chunk_*

# Avec remplaçant {}
parallel 'gzip -c {} > {.}.gz' ::: *.txt

# Nombre de jobs
parallel -j 4 'command {}' ::: files*
# -j 4 : 4 processus parallèles
# -j 0 : Tous les cores disponibles
```

### Exemples pratiques

```bash
# Compress tous les CSV en parallèle
parallel 'gzip {}' ::: *.csv

# Traiter logs en parallèle
split -l 100000 huge.log chunk_
parallel 'grep ERROR {} | wc -l' ::: chunk_* | awk '{sum+=$1} END {print sum}'

# Pipeline parallèle
cat data.csv | \
  parallel -k --pipe --block 10M 'sort'

# -k : Maintenir ordre
# --pipe : Lit depuis stdin
# --block : Taille chunk
```

### Benchmark

```bash
#!/bin/bash

FILE="test_10M.csv"

# Sequential (baseline)
time sort "$FILE" > /dev/null

# Parallel avec 4 jobs
time parallel -j 4 'cat {}' ::: chunk_* | sort > /dev/null

# Résultats typiques:
# Séquentiel : 15 secondes
# Parallèle  : 5 secondes (3x plus rapide!)
```

---

## Recettes complètes

### Recette 1 : ETL CSV quotidien

```bash
#!/bin/bash
# daily_etl.sh

SOURCE="crm_export_$(date +%Y%m%d).csv"
WORK_DIR="/tmp/etl_$(date +%s)"
OUTPUT="processed_$(date +%Y%m%d).csv.gz"

mkdir -p "$WORK_DIR"
cd "$WORK_DIR"

echo "[1] Téléchargement..."
# curl -o "$SOURCE" https://api.crm.com/export

echo "[2] Validation..."
lines=$(wc -l < "$SOURCE")
echo "  → $lines lignes"

echo "[3] Nettoyage..."
cat "$SOURCE" | \
  sed 's/, /,/g' | \
  sed 's/"//g' | \
  grep -v '^$' > clean.csv

echo "[4] Transformation..."
cat clean.csv | \
  awk -F',' '{
    gsub(/^ +| +$/, "", $2)  # Trim espaces
    print $1","$2","$3
  }' > transformed.csv

echo "[5] Compression..."
gzip -c transformed.csv > "$OUTPUT"
sha256sum "$OUTPUT" > "$OUTPUT.sha256"

echo "[6] Backup..."
# cp "$OUTPUT" /backup/
# cp "$OUTPUT.sha256" /backup/

echo "✓ Terminé: $OUTPUT"
ls -lh "$OUTPUT"
```

### Recette 2 : Analyser logs JSON streaming

```bash
#!/bin/bash
# analyze_logs.sh

LOGFILE=$1

echo "Analyzing: $LOGFILE"
echo "================================"

# Top 10 erreurs
echo "Top 10 erreurs:"
zcat "$LOGFILE" | \
  jq -r '.level,.message | @csv' | \
  grep ERROR | \
  cut -d',' -f2 | \
  sort | uniq -c | \
  sort -rn | head -10

# Requêtes par service
echo -e "\nRequêtes par service:"
zcat "$LOGFILE" | \
  jq -r '.service' | \
  sort | uniq -c

# Latence moyenne
echo -e "\nLatence moyenne (ms):"
zcat "$LOGFILE" | \
  jq '.latency_ms' | \
  awk '{sum+=$1; count++} END {print sum/count}'
```

### Recette 3 : Synchronisation fiable

```bash
#!/bin/bash
# sync_with_resume.sh

SOURCE=$1
DEST=$2
LOG="sync_$(date +%Y%m%d_%H%M%S).log"

echo "Syncing: $SOURCE → $DEST" | tee -a "$LOG"

# rsync avec reprise et vérification
rsync -avz \
  --partial \
  --append-verify \
  --checksum \
  --progress \
  --log-file="$LOG" \
  "$SOURCE" "$DEST"

# Générer rapport
echo "" | tee -a "$LOG"
echo "=== Rapport ===" | tee -a "$LOG"
lines=$(grep 'sent\|total' "$LOG" | tail -1)
echo "$lines" | tee -a "$LOG"
```

---

## 🎓 Exercices pratiques

### Exercice 6.1 : Pipeline simple
Créez un pipeline qui nettoie, filtre et agrège données.

### Exercice 6.2 : Gros fichier
Générez 1GB CSV, testez streaming vs chargement mémoire.

### Exercice 6.3 : Compression comparative
Comparez ratios gzip vs bzip2 vs xz.

### Exercice 6.4 : Checksum
Transférez fichier avec vérification intégrité.

### Exercice 6.5 : Parallélisation
Utilisez `parallel` pour traiter multiples fichiers rapidement.

---

## 📚 Références

- **GNU Parallel** : https://www.gnu.org/software/parallel/
- **Compression Benchmarks** : https://en.wikipedia.org/wiki/Comparison_of_file_archivers
- **rsync Manual** : https://linux.die.net/man/1/rsync

---

**Fin de la Partie II! → [Partie III : Frameworks](../Partie_III_Frameworks/07_Python.md)**
