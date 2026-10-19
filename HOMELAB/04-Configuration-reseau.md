
# 04 - Configuration du réseau virtuel

## Objectif

Résoudre le problème d'attribution des adresses IP rencontré lors des premiers tests DHCP, en configurant un **réseau virtuel isolé sous KVM/libvirt**.

## 1. Identification du problème

Lors des premiers tests, le poste **COMPTA-DEVICE1** ne parvenait pas à obtenir une adresse IP depuis **SERVER-DHCP**.

Malgré une étendue DHCP créée et activée, la commande :

```cmd
ipconfig /renew
```

renvoyait une **erreur de dépassement du délai d'attente**.

J'ai donc poursuivi les vérifications en m'intéressant à la configuration réseau des machines virtuelles.

## 2. Découverte du réseau par défaut

En examinant la configuration de **KVM/libvirt**, j'ai découvert que le réseau virtuel par défaut disposait déjà de son propre service DHCP.

Initialement, je pensais qu'il suffisait de configurer une adresse IP fixe sur Windows Server, puis d'activer le service DHCP pour distribuer des adresses aux clients.

Cette difficulté m'a permis de comprendre qu'il fallait également prendre en compte **l'infrastructure réseau virtuelle**, et pas uniquement la configuration des systèmes Windows.

J'ai donc décidé de créer un réseau dédié à mon projet, sans DHCP fourni par libvirt.

## 3. Création du réseau isolé

Depuis **Virt-Manager**, création d'un nouveau réseau virtuel.

Configuration retenue :

| Paramètre | Valeur |
|-----------|--------|
| Nom | SERVER-TO-COMPTA |
| Type | Réseau isolé |
| DHCP libvirt | Désactivé |
| VM connectées | SERVER-DHCP et COMPTA-DEVICE1 |

L'objectif est que **SERVER-DHCP** soit le seul équipement chargé de distribuer les adresses IP sur ce réseau.

## 4. Raccordement des machines virtuelles

Les interfaces réseau des deux VM ont été rattachées au réseau **SERVER-TO-COMPTA** depuis Virt-Manager.

| Machine | Réseau virtuel |
|---------|----------------|
| SERVER-DHCP | SERVER-TO-COMPTA |
| COMPTA-DEVICE1 | SERVER-TO-COMPTA |

Les deux machines se trouvent ainsi sur un même réseau virtuel, sans nécessiter de routeur.

## 5. Résultat

Le réseau **SERVER-TO-COMPTA** est créé et les deux machines virtuelles y sont connectées.

Cette configuration permet de disposer d'un environnement réseau isolé, dans lequel l'attribution des adresses IP peut être assurée par le service DHCP de Windows Server 2022.

La validation de l'attribution DHCP sera présentée dans l'étape suivante.

*Capture d'écran de la configuration du réseau SERVER-TO-COMPTA dans Virt-Manager à ajouter.*

## 6. Ce que j'ai appris

Cette difficulté m'a permis de mieux comprendre :

- La différence entre **l'adressage IP d'une machine** et la **configuration d'un réseau virtuel**.
- La présence possible de services réseau déjà actifs, comme le DHCP de libvirt.
- L'importance de vérifier l'environnement réseau lors d'un problème de communication entre VM.

---

<sub>**Ressources :** Connaissances personnelles en adressage IP, tutoriels YouTube et utilisation de l'intelligence artificielle pour comprendre le fonctionnement des réseaux virtuels et accompagner le dépannage.</sub>
