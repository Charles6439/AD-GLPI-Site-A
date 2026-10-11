# Mise en exploitation Active Directory + GLPI — Site A

🇫🇷 Français · [🇬🇧 English version](README.en.md)

Travaux pratiques réalisés dans le cadre du **Titre professionnel Administrateur d'Infrastructures Sécurisées (AIS)** — GRETA Essonne.
Laboratoire à domicile (VirtualBox), du 6 au 11 octobre 2026.

## Objectif du projet

Remettre en service, à partir de machines virtuelles déjà existantes, l'environnement d'un **site A** d'entreprise :

- un **contrôleur de domaine Windows Server 2022** (Active Directory + DNS, domaine `exemple.lan`) ;
- un **serveur GLPI 11** sous Debian 13, relié à l'Active Directory par **LDAP** ;
- puis exploiter GLPI comme un vrai service informatique : comptes importés depuis l'AD, profils, parc matériel et cycle complet d'un ticket d'assistance.

## Architecture du laboratoire

| Élément | Rôle | Adresse (labo) |
|---|---|---|
| Serveur_TP (`WIN-RG5VDD2MS10`, DC01) | Windows Server 2022 — AD DS + DNS `exemple.lan` | 192.168.56.10 |
| GLPI-Site-A (`glpi01`) | Debian 13 — Apache, MariaDB, PHP, GLPI 11.0.4 | 192.168.56.20 |
| Poste hôte Windows | Client de test (navigateur, RDP, SSH) | 192.168.56.1 |

Réseau : VirtualBox **host-only** `192.168.56.0/24` (+ carte NAT pour l'accès Internet de GLPI).

Dans le schéma de l'école, les serveurs sont dans le **VLAN 30 – Serveur (172.16.0.64/27)**. Le laboratoire utilise un réseau plat, sans VLAN ni équipements physiques ; le switch, le routeur et le pare-feu sont déclarés manuellement dans GLPI.

![Schéma de référence du site A](captures/00-reference/schema_site_A_ecole.png)

## Ce que j'ai réalisé (8 étapes)

1. **Récolement** — inventaire des VMs existantes, de leurs cartes réseau et de leurs rôles. Constat : cartes « non attachées », AD et GLPI sur des sous-réseaux différents.
2. **Réseau et DNS** — remise en cohérence sur 192.168.56.0/24, nettoyage des anciens enregistrements DNS, enregistrement `glpi01.exemple.lan`, résolution testée dans les deux sens.
3. **Active Directory** — OU `Utilisateurs` (Direction, Employes, IT), `Groupes`, `Postes`, `ComptesService` ; groupes `GRP-Direction`, `GRP-Employes`, `GRP-IT`, `GRP-GLPI-SVC` ; comptes `direction01`, `employe01`, `it01` et compte de service `svc-glpi` (**non administrateur**).
4. **Liaison LDAP** — annuaire `AD-Site-A` dans GLPI (port 389, Base DN `DC=exemple,DC=lan`, connexion avec `svc-glpi`, identifiant `samaccountname`, synchronisation `objectGUID`), test réussi et import des utilisateurs.
5. **Profils** — Self-Service pour les demandeurs, Technician pour `it01` ; vérification des droits (accès à la configuration refusé à `employe01`).
6. **Parc** — entités et lieux du site A, ordinateurs (DC01 inventorié par l'agent GLPI, GLPI01 et CLIENT01 déclarés), équipements réseau ; procédure d'installation de GLPI Agent.
7. **Ticket** — incident créé par `employe01`, attribué à `it01` + groupe IT, suivi, résolu puis **clos** par le demandeur.
8. **Recette** — plan de recette entièrement conforme, écarts et corrections consignés.

## Principaux écarts corrigés

| Écart | Correction |
|---|---|
| Cartes réseau VirtualBox « non attachées » | Rattachement au réseau host-only, câble connecté |
| AD et GLPI sur deux sous-réseaux sans routage | GLPI déplacé en 192.168.56.20 |
| VM GLPI utilisée par un autre cours | Clone complet « GLPI-Site-A », original intact |
| Anciens enregistrements DNS, DNS client de l'AD sur 8.8.8.8 | Nettoyage, DNS de l'AD pointé sur lui-même |
| `resolv.conf` réécrit par dhcpcd | `nohook resolv.conf` + redémarrage de l'interface |
| Ancien annuaire LDAP avec compte administrateur | Nouvel annuaire avec le compte de service `svc-glpi` |

Méthode de dépannage suivie : **réseau → IP → DNS → AD → LDAP → GLPI**, avec un snapshot avant chaque modification.

## Contenu du dépôt

```
rapport/   compte rendu complet du TP (HTML + PDF, en français)
captures/  89 captures d'écran classées par étape (preuves)
docs/      guide des concepts clés (PDF, en anglais)
```

## Sécurité

Aucun mot de passe, clé ou secret ne figure dans ce dépôt ni dans les captures. Les adresses sont celles d'un laboratoire privé (host-only).

## Limites et améliorations prévues

- LDAP en clair (389) → passer en **LDAPS (636)**.
- Pas de VM Windows cliente → ajouter un client avec **GLPI Agent** (option A).
- Synchroniser automatiquement les groupes AD avec les groupes GLPI.

---
Charles MADUBA — Titre professionnel AIS
