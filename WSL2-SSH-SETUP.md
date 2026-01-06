📑 Configuration SSH avec YubiKey sous Windows 11 & WSL2
Ce guide permet de configurer une YubiKey 5 NFC comme agent SSH unique pour Windows 11 (PowerShell/OpenSSH), PuTTY, et WSL2 (Kali/Ubuntu).

📋 Table des matières
Installation des prérequis

Configuration de l'Agent GPG

Démarrage automatique (Windows)

Pont SSH pour WSL2 (Linux)

Récupérer la clé publique SSH

Utilisation (PuTTY & SSH)

Utilisation de plusieurs YubiKeys

Dépannage (Troubleshooting)

1. Installation des prérequis
Windows
GPG4Win : Télécharger ici. Installez au minimum le composant GnuPG.

wsl2-ssh-pageant : Télécharger le binaire .exe.

Placez le fichier dans un dossier stable de votre profil (ex: C:\Users\<VOTRE_NOM>\wsl2-ssh-pageant.exe).

WSL2 (Kali/Ubuntu/...)
Installez socat pour permettre la communication entre Linux et le pont Windows : sudo apt update && sudo apt install socat -y

2. Configuration de l'Agent GPG
Éditez le fichier %APPDATA%\gnupg\gpg-agent.conf.

Contenu (GnuPG 2.5+) : enable-ssh-support enable-putty-support enable-win32-openssh-support default-cache-ttl 600 max-cache-ttl 7200

Note : Ne pas ajouter use-standard-socket (obsolète sous GPG 2.5+).

3. Démarrage automatique (Windows)
Faites Win + R, tapez shell:startup.

Créez un raccourci vers la commande : gpg-connect-agent /bye.

Nommez-le : GPG Agent Startup.

4. Pont SSH pour WSL2 (Linux)
Lien symbolique
Dans votre terminal WSL2 (ajustez <VOTRE_NOM>) : mkdir -p ~/.ssh ln -sf /mnt/c/Users/<VOTRE_NOM>/wsl2-ssh-pageant.exe ~/.ssh/wsl2-ssh-pageant.exe

Configuration du .bashrc
Ajoutez ce bloc à la fin de votre fichier ~/.bashrc :

export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock" if ! ss -a | grep -q "$SSH_AUTH_SOCK"; then rm -f "$SSH_AUTH_SOCK" wsl2_ssh_pageant_bin="$HOME/.ssh/wsl2-ssh-pageant.exe" if test -x "$wsl2_ssh_pageant_bin"; then (setsid nohup socat UNIX-LISTEN:"$SSH_AUTH_SOCK,fork" EXEC:"$wsl2_ssh_pageant_bin" >/dev/null 2>&1 &) fi fi

5. Récupérer la clé publique SSH
Dans un terminal PowerShell : gpg --export-ssh-key

Copiez la ligne générée et ajoutez-la au fichier ~/.ssh/authorized_keys de votre serveur.

6. Utilisation (PuTTY & SSH)
PowerShell / WSL2 : ssh user@ip. La YubiKey clignotera pour confirmation.

PuTTY : Détection automatique via l'émulation Pageant (aucune clé à configurer dans les menus).

7. Utilisation de plusieurs YubiKeys
Détection : GPG expose la clé de la YubiKey branchée.

Changement : Si vous changez de clé, forcez la relecture : gpg-connect-agent "scd serialno" "learn --force" /bye

Clés publiques : Chaque clé physique a sa propre empreinte SSH. Exportez chaque clé individuellement.

8. Dépannage (Troubleshooting)
Reset de l'agent (Windows)
En cas de blocage, exécutez dans PowerShell : taskkill /F /IM gpg-agent.exe Remove-Item -Path "$env:LOCALAPPDATA\gnupg\S.*" -Force gpg-connect-agent /bye

Reset WSL2
Si la connexion échoue dans Linux, relancez le sous-système : wsl --shutdown.
