
# 06 - Évolution de l'infrastructure : OPNsense et relais DHCP

## Objectif

Après avoir réussi à configurer mon premier serveur DHCP sous **Windows Server 2022** et validé son fonctionnement avec un poste client Windows 11, j'ai voulu aller plus loin.

Jusqu'ici, mon infrastructure était composée de deux machines virtuelles, **SERVER-DHCP** et **COMPTA-DEVICE1**, connectées au même réseau virtuel isolé.

Je me suis alors posé une nouvelle question : **si je voulais ajouter un deuxième poste client, mais sur un réseau différent du premier, comment les deux machines pourraient-elles recevoir leurs adresses IP depuis le même serveur DHCP ?**

Après quelques recherches, j'ai découvert les notions de **routage** et de **relais DHCP**.

J'ai donc décidé de faire évoluer mon infrastructure existante, en conservant mon serveur Windows et en ajoutant un routeur pour relier plusieurs réseaux.

## 1. Nouvelle architecture réseau

Dans les étapes précédentes, le serveur DHCP et le poste client communiquaient directement sur le réseau `192.168.1.0/24`.

Pour aller plus loin, j'ai choisi de séparer les machines en trois réseaux distincts :

| Réseau | Sous-réseau | Utilisation |
|--------|-------------|-------------|
| Serveur | `192.168.1.0/24` | Serveur DHCP et routeur |
| Comptabilité | `192.168.2.0/24` | Postes Comptabilité |
| Commercial | `192.168.3.0/24` | Postes Commercial |

Le serveur DHCP conserve son adresse fixe `192.168.1.1`.

En revanche, les postes clients ne seront plus directement connectés à son réseau. Ils utiliseront désormais les réseaux Comptabilité et Commercial.

Pour permettre au serveur DHCP de fonctionner avec ces différents réseaux, j'ai prévu d'ajouter une machine virtuelle dédiée au routage.

*Capture à ajouter : schéma de la nouvelle architecture, avec SERVER-DHCP, OPNsense et les deux réseaux clients. Indiquer les adresses IP des interfaces du routeur.*

## 2. Création de la VM OPNsense

Pour relier mes différents réseaux, je devais maintenant installer mon premier routeur.

Après quelques recherches, j'ai choisi **OPNsense**, une solution gratuite et open source qui me semblait adaptée à mon projet, notamment grâce à son interface web et à la possibilité de configurer un relais DHCP.

Depuis **Virt-Manager**, j'ai donc créé une nouvelle machine virtuelle, nommée **ROUTAGE**, à partir de l'image d'installation d'OPNsense.

### Configuration de la VM

| Paramètre | Configuration |
|-----------|---------------|
| Nom | ROUTAGE |
| Système | OPNsense |
| RAM | À compléter |
| CPU | À compléter |
| Stockage | À compléter |

Les ressources ont été adaptées aux capacités de mon PC hôte, qui dispose de 8 Go de RAM et fait déjà fonctionner les deux VM Windows du projet.

Après la création de la machine virtuelle, j'ai procédé à l'installation d'OPNsense.

*Capture à ajouter : VM ROUTAGE dans Virt-Manager, avec ses ressources matérielles.*

## 3. Première configuration d'OPNsense

Après l'installation, j'ai découvert la console d'administration d'OPNsense.

Pour commencer, il fallait configurer son interface réseau principale afin qu'elle puisse communiquer avec mon serveur Windows.

J'ai utilisé le réseau `192.168.1.0/24`, déjà présent dans mon infrastructure.

### Configuration initiale

Depuis la console OPNsense, j'ai configuré l'interface LAN avec une adresse IPv4 fixe :

| Paramètre | Configuration |
|-----------|---------------|
| Interface | LAN |
| Adresse IPv4 | `192.168.1.254` |
| Masque | `255.255.255.0` |
| Réseau | `192.168.1.0/24` |

J'ai choisi `192.168.1.254` pour disposer d'une adresse de routeur sur le même réseau que **SERVER-DHCP (`192.168.1.1`)**, sans créer de conflit d'adressage.

Cette première configuration permettra ensuite d'accéder à l'interface web d'OPNsense depuis Windows Server.

*Capture à ajouter : console OPNsense montrant l'adresse LAN `192.168.1.254/24`.*

