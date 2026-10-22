
# 06 - Évolution de l'infrastructure : OPNsense et relais DHCP

## Objectif

Après avoir réussi à mettre en place mon premier serveur DHCP sous **Windows Server 2022**, et validé son fonctionnement avec un poste client **Windows 11**, j'ai voulu aller plus loin.

Jusqu'ici, mon infrastructure comportait **un serveur DHCP et un seul poste client**, connectés au même réseau virtuel isolé.

Je me suis alors posé une nouvelle question :

**Si je voulais ajouter un deuxième poste client, mais le placer sur un réseau différent du premier, comment les deux machines pourraient-elles obtenir une adresse IP depuis le même serveur DHCP ?**

En me renseignant sur cette possibilité, j'ai découvert les notions de **sous-réseaux, de routage et de relais DHCP**.

Je n'avais encore jamais installé ni configuré de routeur.

J'ai donc décidé de faire évoluer mon infrastructure existante vers une architecture composée de **trois réseaux distincts**, reliés par un routeur **OPNsense**.

L'objectif était de conserver un seul serveur DHCP Windows, capable de distribuer des adresses IP aux postes des réseaux **Comptabilité** et **Commercial**.

Pour valider cette nouvelle configuration, j'ai utilisé ma VM Windows 11 existante en la connectant successivement aux deux réseaux.

## 1. Évolution de l'architecture réseau

Dans les chapitres précédents, **SERVER-DHCP** et **COMPTA-DEVICE1** étaient connectés au même réseau isolé.

Le serveur distribuait des adresses IP sur le réseau `192.168.1.0/24`.

Pour ajouter un deuxième réseau, j'ai décidé de revoir cette organisation.

### Nouvelle architecture

| Réseau | Sous-réseau | Utilisation |
|--------|-------------|-------------|
| Serveur | `192.168.1.0/24` | Communication entre Windows Server et OPNsense |
| Comptabilité | `192.168.2.0/24` | Postes du service Comptabilité |
| Commercial | `192.168.3.0/24` | Postes du service Commercial |

Le réseau `192.168.1.0/24` est désormais réservé au serveur et à l'interface LAN d'OPNsense.

Les postes clients sont placés sur les réseaux `192.168.2.0/24` et `192.168.3.0/24`.

OPNsense possède une interface sur chacun de ces réseaux afin de permettre leur interconnexion.

*Capture à ajouter : schéma de l'architecture finale avec SERVER-DHCP, OPNsense et les deux réseaux clients.*

## 2. Création de la VM OPNsense

Pour relier les différents réseaux, j'ai créé une nouvelle machine virtuelle dans **Virt-Manager**.

### Système utilisé

- **Nom de la VM :** ROUTAGE
- **Système :** OPNsense
- **Virtualisation :** KVM/QEMU
- **Gestion :** Virt-Manager

OPNsense est une solution de routage et de pare-feu basée sur FreeBSD.

Dans ce projet, je l'utilise principalement pour assurer le **routage entre les réseaux** et le **relais DHCP**.

Après avoir créé la VM à partir de l'image d'installation d'OPNsense, j'ai procédé à l'installation du système.

*Capture à ajouter : VM ROUTAGE dans Virt-Manager.*

## 3. Création des réseaux isolés sous KVM

Pour séparer les différents services, j'ai créé des réseaux virtuels isolés depuis **Virt-Manager**.

### Réseaux utilisés

- Réseau Serveur : réseau initial adapté à la nouvelle architecture.
- **COMPTABILITE** : réseau destiné aux postes Comptabilité.
- **COMMERCIAL** : réseau destiné aux postes Commercial.

L'utilisation de réseaux isolés permet de séparer les machines au niveau du réseau virtuel.

Les communications entre ces sous-réseaux doivent alors passer par le routeur OPNsense.

### Désactivation du DHCP de libvirt

Lors de la première partie du projet, j'avais découvert que le réseau virtuel par défaut de KVM/libvirt disposait de son propre serveur DHCP.

Pour éviter qu'il attribue des adresses IP à la place de mon serveur Windows, j'ai veillé à **désactiver le DHCP de libvirt** sur les réseaux utilisés pour les clients.

L'objectif est que les adresses soient uniquement distribuées par **SERVER-DHCP**.

### Ajout des cartes réseau sur OPNsense

