# Configuration SSH avec YubiKey (Windows 11, PowerShell/OpenSSH, PuTTY) & WSL2

Ce dépôt contient un guide pas-à-pas pour configurer une **YubiKey 5 NFC** comme **agent SSH unique** pour :

- **Windows 11** (PowerShell / OpenSSH)
- **PuTTY**
- **WSL2** (Ubuntu, Kali, etc.)

> Objectif : utiliser la YubiKey pour les connexions SSH sans dupliquer de clés privées sur le disque.

---

## Table des matières

1. [Installation des prérequis](#1-installation-des-prérequis)
2. [Configuration de l’agent GPG](#2-configuration-de-lagent-gpg)
3. [Démarrage automatique (Windows)](#3-démarrage-automatique-windows)
4. [Pont SSH pour WSL2 (Linux)](#4-pont-ssh-pour-wsl2-linux)
5. [Récupérer la clé publique SSH](#5-récupérer-la-clé-publique-ssh)
6. [Utilisation (PuTTY & SSH)](#6-utilisation-putty--ssh)
7. [Utilisation de plusieurs YubiKeys](#7-utilisation-de-plusieurs-yubikeys)
8. [Dépannage](#8-dépannage)

---

## 1. Installation des prérequis

### Windows

- **GPG4Win** : téléchargez et installez (au minimum le composant **GnuPG**).
- **wsl2-ssh-pageant** : téléchargez le binaire `.exe` depuis le dépôt GitHub de **BlackReloaded**.
  - Placez-le dans un dossier stable de votre profil utilisateur, par exemple :
    - `C:\Users\VotreNom\wsl2-ssh-pageant.exe`

### WSL2 (Ubuntu/Kali/…)

Installez `socat` (sert de “pont” entre Linux et l’agent sous Windows) :

```bash
sudo apt update && sudo apt install socat -y
```

---

## 2. Configuration de l’agent GPG

Éditez (ou créez) le fichier :

- `%APPDATA%\gnupg\gpg-agent.conf`

Contenu recommandé (GnuPG **2.5+**) :

```conf
enable-ssh-support
enable-putty-support
enable-win32-openssh-support
default-cache-ttl 600
max-cache-ttl 7200
```

⚠️ **Important :** ne pas ajouter `use-standard-socket` (option obsolète qui provoque des conflits).

---

## 3. Démarrage automatique (Windows)

Pour que l’agent soit disponible dès l’ouverture de session :

1. `Win + R`
2. Tapez : `shell:startup`
3. Clic droit → **Nouveau** → **Raccourci**
4. **Cible** : `gpg-connect-agent /bye`
5. Nom : `GPG Agent Startup`

---

## 4. Pont SSH pour WSL2 (Linux)

### 4.1 Créer le lien symbolique vers wsl2-ssh-pageant.exe

Dans WSL2 (en adaptant `<VOTRE_NOM>`) :

```bash
mkdir -p ~/.ssh
ln -sf /mnt/c/Users/<VOTRE_NOM>/wsl2-ssh-pageant.exe ~/.ssh/wsl2-ssh-pageant.exe
```

### 4.2 Ajouter la configuration au `~/.bashrc`

Ajoutez ce bloc à la fin de `~/.bashrc` :

```bash
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
if ! ss -a | grep -q "$SSH_AUTH_SOCK"; then
  rm -f "$SSH_AUTH_SOCK"
  wsl2_ssh_pageant_bin="$HOME/.ssh/wsl2-ssh-pageant.exe"
  if test -x "$wsl2_ssh_pageant_bin"; then
    (setsid nohup socat UNIX-LISTEN:"$SSH_AUTH_SOCK,fork" EXEC:"$wsl2_ssh_pageant_bin" >/dev/null 2>&1 &)
  fi
fi
```

Rechargez votre shell :

```bash
source ~/.bashrc
```

---

## 5. Récupérer la clé publique SSH

Dans un terminal Windows (PowerShell) :

```powershell
gpg --export-ssh-key
```

Copiez la ligne générée dans `~/.ssh/authorized_keys` sur votre serveur distant.

---

## 6. Utilisation (PuTTY & SSH)

### PowerShell / WSL2

```bash
ssh user@ip
```

La YubiKey doit clignoter pour confirmation (selon votre configuration).

### PuTTY

- Ne configurez **aucune clé** dans : `Connection > SSH > Auth`
- PuTTY utilisera l’agent automatiquement.

---

## 7. Utilisation de plusieurs YubiKeys

Pour forcer l’agent à relire une nouvelle clé branchée “à chaud” :

```powershell
gpg-connect-agent "scd serialno" "learn --force" /bye
```

---

## 8. Dépannage

### Reset de l’agent (Windows)

1. Terminer le processus :

```powershell
taskkill /F /IM gpg-agent.exe
```

2. Nettoyer les sockets `S.*` dans :

- `%APPDATA%\gnupg`
- `%LOCALAPPDATA%\gnupg`

3. Relancer l’agent :

```powershell
gpg-connect-agent /bye
```

---

## Notes

- Pensez à garder votre environnement à jour (GnuPG, WSL, OpenSSH).
- Si vous avez plusieurs shells (PowerShell, Windows Terminal, WSL), vérifiez que l’agent est bien actif avant vos connexions.
