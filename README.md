# Atelier Kubernetes - veille technologique

Ce rapport détaille les étapes nécessaires pour déployer un cluster Kubernetes sur des instances Ubuntu. Kubernetes, souvent abrégé en "K8s", est un système d'orchestration de conteneurs qui automatise le déploiement, la mise à l'échelle et la gestion d'applications conteneurisées dans un environnement de cluster.

L'objectif de ce laboratoire est de configurer un cluster Kubernetes comprenant un nœud contrôleur et deux nœuds de travail. Le nœud contrôleur est responsable de la gestion globale du cluster, tandis que les nœuds de travail exécutent les applications conteneurisées.

Le nœud contrôleur héberge plusieurs services essentiels, notamment `kube-apiserver` pour accéder au plan de contrôle de Kubernetes, `etcd` pour le stockage des données du cluster, et `kube-scheduler` pour la planification des pods. Les nœuds de travail exécutent les conteneurs d'application et sont équipés du `kubelet` pour gérer les pods et du `kube-proxy` pour gérer les règles de réseau.

Ce rapport fournira un guide étape par étape pour installer Docker, les composants Kubernetes (`kubeadm`, `kubelet`, `kubectl`), initialiser le cluster Kubernetes, configurer le réseau pod, et ajouter les nœuds de travail au cluster.

## Vérification des Prérequis : Création des Machines Virtuelles Ubuntu

Pour commencer le déploiement d'un cluster Kubernetes, nous devons créer trois machines virtuelles (VM) Ubuntu en respectant les spécifications requises. Chaque instance doit disposer d'au moins 2 CPU et 2 Go de RAM, ainsi qu'une connectivité réseau complète entre toutes les instances pour permettre la communication au sein du cluster.

### Étapes pour Créer les VM Ubuntu :

1. **Choix de la Plateforme de Virtualisation**

   Utilisez une plateforme de virtualisation comme VirtualBox, VMware, ou un service de cloud comme AWS EC2, Google Cloud Compute Engine, ou Microsoft Azure pour créer les machines virtuelles Ubuntu.

2. **Configuration des VM**

   - Créez trois instances Ubuntu (par exemple, Ubuntu 20.04 LTS) avec les spécifications suivantes :
     - **CPU :** Au moins 2 CPU
     - **RAM :** Au moins 2 Go de RAM
     - **Stockage :** Assurez-vous que chaque VM dispose d'un espace de stockage suffisant pour les installations et les opérations de cluster.
     - **Réseau :** Activez la connectivité réseau complète entre toutes les instances.

3. **Installation d'Ubuntu**

   - Démarrez chaque VM avec l'image ISO d'Ubuntu.
   - Suivez les instructions d'installation pour configurer Ubuntu sur chaque instance. Assurez-vous d'attribuer les ressources (CPU, RAM) selon les spécifications requises.

4. **Configuration du Réseau**

   - Configurez le réseau pour permettre une connectivité complète entre les VM. Utilisez des adresses IP statiques ou DHCP selon votre environnement.

5. **Installation des Outils Requis**

   - Une fois les VM créées et Ubuntu installé, assurez-vous d'installer les outils requis tels que `openssh-server` pour accéder aux VM en utilisant SSH.

6. **Vérification des Spécifications**

   - Vérifiez que chaque VM répond aux exigences minimales :
     - Exécutez la commande `lscpu` pour vérifier le nombre de CPU.
     - Utilisez `free -h` pour vérifier la quantité de RAM disponible.

Après avoir créé les trois machines virtuelles Ubuntu et vérifié qu'elles répondent aux spécifications requises en termes de CPU, RAM et connectivité réseau, vous êtes prêt à passer à l'étape suivante : l'installation des composants nécessaires pour configurer le cluster Kubernetes sur ces instances.

Cette étape de préparation garantit que votre environnement est prêt à accueillir le cluster Kubernetes avec les performances et les capacités nécessaires pour exécuter les charges de travail de manière efficace et fiable.

## Définition des Noms d'Hôtes (Toutes les VM)

Pour assurer une identification claire et unique de chaque instance dans le cluster Kubernetes, nous allons définir des noms d'hôtes spécifiques sur toutes les machines virtuelles (VM).

#### Étapes pour Définir les Noms d'Hôtes :

1. **Accès aux Machines Virtuelles**

   Assurez-vous d'avoir un accès SSH ou une connexion console à chaque VM Ubuntu.

2. **Modification du Fichier `/etc/hostname`**

   Utilisez les commandes suivantes pour définir les noms d'hôtes sur chaque instance :

   - **Sur le Contrôleur (`k8s-controleur`)** :

     ```bash
     sudo hostnamectl set-hostname k8s-controleur
     sudo echo "k8s-controleur" > /etc/hostname
     ```

   - **Sur les Nœuds (`k8s-travailleur01`, `k8s-travailleur02`)** :

     ```bash
     sudo hostnamectl set-hostname k8s-travailleur01  # Pour le premier nœud
     sudo echo "k8s-travailleur01" > /etc/hostname

     sudo hostnamectl set-hostname k8s-travailleur02  # Pour le deuxième nœud
     sudo echo "k8s-travailleur02" > /etc/hostname
     ```
