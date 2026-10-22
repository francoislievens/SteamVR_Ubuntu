# SteamVR + ALVR sur Ubuntu avec Meta Quest 1


Guide reproductible pour utiliser un Meta Quest 1 comme casque PCVR sous Ubuntu avec SteamVR et ALVR.

> Contexte testé :
> - Ubuntu Linux
> - GPU NVIDIA RTX 4090
> - Meta Quest 1 en mode développeur
> - Steam installé nativement en `.deb`
> - SteamVR
> - ALVR 20.14.1
>
> L’objectif est d’obtenir une chaîne fonctionnelle :
>
> **Quest 1 → ALVR client → réseau/USB → ALVR streamer → SteamVR → applications VR**

---

## 1. Ne pas utiliser Steam en Snap

SteamVR fonctionne mal avec Steam installé depuis le Snap Store Ubuntu.

Vérifier :

```bash
snap list | grep steam
which steam
```

Si Steam apparaît comme Snap ou si `which steam` retourne :

```text
/snap/bin/steam
```

le supprimer.

### Suppression propre

Fermer complètement Steam et SteamVR :

```bash
pkill -f steam
pkill -f vrserver
pkill -f vrmonitor
pkill -f vrcompositor
```

Puis :

```bash
sudo snap remove steam --purge
```

Si Snap signale qu'une opération de suppression est déjà en cours :

```bash
snap changes
```

Repérer l'identifiant correspondant à la suppression de Steam, puis :

```bash
sudo snap abort <ID>
sudo systemctl restart snapd
sudo snap remove steam --purge
```

Vérifier ensuite :

```bash
snap list | grep steam
```

Cette commande ne doit plus afficher Steam.

---

## 2. Installer Steam nativement avec le `.deb`

Télécharger le paquet officiel :

```bash
cd ~/Téléchargements
wget https://cdn.cloudflare.steamstatic.com/client/installer/steam.deb
```

Installer :

```bash
sudo apt install ./steam.deb
```

Lancer Steam :

```bash
steam
```

Vérifier :

```bash
which steam
```

Résultat attendu :

```text
/usr/bin/steam
```

---

## 3. Installer SteamVR

Dans Steam :

1. Ouvrir la **Bibliothèque**
2. Rechercher **SteamVR**
3. Installer SteamVR
4. Le lancer une première fois

Il est normal qu'il affiche au début quelque chose comme :

```text
Please plug in your VR headset
```

### Important : branche SteamVR `previous`

Avec les versions récentes de SteamVR Linux, un crash du compositor peut apparaître avec ALVR.

Symptôme typique :

```text
SteamVR has encountered a critical error
```

Dans notre cas, la solution a été de revenir à la branche précédente de SteamVR.

Dans Steam :

1. Clic droit sur **SteamVR**
2. **Propriétés**
3. **Bêtas**
4. Sélectionner la branche :

```text
previous
```

Laisser Steam télécharger cette version.

---

## 4. Installer ADB

ADB est nécessaire pour installer le client ALVR dans le Quest.

```bash
sudo apt update
sudo apt install adb
```

Vérifier :

```bash
adb version
```

Puis brancher le Quest en USB :

```bash
adb devices
```

Résultat attendu :

```text
List of devices attached
XXXXXXXXXXXX    device
```

Si le statut est :

```text
unauthorized
```

mettre le casque et accepter la demande :

```text
Allow USB debugging
```

Le Quest doit être en **Developer Mode**.

---

## 5. Installer ALVR Launcher

Télécharger la version Linux du **ALVR Launcher** depuis les releases officielles ALVR.

Le fichier ressemble à :

```text
alvr_launcher_linux.tar.gz
```

Extraire :

```bash
cd ~/Téléchargements
tar -xzf alvr_launcher_linux.tar.gz
```

Puis lancer :

```bash
cd ~/Téléchargements/ALVR
./ALVR\ Launcher
```

Dans ALVR Launcher :

1. **Add version**
2. Choisir une version stable
3. **Install**
4. Ouvrir la version installée

Dans notre cas :

```text
v20.14.1
```

a été installée dans :

```text
~/.local/share/ALVR-Launcher/installations/v20.14.1/
```

---

## 6. Installer le client ALVR sur le Quest

Le Launcher peut parfois afficher :

```text
Failed to list ADB devices
Caused by:
No such file or directory (os error 2)
```

même si :

```bash
adb devices
```

