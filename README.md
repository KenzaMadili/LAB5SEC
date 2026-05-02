# Lab 5 — Android Native Code Analysis  (Auteur : MADILI Kenza)
## OWASP UnCrackable Level 2

---

<img width="215" height="354" alt="Capture d’écran 2026-05-02 133602" src="https://github.com/user-attachments/assets/b585f8d9-132e-4402-a288-bb213eee6101" />

## 📌 Aperçu

Ce lab consiste à analyser une application Android qui cache sa logique de vérification dans une bibliothèque native compilée (`libfoo.so`). L'objectif est de retrouver le secret attendu en combinant analyse statique (JADX, Ghidra) et analyse dynamique (Frida).

L'application présente un champ de saisie et un bouton de vérification. La comparaison n'est pas visible dans le code Java — elle est déléguée à une fonction native via JNI.

---

## 🎯 Objectifs pédagogiques

- Comprendre le rôle d'un APK dans Android
- Observer le comportement d'une application avant de lire son code
- Identifier le point de départ d'une vérification dans `MainActivity`
- Reconnaître le chargement d'une bibliothèque native avec `System.loadLibrary`
- Comprendre le principe d'une méthode `native` en Java
- Localiser un fichier `.so` dans un APK Android
- Contourner une détection root avec Frida
- Ouvrir une bibliothèque native dans Ghidra
- Retrouver une fonction JNI exportée
- Repérer une comparaison de chaînes avec `strncmp`
- Décoder une valeur hexadécimale ASCII
- Retrouver le secret final et le valider dans l'application

---

## 🧰 Outils utilisés

| Outil | Version | Rôle |
|---|---|---|
| Windows 10/11 | — | Système hôte |
| Android Command-Line Tools | 12.0 | ADB + émulateur + AVD |
| Android Emulator | 36.4.x | Émulateur x86_64 (API 30) |
| ADB | 1.0.41 | Communication avec l'émulateur |
| JADX | 1.5.0 | Décompilation du code Java |
| Ghidra | 11.1.2 | Analyse du binaire natif `.so` |
| Frida | 17.9.3 | Bypass dynamique de la détection root |
| Python | 3.x | Décodage des valeurs hexadécimales |
| APK OWASP UnCrackable L2 | — | Application cible |

---

## ⚙️ Installation de l'environnement

### 1. Android Command-Line Tools

```powershell
# Téléchargement
Invoke-WebRequest -Uri "https://dl.google.com/android/repository/commandlinetools-win-11076708_latest.zip" -OutFile "$env:TEMP\cmdtools.zip" -UseBasicParsing

# Extraction et organisation
Expand-Archive "$env:TEMP\cmdtools.zip" -DestinationPath "$env:LOCALAPPDATA\Android\cmdtools" -Force
New-Item -ItemType Directory -Force -Path "$env:LOCALAPPDATA\Android\Sdk\cmdline-tools\latest"
Move-Item "$env:LOCALAPPDATA\Android\cmdtools\cmdline-tools\*" "$env:LOCALAPPDATA\Android\Sdk\cmdline-tools\latest\" -Force

# Variables d'environnement
[System.Environment]::SetEnvironmentVariable("ANDROID_HOME", "$env:LOCALAPPDATA\Android\Sdk", "User")
$env:ANDROID_HOME = "$env:LOCALAPPDATA\Android\Sdk"
$env:Path += ";$env:ANDROID_HOME\cmdline-tools\latest\bin;$env:ANDROID_HOME\platform-tools;$env:ANDROID_HOME\emulator"
```

### 2. SDK Android + Émulateur

```powershell
# Accepter les licences
echo "y" | sdkmanager --licenses

# Installer les composants
sdkmanager "platform-tools" "emulator" "platforms;android-30" "system-images;android-30;google_apis;x86_64"
```

### 3. Créer et lancer l'AVD

<img width="668" height="292" alt="Capture d’écran 2026-05-02 131214" src="https://github.com/user-attachments/assets/86141e41-97a8-486c-99bb-89e8bad31d57" />

