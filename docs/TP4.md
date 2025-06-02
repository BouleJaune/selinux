# TP SELinux – Conteneurs Podman en rootless

## Objectifs pédagogiques

* Comprendre comment SELinux isole les conteneurs en mode rootless.
* Manipuler les options `:z`, `:Z`, `--security-opt` dans Podman.
* Observer les contextes des processus et volumes dans l’environnement utilisateur.
* Expérimenter avec des politiques personnalisées (Udica).

---

## Pré-requis

* Fedora ou RHEL avec Podman installé (`dnf install podman udica -y`)
* SELinux activé (`getenforce` doit retourner `Enforcing`)
* Utilisateur non-root avec droits sudo
* Accès SSH ou TTY

---

## Mise en place de l’environnement

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

## Lancer un conteneur simple (rootless)

```bash
podman run --rm -d --name nginx -p 8080:80 docker.io/library/nginx
```

Vérifier que le conteneur tourne et identifier son contexte SELinux :

```bash
ps -eZ | grep nginx
```

Lister les contextes des fichiers :

```bash
ls -Z ~/.local/share/containers/
```

---

## Test de montage de volume sans option de relabel

Créer un dossier à monter :

```bash
mkdir ~/nginx_data
```

Tenter un montage :

```bash
podman run --rm -d --name nginx-vol -v ~/nginx_data:/usr/share/nginx/html:ro -p 8081:80 nginx
```

Inspecter les logs (vous devriez avoir une erreur AVC) :

```bash
journalctl --user -b | grep AVC
```

---

## Résolution via `:z` et `:Z`

Rejouer avec `:z` :

```bash
podman run --rm -d --name nginx-z -v ~/nginx_data:/usr/share/nginx/html:ro,z -p 8082:80 nginx
```

Puis avec `:Z` :

```bash
podman run --rm -d --name nginx-Z -v ~/nginx_data:/usr/share/nginx/html:ro,Z -p 8083:80 nginx
```

Comparer les contextes après chaque lancement :

```bash
ls -Z ~/nginx_data
```

**Questions** :

* Quelle différence entre `:z` et `:Z` ?
* Dans quel cas utiliser l’un ou l’autre ?

---

## Observation du contexte réseau

Lister les sockets utilisés par les conteneurs avec :

```bash
ss -Ztulpn
```

Ou bien :

```bash
sudo ss -tulpnZ | grep 808
```

---

## Utilisation de `--security-opt label=...`

Créer un conteneur avec un contexte explicite :

```bash
podman run --rm -d --name secure-nginx \
  --security-opt label=type:container_runtime_t \
  -p 8084:80 nginx
```

Inspecter :

```bash
ps -eZ | grep nginx
```

---

## Création d’une politique personnalisée avec Udica

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

## Nettoyage

```bash
podman stop -a
podman rm -a
podman rmi nginx
sudo semodule -r secure-nginx
rm -rf ~/nginx_data podman-secure.service
```

---

## Questions bonus

* Pourquoi `:Z` est risqué à utiliser sur des fichiers partagés ?
* Peut-on utiliser `Udica` avec Docker ?
* Que permet de faire `--security-opt label=disable` et dans quels cas est-ce utile ?
