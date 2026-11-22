# Mini-projet Ansible – Automatisation et configuration de serveurs Linux

## 1. Présentation du projet

### 1.1 Contexte

Ce mini-projet a pour objectif de mettre en pratique les principes de l'automatisation de l'administration des systèmes Linux avec **Ansible**.

Il consiste à mettre en place une infrastructure composée de plusieurs machines Linux, hébergées dans deux environnements différents :

- Un environnement local virtualisé avec Oracle VirtualBox et Vagrant.
- Un environnement cloud avec Amazon Web Services (AWS), à travers deux instances EC2.

Une machine de contrôle, exécutant Ansible depuis Ubuntu sous WSL2, permet d'administrer à distance les différentes machines à travers SSH.

L'objectif est de centraliser la configuration, l'installation des composants nécessaires et le déploiement d'une application web à l'aide de playbooks et de rôles Ansible.

### 1.2 Objectifs techniques

Le projet vise à :

- Installer et configurer un environnement Ansible sous Ubuntu WSL2.
- Comprendre le fonctionnement d'une architecture Ansible sans agent.
- Créer et organiser un inventaire de machines Linux.
- Configurer les connexions SSH avec authentification par clé privée.
- Mettre en place une machine virtuelle Ubuntu avec VirtualBox et Vagrant.
- Déployer et administrer des serveurs Ubuntu et Rocky Linux sur AWS EC2.
- Automatiser l'installation et la configuration de Docker.
- Déployer un serveur web Apache et une application web.
- Utiliser les variables, les groupes d'hôtes, les tâches et les handlers Ansible.
- Tester et valider les déploiements sur les différentes machines.

### 1.3 Architecture générale

L'infrastructure repose sur une architecture Ansible de type agentless : Ansible est installé uniquement sur la machine de contrôle et communique avec les machines administrées à travers SSH.

```text
                          POSTE DE TRAVAIL
                             Windows 11
                                 |
                                WSL2
                                 |
                          Ubuntu (Linux)
                                 |
                         Ansible Core 2.21.5
                                 |
                             SSH / TCP 22
                                 |
              +------------------+------------------+
              |                                     |
      ENVIRONNEMENT LOCAL                   ENVIRONNEMENT CLOUD
              |                                     |
       VirtualBox / Vagrant                        AWS
              |                                     |
        Ubuntu 24.04 VM                 +-----------+-----------+
        ubuntu-vagrant                  |                       |          
        192.168.56.101             EC2 - Ubuntu          EC2 - Rocky Linux
                                   ubuntu-server          rocky-server
                                   15.237.254.126
                                                            
```

*Les adresses publiques et privées sont des paramètres d'inventaire à actualiser lorsqu'elles changent.*

### 1.4 Technologies utilisées

| Technologie | Utilisation |
|---|---|
| Windows | Système d'exploitation du poste de travail |
| WSL2 | Environnement Linux sous Windows |
| Ubuntu | Système de la machine de contrôle |
| Ansible Core 2.21.5 | Automatisation et configuration |
| Python 3.14.4 | Environnement Python de la machine de contrôle |
| VirtualBox | Hyperviseur de virtualisation |
| Vagrant 2.4.9 | Création et gestion de la VM locale |
| Ubuntu 24.04 | Machine virtuelle locale |
| AWS EC2 | Hébergement des serveurs cloud |
| Rocky Linux | Deuxième distribution Linux cible |
| SSH | Connexion sécurisée aux machines |
| Docker | Gestion des conteneurs |
| Apache | Serveur web |

---

## 2. Configuration du poste de travail

### 2.1 Environnement Windows

Le poste de travail principal fonctionne sous Windows.

Les outils suivants ont été installés :

- Windows Subsystem for Linux (WSL2).
- Une distribution Ubuntu.
- Oracle VirtualBox.
- HashiCorp Vagrant.
- Un terminal permettant d'accéder à Windows et à WSL2.

WSL2 permet de disposer d'un environnement Linux sans devoir installer une machine virtuelle Linux dédiée uniquement à l'exécution d'Ansible.

### 2.2 Vérification de WSL2

Depuis PowerShell :

```powershell
wsl --status
```

Afficher les distributions installées :

```powershell
wsl --list --verbose
```

Vérifier que la distribution Ubuntu utilise WSL2.

### 2.3 Accès à Ubuntu sous WSL2

Depuis PowerShell :

```powershell
wsl
```

Vérifier la distribution :

```bash
cat /etc/os-release
```

Afficher la version du noyau :

```bash
uname -a
```

Mettre à jour les paquets :

```bash
sudo apt update
sudo apt upgrade -y
```

Installer les outils indispensables :

```bash
sudo apt install -y \
    openssh-client \
    curl \
    wget \
    git \
    unzip \
    python3 \
    python3-pip \
    python3-venv \
    netcat-openbsd
```

