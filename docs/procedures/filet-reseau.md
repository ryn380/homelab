# Filet de sécurité réseau

> Je travaille sur une machine louée dont je n'ai pas d'accès physique. Une mauvaise configuration peut donc me couper l'accès au serveur. Avant toute modification de ce genre, j'arme un retour arrière différé : une tâche qui s'exécute seule 10 minutes plus tard si je n'ai pas confirmé que tout fonctionne. Cette page décrit le principe, puis deux cas : les interfaces réseau et le pare-feu nftables.

## Pourquoi ce filet

Mon serveur est loué chez Hetzner : je n'ai pas de console physique, tout se fait à distance. Une modification des interfaces ou du pare-feu peut donc me couper l'accès. Pour le récupérer, je devrais passer par le support Hetzner, ou par un reset suivi d'une réparation manuelle depuis leur système de secours.

| Option | Délai | Interruption des VM | Effort |
|---|---|---|---|
| Filet `at` | 10 min max, automatique | Aucune | Une commande à armer avant la modification |
| Console KVM via le support | Variable, selon la charge du support | Aucune | Se connecter à la console et corriger la configuration à la main |
| Système de secours + reset | Le temps d'un redémarrage | Oui, toutes les VM | Monter les disques depuis un système minimal et réparer à la main |

Pour une erreur sur les interfaces ou le pare-feu, le filet me rend l'accès en 10 minutes maximum, sans redémarrage. Les deux autres options dépendent du support ou d'un redémarrage.

## Le principe

Le principe est le même pour tous les cas. Seule la commande de retour arrière change.

1. **Armer** : je programme une tâche `at` qui défait ma modification dans 10 minutes.
2. **Modifier** : j'effectue ma modification, sans l'appliquer.
3. **Appliquer** : j'applique ma modification.
4. **Vérifier** : je vérifie ma modification et me reconnecte depuis une seconde session SSH.
5. **Annuler** : si tout fonctionne, je supprime la tâche. Sinon, elle s'exécute seule au bout de 10 minutes.

**Je l'arme avant d'appliquer, parce que c'est l'étape « appliquer » qui risque de me couper l'accès.** Si le filet n'est pas en place à ce moment-là, plus rien ne peut me rattraper.

**Ordre à retenir : armer, modifier, appliquer, vérifier, annuler le filet.**

## Procédure pas à pas

**Prérequis : le service `atd` doit être actif.** Le filet repose sur lui. S'il est arrêté, la tâche est acceptée mais ne s'exécute pas à l'heure prévue : elle ne part qu'au démarrage du service (voir « Limites et pièges »). Je m'en assure avant toute chose :

```bash
systemctl is-active atd.service
# doit retourner active
```

1. **Armer**

   Je programme la commande de retour arrière. Je relis le contenu réel de la tâche avec `at -c` et je note son numéro.

   ```bash
   echo "<commande de retour arrière>" | at now + 10 minutes
   at -l          # noter le numéro de tâche
   at -c <num>    # vérifier le contenu réel de la tâche
   ```

2. **Modifier**

   Je prépare mes modifications. Rien n'est encore appliqué.

3. **Appliquer**

   Je teste la syntaxe quand l'outil le permet. Puis j'applique.

4. **Vérifier**

   J'ouvre une seconde session SSH, sans fermer la première : c'est la preuve que je peux me reconnecter, alors que la session déjà ouverte peut survivre à une mauvaise configuration. Je vérifie aussi la bonne prise en compte de mes modifications.

5. **Annuler**

   Si tout fonctionne, je supprime la tâche différée, pour que le retour arrière ne défasse pas une configuration qui marche. Je vérifie ensuite avec `at -l` qu'elle n'est plus en file d'attente.

   ```bash
   at -r <num>    # suppression du retour arrière
   at -l          # aucune sortie
   ```

## Cas 1 : les interfaces réseau

Exemple : ajout des bridges internes dans mon propre fichier `/etc/network/interfaces.d/10-bridges`.

**Retour arrière : supprimer mon fichier, puis recharger les interfaces.** Comme le fichier est le mien, je n'ai rien à sauvegarder.

