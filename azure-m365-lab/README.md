# Lab Azure / Microsoft 365

Mise en place d'un environnement Microsoft 365 complet en lab : gestion des identités cloud (Entra ID), services M365 (SharePoint, Teams), messagerie Exchange Online, et identité hybride avec synchronisation d'un Active Directory local vers le cloud.


---

## Architecture globale

```
┌─────────────────────────────┐         ┌──────────────────────────────────────┐
│  Windows Server 2016         │         │  Tenant lcsbgt1207gmail              │
│  AD DS — lucas.local         │         │  .onmicrosoft.com                    │
│                              │  PHS    │                                      │
│  Entra Connect Sync    ──────┼────────▶│  Microsoft Entra ID (identités)      │
│                              │         │  Microsoft 365 (SharePoint, Teams)   │
│                              │         │  Exchange Online (messagerie)         │
└─────────────────────────────┘         │  Intune (MDM — limite licence)       │
       on-premises                       └──────────────────────────────────────┘
                                                  cloud Microsoft
```

---

## Environnement

| Composant | Détail |
|---|---|
| Hyperviseur | VirtualBox |
| Serveur on-prem | Windows Server 2016 Standard |
| Domaine AD local | `lucas.local` (non routable) |
| Tenant M365 | `lcsbgt1207gmail.onmicrosoft.com` |
| Licence | Microsoft 365 Business Standard |
| Outil de sync | Microsoft Entra Connect Sync (Password Hash Sync) |

---

## Bloc 1 — Entra ID (gestion des identités cloud)

Création d'un tenant Microsoft 365 from scratch, gestion des utilisateurs, groupes, rôles et MFA.

### Utilisateurs

14 utilisateurs au total : comptes cloud natifs (Synchronisation = Non) et comptes synchronisés depuis l'AD local (Synchronisation = Oui).

![Liste des utilisateurs Entra ID](screenshots/entra-id/01-users-synced.png)

### Groupes de sécurité

Groupes créés dans le cloud (`GRP-IT`, `GRP-RH`) et groupes synchronisés depuis l'AD (source « Windows Server AD »).

![Groupes Entra ID](screenshots/entra-id/02-groups.png)

**Gestion des accès par groupes (IAM)** : les droits sont attribués aux groupes, pas aux utilisateurs individuellement. Scalable et maintenable en entreprise.

![Membres du groupe GRP-IT](screenshots/entra-id/03-grp-it-members.png)

### Rôle Helpdesk Administrator

Le compte `admin.it` a le rôle **Administrateur de support technique** et non Global Admin : il peut réinitialiser les mots de passe et gérer les utilisateurs de base, sans toucher à la configuration du tenant.

![Rôle Helpdesk sur admin.it](screenshots/entra-id/04-helpdesk-role.png)

**Principe du moindre privilège** appliqué : on donne le minimum de droits nécessaires à chaque rôle.

### MFA — Paramètres de sécurité par défaut

Le MFA est activé sur tous les comptes via les Paramètres de sécurité par défaut. Selon Microsoft, cela bloque 99,9 % des compromissions de comptes.

![Paramètres de sécurité par défaut](screenshots/entra-id/05-mfa-security-defaults.png)

---

## Bloc 2 — Microsoft 365 (services cloud)

### Licences

Attribution des licences Microsoft 365 Business Standard aux utilisateurs principaux.

![Utilisateurs et licences M365](screenshots/m365/01-users-licenses.png)

### SharePoint

Site d'équipe `AzureLucas-IT` pour le stockage et la gestion documentaire (GED) — l'équivalent d'un serveur de fichiers dans le cloud.

![Site SharePoint AzureLucas-IT](screenshots/m365/02-sharepoint-site.png)

### Teams

Équipe `AzureLucas-IT` avec 3 canaux : Documentation, Incidents, Projets.

![Équipe Teams avec canaux](screenshots/m365/03-teams-channels.png)

**Séparation des services** : SharePoint gère le stockage documentaire, Teams la communication. Chaque équipe Teams a un site SharePoint associé automatiquement.

---

## Bloc 3 — Exchange Online (messagerie)

### Boîtes aux lettres

Boîtes utilisateurs provisionnées automatiquement avec les licences.

![Boîtes aux lettres Exchange](screenshots/exchange/01-mailboxes.png)

### Boîte partagée

Boîte partagée `Support IT` avec délégation (Envoyer en tant que + Accès total). Pas de licence requise — plusieurs techniciens peuvent lire et répondre depuis la même adresse. Cas d'usage classique : helpdesk, comptabilité, accueil.

![Boîte partagée Support IT](screenshots/exchange/02-shared-mailbox-support-it.png)

### Alias de messagerie

Ajout d'un alias `rh@` : un utilisateur reçoit des mails sur plusieurs adresses tout en n'ayant qu'une seule boîte.

![Alias de messagerie](screenshots/exchange/03-email-alias.png)

### Règle de flux de courrier