### 2.4 Création du répertoire du projet

Le projet est situé dans le système de fichiers Linux de WSL2.

Chemin utilisé :

```text
/home/micht/projects/mini-projet-ansible
```

Création du répertoire :

```bash
mkdir -p ~/projects/mini-projet-ansible
cd ~/projects/mini-projet-ansible
```

Vérifier le répertoire courant :

```bash
pwd
```

Résultat attendu :

```text
/home/micht/projects/mini-projet-ansible
```

**Important :** le projet a été déplacé du système de fichiers Windows (`/mnt/c/...`) vers le système de fichiers Linux de WSL2.

Cette modification permet notamment d'éviter les problèmes de permissions rencontrés avec Ansible lorsqu'un fichier `ansible.cfg` se trouve dans un répertoire considéré comme accessible en écriture par tous.

---

## 3. Installation et configuration d'Ansible

### 3.1 Version installée

L'environnement utilise :

- Ansible Core : 2.21.5
- Python : 3.14.4
- Jinja : 3.1.6
- PyYAML : 6.0.3

Vérifier la version :

```bash
ansible --version
```

La configuration locale doit apparaître dans la sortie :

```text
config file = /home/micht/projects/mini-projet-ansible/ansible.cfg
```

### 3.2 Vérification de la configuration

Afficher les paramètres de configuration :

```bash
ansible-config dump --only-changed
```

Afficher les paramètres disponibles :

```bash
ansible-config list
```

Ansible recherche notamment son fichier de configuration dans cet ordre :

1. Variable d'environnement `ANSIBLE_CONFIG`.
2. `ansible.cfg` dans le répertoire courant.
3. `~/.ansible.cfg`.
4. `/etc/ansible/ansible.cfg`.

Il utilise le premier fichier trouvé.

### 3.3 Fichier ansible.cfg

Le fichier `ansible.cfg` est placé à la racine du projet.

Il permet de centraliser les paramètres d'exécution d'Ansible.

Exemple de configuration à conserver ou à adapter à la configuration finale du dépôt :

```ini
[defaults]
inventory = ./host.yml
roles_path = ./roles
host_key_checking = False
retry_files_enabled = False
interpreter_python = auto_silent

[ssh_connection]
pipelining = True
```

Explications :

| Paramètre | Rôle |
|---|---|
| inventory | Définit le fichier d'inventaire par défaut |
| roles_path | Indique le répertoire contenant les rôles |
| host_key_checking | Désactive la vérification interactive des clés SSH |
| retry_files_enabled | Désactive les fichiers de reprise |
| interpreter_python | Gestion de la découverte de Python distant |
| pipelining | Optimise certaines opérations SSH |

La désactivation de `host_key_checking` a été utilisée pour simplifier les connexions pendant les tests. Pour un environnement de production, il est préférable de conserver la vérification des clés SSH.

---

## 4. Gestion des clés SSH

Ansible utilise SSH pour communiquer avec les serveurs Linux.

Deux environnements de connexion sont utilisés :

- La VM Vagrant locale.
- Les instances EC2 sur AWS.

### 4.1 Clé SSH AWS

La paire de clés AWS créée pour accéder aux instances EC2 utilise le fichier :

```text
~/.ssh/ansible-key-pair.pem
```

Vérifier la présence de la clé :

```bash
ls -l ~/.ssh/ansible-key-pair.pem
```

Restreindre les permissions :

```bash
chmod 400 ~/.ssh/ansible-key-pair.pem
```

Tester la connexion à l'instance Ubuntu :

```bash
ssh -i ~/.ssh/ansible-key-pair.pem \
    ubuntu@15.237.254.126
```

L'utilisateur SSH utilisé sur Ubuntu est `ubuntu`.

### 4.2 Clé SSH Vagrant

Une clé dédiée est référencée dans la configuration de l'hôte Vagrant :

```text
/home/micht/.ssh/vagrant_key
```

Restreindre ses permissions :

```bash
chmod 600 ~/.ssh/vagrant_key
```

Tester la connexion :

```bash
ssh -i ~/.ssh/vagrant_key \
    vagrant@192.168.56.101
```

L'utilisateur distant est `vagrant`.

### 4.3 Vérification de la configuration SSH Vagrant

Depuis le répertoire contenant le `Vagrantfile` :

```bash
vagrant ssh-config
```

Cette commande affiche les paramètres SSH générés par Vagrant, notamment :

- Le nom de l'hôte.
- L'adresse de connexion.
- Le port SSH.
- L'utilisateur.
- Le chemin de la clé privée.
- Les options SSH.

La clé privée générée automatiquement par Vagrant se trouve également dans le répertoire :

```text
.vagrant/machines/default/virtualbox/private_key
```

La clé référencée dans l'inventaire doit correspondre à la clé réellement utilisée pour se connecter à la VM.

---