fonctionne correctement.

Ce problème n'est pas bloquant.

L'APK ALVR est déjà téléchargé dans :

```text
~/.local/share/ALVR-Launcher/installations/v20.14.1/alvr_client_android.apk
```

Installer manuellement :

```bash
adb install -r ~/.local/share/ALVR-Launcher/installations/v20.14.1/alvr_client_android.apk
```

Résultat attendu :

```text
Success
```

Dans le Quest :

1. Ouvrir la bibliothèque
2. Aller dans **Unknown Sources / Sources inconnues**
3. Lancer **ALVR**

---

## 7. Configurer le firewall ALVR

ALVR peut afficher :

```text
Setting firewall rules failed! code: 127
```

Sous Ubuntu avec locale française, le script ALVR peut mal interpréter :

```bash
sudo ufw status
```

qui retourne par exemple :

```text
État : inactif
```

alors que le script cherche explicitement :

```text
Status: inactive
```

Le script ALVR est ici :

```text
~/.local/share/ALVR-Launcher/installations/v20.14.1/alvr_streamer_linux/libexec/alvr/alvr_fw_config.sh
```

Lancer manuellement :

```bash
sudo ~/.local/share/ALVR-Launcher/installations/v20.14.1/alvr_streamer_linux/libexec/alvr/alvr_fw_config.sh add
```

Les ports ALVR utilisés sont :

```text
9943-9944 TCP
9943-9944 UDP
```

Pour vérifier UFW :

```bash
sudo ufw status
```

Dans notre cas UFW était inactif, donc il ne bloquait rien.

---

## 8. Vérifier le runtime OpenVR

SteamVR crée normalement :

```text
~/.config/openvr/openvrpaths.vrpath
```

Afficher le fichier :

```bash
cat ~/.config/openvr/openvrpaths.vrpath
```

Il doit contenir un runtime SteamVR similaire à :

```json
"runtime" :
[
    "/home/<USER>/.local/share/Steam/steamapps/common/SteamVR"
]
```

et ALVR comme driver externe :

```json
"external_drivers" :
[
    "/home/<USER>/.local/share/ALVR-Launcher/installations/v20.14.1/alvr_streamer_linux/lib64/alvr"
]
```

---

## 9. Cas NVIDIA + iGPU AMD

Sur une machine possédant à la fois un iGPU AMD et une carte NVIDIA, il est préférable de forcer SteamVR à utiliser NVIDIA.

Vérifier les GPU :

```bash
lspci | grep -Ei 'VGA|3D|Display'
```

Vérifier NVIDIA :

```bash
nvidia-smi
```

Vérifier les ICD Vulkan :

```bash
ls /usr/share/vulkan/icd.d/
```

Les fichiers utiles doivent notamment inclure :

```text
nvidia_icd.json
radeon_icd.json
```

Dans Steam → SteamVR → **Propriétés → Options de lancement**, utiliser :

```bash
WAYLAND_DISPLAY='' __NV_PRIME_RENDER_OFFLOAD=1 __VK_LAYER_NV_optimus=NVIDIA_only __GLX_VENDOR_LIBRARY_NAME=nvidia VK_DRIVER_FILES=/usr/share/vulkan/icd.d/nvidia_icd.json ~/.local/share/Steam/steamapps/common/SteamVR/bin/vrmonitor.sh %command%
```

Cette ligne :

- force l'utilisation de la carte NVIDIA ;
- force Vulkan sur `nvidia_icd.json` ;
- force SteamVR à passer par XWayland plutôt que Wayland natif ;
- démarre SteamVR via `vrmonitor.sh`, nécessaire avec ALVR sous Linux.

> `%command%` est interprété par Steam.  
> Ne pas lancer cette ligne directement dans un terminal.

---

## 10. Démarrage normal de la chaîne VR

Ordre recommandé :

### Côté PC

1. Lancer Steam
2. Lancer ALVR Launcher / Dashboard
3. Lancer SteamVR

### Côté Quest

1. Lancer le Quest
2. Ouvrir **ALVR** depuis les sources inconnues
3. Attendre la connexion au PC

Le casque doit apparaître dans ALVR Dashboard.

S'il apparaît comme nouveau périphérique ou non approuvé :

```text
Trust
```

le client.

Lorsque la connexion fonctionne, SteamVR doit reconnaître un casque du type :

```text
alvr_server...
```

---

