
# 05 - Tests et validation de l'infrastructure

## Objectif

Vérifier le bon fonctionnement du **serveur DHCP** et la communication réseau entre les deux machines virtuelles :

- **SERVER-DHCP :** Windows Server 2022
- **COMPTA-DEVICE1 :** Windows 11

## 1. Vérification de l'attribution DHCP

Sur **COMPTA-DEVICE1**, configuration de la carte réseau pour obtenir automatiquement une adresse IPv4.

Ouverture du terminal Windows et exécution des commandes :

```cmd
ipconfig /release
ipconfig /renew
```

Ces commandes permettent de libérer l'adresse IP actuelle, puis de demander une nouvelle configuration au serveur DHCP.

**Résultat :** le client obtient correctement une adresse IP depuis le serveur Windows.

## 2. Vérification de la configuration IP

Exécution de la commande :

```cmd
ipconfig /all
```

Vérification des informations reçues :

| Paramètre | Résultat attendu |
|-----------|------------------|
| DHCP activé | Oui |
| Adresse IPv4 | 192.168.1.2 à 192.168.1.4 |
| Masque | 255.255.255.0 |
| Serveur DHCP | 192.168.1.1 |
| Passerelle | Non configurée |

Le poste client reçoit une adresse appartenant à l'étendue configurée sur **SERVER-DHCP**.

*Capture d'écran du résultat de `ipconfig /all` à ajouter.*

## 3. Test de connectivité

Depuis **COMPTA-DEVICE1**, lancement d'un test de communication vers le serveur :

```cmd
ping 192.168.1.1
```

**Résultat initial : échec du ping**, malgré le bon fonctionnement du DHCP.

### Problème rencontré

Face à cet échec, j'ai commencé par vérifier les éléments que je connaissais :

- L'adresse IP attribuée au client.
- L'adresse IP fixe du serveur.
- Le fonctionnement du service DHCP.
- La configuration IP du serveur avec `ipconfig`.

Toutes ces vérifications étant correctes, j'ai supposé que le problème pouvait provenir d'un élément que je ne connaissais pas encore.

### Recherche et résolution

Après des recherches et avec l'aide de l'IA, j'ai découvert que le **pare-feu Windows Server** pouvait bloquer les requêtes **ICMPv4**, utilisées par la commande `ping`.

Sur **SERVER-DHCP**, ouverture de la configuration avancée du pare-feu :

```cmd
wf.msc
```

Puis :

1. Accès aux **règles de trafic entrant**.
2. Recherche de la règle **Demande d'écho - Trafic entrant ICMPv4**.
3. Activation de la règle pour autoriser les requêtes ping.

### Vérification après correction

Nouveau test depuis **COMPTA-DEVICE1** :

```cmd
ping 192.168.1.1
```

**Résultat : ping réussi.**

Le problème provenait donc du filtrage ICMPv4 par le pare-feu Windows Server.

*Capture d'écran du ping réussi à ajouter.*

## 4. Vérification côté serveur DHCP

Depuis la console DHCP de Windows Server 2022 :

- Vérification de l'activation de l'étendue `192.168.1.2` à `192.168.1.4`.
- Consultation des **baux d'adresses**.
- Vérification de l'adresse attribuée au poste client.

Cette vérification permet de confirmer que le serveur Windows assure bien l'attribution des adresses IP.

## 5. Résultat final

La première infrastructure réseau est **opérationnelle**.

| Test | Résultat |
|------|----------|
| Attribution automatique DHCP | ✅ Validé |
| Renouvellement du bail DHCP | ✅ Validé |
| Adresse IP dans l'étendue prévue | ✅ Validé |
| Communication entre les VM (ping) | ✅ Validé |
| Correction du problème ICMPv4 | ✅ Validé |

## 6. Ce que j'ai appris

Cette étape m'a permis de comprendre qu'un **service DHCP fonctionnel ne garantit pas que toutes les communications réseau sont autorisées**.

J'ai notamment découvert que :

- Le **DHCP** utilise UDP, tandis que le **ping** utilise ICMP.
- Le pare-feu peut bloquer un protocole sans empêcher les autres services de fonctionner.
- Une règle de pare-feu peut être adaptée sans désactiver entièrement la protection.

Cette découverte m'a également amené à m'interroger sur les bonnes pratiques de filtrage réseau.

Une étape de **sécurisation de l'infrastructure** sera ajoutée après la mise en place complète du homelab.

## 7. Prochaine évolution

Après validation de cette première infrastructure :

- Ajout d'une **deuxième VM cliente** sur un autre sous-réseau.
- Création d'une **VM routeur** pour relier les réseaux.
- Configuration d'un **relais DHCP** pour distribuer les adresses IP sur plusieurs sous-réseaux.

---

<sub>**Ressources :** Connaissances personnelles en réseau, tutoriels YouTube et utilisation de l'intelligence artificielle pour accompagner les tests, approfondir les notions techniques et résoudre les difficultés rencontrées.</sub>
