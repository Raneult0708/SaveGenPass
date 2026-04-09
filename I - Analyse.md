## ANALYSE — Définition des besoins

### Ce que l'application doit faire (traduit en besoins techniques)

**Besoin 1 — Générer des mots de passe** L'utilisateur contrôle 5 paramètres : longueur totale, nb minuscules, nb majuscules, nb chiffres, nb spéciaux. La génération doit être non-prédictible → `secrets`, pas `random`.

**Besoin 2 — Stocker de façon sécurisée** Chaque mot de passe doit être chiffré avant d'être écrit sur le disque. Le fichier `vault.enc` doit être inutilisable sans le mot de passe maître.

**Besoin 3 — Authentifier l'utilisateur** Un seul facteur : le mot de passe maître. Pas de compte, pas de récupération. La vérification est implicite via le tag AES-GCM (aucun hash stocké).

**Besoin 4 — Interface graphique** Qt Widgets (PySide6) sous Qt Creator. Fenêtres séparées par responsabilité.

---

### Décisions d'architecture prises à l'analyse

Voici les choix qu'il faut valider **maintenant** avant de coder, parce qu'ils sont irréversibles une fois le vault créé :

**Décision 1 — Format de vault.enc** JSON encodé en UTF-8, champs binaires en base64. Raison : lisible, debuggable, versionnez facilement, pas de dépendance à un format binaire propriétaire.

**Décision 2 — Une clé AES par entrée (via RSA)** Chaque entrée a sa propre `clé_entrée` AES-256 chiffrée par RSA. Alternative rejetée : une seule clé AES pour tout le vault. Raison du rejet : si on veut supprimer une entrée proprement ou changer de structure, on est bloqué.

**Décision 3 — Nonce regénéré à chaque écriture** Jamais de nonce réutilisé avec la même clé. À chaque sauvegarde d'une entrée modifiée → nouveau nonce, nouveau chiffrement complet de l'entrée.

**Décision 4 — Chargement lazy de la clé privée RSA** La clé privée RSA est déchiffrée une seule fois au login et gardée en RAM pendant toute la session. Elle n'est pas rechargée à chaque lecture d'entrée. Effacée à la fermeture.