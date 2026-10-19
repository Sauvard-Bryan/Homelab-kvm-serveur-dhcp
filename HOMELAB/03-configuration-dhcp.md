
# 03 - Configuration du réseau virtuel

## Objectif

Mettre en place un **réseau virtuel isolé** sous KVM, permettant la communication entre les deux VM :

- **SERVER-DHCP :** Windows Server 2022
- **COMPTA-DEVICE1 :** Windows 11

Le but est de disposer d'un réseau indépendant, sur lequel le serveur Windows pourra gérer l'attribution des adresses IP via le service DHCP.

## 1. Configuration initiale

Initialement, je pensais qu'il suffisait de :

- Attribuer une **adresse IP fixe** au serveur Windows.
- Installer et configurer le service **DHCP**.
- Configurer le poste client pour obtenir automatiquement une adresse IP.

Cependant, lors de la mise en pratique, j'ai rencontré plusieurs difficultés liées au fonctionnement du réseau virtuel sous KVM.

## 2. Difficultés rencontrées

### Reconnaissance des interfaces réseau

Lors de l'installation des systèmes Windows, les interfaces réseau virtuelles n'étaient pas directement reconnues.

Il a donc fallu installer les **pilotes réseau VirtIO** afin de permettre aux VM de détecter leurs cartes réseau virtuelles.

### Présence d'un DHCP automatique

Lors des premiers essais, j'ai découvert que le réseau par défaut de **libvirt** disposait déjà de son propre service DHCP.

Or, l'objectif était de laisser **SERVER-DHCP** gérer seul l'attribution des adresses IP.

J'ai donc dû revoir la configuration du réseau virtuel pour éviter que deux services DHCP interviennent sur le même réseau.

## 3. Solution appliquée

### Création d'un réseau isolé

Création d'un réseau virtuel nommé **SERVER-TO-COMPTA** depuis Virt-Manager.

Configuration retenue :

- **Nom :** SERVER-TO-COMPTA
- **Type :** Réseau virtuel isolé
- **DHCP libvirt :** Désactivé
- **Machines connectées :** SERVER-DHCP et COMPTA-DEVICE1

Ce réseau permet aux deux VM de communiquer sans utiliser le DHCP automatique du réseau par défaut de libvirt.

### Raccordement des VM

Les interfaces réseau des deux machines ont été rattachées au réseau **SERVER-TO-COMPTA**.

| Machine | Réseau virtuel |
|---------|----------------|
| SERVER-DHCP | SERVER-TO-COMPTA |
| COMPTA-DEVICE1 | SERVER-TO-COMPTA |

## 4. Résultat

Les deux machines virtuelles sont désormais connectées au réseau isolé **SERVER-TO-COMPTA**.

Cette configuration permet de préparer l'installation du service DHCP sous Windows Server 2022, sans interférence avec le DHCP du réseau par défaut de libvirt.

*Capture d'écran de la configuration du réseau SERVER-TO-COMPTA dans Virt-Manager à ajouter.*

## 5. Ce que j'ai appris

Cette étape m'a permis de comprendre que la configuration réseau d'une VM ne se limite pas à son adresse IP.

Il faut également prendre en compte :

- **Le réseau virtuel** auquel la machine est connectée.
- **Les services déjà actifs**, notamment le DHCP de libvirt.
- **Les pilotes réseau** nécessaires au fonctionnement des interfaces virtuelles.

---

<sub>**Ressources :** Connaissances personnelles en adressage IP, tutoriels YouTube et utilisation de l'intelligence artificielle pour comprendre les réseaux virtuels sous KVM et résoudre les difficultés rencontrées.</sub>
