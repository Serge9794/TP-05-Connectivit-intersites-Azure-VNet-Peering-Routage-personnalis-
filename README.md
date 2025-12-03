# TP-05-Connectivit-intersites-Azure-VNet-Peering-Routage-personnalis-
Ce TP explore la connectivité intersite dans Azure via le peering de réseaux virtuels (VNet Peering) pour établir une communication privée entre réseaux, y compris entre régions.  Il couvre la configuration de routage personnalisé (UDR) pour diriger le trafic, notamment vers un pare-feu Azure dans le hub.  

---

# 📘 TP 05 – Connectivité Intersites dans Azure

**Mise en œuvre du Peering VNet + Tests de connectivité + Routes personnalisées**

---

## 🎯 Objectifs du TP

Dans ce laboratoire, je vais :

* Créer deux réseaux virtuels isolés
* Déployer une machine virtuelle dans chaque réseau
* Tester la connectivité avant et après peering
* Configurer un peering entre les VNets
* Vérifier la communication via Network Watcher et via PowerShell
* Créer une route personnalisée (UDR — User Defined Route)
* Documenter le tout avec des captures d’écran

---

# 🏗️ Scénario

Mon entreprise dans laquelle je travaille souhaite  séparer les services **internes** (DNS, sécurité, cœur SI) et les services **production**.
Cependant, certains modules doivent communiquer entre eux.

Ce TP simule cette architecture :

* **VNet-Socle** (services essentiels internes)
* **VNet-Prod** (production / fabrication)

Je vais activer la communication grâce au **peering VNet**.

---

# 🛠️ Compétences du TP

1. Création de VM + VNet
2. Création d’un second VNet + second VM
3. Test de connectivité avec Network Watcher
4. Peering entre VNets
5. Test via PowerShell (Run Command)
6. Création d’une route personnalisée

---

# 🔥 **Tâche 1 — Créer un VNet + une VM (Services internes)**

### 1.1. Se connecter au portail Azure