Pour raccorder OPNsense aux trois réseaux, la VM dispose de **trois cartes réseau au total**.

J'ai donc ajouté deux cartes supplémentaires à celle présente initialement.

| Carte | Interface OPNsense | Réseau associé |
|-------|--------------------|---------------|
| Carte 1 | LAN (`vtnet0`) | Serveur |
| Carte 2 | OPT1 (`vtnet1`) | COMPTABILITE |
| Carte 3 | OPT2 (`vtnet2`) | COMMERCIAL |

Chaque carte est connectée au réseau virtuel correspondant dans Virt-Manager.

*Capture à ajouter : liste des réseaux virtuels isolés dans Virt-Manager.*

*Capture à ajouter : les trois cartes réseau de la VM ROUTAGE et leurs réseaux associés.*

## 4. Configuration des interfaces OPNsense

Après avoir raccordé les cartes réseau, j'ai configuré leurs adresses IPv4 depuis l'interface d'administration d'OPNsense.

J'ai choisi une configuration **Static IPv4**, afin que les adresses du routeur restent fixes.

### Adresses configurées

| Interface | Adresse IPv4 | Masque | Réseau |
|-----------|--------------|--------|--------|
| LAN | `192.168.1.254` | `255.255.255.0` | Serveur |
| OPT1 - COMPTABILITE | `192.168.2.1` | `255.255.255.0` | Comptabilité |
| OPT2 - COMMERCIAL | `192.168.3.1` | `255.255.255.0` | Commercial |

Ces adresses deviennent les **passerelles des différents réseaux**.

### Rôle des passerelles

Une passerelle permet à une machine de communiquer avec un réseau différent du sien.

Par exemple, un poste Comptabilité ayant reçu l'adresse `192.168.2.2` utilise `192.168.2.1` comme passerelle.

Pour atteindre le serveur DHCP situé sur le réseau `192.168.1.0/24`, les communications passent par OPNsense.

Il est donc important que chaque réseau utilise **la passerelle correspondant à son propre sous-réseau**.

*Capture à ajouter : configuration des interfaces LAN, COMPTABILITE et COMMERCIAL dans OPNsense.*

## 5. Modification de la configuration DHCP Windows

Dans la première version de mon infrastructure, le serveur DHCP distribuait des adresses sur le réseau `192.168.1.0/24`.

L'étendue initiale utilisait la plage :

`192.168.1.2` à `192.168.1.4`

Avec la nouvelle architecture, ce réseau est réservé à la communication entre **SERVER-DHCP** et **OPNsense**.

Cette ancienne plage n'est donc plus utilisée pour les postes clients.

J'ai adapté la configuration du serveur Windows pour prendre en charge les réseaux Comptabilité et Commercial.

### 5.1. Configuration IP du serveur

Le serveur conserve son adresse IPv4 fixe `192.168.1.1`.

En revanche, avec l'ajout du routeur, une passerelle est désormais nécessaire pour communiquer avec les autres réseaux.

| Paramètre | Configuration |
|-----------|---------------|
| Adresse IPv4 | `192.168.1.1` |
| Masque | `255.255.255.0` |
| Passerelle | `192.168.1.254` |
| DNS préféré | `192.168.1.1` |

La passerelle `192.168.1.254` correspond à l'interface LAN d'OPNsense, située sur le même réseau que Windows Server.

### 5.2. Adaptation de l'étendue Serveur

L'étendue initiale du réseau `192.168.1.0/24` avait été créée lorsque le serveur et le client se trouvaient sur le même réseau.

Avec la nouvelle organisation, elle n'est plus destinée à distribuer des adresses aux postes Comptabilité ou Commercial.

La passerelle correspondant au réseau Serveur est maintenant :

`192.168.1.254`

Dans les options DHCP, cette information correspond à l'option **003 Routeur**.

### 5.3. Création de l'étendue Comptabilité

Depuis la console DHCP de Windows Server, j'ai créé une nouvelle étendue pour les postes Comptabilité.

| Paramètre | Configuration |
|-----------|---------------|
| Réseau | `192.168.2.0/24` |
| Adresse de début | `192.168.2.2` |
| Adresse de fin | `192.168.2.4` |
| Masque | `255.255.255.0` |
| Passerelle - Option 003 | `192.168.2.1` |
| DNS - Option 006 | `192.168.1.1` |

