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

### Création d’un environnement contrôlé

#### Création des utilisateurs Linux et SELinux

Créez 2 utilisateurs linux ``alice`` et `bob`, puis avec `semanage` créez 2 utilisateurs SELinux `mcs_alice` et `mcs_bob` ayant le rôle `staff_r` et respectivement les ranges `s0:c0` et `s0:c1`.

Enfin assignez `bob` et `alice` à leur user SELinux.

??? Note "Commandes"

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

Prenez ces deux fichiers et assignez leur la bonne range vis à vis du user.

```bash
echo "ok" > /tmp/data_alice 
echo "ok" > /tmp/data_bob
```

??? Note "Commandes"

    ```bash
    chcon -t user_home_t -l s0:c0 /tmp/data_alice
    chcon -t user_home_t -l s0:c1 /tmp/data_bob
    ```

#### Test de lecture

Connectez-vous en tant que `alice` ou `bob`, testez l'accès à chaque fichier et observez ce qu'il se passe.


??? Note "Commandes"
    On se connecte sur le user `alice`:

    ```bash
    su - alice
    cat /tmp/data_alice
    cat /tmp/data_bob
    ```

    Rien ne semble bloqué ! En effet, via ``su -`` le contexte de l'utilisateur n'est pas changé, il faut se connecter via ssh directement sur le user. On peut vérifier le contexte actuel du shell avec ``id -Z``.
   
    ```bash
    passwd alice
    # sur un autre terminal
    ssh alice@server
    id -Z
    cat /tmp/data_bob
    ```

    Cela ne bloque pas non plus !

#### Attribut ``mcs_constrained_type``

Par défaut MCS est activé sur targeted, mais depuis quelques versions de RHEL les catégories ne sont contraignantes que si l'attribut ``mcs_constrained_type`` est rattaché au type.

On peut lister les types rattachés à cet attribut avec : ``seinfo -a mcs_constrained_type -x``:

```bash
Type Attributes: 1
   attribute mcs_constrained_type;
        container_device_plugin_init_t
        container_device_plugin_t
        container_device_t
        container_engine_t
        container_init_t
        container_kvm_t
        container_logreader_t
        container_logwriter_t
        container_t
        container_userns_t
        netlabel_peer_t
        openshift_app_t
        openshift_t
        sandbox_min_t
        sandbox_net_t
        sandbox_web_t
        sandbox_x_t
        svirt_kvm_net_t
        svirt_qemu_net_t
        svirt_t
        svirt_tcg_t
```

On voit que MCS est principalement utilisé par les containers, virtualisation, kubernetes et du sandboxing.

Il nous faut donc rajouter cet attribut à notre type. Le type de nos utilisateurs ici est ``staff_t``.

On peut créer un module avec un fichier ``.te`` puis le compiler pour cela.

??? Note "Ajout de l'attribut au type ``staff_t``"

    ``mystaff.te``:
    ```bash
    policy_module(mystaff, 1.0)
    gen_require(`
        type staff_t;
        attribute mcs_constrained_type;
    ')

    typeattribute staff_t mcs_constrained_type;
    ```
    Compilation et activation:
    ```bash
    make -f /usr/share/selinux/devel/Makefile
    semodule -i mystaff.pp
    ```

Maintenant les users ayant le type ``staff_t`` donc le role ``staff_r`` seront contraints par les catégories MCS. Testez après relogin.



??? Note "Tips"
    Vous pouvez temporairement changer votre context, notamment pour baisser en permissions, avec :

    ```bash
    sudo runcon -l s0:c0 -- bash
    ```

---

### Plages MLS

#### Activation de la policy MLS

Nous allons maintenant installer et activer la policy MLS:

```bash
dnf install selinux-policy-mls
```

Il faut faire un relabel de tout le système au boot pour activer SELinux. Changez le fichier de config SELinux pour passer en MLS et en Permissive.

Puis créez un fichier pour trigger le relabel au bot :
```bash
touch /.autorelabel
```

Redémarrez et vérifier que vous êtes bien en MLS / Permissive après.


#### Assignation d'une plage MLS


Changer de policy nous a enlevé nos configurations custom. Recréez l'user SELinux d'`alice` puis assignez lui une plage de niveaux MLS `s0-s3`.

Les plages MLS se définissent avec des `-` et les catégories des `.`, on peut aussi lister les catégories avec `,`.




??? Note "Commandes"
    ```bash
    semanage user -m -R "staff_r" -r "s0-s3:c0.c3,c8" mcs_alice
    semanage login -a -s mcs_alice alice
    ```


Créez un fichier avec un niveau supérieur :

```bash
chcon -t user_home_t -l s5 /tmp/data_topsecret
```

Testez l’accès avec `alice`. Essayez de changer de niveau avec `runcon` et d'écrire dans un fichier ayant un niveau inférieur.



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

Repassez en `targeted` avec un `/.autorelabel` et nettoyez encore.
