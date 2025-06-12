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

    Modifier la configuration Apache

    ```bash
    cp /etc/httpd/conf/httpd.conf /tmp/httpd.conf.old
    sed -i 's|/var/www/html|/webdata|g' /etc/httpd/conf/httpd.conf
    sed -i 's|IncludeOptional|#IncludeOptional|g' /etc/httpd/conf/httpd.conf
    systemctl restart httpd
    ```


Accédez à `http://{ip-serveur}` et notez l'erreur provoquée par SELinux.

#### Correction

Analysez et corrigez le problème de manière permanente avec semanage.

??? Note "Solution"

    ```bash
    ls -Zd /webdata
    ```
    Le contexte de ``/webdata`` est `default_t`, ce qui empêche surement Apache d'accéder aux fichiers.

    Utilisons ``sealert`` pour voir l'erreur :

    ```bash
    sealert  -a /var/log/audit/audit.log
    ```
    ``sealert`` nous dit de changer le contexte par défaut et nous donnne une longue liste peu pertinente puis nous dit de faire un ``restorecon``.

    On détermine de manière plus précise le bon type en regardant les types liés à ``httpd`` mais aussi plus simplement le type des fichiers par défaut de ``/var/www/html``.

    ```bash
    seinfo -t | grep httpd # On peut voir des types liés au "content"
    ls -lZ /var/www/html/ # Le type de base est httpd_sys_content_t 
    ```
    Correction : 

    ```bash
    # On mets une regex simple pour tout les fichiers dans /webdata
    semanage fcontext -a -t httpd_sys_content_t "/webdata(/.*)?"
    restorecon -Rv /webdata
    ```

    Rechargez la page web. L'accès doit maintenant être fonctionnel.

---


## Utilisation des booléens SELinux


### Objectif

Activer un booléen SELinux pour permettre une fonctionnalité bloquée par défaut (ex. shell restreint via SSH).

### Étapes

#### Faire en sorte qu'Apache se connecte au réseau

Faites un script dans ``/var/www/cgi-bin/`` qui sera atteignable sur le serveur web, exécuté par ``httpd`` et récupère des infos sur d'autres site web du net.


L'url sera sur http://{ip-serveur}/cgi-bin/testnet.sh, mais elle ne devrait pas fonctionner directement.

??? Note "Solution"
    Commençons par remettre à zéro notre configuration httpd :

    ```bash
    cp /tmp/httpd.conf.old /etc/httpd/conf/httpd.conf
    ```

    Le script: 

    ```bash
    mkdir -p /var/www/cgi-bin
    cat > /var/www/cgi-bin/testnet.sh <<EOF
    #!/bin/bash
    echo "Content-type: http"
    echo ""
    curl -s https://channels.nixos.org
    EOF
    chmod +x /var/www/cgi-bin/testnet.sh
    ```

    Rajoutez dans la conf httpd : 

    ```html
    ScriptAlias /cgi-bin/ /var/www/cgi-bin/
    <Directory "/var/www/cgi-bin">
        AllowOverride None
        Options +ExecCGI
        Require all granted
    </Directory>
    ```

    Puis redémarrez ``httpd`` avec ``systemctl restart httpd``

#### Correction du problème SELinux

Observez le blocage SELinux et corrigez via un booléen 

??? Note "Solution"
    ```bash
    sealert -a /var/log/audit/audit.log
    ```
    ``sealert`` nous suggère pour notre erreur 3 solution avec divers scores de confiance.
    La première nous demande si on veut que "htppd can network connect", ce qui est bien notre cas. La solution est suggérée est d'activer un booléen et la commande est fournie.

    ```bash
    setsebool -P httpd_can_network_connect 1 # active le booléen de manière permanente
    ```
    Le site devrait maintenant être accessible.