Cette étendue permet de distribuer trois adresses IP sur le réseau Comptabilité.

La passerelle correspond à l'interface OPT1 d'OPNsense.

### 5.4. Création de l'étendue Commercial

J'ai ensuite créé une troisième étendue pour les postes Commercial.

| Paramètre | Configuration |
|-----------|---------------|
| Réseau | `192.168.3.0/24` |
| Adresse de début | `192.168.3.2` |
| Adresse de fin | `192.168.3.4` |
| Masque | `255.255.255.0` |
| Passerelle - Option 003 | `192.168.3.1` |
| DNS - Option 006 | `192.168.1.1` |

Cette étendue permet également de distribuer trois adresses IP.

La passerelle correspond cette fois à l'interface OPT2 d'OPNsense.

### 5.5. Comprendre les différentes passerelles

J'ai compris qu'il ne fallait pas simplement renseigner la passerelle du serveur DHCP dans toutes les étendues.

Chaque client doit recevoir la passerelle de **son propre réseau**.

| Réseau | Passerelle |
|--------|------------|
| Serveur | `192.168.1.254` |
| Comptabilité | `192.168.2.1` |
| Commercial | `192.168.3.1` |

En revanche, les trois réseaux peuvent utiliser le même serveur DNS : `192.168.1.1`.

Le serveur DHCP reste également unique, même s'il gère plusieurs étendues.

*Capture à ajouter : console DHCP Windows montrant les étendues configurées.*

*Capture à ajouter : options d'étendue Comptabilité avec la passerelle et le DNS.*

*Capture à ajouter : options d'étendue Commercial avec la passerelle et le DNS.*

## 6. Raccordement des machines aux bons réseaux

Après avoir configuré les réseaux et les interfaces, j'ai vérifié l'association des machines virtuelles dans Virt-Manager.

| Machine | Réseau virtuel |
|---------|---------------|
| SERVER-DHCP | Serveur |
| ROUTAGE - LAN | Serveur |
| ROUTAGE - OPT1 | COMPTABILITE |
| ROUTAGE - OPT2 | COMMERCIAL |
| COMPTA-DEVICE1 | COMPTABILITE pour le premier test |

Cette vérification est importante : même si les adresses IP sont correctement configurées, une VM connectée au mauvais réseau virtuel ne pourra pas fonctionner comme prévu.

Pour tester Commercial, j'ai ensuite déplacé **COMPTA-DEVICE1** sur ce réseau.

Cela m'a permis de tester les deux réseaux sans installer immédiatement une deuxième VM Windows 11.

*Capture à ajouter : configuration de la carte réseau de COMPTA-DEVICE1 dans Virt-Manager.*

## 7. Configuration du relais DHCP sur OPNsense

Une nouvelle difficulté apparaît lorsqu'un serveur DHCP doit distribuer des adresses sur plusieurs sous-réseaux.

Une demande DHCP initiale est envoyée en diffusion sur le réseau local du client. Un routeur ne transmet normalement pas directement cette diffusion vers les autres réseaux.

J'ai donc découvert l'utilité du **relais DHCP**.

Son rôle est de transmettre les demandes DHCP des postes Comptabilité et Commercial vers le serveur Windows situé sur le réseau Serveur.

### Configuration dans OPNsense

Depuis :

**Services → DHCRelay**

J'ai configuré une destination correspondant au serveur DHCP Windows.

| Paramètre | Configuration |
|-----------|---------------|
| Destination DHCP | `192.168.1.1` |

J'ai ensuite associé cette destination aux interfaces des réseaux clients.

| Interface | Destination |
|-----------|-------------|
| COMPTABILITE - OPT1 | Serveur DHCP `192.168.1.1` |
| COMMERCIAL - OPT2 | Serveur DHCP `192.168.1.1` |

### Fonctionnement attendu

Lorsqu'un poste Comptabilité demande une adresse IP :

1. La demande DHCP est envoyée sur le réseau Comptabilité.
2. OPNsense la reçoit sur l'interface OPT1.
3. Le relais DHCP transmet la demande à `192.168.1.1`.
4. Windows Server utilise l'étendue correspondant au réseau Comptabilité.
5. Le client reçoit sa configuration IP par l'intermédiaire du relais.

Le fonctionnement est identique pour Commercial, en passant cette fois par OPT2.

