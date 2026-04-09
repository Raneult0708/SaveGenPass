# Cahier des charges

## Projet : SaveGenPass — Générateur et gestionnaire local de mots de passe

---

## 1. Présentation du projet

### 1.1 Contexte

La généralisation des services numériques (réseaux sociaux, messageries, services bancaires, plateformes professionnelles, outils éducatifs) contraint les utilisateurs à gérer un volume croissant de mots de passe distincts. Cette multiplication engendre trois comportements problématiques récurrents :

- La réutilisation du même mot de passe sur plusieurs services, qui expose l'ensemble des comptes dès qu'un seul service est compromis.
- La notation des mots de passe sur des supports non sécurisés (papier, fichier texte, notes de téléphone).
- L'abandon des bonnes pratiques de sécurité au profit de mots de passe courts et mémorisables, donc prévisibles.

Les solutions existantes (gestionnaires en ligne : LastPass, 1Password, Bitwarden cloud) répondent partiellement à ce besoin mais introduisent des dépendances réseau, des abonnements payants et une confiance implicite dans un tiers. SaveGenPass vise à offrir une alternative locale, transparente et souveraine, sans aucune donnée transmise à l'extérieur.

### 1.2 Objectifs

**Objectif principal :** Développer une application locale permettant la génération de mots de passe sécurisés, leur stockage chiffré dans un fichier local, et leur récupération après authentification par mot de passe maître — sans connexion Internet, sans base de données, sans serveur.

**Objectifs secondaires :**

- Mettre en place une architecture cryptographique robuste à trois niveaux : dérivation de clé résistante au bruteforce (Argon2id), chiffrement asymétrique des clés d'entrée (RSA-4096-OAEP), chiffrement symétrique authentifié des données (AES-256-GCM).
- Garantir qu'aucun élément secret (mot de passe maître, clé privée RSA en clair, mots de passe stockés) n'est jamais écrit sur le disque en clair.
- Fournir une interface graphique moderne développée avec **Qt for Python (PySide6)** via **Qt Creator**.
- Sensibiliser l'utilisateur aux bonnes pratiques de sécurité via des indicateurs visuels et messages contextuels.
- Permettre la personnalisation fine des mots de passe générés.

**Résultats attendus :**

- Un exécutable fonctionnel sur Linux et Windows, 100 % hors ligne.
- Un fichier `vault.enc` chiffré, inutilisable sans le mot de passe maître.
- Une interface graphique Qt Widgets fluide avec fenêtres distinctes par fonctionnalité.

### 1.3 Périmètre

**Inclus :**

- Génération de mots de passe paramétrables (longueur, types de caractères)
- Interface graphique Qt Widgets (PySide6) développée sous Qt Creator
- Création et gestion d'un mot de passe maître
- Stockage local chiffré dans `vault.enc`
- Architecture cryptographique complète : Argon2id + RSA-4096-OAEP + AES-256-GCM
- Consultation, ajout et suppression d'entrées après authentification
- Fonctionnement 100 % hors ligne
- Compilation en exécutables PE (Windows) et ELF (Linux) via PyInstaller

**Exclus :**

- Connexion Internet, synchronisation cloud
- Gestion de comptes multi-utilisateurs
- Récupération du mot de passe maître en cas d'oubli (par conception cryptographique)
- Base de données relationnelle
- Application mobile

---

## 2. Spécifications fonctionnelles

### 2.1 Fonctionnalités principales

**Génération de mot de passe :**