<img width="658" height="384" alt="Capture d’écran 2026-05-02 133243" src="https://github.com/user-attachments/assets/5ac4554b-fbae-4782-9299-337389c5fd2d" />


### 4. JADX

```powershell
Invoke-WebRequest -Uri "https://github.com/skylot/jadx/releases/download/v1.5.0/jadx-1.5.0.zip" -OutFile "$env:TEMP\jadx.zip" -UseBasicParsing
Expand-Archive "$env:TEMP\jadx.zip" -DestinationPath "C:\jadx" -Force
```

### 5. Ghidra

```powershell
Invoke-WebRequest -Uri "https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_11.1.2_build/ghidra_11.1.2_PUBLIC_20240709.zip" -OutFile "$env:TEMP\ghidra.zip" -UseBasicParsing
Expand-Archive "$env:TEMP\ghidra.zip" -DestinationPath "C:\ghidra" -Force
Start-Process "C:\ghidra\ghidra_11.1.2_PUBLIC\ghidraRun.bat"
```

### 6. Frida

<img width="671" height="69" alt="Capture d’écran 2026-05-02 134035" src="https://github.com/user-attachments/assets/6d866cbd-1a76-4fde-8f06-43ae646dae3f" />

## 🚀 Déroulement du lab

### Partie 1 — Installation de l'APK

<img width="666" height="94" alt="Capture d’écran 2026-05-02 133404" src="https://github.com/user-attachments/assets/6e4f432b-da06-4f46-8db9-bf46a7738ac9" />

> L'application affiche **"Root detected!"** — c'est une protection qu'on va contourner.

---

### Partie 2 — Bypass de la détection root avec Frida

#### Lancer frida-server sur l'émulateur

<img width="514" height="88" alt="Capture d’écran 2026-05-02 134344" src="https://github.com/user-attachments/assets/497d7ba4-d9c4-480b-862e-a1822e621911" />

#### Créer le script de bypass

<img width="428" height="134" alt="Capture d’écran 2026-05-02 134544" src="https://github.com/user-attachments/assets/969b6200-28f0-41c3-ae08-cc6163caa721" />

> La classe `sg.vantagepoint.a.b` contient les 3 méthodes de détection root : `a()`, `b()`, `c()`. On les force toutes à retourner `false`.

#### Lancer Frida avec le bypass

<img width="1324" height="753" alt="Capture d&#39;écran 2026-05-02 135923" src="https://github.com/user-attachments/assets/90383332-8df8-4492-b9e5-7450fd852670" />

✅ L'application s'ouvre sans message d'erreur.

---

### Partie 3 — Analyse statique avec JADX

```powershell
# Décompiler l'APK
C:\jadx\bin\jadx.bat "$env:USERPROFILE\Desktop\UnCrackable-Level2.apk" -d "C:\jadx-output"
```

#### MainActivity.java — Points clés

```java
static {
    System.loadLibrary("foo");  // Charge libfoo.so
}

// Vérification root
if (b.a() || b.b() || b.c()) {
    a("Root detected!");
}

// Vérification du secret
public void verify(View view) {
    String obj = ((EditText) findViewById(R.id.edit_text)).getText().toString();
    if (this.m.a(obj)) {
        // Success
    }
}
```

#### CodeCheck.java — La vérification est native

```java
public class CodeCheck {
    private native boolean bar(byte[] bArr);  // Fonction native JNI

    public boolean a(String str) {
        return bar(str.getBytes());           // Délègue au code natif
    }
}
```

> La logique de comparaison n'est pas dans le Java — elle est dans `libfoo.so`.

---

### Partie 4 — Extraction de libfoo.so

```powershell
# Un APK est un ZIP — on peut l'extraire directement
Expand-Archive "$env:USERPROFILE\Desktop\UnCrackable-Level2.apk" -DestinationPath "C:\apk-extracted" -Force

# Lister les bibliothèques natives
Get-ChildItem "C:\apk-extracted" -Recurse -Filter "*.so"
```

On prend la version **x86_64** correspondant à notre émulateur :
```
C:\apk-extracted\lib\x86_64\libfoo.so
```

---

### Partie 5 — Analyse avec Ghidra

