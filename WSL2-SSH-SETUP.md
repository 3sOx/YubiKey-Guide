# 📑 Configuration SSH avec YubiKey sous Windows 11 & WSL2

Ce guide permet de configurer une **YubiKey 5 NFC** comme agent SSH unique pour Windows 11 (PowerShell/OpenSSH), PuTTY, et WSL2 (Kali/Ubuntu).

---

## 📋 Table des matières
1. [Installation des prérequis](#1-installation-des-prérequis)
2. [Configuration de l'Agent GPG](#2-configuration-de-lagent-gpg)
3. [Démarrage automatique (Windows)](#3-démarrage-automatique-windows)
4. [Pont SSH pour WSL2 (Linux)](#4-pont-ssh-pour-wsl2-linux)
5. [Récupérer la clé publique SSH](#5-récupérer-la-clé-publique-ssh)
6. [Utilisation (PuTTY & SSH)](#6-utilisation-putty--ssh)
7. [Utilisation de plusieurs YubiKeys](#7-utilisation-de-plusieurs-yubikeys)
8. [Dépannage (Troubleshooting)](#8-dépannage-troubleshooting)

---

## 1. Installation des prérequis

### Windows
* **GPG4Win** : [Télécharger ici](https://gpg4win.org/). Installez au minimum le composant `GnuPG`.
* **wsl2-ssh-pageant** : [Télécharger le binaire .exe](https://github.com/BlackReloaded/wsl2-ssh-pageant/releases).
  * Placez le fichier dans un dossier stable de votre profil utilisateur, par exemple : `C:\Users\<VOTRE_NOM>\wsl2-ssh-pageant.exe`.

### WSL2 (Kali/Ubuntu/...)
Installez `socat` pour permettre la communication entre Linux et le pont Windows :
```bash
sudo apt update && sudo apt install socat -y
2. Configuration de l'Agent GPG
Éditez le fichier %APPDATA%\gnupg\gpg-agent.conf. S'il n'existe pas, créez-le.

Contenu recommandé pour Windows 11 & GnuPG 2.5+ :

Plaintext

enable-ssh-support
enable-putty-support
enable-win32-openssh-support
default-cache-ttl 600
max-cache-ttl 7200
[!IMPORTANT] Ne pas ajouter use-standard-socket. Cette option est désormais obsolète et provoque des conflits d'instance sur les versions récentes de GPG.

3. Démarrage automatique (Windows)
Pour que l'agent SSH soit disponible dès l'ouverture de session :

Faites Win + R, tapez shell:startup et validez.

Clic droit > Nouveau > Raccourci.

Cible : gpg-connect-agent /bye

Nommez-le : GPG Agent Startup.

4. Pont SSH pour WSL2 (Linux)
Création du lien symbolique
Dans votre terminal WSL2, créez un lien vers le binaire Windows (ajustez <VOTRE_NOM>) :

Bash

mkdir -p ~/.ssh
ln -sf /mnt/c/Users/<VOTRE_NOM>/wsl2-ssh-pageant.exe ~/.ssh/wsl2-ssh-pageant.exe
Configuration du .bashrc (ou .zshrc)
Ajoutez ce bloc à la fin de votre fichier ~/.bashrc pour initialiser le tunnel automatiquement :

Bash

# === Pont SSH YubiKey ===
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
if ! ss -a | grep -q "$SSH_AUTH_SOCK"; then
  rm -f "$SSH_AUTH_SOCK"
  wsl2_ssh_pageant_bin="$HOME/.ssh/wsl2-ssh-pageant.exe"
  if test -x "$wsl2_ssh_pageant_bin"; then
    (setsid nohup socat UNIX-LISTEN:"$SSH_AUTH_SOCK,fork" EXEC:"$wsl2_ssh_pageant_bin" >/dev/null 2>&1 &)
  fi
fi

# === Pont GPG (Optionnel pour signature Git) ===
export GPG_AGENT_SOCK="$HOME/.gnupg/S.gpg-agent"
if ! ss -a | grep -q "$GPG_AGENT_SOCK"; then
  rm -rf "$GPG_AGENT_SOCK"
  wsl2_ssh_pageant_bin="$HOME/.ssh/wsl2-ssh-pageant.exe"
  if test -x "$wsl2_ssh_pageant_bin"; then
    (setsid nohup socat UNIX-LISTEN:"$GPG_AGENT_SOCK,fork" EXEC:"$wsl2_ssh_pageant_bin --gpg S.gpg-agent" >/dev/null 2>&1 &)
  fi
fi
Appliquez les changements : source ~/.bashrc.

5. Récupérer la clé publique SSH
Pour autoriser l'accès à un serveur, vous devez extraire la clé publique stockée dans la YubiKey.

Dans un terminal Windows (PowerShell), tapez :

PowerShell

gpg --export-ssh-key
Copiez la ligne (ex: ssh-rsa AAAA...) et ajoutez-la au fichier ~/.ssh/authorized_keys de votre serveur distant.

6. Utilisation (PuTTY & SSH)
PowerShell / OpenSSH : Tapez simplement ssh user@ip. La YubiKey clignotera pour confirmation.

WSL2 : Tapez ssh user@ip. Le tunnel socat transfère la requête à l'agent Windows de manière transparente.

PuTTY : Ne configurez aucune clé dans Connection > SSH > Auth. PuTTY détectera automatiquement l'agent via le support Pageant activé dans GPG.

7. Utilisation de plusieurs YubiKeys
Si vous possédez plusieurs YubiKeys (ex: une principale et une de secours) :

Détection automatique : GPG présente à l'agent SSH les clés de la YubiKey qui est actuellement branchée.

Changement à chaud : Si vous changez de clé, il est parfois nécessaire de forcer l'agent à "apprendre" la nouvelle carte :

PowerShell

gpg-connect-agent "scd serialno" "learn --force" /bye
Clés publiques : Notez que chaque YubiKey possède sa propre clé publique. Vous devez ajouter les clés publiques de vos deux clés sur vos serveurs distants.

8. Dépannage (Troubleshooting)
L'agent ne répond plus (Windows)
Si ssh-add -L ne renvoie rien alors que la clé est branchée :

Tuez les processus GPG : taskkill /F /IM gpg-agent.exe.

Nettoyez les sockets orphelins (fichiers S.*) dans :

%APPDATA%\gnupg\

%LOCALAPPDATA%\gnupg\

Relancez l'agent : gpg-connect-agent /bye.

Problème de tunnel WSL2
Si Windows voit la clé mais que WSL2 renvoie "Error connecting to agent" :

Forcez l'arrêt de WSL : wsl --shutdown (depuis PowerShell).

Vérifiez que le service Carte à puce (SCardSvr) est bien démarré dans les services Windows (services.msc).
