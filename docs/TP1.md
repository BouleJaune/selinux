
# Introduction à SELinux et Analyse des blocages

## Découverte et état de SELinux
### Objectifs

* Comprendre le fonctionnement général de SELinux
* Identifier et interpréter les contextes SELinux
* Manipuler les outils de diagnostic des blocages (AVC)
* Résoudre des problèmes classiques liés à la politique de sécurité

---

### Prérequis

* VM Fedora Server avec SELinux
* Accès root
* Paquets utiles installés : `policycoreutils`, `policycoreutils-python-utils`, `setroubleshoot`, `audit`, `selinux-policy-devel`

---


### Vérification de l’état SELinux

Utilisez des commandes CLI pour voir l'état de SELinux et voir quelle elle la policy actuellement chargée.

Vous pouvez aussi utiliser une commande renvoyant un code de retour 1 ou 0 pour voir l'état de SELinux, utile dans les scripts.

??? Note "Commandes"
    ```bash
    getenforce
    sestatus
    selinuxenabled && echo "SELinux est activé"
    ```


### Observation des contextes

Lister les contextes de : fichiers, processes, ports, utilisateur...

??? Note "Commandes"
    ```bash
    ls -Z /
    ps -eZ 
    semanage port -l
    id -Z
    ```

---

### Mise en évidence d’un blocage SELinux

#### Mise en place d’un blocage SELinux volontaire

Nous allons créer un service systemd qui exécute un script simple pour générer des erreurs SELinux.

Le script: 
```bash
echo '#! /bin/bash' > ~/script.sh
echo 'echo "Run OK" >> /tmp/log.txt' >> ~/script.sh
chmod +x ~/script.sh
```

Le service:
```bash
cat <<EOF | sudo tee /etc/systemd/system/myscript.service
[Unit]
Description=Test SELinux Script

[Service]
ExecStart=/root/script.sh

[Install]
WantedBy=multi-user.target
EOF
```

Activation du service:
```bash
systemctl daemon-reload
systemctl start myscript.service
```

Le service devrait être bloqué par SELinux.

### Analyse des messages AVC

#### Lecture brute des logs

Retrouvez ces blocages dans les logs du système de différentes manières et identifiez la source, cible et action interdite.


??? Note "Commandes"
    ```bash
    less /var/log/audit/audit.log
    ausearch -m AVC -ts recent
    ```

### Réitérer en ``Permissive``

Relancez le service en ``Permissive`` et observez les erreurs.
??? Note "Commandes"
    ```bash
    setenforce 0 # Met en permissive
    systemctl restart myscript.service
    cat /tmp/log.txt # Observez que le script s'éxécute bien maintenant
    ausearch -m AVC -ts recent # De multiples erreurs sont visibles
    ```

??? Note "Les différentes erreurs visibles en Permissive"
    Il y a maintenant plusieurs erreurs visibles dans les logs car le script ne s'arrête pas à la première en Permissive.

    On peut voir les diverses erreurs AVC, néanmoins c'est peu lisible.

#### Interface plus simple : sealert

Utilisez SELinux pour avoir une vue plus lisible des erreurs en Permissive

??? Note "Commandes"
    ```bash
    sealert -a /var/log/audit/audit.log
    ```

### Correction du problème

Un script éxécuté par un service est en général dans le dossier ``/usr/bin``, utilisez cette information pour trouver quel contexte mettre à ``script.sh`` et corriger le problème. 


??? Note "Solution"
    ```bash
    # Observer le contexte des fichiers /usr/bin 
    ls -lZ /usr/bin | awk 'NR>0 { print $5}' | sort | uniq -c | sort -k1 -n -r | head 
    # Le type d'un binaire est bin_t, assignons le à myscript.sh
    chcon -t bin_t /root/myscript.sh
    systemctl restart myscript # Fonctionne !
    ```
    Vous pouvez voir ce que ``bin_t`` peut faire avec ``sesearch -s bin_t -A``

    ```bash
    allow bin_t bin_t:dir { getattr open search };
    allow bin_t bin_t:filesystem associate;
    allow bin_t bin_t:lnk_file { getattr read };
    allow bin_t device_t:filesystem associate;
    allow file_type fs_t:filesystem associate;
    allow file_type hugetlbfs_t:filesystem associate;
    allow file_type noxattrfs:filesystem associate;
    allow file_type tmp_t:filesystem associate;
    allow file_type tmpfs_t:filesystem associate;
    ```
    Cela se lit par exemple : Autorise à ``bin_t`` de { getattr open search } dans les dossiers de type ``bin_t`` (première ligne)

