
# 06 - Évolution de l'infrastructure : OPNsense et relais DHCP

## Objectif

Après avoir réussi à mettre en place mon premier serveur DHCP et validé son fonctionnement avec un poste client Windows 11, j'ai voulu aller plus loin.

Jusqu'ici, mon infrastructure était composée de deux machines virtuelles, **SERVER-DHCP** et **COMPTA-DEVICE1**, connectées au même réseau isolé.

Je me suis alors posé une nouvelle question : **si je voulais ajouter un deuxième poste client, mais sur un réseau différent du premier, comment les deux machines pourraient-elles recevoir une adresse IP depuis le même serveur DHCP ?**

Après quelques recherches, j'ai découvert qu'il était possible de mettre en place plusieurs sous-réseaux, reliés par un routeur, et d'utiliser un **relais DHCP** pour transmettre les demandes des clients au serveur Windows.

J'ai donc décidé de faire évoluer mon infrastructure initiale plutôt que de repartir de zéro.

L'objectif de cette nouvelle étape est de créer **trois réseaux distincts** :

- Un réseau réservé au serveur DHCP et au routeur.
- Un réseau pour les postes Comptabilité.
- Un réseau pour les postes Commercial.

Pour les tests, je réutiliserai ma VM Windows 11 en la connectant successivement aux deux réseaux clients.

## 1. Modification de l'architecture réseau

Dans la première version du projet, le serveur DHCP et le poste Windows 11 communiquaient directement sur le réseau `192.168.1.0/24`.

Pour accueillir deux réseaux clients distincts, j'ai choisi de revoir cette organisation.

### Architecture retenue

| Réseau | Sous-réseau | Utilisation |
|--------|-------------|-------------|
| Serveur | `192.168.1.0/24` | Windows Server et routeur |
| Comptabilité | `192.168.2.0/24` | Postes Comptabilité |
| Commercial | `192.168.3.0/24` | Postes Commercial |

Le serveur DHCP conserve son adresse fixe `192.168.1.1`.

Le réseau `192.168.1.0/24` n'est désormais plus destiné aux postes clients. Ces derniers seront placés sur les réseaux Comptabilité et Commercial.

Pour relier les trois sous-réseaux, il me fallait donc ajouter un routeur à mon infrastructure.

*Capture à ajouter : schéma de la nouvelle architecture montrant Windows Server, le routeur et les deux réseaux clients.*

## 2. Création de la VM OPNsense

Pour relier mes différents réseaux, je devais maintenant installer mon premier routeur.

Après quelques recherches, j'ai choisi **OPNsense**, une solution gratuite et open source qui me semblait adaptée à mon projet, notamment grâce à son interface web et à la possibilité de configurer un relais DHCP.

J'ai donc créé une nouvelle VM sous **Virt-Manager**, nommée **ROUTAGE**, puis installé OPNsense.

### Configuration de la VM

| Paramètre | Configuration |
|-----------|---------------|
| Nom | ROUTAGE |
| Système | OPNsense |
| Virtualisation | KVM/QEMU |
| Gestion | Virt-Manager |

Après l'installation, je suis passé à la création des réseaux nécessaires et au raccordement du routeur.

*Capture à ajouter : VM ROUTAGE dans Virt-Manager.*

## 3. Création des réseaux virtuels isolés

Dans la première partie du projet, j'avais créé un réseau isolé pour permettre à SERVER-DHCP et COMPTA-DEVICE1 de communiquer sans utiliser le DHCP fourni par libvirt.

Avec cette nouvelle architecture, il fallait maintenant séparer les machines en plusieurs réseaux.

Depuis **Virt-Manager**, j'ai donc créé les réseaux virtuels nécessaires pour Comptabilité et Commercial, tout en conservant un réseau dédié au serveur.

### Réseaux utilisés

| Réseau virtuel | Machines concernées |
|----------------|---------------------|
| Serveur | SERVER-DHCP et ROUTAGE |
| COMPTABILITE | ROUTAGE et postes Comptabilité |
| COMMERCIAL | ROUTAGE et postes Commercial |

J'ai également veillé à désactiver le **DHCP de libvirt** sur les réseaux clients.

