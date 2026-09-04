# Guide développeur — `event_app`

Application mobile Flutter **white-label** de gestion de congrès médicaux : programme,
speakers, abstracts, comités, sponsors, VOD, FAQ et notifications locales.

Une seule base de code sert plusieurs congrès. Tout ce qui change d'un événement à
l'autre (nom, package, couleur, API, slug) vit dans un **fichier de config JSON**
injecté au moment du build. Aucune variante de code à maintenir.

> **Ce guide part de zéro.** Si vous n'avez jamais fait de Flutter, suivez les sections
> dans l'ordre : §1 (installer) → §2 (premier lancement) → §3 (tester) → §4 (produire un APK).

> **Sommaire**
> 1. [Environnement de travail](#1-environnement-de-travail)
> 2. [Premier lancement, pas à pas](#2-premier-lancement-pas-à-pas)
> 3. [Lancer et tester l'application](#3-lancer-et-tester-lapplication)
> 4. [Générer un APK en local](#4-générer-un-apk-en-local)
> 5. [White-label : pointer vers un nouveau congrès](#5-white-label--pointer-vers-un-nouveau-congrès)
> 6. [Builds Codemagic (CI)](#6-builds-codemagic-ci)
> 7. [Architecture du code](#7-architecture-du-code)
> 8. [Dépannage](#8-dépannage)
> 9. [Aide-mémoire](#9-aide-mémoire)

---

## 1. Environnement de travail

### 1.1 Vocabulaire de base

Si ces mots sont nouveaux, voici l'essentiel — le reste du guide s'appuie dessus.

| Terme | Ce que c'est |
|---|---|
| **Flutter** | Le framework. Un seul code source → Android, iOS et web. |
| **Dart** | Le langage de programmation utilisé par Flutter. |
| **SDK** | L'ensemble des outils d'un environnement (Flutter SDK, Android SDK…). |
| **`pubspec.yaml`** | La liste des dépendances du projet (équivalent `package.json`). |
| **`flutter pub get`** | Télécharge les dépendances déclarées dans `pubspec.yaml`. |
| **APK** | Le fichier installable d'une app Android (installation directe / test). |
| **AAB** | Le format exigé par le Google Play Store (*Android App Bundle*). |
| **IPA** | L'équivalent Apple, pour l'App Store / TestFlight. |
| **Émulateur** | Un téléphone Android virtuel qui tourne sur le PC. |
| **Hot reload** | Injecte le code modifié dans l'app **déjà lancée**, en moins d'une seconde. |
| **`dart-define`** | Une variable injectée **au build**. C'est le cœur du white-label ici. |
| **Keystore** (`.jks`) | Le certificat qui signe l'app. Indispensable pour publier. |
| **Version name** | Le numéro vu par l'utilisateur (`1.0.1`). Voir §5.6. |
| **Build number** | Le compteur interne aux stores (`+6`). Ne doit **jamais** reculer. Voir §5.6. |
| **White-label** | Un même code décliné sous plusieurs identités (nom, logo, couleurs). |

### 1.2 Versions de référence

Combinaison validée en build réel le **2026-09-04** (Windows 11) :

| Outil | Version | Note |
|---|---|---|
| Flutter SDK | **3.47.2** (stable) | contrainte pubspec : `sdk: ^3.10.7` |
| Dart | 3.13.2 | fourni avec Flutter, rien à installer |
| JDK | **17 minimum** — testé avec Temurin 21 | `build.gradle` cible `VERSION_17` |
| Android SDK | API 36 (`android-36.1`) | |
| Build-tools | 36.0.0 | |
| Gradle | 8.14.0 | ⚠️ voir [§1.7](#17-migration-android-à-prévoir) |
| AGP | 8.11.1 | ⚠️ idem |
| Kotlin | 2.2.20 | ⚠️ idem |
| CMake | 3.22.1 | installé automatiquement au 1ᵉʳ build |

### 1.3 À installer

**1. Flutter SDK**
Télécharger le canal `stable` : <https://docs.flutter.dev/get-started/install/windows>
Décompresser dans un chemin **sans espace ni accent** (ex. `D:\dev\flutter`), puis ajouter
`D:\dev\flutter\bin` au `PATH` Windows.

> *Ajouter au PATH* : touche Windows → « variables d'environnement » → *Variables
> d'environnement* → sélectionner `Path` → *Modifier* → *Nouveau* → coller le chemin.
> **Fermer et réouvrir le terminal** pour que ce soit pris en compte.

**2. Android Studio** (fournit le SDK, l'émulateur et le plugin Flutter)
<https://developer.android.com/studio>
Puis *Settings → Languages & Frameworks → Android SDK*, onglet *SDK Tools*, cocher :
- `Android SDK Platform 36`
- `Android SDK Build-Tools 36.0.0`
- `Android SDK Command-line Tools (latest)` ← **indispensable** (licences, cf. §2)
- `Android SDK Platform-Tools`
- `NDK (Side by side)` — requis par certains plugins natifs
- `Android Emulator`

**3. JDK 17 ou plus**
Android Studio en embarque un (JBR) — vérifiez avec `flutter doctor -v` quel JDK est
utilisé. Sinon Temurin : <https://adoptium.net>. Pour forcer celui de Flutter :
```powershell
flutter config --jdk-dir="C:\Program Files\Eclipse Adoptium\jdk-21..."
```

**4. Git** — <https://git-scm.com/download/win>

**5. Éditeur** — VS Code + extensions **Flutter** et **Dart** (le plus léger), ou
Android Studio.

**6. Google Chrome** — permet de tester l'app dans le navigateur (cf. §3.2).

### 1.4 Mode développeur Windows ⚠️ obligatoire

Flutter a besoin des **liens symboliques** pour construire les plugins. Sans ça, le build
échoue avec `Building with plugins requires symlink support`.

```powershell
start ms-settings:developers    # puis activer « Mode développeur »
```
ou, dans un terminal **administrateur** :
```powershell
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock" /v AllowDevelopmentWithoutDevLicense /t REG_DWORD /d 1 /f
```

### 1.5 Créer un émulateur Android

Dans Android Studio : *More Actions → Virtual Device Manager → Create Device*.
Choisir **Pixel 8**, une image système **API 35** ou plus, puis *Finish*.

En ligne de commande :
```powershell
flutter emulators                       # lister ceux qui existent
flutter emulators --launch Pixel_8      # en démarrer un
```

### 1.6 Commandes Git de base

```powershell
git clone <URL_DU_REPO>      # récupérer le projet
git status                   # voir ses modifications
git pull                     # récupérer les changements des autres
git checkout -b ma-branche   # créer une branche de travail
git add .                    # préparer ses modifications
git commit -m "message"      # enregistrer
git push -u origin ma-branche# envoyer sur GitHub
git checkout -- lib          # ANNULER ses modifications dans lib/ (irréversible)
```

### 1.7 Migration Android à prévoir

Flutter 3.47 avertit à chaque build que le support des versions actuelles sera bientôt
retiré. **Non bloquant aujourd'hui**, mais à planifier :

| | Actuel | Requis à terme |
|---|---|---|
| Gradle | 8.14.0 | ≥ 9.1.0 |
| AGP | 8.11.1 | ≥ 9.0.1 |
| Kotlin | 2.2.20 | ≥ 2.3.20 |

Deux flags de compatibilité ont été ajoutés par le migrator Flutter dans
`android/gradle.properties` — **ne pas les supprimer** avant d'avoir fait la migration :
```properties
android.builtInKotlin=false
android.newDsl=false
```
Pour masquer temporairement les avertissements : `--android-skip-build-dependency-validation`.

---

## 2. Premier lancement, pas à pas

```powershell
# 1. Récupérer le projet
git clone <URL_DU_REPO>
cd event_app

# 2. Installer les dépendances
flutter pub get

# 3. Vérifier l'environnement (tout doit être vert)
flutter doctor

# 4. Accepter les licences Android (nécessite Command-line Tools, cf. §1.3)
flutter doctor --android-licenses

# 5. Vérifier qu'un appareil est détecté
flutter devices

# 6. Lancer l'app
flutter run -d chrome --dart-define-from-file=configs/smr26.json
```

> ⚠️ **Le `--dart-define-from-file` n'est pas optionnel.** Sans lui, l'app utilise les
> valeurs par défaut codées dans `lib/config/app_config.dart` — vous ne saurez pas
> quelle configuration vous testez réellement. Prenez l'habitude de toujours l'ajouter.

Le **premier build est lent** (5 à 10 minutes) : Gradle télécharge ses dépendances et
installe CMake. Les suivants prennent moins d'une minute.

---

## 3. Lancer et tester l'application

L'app tourne sur **trois cibles**. Toutes les trois ont été vérifiées sur ce projet.

| Cible | Vitesse | Ce qu'on peut tester | Limite |
|---|---|---|---|
| **Chrome (web)** | ⚡ la plus rapide | Interface, navigation, données API | Pas de notifications ni de vraies WebView |
| **Émulateur Android** | 🐢 moyenne | Quasiment tout, notifications incluses | Pas d'appareil photo réel, GPS simulé |
| **Téléphone réel** | 🐢 moyenne | Tout, performances réelles | Nécessite un câble et le mode développeur |

### 3.1 Choisir sa cible

```powershell
flutter devices
```
Exemple de sortie :
```
sdk gphone64 x86 64 (mobile) • emulator-5554 • android-x64    • Android 15 (API 35)
Windows (desktop)            • windows       • windows-x64
Chrome (web)                 • chrome        • web-javascript
Edge (web)                   • edge          • web-javascript
```
La 2ᵉ colonne est **l'identifiant** à passer à `-d`.

### 3.2 Tester dans Chrome (le plus rapide) ✅

Idéal pour itérer sur l'interface : démarrage en quelques secondes.

```powershell
flutter run -d chrome --dart-define-from-file=configs/smr26.json
```

Chrome s'ouvre tout seul sur l'app. Utile aussi :
```powershell
flutter run -d chrome --web-port=8080 --dart-define-from-file=configs/smr26.json
flutter run -d edge --dart-define-from-file=configs/smr26.json
flutter build web --dart-define-from-file=configs/smr26.json   # build statique
```

**Ce qui ne marche pas sur web**, et c'est normal — le code le désactive volontairement
avec des garde-fous `kIsWeb` :

| Fonction | Fichier | Comportement sur web |
|---|---|---|
| Notifications locales | `services/notification_service.dart:19` | désactivées |
| Infos appareil | `services/device_service.dart:10` | valeurs de repli |
| Détection réseau | `utils/connectivity_service.dart:17` | toujours « connecté » |
| Lien vers le store | `main.dart:200` | bouton « OK » au lieu de « Mettre à jour » |

> Ne signalez donc pas comme bug l'absence de notifications dans Chrome.

### 3.3 Tester sur l'émulateur Android ✅

```powershell
flutter emulators --launch Pixel_8       # 1. démarrer l'émulateur, attendre le bureau
flutter devices                          # 2. relever son id (ex. emulator-5554)
flutter run -d emulator-5554 --dart-define-from-file=configs/smr26.json
```

C'est la cible à utiliser pour tout ce qui touche aux **notifications**, au **hors ligne**
et aux **WebView** (VOD YouTube).

### 3.4 Tester sur un téléphone réel

1. Sur le téléphone : *Paramètres → À propos* → taper 7× sur « Numéro de build » pour
   activer les options développeur.
2. *Options pour les développeurs* → activer **Débogage USB**.
3. Brancher en USB et accepter la demande d'autorisation qui s'affiche.
4. Vérifier puis lancer :
```powershell
flutter devices
flutter run -d <id_du_telephone> --dart-define-from-file=configs/smr26.json
```

### 3.5 Les raccourcis pendant `flutter run`

Le terminal reste actif : tapez une lettre puis Entrée.

| Touche | Effet |
|---|---|
| `r` | **Hot reload** — applique le code modifié en gardant l'état de l'app |
| `R` | **Hot restart** — redémarre l'app à zéro (nécessaire si vous changez `main.dart` ou un provider) |
| `q` | Quitter |
| `p` | Affiche la grille de debug des layouts |
| `o` | Bascule Android / iOS pour le rendu |
| `v` | Ouvre **DevTools** dans le navigateur |

> Un changement dans un **`--dart-define`** ou dans `pubspec.yaml` n'est **pas** pris en
> compte par un hot reload : il faut arrêter (`q`) et relancer.

### 3.6 Voir les logs et déboguer

```powershell
flutter logs                        # logs de l'appareil connecté
adb logcat *:E                      # erreurs Android brutes
adb devices                         # appareils vus par Android
```

**DevTools** (inspecteur de widgets, réseau, mémoire) : touche `v` pendant
`flutter run`, ou `dart devtools`.

Dans le code, `debugPrint('...')` est préférable à `print('...')` (non tronqué,
retiré en release).

### 3.7 Tests automatisés et qualité du code

```powershell
flutter analyze          # analyse statique — le premier réflexe avant de commiter
flutter test             # tests unitaires (dossier test/)
dart format lib/         # reformate le code selon le style Dart officiel
```

État actuel attendu de `flutter analyze` : **2 avertissements connus** dans
`lib/services/update_service.dart` (lignes 43-44, `unawaited_return_in_try_block`).
Toute autre alerte vient de vos modifications.

### 3.8 Repartir propre

Si un build devient incohérent (erreurs inexplicables, plugin fantôme) :

```powershell
flutter clean            # supprime build/ et .dart_tool/
flutter pub get          # réinstalle les dépendances
```

---

## 4. Générer un APK en local

### 4.1 APK de test, tout de suite

Sans aucune configuration de signature, ceci fonctionne déjà — Gradle utilise
automatiquement la **signature debug** :

```powershell
flutter build apk --release --dart-define-from-file=configs/smr26.json
```

C'est parfait pour faire tester l'app à quelqu'un. En revanche un APK signé en debug
est **impubliable sur le Play Store**.

### 4.2 Signature release (pour publier)

1. Générer un keystore — **une seule fois, et à conserver précieusement** : le perdre
   interdit définitivement toute mise à jour de l'app sur le Play Store.
   ```powershell
   keytool -genkey -v -keystore smr26.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
   ```
2. Copier `android/key.properties.template` → `android/key.properties` et le remplir :
   ```properties
   storePassword=...
   keyPassword=...
   keyAlias=upload
   storeFile=../keystore/smr26.jks
   ```

`build.gradle` détecte la présence de `key.properties` et signe en release
automatiquement. S'il est absent, il retombe sur la signature debug.

> 🔒 `key.properties`, `*.jks`, `kys/` et `envs/` sont déjà dans `.gitignore`.
> **Ne jamais les commiter, ne jamais les envoyer par email.**

### 4.3 Méthode recommandée : le script

Il gère l'icône **et** le build en une seule commande :

```powershell
scripts\build_android.bat smr26          # APK release
scripts\build_android.bat smr26 aab      # AAB (Play Store)
scripts\build_android.bat smr26 all      # les deux
```

Ce que fait le script :
1. si `assets/icons/smr26.png` existe → le copie en `assets/icon/app_icon.png` puis
   régénère toutes les icônes (`dart run flutter_launcher_icons`) ;
2. lance le build avec `--dart-define-from-file=configs/smr26.json` ;
3. renomme le binaire produit.

Sur macOS / Linux, l'équivalent est `./scripts/build.sh smr26 [apk|aab|ios|all]`
(il lit `envs/client_*.env` au lieu de `configs/*.json`).

### 4.4 Où sont les fichiers produits

`build.gradle` renomme l'APK d'après le **dernier segment** de `APP_PACKAGE`
(ex. `com.hashtagsante.smr2026` → `smr2026-release.apk`) :

| Cible | Chemin |
|---|---|
| APK | `build/app/outputs/flutter-apk/<client>-release.apk` |
| AAB | `build/app/outputs/bundle/release/app-release.aab` |

Variante « second store » : `.\build_store2.ps1` (config `smr26_store2.json`, package
`com.SMR2026`, sortie rangée dans `releases/store2/`).

### 4.5 Installer l'APK sur un téléphone

```powershell
adb install -r build\app\outputs\flutter-apk\smr2026-release.apk
```
Ou copier le fichier sur le téléphone et l'ouvrir (autoriser les « sources inconnues »).

---

## 5. White-label : pointer vers un nouveau congrès

### 5.1 Le fichier de config

Créer `configs/<nouveau>.json` sur le modèle de `configs/smr26.json` :

```json
{
  "APP_NAME": "SMR 26",
  "APP_PACKAGE": "com.hashtagsante.smr2026",
  "EVENT_SLUG": "smr26_9v9z4",
  "EVENT_KEY": "ev_jkfIj6Fu362m0iOK",
  "BASE_URL": "https://mob-app-ascrea.hashtagsante.com/api/v1",
  "PRIMARY_COLOR": "6A1B62",
  "STORE_URL_ANDROID": "https://play.google.com/store/apps/details?id=com.hashtagsante.smr2026",
  "STORE_URL_IOS": "https://apps.apple.com/app/id6763072246"
}
```

| Clé | Rôle | Consommée par |
|---|---|---|
| `APP_NAME` | Nom affiché sous l'icône | `build.gradle` → `android:label` |
| `APP_PACKAGE` | `applicationId` Android + nom du fichier APK | `build.gradle` |
| `EVENT_SLUG` | Identifie l'événement côté API | `AppConfig.eventSlug` |
| `EVENT_KEY` | Clé d'accès à l'API de l'événement | `AppConfig.eventAccessKey` |
| `BASE_URL` | Racine de l'API backend | `AppConfig.baseUrl` |
| `PRIMARY_COLOR` | Couleur principale, hex **sans `#`** | `AppConfig.primaryColor` |
| `STORE_URL_ANDROID` / `STORE_URL_IOS` | Cibles du dialogue « mise à jour disponible » | `AppConfig.storeUrl` |
| `FORCE_UPDATE` | `"true"` = bloque l'app jusqu'à la mise à jour | `AppConfig.forceUpdate` (défaut `false`) |

Toutes ces clés sont lues dans **`lib/config/app_config.dart`** via
`String.fromEnvironment`. Pour en ajouter une : la déclarer là, puis dans le JSON.

#### ⚠️ Incohérence connue sur `configs/smr26.json` — ne pas « corriger » à l'aveugle

Dans la config du congrès SMR 26, deux identifiants divergent d'un caractère :

| Clé | Valeur |
|---|---|
| `APP_PACKAGE` | `com.hash**tag**sante.smr2026` |
| `STORE_URL` / `STORE_URL_ANDROID` | `...?id=com.htagsante.smr2026` ← sans `hash` |

`APP_PACKAGE` est cohérent avec le reste du projet (bundle ID iOS dans
`project.pbxproj`, `BUNDLE_ID` dans `codemagic.yaml`) ; ce sont les **URL de store** qui
portent l'ancien nom.

**Conséquence** : sur Android, le dialogue « mise à jour disponible » renvoie vers une
fiche Play Store qui ne correspond pas à l'app installée.

**C'est un héritage assumé** : le package initial avait été mal choisi et l'app est déjà
publiée sous cette identité — le changer casserait la continuité sur le store. On garde
donc l'état actuel pour SMR 26.

> ✅ **Pour tout nouveau congrès**, choisissez l'`APP_PACKAGE` **une fois pour toutes
> avant la première publication**, et veillez à ce que `APP_PACKAGE`,
> `STORE_URL_ANDROID`, le bundle ID iOS et `codemagic.yaml` portent **exactement** le
> même identifiant. Un `applicationId` ne peut plus jamais être modifié après
> publication sur le Play Store.

### 5.2 Ce qui est automatique ✅

`android/app/build.gradle` décode les `dart-defines` (transmis en base64 par Flutter) et
en tire `applicationId` et `android:label`. Côté Android, **le JSON suffit** :

```
configs/xxx.json → --dart-define-from-file → build.gradle → applicationId + nom de l'app
```

Le `namespace` Android reste `com.hashtagsante.eventapp` : c'est normal et sans effet —
seul l'`applicationId` identifie l'app sur le store.

### 5.3 Icône / launcher

1. Déposer l'icône dans `assets/icons/<nouveau>.png`
   → **1024×1024 px, PNG, sans transparence** (exigé par l'App Store).
2. Builder via `scripts\build_android.bat <nouveau>` : copie et génération automatiques.

Pour régénérer à la main :
```powershell
copy /Y assets\icons\<nouveau>.png assets\icon\app_icon.png
dart run flutter_launcher_icons
```
Configuration dans `pubspec.yaml`, section `flutter_launcher_icons`.

### 5.4 Logo dans l'interface ⚠️ codé en dur

Le logo affiché en pied d'écran **n'est pas piloté par le JSON**. Il est référencé en dur
à deux endroits :

- `lib/main.dart:329`
- `lib/views/main_shell.dart:478`

Deux options :
- **rapide** : remplacer le fichier `assets/images/logo-ascrea.png` en gardant le nom ;
- **propre** : ajouter une clé `LOGO_ASSET` dans `AppConfig` et remplacer les deux
  chemins en dur — recommandé si vous gérez plusieurs congrès en parallèle.

### 5.5 iOS ⚠️ entièrement manuel

Le mécanisme `dart-define` ne touche **pas** le projet Xcode. À modifier à la main :

| Élément | Fichier | Valeur actuelle |
|---|---|---|
| Nom affiché | `ios/Runner/Info.plist` → `CFBundleDisplayName` **et** `CFBundleName` | `SMR 2026` |
| Bundle ID | `ios/Runner.xcodeproj/project.pbxproj` → `PRODUCT_BUNDLE_IDENTIFIER` (lignes ~375 et ~557) | `com.hashtagsante.smr2026` |
| Bundle ID CI | `codemagic.yaml` → `BUNDLE_ID` du workflow `ios-release` | idem |
| Apple ID app | `codemagic.yaml` → `APP_STORE_APPLE_ID` | `6763072246` |

> Ne pas toucher les `PRODUCT_BUNDLE_IDENTIFIER` se terminant par `.RunnerTests`.
> Le plus sûr est de passer par Xcode (*Runner → General → Identity*).
> Un build iOS exige **un Mac** (ou Codemagic, cf. §6).

### 5.6 Gérer les numéros de version ⚠️ à lire avant tout build

C'est la source d'erreur n°1 lors d'une publication. Une seule ligne dans
`pubspec.yaml` commande tout :

```yaml
version: 1.0.1+6
#        ↑     ↑
#        │     └── build number  → technique, vu par les stores uniquement
#        └──────── version name  → visible par l'utilisateur
```

#### Les deux nombres ne servent pas à la même chose

| | **Version name** (`1.0.1`) | **Build number** (`6`) |
|---|---|---|
| Qui le voit | L'utilisateur, sur la fiche du store | Personne — usage interne aux stores |
| Rôle | Communiquer l'ampleur du changement | Identifier **chaque envoi** de façon unique |
| Peut se répéter ? | Oui (plusieurs builds pour une même version) | **Jamais** |
| Où ça atterrit | `versionName` (Android) / `CFBundleShortVersionString` (iOS) | `versionCode` (Android) / `CFBundleVersion` (iOS) |

#### Règle absolue : le build number ne recule jamais

À chaque binaire envoyé sur un store, **incrémentez le build number**, même pour une
correction minuscule, même si le version name ne change pas.

```yaml
version: 1.0.1+6     # envoyé au store
version: 1.0.1+7     # ✅ correctif, même version affichée, nouveau build
version: 1.0.2+8     # ✅ version affichée mise à jour
version: 1.0.2+7     # ❌ REFUSÉ : le build 7 a déjà été utilisé
```

Un build number déjà consommé est **définitivement brûlé**, même si vous supprimez le
binaire du store. Les erreurs typiques :

- Google Play : *« Version code 6 has already been used »*
- App Store : *« The bundle version must be higher than the previously uploaded version »*

En cas de doute, sautez des numéros — passer de `+6` à `+10` ne pose aucun problème.

#### Nouveau congrès / nouvelle marque : on repart à zéro

Un nouveau congrès a un **`APP_PACKAGE` différent** (cf. §5.1), donc c'est une
**application entièrement nouvelle** aux yeux des stores. Elle n'a aucun historique :
vous repartez donc du début.

```yaml
# Dans pubspec.yaml, pour une toute première publication :
version: 0.0.1+1
```

Repères usuels :

| Situation | Version à mettre |
|---|---|
| Premières maquettes, tests internes | `0.0.1+1` |
| Version de test envoyée aux testeurs (TestFlight, Firebase) | `0.1.0+1` puis `+2`, `+3`… |
| Première publication publique sur les stores | `1.0.0+1` |
| Correctif après publication | `1.0.1+2` |
| Nouvelle fonctionnalité | `1.1.0+3` |
| Refonte majeure | `2.0.0+4` |

> ⚠️ **Ne remettez à `0.0.1+1` que pour une app réellement neuve** (nouveau
> `APP_PACKAGE`, jamais publiée). Si vous faites ça sur une app **déjà en ligne**, tous
> vos envois seront refusés : le store attend un build number supérieur à celui déjà
> reçu. Dans ce cas il faut au contraire **repartir du dernier numéro utilisé + 1**.

#### Le sens des trois chiffres du version name

`MAJEUR.MINEUR.CORRECTIF` — c'est la convention *semantic versioning* :

- **majeur** (`1.x.x` → `2.0.0`) : refonte, changement visible important
- **mineur** (`1.0.x` → `1.1.0`) : nouvelle fonctionnalité, sans casser l'existant
- **correctif** (`1.0.0` → `1.0.1`) : correction de bug uniquement

C'est une convention, pas une contrainte technique — mais elle rend l'historique
lisible pour tout le monde.

#### En pratique

1. Ouvrir `pubspec.yaml`, modifier la ligne `version:`.
2. **C'est tout** — Android et iOS lisent cette valeur automatiquement (via
   `flutter.versionName` / `flutter.versionCode` dans `build.gradle`). Ne touchez
   ni `build.gradle` ni `Info.plist` pour ça.
3. Rebuilder. Vérifier au besoin :
   ```powershell
   grep "^version:" pubspec.yaml
   ```
4. Le libellé du workflow iOS dans `codemagic.yaml` (`name: iOS Release (IPA) v1.0.1+6`)
   mentionne la version en dur : purement cosmétique, mais autant le tenir à jour.

> L'app affiche sa propre version à l'écran via `package_info_plus`, et
> `services/update_service.dart` la compare à celle renvoyée par l'API pour proposer la
> mise à jour. Une version mal renseignée fait donc apparaître — ou disparaître à tort —
> le dialogue « mise à jour disponible ».

### 5.7 Checklist « nouveau congrès »

- [ ] `configs/<nouveau>.json` créé et rempli
- [ ] `EVENT_SLUG` / `EVENT_KEY` / `BASE_URL` validés avec le backend
- [ ] **`APP_PACKAGE` identique partout** : `STORE_URL_ANDROID`, bundle ID iOS,
      `codemagic.yaml` — à figer avant la 1ᵉʳᵉ publication, plus modifiable ensuite (§5.1)
- [ ] `assets/icons/<nouveau>.png` en 1024×1024 sans alpha
- [ ] Logo interface remplacé ou rendu configurable (§5.4)
- [ ] `version:` dans `pubspec.yaml` (§5.6) — **nouvelle app** : repartir à `0.0.1+1` ·
      **app existante** : incrémenter le build number, jamais le réutiliser
- [ ] Testé sur Chrome **et** sur émulateur (§3)
- [ ] `flutter analyze` propre
- [ ] Android : `scripts\build_android.bat <nouveau> all`
- [ ] iOS : `Info.plist` + bundle ID mis à jour (§5.5)
- [ ] Keystore présent et `key.properties` rempli (§4.2)
- [ ] `CONFIG_FILE` mis à jour dans `codemagic.yaml` si CI utilisée (§6)
- [ ] APK testé sur appareil réel avant publication

---

## 6. Builds Codemagic (CI)

Codemagic construit l'app **sur ses serveurs** — c'est la seule façon de produire un
build iOS sans posséder de Mac. Toute la configuration est dans **`codemagic.yaml`**.

| Workflow | Produit | Destination |
|---|---|---|
| `android-release` | AAB + APK | artefacts téléchargeables (publication Play Store commentée) |
| `android-firebase` | APK | Firebase App Distribution (testeurs) |
| `ios-release` | IPA | TestFlight (`submit_to_app_store: false`) |
| `ios-simulator` | `Runner.app.zip` | Appetize.io (démo dans le navigateur) |

### 6.1 Mise en route

1. Connecter le dépôt GitHub sur <https://codemagic.io>.
2. Codemagic détecte `codemagic.yaml` automatiquement.
3. Créer les **groupes de variables** attendus (*Settings → Environment variables*) :

| Groupe | Variables |
|---|---|
| `android_creds` | `FCI_KEYSTORE` (le `.jks` encodé en **base64**), `FCI_KEYSTORE_PASSWORD`, `FCI_KEY_PASSWORD`, `FCI_KEY_ALIAS` |
| `firebase_creds` | `FIREBASE_TOKEN` |
| `ios_creds` | `APP_STORE_CONNECT_PRIVATE_KEY`, `APP_STORE_CONNECT_KEY_IDENTIFIER`, `APP_STORE_CONNECT_ISSUER_ID` |
| `appetize_creds` | `APPETIZE_API_TOKEN` |

Encoder le keystore en base64 :
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("smr26.jks")) | Set-Clipboard
```

4. Cocher **Secure** sur toutes les variables sensibles.
5. Lancer un build : *Start new build* → choisir le workflow.

### 6.2 Changer de congrès en CI

Modifier `CONFIG_FILE` dans le bloc `vars` du workflow concerné :
```yaml
vars:
  CONFIG_FILE: configs/<nouveau>.json
```
Pour iOS, ajuster aussi `BUNDLE_ID` et `APP_STORE_APPLE_ID`.

### 6.3 Notes utiles

- Le workflow Firebase convertit le JKS en **PKCS12** : contournement d'une erreur
  `Tag number over 30` de `keytool` sur certains keystores (avec repli automatique sur
  le `.jks` d'origine).
- `FIREBASE_APP_ID` est en clair dans `codemagic.yaml` — ce n'est pas un secret, mais il
  est propre à ce projet Firebase : à changer pour un autre congrès.
- Le workflow `ios-simulator` fabrique un binaire *fat* (`x86_64` + `arm64`) via `lipo`,
  nécessaire à Appetize.
- La publication Play Store est **désactivée** (bloc `publishing` commenté dans
  `android-release`) : la décommenter et fournir `GCLOUD_SERVICE_ACCOUNT_CREDENTIALS`.

---

## 7. Architecture du code

### 7.1 Vue d'ensemble

```
lib/
├── main.dart              Point d'entrée, MultiProvider, thème, splash, check version
├── config/
│   └── app_config.dart    ★ Toutes les valeurs white-label (dart-defines)
├── core/
│   └── api_client.dart    Client Dio partagé : baseUrl, en-têtes, token, retry
├── models/                Objets métier + parsing JSON (fromJson / toJson)
├── services/              Appels réseau : 1 service par domaine
├── providers/             État applicatif (ChangeNotifier) + cache
├── views/                 Écrans, groupés par section fonctionnelle
├── widgets/               Composants réutilisables transverses
└── utils/                 Helpers : couleurs, HTML, responsive, connectivité
```

### 7.2 Le patron à connaître

Chaque section fonctionnelle suit **le même trio**. Comprendre un domaine suffit à
comprendre les neuf autres :

```
models/speaker_model.dart       →  forme des données (fromJson)
services/speaker_service.dart   →  récupère depuis l'API (via ApiClient)
providers/speaker_provider.dart →  état + cache + notifyListeners()
views/speakers/                 →  affichage (Consumer / context.watch)
```

Domaines existants : `program`, `speaker`, `abstract`, `committee`, `sponsor`, `vod`,
`faq`. S'y ajoutent deux providers transverses : `theme_provider` (thème et **modules
activés/désactivés depuis le back-office**) et `connectivity_provider`.

**Gestion d'état** : `provider` (`ChangeNotifier`). `get` est aussi présent, utilisé
ponctuellement.
**Persistance** : `shared_preferences` (cache, agenda perso), `flutter_secure_storage`
(données sensibles).

### 7.3 Les écrans

`views/main_shell.dart` est la coquille : navigation par onglets, en-tête, logo.

| Dossier | Contenu |
|---|---|
| `home/` | Accueil : bannière, raccourcis, prochaines sessions, speakers, sponsors, badge |
| `program/` | Programme : onglets par jour, filtres, agenda perso, détail de session |
| `speakers/` | Liste et fiche intervenant |
| `abstracts/` | Abstracts / posters |
| `committee/` | Membres du comité |
| `sponsors/` | Partenaires |
| `vod/` | Replays vidéo (YouTube) |
| `infos/` | Infos pratiques, mot du président |
| `faq/` | Questions fréquentes |
| `search/` | Recherche transverse |

### 7.4 Points d'attention

- **`theme_provider`** décide quels modules sont visibles (valeurs `1`/`0` venant du
  back-office). Un écran vide vient souvent de là, **pas d'un bug d'affichage**.
- **Notifications** : `services/notification_service.dart`
  (`flutter_local_notifications` + `timezone`), repli sur `Europe/Paris` si le fuseau
  n'est pas résolu. Ce sont des notifications **locales**, pas du push.
- **Contenu HTML** : les champs riches du back-office (mot du président…) sont rendus
  avec `flutter_widget_from_html` — voir `utils/html_utils.dart`.
- **Couleurs** : le back-office renvoie parfois du `rgba(...)`;
  `utils/color_parser.dart` s'en charge. Ne pas parser une couleur à la main ailleurs.
- **Mise à jour** : `services/update_service.dart` compare la version installée
  (`package_info_plus`) à celle de l'API et propose `STORE_URL_*`.
- **Hors ligne** : les providers mettent en cache dans `shared_preferences` ;
  `connectivity_provider` surveille le réseau.

### 7.5 Ajouter une nouvelle section

1. `models/<x>_model.dart` — la classe + `fromJson`
2. `services/<x>_service.dart` — la requête via `ApiClient.dio`
3. `providers/<x>_provider.dart` — `ChangeNotifier` + cache
4. Enregistrer le provider dans le `MultiProvider` de `main.dart`
5. `views/<x>/` — l'écran
6. Brancher la navigation dans `views/main_shell.dart`
7. Si le module doit être activable côté back-office, ajouter son drapeau dans
   `theme_provider.dart`

### 7.6 Dossiers hors `lib/`

| Chemin | Rôle |
|---|---|
| `configs/` | ★ Configs white-label par congrès (versionnées) |
| `envs/` | Anciennes configs `.env` (utilisées par `scripts/build.sh`) — **git-ignoré** |
| `scripts/` | `build_android.bat` (Windows), `build.sh` (macOS/Linux, gère aussi iOS) |
| `assets/icon/` | Icône active consommée par `flutter_launcher_icons` |
| `assets/icons/` | Icônes sources, une par congrès |
| `assets/images/` | Images de l'interface (dont le logo, cf. §5.4) |
| `docs/` | Pages légales publiées (confidentialité, support, copyright) |
| `android/` `ios/` `web/` | Projets natifs par plateforme |
| `test/` | Tests automatisés |
| `kys/` `releases/` `build/` | Keystores, binaires, artefacts — **git-ignorés** |
| `codemagic.yaml` | Pipelines CI/CD |

> `build_store2.ps1` produit une seconde variante Play Store depuis
> `configs/smr26_store2.json`.

### 7.7 Sécurité — à savoir avant de commiter

Le `.gitignore` protège déjà `kys/`, `*.jks`, `*.keystore`, `*.p8`, `*.p12`, `envs/`,
`*.env`, `android/key.properties` et `releases/`. **Aucun secret n'a jamais été commité
dans l'historique de ce dépôt** (vérifié).

`EVENT_KEY` est en revanche présent en clair dans `configs/*.json` **et** comme valeur
par défaut dans `app_config.dart`. Ce n'est pas une fuite au sens strict — la clé est de
toute façon embarquée dans le binaire distribué et extractible d'un APK — mais **ne la
traitez pas comme un secret** : les droits qu'elle ouvre doivent rester limités à la
lecture des données publiques de l'événement.

---

## 8. Dépannage

| Symptôme | Cause / correctif |
|---|---|
| `Building with plugins requires symlink support` | Mode développeur Windows désactivé → §1.4 |
| `Android sdkmanager not found` / `Android license status unknown` | `cmdline-tools` non installé (§1.3). **Non bloquant** pour builder si `Sdk/licenses/` existe déjà. |
| `detected dubious ownership in repository` | Projet copié depuis un autre PC/disque : `git config --global --add safe.directory <chemin>` |
| `LF will be replaced by CRLF` | Avertissement Git bénin sous Windows, à ignorer |
| Build très lent la 1ᵉʳᵉ fois | Normal : Gradle télécharge ses dépendances (5-10 min), puis < 1 min |
| L'app affiche les données du **mauvais congrès** | `--dart-define-from-file` oublié → valeurs par défaut de `app_config.dart` (§2) |
| Un écran est **vide** sans erreur | Module probablement désactivé dans le back-office → `theme_provider` (§7.4) |
| Pas de notifications dans Chrome | Normal, désactivé par `kIsWeb` → tester sur émulateur (§3.2) |
| Erreurs de build inexplicables | `flutter clean` puis `flutter pub get` (§3.8) |
| Modification du JSON sans effet | Un hot reload ne recharge pas les `dart-define` : quitter (`q`) et relancer (§3.5) |
| `Gradle task assembleDebug failed` | Lire l'erreur complète plus haut dans la sortie ; souvent JDK (§1.3) ou mode dev (§1.4) |
| L'émulateur n'apparaît pas dans `flutter devices` | Attendre qu'il ait fini de démarrer, puis `adb devices` |
| Avertissements Gradle / AGP / Kotlin | Attendus → §1.7, non bloquants |
| `Version code N has already been used` (Play) | Build number déjà consommé → incrémenter le `+N` dans `pubspec.yaml` (§5.6) |
| `The bundle version must be higher than...` (App Store) | Même cause, côté Apple → §5.6 |
| Le dialogue « mise à jour » s'affiche à tort (ou jamais) | Décalage entre `version:` et la version renvoyée par l'API → §5.6 |

---

## 9. Aide-mémoire

```powershell
# ── Installation / diagnostic ─────────────────────────────────
flutter pub get                    # dépendances
flutter doctor -v                  # diagnostic environnement détaillé
flutter devices                    # appareils disponibles
flutter emulators                  # émulateurs installés
flutter emulators --launch Pixel_8 # démarrer un émulateur

# ── Lancer / tester ───────────────────────────────────────────
flutter run -d chrome        --dart-define-from-file=configs/smr26.json
flutter run -d emulator-5554 --dart-define-from-file=configs/smr26.json
flutter analyze                    # lint (à faire avant chaque commit)
flutter test                       # tests unitaires
dart format lib/                   # reformater le code
flutter logs                       # logs de l'appareil

# ── Version (avant chaque envoi sur un store, cf. §5.6) ───────
grep "^version:" pubspec.yaml   # version actuelle : <name>+<build>
#   nouvelle app  → 0.0.1+1
#   app existante → incrémenter le build, JAMAIS le réutiliser

# ── Produire les binaires ─────────────────────────────────────
scripts\build_android.bat smr26 all                             # APK + AAB
flutter build apk       --release --dart-define-from-file=configs/smr26.json
flutter build appbundle --release --dart-define-from-file=configs/smr26.json
flutter build web                 --dart-define-from-file=configs/smr26.json
dart run flutter_launcher_icons                                 # régénérer les icônes
adb install -r build\app\outputs\flutter-apk\smr2026-release.apk

# ── En cas de problème ────────────────────────────────────────
flutter clean; flutter pub get
```

---

*Environnement validé par build et exécution réels le 2026-09-04 : APK debug Android
compilé, app lancée sur émulateur Pixel (Android 15) et dans Chrome.*
