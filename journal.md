# Journal de bord

Notes prises au fil de l'eau pendant que je monte le lab. C'est volontairement brut, ce n'est pas de la documentation. Ça me sert de matière première pour rédiger les README.

---
## 2026-09-16 - 

**Fait**
- Etat des lieux pour savoir ce qui est en écoute sur les ports TCP et UDP (`ss -tulnp`).
        - Services en écoutes en UDP : chrony (boucle locale) et rpcbind (ipv4 + ipv6).
        - Services en écoutes en TCP : rpcbind (ipv4 + ipv6), ssh (ipv4 + ipv6) et services proxmox sur la boucle locale.
- Edition de `/etc/default/rpcbind` :
        - Ajout d'une ligne pour restreindre rpcbind à la boucle locale pour les requêtes UDP (`OPTIONS="${OPTIONS} -h 127.0.0.1 -h ::1"`).

**Décisions**
| Décisions | Raison | Contrepartie |
|---|---|---|
|/etc/default/rpcbind modifié|Restreindre rpcbind à la boucle locale|Fichier susceptible d'être écrasé lors d'une mise à jour.|
|Restreindre rpcbind plutôt que désinstaller|Dépendances Proxmox|Laisse un service en fonctionnement dont je n'ai pas l'usage. Surface d'attaque réduite mais toujours existante|

**Problèmes**
- Signalement informatif de la team abuse de Hetzner : `rpcbind` en écoute en TCP et UDP suite à l'installation de proxmox. Risque de DDOS et de divulgation d'information. Note à même de faire un état des lieux à chaque installation importante.
- Tentative de désinstallation de rpcbind car je n'utilise aucun services basés sur RPC. Désinstallation impossible à cause des dépendances Proxmox. 

## 2026-09-11 - Segmentation : Isolation de vmbr2 (rédigé 4 jours après, certains détails perdus)

**Objectif**
Permettre au bridge vmbr2 d'accéder à Internet, tout en bloquant l'accès au réseau 10.10.10.0/24

**Fait**
- Edition de `/etc/nftables.conf` :
  - Création de la table `filter`, chaîne `forward`, politique `drop` par défaut, avec règle permettant le trafic vers l'extérieur.
  - Ajout du réseau 10.10.20.0/24 à la table `nat` via un ensemble d'adresses avec `set`.

**Décisions**
| Décisions | Raison | Contrepartie |
|---|---|---|
|`nftables` plutôt que `pve-firewall`|Apprendre l'outil natif sur les distributions Linux modernes. `pve-firewall` basé sur iptables|Configuration manuelle à maintenir si je passe un jour cluster|
|Politique `drop` par défaut sur `forward`|Autoriser uniquement les services dont j'ai besoin, et bloquer tout le reste|Chaque service ajouté doit être autorisé manuellement|
|Politique `accept` par défaut sur `input` et `output`|Décision assumée de ne pas durcir pour le moment, le durcissement viendra plus tard.|Risque d'exposition des ports ouverts|
|Utilisation de `set` plutôt que 2 règles de masquerading.|Plus lisible et plus simple à maintenir||
|Ajout de la règle `ct state established,related`|Autoriser le trafic de retour, sans quoi rien ne fonctionne||

**Problèmes**
- Syntaxe `set` incomplète : Il manquait le paramètre `type`, et le `flags` permettant au set de stocker des plages d'adresses.
- Politique par défaut invalide sur une chaine : je tentais de mettre une `policy reject` en `forward`. Seuls `accept` et `drop` sont des politiques valides. `reject` est une action de règle.
- Oubli du trafic de retour dans un premier temps.
- @s1 alors que le set avait été renommé nat_networks.
- Règle de trafic intra-vmbr1 écrite alors qu'elle ne peut jamais matcher : deux VMs sur le même bridge sont commutées, le hook forward n'est pas sollicité.

**À faire**
- Durcissement de la chaine `input`.
- Ajouts des compteurs nftables.
- Tests du NAT via une vraie VM.

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
