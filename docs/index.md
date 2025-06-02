# Comprendre SELinux et savoir modifier la politique de sécurité
## Objectifs pédagogiques :

* Approfondir la compréhension du fonctionnement de SELinux
* Maîtriser les outils de gestion et d’analyse des politiques SELinux
* Savoir écrire, modifier et déployer des modules de politique personnalisée
* Comprendre l’usage de SELinux dans les environnements à haute sécurité (MLS/MCS)
* Appliquer SELinux efficacement dans des environnements conteneurisés

## Programme :

### Introduction à SELinux

* Rappel du système de droits classique Linux et mécanisme MAC
* Présentation du contexte de sécurité : `user:role:type:level`
* Modes de fonctionnement et politiques disponibles (`targeted`, `mls`, `strict`)
* États de fonctionnement (`getenforce`, `setenforce`, `sestatus`, `selinuxenabled`)

### Diagnostic et analyse des blocages SELinux

* Lecture et compréhension des messages AVC
* Utilisation des outils `ausearch`, `audit2why`, `audit2allow`
* Présentation de `sealert` et du service `setroubleshoot`
* Mise en évidence de cas pratiques de blocage et résolution

### Gestion avancée de la politique SELinux

* Exploration d’une politique : `seinfo`, `sesearch`, `apol`
* Commandes : `restorecon`, `setfiles`, `fixfiles`, `chcon`, `semanage`
* Gestion des booléens : `getsebool`, `setsebool`

### Création et modification de modules de politique personnalisée

* Structure des fichiers
* Compilation et déploiement de modules
* Écriture de règles personnalisées
* Création de modules simples et modules plus complexes
* Bonnes pratiques de maintenance des modules personnalisés

### SELinux et MCS/MLS

* Présentation de MCS (Multi Category Security) et MLS (Multi Level Security)
* Gestion des niveaux de sécurité et catégories MCS
* Cas d’usage et application concrète de l’isolement via MCS/MLS

### SELinux et conteneurs (Podman / rootless)

* Fonctionnement de SELinux avec des conteneurs OCI (``container_t``, policy ``container-selinux``)
* Impact des options `:z` et `:Z` sur les contextes des volumes
* Utilisation de `--security-opt label=...` pour gérer les contextes
* Policy custom pour container (Udica)
* Cas pratiques 
