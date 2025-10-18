# Manipulation Professionnelle des CSV & JSON — Guide Complet

## 📚 Vue d'ensemble

Ce livre s'adresse aux **ingénieurs données**, **développeurs backend**, **analystes** et **responsables DevOps** qui manipulent quotidiennement des fichiers CSV et JSON — que ce soit en local, sur des serveurs distants ou dans des pipelines de production.

**Objectif principal** : Fournir une référence pratique, structurée et opérationnelle couvrant :
- ✅ Les formats (CSV, JSON) : principes, encodages, pièges classiques
- ✅ Les outils CLI : csvkit, xsv, miller, jq, et commandes Unix essentielles
- ✅ Les frameworks & bibliothèques : Python, Node.js, Java, Go, Rust, R
- ✅ Les workflows ETL/ELT : orchestration, pipelines, monitoring
- ✅ La manipulation avancée : streaming, optimisation, performance
- ✅ La sécurité & exploitation distante : SSH, SFTP, transferts sécurisés, Kubernetes
- ✅ Des cas pratiques réels : nettoyage, agrégation, migration, pipelines complets

---

## 📂 Structure du projet

```
Partie_I_Fondations/
  01_Introduction.md
  02_Principes_Base.md
  03_Problemes_Classiques.md

Partie_II_Outils_CLI/
  04_Outils_Essentiels.md
  05_Commandes_Unix.md
  06_Workflows_Ligne_Commande.md

Partie_III_Frameworks/
  07_Python.md
  08_JavaScript_NodeJS.md
  09_Java_Scala.md
  10_Go.md
  11_Rust.md
  12_R.md
  13_Multi_langages_Formats.md

Partie_IV_ETL_Orchestration/
  14_Frameworks_ETL.md
  15_CI_CD_GitOps.md
  16_Monitoring_Observabilite.md

Partie_V_Manipulation_Avancee/
  17_Gros_Fichiers.md
  18_Optimisation_IO_Memoire.md
  19_Tests_Validation.md

Partie_VI_Securite_Distance/
  20_Securite_Donnees.md
  21_SSH_Transfert_Distance.md
  22_Stockage_Deplacement_Securise.md
  23_Conteneurs_Kubernetes.md

Partie_VII_Cas_Pratiques/
  24_Nettoyage_Export_CRM.md
  25_Agregation_Logs_JSON.md
  26_Pipeline_Complet.md
  27_Migration_Parquet_Warehouse.md

Annexes/
  A_Cheatsheets.md
  B_Scripts_Modeles.md
  C_Checklists_Securite.md
  D_Ressources.md

Scripts_Exemples/
  python/
    csv_basic.py
    pandas_advanced.py
    conversion_parquet.py
    streaming_json.py
  nodejs/
    csv_parse_example.js
    json_stream_example.js
  bash/
    csv_processing.sh
    remote_operations.sh

Resources/
  datasets/
  templates/
```

---

## 🎯 Public cible et objectifs d'apprentissage

### Qui devrait lire ce livre ?
- 👨‍💼 Ingénieurs données (Data Engineers)
- 👨‍💻 Développeurs backend
- 📊 Analystes données et BI
- 🔧 DevOps et administrateurs système
- 🚀 Architectes d'intégration et pipelines

### Ce que vous apprendrez
1. **Compréhension approfondie** des formats CSV et JSON
2. **Maîtrise d'outils** essentiels pour la manipulation au quotidien
3. **Frameworks et bibliothèques** pour chaque langage majeur
4. **Patterns de conception** pour pipelines ETL/ELT
5. **Optimisations de performance** pour gros volumes
6. **Sécurité et conformité** dans la manipulation distante
7. **Cas d'usage réels** et solutions éprouvées

---

## 📖 Méthodologie pédagogique

Chaque chapitre contient :

✅ **Objectifs clairs** — Ce que vous saurez faire à la fin du chapitre  
✅ **Concepts clés** — Les principes fondamentaux  
✅ **Pièges courants** — Les erreurs à éviter  
✅ **Exemples opérationnels** — Scripts, commandes, cas réels  
✅ **Exercices pratiques** — Pour renforcer les apprentissages  
✅ **Références** — Liens vers documentation officielle  

---

## 🚀 Comment utiliser ce livre

### Pour débuter
1. Lisez la **Partie I (Fondations)** pour comprendre les bases
2. Explorez la **Partie II (Outils CLI)** pour manipuler localement
3. Sélectionnez la **Partie III (Frameworks)** correspondant à vos langages

### Pour approfondir
4. Étudiez la **Partie IV (ETL/Orchestration)** pour les pipelines
5. Maîtrisez la **Partie V (Manipulation avancée)** pour la performance
6. Apprenez la **Partie VI (Sécurité & Distance)** pour l'exploitation opérationnelle