Cette précaution faisait suite au problème rencontré dans les premiers chapitres : le DHCP intégré à KVM avait attribué une adresse IP à mon poste à la place du serveur Windows.

### Ajout des cartes réseau sur OPNsense

Pour connecter le routeur aux trois réseaux, j'ai ajouté **deux cartes réseau supplémentaires** à la VM ROUTAGE.

Elle dispose donc de trois cartes au total.

| Carte | Interface OPNsense | Réseau virtuel |
|-------|--------------------|---------------|
| Carte 1 | LAN (`vtnet0`) | Serveur |
| Carte 2 | OPT1 (`vtnet1`) | COMPTABILITE |
| Carte 3 | OPT2 (`vtnet2`) | COMMERCIAL |

Chaque carte est associée au réseau correspondant depuis les paramètres de la VM dans Virt-Manager.

*Capture à ajouter : réseaux isolés créés dans Virt-Manager, avec le DHCP libvirt désactivé.*

*Capture à ajouter : les trois cartes réseau de la VM ROUTAGE et leurs réseaux associés.*

## 4. Configuration des interfaces OPNsense

Une fois les cartes réseau raccordées, j'ai poursuivi la configuration directement dans OPNsense.

J'ai activé les interfaces correspondant aux réseaux Comptabilité et Commercial, puis configuré une **adresse IPv4 fixe** pour chacune des trois interfaces.

### Configuration retenue

| Interface | Réseau | Adresse IPv4 | Masque |
|-----------|--------|--------------|--------|
| LAN | Serveur | `192.168.1.254` | `255.255.255.0` |
| OPT1 - COMPTABILITE | Comptabilité | `192.168.2.1` | `255.255.255.0` |
| OPT2 - COMMERCIAL | Commercial | `192.168.3.1` | `255.255.255.0` |

Ces adresses ont été choisies pour servir de **passerelles** aux machines de chaque réseau.

Ainsi, le serveur Windows utilisera `192.168.1.254`, tandis que les futurs postes Comptabilité et Commercial utiliseront respectivement `192.168.2.1` et `192.168.3.1`.

Une fois les interfaces configurées, j'ai pu accéder à l'administration web d'OPNsense depuis le serveur Windows à l'adresse :

`https://192.168.1.254`

*Capture à ajouter : configuration des trois interfaces réseau dans OPNsense, avec leurs adresses IPv4.*

## 5. Adaptation du serveur DHCP Windows

La nouvelle organisation nécessitait également de modifier la configuration de **SERVER-DHCP**.

Dans la première version de mon infrastructure, l'étendue DHCP distribuait des adresses comprises entre `192.168.1.2` et `192.168.1.4`.

Cette plage était adaptée au premier projet, puisque le serveur et le poste client se trouvaient sur le même réseau.

Avec l'ajout des réseaux Comptabilité et Commercial, cette étendue n'est plus destinée à fournir des adresses aux postes clients.

J'ai donc revu la configuration du serveur Windows et ajouté deux nouvelles étendues.

### 5.1. Modification de la configuration IP du serveur

SERVER-DHCP conserve son adresse IPv4 fixe `192.168.1.1`.

J'ai cependant ajouté une passerelle à sa configuration, puisque le serveur doit maintenant pouvoir communiquer avec des machines situées sur d'autres réseaux.

| Paramètre | Configuration |
|-----------|---------------|
| Adresse IPv4 | `192.168.1.1` |
| Masque de sous-réseau | `255.255.255.0` |
| Passerelle | `192.168.1.254` |
| DNS préféré | `192.168.1.1` |

La passerelle correspond à l'interface LAN d'OPNsense, connectée au même réseau que Windows Server.

### 5.2. Adaptation de l'étendue initiale

L'étendue du réseau `192.168.1.0/24` avait été créée pour les premiers tests du DHCP.

Avec la nouvelle architecture, les postes clients utilisent désormais les réseaux `192.168.2.0/24` et `192.168.3.0/24`.

J'ai donc adapté les options du réseau Serveur, notamment l'option **003 Routeur**, correspondant à la passerelle `192.168.1.254`.

L'ancienne plage `192.168.1.2` à `192.168.1.4` n'est plus utilisée pour tester les postes Comptabilité et Commercial.