- Choisir la longueur totale du mot de passe (entre 8 et 128 caractères)
- Définir le nombre minimal de lettres minuscules (a-z)
- Définir le nombre minimal de lettres majuscules (A-Z)
- Définir le nombre minimal de chiffres (0-9)
- Définir le nombre minimal de caractères spéciaux (!, @, #, $, %, etc.)
- Vérification de la cohérence des paramètres (somme des minimums ≤ longueur totale)
- Génération aléatoire via le module `secrets` de Python (cryptographiquement sûr)
- Affichage du mot de passe généré avec option de copie dans le presse-papier
- Indicateur visuel de robustesse (entropie calculée en bits)

**Gestion sécurisée du vault :**

- Création du mot de passe maître lors de la première utilisation, avec confirmation
- Dérivation de la KEK (Key Encryption Key) depuis le mot de passe maître via Argon2id
- Génération d'une paire RSA-4096 lors du setup initial
- Chiffrement de la clé privée RSA par AES-256-GCM(KEK) avant tout stockage
- Association de chaque entrée à un nom de service (Gmail, GitHub, etc.) et une note optionnelle
- Chiffrement de chaque mot de passe avec une clé AES-256 aléatoire unique par entrée
- Chiffrement de cette clé AES par la clé publique RSA-4096-OAEP
- Stockage de l'ensemble dans `vault.enc` (JSON chiffré)
- Consultation des entrées uniquement après authentification par mot de passe maître
- Suppression individuelle d'une entrée sans rechiffrement des autres

### 2.2 Fonctionnalités secondaires (optionnelles)

- Évaluation de la robustesse d'un mot de passe saisi manuellement
- Auto-verrouillage de l'application après une période d'inactivité paramétrable
- Export sécurisé du vault pour sauvegarde
- Générateur de phrases mémorisables (passphrase)
- Détection de mots de passe dupliqués dans le vault

### 2.3 Cas d'utilisation

**Cas 1 — Première utilisation (setup) :**

1. L'utilisateur lance SaveGenPass pour la première fois
2. L'application détecte l'absence de `vault.enc` et ouvre la fenêtre de configuration initiale
3. L'utilisateur saisit et confirme son mot de passe maître
4. Un salt aléatoire de 16 bytes est généré (`salt_kdf`) via `os.urandom()`
5. `Argon2id(mot de passe maître, salt_kdf)` produit la KEK de 256 bits
6. Une paire RSA-4096 est générée (durée : 3 à 8 secondes, barre de progression affichée)
7. La clé privée RSA est chiffrée avec `AES-256-GCM(KEK)` et stockée dans `vault.enc`
8. La clé publique RSA est stockée en clair dans `vault.enc`
9. La KEK est effacée de la RAM. Le vault vide est initialisé

**Cas 2 — Génération et sauvegarde d'un mot de passe :**

1. L'utilisateur définit les paramètres dans l'interface de génération
2. SaveGenPass vérifie la cohérence des paramètres
3. Un mot de passe aléatoire est généré via `secrets.choice()`
4. L'utilisateur saisit le nom du service et choisit de sauvegarder
5. Une clé AES-256 aléatoire (`clé_entrée`) est générée pour cette entrée
6. Le mot de passe est chiffré : `AES-256-GCM(clé_entrée)` → `ct_data + nonce + tag`
7. La clé d'entrée est chiffrée : `RSA-4096-OAEP(clé_publique, clé_entrée)` → `ct_key`
8. L'entrée `{service, ct_key, ct_data, nonce, tag}` est ajoutée à `vault.enc`
9. La `clé_entrée` est effacée de la RAM

**Cas 3 — Consultation des mots de passe :**

1. L'utilisateur saisit son mot de passe maître dans la fenêtre d'authentification
2. `Argon2id(mot de passe maître, salt_kdf)` produit la KEK candidate
3. AES-256-GCM tente de déchiffrer la clé privée RSA avec la KEK candidate
4. Si le tag GCM est invalide → mauvais mot de passe → accès refusé immédiatement
5. Si succès → clé privée RSA chargée en RAM uniquement
6. Pour chaque entrée demandée : RSA-OAEP déchiffre `ct_key` → `clé_entrée`, puis AES-GCM déchiffre `ct_data` → mot de passe en clair
7. Les mots de passe sont affichés masqués par défaut, révélables à la demande
8. À la fermeture de session, clé privée RSA et clés d'entrée sont effacées de la RAM

---

## 3. Contraintes techniques

### 3.1 Technologies utilisées

**Langage :**

- Python 3.10+

**Interface graphique :**

- **PySide6** — binding officiel Qt6, licence LGPL
- **Qt Creator** — IDE utilisé pour la conception des fenêtres Qt Widgets (fichiers `.ui`)
- Template utilisé : **Qt Widgets Application (Qt for Python)** sous Qt Creator
- Conception visuelle des interfaces via Qt Designer intégré à Qt Creator
- Connexion des signaux et slots en Python

**Bibliothèques cryptographiques :**

- `cryptography` (PyCA) — AES-256-GCM, RSA-4096-OAEP, primitives bas niveau
- `argon2-cffi` — implémentation Argon2id pour la dérivation de clé
- `secrets` (stdlib Python) — génération cryptographiquement sûre des mots de passe et valeurs aléatoires
- `os.urandom()` — génération des salts et nonces

**Packaging et distribution :**

- PyInstaller — compilation en exécutables PE (Windows `.exe`) et ELF (Linux)
- Git — gestion de version

### 3.2 Contraintes

**Matérielles :**

- RAM minimale : 256 MB (Argon2id avec `m=65536` requiert ~64 MB lors de la dérivation)
- Espace disque : moins de 50 MB pour l'exécutable packagé

**Logicielles :**

- Python 3.10 minimum
- Compatible Linux (Ubuntu 20.04+) et Windows (10/11)
- Aucun accès réseau requis ni autorisé

**Compatibilité :**

- `vault.enc` doit rester lisible entre les versions (numéro de version inclus dans le fichier)
- Encodage UTF-8 obligatoire pour tous les noms de service et mots de passe

### 3.3 Sécurité

**Architecture cryptographique — trois couches imbriquées :**

**Couche 1 — Argon2id (dérivation de clé) :**

Argon2id est le gagnant du Password Hashing Competition (2015) et la recommandation actuelle du NIST et de l'OWASP. Il est intentionnellement lent et gourmand en mémoire pour contrecarrer le bruteforce GPU.

Paramètres retenus :

- Mémoire `m` : 65 536 KB (64 MB)
- Itérations `t` : 3 passes
- Parallélisme `p` : 4 threads
- Sortie : 32 bytes (256 bits) — la KEK
- Salt : 16 bytes aléatoires via `os.urandom()`, stocké en clair dans `vault.enc`
- Durée estimée : ~200 ms (intentionnel)

**Couche 2 — RSA-4096-OAEP (chiffrement asymétrique des clés d'entrée) :**

Une paire RSA-4096 est générée au setup. La clé publique est stockée en clair. La clé privée est chiffrée par AES-256-GCM(KEK) avant d'être stockée — elle n'existe jamais en clair sur le disque. Le padding OAEP (MGF1-SHA256) est obligatoire ; RSA sans padding est vulnérable. Chaque entrée du vault possède sa propre clé AES-256 chiffrée par RSA, ce qui permet la suppression individuelle sans rechiffrement global.

Paramètres retenus :

- Taille de clé : 4096 bits
- Padding : OAEP avec MGF1-SHA256
- Usage exclusif : chiffrement des clés AES des entrées (jamais des données directement)

**Couche 3 — AES-256-GCM (chiffrement symétrique authentifié) :**

AES-GCM est un mode AEAD (Authenticated Encryption with Associated Data). Il chiffre et authentifie simultanément. Le tag GCM de 128 bits permet la vérification implicite du mot de passe maître : si la KEK est incorrecte, le tag ne correspond pas et une exception est levée — sans qu'aucune information sur les données ne soit révélée. AES-GCM est utilisé à deux niveaux : chiffrement de la clé privée RSA (avec KEK), et chiffrement de chaque mot de passe (avec la clé d'entrée dédiée).

Paramètres retenus :

- Clé : 256 bits
- Nonce : 96 bits aléatoires via `os.urandom()`, regénéré à chaque chiffrement
- Tag : 128 bits (vérification d'intégrité et d'authenticité)

**Vérification du mot de passe maître :**

> Le mot de passe maître n'est **jamais** stocké — ni en clair, ni hashé, ni chiffré. La vérification est implicite : si Argon2id produit la bonne KEK, AES-GCM déchiffre la clé privée RSA avec un tag valide. Sinon, exception levée et accès refusé. C'est le tag GCM qui joue ce rôle, pas un hash séparé.

**Structure de `vault.enc` (ce qui est sur le disque) :**

```json
{
  "version": 1,
  "kdf": "argon2id",
  "kdf_params": { "m": 65536, "t": 3, "p": 4 },
  "salt_kdf": "<base64 16 bytes>",
  "rsa_public_key": "<PEM en clair>",
  "rsa_private_key_nonce": "<base64 12 bytes>",
  "rsa_private_key_ct": "<clé privée RSA chiffrée AES-GCM>",
  "rsa_private_key_tag": "<base64 16 bytes>",
  "entries": {
    "gmail": {
      "ct_key": "<clé AES chiffrée par RSA-OAEP>",
      "ct_data": "<mot de passe chiffré par AES-GCM>",
      "nonce": "<base64 12 bytes>",
      "tag": "<base64 16 bytes>"
    }
  }
}
```

**Aucun secret n'est jamais écrit sur le disque en clair.** La clé privée RSA est le seul élément secret stocké, et elle est elle-même chiffrée.

**Bonnes pratiques appliquées :**

- Utilisation de `secrets` au lieu de `random` pour toute génération aléatoire
- Effacement des variables sensibles de la RAM après usage
- Nonce unique regénéré à chaque opération de chiffrement
- Aucune tentative de récupération du mot de passe maître par design

**Limitations :**

- Sans le mot de passe maître, le vault est définitivement inaccessible (par conception)
- RSA-4096 prend 3 à 8 secondes à la génération (une seule fois au setup)
- Argon2id introduit ~200 ms de délai au login (intentionnel, résistance bruteforce)

---

## 4. Design et ergonomie

### 4.1 Interface utilisateur

- Interface développée avec **PySide6 (Qt Widgets)** sous **Qt Creator**
    
- Les fichiers `.ui` sont conçus dans Qt Designer (intégré à Qt Creator) et chargés dynamiquement en Python via `QUiLoader` ou convertis avec `pyside6-uic`
    
- Fenêtres distinctes par fonctionnalité :
    
    - Écran de setup — création du mot de passe maître + barre de progression génération RSA
    - Écran de login — saisie du mot de passe maître, masqué par défaut
    - Écran principal — tableau de bord d'accès aux fonctionnalités
    - Écran de génération — formulaire paramétrable avec indicateur d'entropie en temps réel
    - Écran du vault — liste des entrées avec boutons Afficher / Copier / Supprimer
- Composants Qt Widgets utilisés : `QLineEdit`, `QSpinBox`, `QPushButton`, `QTableWidget`, `QProgressBar`, `QMessageBox`, `QLabel`
    
- Champs de saisie pour les paramètres du mot de passe et le mot de passe maître
    
- Boutons : Générer, Sauvegarder, Afficher, Supprimer, Copier
    

### 4.2 Expérience utilisateur

- Navigation simple, aucune connaissance cryptographique requise
- Messages explicites : "Mot de passe maître incorrect", "Longueur insuffisante pour les contraintes définies", etc.
- Feedback visuel immédiat sur la robustesse du mot de passe (barre colorée : rouge / orange / vert)
- Confirmation demandée avant toute suppression d'entrée
- Mots de passe masqués par défaut, révélables individuellement
- Copie dans le presse-papier avec effacement automatique après 30 secondes
- Indication claire lors du setup que le mot de passe maître est irrécupérable par conception

---

## 5. Livrables et planning

### 5.1 Livrables

- Code source Python structuré en modules (`crypto.py`, `vault.py`, `generator.py`, `ui/`)
- Fichiers `.ui` Qt Creator pour chaque fenêtre
- Interface graphique PySide6 fonctionnelle
- Fichier `vault.enc` d'exemple chiffré pour démonstration
- Cahier des charges
- README général : installation, dépendances, utilisation, architecture cryptographique
- Commentaires et documentation interne (docstrings PEP 257)
- Exécutables PyInstaller : `SaveGenPass.exe` (Windows) et `SaveGenPass` (Linux ELF)

### 5.2 Planning prévisionnel

|Étape|Description|Durée|
|---|---|---|
|Analyse|Définition des besoins, choix cryptographiques, validation de l'architecture|1/2 jour|
|Conception|Architecture des modules, structure vault.enc, schémas cryptographiques|1/2 jour|
|Module crypto|Implémentation Argon2id + RSA-4096-OAEP + AES-256-GCM + tests unitaires|1,5 jour|
|Module vault|Lecture/écriture vault.enc, gestion des entrées, authentification|1 jour|
|Module generator|Générateur de mots de passe via secrets, calcul entropie|1/2 jour|
|Interface Qt|Écrans PySide6 via Qt Creator, connexion signaux/slots|1,5 jour|
|Intégration|Liaison UI ↔ crypto ↔ vault, tests end-to-end|1/2 jour|
|Tests|Sécurité (mauvais mot de passe, intégrité vault) et ergonomie|1 jour|
|Packaging|PyInstaller PE + ELF, tests sur Linux et Windows|1/2 jour|
|Documentation|README, docstrings, commentaires|1/2 jour|

---

## 6. Critères de validation

Le projet sera validé si :

- Les mots de passe sont correctement générés selon les paramètres définis par l'utilisateur
- `vault.enc` ne contient aucun élément secret en clair (vérifié par inspection du JSON)
- Un mauvais mot de passe maître est rejeté sans révéler d'information sur les données stockées
- Un vault falsifié (modification d'un byte dans `ct_data`) est détecté via le tag GCM
- La suppression d'une entrée ne nécessite pas de rechiffrement des autres
- La clé privée RSA n'est jamais présente sur le disque en clair
- L'interface PySide6 est fonctionnelle sur Linux et Windows
- Le code est structuré en modules avec docstrings et commentaires
- Les exécutables PyInstaller fonctionnent sans installation Python préalable

---

## 7. Évolutions possibles

- Migration vers des paramètres Argon2id adaptatifs selon le matériel détecté au premier lancement
- Remplacement RSA-4096 par X25519 (ECDH) + ChaCha20-Poly1305 pour réduire la latence au login
- Générateur de phrases mémorisables (passphrase) via liste EFF Large Wordlist
- Évaluation de la robustesse des mots de passe existants dans le vault
- Export sécurisé du vault rechiffré avec une clé de backup
- Auto-verrouillage de l'application après N minutes d'inactivité (paramétrable)
- Thème sombre / clair via Qt Style Sheets (QSS)
- Plugin navigateur pour remplissage automatique des formulaires (hors périmètre initial)