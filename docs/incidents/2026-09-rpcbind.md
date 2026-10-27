# Incident : rpcbind exposé sur Internet

| | |
|---|---|
| Signalement | 2026-09-16, équipe abuse de Hetzner, à la suite d'un rapport du BSI |
| Correction | 2026-09-19 |
| Machine | `pve1` |
| Service concerné | `rpcbind`, port 111 en TCP et UDP |
| Résultat | Seul SSH reste exposé sur Internet |

## Résumé

Le 16 septembre 2026, Hetzner m'a transmis un signalement du BSI allemand : le port 111 de mon serveur répondait depuis Internet. J'ai arrêté le service le jour même. Le 19 septembre, je l'ai restreint à la boucle locale, sans désinstaller `rpcbind` parce que Proxmox en dépend.

## Pourquoi c'est un problème

`rpcbind` annonce les services RPC disponibles sur une machine. Exposé sur Internet, il permet de les énumérer et peut servir de relais dans une attaque DDoS par réflexion/amplification. Je n'en ai aucun usage.

## Chronologie

| Date | Événement |
|---|---|
| 2026-08-20 | État des lieux avec `ss -tlnp` : seul SSH écoute. Puis installation de Proxmox, qui apporte `rpcbind`.|
| 2026-09-16 | Réception du signalement. `ss -tulnp` confirme que `rpcbind` écoute sur toutes les interfaces, en IPv4 et IPv6, TCP et UDP. J'arrête `rpcbind.service` et `rpcbind.socket`. |
| 2026-09-16 à 19 | Service arrêté. Analyse des dépendances, puis recherche d'une correction qui ne casse pas Proxmox. |
| 2026-09-19 | Écoute restreinte à la boucle locale, service réactivé, vérification. |

Le message de Hetzner précisait qu'aucune réponse n'était attendue, ni de ma part ni du BSI. Je n'ai donc pas répondu.

## Cause

**Cause principale : je n'ai pas refait l'inventaire des ports après l'installation de Proxmox.** `rpcbind` est arrivé avec Proxmox, pas avec l'image Hetzner. Mon état des lieux du 20 août a été fait avant l'installation : il était exact, seul SSH écoutait, mais `rpcbind` n'existait pas encore. Un inventaire refait après l'installation l'aurait montré, même sans l'UDP, puisque `rpcbind` écoute aussi en TCP sur le port 111.

**Faiblesse de méthode : mon inventaire était limité au TCP** (`ss -tlnp`, sans le `u`). Ça n'a rien changé dans ce cas, mais un service qui n'écouterait qu'en UDP m'aurait échappé.

## Impact

Je n'ai relevé aucun signe d'utilisation abusive, mais je n'ai aucun moyen de le prouver : `rpcbind` ne journalise pas par défaut les requêtes qu'il reçoit. Le port 111 est resté exposé environ quatre semaines, du 20 août (installation de Proxmox) au 16 septembre.

## Correction

**Désinstaller n'était pas possible.** La chaîne de dépendances est `rpcbind` ← `nfs-common` ← `libpve-storage-perl` ← `proxmox-ve`. Purger `rpcbind` aurait emporté l'hyperviseur. Je l'ai vu en lisant la liste de `apt purge` avant de confirmer.

**Restreindre le service ne suffit pas.** J'ai d'abord limité l'écoute dans `/etc/default/rpcbind`, qui contenait deux lignes `OPTIONS` dont la seconde écrasait la première. Mais avec systemd, le socket écoute, pas seulement le service. `systemctl cat rpcbind.socket` montrait des `ListenStream` et `ListenDatagram` sur `0.0.0.0` et `[::]`.

**Surcharge du socket.** J'ai ajouté une surcharge systemd qui remplace les adresses d'écoute :

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

Les deux lignes vides en tête remettent les listes à zéro. Sans elles, les nouvelles adresses s'ajouteraient aux anciennes.

Fichiers dans ce dépôt :

- [`configs/pve1/etc/default/rpcbind`](../../configs/pve1/etc/default/rpcbind)
- [`configs/pve1/etc/systemd/system/rpcbind.socket.d/override.conf`](../../configs/pve1/etc/systemd/system/rpcbind.socket.d/override.conf)

## Vérification

```bash
ss -tulnp                                   # seul 127.0.0.1:111 et [::1]:111 pour le port 111
systemctl status pveproxy pvedaemon         # Proxmox fonctionne
ssh -L 8006:127.0.0.1:8006 root@203.0.113.1 # l'interface web répond par le tunnel
```

Cette même sortie montre que les autres services de Proxmox (`pveproxy`, `pvedaemon`, `spiceproxy`) et `chronyd` n'écoutent que sur la boucle locale. Seul `sshd` écoute sur toutes les interfaces.

Contrôle depuis l'extérieur, le 2026-10-24, avec un scan TCP complet puis un scan UDP du port 111 :

```bash
nmap -Pn -sT -p- 203.0.113.1
```

```text
PORT   STATE    SERVICE
22/tcp open     ssh
25/tcp filtered smtp
(65533 autres ports TCP fermés)
```

```bash
nmap -Pn -sU -p 111 203.0.113.1
```

```text
PORT    STATE  SERVICE
111/udp closed rpcbind
```

Seul SSH est ouvert, et le port 111 est fermé en TCP comme en UDP. Le port 25 est « filtered », c'est-à-dire sans réponse. Le `ss -tulnp` du 19 septembre montre que Postfix n'écoute que sur `127.0.0.1:25` et `[::1]:25` : le filtrage ne vient donc pas de mon serveur. Son origine reste à trouver.

Limite : je n'ai pas testé l'IPv6.

## Ce que j'en retiens

- Je refais un inventaire des ports après chaque installation importante, en TCP **et** en UDP.
- Avec l'activation par socket, configurer le service seul ne ferme rien. C'est l'unité `.socket` qu'il faut regarder.
- Les directives systemd de type liste s'additionnent. Pour remplacer une valeur, il faut d'abord vider la liste.
- Quand la solution évidente (désinstaller) est bloquée, il faut viser l'objectif plutôt que la méthode : ici, fermer le port sans casser le reste.

## Ce qui reste ouvert

- `rpcbind` tourne toujours. La surface d'attaque est réduite mais pas supprimée.
- La chaîne `input` du pare-feu est encore en `policy accept`. Avec un refus par défaut, ce port serait resté fermé même mal configuré, ainsi que tous ceux que je ne connais pas encore.
- Le port 25 apparaît « filtered » dans mon scan externe. Le filtrage ne vient pas de mon serveur, mais je n'en ai pas encore trouvé l'origine. Je n'ai pas non plus testé l'IPv6, alors que `rpcbind` écoutait aussi sur `[::]`.