### 5.3. Création de l'étendue Comptabilité

Depuis la console DHCP de Windows Server, j'ai créé une nouvelle étendue pour le réseau Comptabilité.

| Paramètre | Configuration |
|-----------|---------------|
| Réseau | `192.168.2.0/24` |
| Adresse de début | `192.168.2.2` |
| Adresse de fin | `192.168.2.4` |
| Masque | `255.255.255.0` |
| Passerelle - Option 003 | `192.168.2.1` |
| DNS - Option 006 | `192.168.1.1` |

Cette étendue permet de distribuer trois adresses IP aux postes Comptabilité.

J'ai renseigné `192.168.2.1` comme passerelle, puisqu'il s'agit de l'adresse d'OPNsense sur ce réseau.

### 5.4. Création de l'étendue Commercial

J'ai ensuite créé une troisième étendue pour le réseau Commercial.

| Paramètre | Configuration |
|-----------|---------------|
| Réseau | `192.168.3.0/24` |
| Adresse de début | `192.168.3.2` |
| Adresse de fin | `192.168.3.4` |
| Masque | `255.255.255.0` |
| Passerelle - Option 003 | `192.168.3.1` |
| DNS - Option 006 | `192.168.1.1` |

Cette étendue permet également de distribuer trois adresses IP.

La passerelle `192.168.3.1` correspond à l'interface Commercial d'OPNsense.

Les deux étendues utilisent le même serveur DNS, `192.168.1.1`, et seront gérées par le même serveur DHCP Windows.

*Capture à ajouter : console DHCP Windows avec les étendues configurées et leur état d'activation.*

*Capture à ajouter : options d'étendue Comptabilité montrant la passerelle et le DNS.*

*Capture à ajouter : options d'étendue Commercial montrant la passerelle et le DNS.*

## 6. Raccordement des machines aux réseaux isolés

Après avoir configuré les réseaux, le routeur et les étendues DHCP, j'ai vérifié que chaque machine virtuelle était connectée au bon réseau dans Virt-Manager.

| Machine | Réseau virtuel |
|---------|---------------|
| SERVER-DHCP | Serveur |
| ROUTAGE - LAN | Serveur |
| ROUTAGE - OPT1 | COMPTABILITE |
| ROUTAGE - OPT2 | COMMERCIAL |
| COMPTA-DEVICE1 | COMPTABILITE pour le premier test |

Dans la première version du projet, COMPTA-DEVICE1 était directement connecté au même réseau que SERVER-DHCP.

Je l'ai donc déplacé sur le réseau **COMPTABILITE** afin de tester l'attribution DHCP depuis un autre sous-réseau.

Pour tester Commercial, je prévoyais d'utiliser la même VM en changeant simplement son réseau virtuel.

*Capture à ajouter : carte réseau de COMPTA-DEVICE1 associée au réseau COMPTABILITE dans Virt-Manager.*

## 7. Configuration du relais DHCP dans OPNsense

Après avoir créé les étendues et raccordé les machines, il restait à permettre aux postes des réseaux Comptabilité et Commercial de contacter le serveur DHCP Windows.

Lors de mes recherches, j'avais découvert que les demandes DHCP initiales ne traversaient pas directement les routeurs.

J'ai donc utilisé le **relais DHCP** d'OPNsense pour transmettre ces demandes vers SERVER-DHCP.

### Configuration

Depuis l'interface web d'OPNsense :

**Services → DHCRelay**

J'ai commencé par définir une destination correspondant à l'adresse de mon serveur Windows.

| Paramètre | Configuration |
|-----------|---------------|
| Destination | `192.168.1.1` |

J'ai ensuite créé les relais associés aux interfaces des réseaux clients.

| Interface | Destination |
|-----------|-------------|
| COMPTABILITE - OPT1 | Serveur DHCP `192.168.1.1` |
| COMMERCIAL - OPT2 | Serveur DHCP `192.168.1.1` |

L'objectif était que les demandes provenant de ces deux réseaux soient transmises à SERVER-DHCP, qui pourrait ensuite distribuer une adresse depuis l'étendue correspondante.

À ce stade, les paramètres du relais étaient renseignés. Cependant, je n'avais pas encore remarqué que les relais étaient désactivés.

