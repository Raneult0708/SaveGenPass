#  Cahier des charges

## Projet : Générateur et gestionnaire local de mots de passe

---

## 1. Présentation du projet

### 1.1 Contexte

Avec la généralisation des services numériques (réseaux sociaux, messageries, plateformes professionnelles, services bancaires, outils éducatifs, etc.), les utilisateurs sont contraints de créer et d’utiliser **un grand nombre de mots de passe différents**.

Bien que les générateurs de mots de passe permettent de produire des mots de passe robustes, un problème majeur subsiste :  
 **les utilisateurs ont des difficultés à mémoriser ces mots de passe complexes**.

Dans la majorité des cas, cela conduit à :

- la réutilisation du même mot de passe,
    
- l’écriture des mots de passe sur des supports non sécurisés,
    
- ou l’abandon des bonnes pratiques de sécurité.
    

Ce projet vise à résoudre **à la fois le problème de génération ET celui de la mémorisation**, sans recourir à une base de données ni à un système de comptes en ligne.

---

### 1.2 Problématique

> Comment permettre à un utilisateur de générer des mots de passe sécurisés **et** de les retrouver facilement,  
> **sans connexion Internet**, **sans base de données**, et **sans stocker de données sensibles en clair** ?

---

### 1.3 Objectifs

#### Objectif principal

Développer une application locale permettant :

- la génération de mots de passe sécurisés et personnalisés,
    
- le stockage sécurisé de ces mots de passe,
    
- leur récupération ultérieure par l’utilisateur.
    

#### Objectifs spécifiques

- Générer des mots de passe aléatoires robustes
    
- Permettre une personnalisation fine des caractéristiques du mot de passe
    
- Mettre en place un **système de stockage chiffré local**
    
- Garantir que **les mots de passe ne sont jamais stockés en clair**
    
- Proposer une interface graphique simple et intuitive
    
- Sensibiliser l’utilisateur aux bonnes pratiques de cybersécurité
    

---

### 1.4 Périmètre du projet

#### Inclus

- Génération de mots de passe personnalisés
    
- Interface graphique (GUI) avec **Pygame**
    
- Création d’un **mot de passe maître**
    
- Stockage local des mots de passe dans un fichier **chiffré**
    
- Consultation des mots de passe après authentification
    
- Fonctionnement 100 % hors ligne
    

#### Exclus

- Connexion Internet
    
- Base de données
    
- Gestion de comptes multi-utilisateurs
    
- Synchronisation cloud
    
- Récupération du mot de passe maître en cas d’oubli
    

---

## 2. Spécifications fonctionnelles

### 2.1 Fonctionnalités principales

L’application devra permettre à l’utilisateur de :

#### Génération de mot de passe

- Choisir la **longueur totale** du mot de passe
    
- Définir le nombre de :
    
    - caractères numériques
        
    - lettres alphabétiques minuscules
        
    - lettres alphabétiques majuscules
        
    - caractères spéciaux
        
- Générer un mot de passe aléatoire respectant ces contraintes
    

#### Gestion sécurisée

- Définir un **mot de passe maître** lors de la première utilisation
    
- Associer chaque mot de passe généré à :
    
    - un nom de service (ex : Gmail, Facebook, GitHub)
        
- Enregistrer les mots de passe dans un **fichier local chiffré**
    
- Déverrouiller le fichier uniquement après saisie correcte du mot de passe maître
    
- Afficher les mots de passe stockés après authentification
    

---

### 2.2 Cas d’utilisation

#### Cas 1 : Première utilisation

1. L’utilisateur lance l’application
    
2. L’application demande la création d’un mot de passe maître
    
3. Le mot de passe maître est utilisé pour créer la clé de chiffrement
    
4. Un fichier sécurisé est initialisé
    

#### Cas 2 : Génération et sauvegarde

1. L’utilisateur saisit les paramètres du mot de passe
    
2. Le programme vérifie la cohérence des paramètres
    
3. Le mot de passe est généré
    
4. L’utilisateur choisit de l’enregistrer
    
5. Le mot de passe est chiffré et stocké
    

#### Cas 3 : Consultation

1. L’utilisateur saisit le mot de passe maître
    
2. Le fichier est déchiffré en mémoire
    
3. Les mots de passe sont affichés à l’écran
    

---

## 3. Contraintes techniques

### 3.1 Technologies utilisées

- Langage : **Python**
    
- Interface graphique : **Pygame**
    
- Bibliothèques :
    
    - `random`
        
    - `string`
        
    - bibliothèques de chiffrement (clé dérivée + chiffrement symétrique)
        

---

### 3.2 Contraintes

- Application locale uniquement
    
- Compatible Python 3.x
    
- Aucun accès réseau
    
- Fonctionnement sur Linux / Windows
    
- Code clair, structuré et commenté
    

---

### 3.3 Sécurité des données (POINT CLÉ)

- Aucun mot de passe n’est stocké en clair
    
- Les mots de passe sont stockés dans un **fichier chiffré**
    
- La clé de chiffrement est dérivée du **mot de passe maître**
    
- Le mot de passe maître n’est jamais stocké
    
- Sans le mot de passe maître, le fichier est inutilisable
    

> Cette approche s’inspire des principes fondamentaux des gestionnaires de mots de passe sécurisés.

---

## 4. Design et ergonomie

### 4.1 Interface graphique (GUI)

- Interface développée avec **Pygame**
    
- Fenêtres claires et organisées
    
- Champs de saisie pour :
    
    - paramètres du mot de passe
        
    - mot de passe maître
        
- Boutons :
    
    - Générer
        
    - Sauvegarder
        
    - Afficher
        
    - Supprimer
        

---

### 4.2 Expérience utilisateur

- Navigation simple
    
- Messages explicites
    
- Alertes en cas d’erreur
    
- Feedback visuel immédiat
    

---

## 5. Livrables et jalons

### 5.1 Livrables

- Code source Python
    
- Interface graphique fonctionnelle
    
- Fichier de stockage chiffré
    
- Cahier des charges
    
- README général
    
- Commentaires et documentation interne
    
- Exécutable PE et ELF
---

### 5.2 Planning prévisionnel

| Étape         | Description              | Durée    |
| ------------- | ------------------------ | -------- |
| Analyse       | Définition des besoins   | 1/2 jour |
| Conception    | Architecture + sécurité  | 1/2 jour |
| GUI           | Interface Pygame         | 1 jours  |
| Développement | Génération + chiffrement | 1 jours  |
| Tests         | Sécurité et ergonomie    | 1 jour   |
| Documentation | README + rapport         | 1/2 jour |

---

## 6. Critères de validation

Le projet sera validé si :

- Les mots de passe sont correctement générés
    
- Le stockage est chiffré et sécurisé
    
- Le mot de passe maître est requis pour l’accès
    
- L’interface graphique est fonctionnelle
    
- Le code est propre et documenté
    

---

## 7. Évolutions possibles

- Générateur de phrases mémorisables
    
- Évaluation de la robustesse des mots de passe
    
- Export sécurisé
    
- Auto-verrouillage de l’application