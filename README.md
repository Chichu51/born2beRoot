*This activity has been created as part of the 42 curriculum by sperrin.*

## Description

Born2beRoot est un exercice d'administration système : mettre en place
une machine virtuelle sous VirtualBox avec une configuration durcie.
L'objectif est d'installer un système minimal, de chiffrer le stockage
avec LVM + LUKS, et d'appliquer des règles de sécurité strictes (SSH sur
un port personnalisé, pare-feu, politique de mots de passe, durcissement
de sudo), ainsi qu'un script de monitoring.

### Choix du système d'exploitation mais pas que
J'ai choisi Debian (stable), la distribution recommandée pour débuter en
administration système, grâce à sa documentation très complète.

- **Debian vs Rocky** : Debian s'appuie sur apt, AppArmor et UFW, plus
  accessibles pour un premier serveur ; Rocky utilise dnf, SELinux et
  firewalld, plus proches de l'environnement entreprise (RHEL) mais plus
  exigeants à configurer.
- **AppArmor vs SELinux** : AppArmor confine les programmes via des
  profils basés sur les chemins (plus simple à mettre en œuvre) ; SELinux
  est plus granulaire (étiquettes sur chaque objet) mais plus difficile à
  régler.
- **UFW vs firewalld** : UFW est une interface simple au-dessus d'iptables ;
  firewalld est dynamique et fonctionne par zones.
- **VirtualBox vs UTM** : VirtualBox est la référence pour ce projet ; UTM
  sert de solution de repli sur les machines à puce Apple Silicon.
- **apt vs aptitude** : les deux gèrent les paquets Debian. apt est le
  gestionnaire en ligne de commande standard ; aptitude est une interface
  plus riche (résolution de dépendances interactive, mode curses).
- **AppArmor** : module de sécurité du noyau Linux qui restreint les
  capacités de chaque programme via des profils (accès fichiers, réseau…),
  pour limiter les dégâts en cas de compromission.

## Instructions

### Vérifier la signature

A faire avant l'ouverture de la VM

1. La commande pour trouver le hash de la vm:
Linux: sha1sum cheminvers/sperrin42.vdi

### Mise en route

1. importer la machine virtuelle dans VirtualBox.
   machine -> open -> selection de la boite bleue
2. Snapshot: dans VirtualBox cliquer sur la VM
  Snapshot->Take->"donner un nom"
