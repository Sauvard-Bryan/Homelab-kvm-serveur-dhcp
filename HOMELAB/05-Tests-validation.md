
# 05 - Tests et validation de l'infrastructure

## Objectif

Vérifier le fonctionnement du **serveur DHCP** et la communication entre les deux machines virtuelles :

- **SERVER-DHCP :** Windows Server 2022
- **COMPTA-DEVICE1 :** Windows 11

## 1. Vérification de l'attribution DHCP

Sur **COMPTA-DEVICE1**, configuration de la carte réseau pour obtenir automatiquement une adresse IP.

Exécution des commandes :

```cmd
ipconfig /release
ipconfig /renew
```

- `ipconfig /release` : libère l'adresse IP actuelle.
- `ipconfig /renew` : demande une nouvelle configuration au serveur DHCP.

**Résultat :** le client reçoit automatiquement une adresse IP depuis SERVER-DHCP.

## 2. Vérification de la configuration IP

Sur le poste client :

```cmd
ipconfig /all
```

Vérification des paramètres réseau :

| Paramètre | Résultat attendu |
|-----------|------------------|
| DHCP activé | Oui |
| Adresse IPv4 | 192.168.1.2 à 192.168.1.4 |
| Masque | 255.255.255.0 |
| Serveur DHCP | 192.168.1.1 |
| Passerelle | Non configurée |

Le poste client reçoit une adresse appartenant à l'étendue configurée sur le serveur Windows.

![Configuration IP du client](../IMAGES/DEVICE1-IPCONFIGALL.png)

## 3. Vérification côté serveur DHCP

Sur **SERVER-DHCP**, ouverture de PowerShell en administrateur.

### Vérification du service DHCP

```powershell
Get-Service DHCPServer
```

Cette commande permet de vérifier que le service DHCP est en cours d'exécution (`Running`).

### Vérification de l'étendue DHCP

```powershell
Get-DhcpServerv4Scope
```

Vérification des paramètres :

- **ScopeId :** 192.168.1.0
- **StartRange :** 192.168.1.2
- **EndRange :** 192.168.1.4
- **SubnetMask :** 255.255.255.0
- **State :** Active

### Vérification des baux DHCP

```powershell
Get-DhcpServerv4Lease -ScopeId 192.168.1.0
```

Cette commande permet de consulter les adresses IP attribuées aux machines clientes.

![Vérification DHCP avec PowerShell](../IMAGES/POWERSELL-DHCP.png)

## 4. Test de connectivité

Depuis **COMPTA-DEVICE1**, test de communication avec le serveur :

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
3. Activation de la règle autorisant les requêtes ping.

### Vérification après correction

Nouveau test depuis **COMPTA-DEVICE1** :

```cmd
ping 192.168.1.1
```

**Résultat : ping réussi.**

Le problème provenait donc du filtrage ICMPv4 par le pare-feu Windows Server.

![Test de connectivité réussi](../IMAGES/PING-192.168.1.1.png)

## 5. Résultat final

La première infrastructure réseau est **opérationnelle**.

| Test | Résultat |
|------|----------|
| Service DHCP fonctionnel | ✅ Validé |
| Étendue DHCP configurée | ✅ Validé |
| Attribution automatique DHCP | ✅ Validé |
| Renouvellement du bail DHCP | ✅ Validé |
| Communication entre les VM | ✅ Validé |
| Correction du problème ICMPv4 | ✅ Validé |

## 6. Ce que j'ai appris

Cette étape m'a permis de comprendre :

- L'utilisation de **PowerShell** pour contrôler les services et les paramètres DHCP.
- La différence entre les protocoles **UDP (DHCP)** et **ICMP (ping)**.
- Le rôle du **pare-feu Windows** dans le filtrage des communications.
- L'importance de vérifier plusieurs éléments avant de rechercher une nouvelle cause à un problème.

Cette découverte m'a également amené à m'interroger sur les bonnes pratiques de filtrage réseau.

Une étape de **sécurisation de l'infrastructure** sera ajoutée après la mise en place complète du homelab.

## 7. Prochaine évolution

- Ajout d'une **deuxième VM cliente** sur un autre sous-réseau.
- Création d'une **VM routeur** pour relier les réseaux.
- Mise en place d'un **relais DHCP** pour distribuer les adresses IP sur plusieurs sous-réseaux.

---

<sub>**Ressources :** Connaissances personnelles en réseau, tutoriels YouTube et utilisation de l'intelligence artificielle pour accompagner les tests, approfondir les notions techniques et résoudre les difficultés rencontrées.</sub>
