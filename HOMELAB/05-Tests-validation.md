
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

*Capture d'écran du résultat de `ipconfig /all`.*

## 3. Test de connectivité

Depuis **COMPTA-DEVICE1**, lancement d'un test de communication vers le serveur :

```cmd
ping 192.168.1.1
```

**Résultat initial : échec du ping**, malgré le bon fonctionnement du service DHCP.

### Problème rencontré : pare-feu Windows

Après vérification, le problème provenait du **pare-feu Windows Server**, qui bloquait les requêtes ICMPv4 entrantes.

Le DHCP fonctionnait car il utilise le protocole **UDP**, tandis que le ping utilise **ICMP**.

### Solution appliquée

Sur **SERVER-DHCP** :

1. Ouverture du pare-feu avancé avec :

```cmd
wf.msc
```

2. Accès aux **règles de trafic entrant**.
3. Recherche de la règle **Demande d'écho - Trafic entrant ICMPv4**.
4. Activation de la règle pour autoriser les requêtes ping.

### Vérification après correction

Nouveau test depuis **COMPTA-DEVICE1** :

```cmd
ping 192.168.1.1
```

**Résultat : ping réussi.**

La communication entre les deux machines est confirmée.

*Capture d'écran du ping réussi.*

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

- Le **DHCP** et le **ping** utilisent des protocoles différents.
- Le pare-feu Windows peut bloquer certaines communications sans empêcher le fonctionnement des autres services.
- Une règle de pare-feu peut être adaptée sans désactiver entièrement la protection.

Cette découverte m'a également amené à m'interroger sur les bonnes pratiques de filtrage réseau. Une étape de **sécurisation de l'infrastructure** sera donc ajoutée après la mise en place complète du homelab.

## 7. Prochaine évolution

Après validation de cette première infrastructure :

- Ajout d'une **deuxième VM cliente** sur un autre sous-réseau.
- Création d'une **VM routeur** pour relier les réseaux.
- Configuration d'un **relais DHCP** pour distribuer les adresses IP sur plusieurs sous-réseaux.

---

<sub>**Ressources :** Connaissances personnelles en réseau, tutoriels YouTube et utilisation de l'intelligence artificielle pour accompagner les tests, comprendre les protocoles et résoudre les difficultés rencontrées.</sub>