3. La démarrer (aucune interface graphique — serveur uniquement,
   `systemctl get-default` renvoie `multi-user.target` car pas d'interface graphique).
   `sudo systemctl status apparmor` ou `sudo aa-status` verifie que apparmor est bien actif.
4. Se connecter en SSH : `ssh -p 4242 <login>@<ip>`. pour voir l ip : ip addr
dans virtualbox setting -> network -> expert -> port forwarding : ssh | TCP | ip de la vm (ip addr) | 4242 | "" | 4242

### Partitionnement chiffré (LVM + LUKS)
Deux partitions chiffrées minimum. Le chiffrement est fait avec LUKS
le lvm est une couche de gestion flexible des disques. permet de creer des volumes logiques redimentionnables. 
Agrandir/reduire une partition sans formater
(`cryptsetup`), LVM par-dessus pour la flexibilité (redimensionner,
ajouter des volumes logiques sans re-partitionner le disque).
Vérification :
- `lsblk -f` → voir les volumes `crypt` puis les volumes logiques LVM.
- `sudo pvs` / `sudo vgs` / `sudo lvs` → Physical / Volume / Logical Volumes.
- `cat /etc/crypttab` → mapping des volumes chiffrés.

### Hostname
Le hostname est `sperrin42`. Il se définit dans `/etc/hostname` (et l'association IP dans `/etc/hosts`), puis `hostnamectl set-hostname sperrin42` et redémarrage pour prise en compte.
voir les info: hostnamectl

### SSH (port 4242, root interdit)
Configuré dans `/etc/ssh/sshd_config` :
- `Port 4242` → le service n'écoute que sur ce port.
- `PermitRootLogin no` → connexion root par SSH impossible.
Vérification : `sudo systemctl status ssh` et `sudo ss -tlnp | grep 4242`.
SSH = protocole chiffré pour administrer une machine à distance ; il remplace Telnet/rlogin qui passaient en clair.
-`sudo ss -tlnp | grep sshd` pour verifier si seul 4242 est autorise.
### Pare-feu (UFW)
- `sudo ufw status` → vérifier qu'il est actif au démarrage.
- `sudo ufw allow 8080/tcp` puis `sudo ufw delete allow 8080/tcp` →
  ajouter/retirer une règle à la volée.
UFW filtre les connexions entrantes : seul le strict nécessaire est ouvert.

### Politique de mots de passe
Deux fichiers modifiés : `/etc/login.defs` (durée de vie) et
`/etc/security/pwquality.conf` (complexité).
- `PASS_MAX_DAYS 30`, `PASS_MIN_DAYS 2`, `PASS_WARN_AGE 7` dans login.defs.
- `minlen=10`, `ucredit=-1`, `lcredit=-1`, `dcredit=-1`, `maxrepeat=3`,
  `usercheck=0` dans pwquality.conf.
- `chage -m 2 -M 30 -W 7 <login>` ajuste directement les bornes d'un
  compte (`-m` = jours min entre deux changements, `-M` = expiration,
  `-W` = préavis). `chage -l <login>` pour vérifier.
Avantage : force des mots de passe robustes et renouvelés, limite le
brute-force et le vol de compte. Inconvénient : risque que l'utilisateur
choisisse des mots de passe faibles mais « conformes », ou les note.

### Utilisateurs et groupes
- `adduser <login>` puis `usermod -aG user42,sudo <login>` → le compte
  principal appartient à `user42` et `sudo`.
- `id <login>` / `getent group user42` → vérifier l'appartenance.
- En review, on me demandera de créer un groupe et d'y mettre un
  utilisateur : `groupadd evaluating`, `usermod -aG evaluating <user>`,
  puis `id <user>` pour confirmer.

### Sudo durci
Configuré via `visudo` (dans `/etc/sudoers` ou un drop-in de
`/etc/sudoers.d/`) :
- `Defaults passwd_tries=3` → 3 tentatives max.
- `Defaults badpass_message="..."` → message personnalisé en cas d'erreur.
- `Defaults log_input, log_output, logfile="/var/log/sudo/log"` →
  journalisation des entrées/sorties dans `/var/log/sudo/`.
- `Defaults requiretty` → sudo uniquement depuis un vrai terminal (TTY).
- `Defaults secure_path="..."` → chemins d'exécution restreints.
Vérification : `sudo visudo -c` (syntaxe), `ls -la /var/log/sudo/` et
`sudo cat /var/log/sudo/log` (l'historique des commandes sudo s'y trouve).
sudo = exécuter une commande avec les droits root après authentification,
sans partager le mot de passe root et avec traçabilité.

### Script de monitoring
`monitoring.sh` (bash) affiche toutes les infos demandées (arch/kernel,
CPU physiques/virtuels, RAM, disque, charge CPU, dernier boot, LVM,
connexions TCP, utilisateurs, IPv4 + MAC, nb de commandes sudo) sur tous
les terminaux via `wall`.
- Planification : une ligne `*/10 * * * *` dans `crontab -l` (cron =
  ordonnanceur qui lance une tâche à intervalle régulier).
- Pour le passer à la minute : `* * * * *`.
- Pour l'arrêter **sans modifier le script** : `crontab -e` et commenter
  ou supprimer la ligne (le fichier `monitoring.sh` reste intact, mêmes
  droits, même emplacement).

### Soumission
- `README.md` et `signature.txt` à la racine du dépôt.
- `signature.txt` contient le SHA-1 du fichier-disque de la VM
  (`sha1sum <disque>.vdi`), calculé **VM éteinte, avant tout redémarrage**.

## Ressources

- Guide d'administration Debian, manuel VirtualBox, pages de manuel
  (`sshd_config(5)`, `sudoers(5)`, `lvm(8)`, `cryptsetup(8)`).
- Concepts réseau utilisés dans le projet : adressage TCP/IP, masque de
  sous-réseau, passerelle par défaut, routeurs et commutateurs, couches
  du modèle OSI.
- et un paquet de sites internet dont je n'ai absolument pas noté la
  source bien sûr :/

### Utilisation de l'IA
L'IA a servi à comprendre certains concepts, rédiger ce README, et
retrouver des commandes que je ré-oubliais (le `chage -m 2` par exemple une bonne dizaine de fois, je pense). Aucun code ni aucune configuration n'a été généré par l'IA : chaque fichier a été écrit et compris par moi. (À part le README, écrit en grande partie avec l'aide de
l'IA puis relu et réécrit par moi — là par exemple, ou encore là, et enfin là.)