## 4. Création des réseaux virtuels sous KVM

Dans la première version du projet, j'avais créé un réseau isolé pour relier SERVER-DHCP et COMPTA-DEVICE1.

Avec la nouvelle architecture, il fallait désormais séparer les futurs postes Comptabilité et Commercial.

Depuis **Virt-Manager**, j'ai donc créé les réseaux virtuels isolés nécessaires.

### Réseaux utilisés

| Réseau | Rôle |
|--------|------|
| Réseau Serveur | Communication entre SERVER-DHCP et ROUTAGE |
| COMPTABILITE | Réseau des postes Comptabilité |
| COMMERCIAL | Réseau des postes Commercial |

J'ai également veillé à désactiver le **service DHCP de libvirt** sur les réseaux clients.

Lors de la première partie du projet, j'avais déjà rencontré un problème lié au DHCP intégré à KVM : le poste Windows recevait une adresse provenant de libvirt, et non de mon serveur Windows.

Je voulais donc éviter de reproduire cette situation avec les nouveaux réseaux.

### Ajout des cartes réseau sur ROUTAGE

La VM OPNsense devait maintenant être reliée aux trois réseaux.

Depuis les paramètres matériels de la VM dans Virt-Manager, j'ai ajouté **deux cartes réseau supplémentaires**, pour obtenir trois cartes au total.

| Carte réseau | Interface OPNsense | Réseau |
|--------------|--------------------|--------|
| Carte 1 | LAN (`vtnet0`) | Serveur |
| Carte 2 | OPT1 (`vtnet1`) | COMPTABILITE |
| Carte 3 | OPT2 (`vtnet2`) | COMMERCIAL |

Chaque carte réseau a ensuite été associée au réseau virtuel correspondant.

*Capture à ajouter : réseaux virtuels isolés dans Virt-Manager, avec le DHCP de libvirt désactivé.*

*Capture à ajouter : les trois cartes réseau de la VM ROUTAGE et les réseaux auxquels elles sont associées.*

## 5. Raccordement des VM et configuration des interfaces

Une fois les réseaux créés, j'ai vérifié le raccordement des différentes machines virtuelles dans Virt-Manager.

### Association des machines

| Machine ou interface | Réseau virtuel |
|----------------------|---------------|
| SERVER-DHCP | Réseau Serveur |
| ROUTAGE - LAN | Réseau Serveur |
| ROUTAGE - OPT1 | COMPTABILITE |
| ROUTAGE - OPT2 | COMMERCIAL |
| COMPTA-DEVICE1 | COMPTABILITE |

Dans les chapitres précédents, COMPTA-DEVICE1 était connecté au même réseau que SERVER-DHCP.

Je l'ai donc déplacé sur **COMPTABILITE** afin de tester le fonctionnement du DHCP depuis un autre sous-réseau.

Pour le réseau Commercial, j'ai prévu d'utiliser cette même VM en modifiant temporairement son réseau virtuel.

### Configuration des interfaces supplémentaires

Après avoir ajouté et raccordé les cartes réseau, j'ai poursuivi la configuration d'OPNsense.

Depuis le navigateur de SERVER-DHCP, j'ai accédé à son interface web :

`https://192.168.1.254`

J'ai ensuite configuré les interfaces des deux nouveaux réseaux en **Static IPv4**.

| Interface | Adresse IPv4 | Masque |
|-----------|--------------|--------|
| LAN | `192.168.1.254` | `255.255.255.0` |
| COMPTABILITE (OPT1) | `192.168.2.1` | `255.255.255.0` |
| COMMERCIAL (OPT2) | `192.168.3.1` | `255.255.255.0` |

Chaque réseau possède maintenant sa propre adresse de passerelle.

Ces adresses seront également renseignées dans les options DHCP du serveur Windows afin que les futurs clients reçoivent la passerelle correspondant à leur réseau.

*Capture à ajouter : configuration des interfaces LAN, COMPTABILITE et COMMERCIAL depuis l'interface web d'OPNsense.*

*Capture à ajouter : carte réseau de COMPTA-DEVICE1 associée au réseau COMPTABILITE dans Virt-Manager.*

## 6. Adaptation du serveur DHCP Windows

Avec les trois réseaux configurés, je devais maintenant adapter mon serveur DHCP à la nouvelle infrastructure.

