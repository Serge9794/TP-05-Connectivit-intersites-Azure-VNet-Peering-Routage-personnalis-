# TP-05-Connectivit-intersites-Azure-VNet-Peering-Routage-personnalis-
Ce TP explore la connectivité intersite dans Azure via le peering de réseaux virtuels (VNet Peering) pour établir une communication privée entre réseaux, y compris entre régions.  Il couvre la configuration de routage personnalisé (UDR) pour diriger le trafic, notamment vers un pare-feu Azure dans le hub.  


# 🧪 TP 05 — Connectivité intersites Azure (VNet Peering & Routage personnalisé)

Dans ce laboratoire, je mets en œuvre la connectivité entre deux réseaux virtuels Azure isolés.
Pour cela, j’ai créé deux machines virtuelles dans deux Virtual Networks différents, testé la communication, configuré un VNet Peering, validé la connexion via Network Watcher et PowerShell, puis ajouté une route personnalisée pour contrôler le trafic réseau.

Ce TP est prévu pour être exécuté dans Azure Portal, étape par étape, même pour un débutant.

## 🎯 Objectifs du TP

Comprendre comment les réseaux virtuels Azure communiquent

Configurer un VNet peering entre deux zones réseau segmentées

Tester la connectivité avec Network Watcher et PowerShell

Mettre en place une User Defined Route (UDR)

Apprendre la structure d’un réseau Azure “Core” + “Production”

## 🗺️ Architecture cible du TP
CoreInfraVnet (10.10.0.0/16)
 ├── CoreSubnet (10.10.0.0/24)
 └── PerimeterSubnet (10.10.1.0/24)
       ↑ Future NVA (10.10.1.7)

ProdServicesVnet (172.20.0.0/16)
 └── ProdSubnet (172.20.0.0/24)

Peering bidirectionnel entre les deux VNets
Routage personnalisé du Core vers la NVA

## 🛠️ Prérequis

Un abonnement Azure (même gratuit)

Accès au portail : https://portal.azure.com

Permission Contributor ou plus

# ✅ Tâche 1 — Création du réseau "Core" et de la VM CoreInfraVM
🎯 Objectif

Déployer le réseau “services centraux” et une VM associée.

👉 Étapes détaillées
Étape 1 : Créer une machine virtuelle

Dans le portail Azure, rechercher Virtual Machines

Cliquer sur Create → Azure virtual machine

Renseigner les valeurs suivantes :

Paramètre	Valeur
Resource Group	tp05-rg (crée-le si besoin)
Virtual machine name	CoreInfraVM
Region	East US
Image	Windows Server 2019 Datacenter
Size	Standard_DS2_v3
Username	infraadmin
Password	un mot de passe complexe
Public inbound ports	None
Étape 2 : Créer le Virtual Network lié à la VM

Accéder à l’onglet Networking

Sous "Virtual Network", cliquer sur Create new

Renseigner :

Paramètre	Valeur
VNet Name	CoreInfraVnet
Address space	10.10.0.0/16
Subnet name	CoreSubnet
Subnet range	10.10.0.0/24

Valider → OK.

Étape 3 : Finaliser la création

Aller dans l'onglet Monitoring

Désactiver Boot diagnostics

Cliquer sur Review + Create

Puis Create

# ✅ Tâche 2 — Création du réseau “Production” et de la VM ProdServicesVM
🎯 Objectif

Créer un second réseau totalement isolé du premier.

👉 Étapes détaillées

Aller dans Virtual Machines → Create

Renseigner :

Paramètre	Valeur
Virtual machine name	ProdServicesVM
Resource group	tp05-rg
Region	East US
Image	Windows Server 2019
Size	Standard_DS2_v3
Username	infraadmin
Ports publics	None
Créer un nouveau réseau virtuel pour la production

Dans l’onglet Networking :

Virtual Network → Create new

Renseigner :

