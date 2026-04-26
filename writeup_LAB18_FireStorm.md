# 🔥 LAB 18 – FireStorm | Writeup

> **Platform:** pwnsec.io  
> **Category:** Mobile Security / Android Reverse Engineering  
> **Difficulty:** Medium  
> **Flag:** `PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}`

---

## 📋 Description

L'application Android `com.pwnsec.firestorm` expose des credentials Firebase protégés par une méthode Java qui fait appel à une fonction native (bibliothèque `.so`). L'objectif est de récupérer le mot de passe Firebase généré dynamiquement à l'exécution, puis de l'utiliser pour s'authentifier sur Firebase et lire le flag dans la base de données.

---

## 🛠️ Outils utilisés

| Outil | Usage |
|-------|-------|
| **Frida 17.9.1** | Instrumentation dynamique Android |
| **ADB** | Communication avec l'émulateur |
| **jadx-gui** | Analyse statique du code Java décompilé |
| **Android Emulator (x86_64)** | Environnement d'exécution rooté |
| **Python + requests** | Connexion à l'API Firebase |

---

## 🔍 Étape 1 – Analyse statique avec jadx

On commence par décompiler l'APK avec `jadx-gui` pour comprendre la logique de l'application.

```bash
jadx-gui FireStorm.apk
```

Dans `com.pwnsec.firestorm.MainActivity`, on identifie la méthode critique :

```java
public String Password() {
    StringBuilder sb = new StringBuilder();
    String string  = getString(R.string.Friday_Night);
    String string2 = getString(R.string.Author);
    String string3 = getString(R.string.JustRandomString);
    String string4 = getString(R.string.URL);
    String string5 = getString(R.string.IDKMaybethepasswordpassowrd);
    String string6 = getString(R.string.Token);

    sb.append(string.substring(5, 9));
    sb.append(string4.substring(1, 6));
    sb.append(string2.substring(2, 6));
    sb.append(string5.substring(5, 8));
    sb.append(string3);
    sb.append(string6.substring(18, 26));

    return generateRandomString(String.valueOf(sb));
}

public native String generateRandomString(String str);
```

**Observations clés :**
- `Password()` construit une chaîne à partir de ressources Android (strings.xml)
- Elle appelle `generateRandomString()` — une **fonction native** dans une lib `.so`
- Le résultat final est imprévisible sans exécuter le code natif

👉 L'analyse statique seule ne suffit pas — il faut exécuter la méthode dynamiquement avec **Frida**.

---

## ⚙️ Étape 2 – Configuration de l'environnement

### 2.1 Préparer l'émulateur

```powershell
# Vérifier l'architecture de l'émulateur
adb shell getprop ro.product.cpu.abi
# → x86_64
```

> ⚠️ Utiliser une image **AOSP / Google APIs** (sans Google Play) pour avoir le root.

### 2.2 Installer frida-server sur l'émulateur

Télécharger `frida-server-17.9.1-android-x86_64.xz` depuis :  
`https://github.com/frida/frida/releases/tag/17.9.1`

```powershell
# Pousser et lancer frida-server
adb push frida-server-17.9.1-android-x86_64 /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server &"

# Vérifier qu'il tourne
adb shell ps -e | grep frida
# → root   8735   1   ...   frida-server ✅
```

### 2.3 Installer l'APK

```powershell
adb install FireStorm.apk
# → Success ✅
```

---

## 💉 Étape 3 – Instrumentation dynamique avec Frida

### Script `frida_firestorm.js`

```javascript
Java.perform(function() {

    function getPassword() {
        console.log("[*] Recherche des instances de MainActivity...");

        Java.choose('com.pwnsec.firestorm.MainActivity', {

            onMatch: function(instance) {
                console.log("[+] Instance trouvée : " + instance);
                try {
                    var pass = instance.Password();
                    console.log("[+] Mot de passe Firebase généré : " + pass);
                } catch (e) {
                    console.log("[-] Erreur : " + e);
                }
            },

            onComplete: function() {
                console.log("[*] Recherche terminée.");
            }
        });
    }

    console.log("[*] Script chargé. Attente de 3 secondes...");
    setTimeout(getPassword, 3000);
});
```

**Explication :**
- `Java.perform()` — initialise l'environnement Java de Frida
- `Java.choose()` — parcourt la heap pour trouver les instances vivantes de `MainActivity`
- `instance.Password()` — appelle directement la méthode qui déclenche aussi le code natif
- `setTimeout(..., 3000)` — attend que l'app et la lib native soient chargées

### Lancement

```powershell
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js
```

### Résultat

```
[*] Script chargé. Attente de 3 secondes avant exécution...
[*] Début de la recherche d'instances de MainActivity...
[+] MainActivity instance trouvée : com.pwnsec.firestorm.MainActivity@6728361
[+] Mot de passe Firebase généré : C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC
[*] Recherche des instances terminée.
```

---

## 🔐 Étape 4 – Authentification Firebase et récupération du flag

Avec le mot de passe obtenu, on s'authentifie directement sur l'API Firebase REST :

```python
import requests

API_KEY = "AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY"
email   = "TK757567@pwnsec.xyz"
password = "C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC"

# Authentification
auth_url = f"https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key={API_KEY}"
auth_response = requests.post(auth_url, json={
    "email": email,
    "password": password,
    "returnSecureToken": True
})

auth_data = auth_response.json()

if "idToken" in auth_data:
    print("Connexion reussie !")
    id_token = auth_data["idToken"]

    # Lecture du flag dans la Realtime Database
    db_url = f"https://firestorm-9d3db-default-rtdb.firebaseio.com/.json?auth={id_token}"
    db_response = requests.get(db_url)
    print("FLAG :", db_response.json())
```

### Résultat

```
Connexion reussie !
FLAG : PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
```

---

## 🧠 Concepts clés appris

| Concept | Description |
|---------|-------------|
| **Analyse statique** | Décompiler un APK avec jadx pour comprendre la logique |
| **Analyse dynamique** | Exécuter du code Java à runtime avec Frida |
| **Java.choose()** | Trouver des instances d'objets vivants en mémoire heap |
| **Fonctions natives** | Appeler du code C/C++ via JNI sans le désassembler |
| **Firebase REST API** | S'authentifier et lire des données sans SDK |

---

## 🏁 Flag

```
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
```

---

*Writeup rédigé par **ELAANTRI Malika** — pwnsec.io LAB 18 FireStorm*