Dans la première version du projet, le serveur et le poste client étaient sur le réseau `192.168.1.0/24`.

L'étendue DHCP distribuait des adresses comprises entre `192.168.1.2` et `192.168.1.4`.

Cette configuration avait permis de valider le fonctionnement du DHCP sur un seul réseau.

Avec ma nouvelle organisation, le réseau `192.168.1.0/24` est désormais réservé à SERVER-DHCP et à l'interface LAN d'OPNsense.

Les postes clients doivent recevoir des adresses appartenant aux réseaux Comptabilité ou Commercial.

### 6.1. Modification de l'adresse IP fixe du serveur

SERVER-DHCP conserve son adresse IPv4 fixe `192.168.1.1`.

J'ai cependant ajouté une passerelle à sa configuration pour qu'il puisse communiquer avec les autres réseaux.

| Paramètre | Configuration |
|-----------|---------------|
| Adresse IPv4 | `192.168.1.1` |
| Masque | `255.255.255.0` |
| Passerelle | `192.168.1.254` |
| DNS préféré | `192.168.1.1` |

La passerelle correspond à l'interface LAN d'OPNsense, située sur le même réseau que le serveur.

### 6.2. Adaptation de l'étendue initiale

L'étendue initiale `192.168.1.0/24` n'est plus utilisée pour mes tests sur les postes clients.

J'ai donc adapté sa configuration à la nouvelle organisation, notamment en renseignant la passerelle `192.168.1.254` dans l'option **003 Routeur**.

Les deux nouveaux réseaux auront leurs propres étendues DHCP.

### 6.3. Création de l'étendue Comptabilité

Depuis la console DHCP de Windows Server, j'ai créé une nouvelle étendue pour le réseau Comptabilité.

| Paramètre | Configuration |
|-----------|---------------|
| Réseau | `192.168.2.0/24` |
| Adresse de début | `192.168.2.2` |
| Adresse de fin | `192.168.2.4` |
| Masque | `255.255.255.0` |
| Passerelle - Option 003 | `192.168.2.1` |
| DNS - Option 006 | `192.168.1.1` |

Cette étendue permet de distribuer trois adresses IP.

La passerelle `192.168.2.1` correspond à l'interface Comptabilité d'OPNsense.

### 6.4. Création de l'étendue Commercial

J'ai ensuite créé une autre étendue pour le réseau Commercial.

| Paramètre | Configuration |
|-----------|---------------|
| Réseau | `192.168.3.0/24` |
| Adresse de début | `192.168.3.2` |
| Adresse de fin | `192.168.3.4` |
| Masque | `255.255.255.0` |
| Passerelle - Option 003 | `192.168.3.1` |
| DNS - Option 006 | `192.168.1.1` |

Cette étendue permet également de distribuer trois adresses IP.

Lors de la configuration, j'ai fait attention à attribuer à chaque étendue la passerelle correspondant à son propre réseau, tout en conservant `192.168.1.1` comme serveur DNS.

Les trois réseaux utilisent donc le même serveur Windows pour le DHCP et le DNS, mais disposent de passerelles différentes.

*Capture à ajouter : console DHCP Windows montrant les étendues et leur état d'activation.*

*Capture à ajouter : options d'étendue Comptabilité, avec la passerelle et le DNS.*

*Capture à ajouter : options d'étendue Commercial, avec la passerelle et le DNS.*

## 7. Configuration du relais DHCP

Il restait maintenant à permettre aux postes Comptabilité et Commercial de recevoir leurs adresses IP depuis SERVER-DHCP, bien qu'ils soient connectés à des réseaux différents.

Pour cela, j'ai utilisé le **relais DHCP** d'OPNsense.

Depuis l'interface web :

**Services → DHCRelay**

### Configuration de la destination

J'ai commencé par renseigner l'adresse de mon serveur DHCP Windows.

| Paramètre | Configuration |
|-----------|---------------|
| Destination | `192.168.1.1` |

### Configuration des relais

J'ai ensuite associé cette destination aux deux interfaces des réseaux clients.

| Interface | Destination |
|-----------|-------------|
| COMPTABILITE (OPT1) | SERVER-DHCP - `192.168.1.1` |
| COMMERCIAL (OPT2) | SERVER-DHCP - `192.168.1.1` |

