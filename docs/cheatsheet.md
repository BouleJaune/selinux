# SELinux cheatsheet

## policies

## contextes

## CIL Language
compilation
ref


Attribut groupe de types


kernel/mcs. défini macro `mcs_constraint`
/usr/share/selinux/devel/include
system/init.if => def de init_daemon_domain

flask/access_vectors : list des perm possibles 


|    Kernel        |      CIL           |
| ---------------- | ------------------ |
| *attribute*      | *typeattribute*    |
| *typeattribute*  | *typeattributeset* |
| *attribute_role* | *roleattribute*    |
| *roleattribute*  | *roleattributeset* |
| *allow*          | *allow*            |
| *allow* (role)   | *roleallow*        |
| *dominance*      | *sensitivityorder* |

Voici d’abord les réponses à tes questions, puis une **cheatsheet** des macros et attributs SELinux les plus utiles.

---

## Questions / Réponses

### 1. Rôle des fichiers `.te` et `.if`

* **`.te`** (*Type Enforcement*) contient les **règles SELinux** (déclarations de types, `allow`, `type_transition`, etc.) et les appels aux macros M4 pour générer ces règles.
* **`.if`** (*Interface File*) sert à **définir des interfaces et des macros spécifiques à un module**, réutilisables par d’autres modules. On y déclare des points d’extension (avec `interface` et `provides`) ou des petits morceaux de code M4 à inclure.

### 2. Macro vs Interface (`.if`)

* Les **macros** SELinux sont écrites en M4 et, à l’exécution, sont **expansées** en règles CIL.
* Beaucoup de ces macros sont définies dans les **fichiers `.spt`** de `/usr/share/selinux/devel/include/`.
* Les fichiers `.if` regroupent en partie ces macros ou déclarent de nouvelles **interfaces** (fonctions) que tu peux appeler depuis ton `.te`.

### 3. Attributs SELinux

* Un **attribute** est un **groupe logique** de types, permettant d’écrire des règles sur tous les types qui lui sont associés.
* On déclare un attribut :

  ```te
  attribute myattr;
  type mytype;  
  typeattribute mytype, myattr;
  ```
* On peut ensuite écrire

  ```te
  allow some_t myattr:class { perms… };
  ```

  pour autoriser **tous** les types du groupe `myattr` à effectuer certaines opérations.

### 4. Où trouver la liste des macros et attributs disponibles ?

* **Sur ta machine** (Fedora/RHEL) :

  ```bash
  grep -R "define(" /usr/share/selinux/devel/include/*.spt
  grep -R "attribute" /usr/share/selinux/devel/include/*.spt
  ```
* **Documentation upstream** sur GitHub refpolicy :

  * Macros : `policy/modules/support/*.spt`
  * Attributs : `policy/modules/support/constraints.spt`
* **Pages de manuel** (si installées) :

  ```bash
  man sepolicy-macros
  ```

---

## Cheatsheet des macros et attributs

| **Macro**                              | **But**                                                                                |
| -------------------------------------- | -------------------------------------------------------------------------------------- |
| `init_daemon_domain(domain,exec_t)`    | Configure un domaine système complet : transitions, accès logs, sockets, cgroups, etc. |
| `unconfined_domain(domain)`            | Rend le domaine aussi permissif que `unconfined_t` (pas de confinement MAC).           |
| `domain_type(domain)`                  | Marque un type comme domaine (processus) pour recevoir des règles `allow`.             |
| `domain_entry_file(domain, exec_type)` | Crée la transition auto depuis `init_t` lors de l’exécution du binaire `exec_type`.    |
| `corecmd_search_bin(domain)`           | Autorise l’accès aux binaires standard (`/bin`, `/usr/bin`) pour `domain`.             |
| `corenet_tcp_connect_ports(domain)`    | Autorise toutes les connexions TCP sortantes de `domain`.                              |
| `files_read_generic(domain)`           | Autorise la lecture de la plupart des fichiers de configuration système pour `domain`. |
| `allow_network_tcp(domain)`            | Autorise les connexions TCP sortantes de `domain` (nom / port).                        |
| `secsock_stream_connect(domain)`       | Autorise `stream_socket` connect (UNIX sockets).                                       |

| **Attribut**           | **But**                                                                 |
| ---------------------- | ----------------------------------------------------------------------- |
| `daemon_type`          | Regroupe tous les types de services système (httpd\_t, sshd\_t, etc.)   |
| `file_type`            | Regroupe les types de fichiers standards (etc\_t, var\_log\_t, tmp\_t…) |
| `netif_type`           | Regroupe les types d’interfaces réseau SELinux                          |
| `port_type`            | Tous les types de ports réseau (`http_port_t`, `ssh_port_t`…)           |
| `user_home_type`       | Types de répertoires et fichiers sous `/home/*`                         |
| `unlabeled_type`       | Objets sans label ou label par défaut                                   |
| `mcs_constrained_type` | Types qui respectent les contraintes MCS (catégories)                   |