## 5. VirtualBox : configuration de la virtualisation locale

### 5.1 Présentation

Oracle VirtualBox est utilisé comme hyperviseur pour exécuter une machine virtuelle Ubuntu sur le poste Windows.

Cette VM permet de tester les playbooks Ansible localement, sans dépendre systématiquement des ressources AWS.

### 5.2 Configuration logicielle

Les outils installés sont :

- Oracle VirtualBox.
- HashiCorp Vagrant.

Vérifier l'installation de Vagrant :

```bash
vagrant --version
```

Version utilisée :

```text
Vagrant 2.4.9
```

Vérifier l'installation de VirtualBox depuis PowerShell :

```powershell
VBoxManage --version
```

### 5.3 Fonctionnement réseau

La VM utilise deux mécanismes réseau :

**Réseau NAT**

Le NAT permet à la machine virtuelle d'accéder au réseau extérieur à travers la connexion de l'hôte.

Des redirections de ports ont été utilisées :

| Port hôte | Port invité | Service |
|---|---|---|
| 8081 | 80 | HTTP |
| 2222 | 22 | SSH |

Ces redirections permettent notamment d'accéder au serveur web depuis Windows et de tester une connexion SSH via le port 2222.

**Réseau privé Host-Only**

Un réseau privé VirtualBox a également été utilisé afin de permettre une communication directe entre WSL2 et la VM.

Adresse IP de la VM :

```text
192.168.56.101
```

Cette adresse est utilisée dans l'inventaire Ansible.

La configuration exacte des adaptateurs VirtualBox doit être vérifiée dans le gestionnaire VirtualBox pour documenter les noms et les adresses du réseau Host-Only effectivement utilisés.

---

## 6. Vagrant : création et configuration de la VM Ubuntu

### 6.1 Présentation

Vagrant permet de créer et de gérer automatiquement la machine virtuelle Ubuntu.

Il s'appuie sur VirtualBox comme fournisseur de virtualisation.

### 6.2 Box utilisée

La box installée est :

```text
bento/ubuntu-24.04
```

Informations relevées :

| Paramètre | Valeur |
|---|---|
| Box | bento/ubuntu-24.04 |
| Provider | VirtualBox |
| Version de box | 202510.26.0 |
| Architecture | amd64 |
| Système invité | Ubuntu 24.04 |
| Nom de la VM | ubuntu-ansible |
| Nom d'hôte invité | ubuntu-vagrant |

### 6.3 Vérification des boxes

Afficher les boxes installées :

```bash
vagrant box list
```

Ajouter la box utilisée :

```bash
vagrant box add bento/ubuntu-24.04
```

### 6.4 Création du Vagrantfile

Le fichier `Vagrantfile` définit les paramètres de la machine virtuelle.

Les paramètres fonctionnels à reproduire sont :

- Box Ubuntu 24.04 Bento.
- Provider VirtualBox.
- Adresse IP privée statique `192.168.56.101`.
- Redirection HTTP 8081 vers 80.
- Redirection SSH 2222 vers 22.

Exemple de configuration correspondant à ces paramètres :

```ruby
Vagrant.configure("2") do |config|

  config.vm.box = "bento/ubuntu-24.04"

  config.vm.hostname = "ubuntu-vagrant"

  config.vm.network "private_network",
    ip: "192.168.56.101"

  config.vm.network "forwarded_port",
    guest: 80,
    host: 8081

  config.vm.network "forwarded_port",
    guest: 22,
    host: 2222

  config.vm.provider "virtualbox" do |vb|
    vb.name = "ubuntu-ansible"

    # Les ressources CPU et RAM sont à renseigner
    # selon la configuration VirtualBox réellement utilisée.
  end

end
```

Ce bloc constitue une configuration de référence à comparer avec le `Vagrantfile` final du projet, et non une reproduction garantie de toutes ses lignes d'origine.

### 6.5 Commandes Vagrant utilisées

Démarrer la VM :

```bash
vagrant up
```

Afficher son état :

```bash
vagrant status
```

Se connecter à la VM :

```bash
vagrant ssh
```

Afficher la configuration SSH :

```bash
vagrant ssh-config
```

Arrêter la VM :

```bash
vagrant halt
```

Redémarrer la VM :

```bash
vagrant reload
```

Supprimer la VM :

```bash
vagrant destroy
```

Afficher les boxes installées :

```bash
vagrant box list
```

### 6.6 Vérification du système invité

Depuis la VM :

```bash
hostname
```

```bash
cat /etc/os-release
```

```bash
ip addr
```

Résultats attendus :

```text
Ubuntu 24.04 LTS
Hostname : ubuntu-vagrant
Adresse IP : 192.168.56.101
```

### 6.7 Tests de connectivité

Depuis Windows PowerShell :

```powershell
Test-NetConnection 127.0.0.1 -Port 2222
```

