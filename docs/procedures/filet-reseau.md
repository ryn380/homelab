# Filet de sécurité réseau

> Je travaille sur une machine louée dont je n'ai pas d'accès physique. Une mauvaise configuration peut donc me couper l'accès au serveur. Afin d'anticiper ce problème, je programme un retour arrière différé, avec les commandes `at` et `ifreload`, qui restaure la configuration initiale.

## Pourquoi ce filet

L'un des inconvénients de monter ce homelab sur une machine louée chez un fournisseur est que je n'ai pas d'accès à une console physique. Tout doit se faire à distance. Une modification de la configuration réseau (interfaces, adressage IP, etc.), de SSH ou du pare-feu peut donc me couper l'accès. Une perte d'accès m'obligerait à passer soit par le support Hetzner, soit par un reset matériel qui couperait toutes mes VM, suivi d'une réparation manuelle depuis un système de secours Hetzner pour accéder à une console de secours. 

| Option | Délai | Interruption | Effort |
|---|---|---|---|
| Filet `at` + `ifreload` | 10 min max | Aucune | Console KVM via le support | | |
| Système de secours + Reset | | |

Le filet de sécurité me permettra de récupérer l'accès en moins de 10 minutes, sans redémarrage et sans dépendre de personne.