*Capture à ajouter : destination `192.168.1.1` dans DHCRelay.*

*Capture à ajouter : relais Comptabilité et Commercial dans OPNsense, de préférence après leur activation.*

## 8. Premiers tests et difficultés

Une fois la configuration terminée, j'ai voulu vérifier que **COMPTA-DEVICE1** pouvait recevoir une adresse IP depuis l'étendue Comptabilité.

Le poste était connecté au réseau COMPTABILITE et sa carte réseau Windows était configurée pour obtenir automatiquement une adresse IPv4.

Depuis l'invite de commandes, j'ai exécuté :

```cmd
ipconfig /release
ipconfig /renew
```

**Résultat : le client ne recevait pas l'adresse attendue.**

Pour vérifier sa configuration, j'ai utilisé :

```cmd
ipconfig /all
```

J'ai constaté qu'il possédait une adresse IPv4 commençant par `169.254`.

Après m'être renseigné sur cette adresse, j'ai découvert qu'il s'agissait d'une adresse **APIPA**, attribuée automatiquement par Windows lorsqu'un poste configuré en DHCP ne parvient pas à obtenir une adresse depuis un serveur DHCP.

Cette information confirmait que le poste ne recevait pas son bail DHCP depuis SERVER-DHCP.

Il fallait donc reprendre les vérifications pour trouver l'origine du problème.

*Capture facultative à ajouter : premier test DHCP montrant une adresse APIPA, si une capture a été conservée.*

## 9. Recherche et résolution du problème

Face à cet échec, j'ai décidé de reprendre les vérifications depuis les éléments de base de mon infrastructure.

### Vérification des réseaux virtuels

J'ai commencé par vérifier la configuration dans Virt-Manager :

- Présence des réseaux virtuels isolés.
- Association des cartes réseau d'OPNsense aux bons réseaux.
- Raccordement de COMPTA-DEVICE1 au réseau COMPTABILITE.
- Désactivation du service DHCP de libvirt sur les réseaux clients.

Je voulais m'assurer que le problème ne provenait pas d'une erreur de raccordement entre les machines virtuelles.

### Vérification du serveur DHCP

Je suis ensuite retourné sur Windows Server pour vérifier :

- Son adresse IPv4 fixe et sa passerelle.
- Les plages d'adresses des étendues.
- Les masques de sous-réseau.
- Les passerelles configurées dans les options DHCP.
- Les adresses DNS.
- L'activation des étendues.

Les paramètres vérifiés correspondaient à la configuration prévue.

### Vérification d'OPNsense

Enfin, j'ai repris la configuration du routeur.

J'ai d'abord contrôlé les adresses IPv4 de ses interfaces.

| Interface | Adresse IPv4 |
|-----------|--------------|
| LAN | `192.168.1.254/24` |
| COMPTABILITE | `192.168.2.1/24` |
| COMMERCIAL | `192.168.3.1/24` |

Je suis ensuite retourné dans **Services → DHCRelay**.

La destination `192.168.1.1` était correctement renseignée et les relais étaient bien associés aux interfaces Comptabilité et Commercial.

**C'est à ce moment-là que j'ai remarqué que les relais DHCP étaient configurés, mais qu'ils n'étaient pas activés.**

J'avais simplement oublié de les mettre en service.

### Correction

J'ai activé les deux relais DHCP depuis OPNsense et appliqué la configuration.

J'ai ensuite repris les tests pour vérifier si le problème était résolu.

## 10. Validation du réseau Comptabilité

Après l'activation du relais DHCP, j'ai effectué un nouveau test depuis COMPTA-DEVICE1, toujours connecté au réseau COMPTABILITE.

Depuis l'invite de commandes :

```cmd
ipconfig /release
ipconfig /renew
ipconfig /all
```

Cette fois, le poste reçoit une adresse IPv4 correspondant à l'étendue Comptabilité.

### Configuration attendue

| Paramètre | Valeur |
|-----------|--------|
| Adresse IPv4 | `192.168.2.2` à `192.168.2.4` |
| Masque | `255.255.255.0` |
| Passerelle | `192.168.2.1` |
| Serveur DHCP | `192.168.1.1` |
| DNS | `192.168.1.1` |

