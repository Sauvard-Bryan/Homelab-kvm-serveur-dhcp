
# 03 - Installation et configuration du DHCP

## Objectif

Installer et configurer un **serveur DHCP sous Windows Server 2022** afin de distribuer automatiquement des adresses IP aux machines clientes du réseau `192.168.1.0/24`.

## 1. Installation des rôles

Depuis le **Gestionnaire de serveur** de Windows Server 2022 :

1. Sélection de **Ajouter des rôles et fonctionnalités**.
2. Installation du rôle **Serveur DHCP**.
3. Installation du rôle **Serveur DNS**.

Le rôle DHCP permet l'attribution automatique des adresses IP. Le rôle DNS a été installé en complément pour la résolution de noms.

## 2. Configuration de l'étendue DHCP

Création d'une nouvelle étendue depuis la console DHCP.

| Paramètre | Configuration |
|-----------|---------------|
| Réseau | 192.168.1.0/24 |
| Adresse de début | 192.168.1.2 |
| Adresse de fin | 192.168.1.4 |
| Masque | 255.255.255.0 |
| Durée du bail | 8 jours |
| Passerelle | Non configurée |

La plage `192.168.1.2` à `192.168.1.4` permet de distribuer automatiquement **3 adresses IPv4**.

La durée du bail de **8 jours** correspond à la valeur proposée par défaut lors de la configuration.

## 3. Configuration de l'adresse IP fixe

Après la création de l'étendue, j'ai constaté que je n'avais pas encore attribué d'adresse IP fixe au serveur.

J'ai donc procédé à sa configuration :

1. Ouverture de **Exécuter** avec `Win + R`.
2. Saisie de la commande :

```cmd
ncpa.cpl
```

3. Ouverture des propriétés de la carte réseau.
4. Sélection de **Protocole Internet version 4 (TCP/IPv4)**.
5. Configuration manuelle de l'adresse IP.

| Paramètre | Configuration |
|-----------|---------------|
| Adresse IPv4 | 192.168.1.1 |
| Masque | 255.255.255.0 |
| Passerelle | Non configurée |
| DNS préféré | 192.168.1.1 |

L'adresse `192.168.1.1` a été choisie pour placer le serveur dans le même sous-réseau que les futurs clients, tout en restant **en dehors de la plage DHCP**.

## 4. Vérification de la configuration

Ouverture du terminal Windows et exécution de la commande :

```cmd
ipconfig
```

Cette commande a permis de vérifier que le serveur utilisait bien l'adresse IPv4 statique `192.168.1.1` et le masque `255.255.255.0`.

Vérification également de l'activation de l'étendue depuis la console DHCP.

## 5. Premiers tests et difficultés

Après la configuration du serveur, j'ai effectué un premier test depuis **COMPTA-DEVICE1**, configuré pour obtenir automatiquement son adresse IP.

Commande utilisée :

```cmd
ipconfig /renew
```

**Résultat : échec de l'attribution d'une adresse IP**, avec une erreur de dépassement du délai d'attente.

Malgré l'installation du service DHCP et l'activation de l'étendue, le poste client ne parvenait pas à obtenir sa configuration réseau.

Les investigations m'ont amené à examiner la configuration du réseau virtuel sous **KVM/libvirt**, notamment la présence du DHCP automatique sur le réseau par défaut.

Ce problème sera traité dans l'étape suivante : **création d'un réseau virtuel isolé**.

## 6. Résultat de cette étape

- Rôles **DHCP et DNS** installés sur Windows Server 2022.
- Adresse IPv4 statique `192.168.1.1/24` configurée.
- Étendue DHCP `192.168.1.2` à `192.168.1.4` créée et activée.
- Configuration IP du serveur vérifiée.
- Attribution DHCP au poste client **non fonctionnelle à ce stade**.

La prochaine étape consistera à corriger la configuration du réseau virtuel pour permettre la communication entre les deux machines.

---

<sub>**Ressources :** Connaissances personnelles en adressage IP et DHCP, tutoriels YouTube et utilisation de l'intelligence artificielle pour accompagner la configuration et le diagnostic des problèmes rencontrés.</sub>
