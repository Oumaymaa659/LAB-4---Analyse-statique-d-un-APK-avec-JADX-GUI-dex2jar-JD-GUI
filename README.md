# 🛡️ Lab 4 : Analyse statique d'un APK

## 🎯 Objectif du Lab
L'objectif de ce laboratoire est de comprendre la structure interne d'une application Android (fichier `.apk`), d'analyser ses composants et de préparer le terrain pour trouver d'éventuelles vulnérabilités sans exécuter l'application.

## 🛠️ Environnement de travail
* **Système d'exploitation :** Windows
* **Terminal utilisé :** PowerShell
* **Application cible :** `app-debug.apk`

---

## 📝 Task 1 : Préparation du Workspace et vérification de l'APK

Pour commencer cette analyse proprement, j'ai mis en place une "salle blanche" (un dossier de travail dédié) et j'ai vérifié l'intégrité de mon fichier.

### 1. Création du dossier de travail
Création du dossier `C:\APK-Analysis` et copie du fichier `.apk` à l'intérieur.

### 2. Vérification de la signature ZIP de l'APK
Un fichier APK est en réalité une archive ZIP. J'ai utilisé PowerShell pour lire les premiers octets du fichier et vérifier sa signature magique (Hex: `50 4B` / ASCII: `PK`).

![Création du dossier et vérification de la signature ZIP](images/1.png)

### 3. Lecture du contenu de l'APK sans extraction
Utilisation des librairies de compression de Windows pour lister les 20 premiers fichiers cachés dans l'APK (comme `AndroidManifest.xml` ou `classes.dex`).

### 4. Calcul de l'empreinte (Hash SHA-256)
Pour assurer la traçabilité de l'audit et prouver que le fichier n'a pas été altéré, j'ai calculé le hash SHA-256 de l'application.

![Liste des fichiers internes et calcul du Hash SHA-256](images/2.png)

---

## 📦 Task 2 : Extraire/obtenir l'APK

**En résumé :** L'objectif de cette étape est de valider la présence de l'APK dans notre environnement et de documenter son origine. J'ai opté pour l'Option A de mon laboratoire.

**Check-list de validation :**
* ✅ **Disponibilité :** L'APK est bien présent dans le dossier de travail `C:\APK-Analysis`.
* ✅ **Provenance :** Application d'entraînement fournie (OWASP UnCrackable Level 1), renommée en `app-debug.apk` pour les besoins du lab.
* ✅ **Taille de l'APK :** 66 Ko.

![Présence de l'APK dans le dossier de travail](images/3.png)

---

## 🔍 Task 3 : Analyse avec JADX GUI

**En résumé :** J'ai utilisé JADX GUI pour décompiler l'APK et analyser son fichier `AndroidManifest.xml`. Ce fichier est crucial car il déclare les permissions, les composants et les règles de sécurité de l'application.

### 1. Informations générales de l'application
En lisant le Manifeste, j'ai pu extraire la carte d'identité de l'APK :
* **Package principal :** `owasp.mstg.uncrackable1`
* **Version (versionName) :** `1.0`
* **Version Minimale du SDK (minSdk) :** `19`
* **Version Cible du SDK (targetSdk) :** `28`

### 2. Analyse des permissions
* **Résultat :** Aucune balise `<uses-permission>` n'est présente dans le manifeste. L'application ne requiert aucun accès spécifique au système (caméra, contacts, localisation, etc.).

### 3. Analyse des composants
* L'application possède une activité principale : `sg.vantagepoint.uncrackable1.MainActivity`.
* ⚠️ **Attention de sécurité (Surface d'attaque) :** Cette activité possède un `<intent-filter>` (`android.intent.action.MAIN`). Cela signifie qu'elle est implicitement exportée et peut être lancée par le système ou d'autres applications.

### 4. Configurations de sécurité sensibles
* **CleartextTraffic / Debuggable :** Les attributs `android:usesCleartextTraffic="true"` et `android:debuggable="true"` sont absents. Bonne pratique de sécurité respectée.
* 🚨 **Vulnérabilité identifiée :** L'attribut `android:allowBackup="true"` est présent. Cela constitue un risque de fuite de données, car un attaquant disposant d'un accès physique pourrait extraire les données de l'application via la commande `adb backup`.

![Analyse du Manifeste avec JADX](images/4.png)

---

## 🕵️ Task 4 : Recherche de chaînes sensibles

**En résumé :** L'objectif de cette étape est d'utiliser la fonction de recherche globale de JADX GUI pour débusquer d'éventuelles informations sensibles (mots de passe, clés d'API, URLs cachées) codées en dur par les développeurs. Conformément à la grille de sévérité du laboratoire, voici le rapport des 5 observations demandées.

### Observation 1 : Recherche d'URLs (`http://` / `https://`)
* **Valeur trouvée :** `http://schemas.android.com/apk/res/android`
* **Emplacement :** `AndroidManifest.xml` (balise `manifest`)
* **Niveau de risque :** Faible 🟢
* **Description :** Il s'agit simplement de l'URL standard et publique utilisée par le système Android pour définir les règles du fichier XML. Il n'y a aucune fuite de données ici.

![Recherche de chaînes sensibles dans JADX](images/5.png)

---

## 🔄 Task 5 : Convertir DEX en JAR avec dex2jar

**En résumé :** Le code de l'application est initialement compilé au format `.dex` (Dalvik Executable), optimisé pour Android mais difficile à analyser avec des outils Java classiques. L'objectif de cette étape est d'isoler ce fichier et de le traduire en une archive standard `.jar`.

### 1. Extraction du fichier DEX
J'ai utilisé PowerShell pour décompresser l'APK de manière ciblée et extraire uniquement le cœur de l'application (`classes.dex`) dans un dossier `dex_out`.

![Création du dossier dex_out](images/6.png)

![Extraction du fichier classes.dex](images/7.png)

### 2. Conversion en JAR
J'ai ensuite utilisé l'outil en ligne de commande `dex2jar` pour effectuer la traduction. L'opération a généré avec succès le fichier `app.jar`.

```powershell
cd C:\APK-Analysis\dex2jar
.\d2j-dex2jar.bat "C:\APK-Analysis\dex_out\classes.dex" -o "C:\APK-Analysis\app.jar"
```

![Conversion DEX vers JAR avec dex2jar](images/8.png)
