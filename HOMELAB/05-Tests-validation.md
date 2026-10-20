
# 05 - Tests et validation de l'infrastructure

## Objectif

Valider le fonctionnement du **serveur DHCP** et la communication entre les deux machines virtuelles, après la création du réseau isolé **SERVER-TO-COMPTA**.

- **SERVER-DHCP :** Windows Server 2022
- **COMPTA-DEVICE1 :** Windows 11

## 1. Test du DHCP côté client

Après avoir corrigé la configuration du réseau virtuel, j'ai repris les tests depuis **COMPTA-DEVICE1**.

Pour vérifier que le client pouvait obtenir une nouvelle adresse IP depuis mon serveur DHCP, j'ai utilisé les commandes suivantes :

```cmd
ipconfig /release
ipconfig /renew
```

- **`ipconfig /release` :** libère le bail DHCP actuel.
- **`ipconfig /renew` :** demande une nouvelle configuration IP au serveur DHCP.

J'ai volontairement utilisé ces deux commandes afin de ne pas simplement me fier à une adresse IP qui aurait déjà été attribuée au client.

**Résultat :** le client obtient une nouvelle adresse IP. Il reste à vérifier qu'elle provient bien de **SERVER-DHCP**, contrairement au premier test où elle était attribuée par le DHCP de libvirt.

![Test DHCP - Release et Renew](../IMAGES/IPCONFIG-RELEASE-RENEW.png)

### Vérification de la configuration obtenue

Pour rappel, **COMPTA-DEVICE1** doit recevoir une adresse IP appartenant à l'étendue configurée sur **SERVER-DHCP**.

La configuration attendue est la suivante :

| Paramètre | Configuration attendue |
|-----------|------------------------|
| DHCP activé | Oui |
| Adresse IPv4 | 192.168.1.2 à 192.168.1.4 |
| Masque de sous-réseau | 255.255.255.0 (/24) |
| Serveur DHCP | 192.168.1.1 |
| Passerelle | Non configurée |

Pour contrôler ces informations, j'ai exécuté :

```cmd
ipconfig /all
```

**Résultat :** l'adresse IPv4 appartient bien à la plage prévue et le serveur DHCP indiqué est `192.168.1.1`.

Cela confirme que **COMPTA-DEVICE1** reçoit désormais sa configuration IP depuis **SERVER-DHCP**, et non depuis le DHCP de libvirt.

![Configuration IP du client](../IMAGES/DEVICE1-IPCONFIGALL.png)

## 2. Vérification côté serveur

Après avoir confirmé l'attribution DHCP sur le client, j'ai également vérifié le fonctionnement du service directement depuis **SERVER-DHCP**, à l'aide de PowerShell.

### État du service DHCP

```powershell
Get-Service DHCPServer
```

Cette commande permet de vérifier que le service **DHCPServer** est bien en cours d'exécution.

### Vérification de l'étendue DHCP

```powershell
Get-DhcpServerv4Scope
```

Elle permet de retrouver les paramètres de l'étendue configurée précédemment, notamment la plage `192.168.1.2` à `192.168.1.4` et son état d'activation.

### Vérification des baux DHCP

```powershell
Get-DhcpServerv4Lease -ScopeId 192.168.1.0
```

Cette dernière commande permet de consulter les adresses IP attribuées aux clients par le serveur.

Ces vérifications permettent de contrôler le DHCP des deux côtés : **la configuration reçue par le client et les informations enregistrées sur le serveur**.

![Vérification du DHCP avec PowerShell](../IMAGES/POWERSELL-DHCP.png)

## 3. Test de connectivité

Une fois l'attribution DHCP validée, j'ai voulu vérifier que **COMPTA-DEVICE1** pouvait également communiquer avec **SERVER-DHCP**.

Depuis le client, j'ai exécuté :

```cmd
ping 192.168.1.1
```

**Résultat : échec du ping**, alors que l'attribution DHCP fonctionnait correctement.

### Vérifications effectuées

Face à cet échec, j'ai commencé par vérifier les éléments que je connaissais :

- L'adresse IP attribuée au client.
- L'adresse IP fixe du serveur.
- Le fonctionnement du service DHCP.
- La configuration IP du serveur avec `ipconfig`.

Ces vérifications ne montrant pas d'anomalie, j'ai envisagé que le problème pouvait provenir d'un élément que je ne connaissais pas encore.

### Recherche et résolution du problème

Après des recherches et avec l'aide de l'IA, j'ai découvert que le **pare-feu Windows Server** pouvait bloquer les requêtes **ICMPv4**, utilisées notamment par la commande `ping`.

J'ai donc ouvert la configuration avancée du pare-feu sur **SERVER-DHCP** :

```cmd
wf.msc
```

Depuis cette interface :

1. Accès aux **règles de trafic entrant**.
2. Recherche de la règle **Demande d'écho - Trafic entrant ICMPv4**.
3. Activation de la règle correspondant au profil réseau utilisé.

J'ai ensuite relancé le test depuis **COMPTA-DEVICE1** :

```cmd
ping 192.168.1.1
```

**Résultat : ping réussi.**

Le problème provenait donc du filtrage des requêtes ICMPv4 par le pare-feu Windows Server, et non du service DHCP.

![Test de connectivité réussi](../IMAGES/PING-192.168.1.1.png)

## 4. Résultat de cette étape

Les tests confirment le fonctionnement de cette première infrastructure réseau :

- **COMPTA-DEVICE1** obtient automatiquement une adresse IP depuis **SERVER-DHCP**.
- L'adresse attribuée correspond à l'étendue DHCP configurée.
- Le service DHCP, son étendue et ses baux peuvent être vérifiés avec PowerShell.
- La communication entre le client et le serveur fonctionne après correction de la règle ICMPv4.

La première infrastructure est donc **opérationnelle**.

## 5. Ce que j'ai appris

Cette étape m'a permis de mieux comprendre :

- L'intérêt de forcer un renouvellement DHCP pour vérifier l'attribution d'une adresse IP.
- L'utilisation de **PowerShell** pour contrôler le fonctionnement d'un serveur DHCP.
- La différence entre le fonctionnement du **DHCP (UDP)** et les tests de connectivité utilisant **ICMP**.
- Le rôle du **pare-feu Windows** dans le filtrage des communications réseau.

J'ai notamment retenu qu'une attribution DHCP réussie ne signifie pas nécessairement que toutes les communications entre deux machines sont autorisées.

Cette découverte m'a également amené à m'interroger sur les règles de pare-feu et leur impact sur la sécurité du réseau. Une étape de **sécurisation de l'infrastructure** sera ajoutée après la mise en place complète du homelab.

---

<sub>**Ressources :** Connaissances personnelles en réseau, tutoriels YouTube et utilisation de l'intelligence artificielle pour approfondir les notions techniques et accompagner le diagnostic des problèmes rencontrés.</sub>
