# event_app

Application mobile Flutter **white-label** de gestion de congrès médicaux : programme,
speakers, abstracts, comités, sponsors, VOD, FAQ et notifications locales.

Une seule base de code sert plusieurs congrès : tout ce qui change d'un événement à
l'autre (nom, package, couleur, API, slug) vit dans un fichier de configuration JSON
injecté au build.

**Cibles** : Android · iOS · Web

---

## 📖 Documentation

👉 **[ONBOARDING.md](ONBOARDING.md) — guide développeur complet**

Il part de zéro et couvre : l'installation de l'environnement, le premier lancement, les
tests (Chrome / émulateur / téléphone), la génération d'APK, la déclinaison white-label
vers un nouveau congrès, les builds Codemagic, l'architecture du code et le dépannage.

---

## ⚡ Démarrage rapide

```powershell
flutter pub get
flutter doctor
flutter run -d chrome --dart-define-from-file=configs/smr26.json
```

> ⚠️ Toujours passer `--dart-define-from-file`. Sans lui, l'app utilise les valeurs par
> défaut de `lib/config/app_config.dart` et vous ne savez pas quelle configuration vous
> testez. Voir [ONBOARDING.md §2](ONBOARDING.md#2-premier-lancement-pas-à-pas).

Générer un APK release :
```powershell
scripts\build_android.bat smr26
```

---

## 🗂 Structure

| Chemin | Rôle |
|---|---|
| `lib/config/app_config.dart` | ★ Toutes les valeurs white-label |
| `configs/*.json` | Une configuration par congrès |
| `lib/models` · `services` · `providers` · `views` | Le patron répété par domaine |
| `scripts/` | Scripts de build Android / iOS |
| `codemagic.yaml` | Pipelines CI/CD (Android, iOS, Firebase, Appetize) |

Détail complet dans [ONBOARDING.md §7](ONBOARDING.md#7-architecture-du-code).

---

## 🔒 Sécurité

`kys/`, `*.jks`, `*.keystore`, `envs/`, `*.env`, `android/key.properties` et `releases/`
sont exclus par `.gitignore` — **ne jamais les commiter**.

Voir [ONBOARDING.md §7.7](ONBOARDING.md#77-sécurité--à-savoir-avant-de-commiter).
