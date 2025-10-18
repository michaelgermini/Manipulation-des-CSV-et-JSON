# Chapitre 21 : SSH, transfert et opérations à distance

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Authentification SSH par clé (sécurisée vs mot de passe)
- Commandes de base : `ssh`, `scp`, `sftp`, `rsync`
- Tunnels et port forwarding
- Automatisation avec Ansible et Paramiko
- Résilience : gestion des coupures réseau
- Bonnes pratiques de sécurité

---

## 📖 Table des matières

1. [Concepts de sécurité SSH](#concepts-de-sécurité-ssh)
2. [Commandes essentielles](#commandes-essentielles)
3. [Tunnels et forwarding](#tunnels-et-forwarding)
4. [Transferts fiables et résilience](#transferts-fiables-et-résilience)
5. [Automation : Ansible & Paramiko](#automation--ansible--paramiko)
6. [Cas pratiques et recettes](#cas-pratiques-et-recettes)

---

## Concepts de sécurité SSH

### Authentification par clé publique

**SSH par défaut = mot de passe** (risqué).  
**SSH moderne = clé RSA/Ed25519** (sécurisé).

#### Génération de clés

```bash
# RSA (2048 bits, classique)
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N "passphrase"

# Ed25519 (meilleur, 256-bit)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "passphrase"

# Options
# -t : type (rsa, dsa, ecdsa, ed25519)
# -b : bits (2048, 4096)
# -f : fichier de sortie
# -N : passphrase (protection de la clé privée)
```

**Résultat** :
```
~/.ssh/id_ed25519       (PRIVÉE - garde secrète!)
~/.ssh/id_ed25519.pub   (PUBLIQUE - peut être distribuée)
```

#### Installation de la clé sur serveur

```bash
# Copier clé pub sur serveur
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@example.com

# Ou manuel (si ssh-copy-id non disponible)
cat ~/.ssh/id_ed25519.pub | ssh user@example.com "cat >> ~/.ssh/authorized_keys"
```

**Sur le serveur** :
```bash
# La clé pub doit être dans ~/.ssh/authorized_keys
ls -la ~/.ssh/authorized_keys
# -rw------- 1 user user 255 Jan 15 10:00 authorized_keys
```

### SSH Agent (gestion des passphrases)

**Problème** : Demander la passphrase à chaque fois = pénible + pas d'automation.  
**Solution** : `ssh-agent` + `ssh-add` = saisir une fois par session.

```bash
# Démarrer agent (souvent auto en Linux)
eval $(ssh-agent)

# Ajouter clé
ssh-add ~/.ssh/id_ed25519
# Enter passphrase for ~/.ssh/id_ed25519:

# Lister clés chargées
ssh-add -l
# 256 SHA256:... user@localhost (ED25519)

# Utilisation
ssh user@example.com  # Pas de demande de passphrase!
```

### Format de clés et recommandations

| Type | Bits | Sécurité | Vitesse | Avenir |
|------|------|----------|---------|--------|
| **RSA** | 2048+ | ⚠️ Bon | ⚡ Rapide | ⚠️ Dépréciable |
| **ECDSA** | 256-521 | ✅ Très bon | ⚡⚡ Très rapide | ✅ Bon |
| **Ed25519** | 256 | ✅ Excellent | ⚡⚡⚡ | ✅ Excellent |

**Recommandation 2025** : **Ed25519** (meilleur compromis).

---

## Commandes essentielles

### 1. `ssh` — Exécution interactive et distante

```bash
# Session interactive
ssh user@host

# Exécuter une commande
ssh user@host "ls -la /tmp"

# Avec options
ssh -p 2222 user@host  # Port personnalisé
ssh -i ~/.ssh/id_ed25519 user@host  # Clé spécifique
ssh -l alice host  # User alternatif
ssh -X user@host  # X11 forwarding (GUI)
ssh -v user@host  # Verbose (debug)
```

### 2. `scp` — Copie simple

```bash
# Local → Remote
scp file.csv user@host:/remote/path/

# Remote → Local
scp user@host:/remote/file.csv .

# Répertoire récursif
scp -r directory/ user@host:/remote/

# Options
scp -P 2222 file user@host:/path/  # Port 2222
scp -p file user@host:/path/       # Préserver permissions
```

**Limitations de scp** :
- ❌ Pas de reprise après coupure
- ❌ Pas efficient sur réseau lent
- ✅ Usage simple pour fichiers uniques

### 3. `sftp` — Session interactive

```bash
# Connexion
sftp user@host

# Commandes dans sftp
> ls                 # Lister remote
> lls                # Lister local
> cd /tmp            # Changer dir remote
> lcd /local/path    # Changer dir local
> get file.csv       # Télécharger
> put data.csv       # Uploader
> bye                # Quit
```

### 4. `rsync` — Synchronisation avec reprise

**Avantage majeur** : Détection delta (ne transfère que changements).

```bash
# Basic sync
rsync -avz source/ user@host:/dest/

# Avec reprise (très important!)
rsync -avz --partial --progress source/ user@host:/dest/

# Limiter bande passante
rsync -avz --bwlimit=1000 source/ user@host:/dest/
# Limite à 1000 KB/s

# Dry-run (vérifier avant)
rsync -avz --dry-run source/ user@host:/dest/

# Options courantes
# -a : archive (récursif, permissions, timestamps)
# -v : verbose
# -z : compression
# --partial : conserver fichiers incomplets pour reprise
# --progress : afficher progression
# --stats : statistiques de transfert
```

**Exemple : transfert fiable avec reprise**

```bash
# Première tentative
rsync -avz --partial --progress huge.csv user@host:/data/

# Coupure réseau après 30 minutes...
# Relancer = continue à partir de là!
rsync -avz --partial --progress huge.csv user@host:/data/
# Reprend depuis le dernier byte écrit
```

### Comparatif : scp vs sftp vs rsync

| Outil | Reprise | Delta | Performance | Complexité |
|-------|---------|-------|-------------|-----------|
| **scp** | ❌ | ❌ | ⚡ Rapide | 🟢 Simple |
| **sftp** | ❌ | ❌ | ⚠️ Moyen | 🟡 Manuel |
| **rsync** | ✅ | ✅ | ⚡⚡⚡ | 🟡 Modéré |

**Recommandation** : **rsync** pour tout transfert > 100MB.

---

## Tunnels et forwarding

### Local Port Forwarding (accéder à service distant)

**Cas** : Base de données interne (port 5432) non exposée publiquement.

```bash
# Tunnel : local:3333 → host:5432
ssh -L 3333:localhost:5432 user@host

# Maintenant, en local
psql -h localhost -p 3333 -U user -d database
```

**Diagramme** :
```
Client Local (port 3333)
    ↓ (encrypted tunnel)
SSH Server
    ↓ (local)
Database (port 5432)
```

### Remote Port Forwarding (exposer service local)

**Cas** : Développement local, exposer app sur jump host.

```bash
# Forward : remote:8080 ← local:3000
ssh -R 8080:localhost:3000 user@host

# Sur host distant
curl localhost:8080  # Accède à local:3000!
```

### SOCKS Proxy

**Cas** : Tunnel tout le trafic via SSH.

```bash
# Créer proxy SOCKS5 local:1080
ssh -D 1080 user@host

# Configurer navigateur/app
# SOCKS5 proxy = localhost:1080
```

---

## Transferts fiables et résilience

### Gestion des coupures réseau

```bash
# Timeout SSH (utile pour réseaux instables)
ssh -o ConnectTimeout=5 -o ServerAliveInterval=60 user@host

# Résilience rsync
rsync -avz \
  --partial \
  --timeout=60 \
  --retries=3 \
  source/ user@host:/dest/

# Logs
rsync -avz --partial --log-file=rsync.log source/ user@host:/dest/
```

### Wrapper bash pour retries

```bash
#!/bin/bash

retry_rsync() {
  local max_attempts=5
  local attempt=1

  while [ $attempt -le $max_attempts ]; do
    echo "Attempt $attempt/$max_attempts..."
    rsync -avz --partial --progress "$1" "$2"
    
    if [ $? -eq 0 ]; then
      echo "✅ Transfert réussi"
      return 0
    fi
    
    echo "❌ Tentative $attempt échouée, nouvelle tentative..."
    sleep 5
    ((attempt++))
  done

  echo "❌ Transfert échoué après $max_attempts tentatives"
  return 1
}

# Usage
retry_rsync "huge.csv" "user@host:/data/"
```

### Monitoring transfert long

```bash
# Garder session open (screen/tmux)
ssh user@host

# Ou lancer en background
nohup rsync -avz --partial source/ user@host:/dest/ > transfer.log 2>&1 &

# Monitorer
tail -f transfer.log
```

---

## Automation : Ansible & Paramiko

### Ansible (orchestration)

```yaml
# playbook.yml
---
- hosts: all
  tasks:
    - name: Copier CSV
      copy:
        src: /local/data.csv
        dest: /remote/data/

    - name: Exécuter script
      shell: python3 /remote/process.py

    - name: Télécharger résultat
      fetch:
        src: /remote/output.json
        dest: ./results/
```

Exécuter :
```bash
ansible-playbook -i hosts.ini playbook.yml
```

### Paramiko (Python)

```python
import paramiko

# Connexion
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect('example.com', username='user', key_filename='~/.ssh/id_ed25519')

# Exécuter commande
stdin, stdout, stderr = ssh.exec_command('ls -la /tmp')
print(stdout.read().decode())

# SCP
sftp = ssh.open_sftp()
sftp.put('/local/file.csv', '/remote/file.csv')
sftp.get('/remote/output.json', '/local/output.json')
sftp.close()

ssh.close()
```

---

## Cas pratiques et recettes

### Cas 1 : Transfert fiable d'export CRM massif

```bash
#!/bin/bash
# transfer_crm_export.sh

SOURCE="crm_export_2025_01_15.csv"
REMOTE="deploy@warehouse.example.com:/data/staging/"
LOG="transfer_$(date +%Y%m%d_%H%M%S).log"

echo "[$(date)] Démarrage transfert $SOURCE" | tee -a $LOG

rsync -avz \
  --partial \
  --append-verify \
  --timeout=120 \
  --checksum \
  --log-file=$LOG \
  "$SOURCE" "$REMOTE"

if [ $? -eq 0 ]; then
  echo "[$(date)] ✅ Transfert réussi" | tee -a $LOG
  # Verifier intégrité
  ssh deploy@warehouse.example.com "md5sum /data/staging/$SOURCE"
else
  echo "[$(date)] ❌ Transfert échoué" | tee -a $LOG
  exit 1
fi
```

### Cas 2 : Exécuter ETL distant et rapatrier résultats

```bash
#!/bin/bash

HOST="etl.example.com"
USER="dataeng"

echo "1. Copier données vers serveur ETL"
rsync -avz --partial data/*.csv $USER@$HOST:/etl/input/

echo "2. Lancer ETL"
ssh $USER@$HOST "cd /etl && python3 etl_pipeline.py"

echo "3. Rapatrier résultats"
rsync -avz --partial $USER@$HOST:/etl/output/ ./results/

echo "✅ Pipeline complet"
```

### Cas 3 : Tunnel vers base distante

```bash
#!/bin/bash

# Terminal 1 : ouvrir tunnel
ssh -L 5432:internal-db:5432 bastion@gateway.example.com

# Terminal 2 : se connecter localement
psql -h localhost -p 5432 -U analyst -d warehouse

# Toutes les requêtes passent via SSH tunnel!
```

---

## 🎓 Exercices pratiques

### Exercice 21.1 : Clés SSH
Générez une paire Ed25519, installez la clé pub sur un serveur de test, connectez-vous sans mot de passe.

### Exercice 21.2 : rsync
Créez un dossier local avec 100 fichiers CSV, synchronisez via rsync, interrompez et reprenez.

### Exercice 21.3 : Tunnel
Créez un tunnel SSH vers une base de données distante et exécutez une requête.

---

## 📚 Références

- **SSH Best Practices** : https://infosec.mozilla.org/guidelines/openssh
- **rsync Manual** : https://linux.die.net/man/1/rsync
- **Ansible Docs** : https://docs.ansible.com/
- **Paramiko Docs** : https://www.paramiko.org/

---

**Prêt pour le stockage sécurisé ? → [Chapitre 22 : Stockage et déplacement sécurisé](./22_Stockage_Deplacement_Securise.md)**