Ce test permet de vérifier l'accessibilité du port SSH redirigé.

Depuis WSL2 :

```bash
ping -c 4 192.168.56.101
```

Tester SSH :

```bash
ssh -i ~/.ssh/vagrant_key \
    vagrant@192.168.56.101
```

---

## 7. AWS : configuration de l'infrastructure cloud

### 7.1 Présentation

Amazon Web Services est utilisé pour héberger deux serveurs Linux accessibles à distance :

1. Une instance Ubuntu Server.
2. Une instance Rocky Linux.

Les instances sont créées à l'aide du service Amazon EC2.

### 7.2 Région AWS

La région sélectionnée est :

```text
Europe (Paris) - eu-west-3
```

Cette région est utilisée pour héberger les instances du projet.

### 7.3 Instances EC2

L'architecture cible est la suivante :

| Nom logique | Système | Groupe Ansible |
|---|---|---|
| ubuntu-server | Ubuntu Server 24.04 | ubuntu |
| rocky-server | Rocky Linux | rocky |

Les noms logiques correspondent aux alias utilisés dans l'inventaire Ansible.

### 7.4 Instance EC2 Ubuntu

Informations de connexion validées :

| Paramètre | Valeur |
|---|---|
| Alias Ansible | ubuntu-server |
| Distribution | Ubuntu Server 24.04.5 LTS |
| Nom d'utilisateur SSH | ubuntu |
| Adresse IPv4 publique | 15.237.254.126 |
| Adresse IPv4 privée | 172.31.5.235 |
| Port SSH | 22 |
| Clé SSH | ansible-key-pair.pem |
| Région | eu-west-3 |

Les caractéristiques exactes de l'instance (type EC2, identifiant, AMI et ressources) restent à renseigner à partir de la console AWS.

### 7.5 Instance EC2 Rocky Linux

La deuxième instance est destinée à exécuter Rocky Linux.

Alias Ansible :

```text
rocky-server
```

Groupe Ansible :

```text
rocky
```

Les informations suivantes sont à compléter depuis la console AWS :

- Version exacte de Rocky Linux.
- Identifiant de l'instance EC2.
- Type d'instance.
- Adresse IPv4 publique.
- Adresse IPv4 privée.
- Nom d'utilisateur SSH.
- Clé SSH utilisée.
- Groupe de sécurité associé.

Ne pas réutiliser automatiquement l'utilisateur `ubuntu` pour cette instance : il dépend de l'image Rocky Linux choisie.

### 7.6 Configuration du groupe de sécurité AWS

Le groupe de sécurité contrôle les flux réseau entrants et sortants des instances.

Une règle SSH IPv4 a été ajoutée pour résoudre le problème initial de connexion.

Configuration attendue :

| Type | Protocole | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Adresse IPv4 publique du poste de contrôle |
| HTTP | TCP | 80 | Selon les besoins d'accès à l'application |

La règle SSH initiale était configurée uniquement en IPv6. Cette configuration ne permettait pas à la connexion SSH IPv4 utilisée depuis WSL2 d'aboutir.

L'ajout d'une règle SSH IPv4 limitée à l'adresse IP publique du poste a résolu le problème.

**Bonne pratique de sécurité :**

Ne pas utiliser `0.0.0.0/0` pour ouvrir SSH à l'ensemble d'Internet. Autoriser uniquement l'adresse IP publique nécessaire.

### 7.7 Tests de connexion AWS

Tester l'ouverture du port 22 :

```bash
nc -vz 15.237.254.126 22
```

Tester SSH :

```bash
ssh -i ~/.ssh/ansible-key-pair.pem \
    ubuntu@15.237.254.126
```

Vérifier le système :

```bash
hostnamectl
```

```bash
cat /etc/os-release
```

```bash
python3 --version
```

Les vérifications effectuées ont confirmé :

- Ubuntu 24.04.5 LTS.
- Python 3.12 sur la machine distante.
- Connexion SSH fonctionnelle.
- Accessibilité du port TCP 22.

---

## 8. Organisation du projet Ansible

### 8.1 Répertoire principal

Chemin :

```text
/home/micht/projects/mini-projet-ansible
```

### 8.2 Structure des fichiers

La structure de travail identifiée est :

```text
mini-projet-ansible/
│
├── ansible.cfg
│
├── host.yml
│
├── playbook.yml
│
├── group_vars/
│   ├── all.yml
│   ├── ubuntu.yml
│   └── rocky.yml
│
├── host_vars/
│   ├── ubuntu-server.yml
│   ├── rocky-server.yml
│   └── ubuntu-vagrant.yml
│
└── roles/
    └── webapp/
        ├── defaults/
        ├── files/
        ├── handlers/
        │   └── main.yml
        ├── meta/
        │   └── main.yml
        ├── tasks/
        │   ├── main.yml
        │   ├── docker-ubuntu.yml
        │   └── docker-centos.yml
        ├── templates/
        │   └── index.html.j2
        └── vars/
```