3. **Redémarrage des Machines Virtuelles**

   Redémarrez chaque VM pour appliquer les nouveaux noms d'hôtes :

   ```bash
   sudo reboot
   ```

#### Vérification des Noms d'Hôtes

Après le redémarrage, assurez-vous que les noms d'hôtes ont été correctement définis en utilisant la commande `hostname` :

```bash
hostname
```

Cette commande devrait afficher le nom d'hôte correspondant à chaque instance (par exemple, `k8s-controleur`, `k8s-travailleur01`, `k8s-travailleur02`).

## Installation de Docker Engine

**⚠️Pour installer Docker CE sur toutes les machines virtuelles (contrôleur et nœuds de travail) en suivant un tutoriel détaillé, je vous recommande de consulter le guide que vous avez créé sur votre GitHub : [Procédure d'Installation de Docker Engine sur Ubuntu](https://github.com/marocainperdu/tp-docker).⚠️**

Ce guide fournit des instructions étape par étape pour l'installation de Docker Engine sur Ubuntu jusqu'à **l'étape 4**, y compris l'ajout du dépôt Docker, l'installation des packages Docker, et la vérification de l'installation avec un conteneur `hello-world`.

Suivez attentivement les étapes fournies dans votre tutoriel pour configurer Docker sur toutes les VMs de manière efficace et conforme à vos spécifications.

N'hésitez pas à vous référer à votre guide GitHub pour des instructions détaillées et spécifiques sur l'installation de Docker CE sur Ubuntu. Si vous avez des questions ou des problèmes spécifiques pendant le processus d'installation, n'hésitez pas à demander de l'aide ou à poser des questions supplémentaires ici.

Pour configurer Docker pour utiliser `systemd` comme pilote cgroup, suivez les étapes suivantes après avoir installé Docker CE sur vos machines virtuelles :

1. **Création du Fichier de Configuration pour Docker**

   Ouvrez un terminal sur la machine virtuelle et créez un fichier de configuration pour Docker en utilisant l'éditeur de texte `nano` :

   ```bash
   sudo nano /etc/docker/daemon.json
   ```

2. **Ajout du Contenu dans daemon.json**

   Ajoutez le contenu suivant dans le fichier `daemon.json` :

   ```json
   {
     "exec-opts": ["native.cgroupdriver=systemd"]
   }
   ```

   Enregistrez et fermez le fichier en utilisant `Ctrl+O` pour sauvegarder et `Ctrl+X` pour quitter.

3. **Redémarrage du Service Docker**

   Redémarrez le service Docker pour appliquer les nouvelles configurations :

   ```bash
   sudo systemctl restart docker
   ```

Ces étapes permettent à Docker d'utiliser `systemd` comme pilote cgroup, ce qui est nécessaire pour une compatibilité optimale avec Kubernetes.

## Installation de Kubernetes

Pour installer Kubernetes (kubeadm, kubelet, kubectl) sur toutes les machines virtuelles (contrôleur et nœuds de travail), suivez les étapes ci-dessous :

### 1. Mise à Jour du Gestionnaire de Paquets et Installation des Dépendances

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

### 2. Téléchargement de la Clé de Signature Publique pour les Dépôts Kubernetes

Téléchargez la clé de signature publique pour les dépôts Kubernetes dans le répertoire `/etc/apt/keyrings` :

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Note: Si le dossier `/etc/apt/keyrings` n'existe pas par défaut (sur les versions plus anciennes de Debian ou Ubuntu), créez-le avant d'exécuter la commande `curl`.

### 3. Ajout du Dépôt Kubernetes apt

Ajoutez le dépôt Kubernetes apt avec la version spécifiée (v1.30 dans cet exemple) :

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
```

Note: Si vous souhaitez installer une version différente de Kubernetes (par exemple, v1.31), remplacez `v1.30` par la version désirée dans les commandes ci-dessus.

### 4. Mise à Jour du Gestionnaire de Paquets et Installation de kubectl

Mettez à jour le gestionnaire de paquets apt et installez kubectl :

```bash
sudo apt-get update
sudo apt-get install -y kubectl
```

### Installation de kubeadm et kubelet

Pour installer kubeadm et kubelet (composants essentiels de Kubernetes), utilisez les commandes suivantes :

```bash
sudo apt-get install -y kubeadm kubelet
```

### Vérification de l'Installation

Pour vérifier que Kubernetes a été installé avec succès, vous pouvez exécuter les commandes suivantes :

```bash
kubectl version --short
```

Cela affichera la version de kubectl installée sur votre système.

Assurez-vous de répéter ces étapes sur toutes les machines virtuelles (contrôleur et nœuds de travail) pour installer Kubernetes de manière cohérente sur l'ensemble du cluster.

Si vous avez des besoins spécifiques concernant la version de Kubernetes à installer, assurez-vous de remplacer `v1.30` par la version souhaitée dans les étapes d'ajout du dépôt Kubernetes.
