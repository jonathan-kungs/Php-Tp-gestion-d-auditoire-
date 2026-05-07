# Php-Tp-gestion-d-auditoire-
Interface permettant de gérer l’utilisation des auditoires au sein de la Fasi 

TRAVAIL PHP DU GROUPE F1

MBUKU IYOLO BENJAMIN & KUNGERWA AKILI JONATHAN
L2 LMD FASI

PARTIE III — Double authentification (2FA + biométrie) ajoutée au projet SGA

## Vue d'ensemble

Le projet est désormais protégé par une **authentification à deux facteurs**
qui propose **deux méthodes au choix** pour la seconde étape :

1. **Code TOTP à 6 chiffres** (RFC 6238) — Google Authenticator,
   Microsoft Authenticator, Authy, FreeOTP, 1Password, Bitwarden…
2. **Biométrie WebAuthn / FIDO2** — empreinte digitale, Face ID,
   Windows Hello, Android Biometric, clés de sécurité physiques (YubiKey…).

Aucune dépendance externe (pas de Composer, pas de base de données, pas
d'extension PHP non standard) — tout est implémenté en **PHP procédural pur**,
en cohérence avec le reste du projet.

## Architecture des fichiers

### Modules

| Fichier                             | Rôle                                                           |
| ----------------------------------- | --------------------------------------------------------------- |
| `includes/fonctions_auth.php`     | Sessions, comptes, mot de passe, TOTP, helpers WebAuthn         |
| `includes/fonctions_webauthn.php` | CBOR, COSE→PEM, vérification ECDSA, parsing authenticatorData |

### Pages utilisateur

| Fichier            | Rôle                                             |
| ------------------ | ------------------------------------------------- |
| `login.php`      | Étape 1 : identifiant + mot de passe             |
| `verify_2fa.php` | Étape 2 : choix code TOTP**OU** biométrie |
| `setup_2fa.php`  | Enrôlement TOTP (QR code à scanner)             |
| `securite.php`   | Gestion des identifiants biométriques            |
| `logout.php`     | Déconnexion / destruction de session             |

### Endpoints AJAX (WebAuthn)

| Fichier                           | Rôle                                                       |
| --------------------------------- | ----------------------------------------------------------- |
| `webauthn_register_options.php` | Renvoie les options pour `navigator.credentials.create()` |
| `webauthn_register_verify.php`  | Vérifie l'attestation et enregistre l'identifiant          |
| `webauthn_login_options.php`    | Renvoie les options pour `navigator.credentials.get()`    |
| `webauthn_login_verify.php`     | Vérifie l'assertion et valide la session                   |
| `webauthn_delete.php`           | Supprime un identifiant biométrique                        |

### Client

| Fichier                | Rôle                                          |
| ---------------------- | ---------------------------------------------- |
| `assets/webauthn.js` | Code client : enrôlement et auth biométrique |

### Données

| Fichier             | Rôle                                                 |
| ------------------- | ----------------------------------------------------- |
| `data/users.json` | Stockage utilisateurs + secrets TOTP + clés WebAuthn |

## Compte par défaut

À la première exécution, un compte administrateur est automatiquement créé :

| Identifiant | Mot de passe  |
| ----------- | ------------- |
| `admin`   | `admin2026` |

Le mot de passe est stocké hashé via `password_hash()` (bcrypt).

## Parcours utilisateur

### Première connexion (enrôlement)

1. L'utilisateur ouvre une URL du site → redirection vers `login.php`.
2. Il saisit `admin` / `admin2026`.
3. Il est envoyé sur `setup_2fa.php` :
   - QR code et clé secrète Base32 affichés ;
   - il scanne avec son application d'authentification ;
   - il saisit le code à 6 chiffres pour confirmer.
4. Il accède à l'application SGA.
5. **Optionnel** : depuis le menu **Sécurité** → bouton
   *« Ajouter un identifiant biométrique »* → suit les invites du navigateur
   (Touch ID / Face ID / Windows Hello / clé FIDO2).

### Connexions suivantes

1. `login.php` : identifiant + mot de passe.
2. `verify_2fa.php` : deux options affichées
   - **Bouton biométrie** (si au moins un identifiant enregistré) →
     authentification biométrique en un geste.
   - **OU** champ pour saisir le code à 6 chiffres TOTP.
3. Accès à l'application.

## Sécurité

### Mots de passe

- Hashage `password_hash` (bcrypt) + `password_verify`.

### TOTP (RFC 6238)

- SHA-1, 6 chiffres, fenêtre de 30 s.
- Tolérance ±1 fenêtre pour la dérive d'horloge.
- Comparaison via `hash_equals()` (résistant au timing).
- Secret 160 bits (`random_bytes(20)`).

### Biométrie (WebAuthn)

- Algorithme ES256 (ECDSA P-256, SHA-256).
- Vérification de la signature via `openssl_verify` natif.
- `userVerification: required` → biométrie obligatoire (pas de simple PIN).
- Challenge aléatoire 256 bits (`random_bytes(32)`) à chaque tentative.
- Vérification stricte de l'origine et du type de cérémonie.
- Comparaisons via `hash_equals()`.
- Décodage CBOR et parsing `authenticatorData` implémentés en PHP pur.

### Anti brute-force

- 5 tentatives maximum sur le code TOTP avant destruction de la session.

## Implémentation technique

### TOTP en PHP pur

Fonctions `base32_encode_bin`, `base32_decode_str`, `generer_secret_totp`,
`calculer_code_totp`, `verifier_code_totp` dans `fonctions_auth.php`.
Le QR code est rendu via le service public `api.qrserver.com`.

### WebAuthn en PHP pur

- **Décodeur CBOR** : sous-ensemble RFC 8949 (types 0–5 et 7) suffisant
  pour parser un `attestationObject` et un `COSE_Key`.
- **COSE → PEM** : reconstruction d'un `SubjectPublicKeyInfo` ASN.1 DER pour
  une clé EC2 P-256, enveloppé en PEM, vérifiable par `openssl_verify`.
- **Parsing `authenticatorData`** : extraction des flags (UP, UV, AT),
  du `signCount`, du `credentialId` et de la clé COSE.
- **Vérification d'assertion** : reconstruction de `signedData = authenticatorData || SHA-256(clientDataJSON)` puis
  `openssl_verify(signedData, signature, pem, OPENSSL_ALGO_SHA256)`.

### Limites volontaires

- Mode `attestation: 'none'` accepté (suffisant pour un second facteur
  d'application interne ; pas besoin d'une chaîne CA d'attestation).
- Algorithme limité à **ES256 (-7)**, qui couvre la quasi-totalité des
  authentificateurs grand public (Touch ID, Face ID, Windows Hello,
  Android Biometric, YubiKey).

## Compatibilité navigateurs

WebAuthn est supporté par :

- Chrome / Edge 67+
- Firefox 60+
- Safari 14+ (macOS / iOS)

Si le navigateur ne supporte pas WebAuthn, le bouton biométrie n'apparaît
simplement pas et l'utilisateur retombe sur le code TOTP — l'application
reste pleinement utilisable.
