# TP : Configuration et utilisation de VNC et noVNC

DGI-ESP 2026-2027, L3 GLSI, Services Réseaux Avancés, Dr GUEYE
Étudiants : Emmanuel Ndione et Cheikh Ahmed Tidiane Chérif Ndoye


## 1. Objectif et environnement

L'objectif du TP est de prendre le contrôle d'une machine Linux à distance avec VNC, de sécuriser l'accès avec un tunnel SSH, puis de l'ouvrir depuis un navigateur avec noVNC.

Le serveur est une machine Kali Linux (`kali`, IP 10.7.7.241) avec TigerVNC 1.16.2 et XFCE 4.20.1. Le client est un PC Windows (IP 10.7.5.216) avec RealVNC Viewer 8.6.2. noVNC est installé dans `/opt/noVNC` avec websockify.

Le sujet a été adapté sur trois points :
- la session VNC est `:2` (port 5902) au lieu de `:1` (port 5901), car `:1` est déjà occupé ;
- le fichier de démarrage `xstartup` se trouve dans `~/.config/tigervnc/` pour cette version de TigerVNC ;
- websockify est installé avec `apt` (`python3-websockify`) au lieu de `pip`.

## 2. Partie 1 : installation et configuration du serveur VNC

Mise à jour de la machine :

```bash
sudo apt update && sudo apt upgrade -y
```

![Mise à jour](images/partie1-01-mise-a-jour.png)

Installation de TigerVNC :

```bash
sudo apt install tigervnc-standalone-server -y
```

![Installation de TigerVNC](images/partie1-02-installation-tigervnc.png)

XFCE est déjà présent sur la machine. On installe quand même `xfce4` et `xfce4-goodies` :

```bash
which startxfce4
sudo apt install xfce4 xfce4-goodies -y
```

![XFCE déjà présent](images/partie1-03-xfce-deja-present.png)

![Installation de xfce4-goodies](images/partie1-04-installation-xfce4-goodies.png)

Le sujet demande de lancer `vncserver :1`, mais l'affichage `:1` est déjà utilisé :

```bash
vncserver :1
ls /tmp/.X11-unix/
```

![Affichage :1 occupé](images/partie1-05-display-1-occupe.png)

![Sockets X11 existants](images/partie1-06-sockets-x11-display-1.png)

On utilise donc l'affichage `:2`. Le serveur demande un mot de passe, puis écoute sur le port 5902.

```bash
vncserver :2
```

![Initialisation de vncserver :2](images/partie1-07-initialisation-vncserver-2.png)

On arrête ensuite le serveur pour configurer le démarrage :

```bash
vncserver -kill :2
```

![Arrêt de la session](images/partie1-08-arret-vncserver-2.png)

Création du fichier `xstartup` pour lancer XFCE :

```bash
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/bash
xrdb $HOME/.Xresources
startxfce4 &
EOF
```

![Création de xstartup](images/partie1-09-creation-xstartup.png)

![Contenu de xstartup](images/partie1-10-xstartup-nano.png)

On le rend exécutable et on vérifie :

```bash
chmod +x ~/.vnc/xstartup
ls -la && cat xstartup
```

![chmod +x](images/partie1-11-chmod-xstartup.png)

![Vérification des droits](images/partie1-12-verification-xstartup.png)

Relance du serveur :

```bash
vncserver :2
```

![Relance de vncserver :2](images/partie1-13-relance-vncserver-2.png)

Cette version de TigerVNC ne lit pas `~/.vnc/xstartup`, ce qui a causé un écran noir. La correction est expliquée dans la section 7.

## 3. Partie 2 : connexion depuis la machine cliente

Le client RealVNC Viewer est installé sur le PC Windows.

![Installation de RealVNC Viewer](images/partie2-01-installation-realvnc-viewer.png)

Le sujet propose une connexion directe sur `10.7.7.241:2`. Elle échoue : le port SSH (22) répond, mais le port VNC (5902) est injoignable depuis le PC, sans doute filtré par le réseau de l'établissement.

```powershell
Test-NetConnection 10.7.7.241 -Port 22
TcpTestSucceeded : True

Test-NetConnection 10.7.7.241 -Port 5902
TcpTestSucceeded : False
```

On passe donc par un tunnel SSH (partie suivante). Le test de connexion demandé par le sujet est présenté dans la partie 4, car il est fait à travers ce tunnel.

## 4. Partie 3 : sécurisation avec un tunnel SSH

Sur le PC Windows, on ouvre le tunnel et on laisse la fenêtre ouverte :