1. Lancer Ghidra : `C:\ghidra\ghidra_11.1.2_PUBLIC\ghidraRun.bat`
<img width="744" height="890" alt="Capture d&#39;écran 2026-05-02 142211" src="https://github.com/user-attachments/assets/c88eb5d7-c01a-40c9-adbb-8474a17832bd" />

2. **File → New Project** → Non-Shared → nom `lab`
3. **File → Import File** → sélectionner `C:\apk-extracted\lib\x86_64\libfoo.so`
4. Double-cliquer sur `libfoo.so` → **Yes** → **Analyze**
5. Dans **Symbol Tree → Functions** → ouvrir `Java_sg_vantagepoint_uncrackable2_CodeCheck_bar`

<img width="946" height="481" alt="Capture d’écran 2026-05-02 143415" src="https://github.com/user-attachments/assets/8e9743ad-cfe8-4944-af8f-9a90e476d5c0" />

#### Code décompilé par Ghidra

```c
undefined8
Java_sg_vantagepoint_uncrackable2_CodeCheck_bar(long *param_1, undefined8 param_2, undefined8 param_3)
{
  int iVar1;
  char *__s1;
  undefined4 local_38;
  undefined4 uStack_34;
  undefined4 uStack_30;
  undefined4 uStack_2c;
  undefined8 local_28;

  if (DAT_0010400c == '\x01') {
    local_38 = 0x6e616854;   // "Than"
    uStack_34 = 0x6620736b;  // "ks f"
    uStack_30 = 0x6120726f;  // "or a"
    uStack_2c = 0x74206c6c;  // "ll t"
    local_28 = 0x68736966206568; // "he fish"

    __s1 = (char *)(**(code **)(*param_1 + 0x5c0))(param_1, param_3, 0);
    iVar1 = (**(code **)(*param_1 + 0x558))(param_1, param_3);

    if (iVar1 == 0x17) {                          // longueur = 23 caractères
      iVar1 = strncmp(__s1, (char *)&local_38, 0x17);
      if (iVar1 == 0) {
        return 1;                                 // Succès !
      }
    }
  }
  return 0;
}
```

---

### Partie 6 — Décodage du secret

Les valeurs hexadécimales sont stockées en **little-endian**. On les décode avec Python :

<img width="328" height="133" alt="Capture d’écran 2026-05-02 143540" src="https://github.com/user-attachments/assets/e3046d0e-04fe-4e87-8818-0d4becf33e95" />

**Résultat :**
```
Thanks for all the fish
```

---

### Partie 7 — Validation

Sur l'émulateur, dans l'application (lancée avec Frida) :
1. Saisir : `Thanks for all the fish`
2. Appuyer sur **VERIFY**
3. Message affiché : **"Success! This is the correct secret."** ✅
<img width="441" height="733" alt="Capture d&#39;écran 2026-05-02 143656" src="https://github.com/user-attachments/assets/120263e3-8ad8-48b0-8da9-13cbf581ddde" />

---

## 🧠 Points clés à retenir

| Concept | Explication |
|---|---|
| **APK = ZIP** | On peut extraire n'importe quel fichier avec `unzip` ou `Expand-Archive` |
| **JNI** | Pont entre Java (`native`) et code C compilé dans un `.so` |
| **`System.loadLibrary`** | Indique qu'une lib native est utilisée — toujours vérifier |
| **Little-endian** | Les entiers sont stockés à l'envers en mémoire x86 |
| **`strncmp`** | Pattern classique de comparaison de secret dans le code natif |
| **Frida** | Permet de modifier le comportement d'une app au runtime sans recompiler |
| **Ghidra** | Décompile un binaire ELF `.so` en C lisible |

---

## 🏁 Secret final

```
Thanks for all the fish
```

---

## 📚 Références

- [OWASP MASTG](https://mas.owasp.org/MASTG/)
- [OWASP UnCrackable Apps](https://github.com/OWASP/owasp-mastg/tree/master/Crackmes)
- [Frida Documentation](https://frida.re/docs/home/)
- [Ghidra](https://ghidra-sre.org/)
- [JADX](https://github.com/skylot/jadx)
