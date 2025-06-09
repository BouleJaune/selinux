## Apache HTTPD et restauration des contextes

### Objectif

Corriger un blocage SELinux causé par un mauvais contexte sur un répertoire personnalisé utilisé par Apache.

### Étapes

#### Installation d'Apache

Installez Apache et accéder à un petit fichier depuis un navigateur internet.

??? Note "Solution"
        Installation d'apache et création d'un index.html
        ```bash
        dnf install -y httpd
        systemctl enable --now httpd
        echo "Coucou" > /var/www/html/index.html
        ```
        Ouverture dans le firewall !
        ```bash
        firewall-cmd --add-service=http --permanent
        firewall-cmd --reload
        ```

#### Créer un dossier personnalisé

Mettez les fichiers du serveur web dans un dossier personnalisé ce qui entrainera un blocage.

??? Note "Solution"
        Création d'un dossier avec un fichier index.html
        ```bash
        mkdir /webdata
        echo "Coucou" > /webdata/index.html
        ```
        # Modifier la configuration Apache

        ```bash
        cp /etc/httpd/conf/httpd.conf /tmp/httpd.conf.old
        sed -i 's|/var/www/html|/webdata|g' /etc/httpd/conf/httpd.conf
        sed -i 's|IncludeOptional|#IncludeOptional|g' /etc/httpd/conf/httpd.conf
        systemctl restart httpd
        ```


Accédez à `http://localhost` et notez l'erreur provoquée par SELinux.

#### Vérifier le contexte SELinux

```bash
ls -Zd /webdata
```
Le contexte est incorrect (`default_t`), ce qui empêche Apache d'accéder aux fichiers.

#### Corriger avec `semanage` et `restorecon`

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/webdata(/.*)?"
sudo restorecon -Rv /webdata
```

Rechargez la page web. L'accès doit maintenant être fonctionnel.

---

### Utilisation des booléens SELinux avec SSH

#### Objectif

Activer un booléen SELinux pour permettre une fonctionnalité bloquée par défaut (ex. shell restreint via SSH).

#### Étapes

##### Installer et configurer un shell restreint

```bash
sudo dnf install -y rssh
```

Créer un utilisateur dédié :

```bash
sudo useradd -m -s /usr/bin/rssh rsshuser
sudo passwd rsshuser
```

##### Tester la connexion SSH

Tentative de connexion via SSH. Échec attendu dû à une restriction SELinux.

##### Diagnostiquer le blocage

```bash
sudo ausearch -m AVC -ts recent
sudo audit2why -a
```

Identifier la recommandation et vérifier les booléens associés :

```bash
getsebool -a | grep ssh
```

##### Appliquer la modification

```bash
sudo setsebool -P allowssh_chroot_full_access on
```

Tester à nouveau la connexion SSH.

---

### Manipulation directe des contextes avec `chcon`

#### Objectif

Modifier temporairement un contexte SELinux sans toucher aux règles persistantes, pour un cas de test ou débogage.

#### Étapes

##### Créer un fichier de test

```bash
mkdir ~/testselinux
touch ~/testselinux/file.txt
ls -Z ~/testselinux/file.txt
```

##### Changer le contexte

```bash
sudo chcon -t httpd_sys_content_t ~/testselinux/file.txt
ls -Z ~/testselinux/file.txt
```

##### Lancer un serveur HTTP local

```bash
python3 -m http.server --directory ~/testselinux 8080
```

Accéder à `http://localhost:8080/file.txt`. SELinux ne devrait pas bloquer si le contexte est correctement appliqué.

##### Réinitialiser le contexte

```bash
sudo restorecon -v ~/testselinux/file.txt
```

Vérifier que le contexte d’origine a été restauré.


## Flask — Gestion avancée de la politique SELinux

### Contexte

Vous travaillez toujours sur l'application `fileserve`, un service web développé en Python qui expose des fichiers depuis différents dossiers (`/data/public`, `/data/private`, etc.).
L'application est installée et fonctionnelle *en permissive*, mais vous souhaitez maintenant la rendre pleinement **compatible avec une configuration SELinux en mode `enforcing`**.

---

### Phase 1 — Revenir en mode enforcing

**Question :**
L’application avait été laissée en mode permissive pour faciliter les tests.
Repassez en **mode enforcing**, puis relancez l’application.

**Correction :**

```bash
sudo setenforce 1
getenforce  # => Enforcing
```

→ Vous devriez observer des erreurs (erreurs 403 côté web, ou plantages silencieux).

---

### Phase 2 — Vérifier les labels sur les fichiers

**Question :**
Quels sont les labels appliqués aux fichiers que sert votre application (`/data/*`) ?
Sont-ils compatibles avec une application web ? Sinon, corrigez-les.

**Correction :**

```bash
ls -lZ /data
# Vous constaterez que les fichiers ne sont pas en type httpd_sys_content_t

sudo semanage fcontext -a -t httpd_sys_content_t "/data(/.*)?"
sudo restorecon -Rv /data
```

---

### Phase 3 — Diagnostiquer avec `audit2why`

**Question :**
Quelles sont les erreurs remontées par SELinux ?
Utilisez les bons outils pour les identifier.

**Correction :**

```bash
sudo ausearch -m AVC -ts recent
# Copier une erreur

# Puis :
sudo ausearch -m AVC -ts recent | audit2why
```

