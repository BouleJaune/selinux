# SELinux cheatsheet

## État de SELinux

Voir l'état de SELinux :

```bash
getenforce
sestatus
cat /etc/selinux/config
```

## Contextes

Voir les contextes de différents objets : 

```bash
netstat -Z
id -Z
ps -Z
ls -Z
```

Changer de contexte :

```bash
chcon contexte fichier # Change un contexte
semanage fcontext # Règles de contextes en dur (regex possible) 
restorecon # Remet le contexte par défaut selon les règles
```

## Logs

Présents dans `/var/log/messages`/`journalctl`.

Si `auditd` actif, alors aussi dans `/var/log/audit/audit.log`.

Le paquet `setroubleshoot` fourni des outils pour analyser les logs, notamment `sealert`.

Les alertes sont des "AVC denials" (access vector cache).

- Filtre les events auditd par type de message AVC

```bash
ausearch -m AVC 
```

- Donne plus de détails et des suggestions de résolutions sur les erreurs

```bash
sealert -a /var/log/audit/audit.log 
sealert -l "ID_ALERTE"
```

- Donne le même type de logs que sealert et aussi la commande pour pour afficher cette alerte avec sealert ( | grep sealert)

```bash
journalctl -u setroubleshootd 
systemctl status setroubleshootd 
```

## Résolutions

Dans un premier temps observer la pertinence des suggestions de résolution ``sealert``.

#### FSHS

Bien penser son application selon les standards, mettre des fichiers binaires dans le dossier binaire, les confs dans `/etc` etc ...

Spec du FSHS (file system hierarchy standard) décrivant les conventions et la structure de l'arborescence des fichiers Linux: [FHS](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)

#### Contextes

S'inspirer des contextes par défaut d'une application pour les mettre sur notre configuration non-standard (ex: contexte du dossier par défaut du contenu apache)

#### Booléens

Les booléens sont des règles activables/désactivables concernant des configurations assez courantes de services.

On peut les afficher et voir si ce que l'on veut pour notre application existe.

```bash
getsebool -a | grep process
setsebool boolean 0 ou 1
```

Afficher ce que font les booléens:

```bash
sepolicy booleans -a
```


#### | audit2allow

On peut générer des règles directement à partir des logs d'alertes en pipant dans `audit2allow`.

```bash
ausearch -m AVC -p PID | audit2allow -M monmodule_local
semodule -i monmodule_local.pp
```

C'est une bonne idée de différencier nos modules customs notamment en rajoutant ``_local`` dans leur nom.

Point d'attention sur le fait de bien filtrer les bonnes alertes avec `ausarch`. On peut filtre par `pid`, `ppid`, nom de commande etc ...

Il faut penser à vérifier ce que le module fait en lisant notamment le `.te` pour voir si c'est pertinent.


#### Génération de templates de modules

On peut générer des templates de modules avec ``sepolicy generate``.

Il y a plusieurs types de templates, fournissant une structure de base pour notre module.

Cela va notamment nous générer des types customs pour notre application.

Génère un template de type "Standard Init Daemon":
```bash
sepolicy generate --init /root/script.sh -n script
```

Voir `man sepolicy generate` pour une description des différents templates.

Par défaut le module sera en permissive dans le `.te` !


## Language

### Fichiers

- `.te` contient les règles SELinux

- `.if` définit des interfaces et macros spécifiques pour pouvoir les réutiliser, et notamment des transitions de domaines fichier`=>`process

- `.fc` défini des contextes par défaut pour des fichiers


### Macros et attributs

Les macros et attributs permettent de donner des droits de manières factorisées.

Une macro est comme une fonction dans un autre langage, permettant d'appliquer différentes choses sur les éléments en entrée de la macro.

Un attribut représente un groupe de types, on peut assigner un type à un attribut pour y rattacher les permissions de l'attribut.

Les sources des attributs et macros sont visibles dans le dossier `/usr/share/selinux/devel/include`.

On peut aussi avoir des exemples sur : [refPolicy](https://github.com/SELinuxProject/refpolicy/tree/master)

- Liste des attributs: 

```bash
seinfo -a -x
```

- Liste des macros avec explication :

```bash
sepolicy interface -vl
```
Cela ne liste pas toutes les macros possibles









### Structure `.te`
- Entête

```bash
module monservice 1.0;
```

Une macro est aussi disponible : 

```bash
policy_module(monservice, 1.0)
```

La macro va notamment prendre en require tout les class de permissions.


---

- `require`

La keyword `require` permet de récupérer des variables existantes à utiliser dans le reste du fichier.

```bash
require {
    type       unconfined_t;
    type       var_log_t;
    class      file { read write open getattr };
}
```

Il existe aussi une macro `gen_require()`

### Déclaration de types & attributs

On peut déclarer des nouveaux types, attributs et typeattribute (liens entre un type et un attribut.

```bash
type           montype;
attribute           monattr;
typeattribute  montype monattr;
```

### Règles 


Une règle est de la forme : 

```bash
regle source_type target_type : class perms;
```

Les types de règles possibles sont : 

- `allow`: autorise l'action

- `dontaudit`: ne log pas l'action (en cas de refus attendu)

- `auditallow` : log l'action même si autorisé (n'`allow` pas)

Exemples:

```bash
allow monapp_t var_log_t:file { read write open getattr };
dontaudit monapp_t etc_t:dir { search };
```


À la place du type on peut mettre un attribut.

### Constrain

On peut rajouter des restrictions avec le keyword `constrain`:

```bash
constrain class perms expression_logique
```

Exemple tiré de [refPolicy](https://github.com/SELinuxProject/refpolicy/blob/master/policy/constraints):

```bash
constrain socket_class_set { create relabelto relabelfrom }
(
	u1 == u2
	or t1 == can_change_object_identity
);
```

On peut faire des expressions logiques pour mieux cerner ces contraintes.

Ici `u1` représente l'user source et `u2` l'user target.
`t1` le type source.

On voit ici que la contrainte s'applique à un attribut représentant les classes de socket.

### Classes d'objets et permissions

On peut définir des nouvelles classes et permissions mais totalement hors scope.

#### Classes de fichiers

```bash
filesystem, dir, file, lnk_file, fifo_file
```

#### Classes d'objets réseau

```bash
socket, tcp_socket, udp_socket, rawip_socket, unix_stream_socket, netif
```

#### Classe de process

Simplement .... `process`



### Permissions

#### Permissions sur fichiers
```bash
append, create, execute, getattr, ioctl, link, read, rename, write
```

#### Permissions sur les sockets
```bash
accept, append, bind, connect, create, ioctl, read, write
```



## Compilation

```bash
checkmodule    -M -m -o monservice.mod monservice.te
semodule_package -o monservice.pp -m monservice.mod
semodule  -i monservice.pp
```
