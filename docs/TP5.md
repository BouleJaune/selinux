## Conteneurs Podman


* Comprendre comment SELinux isole les conteneurs.
* Manipuler les options `:z`, `:Z`, `--security-opt` dans Podman.
* Observer les contextes des processus et volumes dans l’environnement utilisateur.
* Expérimenter avec des politiques personnalisées (Udica).

---

### Pré-requis

* Fedora ou RHEL avec Podman et udica installés (`dnf install podman udica -y`)
* SELinux activé en `targeted`


---

## Lancement d’un conteneur de base

Créez un utilisateur dédié à podman et connectez vous dessus.



??? Note "Commandes"
  ```bash
  useradd -m podtest
  passwd podtest
  su - podtest
  ```

Lancez un conteneur nginx nommé `tp-web` sur le port `8080` et observez son contexte.


??? Note "Commandes"

  ```bash
  podman run -d --name tp-web -p 8080:80 docker.io/library/nginx
  firewall-cmd --add-port=8080/tcp 
  ```


  ```bash
  ps -efZ | grep nginx
  ```

---

## Montage d’un volume sans option de label

* Créez un répertoire local et copiez-y un `index.html`
* Montez ce répertoire dans un nouveau conteneur nginx dans `/usr/share/nginx/html`.


??? Note "Commandes"

    ```bash
    mkdir nginx && echo "ok" > index.html
    podman stop tp-web && podman rm tp-web
    podman run -d -name tp-web -p 8080:80 -v ./nginx:/usr/share/nginx/html nginx
    ```

* Testez l'accès au serveur web

* L’accès fonctionne-t-il ? Si non, pourquoi ?

---

## Résolution avec les labels de volumes


Utilisez l'option `:z` ou `:Z` pour résoudre le problème, observez les différences entre les deux options.

??? Note "Commandes"
    Relancez le conteneur avec l’option `:z` :

    ```bash
    podman run -d --name tp-web -p 8080:80 -v ./nginx:/usr/share/nginx/html:z nginx
    ```

    Observez les contextes du répertoire :

    ```bash
    ls -Z
    ```
    Réitérez avec `:Z`. Cette fois ci le volume a les catégories du conteneur en plus.


### Création d’une politique personnalisée avec Udica

Générez un JSON décrivant le conteneur et l'envoyez dans `udica`.
Une fois ceci fait, activez le module.


??? Note "Commandes"

    ```bash
    podman inspect tp-web > container.json
    udica -j container.json tp-web # en root
    ```

    Installer le module généré avec la commande fournie:

    ```bash
    semodule -i tp-web.cil /usr/share/udica/templates/{base_container.cil,net_container.cil}
    ```

    Relancer le conteneur avec le nouveau label.

    ```bash
    podman run -d --name tp-web \
      --security-opt label=type:tp-web.process \
      -v ~/nginx_data:/usr/share/nginx/html \
      -p 8080:80 nginx
    ```

Vérifier que tout fonctionne et observez les contextes des fichiers montés et du container.

```bash
ll -Z nginx
ps -eZ | grep nginx
```

Le dossier nginx n'a pas son contexte modifié sans l'option `:Z`/`:z` mais fonctionne quand même, car la permission a été rajoutée via la policy d'udica.
