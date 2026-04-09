##  Structure des fichiers du projet

```bash
SaveGenPass/
│
├── main.py
│
├── core/
│   ├── __init__.py
│   ├── crypto.py        # Fonctions cryptographiques (RSA, chiffrement, etc.)
│   ├── vault.py         # Gestion du coffre-fort chiffré
│   └── generator.py     # Générateur de mots de passe sécurisés
│
├── ui/
│   ├── __init__.py
│   ├── app.py               # Point d'entrée Qt + gestion de session
│   ├── phase_a.py           # Phase initiale (Setup / Login / Attente)
│   │   ├── setup_widget.py  # Page 0 : configuration initiale
│   │   ├── login_widget.py  # Page 1 : connexion utilisateur
│   │   └── wait_widget.py   # Page 2 : génération RSA (chargement)
│   │
│   └── phase_b.py           # Interface principale (après login)
│       ├── generator_widget.py  # Génération de mots de passe
│       └── vault_widget.py      # Gestion du coffre-fort
│
├── vault.enc                # Fichier chiffré (stocké dans ~/.savegenpass/)
└── requirements.txt         # Dépendances du projet
```

# `Responsabilités de chaque module`
## `core/crypto.py`

Aucune connaissance de Qt. Aucune connaissance de vault.enc. Fonctions pures uniquement.

```python
# Ce que crypto.py doit exposer :

derive_key(password: str, salt: bytes) -> bytes
# Argon2id → KEK 256 bits

generate_rsa_keypair() -> (public_key, private_key)
# RSA-4096

encrypt_private_key(private_key, kek: bytes) -> (nonce, ct, tag)
# AES-256-GCM(KEK, clé_privée)

decrypt_private_key(nonce, ct, tag, kek: bytes) -> private_key
# lève InvalidTag si mauvais mot de passe → c'est LA vérification

encrypt_entry(password: str, public_key) -> (ct_key, nonce, ct_data, tag)
# génère clé_entrée → AES-GCM(clé_entrée, password) + RSA-OAEP(pub, clé_entrée)

decrypt_entry(ct_key, nonce, ct_data, tag, private_key) -> str
# RSA-OAEP(priv, ct_key) → clé_entrée → AES-GCM(clé_entrée, ct_data)
```


```python
def derive_key(
    password: str,
    salt: bytes          # 16 bytes, généré par os.urandom(16)
) -> bytes:              # retourne 32 bytes (KEK 256 bits)
```

> Argon2id. Paramètres fixes en interne : m=65536, t=3, p=4. La fonction ne choisit pas le salt, elle le reçoit.


```python
def generate_rsa_keypair() -> tuple:
    # retourne (public_key_object, private_key_object)
    # types : cryptography.hazmat.primitives.asymmetric.rsa.RSAPublicKey
    #                                                        RSAPrivateKey
```

> RSA-4096. Lent (~5s). Appelée une seule fois au setup. L'appelant gère la barre de progression (dans un QThread).


```python
def serialize_public_key(
    public_key           # RSAPublicKey object
) -> bytes:              # retourne PEM encodé en bytes
```


```python
def deserialize_public_key(
    pem: bytes           # PEM bytes lu depuis vault.enc
):                       # retourne RSAPublicKey object
```


```python
def encrypt_private_key(
    private_key,         # RSAPrivateKey object
    kek: bytes           # 32 bytes, la KEK dérivée par derive_key()
) -> tuple:              # retourne (nonce: bytes, ct: bytes, tag: bytes)
                         # nonce = 12 bytes, tag = 16 bytes (derniers bytes du ct en pratique)
```

> AES-256-GCM. La fonction génère le nonce en interne via os.urandom(12).

```python
def decrypt_private_key(
    nonce: bytes,        # 12 bytes
    ct: bytes,           # ciphertext
    tag: bytes,          # 16 bytes
    kek: bytes           # 32 bytes
):                       # retourne RSAPrivateKey object
                         # lève cryptography.exceptions.InvalidTag si KEK incorrecte
                         # → c'est cette exception qui signifie "mauvais mot de passe"
```

