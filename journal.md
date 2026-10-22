# Journal de bord

Notes prises au fil de l'eau pendant que je monte le lab. J'y note ce que je fais, ce qui a coincé, et ce que j'en ai retenu. Les entrées les plus récentes sont en haut.

Les adresses IP publiques sont remplacées par `203.0.113.1` (plage de documentation réservée, RFC 5737).

---

## 2026-09-19 - Sécurité : restriction de rpcbind à la boucle locale

Suite du signalement reçu le 16/09 (voir entrée précédente). Service resté arrêté 3 jours entre le signalement et la correction.

**Objectif**
Fermer le port 111 sur l'interface publique sans casser Proxmox, et remettre le service en état nominal.

**Fait**

Tentative de désinstallation, bloquée par les dépendances.
```bash
apt-cache rdepends --installed rpcbind   # paquets dépendants
apt purge rpcbind                        # lire la liste AVANT de confirmer
```
Chaîne de dépendances : `rpcbind` ← `nfs-common` ← `libpve-storage-perl` ← `proxmox-ve`.
Purger `rpcbind` aurait emporté l'hyperviseur. Abandon de cette option.

Restriction de l'écoute du service via `/etc/default/rpcbind`.
Le fichier contenait deux lignes `OPTIONS` successives, la seconde écrasant la première. Remplacées par une seule ligne explicite :
```text
OPTIONS="-w -h 127.0.0.1 -h ::1"
```

Surcharge du socket systemd, qui est ce qui écoute réellement.
```bash
systemctl cat rpcbind.socket    # constat : ListenStream/ListenDatagram sur 0.0.0.0 et [::]
systemctl edit rpcbind.socket
```
```text
[Socket]
ListenStream=
ListenDatagram=

ListenStream=/run/rpcbind.sock

ListenStream=127.0.0.1:111
ListenDatagram=127.0.0.1:111
ListenStream=[::1]:111
ListenDatagram=[::1]:111
```
Les deux premières lignes vides remettent les listes à zéro. Sans elles, les nouvelles adresses s'ajoutent aux anciennes au lieu de les remplacer.

Rechargement, redémarrage et réactivation.
```bash
systemctl daemon-reload
systemctl start rpcbind.socket     # socket en premier, puis service
systemctl start rpcbind.service
systemctl enable rpcbind.socket rpcbind.service
systemctl cat rpcbind.socket       # vérifier la config effective (unité + surcharge)
```

Vérifications.
```bash
ss -tulnp                          # plus que 127.0.0.1:111 et [::1]:111
systemctl status pveproxy pvedaemon
ssh -L 8006:127.0.0.1:8006 root@203.0.113.1
```
Résultat : seul SSH reste exposé sur Internet. Proxmox tourne sans interruption depuis le 20/08, interface web accessible.

**Décisions**

| Décision                                             | Raison                                                                                          | Contrepartie                                                                                 |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Restreindre rpcbind plutôt que le désinstaller       | `proxmox-ve` en dépend par la chaîne `nfs-common` / `libpve-storage-perl`                       | Laisse tourner un service dont je n'ai pas l'usage. Surface d'attaque réduite, pas supprimée |
| Modification de `/etc/default/rpcbind`               | Restreindre l'écoute du service à la boucle locale                                              | Fichier de paquet, susceptible d'être écrasé à une mise à jour                               |
| Surcharge systemd plutôt que modification de l'unité | L'unité du paquet peut être écrasée à la mise à jour                                            | Config éclatée entre deux fichiers, `systemctl cat` obligatoire pour la lire                 |
| Service réactivé après restriction                   | Proxmox peut en avoir besoin, un service désactivé créerait une panne difficile à diagnostiquer | Service en écoute conservé                                                                   |

