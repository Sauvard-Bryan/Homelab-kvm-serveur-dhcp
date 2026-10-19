
# 02 - Création des machines virtuelles

## Objectif

Créer une première infrastructure réseau composée de **deux machines virtuelles (VM)** : un serveur DHCP et un poste client, connectés au **même réseau virtuel**.

L'objectif est de tester le fonctionnement du service DHCP sur une infrastructure simple avant de l'étendre à plusieurs réseaux.

## Infrastructure initiale

Création de deux machines virtuelles sous **KVM**, à l'aide de **Virt-Manager**.

| Machine | Système d'exploitation | Rôle |
|---------|------------------------|------|
| SERVER-DHCP | Windows Server 2022 | Serveur DHCP |
| COMPTA-DEVICE1 | Windows 11 | Poste client |

## Création des machines virtuelles

### 1. VM Serveur DHCP

Création d'une première machine virtuelle destinée à héberger le service DHCP.

- **Nom :** SERVER-DHCP
- **Système d'exploitation :** Microsoft Windows Server 2022
- **RAM :** 4 Go
- **CPU :** 2 vCPU
- **Stockage :** 40 Go

### 2. VM Cliente

Création d'une deuxième machine virtuelle permettant de tester l'attribution automatique des adresses IP par le serveur DHCP.

- **Nom :** COMPTA-DEVICE1
- **Système d'exploitation :** Microsoft Windows 11
- **RAM :** 4 Go
- **CPU :** 2 vCPU
- **Stockage :** 40 Go

## Configuration réseau initiale

Les deux machines virtuelles sont connectées au **même réseau virtuel**.

Cette première configuration permet de tester la communication entre le serveur et le client, sans nécessiter de routeur ni de relais DHCP.

La configuration réseau et le plan d'adressage seront détaillés dans la prochaine étape.

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
- Mise en place d'un **relais DHCP** permettant de distribuer les adresses IP sur plusieurs sous-réseaux.