```python
def encrypt_entry(
    password: str,       # mot de passe en clair à stocker
    public_key           # RSAPublicKey object (en RAM depuis la session)
) -> tuple:              # retourne (ct_key: bytes, nonce: bytes, ct_data: bytes, tag: bytes)
                         # ct_key  = clé AES chiffrée par RSA-OAEP
                         # nonce   = 12 bytes aléatoires
                         # ct_data = mot de passe chiffré AES-GCM
                         # tag     = 16 bytes
```

> Génère clé_entrée (32 bytes) en interne. Ne la retourne jamais.


```python
def decrypt_entry(
    ct_key: bytes,       # clé AES chiffrée par RSA-OAEP
    nonce: bytes,        # 12 bytes
    ct_data: bytes,      # mot de passe chiffré
    tag: bytes,          # 16 bytes
    private_key          # RSAPrivateKey object (depuis Session)
) -> str:                # retourne le mot de passe en clair
```

---

## `core/vault.py`

Gère uniquement la persistance. Ne connaît pas Qt. Ne fait aucune crypto.


```python
# Ce que vault.py doit exposer :

vault_exists() -> bool
# vérifie si vault.enc est présent

init_vault(nonce_pk, ct_pk, tag_pk, salt_kdf, public_key_pem, kdf_params) -> None
# crée vault.enc avec le vault vide

load_vault() -> dict
# lit et parse vault.enc

save_vault(data: dict) -> None
# écrit vault.enc

add_entry(service: str, ct_key, nonce, ct_data, tag) -> None
# ajoute une entrée dans vault.enc

delete_entry(service: str) -> None
# supprime une entrée

list_services() -> list[str]
# retourne les noms des services stockés

get_entry(service: str) -> dict
# retourne {ct_key, nonce, ct_data, tag} d'une entrée
```

```python
def get_vault_path() -> Path:
    # retourne ~/.savegenpass/vault.enc sur Linux
    #           %APPDATA%/SaveGenPass/vault.enc sur Windows
    # crée le dossier parent si nécessaire
```


```python
def vault_exists() -> bool:
    # retourne True si vault.enc existe au chemin retourné par get_vault_path()
```


```python
def init_vault(
    salt_kdf: bytes,          # 16 bytes
    kdf_params: dict,         # {"m": 65536, "t": 3, "p": 4}
    public_key_pem: bytes,    # clé publique RSA sérialisée
    nonce_pk: bytes,          # 12 bytes
    ct_pk: bytes,             # clé privée chiffrée
    tag_pk: bytes             # 16 bytes
) -> None:
    # crée vault.enc avec les métadonnées et entries vide {}
    # écrit le JSON sur disque
```


```python
def load_vault() -> dict:
    # lit vault.enc, parse le JSON
    # retourne le dict complet (toutes les clés en base64 décodées en bytes)
    # lève FileNotFoundError si vault.enc absent
```


```python
def add_entry(
    service: str,        # nom du service, ex: "gmail"
    ct_key: bytes,
    nonce: bytes,
    ct_data: bytes,
    tag: bytes
) -> None:
    # charge vault.enc, ajoute l'entrée, réécrit vault.enc
    # lève ValueError si service existe déjà
```


```python
def update_entry(
    service: str,
    ct_key: bytes,
    nonce: bytes,
    ct_data: bytes,
    tag: bytes
) -> None:
    # même structure que add_entry mais écrase l'entrée existante
```


```python
def delete_entry(
    service: str         # nom du service à supprimer
) -> None:
    # charge vault.enc, supprime l'entrée, réécrit vault.enc
    # lève KeyError si service introuvable
```


