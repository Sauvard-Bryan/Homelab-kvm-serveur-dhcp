
# 02 - Création des machines virtuelles

## Objectif

Créer une première infrastructure réseau composée de **deux machines virtuelles (VM)** : un serveur DHCP et un poste client, connectés au **même réseau virtuel**.

Cette première configuration permet de tester le fonctionnement du DHCP avant d'étendre l'infrastructure à plusieurs réseaux.

## Infrastructure initiale

Création de deux machines virtuelles sous **KVM**, à l'aide de **Virt-Manager**.

| Machine | Système d'exploitation | Rôle |
|---------|------------------------|------|
| SERVER-DHCP | Windows Server 2022 | Serveur DHCP |
| COMPTA-DEVICE1 | Windows 11 | Poste client |

## Création des machines virtuelles

### 1. VM Serveur DHCP

- **Nom :** SERVER-DHCP
- **Système d'exploitation :** Microsoft Windows Server 2022
- **RAM :** 2 Go
- **CPU :** 2 vCPU
- **Stockage :** 40 Go

### 2. VM Cliente

- **Nom :** COMPTA-DEVICE1
- **Système d'exploitation :** Microsoft Windows 11
- **RAM :** 4 Go
- **CPU :** 2 vCPU
- **Stockage :** 40 Go

## Dimensionnement des ressources

### Configuration de la machine hôte

- **Système :** Debian
- **Processeur :** Intel Core i5-1135G7 (4 cœurs / 8 threads)
- **RAM :** 8 Go
- **Stockage :** 225 Go

### Optimisation de la mémoire

Initialement, **4 Go de RAM étaient attribués à chaque VM**, soit la totalité des 8 Go disponibles sur la machine hôte.

Afin de limiter la consommation mémoire et de conserver des ressources pour Debian, la répartition a été ajustée :

| Machine | RAM initiale | RAM optimisée |
|---------|-------------|---------------|
| SERVER-DHCP | 4 Go | 2 Go |
| COMPTA-DEVICE1 | 4 Go | 4 Go |
| **Total VM** | **8 Go** | **6 Go** |

Cette configuration permet de conserver environ **2 Go de RAM pour le système hôte**, sous réserve de la consommation réelle des VM.

Le dimensionnement reste limité, notamment pour Windows Server 2022, et devra être validé par des tests de performance.

## Configuration réseau initiale

Les deux VM sont connectées au **même réseau virtuel**, sans routeur ni relais DHCP.

La configuration réseau et le plan d'adressage seront détaillés dans l'étape suivante.

## Résultat

Deux machines virtuelles créées sous KVM :

- **SERVER-DHCP :** serveur Windows Server 2022.
- **COMPTA-DEVICE1 :** poste client Windows 11.

Infrastructure prête pour la configuration du réseau et du service DHCP.

*[Capture d'écran des deux VM dans Virt-Manager à ajouter]*

## Évolution prévue

Après validation de cette première infrastructure :

- Ajout d'une **deuxième VM cliente** sur un sous-réseau différent.
- Création d'une **VM routeur** pour assurer la communication entre les réseaux.
- Configuration d'un **relais DHCP** pour distribuer les adresses IP sur plusieurs sous-réseaux.

---

<sub>**Ressources :** Tutoriels YouTube et utilisation de l'intelligence artificielle pour accompagner la configuration, comprendre les concepts et résoudre les problèmes rencontrés.</sub>
