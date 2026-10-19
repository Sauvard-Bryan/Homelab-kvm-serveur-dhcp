
# 03 - Configuration du réseau virtuel

## Objectif

Configurer un **réseau virtuel isolé** sous KVM afin de permettre la communication entre les deux VM :

- **SERVER-DHCP :** Windows Server 2022
- **COMPTA-DEVICE1 :** Windows 11

L'objectif est de disposer d'un réseau indépendant, sur lequel le serveur Windows pourra gérer l'attribution des adresses IP.

## 1. Configuration initiale

Initialement, je pensais qu'il suffisait de :

- Attribuer une **adresse IP fixe** au serveur Windows.
- Installer et configurer le service **DHCP**.
- Configurer le poste client pour recevoir automatiquement une adresse IP.

Cependant, lors de la mise en pratique, j'ai rencontré plusieurs difficultés liées à la gestion du réseau virtuel sous KVM.

## 2. Difficultés rencontrées

### Reconnaissance des interfaces réseau

Lors de l'installation des systèmes Windows, les interfaces réseau virtuelles n'étaient pas directement reconnues.

Il a donc fallu installer les **pilotes réseau VirtIO** afin de permettre aux deux VM de détecter et d'utiliser leur carte réseau virtuelle.

### Présence d'un DHCP automatique

Lors des premiers essais, j'ai découvert que le réseau par défaut de **KVM/libvirt** disposait déjà de son propre service DHCP.

Cela posait problème pour mon projet, puisque je souhaitais que **SERVER-DHCP** soit le seul serveur chargé de distribuer les adresses IP.

J'ai donc compris qu'il ne suffisait pas de configurer les adresses IP dans Windows : il fallait également maîtriser le fonctionnement du réseau virtuel sous KVM.

## 3. Solution appliquée

### Création d'un réseau virtuel isolé

Création d'un réseau isolé depuis **Virt-Manager**, sans service DHCP fourni par libvirt.

Ce réseau permet de :

- Connecter les deux VM au même sous-réseau.
- Éviter l'intervention du DHCP automatique de libvirt.
- Confier l'attribution des adresses IP au serveur Windows.

### Raccordement des machines virtuelles

Modification de la configuration réseau des deux VM afin de les connecter au nouveau réseau isolé.

| Machine | Connexion réseau |
|---------|------------------|
| SERVER-DHCP | Réseau virtuel isolé |
| COMPTA-DEVICE1 | Réseau virtuel isolé |

## 4. Résultat

Les deux machines virtuelles sont désormais connectées à un **réseau isolé**, indépendant du réseau par défaut de libvirt.

L'environnement est prêt pour la configuration du service DHCP sous Windows Server 2022.

*Capture d'écran de la configuration du réseau isolé dans Virt-Manager à ajouter.*

## 5. Ce que j'ai appris

Cette étape m'a permis de comprendre la différence entre :

- **L'adressage IP des machines**, configuré dans le système d'exploitation.
- **La gestion du réseau virtuel**, configurée au niveau de KVM/libvirt.

J'ai également découvert l'importance des pilotes VirtIO pour le fonctionnement des périphériques virtuels sous Windows.

---

<sub>**Ressources :** Connaissances personnelles en adressage IP, tutoriels YouTube et utilisation de l'intelligence artificielle pour comprendre le fonctionnement des réseaux KVM et résoudre les difficultés rencontrées.</sub>