À ce stade, les interfaces et la destination étaient configurées, mais j'avais oublié un élément important : **l'activation des relais DHCP**.

Je ne l'ai découvert qu'au moment des tests.

*Capture à ajouter : destination du relais DHCP dans OPNsense.*

*Capture à ajouter : relais Comptabilité et Commercial, dans leur état final activé.*

## 8. Premier test et apparition du problème

Après avoir terminé la configuration, j'ai effectué un premier test depuis la VM **COMPTA-DEVICE1**.

Le poste était connecté au réseau Comptabilité, avec sa carte réseau Windows configurée pour obtenir automatiquement une adresse IP.

Depuis l'invite de commandes, j'ai exécuté :

```cmd
ipconfig /release
ipconfig /renew
```

**Résultat : le poste ne recevait pas l'adresse attendue depuis Windows Server.**

En vérifiant sa configuration avec :

```cmd
ipconfig /all
```

J'ai constaté qu'il utilisait une adresse IPv4 de type :

`169.254.x.x`

### Comprendre l'adresse APIPA

Une adresse commençant par `169.254` peut être attribuée automatiquement par Windows lorsqu'un poste configuré en DHCP ne parvient pas à obtenir une adresse depuis un serveur DHCP.

C'est ce qu'on appelle une adresse **APIPA**.

Cette information m'indiquait que le client n'avait pas obtenu le bail DHCP attendu.

Je devais donc vérifier pourquoi la demande n'aboutissait pas.

*Capture facultative à ajouter : résultat du premier test avec une adresse APIPA, uniquement si une capture a été conservée.*

## 9. Recherche et résolution du problème

Face à ce problème, j'ai choisi de reprendre les vérifications depuis la base de mon infrastructure.

### Vérification des réseaux virtuels

J'ai d'abord vérifié dans Virt-Manager :

- Que les réseaux virtuels isolés étaient correctement créés.
- Que les cartes réseau d'OPNsense étaient associées aux bons réseaux.
- Que COMPTA-DEVICE1 était connecté au réseau Comptabilité.
- Que le service DHCP de libvirt ne distribuait pas d'adresses à la place de Windows Server.

### Vérification du serveur DHCP

Je suis ensuite retourné sur **SERVER-DHCP** pour contrôler :

- Son adresse IPv4 fixe.
- Son masque de sous-réseau.
- Sa passerelle.
- La configuration des étendues DHCP.
- Les plages d'adresses prévues.
- L'activation des étendues.
- Les passerelles et les adresses DNS distribuées aux clients.

Les paramètres vérifiés correspondaient à la nouvelle architecture.

### Vérification d'OPNsense

Enfin, j'ai repris la configuration du routeur.

J'ai vérifié les adresses IPv4 de ses trois interfaces :

| Interface | Adresse |
|-----------|---------|
| LAN | `192.168.1.254/24` |
| COMPTABILITE | `192.168.2.1/24` |
| COMMERCIAL | `192.168.3.1/24` |

J'ai ensuite contrôlé la configuration du relais DHCP.

La destination `192.168.1.1` était correcte et les interfaces Comptabilité et Commercial étaient bien associées.

C'est à ce moment-là que j'ai remarqué **que les relais DHCP étaient configurés, mais désactivés**.

Le problème ne venait donc pas des plages DHCP ou des adresses des interfaces : j'avais simplement oublié d'activer les relais.

### Correction

J'ai activé les relais DHCP sur OPNsense, puis appliqué la configuration.

J'ai ensuite repris les tests pour vérifier si cette modification résolvait le problème.

## 10. Validation sur le réseau Comptabilité

Après l'activation des relais, j'ai refait une demande DHCP depuis **COMPTA-DEVICE1**, toujours connecté au réseau Comptabilité.

Depuis Windows 11 :

```cmd
ipconfig /release
ipconfig /renew
ipconfig /all
```

Cette fois, le poste reçoit une configuration correspondant au réseau `192.168.2.0/24`.

### Configuration attendue

| Paramètre | Valeur |
|-----------|--------|
| Adresse IPv4 | `192.168.2.2` à `192.168.2.4` |
| Masque | `255.255.255.0` |
| Passerelle | `192.168.2.1` |
| Serveur DHCP | `192.168.1.1` |
| DNS | `192.168.1.1` |

**Résultat : attribution DHCP fonctionnelle sur Comptabilité.**