```bash
# 1. Armer
echo -e "rm -f /etc/network/interfaces.d/10-bridges\n/sbin/ifreload -a" | at now + 10 minutes
at -l
at -c <num>

# 2. Modifier
nano /etc/network/interfaces.d/10-bridges

# 3. Appliquer
ifreload -a                 # applique la différence sans couper les interfaces actives

# 4. Vérifier
ip -br a                    # plus une seconde session SSH
```

**Si je perds l'accès.** Au bout de 10 minutes, `10-bridges` est supprimé et les interfaces sont rechargées. Je me reconnecte en SSH comme avant ma modification, et je vérifie que les bridges et la tâche programmée ont disparu :

```bash
ip -br a   # plus de bridges
at -l      # aucune sortie : la tâche s'est exécutée
```

## Cas 2 : le pare-feu nftables

Ce cas servira surtout pour le durcissement de la chaîne `input`, où une règle trop stricte peut me bloquer SSH.

**Retour arrière : vider toutes les règles chargées.** La machine se retrouve sans aucune règle, donc ouverte et joignable.

```bash
# 1. Armer
echo "/usr/sbin/nft flush ruleset" | at now + 10 minutes
at -l
at -c <num>

# 2. Modifier
nano /etc/nftables.conf

# 3. Appliquer
nft -c -f /etc/nftables.conf   # test de syntaxe sans appliquer
nft -f /etc/nftables.conf      # chargement des règles

# 4. Vérifier
nft list ruleset               # plus une seconde session SSH
```

**Si je perds l'accès.** Au bout de 10 minutes, les règles sont vidées et je peux me reconnecter. Trois contreparties :

- La table NAT disparaît avec le reste : les VM perdent leur sortie vers Internet jusqu'au rechargement des règles.
- L'isolation de `vmbr2` disparaît aussi. Sans règles, la chaîne `forward` n'existe plus, donc plus de refus par défaut : le bac à sable peut joindre `vmbr1`. Sans conséquence aujourd'hui, puisqu'il n'y a aucune VM. Le jour où j'y ferai des tests destructifs, un retour arrière ouvrirait la porte vers mes services.
- `/etc/nftables.conf` n'est pas touché. Il contient encore la modification fautive : je la corrige avant de recharger ou de redémarrer.

## Choix et compromis

**`at` plutôt que `cron`.** `at` exécute une tâche une seule fois. C'est le besoin : un retour arrière ponctuel, lié à une modification précise. Avec `cron`, il faudrait une règle périodique à retirer ensuite, et le retour arrière risquerait de se rejouer et de défaire ma configuration.

**Pourquoi 10 minutes.** C'est un compromis. Trop court, je n'ai pas le temps d'éditer, d'appliquer et de vérifier, et le filet se déclenche en plein test. Trop long, je reste coupé du serveur pendant tout ce temps si ça tourne mal. Dix minutes suffisent pour les étapes 2 à 4 sans me presser.

**Une commande de retour arrière aussi simple que possible.** La tâche ne restaure pas de copie : elle supprime mon fichier, ou vide les règles. C'est pour cela que j'utilise mon propre fichier pour les interfaces.

**Si je dépasse les 10 minutes en testant.** Le filet se déclenche quand même, en pleine vérification. Je reprends alors la procédure à l'étape 1. Si je vois que je manque de temps, j'arme un nouveau filet pour 10 minutes de plus, puis j'annule l'ancien avec `at -r <num>`.

## Limites et pièges

- **Je n'arme le filet que pour ce qui peut me couper l'accès.** Tant que la chaîne `input` reste en `policy accept`, les règles `forward` et `nat` ne devraient pas pouvoir le faire, et un réglage comme `ip_forward` non plus. Ce choix sera à revoir quand je durcirai `input`.
- **Une tâche `at` peut s’exécuter en retard.** Les tâches sont stockées sur disque, donc elles survivent à un redémarrage du serveur. Si je programme une tâche alors que `atd` est inactif, elle est bien mise en file d'attente et s'exécute au démarrage du service si l'heure est passée. De même, si l'heure passe pendant que la machine est éteinte, elle s'exécute au démarrage suivant. Le retour arrière peut donc se déclencher bien après l'heure prévue, au moment où je ne l'attends pas. C'est pourquoi je vérifie `atd` avant d'armer, et que j'annule toujours le filet une fois la modification validée.
- **Ce document ne couvre pas les modifications de SSH.** Elles peuvent aussi me couper l'accès. Le principe serait le même, avec un autre retour arrière.