👉 [https://portal.azure.com](https://portal.azure.com)

### 1.2. Créer une machine virtuelle

* Rechercher **Virtual Machines**
* Cliquer **Create → Azure Virtual Machine**

➡️ **Paramètres à renseigner**

| Paramètre            | Valeur                 |
| -------------------- | ---------------------- |
| Nom VM               | `SrvCore-VM01`         |
| Groupe de ressources | `rg-tp05-connectivite` |
| Région               | East US                |
| Image                | Windows Server 2019    |
| Taille               | Standard_DS2_v3        |
| Username             | `Serge`           |
| Password             | (mot de passe fort)    |
| Ports publics        | Aucun                  |

**Capture1  :**
👉 `images/vm-core-basics.png`

---

### 1.3. Créer le réseau virtuel associé

Dans l’onglet **Networking**, cliquer **Create new VNet**

➡️ **Configurer :**

| Paramètre         | Valeur         |
| ----------------- | -------------- |
| Nom VNet          | `VNet-Socle`   |
| Adresse           | `10.10.0.0/16` |
| Sous-réseau       | `Subnet-Core`  |
| Adresse du subnet | `10.10.1.0/24` |

**Capture 2 :**
👉 `images/vnet-socle.png`

---

### 1.4. Finaliser la création

* Désactiver Boot Diagnostics
* Cliquer **Review + Create → Create**

---

# 🔥 **Tâche 2 — Créer un second VNet + seconde VM (Production)**

### 2.1. Créer une nouvelle VM

➡️ **Paramètres**

| Paramètre     | Valeur              |
| ------------- | ------------------- |
| Nom VM        | `Prod-VM01`         |
| Région        | East US             |
| Image         | Windows Server 2019 |
| Username      | `Polo`        |
| Ports publics | Aucun               |

**Capture 3:**

---

### 2.2. Créer le deuxième VNet

Dans l’onglet Réseau → **Create new VNet**

| Paramètre         | Valeur          |
| ----------------- | --------------- |
| Nom VNet          | `VNet-Prod`     |
| Adresse           | `172.20.0.0/16` |
| Sous-réseau       | `Subnet-Prod`   |
| Adresse du subnet | `172.20.1.0/24` |

**Capture 4  :**
👉 `images/vnet-prod.png`

---

# 🔥 **Tâche 3 — Tester la connexion avec Network Watcher**

### 3.1. Accéder à Network Watcher

Menu → **Network Watcher** → *Connection Troubleshoot*

➡️ **Paramètres :**

| Champ       | Valeur       |
| ----------- | ------------ |
| Source      | SrvCore-VM01 |
| Destination | Prod-VM01    |
| Protocole   | TCP          |
| Port        | 3389         |

**Capture 5 :**
👉 `images/network-watcher-before-peering.png`

➡️ Le résultat doit être **Unreachable** (normal !).

---

# 🔥 **Tâche 4 — Configurer le Peering VNet**

### 4.1. Depuis VNet-Socle

* Aller dans :
  **VNet-Socle → Peering → Add**

➡️ **Paramètres :**

| Paramètre               | Valeur          |
| ----------------------- | --------------- |
| Nom du peering          | `Socle-to-Prod` |
| Remote VNet             | `VNet-Prod`     |
| Allow VNet access       | ✔️              |
| Allow forwarded traffic | ✔️              |

**Capture 6:**
---

### 4.2. Depuis VNet-Prod

Même manipulation :

| Paramètre               | Valeur          |
| ----------------------- | --------------- |
| Nom                     | `Prod-to-Socle` |
| Remote VNet             | `VNet-Socle`    |
| Allow VNet access       | ✔️              |
| Allow forwarded traffic | ✔️              |

**Capture 7:**
👉 `images/peering-connected.png`

➡️ Vérifier que l’état passe à **Connected**

---

# 🔥 **Tâche 5 — Tester la connexion via PowerShell Run Command**

### 5.1. Récupérer l’IP privée de SrvCore-VM01

Exemple : **10.10.1.4**

### 5.2. Depuis Prod-VM01

Aller dans :
**Prod-VM01 → Run Command → RunPowerShellScript**

Exécuter :

```powershell
Test-NetConnection 10.10.1.4 -Port 3389
```

Résultat attendu : **TcpTestSucceeded : True**

**Capture 8 :**
👉 `images/test-netconnection-success.png`

---

# 🔥 **Tâche 6 — Créer une route personnalisée (UDR)**

### 6.1. Ajouter un nouveau sous-réseau "Perimeter" dans VNet-Socle

* VNet-Socle → Subnets → **Add**

| Paramètre | Valeur             |
| --------- | ------------------ |
| Nom       | `Subnet-Perimeter` |
| Adresse   | `10.10.2.0/24`     |

**Capture 9:**

---

### 6.2. Créer une table de routage

Menu → **Route tables → Create**

| Paramètre                  | Valeur     |
| -------------------------- | ---------- |
| Nom                        | `rt-socle` |
| Region                     | East US    |
| Propager routes de gateway | No         |

---

### 6.3. Ajouter une route

Table → **Routes → Add**

| Paramètre        | Valeur                             |
| ---------------- | ---------------------------------- |
| Nom route        | `Perimeter-to-Core`                |
| Destination      | `10.10.0.0/16`                     |
| Next hop         | Virtual Appliance                  |
| Adresse next hop | `10.10.2.7` *(futur firewall/NVA)* |

---

### 6.4. Associer la route au sous-réseau

* Route table → **Subnets → Associate**

| Paramètre | Valeur      |
| --------- | ----------- |
| VNet      | VNet-Socle  |
| Subnet    | Subnet-Core |

**Capture 10 :**
👉 `images/udr-routing.png`

---

# 🧹 Nettoyage des ressources

Supprimer le groupe de ressources :

### Via portail

➡️ Groupe de ressources → **Delete resource group**

### Via PowerShell

```powershell
Remove-AzResourceGroup -Name rg-tp05-connectivite
```

### Via Azure CLI

```bash
az group delete --name rg-tp05-connectivite
```

---

# 📌 Points clés à retenir

* Les ressources dans différents VNets **ne communiquent pas nativement**
* Le **VNet peering** permet une connectivité transparente
* Le trafic utilise l’infrastructure **backbone Microsoft**
* Les **UDR** permettent de contrôler le chemin du trafic
* **Network Watcher** est essentiel pour diagnostiquer la connectivité

---

