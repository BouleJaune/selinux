
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
