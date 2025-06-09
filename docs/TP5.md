## Conteneurs Podman en rootless

### Objectifs pédagogiques

* Comprendre comment SELinux isole les conteneurs en mode rootless.
* Manipuler les options `:z`, `:Z`, `--security-opt` dans Podman.
* Observer les contextes des processus et volumes dans l’environnement utilisateur.
* Expérimenter avec des politiques personnalisées (Udica).

---

### Pré-requis

* Fedora ou RHEL avec Podman installé (`dnf install podman udica -y`)
* SELinux activé (`getenforce` doit retourner `Enforcing`)
* Utilisateur non-root avec droits sudo
* Accès SSH ou TTY

---

### Mise en place de l’environnement

Créer un utilisateur dédié :

```bash
sudo useradd -m podtest
sudo passwd podtest
```

Se connecter avec ce nouvel utilisateur :

```bash
su - podtest
```

---

## Lancement d’un conteneur de base

Lancez un simple conteneur rootless et observer ses paramètres SELinux.

**Instructions** :

* Connectez-vous avec un utilisateur non privilégié
* Lancez un conteneur simple :

  ```bash
  podman run -d --name tp-web -p 8080:80 docker.io/library/nginx
  ```
* Observez les contextes :

  ```bash
  ps -efZ | grep nginx
  id -Z
  ```

**Questions** :

* Quel est le domaine SELinux du processus nginx ?
* Quel est le contexte SELinux de votre utilisateur ?

---

## Montage d’un volume sans option de label

**But** : Observer un échec lié au contexte SELinux.

**Instructions** :

* Créez un répertoire local et copiez-y un `index.html`
* Montez ce répertoire dans `/usr/share/nginx/html` sans option :

  ```bash
  podman run -d --rm -p 8080:80 -v ./html:/usr/share/nginx/html nginx
  ```
* Ouvrez le navigateur ou faites un `curl localhost:8080`

**Question** :

* L’accès fonctionne-t-il ? Si non, pourquoi ?

---

## 3. Résolution avec les labels de volumes

**But** : Comprendre l’impact de `:z` et `:Z`.

**Instructions** :

* Relancez le conteneur avec l’option `:z` :

  ```bash
  podman run -d --rm -p 8080:80 -v ./html:/usr/share/nginx/html:z nginx
  ```
* Observez les contextes du répertoire :

  ```bash
  ls -Zd ./html
  ```

**Questions** :

* Quelle est la différence entre `:z` et `:Z` ?
* Pourquoi `:z` fonctionne dans ce cas ?

---

## 4. Suppression manuelle des contextes

**But** : Forcer un blocage et observer un AVC.

**Instructions** :

* Supprimez le label :

  ```bash
  chcon -t etc_t ./html
  ```
* Relancez le conteneur avec le volume.
* Observez :

  ```bash
  journalctl -xe | grep AVC
  ```

**Question** :

* Que dit le message AVC ?
* Quelle commande permettrait de restaurer le bon contexte ?

---

## 5. Nettoyage

* Supprimez tous les conteneurs :

  ```bash
  podman rm -a -f
  ```
* Supprimez le répertoire si besoin :

  ```bash
  rm -rf ./html
  ```
---

### Création d’une politique personnalisée avec Udica

Générer le JSON de profil :

```bash
podman generate systemd --name secure-nginx > podman-secure.service
```

Créer une politique personnalisée :

```bash
udica secure-nginx
```

Installer le module généré :

```bash
sudo semodule -i secure-nginx.cil
```

Relancer le conteneur avec le nouveau label :

```bash
podman run --rm -d --name nginx-udica \
  --security-opt label=type:secure_nginx.process \
  -v ~/nginx_data:/usr/share/nginx/html:ro,Z \
  -p 8085:80 nginx
```

Vérifier :

```bash
ps -eZ | grep nginx
```

---

### Nettoyage

```bash
podman stop -a
podman rm -a
podman rmi nginx
sudo semodule -r secure-nginx
rm -rf ~/nginx_data podman-secure.service
```

---

### Questions bonus

* Pourquoi `:Z` est risqué à utiliser sur des fichiers partagés ?
* Peut-on utiliser `Udica` avec Docker ?
* Que permet de faire `--security-opt label=disable` et dans quels cas est-ce utile ?