L'objectif était que les demandes DHCP provenant de Comptabilité et Commercial soient transmises par OPNsense au serveur Windows.

À ce stade, les interfaces et la destination étaient correctement renseignées.

Cependant, je n'avais pas remarqué que les relais n'étaient pas encore activés.

*Capture à ajouter : destination du relais DHCP dans OPNsense.*

*Capture à ajouter : relais Comptabilité et Commercial configurés, dans leur état final activé.*

## 8. Premier test et difficultés

Une fois la configuration terminée, j'ai lancé un premier test depuis **COMPTA-DEVICE1**, connecté au réseau COMPTABILITE.

Sa carte réseau Windows était configurée pour obtenir automatiquement une adresse IPv4.

Depuis l'invite de commandes, j'ai exécuté :

```cmd
ipconfig /release
ipconfig /renew
```

**Résultat : le client ne recevait pas l'adresse IP attendue.**

J'ai ensuite utilisé :

```cmd
ipconfig /all
```

J'ai constaté que Windows utilisait une adresse IPv4 commençant par `169.254`.

Après quelques recherches, j'ai découvert qu'il s'agissait d'une adresse **APIPA**, attribuée automatiquement par Windows lorsqu'un poste configuré en DHCP ne parvient pas à obtenir une adresse depuis un serveur DHCP.

Cela m'indiquait que la demande DHCP n'avait pas abouti.

Je devais donc reprendre les vérifications pour identifier le problème.

*Capture facultative à ajouter : premier test montrant une adresse APIPA, uniquement si une capture de cet échec a été conservée.*

## 9. Vérifications et résolution du problème

Face à cet échec, j'ai décidé de reprendre les vérifications depuis les éléments de base de mon infrastructure, plutôt que de modifier plusieurs paramètres au hasard.

### 9.1. Vérification des réseaux virtuels

J'ai commencé par contrôler la configuration dans Virt-Manager :

- Présence des différents réseaux isolés.
- Raccordement des cartes réseau d'OPNsense.
- Association de COMPTA-DEVICE1 au réseau COMPTABILITE.
- Désactivation du service DHCP de libvirt sur les réseaux clients.

Je voulais notamment m'assurer que le poste était bien connecté au réseau prévu.

### 9.2. Vérification du serveur DHCP

Je suis ensuite revenu sur **SERVER-DHCP** pour vérifier :

- Son adresse IPv4 fixe et sa passerelle.
- La configuration des étendues DHCP.
- Les plages d'adresses et les masques.
- Les passerelles et les serveurs DNS.
- L'activation des étendues.

Les paramètres vérifiés correspondaient à la nouvelle architecture.

### 9.3. Vérification d'OPNsense

Enfin, j'ai vérifié les interfaces du routeur :

| Interface | Adresse |
|-----------|---------|
| LAN | `192.168.1.254/24` |
| COMPTABILITE | `192.168.2.1/24` |
| COMMERCIAL | `192.168.3.1/24` |

Je suis ensuite retourné dans **Services → DHCRelay**.

La destination `192.168.1.1` était correcte et les relais étaient bien associés aux interfaces Comptabilité et Commercial.

**C'est à ce moment-là que j'ai découvert que les relais DHCP étaient configurés, mais désactivés.**

J'avais renseigné les paramètres nécessaires, mais oublié d'activer les relais.

### Correction

J'ai activé les deux relais DHCP dans OPNsense, puis appliqué la configuration.

J'ai ensuite repris les tests depuis le poste client.

## 10. Tests et validation

### 10.1. Test sur le réseau Comptabilité

Après avoir activé les relais, j'ai effectué une nouvelle demande DHCP depuis COMPTA-DEVICE1, toujours connecté au réseau COMPTABILITE.

```cmd
ipconfig /release
ipconfig /renew
ipconfig /all
```

Cette fois, le poste a reçu une configuration correspondant au réseau `192.168.2.0/24`.

| Paramètre | Configuration attendue |
|-----------|------------------------|
| Adresse IPv4 | `192.168.2.2` à `192.168.2.4` |
| Masque | `255.255.255.0` |
| Passerelle | `192.168.2.1` |
| Serveur DHCP | `192.168.1.1` |
| DNS | `192.168.1.1` |

**Résultat : le DHCP fonctionne sur Comptabilité.**

