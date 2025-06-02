## Multi-Level Security (MLS) & Multi-Category Security (MCS)

### Objectifs pédagogiques

* Comprendre les principes de MLS et MCS dans SELinux.
* Manipuler les niveaux (`s0`, `s1`, etc.) et les catégories (`c0`, `c1`, etc.).
* Mettre en œuvre un isolement de données basé sur les niveaux et catégories.
* Gérer les utilisateurs SELinux avec des plages MLS/MCS définies.

---

### Pré-requis

* SELinux activé en mode `enforcing`.
* Politique `mls` ou `targeted` avec MCS (selon la distribution).
* Utiliser une machine Fedora ou RHEL avec le support de MLS/MCS (ex. : `mls` activé via `/etc/selinux/config`).
* Avoir le paquet `policycoreutils`, `mcstrans`, `setools-console` installé.

---

### Contexte du TP

Nous allons simuler une séparation d’informations classifiées par niveaux (MLS) et par catégories (MCS). Deux utilisateurs vont avoir des droits différents sur des fichiers en fonction de leur niveau ou de leur appartenance à des catégories.

---

### Création d’un environnement contrôlé

#### Création des utilisateurs Linux et SELinux

```bash
useradd alice
useradd bob
```

Définir une plage MLS/MCS restreinte à chaque utilisateur :

```bash
semanage user -a -R "staff_r" -r "s0:c0" mcs_alice
semanage user -a -R "staff_r" -r "s0:c1" mcs_bob
semanage login -a -s mcs_alice alice
semanage login -a -s mcs_bob bob
```

Vérification :

```bash
semanage login -l
```

---

### Manipulation de fichiers avec contexte MCS

Créer un fichier classifié pour chaque catégorie :

```bash
touch /tmp/data_alice /tmp/data_bob
chcon -t user_home_t -l s0:c0 /tmp/data_alice
chcon -t user_home_t -l s0:c1 /tmp/data_bob
```

#### Test de lecture croisée

Connectez-vous successivement en tant que `alice` et `bob`, testez l'accès à chaque fichier :

```bash
su - alice
cat /tmp/data_alice
cat /tmp/data_bob
```

Puis :

```bash
su - bob
cat /tmp/data_bob
cat /tmp/data_alice
```

**Question** : Que constatez-vous ? Pourquoi ?

---

### Changement de contexte de processus utilisateur

Observez les contextes SELinux avec :

```bash
id -Z
```

Puis changez temporairement le contexte MCS d’un shell :

```bash
sudo runcon -l s0:c0 -- bash
```

Tentez à nouveau la lecture de fichiers. Testez d'autres catégories.

---

### Utilisation avancée : plages MLS

Modifiez l'utilisateur `alice` pour qu'il ait une plage de niveaux (ex : `s0 - s1`)

```bash
semanage user -m -R "staff_r" -r "s0 - s1:c0.c3" mcs_alice
```

Créez un fichier avec un niveau supérieur :

```bash
chcon -t user_home_t -l s1 /tmp/data_topsecret
```

Testez l’accès avec `alice`. Essayez de changer de niveau avec `runcon`.

---

### Question bonus

* Que se passe-t-il si `alice` a accès à plusieurs catégories ? Peut-elle combiner les lectures ?
* Peut-on attribuer plusieurs catégories à un fichier ?
* Quel est l’impact de la commande `mcstransd` dans l'affichage des contextes ?

---

### Nettoyage

```bash
userdel -r alice
userdel -r bob
semanage login -d alice
semanage login -d bob
semanage user -d mcs_alice
semanage user -d mcs_bob
rm -f /tmp/data_alice /tmp/data_bob /tmp/data_topsecret
```

---

### Aller plus loin

* Créez un script de vérification automatique du niveau et des catégories autorisées d’un utilisateur.
* Intégrez ce mécanisme dans un environnement Podman ou systemd avec `selinux-label`.




## Flask — SELinux & MCS/MLS

### Objectif général

Vous devez restreindre l’accès aux fichiers produits ou consultés par l’application `fileserve` selon des **niveaux de confidentialité** ou **catégories**.

L’objectif est de simuler un système à compartiments : un utilisateur ne doit accéder qu’à ses propres documents en fonction de son niveau ou de sa catégorie (ex : équipe A vs B, ou classification Confidentiel vs Public).

---

### Phase 1 — Activer la politique MLS

**Question :**
Vérifiez si la politique SELinux utilisée est bien `mls`. Sinon, modifiez-la et redémarrez.

**Correction :**

```bash
sestatus | grep Policy
# Pour changer (si nécessaire)
sudo dnf install selinux-policy-mls
sudo grubby --update-kernel=ALL --args="selinux=1 enforcing=1"
sudo setsebool secure_mode_policyload on
sudo reboot
```

---

### Phase 2 — Vérifier les niveaux de sécurité

**Question :**
Quels niveaux sont définis sur votre système ? Quel est celui de votre utilisateur actuel ?

**Correction :**

```bash
semanage user -l
id -Z
```

---

### Phase 3 — Attribuer des niveaux de confidentialité à des utilisateurs

**Question :**
Attribuez à `userA` le niveau `s0:c0`, et à `userB` le niveau `s0:c1`.

**Correction :**

```bash
sudo semanage login -a -s user_u -r s0-s0:c0 userA
sudo semanage login -a -s user_u -r s0-s0:c1 userB
```

Vérifiez :

```bash
semanage login -l
```

---

### Phase 4 — Créer des fichiers étiquetés par niveau

**Question :**
Créez deux fichiers :

* Un fichier accessible uniquement par `userA` (catégorie c0)
* Un fichier accessible uniquement par `userB` (catégorie c1)

**Correction :**

```bash
touch /data/userA.txt /data/userB.txt
chcon --user=user_u --role=object_r --type=default_t --range=s0:c0 /data/userA.txt
chcon --user=user_u --role=object_r --type=default_t --range=s0:c1 /data/userB.txt
```

---

### Phase 5 — Vérification des accès

**Question :**
Connectez-vous en tant que `userA` et `userB`, testez l’accès croisé aux fichiers. Que constatez-vous ?

**Correction :**

```bash
su - userA
cat /data/userA.txt  # OK
cat /data/userB.txt  # Permission denied (attendu)

su - userB
cat /data/userB.txt  # OK
cat /data/userA.txt  # Permission denied (attendu)
```

---

### Phase 6 — Application concrète avec votre service

**Question :**
Configurez `fileserve` pour fonctionner selon le même principe :

* L’instance lancée par `userA` ne doit lire que les fichiers `s0:c0`
* Celle de `userB` que les `s0:c1`

**Correction :**
Il faut :

* S’assurer que les processus ont un contexte `s0:c0` ou `s0:c1`
* Les fichiers servis doivent être labellisés avec le bon range

```bash
runcon -r system_r -t user_t -l s0:c0 ./fileserve  # pour userA
runcon -r system_r -t user_t -l s0:c1 ./fileserve  # pour userB
```

Labellisez les fichiers correctement :

```bash
chcon --range=s0:c0 /data/userA-files/*
chcon --range=s0:c1 /data/userB-files/*
```

---

### Étape bonus — Multi catégories

**Question :**
Attribuez à un utilisateur les deux catégories `c0,c1`, testez qu’il peut lire les deux fichiers.

**Correction :**

```bash
sudo semanage login -a -s user_u -r s0-s0:c0,c1 userAdmin
runcon -r system_r -t user_t -l s0:c0,c1 ./fileserve
```