Les fichiers de rôle `docker-centos.yml` et les variables historiques doivent être contrôlés afin de confirmer leur adaptation à Rocky Linux.

### 8.3 Inventaire Ansible : host.yml

Le fichier `host.yml` répertorie les machines administrées.

Les groupes configurés sont :

- `ubuntu`
- `rocky`
- `vagrant`

Les hôtes correspondants sont :

- `ubuntu-server`
- `rocky-server`
- `ubuntu-vagrant`

Exemple de structure YAML :

```yaml
all:
  children:

    ubuntu:
      hosts:
        ubuntu-server:

    rocky:
      hosts:
        rocky-server:

    vagrant:
      hosts:
        ubuntu-vagrant:
```

**Important :** le groupe `vagrant` doit être placé sous `children`, au même niveau que les groupes `ubuntu` et `rocky`.

Une erreur de structure avait précédemment provoqué le message :

```text
Skipping unexpected key (vagrant) in group (all)
```

Cette erreur est corrigée en respectant la hiérarchie YAML de l'inventaire.

### 8.4 Variables communes : group_vars/all.yml

Ce fichier centralise les paramètres communs à l'ensemble des machines.

Une configuration SSH utilisée pendant les tests est :

```yaml
ansible_ssh_extra_args: '-o StrictHostKeyChecking=no'
```

Cette option évite les demandes interactives de confirmation de clés SSH pendant les connexions.

Elle doit être remplacée par une gestion sécurisée des clés d'hôtes dans un environnement de production.

### 8.5 Variables du groupe Ubuntu

Fichier :

```text
group_vars/ubuntu.yml
```

Il contient les paramètres communs aux machines Ubuntu.

Les paramètres de connexion spécifiques à chaque instance doivent être définis dans les fichiers `host_vars`.

### 8.6 Variables du groupe Rocky

Fichier :

```text
group_vars/rocky.yml
```

Ce fichier regroupe les variables communes aux serveurs Rocky Linux.

Il permet notamment d'adapter les tâches aux spécificités de cette distribution.

### 8.7 Variables spécifiques aux hôtes

Les paramètres individuels sont organisés dans `host_vars`.

Exemple pour Ubuntu AWS :

```text
host_vars/ubuntu-server.yml
```

Exemple pour Rocky AWS :

```text
host_vars/rocky-server.yml
```

Exemple pour Vagrant :

```text
host_vars/ubuntu-vagrant.yml
```

La configuration Vagrant identifiée comprend :

```yaml
ansible_host: 192.168.56.101
ansible_user: vagrant
ansible_ssh_private_key_file: /home/micht/.ssh/vagrant_key
ansible_ssh_extra_args: '-o StrictHostKeyChecking=no'
ansible_become: false
```

Pour Ubuntu AWS, les paramètres de connexion connus sont :

```yaml
ansible_host: 15.237.254.126
ansible_user: ubuntu
ansible_ssh_private_key_file: ~/.ssh/ansible-key-pair.pem
ansible_become: true
```

Ces extraits sont des références de configuration. Ils doivent être comparés aux fichiers réellement présents dans le dépôt avant de remplacer leur contenu.

---

## 9. Création et utilisation des rôles Ansible

### 9.1 Présentation

Le projet utilise un rôle nommé :

```text
webapp
```

Un rôle Ansible permet de regrouper les tâches, les variables, les fichiers, les templates et les handlers liés à une fonctionnalité.

Dans ce projet, le rôle `webapp` centralise les opérations nécessaires au déploiement de l'application web et à la gestion de ses composants.

### 9.2 Répertoire tasks

Le répertoire :

```text
roles/webapp/tasks/
```

contient les tâches du rôle.

Fichiers identifiés :

- `main.yml`
- `docker-ubuntu.yml`
- `docker-centos.yml`

Le fichier `main.yml` constitue le point d'entrée des tâches.

Les fichiers spécifiques aux distributions permettent de séparer les opérations lorsque les commandes d'installation diffèrent.

**Point d'attention :** Rocky Linux est une distribution de la famille Red Hat. Il faut donc vérifier que les tâches historiquement nommées `docker-centos.yml` utilisent des commandes et des paquets compatibles avec la version Rocky ciblée.

### 9.3 Répertoire handlers

Le fichier :

```text
roles/webapp/handlers/main.yml
```

contient notamment les handlers suivants :

- `restart docker`
- `restart webapp`

Leur rôle est de redémarrer les services concernés lorsqu'une tâche ayant modifié leur configuration le demande.

Exemple de principe :

```yaml
handlers:
  - name: restart docker
    ansible.builtin.service:
      name: docker
      state: restarted

  - name: restart webapp
    ansible.builtin.service:
      name: webapp
      state: restarted
```

