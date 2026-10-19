
# 02 - Création des machines virtuelles

## Objectif

Créer une première infrastructure réseau composée de **deux machines virtuelles (VM)** : un serveur DHCP et un poste client, connectés au **même réseau virtuel**.

Cette configuration permet de tester le fonctionnement du DHCP avant d'étendre l'infrastructure à plusieurs réseaux.

## Configuration de la machine hôte

- **Système :** Debian
- **Processeur :** Intel Core i5-1135G7 (4 cœurs / 8 threads)
- **RAM :** 8 Go
- **Stockage :** 225 Go

## Création des machines virtuelles

Création de deux VM sous **KVM**, à l'aide de **Virt-Manager**.

### 1. Serveur DHCP

- **Nom :** SERVER-DHCP
- **Système d'exploitation :** Microsoft Windows Server 2022
- **RAM :** 2 Go
- **CPU :** 2 vCPU
- **Stockage :** 40 Go

### 2. Poste client

- **Nom :** COMPTA-DEVICE1
- **Système d'exploitation :** Microsoft Windows 11
- **RAM :** 4 Go
- **CPU :** 2 vCPU
- **Stockage :** 40 Go

### Choix des ressources

Les ressources ont été réparties en tenant compte des capacités de la machine hôte :

- **RAM :** 6 Go alloués aux deux VM, laissant environ 2 Go pour Debian.
- **CPU :** 2 vCPU par VM, adaptés aux besoins de ce premier environnement de test.
- **Stockage :** 40 Go par VM, pour l'installation des systèmes d'exploitation et des services.

Cette répartition constitue un compromis entre les besoins des machines virtuelles et les ressources disponibles sur l'hôte.

## Configuration réseau initiale

Les deux VM sont connectées au **même réseau virtuel**, sans routeur ni relais DHCP.

Cette première configuration permet de tester l'attribution automatique d'une adresse IP par le serveur DHCP au poste client.

La configuration réseau et le plan d'adressage seront détaillés dans la prochaine étape.

## Résultat

Deux machines virtuelles créées sous KVM :

- **SERVER-DHCP :** serveur Windows Server 2022.
- **COMPTA-DEVICE1 :** poste client Windows 11.

*Capture d'écran des deux VM dans Virt-Manager à ajouter.*

## Évolution prévue

Après validation de cette première infrastructure :

- Ajout d'une **deuxième VM cliente** sur un autre sous-réseau.
- Création d'une **VM routeur** pour assurer la communication entre les réseaux.
- Mise en place d'un **relais DHCP** pour distribuer les adresses IP sur plusieurs sous-réseaux.

---

<sub>**Ressources :** Tutoriels YouTube et utilisation de l'intelligence artificielle pour accompagner la configuration, comprendre les concepts et résoudre les problèmes rencontrés.</sub>
