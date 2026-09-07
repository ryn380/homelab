
## 2026-08-20 - Prise en main du serveur et installation de Proxmox

**Durée :** Début : 15h - Fin : 19h

### Fait

- Première connexion SSH en root, vérification de l'empreinte de clé du serveur.
- État des lieux : Debian 12, noyau 6.1, Ryzen 5 3600 (12 threads), 62 Gio de RAM, 2 x 476 Gio NVMe.
	- Disques en mdadm RAID 1 : `md0` swap 32 Gio, `md1` /boot 1 Gio, `md2` racine 443 Gio, ext4. Les trois grappes en `[2/2] [UU]`.
	- Services en écoute avant intervention : SSH uniquement (IPv4 + IPv6).
- Durcissement SSH : `PasswordAuthentication no` et `PermitRootLogin prohibit-password` posés dans `/etc/ssh/sshd_config.d/10-hardening.conf`.
- Ajout de la clé de signature et du dépôt `pve-no-subscription`, empreinte de clé comparée à celle du wiki Proxmox.
- Installation du noyau `proxmox-default-kernel` seul, puis redémarrage. Démarrage sur `6.8.12-43-pve` OK.
- Nom d'hôte passé de `Debian-bookworm-latest-amd64-base` à `pve1`, FQDN `pve1.lab.internal`, `/etc/hosts` aligné sur l'IP publique.
- `LISTEN_IP=127.0.0.1` posé dans `/etc/default/pveproxy` **avant** l'installation.
- Installation de `proxmox-ve`, `postfix` (mode local uniquement), `open-iscsi`, `chrony`.
- Retrait des noyaux Debian et de `os-prober`, régénération de GRUB.
- Accès à l'interface web par tunnel SSH (`ssh -L 8006:127.0.0.1:8006`), nœud visible sous `pve1`.

### Décisions et raisons

| Décision                                                              | Raison                                                                                                                          | Contrepartie                                                                                        |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Serveur dédié plutôt que matériel local				| Pour une disponibilité 24/7 indépendante de mon domicile, débit à 1 Gbit/s symétrique						  | Pas d'accès physique, pas de matériel réseau réel, machine exposée sur Internet public		|
| Proxmox installé sur le Debian existant plutôt que réinstallation ZFS | Comprendre ce que Proxmox ajoute à un Debian ; pas de passage par le rescue sur une machine encore inconnue                     | mdadm n'est pas un stockage officiellement pris en charge ; pas d'instantanés ni de réplication ZFS |
| `pveproxy` restreint à la boucle locale                               | Le paquet démarre le service dès la fin de l'installation ; sur une IP publique la fenêtre d'exposition est de quelques minutes | Accès uniquement par tunnel SSH                                                                     |
| Nom d'hôte fixé avant l'installation                                  | Proxmox fige le nom du nœud à l'installation ; le renommer ensuite est mal supporté                                             | —                                                                                                   |
| Noyau installé et redémarrage avant le reste                          | En cas d'échec de démarrage, la machine reste un Debian ordinaire, facile à réparer                                             | Un redémarrage supplémentaire                                                                       |
| Dépôt `pve-no-subscription`                                           | Gratuit, sans abonnement                                                                                                        | Mises à jour moins testées que le dépôt entreprise ; bandeau au login                               |
| Suppression des noyaux Debian                                         | Éviter un redémarrage accidentel sur un noyau sans KVM                                                                          | Plus de repli noyau en cas de régression PVE                                                        |

### Problèmes rencontrés

- Mot de passe root inconnu au premier accès à l'interface web : sur image Hetzner l'accès est par clé, le mot de passe système n'est pas défini. Réglé avec `passwd root`.
