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
2. **Réseau et DNS** — remise en cohérence sur 192.168.56.0/24, nettoyage des anciens enregistrements DNS, enregistrement `glpi01.exemple.lan`, connectivité et résolution testées dans les deux sens (GLPI → AD et AD → GLPI).
3. **Active Directory** — OU `Utilisateurs` (Direction, Employes, IT), `Groupes`, `Postes`, `ComptesService` ; groupes `GRP-Direction`, `GRP-Employes`, `GRP-IT`, `GRP-GLPI-SVC` ; comptes `direction01`, `employe01`, `it01` et compte de service `svc-glpi` (**non administrateur**).
4. **Liaison LDAP** — annuaire `AD-Site-A` dans GLPI (port 389, Base DN `DC=exemple,DC=lan`, connexion avec `svc-glpi`, identifiant `samaccountname`, synchronisation `objectGUID`), test réussi et import des utilisateurs.
5. **Profils** — Self-Service pour les demandeurs, Technician pour `it01` ; vérification des droits (accès à la configuration refusé à `employe01`).
6. **Parc** — entités et lieux du site A, ordinateurs (DC01 inventorié par l'agent GLPI, GLPI01 et CLIENT01 déclarés), équipements réseau ; procédure d'installation de GLPI Agent.
7. **Ticket** — incident créé par `employe01`, attribué à `it01` + groupe IT, suivi, résolu puis **clos** par le demandeur.
8. **Recette** — plan de recette entièrement conforme, écarts et corrections consignés.

## Réponses aux questions du TP (version courte)

1. **IP de la VM ou IP de l'hôte :** l'IP de l'hyperviseur sert uniquement à l'administrer. Chaque VM a sa propre carte réseau virtuelle (MAC et IP), donc on joint GLPI par l'IP de la VM GLPI (192.168.56.20), pas par celle de l'hôte.
2. **DNS Internet sur GLPI :** un DNS Internet ne connaît pas la zone interne `exemple.lan`. GLPI ne trouve plus le contrôleur de domaine, donc la liaison LDAP et les connexions AD échouent (NXDOMAIN constaté).
3. **OU séparées :** elles permettent d'appliquer des règles différentes à chaque type d'objet (GPO, délégation, droits), de garder un annuaire lisible et de limiter la recherche LDAP de GLPI aux vrais utilisateurs.
4. **objectGUID :** identifiant unique qui ne change jamais, même si le compte est renommé ou déplacé. GLPI reconnaît toujours la même personne, et ses tickets et son matériel restent rattachés.
5. **Limiter le Self-Service :** c'est le moindre privilège. Sinon, un utilisateur pourrait s'attribuer des droits d'administrateur, détourner des tickets ou fausser l'inventaire.
6. **Données remontées par un agent :** nom du poste, système d'exploitation, fabricant, modèle et numéro de série, processeur, mémoire, disques, cartes réseau (IP, MAC), logiciels installés et utilisateur connecté.
7. **Résolu ou Clos :** Résolu = le technicien a appliqué et documenté la solution. Clos = le demandeur a validé (ou le délai est dépassé) et le ticket est figé. Tant qu'il n'est pas clos, le demandeur peut refuser la solution et le ticket est rouvert.

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

## Mission complémentaire : sécurisation et recette (13 octobre 2026)

Suite du TP sur les mêmes VMs, dossier [`mission-securisation/`](mission-securisation/) :

1. **État des services** — ADWS, DNS, DHCP, domaine AD, Apache, MariaDB, disque et mémoire contrôlés.
2. **Connectivité et DNS** — GLPI ↔ AD par IP et par nom ; port 80 ouvert, 443 fermé (GLPI en HTTP seulement).
3. **Compte nominatif** — `admin-sitea` membre du groupe `sudo`, `sudo whoami` → root.
4. **SSH et pare-feu** — `PermitRootLogin no` (vérifié avec `sshd -t` / `sshd -T`) ; UFW : entrant refusé par défaut, seuls 22/TCP et 80/TCP ouverts ; MariaDB écoute uniquement sur 127.0.0.1.
5. **Journaux** — journalctl (ssh, apache2, mariadb) et Windows (Sécurité 4624, Système, DNS Server).
6. **Ticket GLPI** — demande n° 2 « Sécurisation de l'accès d'administration au serveur GLPI Site A », catégorie Sécurité > Administration serveur, résolue puis close.
7. **Recette** — tous les contrôles conformes.

Incidents traités : crash de la VM GLPI (mémoire de l'hôte saturée) → restauration du snapshot pris avant la modification ; erreur LDAP après UFW → diagnostic ping + port 389 : contrôleur de domaine éteint, pas le pare-feu.

## Contenu du dépôt

```
rapport/   compte rendu complet du TP (HTML + PDF, en français)
captures/  89 captures d'écran classées par étape (preuves)
docs/      guide des concepts clés (PDF, en anglais), avec accès au labo et rejeu des 8 étapes
mission-securisation/
  rapport/   compte rendu de la mission (HTML + PDF, en français)
  captures/  captures de preuve classées par étape
  docs/      guide des concepts clés de la mission (PDF, en anglais), avec accès au labo et rejeu
```

## Sécurité

Aucun mot de passe, clé ou secret ne figure dans ce dépôt ni dans les captures. Les adresses sont celles d'un laboratoire privé (host-only).

## Limites et améliorations prévues

- LDAP en clair (389) → passer en **LDAPS (636)**.
- Pas de VM Windows cliente → ajouter un client avec **GLPI Agent** (option A).
- Synchroniser automatiquement les groupes AD avec les groupes GLPI.

---
Charles MADUBA — Titre professionnel AIS
