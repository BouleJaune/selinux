
# TP : Introduction à SELinux et Analyse des blocages

## Objectifs

* Comprendre le fonctionnement général de SELinux
* Identifier et interpréter les contextes SELinux
* Manipuler les outils de diagnostic des blocages (AVC)
* Résoudre des problèmes classiques liés à la politique de sécurité

---

## Prérequis

* VM Fedora Server avec SELinux activé (`getenforce` renvoie `Enforcing`)
* Accès root ou via `sudo`
* Paquets utiles installés : `policycoreutils`, `policycoreutils-python-utils`, `setroubleshoot`, `audit`, `selinux-policy-devel`

---

## Partie 1 : Découverte et état de SELinux

### 1. Vérification de l’état SELinux

```bash
getenforce
sestatus
selinuxenabled && echo "SELinux est activé"
```

> Observez le mode (`Enforcing`, `Permissive`, `Disabled`) et la politique en cours (`targeted`, `mls`, etc.)

### 2. Observation des contextes

```bash
ls -Z /
ps -eZ | grep sshd
id -Z
```

> Notez la forme du contexte : `user:role:type:level`. Essayez de repérer les parties utiles.

---

## Partie 2 : Mise en évidence d’un blocage SELinux

### 3. Mise en place d’un blocage SELinux volontaire

```bash
mkdir /srv/testselinux
cp /bin/bash /srv/testselinux/bash
chmod +x /srv/testselinux/bash
/srv/testselinux/bash
```

> Le binaire `bash` copié perd son label habituel. SELinux devrait bloquer son exécution. Que se passe-t-il ? Testez avec `getenforce permissive` pour comparer.

---

## Partie 3 : Analyse des messages AVC

### 4. Lecture brute des logs

```bash
ausearch -m AVC -ts recent
```

> Identifiez la source, la cible, et l’action interdite.

### 5. Utilisation des outils de diagnostic

```bash
audit2why < /var/log/audit/audit.log
audit2allow -w -a
audit2allow -a
```

> Que propose `audit2allow` ? Est-ce acceptable en production ?

### 6. Interface plus simple : sealert

```bash
sealert -a /var/log/audit/audit.log
```

> Essayez de lire les suggestions de résolution. Vérifiez que le service `setroubleshootd` est actif si la commande ne retourne rien.

---

## Partie 4 : Correction du problème

### 7. Restauration des bons contextes

```bash
restorecon -Rv /srv/testselinux
```

> Relancez le binaire ensuite. Est-ce que ça fonctionne ? Vérifiez à nouveau les contextes (`ls -Z`).

---

## Bonus : test en mode permissif

```bash
setenforce 0
```

* Rejouez l’exécution du binaire copié
* Observez le comportement
* Vérifiez que les messages AVC apparaissent quand même

---

## À la fin du TP, vous devez savoir :

* Lire et comprendre un contexte SELinux
* Détecter les erreurs AVC dans les logs
* Utiliser les outils de diagnostic
* Corriger un problème classique d’étiquetage (`restorecon`)

