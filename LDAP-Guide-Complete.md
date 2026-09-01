# 🔐 Guide Complet de Configuration LDAP

## Table des Matières

1. [Introduction à LDAP](#introduction-à-ldap)
2. [Installation de slapd](#installation-de-slapd)
3. [Configuration de slapd.conf](#configuration-de-slapdconf)
4. [Initialisation de la base de données](#initialisation-de-la-base-de-données)
5. [Création des unités organisationnelles](#création-des-unités-organisationnelles)
6. [Ajout des utilisateurs](#ajout-des-utilisateurs)
7. [Création des groupes](#création-des-groupes)
8. [Vérification et tests](#vérification-et-tests)
9. [Prochaines étapes](#prochaines-étapes)

---

## Introduction à LDAP

**LDAP** (Lightweight Directory Access Protocol) est un protocole d'accès à des annuaires utilisés pour stocker et accéder à des informations organisées dans une structure hiérarchique.

### À quoi ça sert ?

- 🔐 **Authentification centralisée** : Un seul identifiant pour tous les services
- 📋 **Annuaire d'entreprise** : Gérer les utilisateurs et groupes
- 🔑 **Gestion des droits** : Contrôler les accès aux ressources

### Architecture

```
Client LDAP
    ↓
Requête (ldapsearch, ldapadd, etc.)
    ↓
Serveur LDAP (slapd)
    ↓
Base de données (MDB)
    ↓
Fichiers sur disque (/var/lib/ldap)
```

---

## Installation de slapd

### Commande d'installation

```bash
sudo apt-get update
sudo apt-get install slapd ldap-utils
```

### Vérification

```bash
sudo systemctl status slapd
```

✅ **Statut attendu** : `Active (running)`

---

## Configuration de slapd.conf

### Fichier de configuration

**Chemin** : `/etc/ldap/slapd.conf`

### Contenu complet

```conf
# Schémas LDAP (définissent les types d'objets)
include /etc/ldap/schema/core.schema
include /etc/ldap/schema/cosine.schema
include /etc/ldap/schema/nis.schema
include /etc/ldap/schema/inetorgperson.schema

# Fichiers système (pour la gestion du processus)
pidfile         /var/run/slapd/slapd.pid
argsfile        /var/run/slapd/slapd.args

# Configuration de la base de données
database        mdb
suffix          "dc=entreprise,dc=local"
rootdn          "cn=admin,dc=entreprise,dc=local"
rootpw          {SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1
directory       /var/lib/ldap

# Index pour les performances
index           objectClass eq
index           cn pres,sub,eq
index           mail eq
```

### Paramètres importants

| Paramètre | Signification |
|-----------|---------------|
| `include` | Charge les schémas LDAP |
| `database` | Type de base (mdb = Modern Database) |
| `suffix` | Domaine principal (racine) |
| `rootdn` | Admin de la base |
| `rootpw` | Mot de passe de l'admin (chiffré SSHA) |
| `directory` | Emplacement des données |
| `index` | Accélère les recherches |

### Mot de passe chiffré

Le mot de passe a été chiffré avec :

```bash
slappasswd
```

**Résultat** : `{SSHA}RqtLmD0tHJ+1XLRd6JFAuV1iNT00H1`

🔒 **Avantage** : Sécurité maximale, même si quelqu'un accède au fichier !

---

## Initialisation de la base de données

### Démarrage du service

```bash
sudo systemctl start slapd
```

### Vérification du statut

```bash
sudo systemctl status slapd
```

**Output attendu** :
```
Active: active (running) since Mon 2026-08-31 19:54:48 UTC; 3h 30min ago
```

### Test de connexion

```bash
ldapwhoami -H ldap:/// -D "cn=admin,dc=entreprise,dc=local" -w clems1082
```

**Output attendu** :
```
dn:cn=admin,dc=entreprise,dc=local
```

---

## Création des unités organisationnelles

### Fichier LDIF

**Chemin** : `/etc/ldap/complete_ldap.ldif`

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

### Ajout à la base

```bash
sudo ldapadd -x -D "cn=admin,dc=entreprise,dc=local" -W -f /etc/ldap/complete_ldap.ldif
```

**Output** :
```
adding new entry "cn=admin,dc=entreprise,dc=local"
adding new entry "ou=users,dc=entreprise,dc=local"
adding new entry "ou=groups,dc=entreprise,dc=local"
```

---

## Ajout des utilisateurs

### Fichier LDIF

**Chemin** : `/etc/ldap/users.ldif`

```ldif
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

### Ajout à la base

```bash
sudo ldapadd -x -D "cn=admin,dc=entreprise,dc=local" -W -f /etc/ldap/users.ldif
```

**Output** :
```
adding new entry "uid=jdupont,ou=users,dc=entreprise,dc=local"
adding new entry "uid=jmarie,ou=users,dc=entreprise,dc=local"
adding new entry "uid=bgates,ou=users,dc=entreprise,dc=local"
```

### Utilisateurs créés

| UID | Nom Complet | Email |
|-----|-------------|-------|
| jdupont | Jean Dupont | jdupont@entreprise.local |
| jmarie | Jean Marie | jmarie@entreprise.local |
| bgates | Bill Gates | bgates@entreprise.local |

---

## Création des groupes

### Fichier LDIF

**Chemin** : `/etc/ldap/groups.ldif`

```ldif
dn: cn=admins,ou=groups,dc=entreprise,dc=local
objectClass: top
objectClass: groupOfNames
cn: admins
member: uid=jdupont,ou=users,dc=entreprise,dc=local
member: uid=bgates,ou=users,dc=entreprise,dc=local

dn: cn=developers,ou=groups,dc=entreprise,dc=local
objectClass: top
objectClass: groupOfNames
cn: developers
member: uid=jmarie,ou=users,dc=entreprise,dc=local
member: uid=bgates,ou=users,dc=entreprise,dc=local

dn: cn=users,ou=groups,dc=entreprise,dc=local
objectClass: top
objectClass: groupOfNames
cn: users
member: uid=jdupont,ou=users,dc=entreprise,dc=local
member: uid=jmarie,ou=users,dc=entreprise,dc=local
member: uid=bgates,ou=users,dc=entreprise,dc=local
```

### Ajout à la base

```bash
sudo ldapadd -x -D "cn=admin,dc=entreprise,dc=local" -W -f /etc/ldap/groups.ldif
```

**Output** :
```
adding new entry "cn=admins,ou=groups,dc=entreprise,dc=local"
adding new entry "cn=developers,ou=groups,dc=entreprise,dc=local"
adding new entry "cn=users,ou=groups,dc=entreprise,dc=local"
```

### Groupes créés

| Groupe | Membres |
|--------|----------|
| **admins** | Jean Dupont, Bill Gates |
| **developers** | Jean Marie, Bill Gates |
| **users** | Jean Dupont, Jean Marie, Bill Gates |

---

## Vérification et tests

### 1. Vérifier tous les utilisateurs

```bash
getent passwd | grep entreprise
```

Ou directement avec LDAP :

```bash
ldapsearch -x -b "ou=users,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -w clems1082
```

### 2. Chercher un utilisateur spécifique

```bash
ldapsearch -x -b "dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -w clems1082 uid=jdupont
```

### 3. Vérifier les groupes

```bash
ldapsearch -x -b "ou=groups,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -w clems1082
```

### 4. Tester la connexion d'un utilisateur

```bash
ldapwhoami -H ldap:/// -D "uid=jdupont,ou=users,dc=entreprise,dc=local" -w clems1082
```

✅ **Output attendu** :
```
dn:uid=jdupont,ou=users,dc=entreprise,dc=local
```

---

## Prochaines étapes

### À configurer demain :

1. **PAM/NSS** - Intégration LDAP au système Linux
   - Configurer `/etc/nsswitch.conf`
   - Configurer `/etc/pam.d/`
   - Permettre aux utilisateurs LDAP de se connecter

2. **SSH avec LDAP**
   - Configurer SSH pour authentifier avec LDAP
   - Permettre les connexions SSH des utilisateurs LDAP

3. **Gestion des permissions**
   - Ajouter les utilisateurs aux groupes
   - Configurer les droits d'accès

4. **Sauvegarde et maintenance**
   - Backup de la base LDAP
   - Scripts de maintenance

---

## Résumé de la configuration

### Structure créée

```
dc=entreprise,dc=local
├── cn=admin
├── ou=users
│   ├── uid=jdupont
│   ├── uid=jmarie
│   └── uid=bgates
└── ou=groups
    ├── cn=admins
    ├── cn=developers
    └── cn=users
```

### Commandes utiles

```bash
# Démarrer slapd
sudo systemctl start slapd

# Vérifier le statut
sudo systemctl status slapd

# Chercher tous les utilisateurs
ldapsearch -x -b "ou=users,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -w clems1082

# Chercher tous les groupes
ldapsearch -x -b "ou=groups,dc=entreprise,dc=local" -D "cn=admin,dc=entreprise,dc=local" -w clems1082

# Ajouter des entrées LDIF
sudo ldapadd -x -D "cn=admin,dc=entreprise,dc=local" -W -f /chemin/fichier.ldif

# Modifier une entrée
sudo ldapmodify -x -D "cn=admin,dc=entreprise,dc=local" -W -f /chemin/modification.ldif

# Supprimer une entrée
sudo ldapdelete -x -D "cn=admin,dc=entreprise,dc=local" -W "uid=username,ou=users,dc=entreprise,dc=local"
```

---

## 📄 Informations du document

- **Date** : Septembre 2026
- **Serveur** : test-server
- **Version LDAP** : OpenLDAP 2.5.x
- **Authentification** : Mot de passe SSHA

---

## Notes importantes

✅ **Sécurité** : Tous les mots de passe sont chiffrés en SSHA
✅ **Schémas** : Configuration complète avec tous les schémas LDAP
✅ **Performance** : Index configurés pour optimiser les recherches
✅ **Flexibilité** : Prêt pour intégration PAM/NSS et SSH

---

**Document complet prêt pour utilisation !** 🚀