**Problèmes**
- **Avec systemd, le service n'est pas toujours celui qui écoute.** L'activation par socket délègue l'écoute à systemd : configurer `/etc/default/rpcbind` seul ne ferme rien. C'est ce qui explique la ligne `systemd` à côté de `rpcbind` dans la sortie de `ss`. Point le plus transposable de la session.
- Les directives systemd de type liste s'additionnent. Une surcharge qui ajoute des `ListenStream` sans vider la liste laisse les anciennes adresses en place.
- La solution évidente (désinstaller) était bloquée par le contexte. Il a fallu un repli : atteindre l'objectif sans casser le reste.

**À faire**
- Durcissement de la chaîne `input` : il fermerait ce port et tous ceux que je ne connais pas encore.
- Refaire un `ss -tulnp` après chaque installation importante.
- Vérifier si un autre service de la pile Proxmox écoute au-delà de la boucle locale.

---

## 2026-09-16 - Signalement Hetzner : port 111 exposé

**Fait**

État des lieux de ce qui écoute, en TCP et UDP.
```bash
ss -tulnp
```
- En UDP : `chronyd` (boucle locale) et `rpcbind` (IPv4 + IPv6, toutes interfaces).
- En TCP : `rpcbind` (IPv4 + IPv6, toutes interfaces), `sshd` (IPv4 + IPv6) et les services Proxmox sur la boucle locale.

Arrêt du service en urgence à réception du mail.
```bash
systemctl stop rpcbind.service rpcbind.socket
```

**Problèmes**
- Signalement informatif transmis par l'équipe abuse de Hetzner, à la suite d'un rapport du BSI allemand. `rpcbind` écoute sur le port 111 en TCP et UDP depuis Internet : risque de réflexion/amplification DDoS et divulgation des services RPC disponibles.
- Cause racine : `rpcbind` est arrivé avec l'installation de Proxmox, pas avec l'image Hetzner. Mon état des lieux du 20/08 ne montrait que SSH, mais il était en TCP seulement (`ss -tlnp` sans le `u`) et je ne l'ai pas refait après l'installation.
- Leçon : refaire un inventaire des ports après chaque installation importante, avec TCP **et** UDP.

**À faire**
- Corriger sans casser Proxmox (voir entrée du 19/09).

---

## 2026-09-11 - Segmentation : isolation de vmbr2

*(rédigé 4 jours après, certains détails perdus)*

**Objectif**
Permettre au bridge `vmbr2` d'accéder à Internet, tout en bloquant l'accès au réseau 10.10.10.0/24.

**Fait**
- Édition de `/etc/nftables.conf` :
  - Création de la table `filter`, chaîne `forward`, politique `drop` par défaut, avec une règle autorisant le trafic vers l'extérieur.
  - Ajout du réseau 10.10.20.0/24 à la table `nat` via un ensemble d'adresses (`set`).

**Décisions**

| Décision | Raison | Contrepartie |
| --- | --- | --- |
| `nftables` plutôt que `pve-firewall` | Apprendre l'outil natif des distributions Linux modernes. `pve-firewall` génère ses règles via la couche iptables | Configuration manuelle à maintenir si je passe un jour en cluster |
| Politique `drop` par défaut sur `forward` | Autoriser uniquement ce dont j'ai besoin, bloquer tout le reste | Chaque service ajouté devra être autorisé explicitement |
| Politique `accept` conservée sur `input` et `output` | Décision assumée de ne pas durcir aujourd'hui, pour ne pas mélanger une modification risquée avec le reste | Ports ouverts exposés, ce que le signalement Hetzner a confirmé 5 jours plus tard |
| `set` plutôt que deux règles de masquerading | Plus lisible quand la liste de réseaux grandit | À défaire si je veux un jour une règle différente par réseau |
| Règle `ct state established,related` | Autoriser le trafic de retour, sans quoi rien ne fonctionne avec une politique `drop` | Aucune |