**Résultat : le DHCP fonctionne désormais sur le réseau Comptabilité.**

Le poste reçoit bien sa configuration depuis Windows Server, alors que les deux machines ne sont plus connectées au même réseau.

*Capture à ajouter : résultat de `ipconfig /all` sur COMPTA-DEVICE1, montrant une adresse `192.168.2.x`, la passerelle et le serveur DHCP.*

## 11. Validation du réseau Commercial

Pour valider la deuxième étendue, j'ai décidé de tester également le réseau Commercial.

Plutôt que d'installer immédiatement une deuxième VM Windows 11, j'ai utilisé COMPTA-DEVICE1 pour simuler un poste connecté à ce réseau.

Depuis Virt-Manager :

1. Arrêt de COMPTA-DEVICE1.
2. Ouverture des paramètres de sa carte réseau.
3. Modification du réseau virtuel de COMPTABILITE vers COMMERCIAL.
4. Redémarrage de Windows 11.

La carte réseau Windows était toujours configurée pour obtenir automatiquement une adresse IP.

J'ai ensuite vérifié sa configuration avec :

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

**Résultat : le DHCP fonctionne également sur le réseau Commercial.**

Ce deuxième test confirme que SERVER-DHCP peut distribuer des adresses IP à des postes situés sur deux sous-réseaux distincts grâce au relais DHCP d'OPNsense.

*Capture à ajouter : résultat de `ipconfig /all` sur COMPTA-DEVICE1 connecté au réseau COMMERCIAL, montrant une adresse `192.168.3.x`, la passerelle et le serveur DHCP.*

## 12. Résultat de cette étape

L'infrastructure initiale, composée d'un serveur DHCP et d'un poste client sur le même réseau, a été adaptée pour fonctionner avec plusieurs sous-réseaux.

La nouvelle configuration comprend :

- Un serveur DHCP Windows Server 2022 sur le réseau `192.168.1.0/24`.
- Un routeur OPNsense disposant de trois interfaces réseau.
- Un réseau Comptabilité en `192.168.2.0/24`.
- Un réseau Commercial en `192.168.3.0/24`.
- Un relais DHCP transmettant les demandes des deux réseaux clients vers SERVER-DHCP.

Les tests réalisés avec COMPTA-DEVICE1 montrent que les postes peuvent recevoir automatiquement une adresse IP appartenant au réseau auquel ils sont connectés.

**L'objectif de cette évolution est atteint : un seul serveur DHCP Windows distribue désormais des adresses sur plusieurs réseaux grâce à OPNsense.**

Le renforcement des règles de pare-feu et le filtrage des communications entre les sous-réseaux pourront être étudiés dans une prochaine étape.

## 13. Ce que j'ai appris

Cette évolution du projet m'a permis de découvrir et de mettre en pratique :

- La création et la première configuration d'un routeur OPNsense.
- L'organisation d'une infrastructure avec plusieurs réseaux virtuels isolés.
- Le raccordement d'une même VM routeur à plusieurs réseaux.
- La configuration des passerelles et des étendues DHCP pour différents sous-réseaux.
- L'utilisation d'un relais DHCP pour conserver un serveur DHCP unique.
- L'identification d'une adresse APIPA lors d'un échec d'attribution DHCP.
- L'importance de vérifier la configuration et l'activation des services lors d'un dépannage.

Le problème rencontré m'a notamment appris qu'un service peut être correctement configuré sans être activé.

J'ai également compris qu'il était possible de **faire évoluer une infrastructure déjà fonctionnelle**, en réorganisant ses réseaux et en adaptant les services existants, sans repartir de zéro.

Au départ, je voulais simplement faire fonctionner mon premier serveur DHCP avec un poste client.

C'est en me demandant comment ajouter un deuxième poste sur un réseau différent que j'ai découvert de nouvelles notions et décidé de les mettre en pratique dans mon homelab.

---

<sub>**Ressources :** Documentation technique, tutoriels en ligne et utilisation de l'intelligence artificielle pour comprendre les notions de routage et de relais DHCP, accompagner la configuration et résoudre les difficultés rencontrées.</sub>
