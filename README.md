# NAS Debian
## Serveur de fichiers multi-protocoles

Déploiement d'un serveur NAS sous Debian 13 (Trixie) sans interface graphique, avec gestion multi-utilisateurs, RAID 5 et accès sécurisé via SFTP, WebDAV, Samba et NFS.

---

## Contexte

L'objectif est de monter un serveur NAS complet sur une VM Debian headless (VMware Workstation), capable de gérer plusieurs utilisateurs simultanément avec des espaces privés, un dossier public partagé et un compte administrateur privilégié.

Le serveur utilise 4 disques virtuels : 1 pour le système et 3 pour le RAID 5.

## Ce qui a été mis en place

**Stockage**

Le RAID 5 est géré par mdadm sur trois disques (sdb, sdc, sdd), ce qui donne environ 20 Go d'espace utile avec une tolérance de panne d'un disque. Le volume est monté directement sur `/srv/nas` via `/etc/fstab`, avec sauvegarde de la configuration dans `mdadm.conf` et mise à jour de l'initramfs.

**Gestion des utilisateurs**

Deux groupes : `nasusers` pour les utilisateurs standards et `nasadmins` pour l'administrateur. Chaque utilisateur (alice, bob, claire) possède un dossier privé en `chmod 700`. Le dossier public est en `chmod 2775` (SGID) pour que les fichiers créés héritent du groupe `nasusers`. Un script `nas-adduser` automatise la création des comptes.

Les ACL (`setfacl`) sont utilisées pour donner à l'admin un accès en lecture sur les dossiers utilisateurs et à `www-data` les droits nécessaires pour Apache, sans modifier les permissions de base.

**Protocoles d'accès**

- SFTP via OpenSSH avec chroot sur `/srv/nas` — les utilisateurs du groupe `nasusers` sont enfermés dans l'arborescence du NAS et n'ont accès qu'au transfert de fichiers (pas de shell)
- WebDAV via Apache sur le port 443 (HTTPS) avec authentification Digest et certificat auto-signé
- Samba pour l'accès natif depuis Windows, avec des partages `[public]`, `[alice]`, `[bob]` configurés dans `smb.conf`
- NFS pour le partage entre machines Linux en réseau local, configuré dans `/etc/exports`

**Sécurité**

- UFW activé, seuls les ports 22, 443, 445, 2049 et 9090 sont ouverts
- Fail2ban configuré sur le jail SSH (maxretry=5, bantime=3600s)
- Durcissement SSH : `PermitRootLogin no`, `MaxAuthTries 3`, timeout 60s
- Mises à jour automatiques via `unattended-upgrades`

**Sauvegarde**

Script `backup-nas.sh` qui utilise rsync avec `--delete` pour maintenir un miroir de `/srv/nas` vers `/srv/backup/nas`. Journalisation dans `/var/log/nas-backup.log` et planification via cron (tous les jours à 2h).

**Interface d'administration**

Cockpit installé et accessible sur le port 9090. Permet de superviser le système (CPU, RAM, disques, réseau), gérer les utilisateurs, les services et le stockage depuis un navigateur.

**Virtualisation**

KVM/QEMU installé avec libvirt. Une VM Debian 12 a été créée via `virt-install` en mode console série (headless). La virtualisation imbriquée est limitée selon le matériel.

## Arborescence du NAS

```
/srv/nas/
    public/         # Partagé entre tous (2775, SGID)
    alice/          # Privé (700)
    bob/            # Privé (700)
    claire/         # Privé (700)
    admin/          # Réservé nasadmins (2770)
```

## Services et ports

| Service       | Port | Protocole |
|---------------|------|-----------|
| SSH / SFTP    | 22   | TCP       |
| Apache WebDAV | 443  | TCP       |
| Samba         | 445  | TCP       |
| NFS           | 2049 | TCP       |
| Cockpit       | 9090 | TCP       |

## Réseau

IP statique configurée dans `/etc/network/interfaces`. L'adresse dépend du réseau (maison vs école). En cas de changement de réseau, passer temporairement en DHCP puis reconfigurer.

## Commandes utiles

```bash
# Etat du RAID
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# Gestion des utilisateurs
sudo nas-adduser <nom>
getent group nasusers

# Vérifier les ACL
sudo getfacl /srv/nas/users/alice

# Services
sudo systemctl status sshd apache2 smbd nfs-kernel-server cockpit

# Sauvegarde manuelle
sudo /usr/local/bin/backup-nas.sh

# Logs
sudo journalctl -u sshd -n 20
sudo tail -f /var/log/apache2/error.log
```

## Tests réalisés

- Connexion SFTP et isolation entre utilisateurs (Permission denied confirmé)
- Upload/download via WebDAV en HTTPS depuis Windows (curl + navigateur)
- Accès Samba depuis l'explorateur Windows (`\\IP\public`)
- Montage NFS depuis un client Linux
- RAID 5 opérationnel avec les 3 disques actifs ([UUU])
- Sauvegarde rsync fonctionnelle avec journalisation
- Cockpit accessible et fonctionnel
- VM Debian 12 créée et démarrée via virsh

## Difficultés rencontrées

- Syntaxe `adduser` (Debian) vs `useradd` (Red Hat) : les options courtes ne sont pas les mêmes
- Les ACL ne sont pas copiées par rsync sans l'option `-A`
- Le chroot SSH est très strict sur les permissions : `/srv` et `/srv/nas` doivent appartenir à root en 755
- Le site Apache par défaut (`000-default.conf`) prenait la priorité sur la config WebDAV
- La virtualisation imbriquée (VM dans VM) n'est pas supportée sur tous les processeurs

## Auteur

Farouk, La Plateforme_