**Problèmes**
- Bloqué plusieurs heures sur la syntaxe nftables. Je venais d'iptables et le modèle est différent : nftables ne fournit aucune table ni chaîne par défaut, il faut tout déclarer soi-même.
- Syntaxe `set` incomplète : il manquait `type` et le `flags interval` qui permet de stocker des plages d'adresses.
- `policy reject` sur la chaîne `forward` : invalide. Seuls `accept` et `drop` sont des politiques de chaîne, `reject` est une action de règle.
- Oubli du trafic de retour dans un premier temps.
- `@s1` alors que j'avais renommé le set en `nat_networks`. Erreur d'inattention.
- Règle de trafic intra-`vmbr1` écrite alors qu'elle ne peut jamais matcher : deux VMs sur le même bridge sont commutées, le hook `forward` n'est pas sollicité.

**À faire**
- Durcissement de la chaîne `input`.
- Ajout des compteurs nftables.
- Test du NAT avec une vraie VM.

---

## 2026-09-08 — Réseau : bridges, routage, NAT

**Objectif**
Créer deux réseaux internes pour les futures VMs, avec sortie Internet par NAT, sans toucher à l'interface qui porte mon accès SSH.

**Fait**

Filet de sécurité armé avant toute modification réseau.
```bash
echo -e "rm -f /etc/network/interfaces.d/10-bridges\n/sbin/ifreload -a" | at now + 10 minutes
at -l          # noter le numéro de tâche
at -c <num>    # vérifier le contenu réel de la tâche
```

Routage IPv4 persistant.
```bash
# /etc/sysctl.d/100-pve-routing.conf
net.ipv4.ip_forward=1
```
```bash
sysctl --system
sysctl net.ipv4.ip_forward   # doit renvoyer 1
```

Bridges internes dans `/etc/network/interfaces.d/10-bridges`.
```text
auto vmbr1
iface vmbr1 inet static
  address 10.10.10.1/24
  bridge-ports none
  bridge-stp off
  bridge-fd 0

auto vmbr2
iface vmbr2 inet static
  address 10.10.20.1/24
  bridge-ports none
  bridge-stp off
  bridge-fd 0
```

NAT dans `/etc/nftables.conf`, table `nat`, chaîne `postrouting`.

Application et vérification.
```bash
nft -c -f /etc/nftables.conf   # test de syntaxe sans appliquer
ifreload -a                    # applique le delta, ne coupe pas les interfaces actives
ip -br a
at -r <num>                    # annulation du filet une fois l'accès confirmé
```
Test de connexion depuis un second terminal avant d'annuler le filet.

**Décisions**

| Décision | Raison | Contrepartie |
| --- | --- | --- |
| Fichier dans `sysctl.d/` plutôt que modifier `99-hetzner.conf` | Fichier fournisseur, peut être écrasé sans prévenir | Un fichier de plus à connaître |
| Préfixe `100-` et pas `99-` | `99-sysctl.conf` est un lien vers `/etc/sysctl.conf`, il serait lu après le mien et gagnerait | Aucune |
| Fichier dans `interfaces.d/` plutôt que le fichier principal | Ne touche pas à la strophe qui porte mon accès. Le retour arrière est une simple suppression de fichier | L'interface web Proxmox ne voit pas forcément ces bridges |
| `bridge-ports none` | Bridges internes. Hetzner ne route qu'une MAC sur le port : une VM sur un bridge relié à l'interface physique ferait bloquer le trafic et déclencherait un signalement abus | Pas d'accès direct des VMs au réseau public, tout passe par le NAT |
| `bridge-stp off` et `bridge-fd 0` | Aucune interconnexion, donc aucune boucle possible. Sans STP, le délai d'apprentissage n'a plus de raison d'être | Aucune tant que la topologie reste plate |
| `masquerade` plutôt que `snat` | Évite d'écrire mon IP publique dans un fichier versionné sur GitHub | Légèrement moins efficace, l'adresse est résolue à chaque connexion |
| `oifname` plutôt que `oif` | `oif` résout le nom au chargement et stocke un index : la règle casse si l'interface change d'index | Un peu plus lent, comparaison de chaîne à l'exécution |
| `/etc/nftables.conf` écrit à la main plutôt que généré par `nft list ruleset` | Le fichier documente ce que je voulais, pas seulement ce qui tourne. Garde les commentaires et les noms symboliques | Penser à vérifier que le chargé correspond à l'écrit |
| Retour arrière avec `at` plutôt que `cron` | Exécution unique, c'est ce que je veux pour un filet | `at` n'est pas installé par défaut partout |