Cet exemple illustre le fonctionnement attendu. Le service `webapp` doit réellement exister sur la machine cible pour que son redémarrage soit valide.

### 9.4 Répertoire templates

Le template identifié est :

```text
roles/webapp/templates/index.html.j2
```

Il permet de générer une page HTML à partir d'un modèle Jinja2.

Cette approche permet de personnaliser le contenu de la page en fonction des variables définies dans Ansible.

### 9.5 Répertoire defaults

```text
roles/webapp/defaults/
```

Ce répertoire est destiné aux variables par défaut du rôle.

### 9.6 Répertoire files

```text
roles/webapp/files/
```

Il contient les fichiers statiques pouvant être copiés sur les machines administrées.

### 9.7 Répertoire meta

```text
roles/webapp/meta/
```

Il contient les métadonnées du rôle, notamment les informations de dépendance éventuelles.

---

## 10. Playbook principal

### 10.1 Fichier playbook.yml

Le fichier :

```text
playbook.yml
```

est le point d'entrée de l'automatisation.

Il orchestre l'exécution des tâches et l'utilisation des rôles sur les hôtes ciblés.

Le rôle `webapp` est utilisé pour automatiser le déploiement.

Les détails exacts des plays, des conditions et des tâches doivent être repris du fichier final du dépôt afin de ne pas présenter une configuration théorique comme étant celle effectivement exécutée.

### 10.2 Vérifier la syntaxe

Avant l'exécution :

```bash
ansible-playbook playbook.yml --syntax-check
```

### 10.3 Lister les hôtes ciblés

```bash
ansible-playbook playbook.yml --list-hosts
```

### 10.4 Exécuter sur la VM Vagrant uniquement

Commande utilisée avec succès :

```bash
ansible-playbook playbook.yml --limit ubuntu-vagrant
```

Résultat enregistré :

```text
ok=11
changed=6
unreachable=0
failed=0
skipped=1
```

Ce résultat confirme que le playbook s'est exécuté sans échec sur la VM Vagrant.

Il indique également que six tâches ont entraîné des changements sur la machine.

### 10.5 Exécuter sur Ubuntu AWS

```bash
ansible-playbook playbook.yml --limit ubuntu-server
```

### 10.6 Exécuter sur Rocky AWS

```bash
ansible-playbook playbook.yml --limit rocky-server
```

### 10.7 Exécuter sur plusieurs machines

Exécuter sur tous les hôtes :

```bash
ansible-playbook playbook.yml
```

Exécuter uniquement sur les deux serveurs AWS :

```bash
ansible-playbook playbook.yml \
    --limit 'ubuntu:rocky'
```

---

## 11. Tests et commandes Ansible

### 11.1 Vérification de l'inventaire

Afficher l'inventaire sous forme d'arbre :

```bash
ansible-inventory --graph
```

Afficher l'inventaire détaillé :

```bash
ansible-inventory --list
```

Afficher les variables d'un hôte :

```bash
ansible-inventory --host ubuntu-vagrant
```

```bash
ansible-inventory --host ubuntu-server
```

### 11.2 Test de connectivité

Tester tous les hôtes :

```bash
ansible all -m ping
```

Tester uniquement Ubuntu :

```bash
ansible ubuntu -m ping
```

Tester Rocky :

```bash
ansible rocky -m ping
```

Tester Vagrant :

```bash
ansible vagrant -m ping
```

Le module `ping` permet de vérifier la communication avec la machine distante et le bon fonctionnement de Python pour les modules Ansible.

### 11.3 Vérification de l'utilisateur distant

Sans élévation de privilèges :

```bash
ansible ubuntu-vagrant \
    -m command \
    -a "whoami"
```

Avec élévation de privilèges :

```bash
ansible ubuntu-vagrant \
    -m command \
    -a "whoami" \
    -b
```

Sur une machine configurée pour utiliser `become`, le résultat attendu est :

```text
root
```

Sur la VM Vagrant, la variable `ansible_become: false` a été utilisée pendant les tests. Il faut donc vérifier la configuration effective avant d'attendre un résultat `root`.

### 11.4 Vérification des informations système

```bash
ansible all -m setup
```

Afficher uniquement les informations relatives au système :

```bash
ansible ubuntu -m setup \
    -a "filter=ansible_distribution*"
```

### 11.5 Vérifier les services

```bash
ansible ubuntu -m command \
    -a "systemctl status docker"
```

```bash
ansible ubuntu -m command \
    -a "systemctl status apache2"
```

Pour Rocky Linux, le nom du service Apache est généralement `httpd` :

```bash
ansible rocky -m command \
    -a "systemctl status httpd"
```

---

## 12. Dépannage et problèmes rencontrés

Cette section reprend les principaux problèmes rencontrés pendant la réalisation du projet et les solutions appliquées.

### 12.1 Ansible ignore ansible.cfg