Le poste reçoit désormais une adresse correspondant à l'étendue configurée sur Windows Server, alors qu'il se trouve sur un autre réseau.

*Capture à ajouter : `ipconfig /all` de COMPTA-DEVICE1 sur Comptabilité, avec l'adresse IPv4, la passerelle et le serveur DHCP.*

## 11. Validation sur le réseau Commercial

Pour tester la deuxième étendue, j'ai utilisé la même VM Windows 11.

Depuis Virt-Manager :

1. Arrêt de **COMPTA-DEVICE1**.
2. Modification de sa carte réseau virtuelle.
3. Remplacement du réseau **COMPTABILITE** par **COMMERCIAL**.
4. Redémarrage de Windows 11.

Le client est toujours configuré pour obtenir automatiquement son adresse IP.

J'ai ensuite contrôlé sa configuration réseau avec :

```cmd
ipconfig /all
```

### Configuration attendue

| Paramètre | Valeur |
|-----------|--------|
| Adresse IPv4 | `192.168.3.2` à `192.168.3.4` |
| Masque | `255.255.255.0` |
| Passerelle | `192.168.3.1` |
| Serveur DHCP | `192.168.1.1` |
| DNS | `192.168.1.1` |

**Résultat : attribution DHCP fonctionnelle sur Commercial.**

Cette vérification confirme que le serveur DHCP Windows peut distribuer les adresses de deux réseaux différents grâce au relais DHCP d'OPNsense.

Pour réaliser cette validation, je n'ai pas eu besoin d'installer un deuxième poste client : j'ai utilisé successivement la même VM sur les deux réseaux.

*Capture à ajouter : `ipconfig /all` de COMPTA-DEVICE1 sur Commercial, avec l'adresse IPv4, la passerelle et le serveur DHCP.*

## 12. Résultat de cette étape

L'infrastructure initiale a été modifiée pour accueillir plusieurs réseaux distincts.

La nouvelle organisation repose sur :

- Un **serveur DHCP Windows Server 2022** sur `192.168.1.1`.
- Un **routeur OPNsense** possédant trois interfaces réseau.
- Un réseau **Serveur** en `192.168.1.0/24`.
- Un réseau **Comptabilité** en `192.168.2.0/24`.
- Un réseau **Commercial** en `192.168.3.0/24`.
- Un **relais DHCP** permettant aux postes des différents réseaux de recevoir leur configuration depuis un serveur unique.

Les tests effectués sur Comptabilité et Commercial ont permis de valider l'attribution automatique des adresses IP.

**L'objectif de cette évolution est donc atteint : le serveur DHCP fonctionne désormais avec plusieurs sous-réseaux.**

Le renforcement des règles de pare-feu et le filtrage des communications entre les réseaux constituent des possibilités d'amélioration pour une prochaine étape.

## 13. Ce que j'ai appris

Cette nouvelle partie du projet m'a permis de découvrir et de mettre en pratique :

- L'installation et la première configuration d'un **routeur OPNsense**.
- La création de plusieurs **réseaux virtuels isolés** sous KVM.
- L'utilisation de plusieurs cartes réseau sur une même machine virtuelle.
- Le rôle des **passerelles** dans une infrastructure composée de plusieurs sous-réseaux.
- La différence entre un **serveur DHCP** et un **relais DHCP**.
- La configuration de plusieurs **étendues DHCP** sur Windows Server.
- La signification d'une adresse **APIPA**.
- L'importance de vérifier méthodiquement la configuration et l'activation des services lorsqu'un problème apparaît.

J'ai également compris qu'une infrastructure fonctionnelle pouvait être **modifiée et faire l'objet d'une évolution**, sans devoir repartir de zéro.

Au départ, je cherchais simplement à faire fonctionner un serveur DHCP avec un poste client.

En me demandant comment ajouter un deuxième poste sur un réseau différent, j'ai découvert de nouvelles notions et décidé de revoir l'organisation de mon homelab pour les appliquer.

Cette expérience m'a permis de progresser dans ma compréhension de l'administration systèmes et réseaux.

---

<sub>**Ressources :** Tutoriels et documentation technique, recherches personnelles et utilisation de l'intelligence artificielle pour comprendre les notions de routage, de sous-réseaux et de relais DHCP, accompagner la configuration et résoudre les difficultés rencontrées.</sub>