Règle qui ajoute automatiquement un avertissement sur tous les mails reçus de l'extérieur de l'organisation. Bonne pratique de sensibilisation anti-phishing.

![Règle de flux mail externe](screenshots/exchange/04-mail-flow-rule-external.png)

---

## Bloc 4 — Identité hybride (Entra Connect Sync)

### Objectif

Relier l'AD local `lucas.local` au tenant M365 : les utilisateurs créés on-premises sont automatiquement synchronisés vers Entra ID et accèdent aux services cloud avec le même compte.

### Étapes réalisées

1. **Préparation du serveur**
   - Désactivation de l'IE Enhanced Security Configuration (Administrateurs)
   - Installation de Chrome (portails M365 incompatibles avec IE)
   - Installation de .NET Framework 4.7.2 (prérequis Entra Connect sur Server 2016)

2. **Installation d'Entra Connect Sync**
   - Agent récupéré depuis le portail Entra (le Download Center est déprécié)
   - Choix de Connect Sync plutôt que Cloud Sync (ce dernier exige une licence Entra ID P1)
   - Mode Personnalisé (domaine non routable oblige)

3. **Configuration**
   - Méthode d'authentification : Password Hash Sync
   - Création automatique du compte de service `MSOL_xxxx` (moindre privilège)
   - Suffixe UPN non vérifié accepté (`lucas.local` non routable)
   - Synchronisation de tous les domaines / OU / utilisateurs

### Validation de la synchronisation

**Aucune erreur de synchronisation** dans Microsoft Entra Connect Health : 0 attribut en double, 0 discordance, 0 conflit.

![Connect Health — 0 erreurs](screenshots/entra-connect/01-sync-errors-zero.png)

**Synchronisation en temps réel** validée : après création d'un utilisateur dans l'AD local, un cycle delta forcé en PowerShell le fait remonter dans Entra ID en moins d'une minute.

```powershell
Import-Module "C:\Program Files\Microsoft Azure AD Sync\Bin\ADSync\ADSync.psd1"
Start-ADSyncSyncCycle -PolicyType Delta
```

![PowerShell — sync delta Success](screenshots/entra-connect/02-powershell-delta.png)

**Résultat dans Entra ID** : les utilisateurs AD local apparaissent avec Synchronisation = Oui.

![Utilisateurs synchronisés](screenshots/entra-connect/03-users-synced.png)

7 utilisateurs AD local synchronisés : Jean Dupont, Jean Marc, Lucas Admin, Marie Martin, Nassime Nadir, Pierre Durand, gg ggggggg.

### Concepts clés (défendables en entretien)

**Identité hybride** — L'AD local et Entra ID forment un seul annuaire synchronisé. Modèle dominant en entreprise : on garde l'AD pour les apps legacy (Kerberos/NTLM, GPO, partages réseau) tout en profitant des services cloud.

**Password Hash Sync (PHS)** — Entra Connect synchronise un hash du hash du mot de passe AD vers Entra ID. L'utilisateur accède au cloud avec le même mot de passe que sa session Windows. Méthode la plus simple, pas d'infra supplémentaire (contrairement à AD FS).

**Domaine non routable** — `lucas.local` n'existe pas sur internet, impossible à vérifier sur le tenant. Les users prennent le suffixe `@lcsbgt1207gmail.onmicrosoft.com`. En entreprise le domaine AD correspond à un domaine public vérifié (ex. `entreprise.fr`).

**GPO vs Intune**

| | AD local | Cloud |
|---|---|---|
| Outil | GPO | Intune |
| Portée | réseau interne | partout via internet |
| Télétravail | VPN obligatoire | aucun VPN |

---

## Limites rencontrées (et ce qu'elles apprennent)

| Limite | Cause | Enseignement |
|---|---|---|
| Cloud Sync indisponible | nécessite Entra ID P1 | différence Cloud Sync / Connect Sync |
| .NET 4.8.1 refusé | non supporté sur Server 2016 | version max = 4.8 sur cet OS |
| IdFix impossible | dépend de .NET 4.8.1 | optionnel en lab, recommandé en prod |
| Intune conformité bloqué | nécessite Intune Plan 1 | MDM complet = Business Premium |
| Connect Health services | nécessite Entra ID P1 | monitoring avancé = licence payante |

Ces limites documentent les **dépendances de licence réelles** entre les fonctionnalités Microsoft 365 — utile à connaître en entreprise.

---

## Pour aller plus loin

- [ ] Tester la suppression : supprimer un user AD local et vérifier dans Entra ID
- [ ] Activer la corbeille Active Directory sur la forêt
- [ ] Préparer les certifications AZ-900 et MS-900

---

## Auteur

**Lucas Bigot** — étudiant Bachelor DSNS, ESIEE-IT  
Objectif : alternance septembre 2026 — Support IT / Systèmes & Réseaux / Cybersécurité  
GitHub : [github.com/lcsbgt1207-web](https://github.com/lcsbgt1207-web)
