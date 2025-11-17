# Projet : Infrastructure Fictive - Services Réseaux et Sécurité

## 1. 🎯 Objectif du Projet

Ce projet vise à construire et configurer une infrastructure d'entreprise multi-site complète au sein d'un laboratoire de virtualisation (VMware Workstation). L'architecture est basée sur le schéma "Infrastructure Fictive Services réseaux et Sécurité", avec des modifications spécifiques pour ce laboratoire.

L'objectif principal est de déployer deux sites distincts, **"NANTES" (Site principal)** et **"RENNES" (Site secondaire)**, de les sécuriser avec des pare-feu, et d'établir une interconnexion sécurisée entre eux via un "Internet" simulé.

## 2. 🏛️ Topologie de l'Infrastructure

L'infrastructure est divisée en trois zones réseau principales :

### Site Principal: NANTES (`172.16.1.0/24`)
Ce site héberge les services critiques de production.
* **Contrôleur de domaine :** `SRV-AD1` (Active Directory, DNS, PKI, Radius)
* **Serveur de fichiers :** `SRV-FIC1` (DFSR)
* **Stockage :** `FREENAS-1`
* **Proxy :** `SRV-PROXY` (Debian 7 / Squid)
* **Pare-feu / Routeur :** `pfSense-NANTES` (IP LAN: `172.16.1.254`)

### Site Secondaire: RENNES (`172.16.0.0/24`)
Ce site sert de site de reprise d'activité (PRA) et de redondance.
* **Contrôleur de domaine :** `SRV-AD2` (AD Secondaire, DNS)
* **Serveur de fichiers :** `SRV-FIC2` (DFSR)
* **Stockage :** `FREENAS-2`
* **Serveur de sauvegarde :** `SRV-VEEAM`
* **Pare-feu / Routeur :** `pfSense-RENNES` (IP LAN: `172.16.0.254`)

### Le Réseau "Internet" (WAN)
Pour simuler un Fournisseur d'Accès Internet (FAI) et connecter nos deux sites, un routeur central est utilisé :
* **Routeur FAI :** `routerwan` (VM Debian 12)
* **Réseau WAN simulé :** `198.51.100.0/24`

---

## 3. 🔧 Modifications Clés (Ce qui change du schéma)

Cette implémentation s'écarte du schéma original sur les points suivants :

* **Remplacement du Sophos :** Le pare-feu Sophos du site de Rennes est remplacé par une **seconde instance de pfSense** (`pfSense-RENNES`) pour standardiser les équipements et permettre la création d'un tunnel VPN Site-to-Site (IPsec ou OpenVPN) entre les deux pare-feu.
* **Pas d'ESXi imbriqué :** Le serveur `SRV-ESX-STAG` ne sera pas déployé en tant qu'hyperviseur imbriqué. Les VM du site de Rennes (`SRV-VEEAM`, `SRV-AD2`, etc.) seront déployées directement sur l'hyperviseur principal (VMware Workstation), tout comme le site de Nantes.
* **Client Externe :** Le `CLIENT` sera utilisé pour les tests de connexion depuis l'extérieur (ex: test de VPN client).

---

## 4. 🛠️ Composants de l'Infrastructure

Cette section détaille la configuration de chaque machine virtuelle du projet.

### 4.1. VM `routerwan` (Routeur FAI / Debian 12)

Cette VM agit comme un routeur FAI pour simuler le réseau WAN de l'infrastructure.

#### 4.1.1. Configuration Réseau

| Interface Logique | Nom (Debian) | Connexion VMware | Configuration IP | Adresse IP |
| :--- | :--- | :--- | :--- | :--- |
| **Carte 1 (Internet)** | `ens33` | `NAT (VMnet8)` | `DHCP` | `192.168.70.129/24` *(Ex.)* |
| **Carte 2 (WAN Labo)** | `ens34` | `LAN Segment (WAN_PFSENSE)`| `Statique` | `198.51.100.2/24` |

#### 4.1.2. Fichier `/etc/network/interfaces`

Pour assurer une configuration IP stable et contourner NetworkManager (GUI), le fichier `/etc/network/interfaces` est configuré comme suit :

```bash
# Fichier /etc/network/interfaces

# The loopback network interface
auto lo
iface lo inet loopback

# CARTE 1 (VMnet8 / NAT pour Internet)
auto ens33
iface ens33 inet dhcp

# CARTE 2 (LAN Segment 'WAN_PFSENSE')
auto ens34
iface ens34 inet static
    address 198.51.100.2
    netmask 255.255.255.0
```
####4.1.3. Services de Routage (NAT)
Pour partager la connexion Internet de ens33 avec le réseau ens34 (où se trouvera le pfSense), le transfert IP et le NAT (Masquerading) ont été configurés.

1. Activation du Forwarding (dans /etc/sysctl.conf) :

```Bash

# Décommenter la ligne suivante:
net.ipv4.ip_forward=1
2. Installation des services (iptables) :
```
```Bash

apt-get update
apt-get install iptables iptables-persistent
```

3. Application des règles de NAT :

```Bash

# Règle de NAT (Masquerading)
iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE

# Autoriser le trafic de pfSense vers Internet
iptables -A FORWARD -i ens34 -o ens33 -j ACCEPT

# Autoriser les réponses d'Internet vers pfSense
iptables -A FORWARD -i ens33 -o ens34 -m state --state RELATED,ESTABLISHED -j ACCEPT

###4. Sauvegarde des règles :

```Bash

iptables-save > /etc/iptables/rules.v4