> **Note** : cette liste est non exhaustive. Utilise les grep sur les `.spt` pour découvrir d’autres macros et attributs disponibles sur ton système.

---

Avec cette **cheatsheet**, tu as une vue d’ensemble rapide des outils M4 à ta disposition pour écrire des `.te` propres et maintenir ta politique SELinux.




Non : les fichiers `.te` (Type Enforcement) ne sont pas du CIL brut, mais utilisent un **langage à base de macros m4** qui, lors de la compilation, est transformé en CIL avant d’être compilé en binaire. Pour toi, qui écris essentiellement des `.te`, voici une **cheatsheet** ciblée sur le contenu d’un `.te` :

---

## 1. En-tête du module

```te
module monservice 1.0;
```

* **`module <nom> <version>;`** : identifie le module et sa version.

---

## 2. `require`

```te
require {
    type       unconfined_t;
    type       var_log_t;
    class      file { read write open getattr };
}
```

* **But** : déclarer tous les types, rôles, classes et permissions dont tu auras besoin.
* **Où** : souvent en tête de fichier, avant toute règle.

---

## 3. Déclaration de types & attributs

```te
type           monsvc_t;
type           monsvc_exec_t;
typeattribute  monsvc_type, daemon_type;
```

* **`type X;`** : crée un nouveau type X (domaine ou objet).
* **`typeattribute T, A;`** : associe le type T à l’attribut A (pour factoriser les règles sur plusieurs types).

---

## 4. Domain setup macros

```te
init_daemon_domain(monsvc_t, monsvc_exec_t)
```

* **`init_daemon_domain(domain, exec_type)`** :

  * Déclare `domain` comme domaine de démon
  * Crée la transition automatiquement depuis `init_t`/`systemd_t`
  * Ajoute un ensemble complet de règles courantes (accès logs, sockets, etc.)
* **Alternative plus stricte** :

  * Éviter les macros « unconfined »
  * Définir manuellement des `allow` ciblés

---

## 5. Règles d’autorisation

```te
allow monsvc_t var_log_t:file { read write open getattr };
dontaudit monsvc_t etc_t:dir { search };
```

* **`allow <src> <tgt>:<class> { perms… };`** : autorise explicitement les permissions.
* **`dontaudit`** : idem `allow` mais **n’écrit pas** de logs AVC (utile pour “nettoyer” les non-pertinents).
* **`auditallow`** : force la journalisation même si la règle existe.

---

## 6. Transitions de type

```te
type_transition init_t monsvc_exec_t:process monsvc_t;
```

* **`type_transition S T:class U;`** :
  quand un objet de type S exécute T dans la classe class,
  le nouveau processus héritera du type U.

---

## 7. File contexts (fcontext)

```te
# fichier contextuel (équivalent semanage fcontext)
file_contexts=$(cat <<EOF
/opt/monservice/bin/helloserver   gen_context(system_u, object_r, monsvc_exec_t, s0)
/opt/monservice/logs(/.*)?         gen_context(system_u, object_r, var_log_t, s0)
EOF
)
```

* En `.te` pur, on ne déclare pas de `file_context`;
  on utilise `monolithic_context` ou **on gère via semanage**.
* Les modules `.fc` séparés contiennent ces lignes.

---

## 8. Interfaces & packaging

```te
interface(`monservice_if')      # déclare un point d’extension
provides(`monservice_if')       # indique qu’on fournit cette interface
```

* **Pour partager des règles** entre modules ou respecter des dépendances.

---

## 9. Compilation résumé

```bash
checkmodule    -M -m -o monservice.mod monservice.te
semodule_package -o monservice.pp -m monservice.mod
sudo semodule  -i monservice.pp
```

---

## 10. Où trouver les macros et détails

* **Fichiers include** (Fedora/RHEL) :
  `/usr/share/selinux/devel/include/*.spt`
  `/usr/share/selinux/devel/include/support/obj_perm_sets.spt`
  `/usr/share/selinux/devel/include/services/*.te`
* **Refpolicy GitHub** :
  [https://github.com/SELinuxProject/refpolicy](https://github.com/SELinuxProject/refpolicy) (dossier `policy/modules` et `include/`)
* **Pages de manuel** :
  `man selinux-policy-devel`
  `man sepolicy`

---

> Cette **cheat-sheet** couvre l’essentiel pour **rédiger des fichiers `.te`** clairs, précis et maintenables, sans te plonger dans le CIL bas-niveau.