```powershell
ssh -L 5902:localhost:5902 emmanuel@10.7.7.241
```

![Tunnel SSH](images/partie3-01-tunnel-ssh.png)

Dans RealVNC Viewer, on se connecte à `localhost:5902`. Le client affiche un avertissement « connexion non chiffrée » : c'est normal, car RealVNC ne voit qu'une connexion locale. Le chiffrement est assuré par le tunnel SSH.

![Avertissement de chiffrement](images/partie3-02-avertissement-connexion-non-chiffree.png)

![Mot de passe VNC](images/partie3-03-authentification-vnc.png)

Le bureau XFCE de Kali s'affiche. La fenêtre d'authentification visible est une demande normale de XFCE (gestion des couleurs), sans lien avec VNC.

![Bureau XFCE dans RealVNC](images/partie3-04-bureau-xfce-via-tunnel.png)

Pour prouver qu'on est bien sur le serveur, on lance `hostname` et `ip a` dans le bureau distant. Le nom est `kali` et l'adresse est 10.7.7.241.

![hostname et ip a](images/partie3-05-hostname-et-ip.png)

Pour vérifier le passage par SSH, on consulte les journaux du serveur (Kali utilise `journald`) :

```bash
sudo journalctl -u ssh -n 15 --no-pager
```

La ligne `Accepted password for emmanuel from 10.7.5.216` prouve que la connexion vient du PC Windows par SSH.

![Journaux SSH](images/partie3-06-journaux-ssh.png)

## 5. TP noVNC

### Installation

Le serveur doit avoir accès à internet, ce qui demande une authentification au portail de l'école (voir section 7). On installe ensuite git et websockify :

```bash
sudo apt install git python3-websockify -y
```

![Installation de git et websockify](images/novnc-01-installation-git-websockify.png)

On clone noVNC dans `/opt` :

```bash
cd /opt
sudo git clone https://github.com/novnc/noVNC.git
```

![Clonage de noVNC](images/novnc-02-clonage-depot.png)

![Contenu de /opt/noVNC](images/novnc-03-contenu-repertoire.png)

La session VNC et le fichier `xstartup` sont ceux de la partie 1 (session `:2`).

### Lancement de noVNC

On lance le proxy en le pointant vers le port 5902 :

```bash
cd /opt/noVNC
./utils/novnc_proxy --vnc localhost:5902
```

Il écoute sur le port 6080 et relaie vers `localhost:5902`.

![Lancement du proxy noVNC](images/novnc-04-lancement-proxy.png)

### Accès depuis le navigateur

Comme pour VNC, on passe par un tunnel SSH, cette fois sur le port 6080 :

```powershell
ssh -L 6080:localhost:6080 emmanuel@10.7.7.241
Test-NetConnection localhost -Port 6080
```

![Test du tunnel sur le port 6080](images/novnc-05-test-tunnel-6080.png)

On ouvre ensuite `http://localhost:6080/vnc.html`, puis on clique sur Connecter et on entre le mot de passe VNC.

![Page noVNC](images/novnc-06-page-novnc-navigateur.png)

### Démarrage automatique

Deux services `systemd` démarrent VNC et noVNC avec la machine. Le service VNC est `/etc/systemd/system/vncserver.service` :

```ini
[Unit]
Description=TigerVNC session :2
After=network.target

[Service]
Type=simple
User=emmanuel
ExecStartPre=-/usr/bin/tigervncserver -kill :2
ExecStart=/usr/bin/tigervncserver :2 -localhost no -fg
ExecStop=/usr/bin/tigervncserver -kill :2
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

![Création de vncserver.service](images/novnc-07-service-vncserver.png)

Le service noVNC est `/etc/systemd/system/novnc.service`. Voici sa version finale. La capture montre la première version, sans les lignes `ExecStartPre`, `RestartSec` et `StartLimitIntervalSec` ajoutées ensuite (voir section 7).

```ini
[Unit]
Description=noVNC proxy
After=vncserver.service
Requires=vncserver.service
StartLimitIntervalSec=0

[Service]
Type=simple
User=emmanuel
ExecStartPre=/bin/bash -c 'until ss -tln | grep -q ":5902 "; do sleep 1; done'
ExecStart=/opt/noVNC/utils/novnc_proxy --vnc localhost:5902
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

![Création de novnc.service](images/novnc-08-service-novnc.png)

