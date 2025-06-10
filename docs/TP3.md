## Création de modules custom 

### Objectif

Créer un module custom pour le service du premier TP.

### Étapes

#### Restaurez le (mauvais) contexte du script

Le ``chcon`` du script est temporaire, remettez le contexte original et regénérez des erreurs.

??? Note "Solution"

    ```bash
    # On peut voir la liste des fcontext avec semanage fcontext -l 
    restorecon /root/script.sh
    ll -Z /root/script.sh
    setenforce 0 # Passage permissif pour avoir toutes les erreurs et non juste la première
    systemctl restart myscript
    ```
    
#### Sugestions ``sealert`` et création de module

Utilisez ``sealert`` et ses suggestions pour créer un module custom. 

??? Note "Solution"

    Récupérer les logs ``sealert``
    
    ```bash
    journalctl | grep script.sh -A 5 | grep "lancez sealert" # Pour récupérer pile la bonne alerte et ne pas afficher tout les logs d'auditd
    sealert -l id
    ```

    ``sealert`` nous donne la commande pour générer et activer le module.

    ```bash
    mkdir module1 && cd module1
    ausearch -c "script.sh" --raw | audit2allow -M my-scriptsh
    semodule -i my-scriptsh.pp 
    ```
    On peut ouvrir le fichier ``.te`` pour voir le CIL utilisé.

#### Analyse plus poussée

Une fois le module activé, on peut relancer le service.... pour voir qu'il y a encore des erreurs !

En effet, un bug fait que le nom de la commande n'est pas toujours vu comme ``script.sh``, mais comme ``(cript.sh)``, le module est générer à partir d'un ``ausearch -c`` se basant sur le nom de commande !

Un autre ``sealert`` nous permet de voir une autre commande qui autorisera le nouveau blocage.

??? Note "Nouvelle commande"

    ```bash
    ausearch -c "(cript.sh)" --raw | audit2allow -M my-cript
    ```

En comparant les ``.te`` on remarque que les permissions accordées au type ne sont pas les mêmes. On pourrait simplement activer ce module, mais c'est un peu redondant et peu propre.

Trouvez une autre manière pour générer un seul module, enlevez le précédent et activez le nouveau.

??? Note "Tips"

    - ``semodule -l`` et ``semodule -r`` sont utiles pour trouver et supprimer des modules.
    - Vous pouvez soit utiliser une manière plus efficace d'utiliser ``ausearch``, soit écrire directement le ``.te`` nécessaire puis compiler.


??? Note "Solution"
    Avec ``ausearch``:

    On filtre d'une manière plus précise, notamment avec le ``pid`` du process ``script.sh`` : 
    
    ```bash
    ausearch -p 4569 | audit2allow -M myscriptfull
    ```

    En éditant les ``.te``:
    
    On remarque que dans le premier ``.te`` il manquait la permission ``execute``, on le rajoute dans le ``require`` et le ``allow`` puis on compile.
    
    ```bash
    checkmodule -M -m -o myscriptfull.mod myscriptfull.te
    semodule_package -o myscriptfull.pp -m myscriptfull.mod
    ```

    Une fois une des deux méthodes faites on peut activer le nouveau module unique :
    
    ```bash
    semodule -i myscriptfull.pp
    ```

Important: ce module donne les droits au type ``admin_home_t`` entier ! Il vaut mieux dans un premier temps placer les choses où il faut, cela réduit les besoins de customisation de SELinux et est plus pratique pour des raisons de standardisations.

Le FSHS (File System Hierarchy Standard) dit qu'un script d'administration local devrait être placé dans ``/usr/local/bin``, qui est un dossier de type ``bin_t`` fonctionnant.  


## Création d'un nouveau type et d'un module custom

Nous allons utiliser ``sepolicy generate`` pour générer un template de policy avec un nouveau type pour notre script.

Paquets nécessaires:

```bash
dnf install rpmbuild sepolicy
```

Commande pour générer une policy sur un template "standard init daemon":

```bash
mkdir new_pol && cd new_pol
sepolicy generate --init /root/script.sh -n script
./script.sh # Le nom est un peu mal choisi, 
# mais ce script.sh est un nouveau fichier généré par la commande
# Il permet d'installer la nouvelle policy
```

Cela va nous générer des types customs et une policy basique pour ce script.
On peut voir le contexte pour le fichier dans le ``.fc`` (File Context). Il doit être de type ``script_exec_t``.

Il nous faut l'appliquer avec ``semanage fcontext -a -t script_exec_t /root/script.sh`` et ``restorecon /root/script.sh``.

Une fois ceci fait on peut relancer le service.

Le service fonctionne et ne retourne aucune erreur.

Il est important de noter que le template générer met le type ``script_t`` (celui du process) en permissive par défaut dans le ``.te``.