Erreur rencontrée :

```text
Ansible is being run in a world writable directory
```

Cause :

Le projet était initialement situé dans un répertoire Windows monté sous WSL2 :

```text
/mnt/c/Users/micht/OneDrive/Documents/Projets DevOps/mini-projet-ansible
```

Ansible ignorait le fichier de configuration local pour des raisons de sécurité.

Solution :

Déplacer le projet dans le système de fichiers Linux :

```bash
mkdir -p ~/projects
```

Puis travailler dans :

```bash
cd ~/projects/mini-projet-ansible
```

Vérifier :

```bash
ansible --version
```

### 12.2 Inventaire mal structuré

Erreur :

```text
Skipping unexpected key (vagrant) in group (all)
```

Cause :

Le groupe `vagrant` était placé au mauvais niveau dans la hiérarchie YAML.

Solution :

Placer les trois groupes sous `all.children`.

Vérifier :

```bash
ansible-inventory --graph
```

### 12.3 Problème de permissions de clé SSH

Erreur :

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
```

Cause :

La clé privée avait des permissions trop ouvertes.

Solution :

```bash
chmod 600 ~/.ssh/vagrant_key
```

Pour la clé AWS :

```bash
chmod 400 ~/.ssh/ansible-key-pair.pem
```

### 12.4 Connexion SSH à AWS en timeout

Erreur :

```text
ssh: connect to host 15.237.254.126 port 22:
Connection timed out
```

Causes identifiées :

- L'instance EC2 était démarrée.
- Les contrôles de statut AWS étaient réussis.
- Le groupe de sécurité autorisait initialement SSH uniquement en IPv6.
- La connexion utilisée depuis WSL2 était en IPv4.

Solution :

Ajouter une règle entrante SSH :

```text
Type : SSH
Protocole : TCP
Port : 22
Source : adresse IPv4 publique du poste de contrôle
```

Tester ensuite :

```bash
nc -vz 15.237.254.126 22
```

Puis :

```bash
ssh -i ~/.ssh/ansible-key-pair.pem \
    ubuntu@15.237.254.126
```

La connexion SSH a ensuite fonctionné.

### 12.5 Problèmes de réseau Vagrant

Des difficultés de connexion ont été rencontrées avec le port NAT `2222` et certaines adresses de passerelle WSL2.

Le choix d'une adresse privée statique pour la VM a permis de travailler avec une adresse stable dans l'inventaire :

```text
192.168.56.101
```

Il faut toutefois distinguer les tests réalisés depuis Windows et ceux réalisés depuis WSL2 : une redirection NAT accessible depuis Windows n'est pas nécessairement accessible de la même manière depuis WSL2.

### 12.6 Interpréteur Python distant

Un avertissement de découverte automatique de Python a été observé sur Ubuntu :

```text
/usr/bin/python3.12
```

Ce comportement est normal lorsque Ansible découvre automatiquement l'interpréteur Python installé sur le serveur distant.

---

## 13. Résultats obtenus

### 13.1 Poste de contrôle

- Ubuntu sous WSL2 opérationnel.
- Ansible Core 2.21.5 installé.
- Fichier `ansible.cfg` reconnu.
- Inventaire organisé en groupes et hôtes.

### 13.2 Environnement local

- VirtualBox installé.
- Vagrant 2.4.9 opérationnel.
- Box Ubuntu 24.04 installée.
- VM `ubuntu-vagrant` créée.
- Adresse IP privée `192.168.56.101` configurée.
- Connexion SSH et tests Ansible réalisés.

### 13.3 Environnement AWS

- Région Paris sélectionnée.
- Instance Ubuntu démarrée.
- Contrôles de santé AWS validés.
- Connexion SSH IPv4 rétablie.
- Connexion Ansible validée avec `ansible ubuntu -m ping`.
- Élévation de privilèges vérifiée.

### 13.4 Déploiement applicatif

Le playbook principal a été exécuté sur la VM Vagrant avec le résultat :

```text
PLAY RECAP

