
# 04 - Configuration du réseau virtuel

## Objectif

Configurer un **réseau virtuel isolé sous KVM/libvirt** afin que les adresses IP soient distribuées par **SERVER-DHCP**, et non par le service DHCP du réseau virtuel par défaut.

## 1. Identification du problème

Lors des premiers tests présentés dans l'étape précédente, **COMPTA-DEVICE1** obtenait bien une adresse IP après l'exécution de :

```cmd
ipconfig /renew
```

Cependant, l'adresse reçue était `192.168.122.170`, alors que j'attendais une adresse comprise entre `192.168.1.2` et `192.168.1.4`.

Avec la commande :

```cmd
ipconfig /all
```

J'avais également constaté que le serveur DHCP indiqué était `192.168.122.1`, et non **SERVER-DHCP (`192.168.1.1`)**.

Le problème n'était donc pas l'absence d'attribution DHCP, mais le fait que **le client recevait une adresse provenant d'un autre serveur DHCP**.

## 2. Découverte du réseau par défaut

Au départ, je pensais qu'il suffisait de configurer le serveur DHCP sous Windows Server pour que les machines clientes récupèrent automatiquement leurs adresses IP.

Après m'être renseigné sur les résultats obtenus, j'ai découvert que **KVM/libvirt disposait d'un réseau virtuel par défaut avec son propre service DHCP**.

Ce réseau utilisait la plage `192.168.122.0/24`, ce qui correspondait à l'adresse attribuée à mon client.

J'ai alors compris qu'il fallait également prendre en compte la configuration réseau de **KVM/libvirt**, et pas uniquement celle des machines Windows.

J'ai donc choisi de créer un réseau dédié à mon projet, sans DHCP fourni par libvirt.

## 3. Création du réseau isolé

Depuis **Virt-Manager**, création d'un nouveau réseau virtuel.

Configuration retenue :

| Paramètre | Valeur |
|-----------|--------|
| Nom | SERVER-TO-COMPTA |
| Type | Réseau isolé |
| DHCP libvirt | Désactivé |
| VM connectées | SERVER-DHCP et COMPTA-DEVICE1 |

L'objectif est que **SERVER-DHCP** soit le seul serveur chargé de distribuer les adresses IP sur ce réseau.

## 4. Raccordement des machines virtuelles

Depuis **Virt-Manager**, les interfaces réseau des deux VM ont été rattachées au réseau **SERVER-TO-COMPTA**.

| Machine | Réseau virtuel |
|---------|----------------|
| SERVER-DHCP | SERVER-TO-COMPTA |
| COMPTA-DEVICE1 | SERVER-TO-COMPTA |

Les deux machines sont ainsi connectées au même réseau virtuel isolé, sans nécessiter de routeur pour communiquer entre elles.

## 5. Résultat

Le réseau **SERVER-TO-COMPTA** est créé, avec le service DHCP de libvirt désactivé.

Les deux machines virtuelles y sont connectées.

Cette configuration permet d'utiliser le **serveur DHCP Windows** pour distribuer les adresses IP aux clients de ce réseau, sans passer par le DHCP du réseau par défaut de libvirt.

La vérification de l'attribution DHCP sera présentée dans l'étape suivante.

![Configuration du réseau isolé SERVER-TO-COMPTA dans Virt-Manager](../IMAGES/SERVER-TO-COMPTA.png)

## 6. Ce que j'ai appris

Cette difficulté m'a permis de mieux comprendre :

- La différence entre **la configuration IP d'une machine** et celle du **réseau virtuel** auquel elle est connectée.
- Le fonctionnement du réseau par défaut de **KVM/libvirt**, notamment son service DHCP.
- L'importance d'identifier quel serveur DHCP attribue réellement les adresses IP.

J'ai surtout compris qu'installer un service DHCP ne suffit pas : **les machines doivent également être connectées à un réseau permettant de l'utiliser**.

---

<sub>**Ressources :** Connaissances personnelles en adressage IP, tutoriels YouTube et utilisation de l'intelligence artificielle pour comprendre le fonctionnement des réseaux virtuels et accompagner le dépannage.</sub>