Paramètre	Valeur
VNet Name	ProdServicesVnet
Address space	172.20.0.0/16
Subnet name	ProdSubnet
Subnet range	172.20.0.0/24

Finaliser : Review + Create → Create

# ✅ Tâche 3 — Tester la connexion entre les VM avec Network Watcher
🎯 Objectif

Vérifier qu’avant le peering les VM NE PEUVENT PAS communiquer.

👉 Étapes détaillées

Rechercher Network Watcher

Dans le menu, cliquer sur Connection Troubleshoot

Configurer :

Paramètre	Valeur
Source type	Virtual machine
Source VM	CoreInfraVM
Destination type	Virtual machine
Destination VM	ProdServicesVM
Protocol	TCP
Port	3389

Cliquer sur Run diagnostic

➡️ Résultat attendu : Unreachable (normal)

# ✅ Tâche 4 — Configurer le VNet Peering
🎯 Objectif

Autoriser les réseaux Core et Production à communiquer.

👉 Étapes détaillées
Depuis CoreInfraVnet

Aller dans Virtual Networks → CoreInfraVnet

Menu Peerings → Add

Renseigner :

Paramètre	Valeur
Peering name	Prod-to-Core
Virtual Network	ProdServicesVnet
Allow VNet access	✔
Allow forwarded traffic	✔

Créer.

Vérification

Aller dans Peerings

Statut attendu : Connected

Faire la même vérification dans ProdServicesVnet.

# ✅ Tâche 5 — Test de connectivité avec PowerShell (Run Command)
🎯 Objectif

Confirmer que le peering fonctionne en testant la communication directe IP privée → IP privée.

👉 Étapes détaillées
1. Récupérer l’IP privée

Ouvrir CoreInfraVM

Dans Networking, noter l’IP privée (ex : 10.10.0.4)

2. Tester la connexion depuis ProdServicesVM

Ouvrir ProdServicesVM

Menu gauche → Run command

Choisir RunPowerShellScript

Exécuter :

Test-NetConnection 10.10.0.4 -Port 3389


➡️ Résultat attendu : TcpTestSucceeded : True

# ✅ Tâche 6 — Créer une User Defined Route (UDR)
🎯 Objectif

Faire passer le trafic Core → Perimeter via une appliance virtuelle fictive.

👉 Étapes détaillées
Étape 1 : Ajouter un subnet “Perimeter”

Ouvrir CoreInfraVnet

Aller dans Subnets → Add

Paramètres :

Paramètre	Valeur
Name	PerimeterSubnet
Range	10.10.1.0/24
Étape 2 : Créer une table de routage

Rechercher Route tables

Cliquer sur Create

Renseigner :

Paramètre	Valeur
Name	rt-CoreInfra
Resource group	tp05-rg
Region	East US
Propagate gateway routes	No

Créer.

Étape 3 : Ajouter la route personnalisée

Ouvrir rt-CoreInfra

Menu Routes → Add

Paramètres :

Paramètre	Valeur
Route name	PerimeterToCore
Destination type	IP address
Destination prefix	10.10.0.0/16
Next hop	Virtual appliance
Next hop IP	10.10.1.7
Étape 4 : Associer la table au CoreSubnet

Menu Subnets → Associate

Sélectionner :

Paramètre	Valeur
VNet	CoreInfraVnet
Subnet	CoreSubnet
# 🧹 Nettoyage des ressources
Via le portail :

Aller dans Resource groups

Choisir tp05-rg

Cliquer Delete resource group

Via PowerShell
Remove-AzResourceGroup -Name tp05-rg

Via Azure CLI
az group delete --name tp05-rg

# 📝 Points essentiels à retenir

Les VNets ne communiquent pas entre eux sans peering

Le VNet peering utilise le backbone Microsoft, rapide et sécurisé

Les UDR permettent d’imposer un chemin réseau

Network Watcher est indispensable pour diagnostiquer la connectivité

Les Run Commands permettent des tests sans se connecter en RDP
