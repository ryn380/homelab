# Journal de bord

Notes prises au fil de l'eau pendant que je monte le lab. C'est volontairement brut, ce n'est pas de la documentation. Ça me sert de matière première pour rédiger les README.

---

## 2026-09-09 - Réseau : Isolation de vmbr2

**Objectif**
vmbr2 sort sur Internet, mais ne peut jamais atteindre 10.10.10.0/24.

**Fait**
- 
- 

**Décisions**
| Décisions | Raison | Contrepartie |
|---|---|---|
||||

## 2026-09-08 — Réseau : bridges, routage, NAT

**Fait**
- Création de `/etc/sysctl.d/100-pve-routing.conf` avec `net.ipv4.ip_forward=1`. Appliqué avec `sysctl --system`.
- Edition de `/etc/nftables.conf` : table `nat`, chaîne `postrouting`, masquerade sur 10.10.10.0/24.
- Activation du service `nftables`, qui était `disabled`.
- Création de `/etc/network/interfaces.d/10-bridges.conf` : `vmbr1` en 10.10.10.1/24 et `vmbr2` en 10.10.20.1/24, `bridge-ports none`.
- Appliqué avec `ifreload -a` après avoir armé un retour arrière différé avec `at`.

**Décisions**

| Décision | Raison | Contrepartie |
|---|---|---|
| Fichier dans `sysctl.d/` plutôt que modifier `99-hetzner.conf` | Fichier fournisseur, peut être écrasé sans prévenir | Un fichier de plus à connaître |
| Préfixe `100-` et pas `99-` | `99-sysctl.conf` est un lien vers `/etc/sysctl.conf`, il passerait après le mien | Aucune |
| `bridge-ports none` | Bridges internes. Hetzner ne route qu'une MAC sur le port, une VM sur un bridge relié à `<IFACE>` ferait bloquer le trafic et déclencherait un signalement abus | Pas d'accès direct des VMs au réseau public, tout passe par du NAT |
| `masquerade` plutôt que `snat` | Évite d'écrire mon IP publique dans un fichier versionné sur GitHub | Légèrement moins efficace, l'adresse est cherchée à chaque connexion |
| `oifname` plutôt que `oif` | `oif` résout le nom au chargement, la règle casse si l'interface change d'index | Un peu plus lent |
| `/etc/nftables.conf` écrit à la main plutôt que généré avec `nft list ruleset` | Le fichier documente ce que je voulais, pas seulement ce qui tourne. Garde commentaires et noms symboliques | Il faut penser à vérifier que le chargé correspond à l'écrit |
| Retour arrière avec `at` plutôt que `cron` | Exécution unique, c'est ce que je veux pour un filet |Aucune |
Choix de nftables plutot que pve-firewall 

**Problèmes**
- Bloqué plusieurs heures sur la syntaxe nftables. Je venais d'iptables et le modèle est différent : nftables ne fournit aucune table ni chaîne par défaut, il faut tout déclarer.
- Première version des bridges avec `bridge-ports <IFACE>`. Erreur : ça aurait mis mon interface publique en mode commutateur, et je l'avais mise sur les deux bridges alors qu'une interface ne peut appartenir qu'à un seul.
- Confusion couche 2 / couche 3 : je pensais que le bridge était la passerelle. C'est l'hôte qui est passerelle, le bridge est un commutateur virtuel et l'adresse `.1` est celle que l'hôte porte dessus.

---

## 2026-08-20 — Prise en main du serveur et installation de Proxmox

**Fait**
- Première connexion SSH, empreinte de clé vérifiée avant acceptation.
- État des lieux : Debian 12, noyau 6.1, Ryzen 5 3600 (12 threads), 62 Gio de RAM, 2 x 476 Gio NVMe.
        - Disques en mdadm RAID 1 : `md0` swap 32 Gio, `md1` /boot 1 Gio, `md2` racine 443 Gio, ext4. Les trois grappes en `[2/2] [UU]`.
        - Services en écoute avant intervention : SSH uniquement (IPv4 + IPv6).
- Durcissement SSH dans `/etc/ssh/sshd_config.d/10-hardening.conf` : mot de passe désactivé, root par clé uniquement.
- Ajout du dépôt `pve-no-subscription`, empreinte de la clé de signature comparée au wiki.
- Noyau PVE installé seul, redémarrage, puis installation de `proxmox-ve`.
- Nom d'hôte passé à `pve1`, `/etc/hosts` aligné sur l'IP publique.
- `LISTEN_IP=127.0.0.1` dans `/etc/default/pveproxy` avant l'installation.
- Noyaux Debian et `os-prober` retirés.
- Accès à l'interface web par tunnel SSH.

**Décisions**

| Décision | Raison | Contrepartie |
|---|---|---|
| Proxmox par-dessus le Debian existant plutôt que réinstallation ZFS | Comprendre ce que Proxmox ajoute à un Debian, sans passer par le rescue sur une machine que je ne connais pas | mdadm n'est pas un stockage officiellement pris en charge |
| `pveproxy` limité à la boucle locale | Le service démarre dès la fin de l'installation, sur IP publique la fenêtre d'exposition est de quelques minutes | Accès uniquement par tunnel SSH |
| Nom d'hôte fixé avant l'installation | Proxmox fige le nom du nœud dans `/etc/pve` et les certificats, le renommer après est mal supporté | Aucune |
| Noyau installé et redémarrage avant le reste | Si le noyau ne démarre pas, la machine est encore un Debian ordinaire | Un redémarrage de plus |
| Dépôt `pve-no-subscription` | Gratuit | Mises à jour moins testées, bandeau au login |
| Fichier séparé dans `sshd_config.d/` | Les mises à jour du paquet peuvent écraser le fichier principal | Il faut penser à `sshd -T` pour voir la config réelle |

**Problèmes**
- Mot de passe root inconnu au premier accès à l'interface web. Sur image Hetzner l'accès est par clé, le mot de passe système n'est pas défini.