```python
def get_entry(
    service: str
) -> dict:               # retourne {"ct_key": bytes, "nonce": bytes,
                         #           "ct_data": bytes, "tag": bytes}
                         # lève KeyError si service introuvable
```


```python
def list_services() -> list[str]:
    # retourne la liste des noms de services stockés
    # retourne [] si vault vide
```

---

## `core/generator.py`

Pur Python. Aucune dépendance externe sauf `secrets` et `string`.


```python
# Ce que generator.py doit exposer :

generate_password(length, min_lower, min_upper, min_digits, min_special) -> str
# via secrets.choice(), garantit les minimums

validate_params(length, min_lower, min_upper, min_digits, min_special) -> (bool, str)
# retourne (True, "") ou (False, "message d'erreur")

calculate_entropy(password: str) -> float
# entropie en bits = log2(taille_alphabet ^ longueur)
```


```python
def validate_params(
    length: int,
    min_lower: int,
    min_upper: int,
    min_digits: int,
    min_special: int
) -> tuple[bool, str]:   # (True, "") si valide
                         # (False, "message explicite") si invalide
                         # vérifie : min_lower+min_upper+min_digits+min_special <= length
                         #           length >= 8
                         #           tous les paramètres >= 0
```

```python
def generate_password(
    length: int,
    min_lower: int,
    min_upper: int,
    min_digits: int,
    min_special: int
) -> str:                # retourne le mot de passe généré
                         # garantit les minimums de chaque catégorie
                         # complète avec des caractères aléatoires jusqu'à length
                         # mélange le résultat (évite que les minimums soient toujours en début)
```

```python
def calculate_entropy(
    password: str
) -> float:              # entropie en bits
                         # détecte les catégories présentes → taille de l'alphabet
                         # formule : log2(taille_alphabet) * len(password)
```

---

## `ui/` — Architecture Qt

### `ui/app.py` — Session + bootstrap ???


```python
class Session:
    private_key = None       # RSAPrivateKey | None
    public_key = None        # RSAPublicKey  | None
    is_authenticated: bool = False

    @classmethod
    def clear(cls) -> None:
        # remet tout à None / False
        # appelé à la déconnexion ou fermeture
```



```python
def run_app() -> None:
    # crée QApplication
    # vérifie vault_exists()
    #   → False : lance PhaseA sur la page Setup (index 0)
    #   → True  : lance PhaseA sur la page Login (index 1)
    # appelé par main.py
```


### `ui/phase_a.py` — Fenêtre d'entrée


```python
class PhaseA(QWidget):
    # contient un QStackedWidget avec 3 pages :
    #   index 0 → SetupWidget   (première utilisation)
    #   index 1 → LoginWidget   (utilisateur existant)
    #   index 2 → WaitWidget    (génération RSA en cours)

    def show_setup(self) -> None    # setCurrentIndex(0)
    def show_login(self) -> None    # setCurrentIndex(1)
    def show_wait(self) -> None     # setCurrentIndex(2)

    def on_setup_complete(self) -> None
        # appelé par SetupWidget quand le vault est initialisé
        # → self.close()
        # → lance PhaseB

    def on_login_success(self) -> None
        # appelé par LoginWidget quand Session.private_key est chargée
        # → self.close()
        # → lance PhaseB
```


### Pages du QStackedWidget


```python
class SetupWidget(QWidget):
    # Signaux émis :
    setup_complete = Signal()     # émis quand vault initialisé avec succès
    show_wait = Signal()          # émis quand la génération RSA commence

    # Ce qu'elle fait (sans implémentation) :
    # - 2 QLineEdit (mot de passe maître + confirmation, masqués)
    # - 1 QPushButton "Créer le vault"
    # - appelle validate_params, derive_key, generate_rsa_keypair, init_vault
    # - génère RSA dans un QThread → émet show_wait pendant, setup_complete après
```


