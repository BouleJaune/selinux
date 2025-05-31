# Apache HTTPD et restauration des contextes

## Objectif

Corriger un blocage SELinux causé par un mauvais contexte sur un répertoire personnalisé utilisé par Apache.

## Étapes

### Installer Apache

```bash
sudo dnf install -y httpd
sudo systemctl enable --now httpd
```

### Créer un dossier personnalisé

```bash
sudo mkdir /webdata
echo "SELinux TP" | sudo tee /webdata/index.html
sudo chmod -R 755 /webdata
```

### Modifier la configuration Apache

```bash
sudo cp /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/tp.conf
sudo sed -i 's|/var/www/html|/webdata|' /etc/httpd/conf.d/tp.conf
sudo systemctl restart httpd
```

Accédez à `http://localhost` et notez l'erreur 403 provoquée par SELinux.

### Vérifier le contexte SELinux

```bash
ls -Zd /webdata
```

Le contexte est incorrect (`default_t`), ce qui empêche Apache d'accéder aux fichiers.

### Corriger avec `semanage` et `restorecon`

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/webdata(/.*)?"
sudo restorecon -Rv /webdata
```

Rechargez la page web. L'accès doit maintenant être fonctionnel.

---

# Utilisation des booléens SELinux avec SSH

## Objectif

Activer un booléen SELinux pour permettre une fonctionnalité bloquée par défaut (ex. shell restreint via SSH).

## Étapes

### Installer et configurer un shell restreint

```bash
sudo dnf install -y rssh
```

Créer un utilisateur dédié :

```bash
sudo useradd -m -s /usr/bin/rssh rsshuser
sudo passwd rsshuser
```

### Tester la connexion SSH

Tentative de connexion via SSH. Échec attendu dû à une restriction SELinux.

### Diagnostiquer le blocage

```bash
sudo ausearch -m AVC -ts recent
sudo audit2why -a
```

Identifier la recommandation et vérifier les booléens associés :

```bash
getsebool -a | grep ssh
```

### Appliquer la modification

```bash
sudo setsebool -P allowssh_chroot_full_access on
```

Tester à nouveau la connexion SSH.

---

# Manipulation directe des contextes avec `chcon`

## Objectif

Modifier temporairement un contexte SELinux sans toucher aux règles persistantes, pour un cas de test ou débogage.

## Étapes

### Créer un fichier de test

```bash
mkdir ~/testselinux
touch ~/testselinux/file.txt
ls -Z ~/testselinux/file.txt
```

### Changer le contexte

```bash
sudo chcon -t httpd_sys_content_t ~/testselinux/file.txt
ls -Z ~/testselinux/file.txt
```

### Lancer un serveur HTTP local

```bash
python3 -m http.server --directory ~/testselinux 8080
```

Accéder à `http://localhost:8080/file.txt`. SELinux ne devrait pas bloquer si le contexte est correctement appliqué.

### Réinitialiser le contexte

```bash
sudo restorecon -v ~/testselinux/file.txt
```

Vérifier que le contexte d’origine a été restauré.