→ Identifier si une politique empêche l’accès à un fichier, un port, ou une ressource.

---

### Phase 4 — Vérifier les ports utilisés

**Question :**
L'application écoute sur le port 8080. Est-ce que ce port est autorisé pour `httpd_t` par la politique SELinux actuelle ?

**Correction :**

```bash
sudo semanage port -l | grep http_port_t
# Si 8080 n’est pas listé :
sudo semanage port -a -t http_port_t -p tcp 8080
```

---

### Phase 5 — Vérifier les booléens SELinux

**Question :**
Certains comportements de l’application sont conditionnés par des booléens (accès à des fichiers utilisateurs, réseau, etc.).
Quelles options peuvent être pertinentes dans votre cas ? Activez-les si nécessaire.

**Correction :**

```bash
getsebool -a | grep httpd
# Exemple : autoriser l’accès réseau
sudo setsebool -P httpd_can_network_connect on
```

---

### Phase 6 — Nettoyage et standardisation

**Question :**
Avez-vous utilisé des `chcon` manuels ? Corrigez cela pour garantir que les étiquettes soient **persistantes** après reboot ou relabel.

**Correction :**

* Identifier les `chcon` :

```bash
sudo find /data -context "*:object_r:admin_home_t:s0"  # Exemple de mauvais type
```

* Fixer avec :

```bash
sudo restorecon -Rv /data
```

---

### Phase 7 — Valider l'intégration propre

**Question :**
Redémarrez l'application `fileserve`. L'application fonctionne-t-elle correctement **sans être en permissive**, et **sans avoir de nouveaux AVC** dans les 2 dernières minutes ?

**Correction :**

```bash
sudo ausearch -m AVC -ts recent
# Rien ne doit ressortir
```

---

### Étape bonus

Vous souhaitez déplacer les fichiers statiques vers `/srv/files`. Refaites l’intégration propre **depuis zéro**, mais sur ce nouveau chemin.




## Flask : Création de modules personnalisés SELinux

### Contexte

Votre application `fileserve`, toujours fonctionnelle en mode `enforcing`, a désormais un nouveau composant : un sous-service qui écrit des logs personnalisés dans `/custom/logs` et accède à un fichier de configuration dans `/custom/conf/config.yml`.

Ces actions déclenchent des alertes SELinux, même si les accès sont légitimes dans le cadre de votre service. Il est donc temps de créer une **politique personnalisée**.

---

### Phase 1 — Identifier les AVC en permissive

**Question :**
Repassez SELinux en mode permissive et déclenchez les actions problématiques de l’application (`logs`, lecture de `config.yml`).
Quelles sont les violations signalées ?

**Correction :**

```bash
sudo setenforce 0
# Lancer l’application
sudo ausearch -m AVC -ts recent
```

---

### Phase 2 — Générer un module temporaire avec audit2allow

**Question :**
Générez un module personnalisé à partir des AVC observés.

**Correction :**

```bash
sudo ausearch -m AVC -ts recent | audit2allow -M fileserve_custom
sudo semodule -i fileserve_custom.pp
```

**Attention :** Cela crée un module, mais sans réelle documentation ni contrôle précis des règles.

---

### Phase 3 — Écrire manuellement un module minimal

**Question :**
Plutôt que d’accepter tous les AVC aveuglément, vous allez créer une version maîtrisée du module avec vos propres règles.
Générez le squelette :

**Correction :**

```bash
mkdir fileserve_custom
cd fileserve_custom
sepolicy generate --init fileserve_custom
```

Cela crée un `.te` avec des permissions de base.

---

### Phase 4 — Modifier les règles

**Question :**
Éditez le fichier `.te` et ajoutez manuellement les permissions nécessaires :

* Autoriser `fileserve_custom_t` à lire `/custom/conf/`
* Autoriser l’écriture dans `/custom/logs/`

**Correction :** (dans `fileserve_custom.te`)

```te
# Ajout manuel dans le .te
allow fileserve_custom_t var_log_t:file { write append open getattr };
allow fileserve_custom_t etc_t:file { read open getattr };
```

Et si besoin :

```te
files_type(custom_logs_t)
files_type(custom_conf_t)
```

---

### Phase 5 — Relabelliser les répertoires utilisés

**Question :**
Ajoutez les bons contextes pour `/custom/logs` et `/custom/conf`

**Correction :**

```bash
sudo semanage fcontext -a -t custom_logs_t "/custom/logs(/.*)?"
sudo semanage fcontext -a -t custom_conf_t "/custom/conf(/.*)?"
sudo restorecon -Rv /custom
```

---

### Phase 6 — Compilation et installation du module

**Question :**
Compilez et installez le module que vous venez d’écrire à la main.

**Correction :**

```bash
make -f /usr/share/selinux/devel/Makefile
sudo semodule -i fileserve_custom.pp
```

---

### Phase 7 — Validation finale

**Question :**
Passez SELinux en enforcing. L’application fonctionne-t-elle correctement avec votre module ?

**Correction :**

```bash
sudo setenforce 1
# Lancer le service et surveiller
sudo ausearch -m AVC -ts recent
```

---

### Étape bonus

**Question :**
Vous souhaitez que votre service puisse aussi utiliser des sockets Unix (`/run/fileserve.sock`). Ajoutez cela à votre module, avec les permissions SELinux nécessaires.