### Pour appliquer
7. Consultez la **Partie VII (Cas pratiques)** pour vos besoins spécifiques
8. Utilisez les **Annexes** comme références rapides (cheatsheets)

---

## 📋 Table des matières complète

### Partie I — Fondations
- Chapitre 1 : Introduction — Rôle du CSV et JSON dans l'écosystème moderne
- Chapitre 2 : Principes de base — Encodages, séparateurs, types, sérialisation
- Chapitre 3 : Problèmes classiques et pièges — Excel, encodage, nombres, dates

### Partie II — Outils CLI et utilitaires
- Chapitre 4 : Outils essentiels — csvkit, xsv, miller, jq, jshon, jo
- Chapitre 5 : Commandes Unix — awk, sed, cut, sort, uniq, join
- Chapitre 6 : Workflows en ligne de commande — Streaming, pipelines, compression

### Partie III — Frameworks & bibliothèques
- Chapitre 7 : Python — csv, pandas, polars, dask, pyarrow
- Chapitre 8 : JavaScript/Node.js — csv-parse, JSONStream, lodash, ramda
- Chapitre 9 : Java/Scala — Jackson, Gson, OpenCSV, Spark
- Chapitre 10 : Go — encoding/csv, encoding/json, gocsv
- Chapitre 11 : Rust — csv crate, serde_json, serde
- Chapitre 12 : R — readr, data.table, jsonlite
- Chapitre 13 : Multi-langages — Apache Arrow, Parquet, Avro, Kafka

### Partie IV — ETL / Orchestration / Pipelines
- Chapitre 14 : Frameworks ETL — Airflow, Prefect, Dagster, Luigi, dbt
- Chapitre 15 : CI/CD et GitOps pour pipelines de données
- Chapitre 16 : Monitoring et observabilité

### Partie V — Manipulation avancée & performance
- Chapitre 17 : Traitement de gros fichiers — Streaming, chunking, mmap
- Chapitre 18 : Optimisation I/O et mémoire
- Chapitre 19 : Tests, validation et contrats

### Partie VI — Sécurité, confidentialité et exploitation distante
- Chapitre 20 : Sécurité des données — Masquage, anonymisation, chiffrement
- Chapitre 21 : SSH, transfert et opérations à distance
- Chapitre 22 : Stockage et déplacement sécurisé — S3, Azure Blob, GCS
- Chapitre 23 : Conteneurs, Kubernetes et flux de données

### Partie VII — Cas pratiques et recettes
- Chapitre 24 : Nettoyage d'un export CRM (CSV)
- Chapitre 25 : Agrégation de logs JSON pour analytics
- Chapitre 26 : Pipeline complet : API → JSON → Parquet → Data Warehouse
- Chapitre 27 : Migration : CSV massif vers Parquet + catalogage

### Annexes
- Annexe A : Cheatsheets — jq, csvkit, SSH, rsync
- Annexe B : Scripts modèles — Python, Node.js, Bash
- Annexe C : Checklists de sécurité et conformité GDPR/ISO
- Annexe D : Ressources en ligne et bibliographie

---

## 🔗 Liens rapides

- [Partie I : Fondations](./Partie_I_Fondations/)
- [Partie II : Outils CLI](./Partie_II_Outils_CLI/)
- [Partie III : Frameworks](./Partie_III_Frameworks/)
- [Partie IV : ETL & Orchestration](./Partie_IV_ETL_Orchestration/)
- [Partie V : Manipulation Avancée](./Partie_V_Manipulation_Avancee/)
- [Partie VI : Sécurité & Distance](./Partie_VI_Securite_Distance/)
- [Partie VII : Cas Pratiques](./Partie_VII_Cas_Pratiques/)
- [Annexes](./Annexes/)

---

## 💡 Tips pour bien utiliser ce guide

1. **Ayez un terminal ouvert** — Testez les exemples au fur et à mesure
2. **Téléchargez les scripts** — Utilisez-les comme base pour vos propres solutions
3. **Explorez les datasets** — Des exemples CSV/JSON sont fournis
4. **Composez les techniques** — Les approches peuvent souvent se combiner
5. **Mesurez les performances** — Chaque problème a souvent plusieurs solutions
6. **Restez à jour** — Certains outils et versions évoluent rapidement

---

## ⚖️ Licence et contributions

Ce guide est conçu à titre éducatif et de référence professionnelle.  
Les exemples de code sont librement utilisables et modifiables.

---

**Prêt à maîtriser CSV et JSON ? Commençons !** 🚀

Pour naviguer, dirigez-vous vers [Partie I : Fondations](./Partie_I_Fondations/)
