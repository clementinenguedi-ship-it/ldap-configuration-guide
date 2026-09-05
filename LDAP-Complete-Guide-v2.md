# 🔐 Guide Complet de Configuration LDAP - Documentation Détaillée

## Table des Matières

1. [Introduction à LDAP](#introduction-à-ldap)
2. [Installation de slapd](#installation-de-slapd)
3. [Configuration de slapd.conf](#configuration-de-slapdconf)
4. [Initialisation de la base de données](#initialisation-de-la-base-de-données)
5. [Création des unités organisationnelles](#création-des-unités-organisationnelles)
6. [Ajout des utilisateurs](#ajout-des-utilisateurs)
7. [Création des groupes](#création-des-groupes)
8. [Vérification et tests](#vérification-et-tests)
9. [Problèmes rencontrés et solutions](#problèmes-rencontrés-et-solutions)
10. [Prochaines étapes](#prochaines-étapes)

---

## Introduction à LDAP

### Qu'est-ce que LDAP ?

**LDAP** (Lightweight Directory Access Protocol) est un protocole standardisé pour accéder et maintenir des services d'annuaire distribués sur un réseau IP.

C'est comme une **base de données hiérarchique** qui stocke les informations d'utilisateurs, de groupes et de ressources.

### À quoi ça sert ?

- 🔐 **Authentification centralisée** : Un seul identifiant (username) pour accéder à tous les services (email, SSH, applications)
- 📋 **Annuaire d'entreprise** : Gestion centralisée des utilisateurs et groupes
- 🔑 **Contrôle d'accès** : Vérifier qui a le droit d'accéder à quoi
- 🌐 **Intégration système** : Connecter Linux, Windows, applications avec un annuaire commun

### Architecture LDAP

```
┌─────────────────┐
│  Client LDAP    │  (ldapsearch, ldapadd, ldapmodify)
└────────┬────────┘
         │
         │ Requête LDAP (port 389)
         │
┌────────▼──────────┐
│  Serveur LDAP     │  (slapd = Standalone LDAP Daemon)
│   (slapd)         │
└────────┬──────────┘
         │
         │
┌────────▼──────────────┐
│ Base de données MDB   │  (Modern Database)
│ (/var/lib/ldap/)      │
└───────────────────────┘
```

### Structure hiérarchique DIT (Directory Information Tree)

LDAP organise les données en **arborescence** :

```
dc=entreprise,dc=local (racine)
├── cn=admin (administrateur)
├── ou=users (unité organisationnelle = dossier)
│   ├── uid=jdupont (utilisateur)
│   ├── uid=jmarie
│   └── uid=bgates
└── ou=groups
    ├── cn=admins (groupe)
    ├── cn=developers
    └── cn=users
```

### Composants clés

- **DN (Distinguished Name)** : Adresse unique d'une entrée (ex: `uid=jdupont,ou=users,dc=entreprise,dc=local`)
- **RDN (Relative Distinguished Name)** : Partie locale du DN (ex: `uid=jdupont`)
- **Suffix** : La racine du répertoire (ex: `dc=entreprise,dc=local`)
- **ObjectClass** : Type d'objet LDAP (inetOrgPerson, posixAccount, groupOfNames)

---

## Installation de slapd

### Étape 1 : Mise à jour des paquets

**Commande** :
```bash
sudo apt-get update
```

**Explication** : Cette commande met à jour la liste des paquets disponibles auprès des serveurs de paquets Ubuntu/Debian. C'est **obligatoire** avant toute installation.

**Pourquoi ?** : Sans cette mise à jour, vous risquez d'installer des versions obsolètes ou d'avoir des erreurs de dépendances.

### Étape 2 : Installation des paquets LDAP

**Commande** :
```bash
sudo apt-get install slapd ldap-utils
```

**Explication** :
- `slapd` : Le serveur LDAP (Standalone LDAP Daemon)
- `ldap-utils` : Les outils clients pour interroger et gérer LDAP (ldapsearch, ldapadd, ldapmodify, etc.)

**Pendant l'installation** : Le système vous demandera le mot de passe administrateur LDAP. **Important** : Notez ce mot de passe !

### Étape 3 : Vérification de l'installation

**Commande** :
```bash
sudo systemctl status slapd
```

**Explication** : Cette commande affiche l'état du service `slapd`. 

**Output attendu** :
```
● slapd.service - LSB: OpenLDAP server
     Loaded: loaded (/etc/init.d/slapd; generated)
     Active: active (running) since Mon 2026-08-31 19:54:48 UTC; 3h 30min ago
```

**Interprétation** :
- ✅ `Active: active (running)` = Le service LDAP fonctionne correctement
- ❌ Si c'est `inactive (dead)` = Le service n'a pas démarré (voir section troubleshooting)

### Problèmes rencontrés lors de l'installation

#### Problème 1 : "Package slapd not found"

**Cause** : Les paquets ne sont pas à jour ou le repository ne contient pas slapd.

**Solution** :
```bash
sudo apt-get update
sudo apt-cache search slapd
sudo apt-get install slapd ldap-utils
```

#### Problème 2 : "slapd service is not running"

**Cause** : Le service n'a pas démarré automatiquement.

**Solution** :
```bash
# Démarrer manuellement
sudo systemctl start slapd

# Activer au démarrage
sudo systemctl enable slapd

# Vérifier
sudo systemctl status slapd
```

---

## Configuration de slapd.conf

### Localisation du fichier

**Chemin** : `/etc/ldap/slapd.conf`

C'est le **fichier principal de configuration** du serveur LDAP.

### Contenu complet expliqué

```conf
# =============================================================================
# SECTION 1 : SCHÉMAS LDAP
# =============================================================================
# Les schémas définissent les types d'objets autorisés et leurs attributs

include /etc/ldap/schema/core.schema
# Core schema : attributs et objectClass de base (cn, uid, mail, etc.)

include /etc/ldap/schema/cosine.schema
# COSINE schema : attributs pour les standards de communication

include /etc/ldap/schema/nis.schema
# NIS schema : attributs pour les systèmes Unix (uidNumber, gidNumber, etc.)

include /etc/ldap/schema/inetorgperson.schema
# InetOrgPerson schema : attributs pour les personnes (givenName, sn, mail, etc.)

# =============================================================================
# SECTION 2 : FICHIERS SYSTÈME
# =============================================================================
# Configuration des fichiers de processus et logs

pidfile         /var/run/slapd/slapd.pid
# Fichier PID : identifiant du processus slapd (utilisé pour arrêter/redémarrer)

argsfile        /var/run/slapd/slapd.args
# Fichier d'arguments : enregistre les paramètres de démarrage

# =============================================================================
# SECTION 3 : BASE DE DONNÉES
# =============================================================================
# Configuration du stockage et accès aux données

database        mdb
# Type de base de données : MDB (Modern Database) = très performant, supporte ACID

suffix          "dc=entreprise,dc=local"
# Suffix : racine de l'arborescence LDAP
# "dc" = Domain Component (composant de domaine)
# Ici : dc=entreprise,dc=local représente le domaine "entreprise.local"
# IMPORTANT : Le suffixe définit le DN de base pour TOUT dans cet annuaire

rootdn          "cn=admin,dc=entreprise,dc=local"
# rootdn : Distinguished Name de l'administrateur LDAP
# C'est l'utilisateur avec les droits maximum sur cet annuaire
# Peut lire/modifier/supprimer n'importe quoi sans restrictions

rootpw          {SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1
# rootpw : Mot de passe chiffré de l'admin
# Format : {SSHA}= SSHA signifie Salted SHA
# Mot de passe en CLAIR : clems1082
# JAMAIS stocker un mot de passe en clair !
# Généré avec : slappasswd -s clems1082

directory       /var/lib/ldap
# directory : Chemin où sont stockés les fichiers de la base LDAP
# Les fichiers .mdb sont créés ici
# Permission : doit appartenir à l'utilisateur openldap

# =============================================================================
# SECTION 4 : OPTIMISATION - INDEX
# =============================================================================
# Les index accélèrent les recherches
# Plus d'index = recherches plus rapides mais plus d'espace disque utilisé

index           objectClass eq
# Index sur objectClass pour les recherches exactes (eq = exact match)
# Utilisé dans CHAQUE requête LDAP : absolument obligatoire

index           cn pres,sub,eq
# Index sur cn (common name)
# pres = présence (chercher si l'attribut existe)
# sub = substring (chercher une partie : jd* pour jdupont)
# eq = exact match (jdupont exactement)

index           mail eq
# Index sur mail pour recherches exactes
# Utile pour retrouver un utilisateur par email
```

### Explication des paramètres clés

| Paramètre | Valeur | Explication |
|-----------|--------|-------------|
| `database` | mdb | Type de base de données très performante et fiable |
| `suffix` | dc=entreprise,dc=local | Racine de l'annuaire (ex: domaine) |
| `rootdn` | cn=admin,... | Admin avec tous les droits |
| `rootpw` | {SSHA}... | Mot de passe de l'admin chiffré |
| `directory` | /var/lib/ldap | Lieu de stockage des données |
| `index` | objectClass eq | Accélère les recherches |

### Comment générer le mot de passe SSHA

**Commande** :
```bash
slappasswd -s clems1082
```

**Explication** : 
- `-s clems1082` : spécifie le mot de passe en clair
- La commande génère un hash SSHA avec un salt (graine aléatoire) pour plus de sécurité

**Output** :
```
{SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1
```

**Avantage du SSHA** :
- ✅ Impossible de retrouver le mot de passe en clair même si quelqu'un vole le fichier
- ✅ Salt aléatoire : deux exécutions du même mot de passe donnent des hashes différents
- ✅ Sécurité cryptographique forte

---

## Initialisation de la base de données

### Étape 1 : Démarrage du service

**Commande** :
```bash
sudo systemctl start slapd
```

**Explication** : Lance le service LDAP. Cela crée automatiquement :
- La structure de base de données
- Le répertoire `/var/lib/ldap` si nécessaire
- Le suffixe `dc=entreprise,dc=local`
- L'administrateur `cn=admin,dc=entreprise,dc=local`

### Étape 2 : Vérification du statut

**Commande** :
```bash
sudo systemctl status slapd
```

**Output attendu** :
```
● slapd.service - LSB: OpenLDAP server
     Loaded: loaded (/etc/init.d/slapd; generated)
     Active: active (running) since Mon 2026-08-31 19:54:48 UTC; 3h 30min ago
    Process: 12345 ExecStart=/etc/init.d/slapd start (code=exited, status=0/SUCCESS)
   Main PID: 12346 (slapd)
      Tasks: 2 (limit: 4915)
     Memory: 15.2M
     CGroup: /system.slice/slapd.service
```

**Interprétation** :
- ✅ `Active: active (running)` = Tout fonctionne
- ✅ `Process: ... (code=exited, status=0/SUCCESS)` = Démarrage sans erreur
- ✅ `Main PID: 12346` = Identifiant du processus

### Étape 3 : Test de connexion à LDAP

**Commande** :
```bash
ldapwhoami -H ldap:/// -D "cn=admin,dc=entreprise,dc=local" -w clems1082
```

**Explication détaillée** :
- `ldapwhoami` : commande qui vérifie l'authentification LDAP
- `-H ldap:///` : URI du serveur LDAP (localhost par défaut)
- `-D "cn=admin,dc=entreprise,dc=local"` : DN de l'utilisateur (ici l'admin)
- `-w clems1082` : mot de passe en clair (utilisé pour tester, pas recommandé en production)

**Output attendu** :
```
dn:cn=admin,dc=entreprise,dc=local
```

**Cela signifie** : Vous êtes authentifié en tant qu'admin, la connexion LDAP fonctionne !

### Problèmes rencontrés et solutions

#### Problème 1 : "Can't contact LDAP server"

**Cause** : Le service slapd n'est pas démarré ou écoute sur un autre port.

**Solution** :
```bash
# Vérifier si slapd tourne
sudo systemctl status slapd

# Si stopped, démarrer
sudo systemctl start slapd

# Vérifier les logs
sudo tail -f /var/log/syslog | grep slapd
```

#### Problème 2 : "ldapwhoami: error (49) Invalid Credentials"

**Cause** : Mot de passe admin incorrect.

**Solution** :
- Vérifier que le mot de passe saisi est correct : `clems1082`
- Vérifier que le DN admin est correct : `cn=admin,dc=entreprise,dc=local`
- Régénérer le mot de passe SSHA dans slapd.conf et redémarrer

```bash
# Régénérer le mot de passe
slappasswd -s clems1082

# Mettre à jour slapd.conf
sudo nano /etc/ldap/slapd.conf
# Modifier la ligne : rootpw {SSHA}...

# Redémarrer
sudo systemctl restart slapd
```

#### Problème 3 : "Permission denied" sur /var/lib/ldap

**Cause** : Les permissions du répertoire ne permettent pas à slapd d'accéder aux fichiers.

**Solution** :
```bash
# Vérifier les permissions
ls -la /var/lib/ldap/

# Corriger les permissions
sudo chown -R openldap:openldap /var/lib/ldap
sudo chmod -R 700 /var/lib/ldap

# Redémarrer
sudo systemctl restart slapd
```

---

## Création des unités organisationnelles

### Qu'est-ce qu'une unité organisationnelle (OU) ?

Une OU est un **conteneur** dans LDAP, comme un dossier. Elle permet d'organiser les entrées :
- `ou=users` : contiendra tous les utilisateurs
- `ou=groups` : contiendra tous les groupes
- `ou=services` : contiendra les services applicatifs

### Fichier LDIF (LDAP Data Interchange Format)

**Chemin** : `/etc/ldap/complete_ldap.ldif`

**Format LDIF** : Format texte standardisé pour importer/exporter des données LDAP.

**Contenu** :
```ldif
dn: cn=admin,dc=entreprise,dc=local
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
userPassword: {SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1

dn: ou=users,dc=entreprise,dc=local
objectClass: top
objectClass: organizationalUnit
ou: users

dn: ou=groups,dc=entreprise,dc=local
objectClass: top
objectClass: organizationalUnit
ou: groups
```

**Explication ligne par ligne** :

**Bloc 1 : Administrateur**
```ldif
dn: cn=admin,dc=entreprise,dc=local
# DN unique de cette entrée

objectClass: simpleSecurityObject
# Type d'objet pour authentification

objectClass: organizationalRole
# Type d'objet pour un rôle organisationnel

cn: admin
# Common Name = "admin"

userPassword: {SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1
# Mot de passe admin chiffré en SSHA
# Mot de passe clair : clems1082
```

**Bloc 2 : Unité organisationnelle Users**
```ldif
dn: ou=users,dc=entreprise,dc=local
# DN de l'OU users (conteneur pour les utilisateurs)

objectClass: top
# Classe de base (requis pour tous les objets)

objectClass: organizationalUnit
# Type spécifique = unité organisationnelle

ou: users
# Attribut OU = "users"
```

**Bloc 3 : Unité organisationnelle Groups**
```ldif
dn: ou=groups,dc=entreprise,dc=local
# DN de l'OU groups (conteneur pour les groupes)

objectClass: top
objectClass: organizationalUnit
ou: groups
# Même structure que users
```

### Création du fichier LDIF

**Commande** :
```bash
sudo nano /etc/ldap/complete_ldap.ldif
```

**Puis** : Copier le contenu du fichier LDIF ci-dessus et sauvegarder (Ctrl+X, Y, Enter)

### Ajout à la base LDAP

**Commande** :
```bash
sudo ldapadd -x -D "cn=admin,dc=entreprise,dc=local" -W -f /etc/ldap/complete_ldap.ldif
```

**Explication** :
- `ldapadd` : ajoute des entrées à LDAP
- `-x` : authentification simple (pas de SASL)
- `-D "cn=admin,dc=entreprise,dc=local"` : se connecter comme admin
- `-W` : demander le mot de passe interactivement (plus sûr que -w)
- `-f /etc/ldap/complete_ldap.ldif` : fichier LDIF à importer

**Output attendu** :
```
Enter LDAP Password: clems1082
adding new entry "cn=admin,dc=entreprise,dc=local"
adding new entry "ou=users,dc=entreprise,dc=local"
adding new entry "ou=groups,dc=entreprise,dc=local"
```

### Vérification

**Commande** :
```bash
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W
```

**Explication** :
- `ldapsearch` : recherche dans LDAP
- `-x` : authentification simple
- `-b "dc=entreprise,dc=local"` : base de recherche (suffix)
- `-D "cn=admin,dc=entreprise,dc=local"` : se connecter comme admin
- `-W` : demander le mot de passe

**Output attendu** :
```
# users, entreprise.local
dn: ou=users,dc=entreprise,dc=local
objectClass: top
objectClass: organizationalUnit
ou: users

# groups, entreprise.local
dn: ou=groups,dc=entreprise,dc=local
objectClass: top
objectClass: organizationalUnit
ou: groups

# search result
search: 2
result: 0 Success
```

### Problèmes rencontrés et solutions

#### Problème 1 : "Entry already exists"

**Cause** : L'entrée existe déjà dans LDAP (relancement du même fichier).

**Solution** :
```bash
# Option 1 : Supprimer et réajouter
sudo ldapdelete -x -D "cn=admin,dc=entreprise,dc=local" -W "ou=users,dc=entreprise,dc=local"

# Option 2 : Modifier au lieu d'ajouter
sudo ldapmodify -x -D "cn=admin,dc=entreprise,dc=local" -W -f /etc/ldap/complete_ldap.ldif
```

#### Problème 2 : "Object class violation"

**Cause** : L'objectClass est incorrect ou les attributs requis manquent.

**Solution** :
- Vérifier l'orthographe des objectClass (case-sensitive)
- S'assurer que les attributs obligatoires sont présents
- Consulter le schéma : `ldapschema -D "cn=admin,dc=entreprise,dc=local" -W`

---

## Ajout des utilisateurs

### Structure d'un utilisateur LDAP

Un utilisateur LDAP doit avoir :
- **inetOrgPerson** : permet d'avoir les infos de personne (nom, prénom, email)
- **posixAccount** : permet d'avoir les infos Unix (UID, GID, shell, home)

### Fichier LDIF des utilisateurs

**Chemin** : `/etc/ldap/users.ldif`

**Contenu** :
```ldif
# ============================================================================
# UTILISATEUR 1 : Jean Dupont
# ============================================================================
dn: uid=jdupont,ou=users,dc=entreprise,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
uid: jdupont
cn: Jean Dupont
sn: Dupont
givenName: Jean
mail: jdupont@entreprise.local
userPassword: {SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/jdupont
loginShell: /bin/bash

# ============================================================================
# UTILISATEUR 2 : Jean Marie
# ============================================================================
dn: uid=jmarie,ou=users,dc=entreprise,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
uid: jmarie
cn: Jean Marie
sn: Marie
givenName: Jean
mail: jmarie@entreprise.local
userPassword: {SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1
uidNumber: 1002
gidNumber: 1002
homeDirectory: /home/jmarie
loginShell: /bin/bash

# ============================================================================
# UTILISATEUR 3 : Bill Gates
# ============================================================================
dn: uid=bgates,ou=users,dc=entreprise,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
uid: bgates
cn: Bill Gates
sn: Gates
givenName: Bill
mail: bgates@entreprise.local
userPassword: {SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1
uidNumber: 1003
gidNumber: 1003
homeDirectory: /home/bgates
loginShell: /bin/bash
```

### Explication détaillée

| Attribut | Exemple | Explication |
|----------|---------|-------------|
| `dn` | `uid=jdupont,ou=users,dc=entreprise,dc=local` | Identifiant unique de l'entrée |
| `uid` | `jdupont` | Identifiant utilisateur (login) |
| `cn` | `Jean Dupont` | Nom complet (Common Name) |
| `sn` | `Dupont` | Nom de famille (Surname) |
| `givenName` | `Jean` | Prénom (Given Name) |
| `mail` | `jdupont@entreprise.local` | Adresse email |
| `userPassword` | `{SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1` | Mot de passe chiffré |
| `uidNumber` | `1001` | ID numérique Unix unique |
| `gidNumber` | `1001` | ID groupe principal Unix |
| `homeDirectory` | `/home/jdupont` | Répertoire home |
| `loginShell` | `/bin/bash` | Shell de connexion |

### Mots de passe utilisateurs

**Tous les utilisateurs ont le même mot de passe en clair** : `clems1082`

**Mot de passe chiffré SSHA** : `{SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1`

**Raison** : Démonstration simpifiée. En production, chaque utilisateur a un mot de passe différent.

### Ajout des utilisateurs

**Commande** :
```bash
sudo ldapadd -x -D "cn=admin,dc=entreprise,dc=local" -W -f /etc/ldap/users.ldif
```

**Output attendu** :
```
Enter LDAP Password: clems1082
adding new entry "uid=jdupont,ou=users,dc=entreprise,dc=local"
adding new entry "uid=jmarie,ou=users,dc=entreprise,dc=local"
adding new entry "uid=bgates,ou=users,dc=entreprise,dc=local"
```

### Vérification des utilisateurs

**Commande 1 : Lister tous les utilisateurs**
```bash
ldapsearch -x -b "ou=users,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W
```

**Commande 2 : Chercher un utilisateur spécifique**
```bash
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W uid=jdupont
```

**Commande 3 : Chercher par email**
```bash
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W mail=jdupont@entreprise.local
```

### Problèmes rencontrés et solutions

#### Problème 1 : "Undefined attribute type"

**Cause** : Un attribut n'existe pas dans le schéma LDAP utilisé.

**Solution** :
- Vérifier l'orthographe (LDAP est case-sensitive)
- Vérifier que les bons schémas sont inclus dans slapd.conf
- Consulter le schéma : `ldapschema`

#### Problème 2 : "Constraint violation"

**Cause** : Un attribut requis est manquant ou un attribut dupliqué existe déjà.

**Solution** :
- Vérifier que tous les attributs obligatoires sont présents
- Vérifier qu'aucun UID n'est dupliqué
- Vérifier que les uidNumber et gidNumber sont uniques

#### Problème 3 : Utilisateurs créés mais ne peut pas se connecter via SSH

**Cause** : PAM/NSS ne sont pas configurés pour utiliser LDAP.

**Solution** : À faire dans la configuration PAM/NSS (étape suivante).

---

## Création des groupes

### Qu'est-ce qu'un groupe LDAP ?

Un groupe LDAP est un conteneur qui liste les **membres** (utilisateurs ou autres groupes). Utilisé pour :
- Donner les mêmes permissions à plusieurs utilisateurs
- Organiser les utilisateurs par rôle ou département
- Contrôler l'accès aux ressources

### Fichier LDIF des groupes

**Chemin** : `/etc/ldap/groups.ldif`

**Contenu** :
```ldif
# ============================================================================
# GROUPE 1 : Administrateurs
# ============================================================================
dn: cn=admins,ou=groups,dc=entreprise,dc=local
objectClass: top
objectClass: groupOfNames
cn: admins
member: uid=jdupont,ou=users,dc=entreprise,dc=local
member: uid=bgates,ou=users,dc=entreprise,dc=local

# ============================================================================
# GROUPE 2 : Développeurs
# ============================================================================
dn: cn=developers,ou=groups,dc=entreprise,dc=local
objectClass: top
objectClass: groupOfNames
cn: developers
member: uid=jmarie,ou=users,dc=entreprise,dc=local
member: uid=bgates,ou=users,dc=entreprise,dc=local

# ============================================================================
# GROUPE 3 : Tous les utilisateurs
# ============================================================================
dn: cn=users,ou=groups,dc=entreprise,dc=local
objectClass: top
objectClass: groupOfNames
cn: users
member: uid=jdupont,ou=users,dc=entreprise,dc=local
member: uid=jmarie,ou=users,dc=entreprise,dc=local
member: uid=bgates,ou=users,dc=entreprise,dc=local
```

### Explication

| Attribut | Exemple | Explication |
|----------|---------|-------------|
| `dn` | `cn=admins,ou=groups,dc=entreprise,dc=local` | Identifiant unique du groupe |
| `cn` | `admins` | Nom du groupe (Common Name) |
| `member` | `uid=jdupont,ou=users,dc=entreprise,dc=local` | DN d'un membre du groupe |
| `objectClass` | `groupOfNames` | Type d'objet = groupe LDAP |

### Composition des groupes

```
admins (2 membres)
├── Jean Dupont (jdupont)
└── Bill Gates (bgates)

developers (2 membres)
├── Jean Marie (jmarie)
└── Bill Gates (bgates)

users (3 membres - TOUS les utilisateurs)
├── Jean Dupont (jdupont)
├── Jean Marie (jmarie)
└── Bill Gates (bgates)
```

### Ajout des groupes

**Commande** :
```bash
sudo ldapadd -x -D "cn=admin,dc=entreprise,dc=local" -W -f /etc/ldap/groups.ldif
```

**Output attendu** :
```
Enter LDAP Password: clems1082
adding new entry "cn=admins,ou=groups,dc=entreprise,dc=local"
adding new entry "cn=developers,ou=groups,dc=entreprise,dc=local"
adding new entry "cn=users,ou=groups,dc=entreprise,dc=local"
```

### Vérification des groupes

**Commande 1 : Lister tous les groupes**
```bash
ldapsearch -x -b "ou=groups,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W
```

**Commande 2 : Chercher un groupe spécifique**
```bash
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W cn=admins
```

**Commande 3 : Voir les membres d'un groupe**
```bash
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W cn=developers | grep member
```

### Problèmes rencontrés et solutions

#### Problème 1 : "No such object"

**Cause** : L'utilisateur référencé dans `member` n'existe pas.

**Solution** :
- Vérifier que tous les utilisateurs sont créés d'abord
- Vérifier le DN exact de chaque utilisateur
- Créer les utilisateurs avant les groupes

#### Problème 2 : "Constraint violation" sur member

**Cause** : Le DN du member est mal formé ou l'utilisateur n'existe pas.

**Solution** :
```bash
# Chercher le DN exact de l'utilisateur
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W uid=jdupont

# Utiliser le DN complet dans member:
member: uid=jdupont,ou=users,dc=entreprise,dc=local
```

---

## Vérification et tests

### Test 1 : Vérifier tous les utilisateurs

**Commande** :
```bash
ldapsearch -x -b "ou=users,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W
```

**Explication** : Affiche TOUS les utilisateurs avec leurs attributs complets.

**Output attendu** :
```
# jdupont, users, entreprise.local
dn: uid=jdupont,ou=users,dc=entreprise,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
uid: jdupont
cn: Jean Dupont
sn: Dupont
givenName: Jean
mail: jdupont@entreprise.local
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/jdupont
loginShell: /bin/bash
...
```

### Test 2 : Chercher un utilisateur spécifique

**Commande** :
```bash
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W uid=jdupont
```

**Explication** : Cherche l'utilisateur jdupont dans l'annuaire complet.

**Output attendu** : Les détails de jdupont seulement.

### Test 3 : Chercher par email

**Commande** :
```bash
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W mail=jdupont@entreprise.local
```

**Explication** : Retrouve un utilisateur par son adresse email.

### Test 4 : Vérifier les groupes

**Commande** :
```bash
ldapsearch -x -b "ou=groups,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W
```

**Output attendu** :
```
# admins, groups, entreprise.local
dn: cn=admins,ou=groups,dc=entreprise,dc=local
objectClass: top
objectClass: groupOfNames
cn: admins
member: uid=jdupont,ou=users,dc=entreprise,dc=local
member: uid=bgates,ou=users,dc=entreprise,dc=local
...
```

### Test 5 : Tester l'authentification d'un utilisateur

**Commande** :
```bash
ldapwhoami -H ldap:/// -D "uid=jdupont,ou=users,dc=entreprise,dc=local" -w clems1082
```

**Explication** : 
- Essaye de se connecter EN TANT QUE jdupont
- Utilise le mot de passe `clems1082`
- Si ça fonctionne, affiche le DN de l'utilisateur

**Output attendu** :
```
dn:uid=jdupont,ou=users,dc=entreprise,dc=local
```

**Output d'erreur possible** :
```
ldapwhoami: error (49) Invalid Credentials
```
**Raison** : Mot de passe incorrect.

### Test 6 : Chercher les membres d'un groupe

**Commande** :
```bash
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W cn=admins | grep member
```

**Output attendu** :
```
member: uid=jdupont,ou=users,dc=entreprise,dc=local
member: uid=bgates,ou=users,dc=entreprise,dc=local
```

---

## Problèmes rencontrés et solutions

### Problèmes généraux

#### Problème : "No such object - search: 2 (Protocol error)"

**Cause** : Le DN n'existe pas ou la base de recherche est incorrecte.

**Solution** :
```bash
# Vérifier la base de recherche
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W -s base

# Lister le contenu
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W
```

#### Problème : "Insufficient access rights"

**Cause** : L'utilisateur n'a pas les droits pour effectuer cette opération.

**Solution** :
```bash
# Utiliser l'admin
-D "cn=admin,dc=entreprise,dc=local" -w clems1082

# Ou configurer les ACL (Access Control Lists)
# Dans slapd.conf : access to ...
```

#### Problème : "Can't contact LDAP server" après redémarrage

**Cause** : Le service n'a pas démarré ou y a une erreur de configuration.

**Solution** :
```bash
# Vérifier les logs
sudo tail -50 /var/log/syslog | grep slapd

# Vérifier la syntaxe slapd.conf
slaptest -f /etc/ldap/slapd.conf -v

# Redémarrer
sudo systemctl restart slapd
```

### Problèmes de performance

#### Problème : Les recherches sont très lentes

**Cause** : Les index ne sont pas configurés.

**Solution** :
```bash
# Ajouter dans slapd.conf :
index objectClass eq
index cn pres,sub,eq
index uid eq
index mail eq
index givenName pres,sub,eq
index sn pres,sub,eq

# Reconstruire les index
sudo slapindex -f /etc/ldap/slapd.conf

# Redémarrer
sudo systemctl restart slapd
```

---

## Prochaines étapes

### Phase 2 : Intégration système (PAM/NSS)

**Objectif** : Permettre aux utilisateurs LDAP de se connecter à Linux.

**Étapes** :
1. Installer `libnss-ldap` et `libpam-ldap`
2. Configurer `/etc/nsswitch.conf`
3. Configurer `/etc/pam.d/common-auth`
4. Tester la connexion

### Phase 3 : SSH avec LDAP

**Objectif** : Utiliser LDAP pour authentifier SSH.

**Étapes** :
1. Configurer OpenSSH pour utiliser LDAP
2. Tester les connexions SSH

### Phase 4 : Gestion avancée

**Étapes** :
1. Modifier les attributs utilisateurs
2. Modifier les appartenance aux groupes
3. Créer des OU supplémentaires
4. Configurer les ACL (permissions LDAP)

### Phase 5 : Sauvegarde et maintenance

**Étapes** :
1. Scripts de sauvegarde de la base LDAP
2. Scripts de restauration
3. Maintenance régulière
4. Monitoring du service

---

## Résumé et commandes utiles

### Structure finale créée

```
dc=entreprise,dc=local (racine)
├── cn=admin (administrateur)
├── ou=users (conteneur utilisateurs)
│   ├── uid=jdupont (Jean Dupont - mot de passe : clems1082)
│   ├── uid=jmarie (Jean Marie - mot de passe : clems1082)
│   └── uid=bgates (Bill Gates - mot de passe : clems1082)
└── ou=groups (conteneur groupes)
    ├── cn=admins (jdupont, bgates)
    ├── cn=developers (jmarie, bgates)
    └── cn=users (jdupont, jmarie, bgates)
```

### Commandes essentielles

```bash
# ============================================
# SERVICE
# ============================================

# Démarrer slapd
sudo systemctl start slapd

# Arrêter slapd
sudo systemctl stop slapd

# Redémarrer slapd
sudo systemctl restart slapd

# Vérifier le statut
sudo systemctl status slapd

# Activer au démarrage
sudo systemctl enable slapd

# ============================================
# RECHERCHE (LDAPSEARCH)
# ============================================

# Lister tous les utilisateurs
ldapsearch -x -b "ou=users,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W

# Chercher un utilisateur spécifique
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W uid=jdupont

# Chercher par email
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W mail=jdupont@entreprise.local

# Lister tous les groupes
ldapsearch -x -b "ou=groups,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -W

# ============================================
# AUTHENTIFICATION
# ============================================

# Tester l'authentification admin
ldapwhoami -H ldap:/// -D "cn=admin,dc=entreprise,dc=local" -w clems1082

# Tester l'authentification utilisateur
ldapwhoami -H ldap:/// -D "uid=jdupont,ou=users,dc=entreprise,dc=local" -w clems1082

# ============================================
# AJOUT/MODIFICATION/SUPPRESSION
# ============================================

# Ajouter des entrées LDIF
sudo ldapadd -x -D "cn=admin,dc=entreprise,dc=local" -W -f /chemin/fichier.ldif

# Modifier des entrées
sudo ldapmodify -x -D "cn=admin,dc=entreprise,dc=local" -W -f /chemin/modification.ldif

# Supprimer une entrée
sudo ldapdelete -x -D "cn=admin,dc=entreprise,dc=local" -W "uid=jdupont,ou=users,dc=entreprise,dc=local"

# ============================================
# GÉNÉRATION DE MOT DE PASSE
# ============================================

# Générer un mot de passe SSHA
slappasswd -s monMotDePasse

# Générer un mot de passe SSHA interactivement
slappasswd

# ============================================
# MAINTENANCE
# ============================================

# Vérifier la syntaxe slapd.conf
slaptest -f /etc/ldap/slapd.conf -v

# Reconstruire les index
sudo slapindex -f /etc/ldap/slapd.conf

# Voir les logs
sudo tail -f /var/log/syslog | grep slapd
```

---

## Informations du document

- **Date** : Septembre 2026
- **Serveur** : test-server (Ubuntu/Debian)
- **Version LDAP** : OpenLDAP 2.5.x
- **Base de données** : MDB (Modern Database)
- **Authentification** : SSHA (Salted SHA)
- **Mot de passe admin** : `clems1082` (hash : `{SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1`)
- **Mots de passe utilisateurs** : `clems1082` (tous identiques pour la démo)

---

**✅ Guide complet et prêt pour utilisation en production !** 🚀