## 11. Diagnostic SteamVR

### Log principal

```bash
tail -150 ~/.local/share/Steam/logs/vrserver.txt
```

### Log compositor

```bash
tail -150 ~/.local/share/Steam/logs/vrcompositor.txt
```

Un démarrage correct doit finir par quelque chose comme :

```text
GPU Vendor: "NVIDIA GeForce RTX ..."
Direct mode: enabled
Using Vulkan Renderer
Compositor render thread started
Startup Complete
```

Dans notre cas, SteamVR récent plantait après le démarrage du compositor.

Le passage à la branche SteamVR :

```text
previous
```

a corrigé le problème.

---

## 12. Google Earth VR sous Ubuntu

Google Earth VR est une application Windows.

Il faut donc utiliser Proton.

Dans Steam :

1. Clic droit sur **Google Earth VR**
2. **Propriétés**
3. **Compatibilité**
4. Forcer une version Proton

Versions à tester :

```text
Proton 9.0-4
Proton 8.0-5
Proton 7.0-6
```

Éviter de commencer directement par Proton Experimental si OpenVR pose problème.

### Activer le log Proton

Dans les options de lancement Google Earth VR :

```bash
PROTON_LOG=1 %command%
```

Après un lancement :

```bash
tail -150 ~/steam-348250.log
```

Google Earth VR a l'AppID Steam :

```text
348250
```

---

## 13. Erreur de permissions Proton

Si le log contient :

```text
pfx is not owned by you
```

le prefix Proton appartient au mauvais utilisateur.

Exemple :

```bash
sudo chown -R <USER>:<USER> ~/.local/share/Steam/steamapps/compatdata/348250
```

Ne jamais lancer Steam avec :

```bash
sudo steam
```

---

## 14. Erreur OpenVR `InterfaceNotFound`

Google Earth VR peut afficher :

```text
OpenVR failed to initialize

VRInitError_Init_InterfaceNotFound
```

Dans ce cas vérifier d'abord :

```bash
cat ~/.config/openvr/openvrpaths.vrpath
```

Puis essayer dans les options de lancement Google Earth VR :

```bash
VR_OVERRIDE=/home/<USER>/.local/share/Steam/steamapps/common/SteamVR XR_RUNTIME_JSON=/home/<USER>/.local/share/Steam/steamapps/common/SteamVR/steamxr_linux64.json PROTON_LOG=1 %command%
```

Pour analyser OpenVR :

```bash
grep -Ei 'openvr|vrclient|interface|VRInit|steamvr' ~/steam-348250.log | tail -100
```

Dans notre cas, Google Earth VR chargeait :

```text
EarthVR/openvr_api.dll
C:\vrclient\bin\vrclient_x64.dll
```

ce qui confirme que l'échec se situe au niveau du bridge OpenVR entre Proton et SteamVR.

---

# Résumé rapide

La configuration qui s'est révélée la plus stable :

```text
Steam natif .deb
        ↓
SteamVR branche "previous"
        ↓
ALVR 20.14.1
        ↓
Quest 1 en mode développeur
        ↓
ALVR client installé via adb
        ↓
SteamVR lancé via vrmonitor.sh
        ↓
RTX NVIDIA forcée pour Vulkan
```

Pour une machine NVIDIA hybride AMD/NVIDIA, options SteamVR :

```bash
WAYLAND_DISPLAY='' __NV_PRIME_RENDER_OFFLOAD=1 __VK_LAYER_NV_optimus=NVIDIA_only __GLX_VENDOR_LIBRARY_NAME=nvidia VK_DRIVER_FILES=/usr/share/vulkan/icd.d/nvidia_icd.json ~/.local/share/Steam/steamapps/common/SteamVR/bin/vrmonitor.sh %command%
```

Installation ALVR Quest :

```bash
adb install -r ~/.local/share/ALVR-Launcher/installations/v20.14.1/alvr_client_android.apk
```

Firewall ALVR :

```bash
sudo ~/.local/share/ALVR-Launcher/installations/v20.14.1/alvr_streamer_linux/libexec/alvr/alvr_fw_config.sh add
```

Logs SteamVR :

```bash
tail -150 ~/.local/share/Steam/logs/vrserver.txt
tail -150 ~/.local/share/Steam/logs/vrcompositor.txt
```

Logs Google Earth VR :

```bash
PROTON_LOG=1 %command%
tail -150 ~/steam-348250.log
```
