
# Introduction à SELinux et Analyse des blocages

## Découverte et état de SELinux
### Objectifs

* Comprendre le fonctionnement général de SELinux
* Identifier et interpréter les contextes SELinux
* Manipuler les outils de diagnostic des blocages (AVC)
* Résoudre des problèmes classiques liés à la politique de sécurité

---

### Prérequis

* VM Fedora Server avec SELinux
* Accès root
* Paquets utiles installés : `policycoreutils`, `policycoreutils-python-utils`, `setroubleshoot`, `audit`, `selinux-policy-devel`

---


### Vérification de l’état SELinux

Utilisez des commandes CLI pour voir l'état de SELinux et voir quelle elle la policy actuellement chargée.

Vous pouvez aussi utiliser une commande renvoyant un code de retour 1 ou 0 pour voir l'état de SELinux, utile dans les scripts.

??? Note "Commandes"
    ```bash
    getenforce
    sestatus
    selinuxenabled && echo "SELinux est activé"
    ```


### Observation des contextes

Lister les contextes de : fichiers, processes, ports, utilisateur...

??? Note "Commandes"
    ```bash
    ls -Z /
    ps -eZ 
    semanage port -l
    id -Z
    ```

---

### Mise en évidence d’un blocage SELinux

#### Mise en place d’un blocage SELinux volontaire

Faites en sorte d'utiliser le binaire ``bash`` de manière non-standard, par exemple en le copiant, pour générer un blocage.
Passez ensuite en ``permissive`` pour la suite et bien voir que c'est SELinux qui bloque.

??? Note "Commandes"
    ```bash
    mkdir /srv/testselinux
    cp /bin/bash /srv/testselinux/bash
    chmod +x /srv/testselinux/bash
    /srv/testselinux/bash
    ```

### Analyse des messages AVC

#### Lecture brute des logs

Retrouvez ces blocages dans les logs du système de différentes manières et identifiez la source, cible et action interdite.


??? Note "Commandes"
    ```bash
    ausearch -m AVC -ts recent
    ```


#### Interface plus simple : sealert

Essayez de lire les suggestions de résolution. Vérifiez que le service `setroubleshootd` est actif si la commande ne retourne rien.

??? Note "Commandes"
    ```bash
    sealert -a /var/log/audit/audit.log
    ```

### Correction du problème

Restaurez le bon contexte pour débloquer le binaire.

??? Note "Commandes"
    ```bash
    restorecon -Rv /srv/testselinux
    ```

## Flask — Partie 1

---

### Contexte

Vous êtes développeur d'une application Python nommée `fileserve`, développée avec Flask. Elle permet :

* de lire des fichiers depuis un répertoire `/data/public`
* d’écrire des logs dans `/var/log/fileserve.log`
* de servir tout cela sur le port 8080

Elle n’est pas encore conteneurisée : vous l'exécuterez directement sur votre Fedora en tant que service utilisateur.

---

### Prérequis techniques

* L'application Python (Disponible ici [TODO])
* Répertoires :

  * `/data/public/` (contenant des fichiers)
  * `/var/log/fileserve.log`

---

### Étapes

#### Installation de l'application


#### Découverte de l'appli

* Vérifier le mode SELinux
* Observer les **contextes** :
  * de fichiers 
  * de processus 
  * de ports 
  * de sockets 

??? Note "Commandes"
        ```sh
        semanage port -l | grep 8080
        ss -Ztulpen
        ps -eZ | grep flask
        ls -Z /data/public
        ```

#### Premiers blocages

* Lancer le script Python avec `python3 app.py`
* Tenter d’accéder aux fichiers via navigateur ou curl
* Observer les messages d’erreur dans :
  * le navigateur / terminal
  * les logs journaux

* Analyser les logs SELinux :

  * `journalctl -t setroubleshoot`
  * `journalctl | grep AVC`
  * `ausearch -m avc -ts recent`
  * `audit2why` avec le résultat

---

#### Actions correctives

* Identifier si un **label** est incorrect (`ls -Z`)
* Corriger avec :

  * `chcon`
  * `restorecon`
* Comparer les effets des 2
* Vérifier les autorisations sur le port 8080 (`semanage port`)

---

#### Exploration pratique

* Modifier le script pour écrire dans `/tmp`, observer le comportement
* Remonter en **permissive**, relancer (`setenforce 0`), puis `audit2allow` pour proposer une règle
* Revenir en **enforcing**, appliquer `audit2allow` en module temporaire (`checkmodule`, `semodule_package`, `semodule -i`)