**Problèmes**
- Première version des bridges avec `bridge-ports <IFACE>`. Erreur : ça aurait mis mon interface publique en mode commutateur, et je l'avais mise sur les **deux** bridges alors qu'une interface ne peut appartenir qu'à un seul.
- Confusion couche 2 / couche 3 : je pensais que le bridge était la passerelle. C'est l'hôte qui est passerelle. Le bridge est un commutateur virtuel, et l'adresse `.1` est celle que l'hôte porte dessus.
- Adresses en `.2` au premier jet, alors que mon propre plan d'adressage réserve `.1` à la passerelle. Une convention qu'on ne respecte pas ne sert à rien.
- `systemctl restart networking` envisagé au départ : couperait l'interface physique et donc ma session. `ifreload -a` applique la différence sans toucher aux interfaces déjà actives.

**À faire**
- Isolation de `vmbr2` : le routage est global, rien ne sépare les deux réseaux pour l'instant.
- Valider le NAT avec une vraie VM.

---

## 2026-08-20 — Prise en main du serveur et installation de Proxmox

**Objectif**
Première connexion SSH, état des lieux du serveur livré, installation de Proxmox.

**Fait**

Première connexion.
```bash
ssh -i ~/.ssh/pve1_ed25519 root@203.0.113.1
```
Empreinte de clé comparée à celle communiquée par Hetzner avant acceptation.

État des lieux.
```bash
cat /etc/os-release | head -2 && uname -r && uptime && hostname -f   # SYSTEME
lscpu | grep -E "^(Model name|CPU\(s\)|Thread)" && free -h           # CPU/RAM
lsblk && df -h /                                                     # DISQUES
cat /proc/mdstat                                                     # RAID
ip -br a && ip r && cat /etc/hosts                                   # RESEAU
ss -tlnp                                                             # ECOUTE
sshd -T | grep -E 'permitrootlogin|passwordauthentication|^port'     # SSH
last -i | head                                                       # CONNEXIONS
```
- Debian 12, noyau 6.1, Ryzen 5 3600 (12 threads), 62 Gio de RAM, 2 x 476 Gio NVMe.
- Disques en RAID 1 mdadm : `md0` swap 32 Gio, `md1` `/boot` 1 Gio, `md2` `/` 443 Gio. Les trois grappes en `[2/2] [UU]`.
- Services en écoute avant intervention : SSH uniquement.

Durcissement SSH dans `/etc/ssh/sshd_config.d/10-hardening.conf`.
```text
PasswordAuthentication no
PermitRootLogin prohibit-password
```
```bash
sshd -t && echo "syntaxe OK"
systemctl reload ssh
sshd -T | grep -E 'passwordauthentication|permitrootlogin'
```
Test de connexion depuis un second terminal avant de fermer la session en cours.

Nom d'hôte fixé avant l'installation, `/etc/hosts` aligné sur l'IP publique.
```bash
hostnamectl set-hostname pve1
hostname -f    # pve1.lab.internal
hostname -i    # doit renvoyer l'IP publique, jamais 127.0.1.1
```

Dépôt Proxmox, avec vérification de l'empreinte de la clé de signature.
```bash
apt update && apt full-upgrade -y
apt install -y curl gnupg
curl -fsSL https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg \
  -o /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg
sha512sum /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg   # comparé au wiki Proxmox

echo "deb [arch=amd64] http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-install-repo.list
apt update
```