ok=11
changed=6
unreachable=0
failed=0
skipped=1
```

Docker et Apache font partie des composants déployés dans le cadre du projet.

---

## 14. Sécurité et bonnes pratiques

Quelques règles doivent être respectées pour conserver un environnement propre et sécurisé.

### 14.1 Protection des clés privées

Ne jamais publier les clés privées SSH dans un dépôt Git.

Ajouter au `.gitignore` :

```gitignore
*.pem
*.key
.vagrant/
*.retry
```

Les permissions des clés doivent être suffisamment restrictives.

### 14.2 Gestion des mots de passe

Éviter de stocker les mots de passe directement dans :

- `host_vars`
- `group_vars`
- `ansible.cfg`
- `playbook.yml`

Utiliser Ansible Vault pour les informations sensibles.

Exemple :

```bash
ansible-vault create group_vars/all/vault.yml
```

Modifier un fichier chiffré :

```bash
ansible-vault edit group_vars/all/vault.yml
```

Exécuter un playbook avec Vault :

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

### 14.3 Sécurité réseau AWS

- Limiter l'accès SSH aux adresses IP autorisées.
- Ne pas exposer inutilement les ports des instances.
- Utiliser des clés SSH dédiées.
- Vérifier régulièrement les règles des groupes de sécurité.

### 14.4 Gestion des privilèges

L'utilisation de `become: true` doit être limitée aux opérations nécessitant les droits administrateur.

Éviter de donner systématiquement les privilèges root à toutes les opérations.

---

## 15. Commandes essentielles : aide-mémoire

### WSL2

```powershell
wsl --status
wsl --list --verbose
wsl
```

### Ansible

```bash
ansible --version
ansible-config dump --only-changed
ansible-inventory --graph
ansible-inventory --list
ansible all -m ping
ansible ubuntu -m ping
ansible-playbook playbook.yml --syntax-check
ansible-playbook playbook.yml --list-hosts
ansible-playbook playbook.yml --limit ubuntu-vagrant
ansible-playbook playbook.yml --limit ubuntu-server
ansible-playbook playbook.yml --limit rocky-server
```

### SSH

```bash
chmod 400 ~/.ssh/ansible-key-pair.pem
chmod 600 ~/.ssh/vagrant_key

ssh -i ~/.ssh/ansible-key-pair.pem \
    ubuntu@15.237.254.126

ssh -i ~/.ssh/vagrant_key \
    vagrant@192.168.56.101
```

### Vagrant

```bash
vagrant --version
vagrant box list
vagrant up
vagrant status
vagrant ssh
vagrant ssh-config
vagrant halt
vagrant reload
vagrant destroy
```

### Réseau

```bash
ip addr
ip route
ping -c 4 192.168.56.101
nc -vz 15.237.254.126 22
```

### AWS

Vérifier les instances dans la console EC2 et contrôler :

- L'état de l'instance.
- Les contrôles de statut.
- Les adresses IPv4.
- Les groupes de sécurité.
- Les clés SSH.

---

## 16. Améliorations et travaux complémentaires

Les prochaines étapes du projet consistent à :

1. Finaliser la configuration de l'instance Rocky Linux.
2. Valider la connexion SSH depuis WSL2 vers Rocky.
3. Valider les tests Ansible sur les deux instances EC2.
4. Vérifier l'adaptation des tâches Docker à Ubuntu et Rocky Linux.
5. Finaliser les handlers de redémarrage des services.
6. Vérifier le déploiement de l'application web sur les différentes machines.
7. Tester l'accès HTTP aux applications déployées.
8. Sécuriser les variables sensibles avec Ansible Vault.
9. Vérifier la reproductibilité du déploiement sur toutes les machines.
10. Ajouter des captures d'écran des configurations VirtualBox, Vagrant et AWS pour compléter la documentation.

---

## 17. Conclusion

Ce mini-projet permet de mettre en œuvre une infrastructure d'administration Linux centralisée avec Ansible.

Il combine la virtualisation locale avec VirtualBox et Vagrant et l'hébergement cloud avec AWS EC2.

La machine Ubuntu sous WSL2 joue le rôle de poste de contrôle et permet d'administrer les différents serveurs à travers SSH.

L'utilisation des inventaires, des variables, des rôles, des playbooks et des handlers permet de structurer l'automatisation et de faciliter la maintenance des machines.

Le projet constitue ainsi une première mise en pratique des principes d'Infrastructure as Code (IaC), d'automatisation DevOps et d'administration de serveurs Linux.

---

## 18. Documentation officielle

- [Ansible – Documentation officielle](https://docs.ansible.com/)
- [Ansible – Configuration](https://docs.ansible.com/projects/ansible/latest/reference_appendices/config.html)
- [Ansible – Inventaires](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [Ansible – Playbooks](https://docs.ansible.com/projects/ansible/latest/playbook_guide/index.html)
- [Ansible – Rôles](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [HashiCorp Vagrant](https://developer.hashicorp.com/vagrant/docs)
- [Vagrant – VirtualBox](https://developer.hashicorp.com/vagrant/docs/providers/virtualbox)
- [Oracle VirtualBox](https://www.virtualbox.org/manual/)
- [AWS – Documentation EC2](https://docs.aws.amazon.com/ec2/)
- [Ubuntu Server Documentation](https://documentation.ubuntu.com/server/)
- [Rocky Linux Documentation](https://docs.rockylinux.org/)

---

**Auteur :** Projet personnel de mise en pratique DevOps / Ansible

**Environnement :** Windows + WSL2 + Ubuntu + VirtualBox + Vagrant + AWS EC2

**Répertoire du projet :** `~/projects/mini-projet-ansible`