On active et on lance les deux services :

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now vncserver.service novnc.service
```

![daemon-reload](images/novnc-09-daemon-reload.png)

![enable --now](images/novnc-10-enable-now.png)

![Statut de vncserver.service](images/novnc-11-statut-vncserver.png)

Le code d'erreur de `ExecStartPre` est sans conséquence : cette commande tente d'arrêter une session qui n'existe pas encore, et le `-` devant fait ignorer l'erreur.

Après correction de noVNC (section 7), les deux services sont actifs :

![Services actifs](images/novnc-12-services-actifs.png)

On redémarre ensuite la machine pour tester le démarrage automatique :

```bash
sudo reboot
```

![Redémarrage](images/novnc-13-redemarrage.png)

Après le redémarrage et la réouverture du tunnel, la page noVNC est de nouveau servie sans aucune commande manuelle.

![Page noVNC après redémarrage](images/novnc-14-page-novnc-apres-redemarrage.png)

## 6. Questions de réflexion

Utilité de websockify : un navigateur ne sait pas parler le protocole VNC (RFB, en TCP brut), il ne connaît que HTTP et WebSocket. Websockify fait la traduction : il reçoit les connexions WebSocket du navigateur et les relaie en TCP vers le serveur VNC. Il sert aussi les fichiers web de noVNC sur le port 6080.

Différences entre noVNC et les clients VNC standards : noVNC est un client JavaScript qui tourne dans le navigateur, sans installation et sur tout appareil. Un client standard (RealVNC, TigerVNC Viewer) est une application à installer, qui se connecte directement au port VNC sans intermédiaire.

Avantages de noVNC : aucune installation côté client, fonctionne partout où il y a un navigateur, simple à utiliser. Inconvénients : un peu moins performant, moins de fonctionnalités (transfert de fichiers, audio), un composant de plus à maintenir (websockify), et un port 6080 en HTTP non chiffré qui impose un tunnel SSH ou HTTPS. Une application native est plus complète et plus rapide, mais demande une installation.

## 7. Problèmes rencontrés et solutions

Affichage :1 occupé. `vncserver :1` est refusé (voir les captures de la partie 1). On utilise `:2` et donc le port 5902.

Session fermée puis écran noir. Le journal indique que la session se termine trop tôt. La version de TigerVNC lit `~/.config/tigervnc/xstartup` et non `~/.vnc/xstartup`, et XFCE tentait de démarrer en Wayland alors que VNC utilise X11.

![Session qui quitte trop tôt](images/probleme-01-session-quitte-trop-tot.png)

La solution est de créer `~/.config/tigervnc/xstartup` en forçant X11, puis de relancer la session :

```bash
#!/bin/bash
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
unset WAYLAND_DISPLAY
export XDG_SESSION_TYPE=x11
export GDK_BACKEND=x11
xrdb $HOME/.Xresources
exec startxfce4
```

```bash
chmod +x ~/.config/tigervnc/xstartup
vncserver -kill :2
tigervncserver :2 -localhost no
```

Port 5902 injoignable depuis le PC. La solution est le tunnel SSH et la connexion sur `localhost:5902` (partie 3).

Portail captif. Les téléchargements (`apt`, `git`) échouent, et `curl` est redirigé vers le portail étudiant. L'authentification faite sur le PC ne vaut pas pour le serveur : il faut se connecter au portail depuis le navigateur du serveur.

![Redirection vers le portail](images/probleme-02-portail-captif-redirection.png)

![Portail connecté](images/probleme-03-portail-connecte.png)

`novnc.service` en échec. Le service boucle en redémarrage et le journal indique `Port 6080 in use` : l'ancien proxy lancé à la main occupait encore le port. De plus, au premier démarrage, le service se lançait avant que le port 5902 soit prêt.

![Port 6080 occupé](images/probleme-04-port-6080-occupe.png)

La solution est d'arrêter l'ancien processus, puis de relancer le service avec la version finale du fichier (qui attend le port 5902) :

```bash
sudo kill 46483
sudo systemctl reset-failed novnc.service
sudo systemctl restart novnc.service
```

## 8. Conclusion

Ce TP a permis de mettre en place un serveur TigerVNC avec XFCE, une connexion avec RealVNC sécurisée par un tunnel SSH, un accès depuis un navigateur avec noVNC et websockify, et le démarrage automatique des services avec `systemd`. Les difficultés rencontrées (affichage occupé, emplacement du fichier `xstartup`, Wayland, portail captif, ordre de démarrage des services) ont permis de mieux comprendre VNC, `systemd` et la sécurisation d'un accès distant.