Noyau PVE installé seul, puis redémarrage.
```bash
apt install -y proxmox-default-kernel
systemctl reboot
uname -r    # doit contenir 'pve'
```

Interface web restreinte à la boucle locale **avant** l'installation, pour qu'il n'y ait aucune fenêtre d'exposition.
```bash
echo "LISTEN_IP=127.0.0.1" > /etc/default/pveproxy
```

Installation de Proxmox.
```bash
apt update && apt install -y proxmox-ve postfix open-iscsi chrony
# postfix : choisir "Local only"
```

Nettoyage des noyaux Debian et d'`os-prober`.
```bash
apt remove -y linux-image-amd64 'linux-image-6.1*' os-prober
update-grub
```

Premier accès à l'interface web par tunnel SSH.
```bash
ssh -L 8006:127.0.0.1:8006 root@203.0.113.1
# puis https://127.0.0.1:8006 dans le navigateur
```

**Décisions**

| Décision | Raison | Contrepartie |
| --- | --- | --- |
| Proxmox par-dessus le Debian existant plutôt qu'une réinstallation ZFS | Comprendre ce que Proxmox ajoute à un Debian, sans passer par le rescue sur une machine que je ne connais pas encore | mdadm n'est pas un stockage officiellement pris en charge. Pas d'instantanés ni de réplication ZFS |
| `pveproxy` limité à la boucle locale | Le service démarre dès la fin de l'installation. Sur une IP publique, la fenêtre d'exposition se compte en minutes | Accès uniquement par tunnel SSH |
| Nom d'hôte fixé avant l'installation | Proxmox fige le nom du nœud dans les chemins de `/etc/pve` et dans les certificats. Le renommer après est mal supporté | Aucune |
| Noyau installé et redémarrage avant le reste | Si le noyau ne démarre pas, la machine est encore un Debian ordinaire, facile à réparer | Un redémarrage supplémentaire |
| Dépôt `pve-no-subscription` | Gratuit, sans abonnement | Mises à jour moins testées que le dépôt entreprise, bandeau au login |
| Fichier séparé dans `sshd_config.d/` | Les mises à jour du paquet peuvent écraser le fichier principal | Penser à `sshd -T` pour lire la configuration réelle |
| Suppression des noyaux Debian | Éviter un redémarrage accidentel sur un noyau sans KVM | Plus de repli noyau en cas de régression PVE |

**Problèmes**
- `resync=PENDING` sur `md0` : pris pour une anomalie. C'est la grappe de swap, jamais écrite depuis le démarrage, donc mdadm reporte la synchronisation initiale. Normal. Ce qu'il faut surveiller, c'est `[2/2] [UU]`.
- `grep` sur `/etc/ssh/sshd_config` donnait une vision fausse : la configuration est éclatée entre le fichier principal et `sshd_config.d/`, et les valeurs par défaut ne sont écrites nulle part. `PasswordAuthentication yes` était actif sans figurer dans aucun fichier. `sshd -T` est la seule source de vérité.
- Mot de passe root inconnu au premier accès à l'interface web. Sur une image Hetzner, l'accès se fait par clé et le mot de passe système n'est pas défini. Réglé avec `passwd root`.
- `open-iscsi` installé en suivant le wiki alors que je n'ai aucun stockage SAN. Sans danger (initiateur, aucun port en écoute), mais je n'en ai pas l'usage.

**À faire**
- Configuration réseau : bridges, réseaux privés, NAT.
- Vérifier que GRUB est installé sur les deux disques.
- Vérifier le contrôle mensuel mdadm et configurer l'alerte mail en cas de disque défaillant.
- Compte d'administration dédié + TOTP, root réservé aux urgences.
- Sauvegarde de la configuration de l'hôte.