## Flask — SELinux et conteneurs (Podman rootless)

### Objectif général

L'application `fileserve` développée précédemment doit désormais être conteneurisée avec Podman en **mode rootless**.

Vous devez vous assurer que :

* Le conteneur tourne avec les bons contextes SELinux
* Les volumes sont correctement montés et accessibles
* Les politiques SELinux empêchent toute élévation de privilèges ou sortie du périmètre

---

### Phase 1 — Conteneurisation simple de `fileserve`

**Question :**
Créez une image Podman contenant `fileserve`, construite depuis un `Dockerfile`. Testez le conteneur en rootless.

**Correction :**

```Dockerfile
FROM fedora:latest
RUN dnf install -y python3
COPY fileserve.py /usr/local/bin/fileserve
ENTRYPOINT ["python3", "/usr/local/bin/fileserve"]
```

```bash
podman build -t fileserve .
podman run --rm localhost/fileserve
```

---

### Phase 2 — Activation de SELinux dans les volumes

**Question :**
Montez un volume `/srv/files` contenant des documents. Quel comportement observe-t-on sans options `:z` ou `:Z` ?

**Correction :**

```bash
mkdir -p /srv/files
echo "data" > /srv/files/test.txt

podman run --rm -v /srv/files:/data:ro localhost/fileserve
# Erreur d'accès probable (Permission denied)
```

---

### Phase 3 — Corriger avec `:Z` ou `:z`

**Question :**
Testez avec les deux options et expliquez la différence entre `:z` et `:Z`.

**Correction :**

```bash
podman run --rm -v /srv/files:/data:ro,z localhost/fileserve
podman run --rm -v /srv/files:/data:ro,Z localhost/fileserve
```

* `:z` = partageable entre plusieurs conteneurs
* `:Z` = usage exclusif pour ce conteneur (isolation plus forte)

---

### Phase 4 — Inspection des contextes

**Question :**
Vérifiez les contextes SELinux appliqués au volume monté.

**Correction :**

```bash
ls -Z /srv/files
# Devrait montrer un contexte du type `system_u:object_r:container_file_t:s0`
```

---

### Phase 5 — Vérifier le contexte du conteneur lui-même

**Question :**
Quel est le contexte SELinux du processus dans le conteneur ? Et celui du volume vu de l’intérieur ?

**Correction :**

```bash
podman run --rm localhost/fileserve id -Z
podman run --rm localhost/fileserve ls -Z /data
```

---

### Phase 6 — Empêcher l'accès à d'autres volumes

**Question :**
Créez un second dossier `/srv/private`, montez-le dans un second conteneur. Tentez d’y accéder depuis le premier conteneur. Est-ce autorisé ?

**Correction :**
Si on utilise `:Z` dans les deux, la séparation fonctionne :

```bash
podman run -v /srv/private:/private:Z ...
# Si les deux conteneurs ont :Z, ils ne partagent pas les permissions
```

---

### Phase 7 — Politique SELinux spécifique via `--security-opt`

**Question :**
Changez le label du conteneur au runtime pour le faire fonctionner dans un autre domaine.

**Correction :**

```bash
podman run --security-opt label=type:my_container_t ...
```

Il faut que `my_container_t` soit une catégorie ou type valide dans SELinux.

---

### Phase 8 — Génération d’une politique personnalisée avec udica

**Question :**
Utilisez **udica** pour générer une politique spécifique à votre conteneur `fileserve`.

**Correction :**

```bash
dnf install udica
udica -j fileserve.json
semodule -i fileserve.cil
```

Fichier `fileserve.json` minimal :

```json
{
  "policy_name": "fileserve",
  "container_name": "fileserve",
  "entrypoint": ["/usr/bin/python3"],
  "allow_network": true,
  "allow_pid": false,
  "allow_execmem": false,
  "allow_mount": false
}
```

---

### Phase 9 — Tester avec la nouvelle politique

**Question :**
Redémarrez le conteneur avec le domaine SELinux généré (`fileserve_t`) et observez le résultat.

**Correction :**

```bash
podman run --rm --security-opt label=type:fileserve_t ...
```