Le poste reçoit une adresse provenant de mon serveur Windows, bien qu'il soit connecté à un réseau différent.

*Capture à ajouter : résultat de `ipconfig /all` sur COMPTA-DEVICE1, montrant une adresse `192.168.2.x`, la passerelle et le serveur DHCP.*

### 10.2. Test sur le réseau Commercial

Pour vérifier le fonctionnement de la deuxième étendue, j'ai utilisé la même VM Windows 11.

Depuis Virt-Manager :

1. Arrêt de COMPTA-DEVICE1.
2. Ouverture des paramètres de sa carte réseau.
3. Modification du réseau virtuel de COMPTABILITE vers COMMERCIAL.
4. Redémarrage de Windows 11.

Le poste était toujours configuré pour recevoir automatiquement une adresse IP.

J'ai ensuite exécuté :

```cmd
ipconfig /all
```

Cette fois, le poste a reçu une configuration correspondant au réseau `192.168.3.0/24`.

| Paramètre | Configuration attendue |
|-----------|------------------------|
| Adresse IPv4 | `192.168.3.2` à `192.168.3.4` |
| Masque | `255.255.255.0` |
| Passerelle | `192.168.3.1` |
| Serveur DHCP | `192.168.1.1` |
| DNS | `192.168.1.1` |

**Résultat : le DHCP fonctionne également sur Commercial.**

Ces deux tests confirment que le serveur Windows peut distribuer des adresses IP sur plusieurs réseaux grâce au relais DHCP d'OPNsense.

Pour cette validation, je n'ai pas installé de deuxième poste Windows 11 : j'ai utilisé successivement la même VM sur Comptabilité et Commercial.

*Capture à ajouter : résultat de `ipconfig /all` sur le réseau COMMERCIAL, montrant une adresse `192.168.3.x`, la passerelle et le serveur DHCP.*

## 11. Résultat de cette étape

L'infrastructure initiale a été modifiée pour fonctionner avec trois réseaux distincts.

La nouvelle configuration comprend :

- **SERVER-DHCP**, sous Windows Server 2022, sur le réseau `192.168.1.0/24`.
- **ROUTAGE**, sous OPNsense, avec trois interfaces réseau.
- Un réseau **Comptabilité** en `192.168.2.0/24`.
- Un réseau **Commercial** en `192.168.3.0/24`.
- Un **relais DHCP** permettant aux postes des deux réseaux clients de recevoir leurs adresses depuis SERVER-DHCP.

Les tests réalisés depuis COMPTA-DEVICE1 ont confirmé l'attribution automatique d'adresses IP sur les réseaux Comptabilité et Commercial.

**L'objectif de cette évolution est atteint : mon serveur DHCP fonctionne désormais avec plusieurs sous-réseaux, grâce à OPNsense.**

Le renforcement des règles de pare-feu et le filtrage des communications entre les réseaux pourront être étudiés dans une prochaine étape.

## 12. Ce que j'ai appris

Cette nouvelle partie du projet m'a permis de découvrir et de mettre en pratique :

- La création et la configuration de mon premier routeur OPNsense.
- L'organisation de plusieurs réseaux virtuels isolés sous KVM.
- L'utilisation de plusieurs cartes réseau sur une même VM.
- La configuration des passerelles et des étendues DHCP sur différents sous-réseaux.
- La mise en place d'un relais DHCP.
- L'identification d'une adresse APIPA lors d'un échec d'attribution.
- L'importance de vérifier l'activation des services lorsqu'un problème apparaît.

Le problème rencontré m'a notamment appris qu'un service pouvait être correctement configuré, sans pour autant être activé.

J'ai également compris qu'une infrastructure déjà fonctionnelle pouvait être **modifiée et améliorée**, plutôt que reconstruite entièrement.

Au départ, mon objectif était simplement de faire fonctionner un serveur DHCP avec un poste client.

C'est en me demandant comment ajouter un deuxième poste sur un réseau différent que j'ai décidé d'aller plus loin et de découvrir de nouvelles notions d'administration systèmes et réseaux.

---

<sub>**Ressources :** Documentation technique, tutoriels en ligne, recherches personnelles et utilisation de l'intelligence artificielle pour comprendre les notions de routage et de relais DHCP, accompagner la configuration et résoudre les problèmes rencontrés.</sub>