```python
class LoginWidget(QWidget):
    # Signaux émis :
    login_success = Signal()      # émis quand decrypt_private_key réussit

    # - 1 QLineEdit (mot de passe maître, masqué)
    # - 1 QPushButton "Déverrouiller"
    # - appelle load_vault, derive_key, decrypt_private_key
    # - si InvalidTag → affiche QMessageBox "Mot de passe incorrect"
    # - si succès → remplit Session.private_key + Session.public_key → émet login_success
```



```python
class WaitWidget(QWidget):
    # - QLabel "Génération des clés RSA en cours…"
    # - QProgressBar en mode indéterminé (setRange(0, 0))
    # - aucun bouton, aucune interaction
    # - affichée pendant le QThread de generate_rsa_keypair()
```


### `ui/phase_b.py` — Fenêtre principale


```python
class PhaseB(QMainWindow):
    # Contient 2 zones accessibles (via QTabWidget ou boutons de navigation) :
    #   - GeneratorWidget
    #   - VaultWidget

    def logout(self) -> None:
        # Session.clear()
        # self.close()
        # relance PhaseA sur LoginWidget
```



```python
class GeneratorWidget(QWidget):
    # - QSpinBox × 5 (length, min_lower, min_upper, min_digits, min_special)
    # - QPushButton "Générer"
    # - QLineEdit (affichage du mot de passe généré, lecture seule)
    # - QLabel (entropie affichée en bits)
    # - QProgressBar (force visuelle : rouge/orange/vert selon entropie)
    # - QPushButton "Copier" → presse-papier + timer 30s
    # - QPushButton "Sauvegarder" → ouvre une QDialog pour saisir le nom du service
    #   → appelle encrypt_entry + add_entry
```



```python
class VaultWidget(QWidget):
    # - QTableWidget (colonnes : Service | Mot de passe | Actions)
    # - bouton "Afficher" par ligne → appelle decrypt_entry → remplace "••••" par le clair
    # - bouton "Copier" par ligne → presse-papier + timer 30s
    # - bouton "Supprimer" par ligne → QMessageBox confirmation → delete_entry
    # - chargée avec list_services() à l'ouverture de PhaseB
```

---

## Flux de démarrage complet 

```
main.py
  └── run_app()
        ├── vault_exists() == False
        │     └── PhaseA.show_setup()
        │           └── [utilisateur crée vault]
        │                 └── QThread: generate_rsa_keypair()
        │                       └── PhaseA.show_wait() pendant
        │                       └── init_vault() après
        │                       └── on_setup_complete() → PhaseB.show()
        │
        └── vault_exists() == True
              └── PhaseA.show_login()
                    └── [utilisateur entre mot de passe]
                          └── derive_key() + decrypt_private_key()
                                ├── InvalidTag → message erreur
                                └── Succès → Session remplie
                                      └── on_login_success() → PhaseB.show()
```

---
### Schéma des flux de données 

```
LOGIN
─────
Utilisateur tape mot de passe maître
    │
    ▼
vault.py.load_vault() → récupère salt_kdf, nonce_pk, ct_pk, tag_pk
    │
    ▼
crypto.derive_key(mdp, salt_kdf) → KEK
    │
    ▼
crypto.decrypt_private_key(nonce_pk, ct_pk, tag_pk, KEK)
    ├── InvalidTag → login_window affiche "Mot de passe incorrect"
    └── Succès → private_key stockée dans app.py (session)


AJOUT D'ENTRÉE
──────────────
generator.generate_password(...) → password_en_clair
    │
    ▼
crypto.encrypt_entry(password_en_clair, public_key)
→ (ct_key, nonce, ct_data, tag)
    │
    ▼
vault.add_entry(service, ct_key, nonce, ct_data, tag)


LECTURE D'ENTRÉE
─────────────────
vault.get_entry(service) → {ct_key, nonce, ct_data, tag}
    │
    ▼
crypto.decrypt_entry(ct_key, nonce, ct_data, tag, private_key)
→ password_en_clair (affiché, puis effacé)
```