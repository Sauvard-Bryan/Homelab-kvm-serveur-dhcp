
# 05 - Tests et validation de l'infrastructure

## Objectif

Vérifier le bon fonctionnement du **serveur DHCP** et de la communication réseau entre les deux machines virtuelles :

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

Contrairement aux premiers essais, le renouvellement s'effectue désormais correctement, sans erreur de délai d'attente.

## 2. Vérification de la configuration IP

Exécution de la commande :

```cmd
ipconfig /all
```

Vérification des informations reçues par le poste client :

| Paramètre | Résultat attendu |
|-----------|------------------|
| DHCP activé | Oui |
| Adresse IPv4 | Entre 192.168.1.2 et 192.168.1.4 |
| Masque de sous-réseau | 255.255.255.0 |
| Serveur DHCP | 192.168.1.1 |
| Passerelle | Non configurée |

Le poste client reçoit automatiquement une adresse appartenant à l'étendue définie sur **SERVER-DHCP**.

*Capture d'écran du résultat de `ipconfig /all` à ajouter.*

## 3. Test de connectivité

Depuis **COMPTA-DEVICE1**, vérification de la communication avec le serveur :

```cmd
ping 192.168.1.1
```

**Résultat :** communication fonctionnelle entre le poste client et le serveur DHCP.

Les réponses au ping confirment que les deux machines peuvent communiquer sur le réseau `192.168.1.0/24`.

*Capture d'écran du résultat du ping à ajouter.*

## 4. Vérification côté serveur

Depuis la console DHCP de **Windows Server 2022** :

- Vérification de l'activation de l'étendue `192.168.1.2` à `192.168.1.4`.
- Consultation des **baux d'adresses**.
- Vérification de la présence du poste client dans les adresses attribuées.

Cette vérification permet de confirmer que l'adresse IP du client a bien été distribuée par le serveur Windows.

## 5. Résultat final

La première infrastructure réseau est **opérationnelle**.

| Test | Résultat |
|------|----------|
| Communication entre les VM | ✅ Validé |
| Attribution automatique DHCP | ✅ Validé |
| Renouvellement du bail DHCP | ✅ Validé |
| Adresse IP dans la plage définie | ✅ Validé |
| Identification du serveur DHCP | ✅ Validé |

Le réseau isolé **SERVER-TO-COMPTA** permet désormais au serveur Windows de distribuer automatiquement les adresses IP au poste client.

## 6. Prochaine évolution

Cette première configuration validée, la prochaine étape consistera à étendre l'infrastructure :

- Ajouter une **deuxième VM cliente** sur un sous-réseau différent.
- Mettre en place une **VM routeur** pour relier les réseaux.
- Configurer un **relais DHCP** pour permettre au serveur actuel de distribuer des adresses IP sur plusieurs sous-réseaux.

---

<sub>**Ressources :** Connaissances personnelles en réseau, tutoriels YouTube et utilisation de l'intelligence artificielle pour accompagner les tests et la résolution des problèmes.</sub>

