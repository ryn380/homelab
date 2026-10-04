# Filet de sécurité réseau

> Je travaille sur une machine louée dont je n'ai pas d'accès physique. Une mauvaise configuration peut donc me couper l'accès au serveur. Afin d'anticiper ce problème, je programme un retour arrière différé, avec les commandes `at` et `ifreload`, qui restaure la configuration initiale.

## Pourquoi ce filet

L'un des inconvénients de monter ce homelab sur une machine louée chez un fournisseur est que je n'ai pas d'accès à une console physique. Tout doit se faire à distance. Une modification de la configuration réseau (interfaces, adressage IP, etc.), de SSH ou du pare-feu peut donc me couper l'accès. Une perte d'accès m'obligerait à passer soit par le support Hetzner, soit par un reset matériel suivi d'une réparation manuelle depuis un système de secours Hetzner.

| Option | Délai | Interruption des VM | Effort |
|---|---|---|---|
| Filet `at` + `ifreload` | 10 min max, automatique | Aucune | Aucun, tout est programmé à l'avance |
| Console KVM via le support | Variable, selon la charge du support | Aucune | Se connecter à la console et corriger la configuration à la main |
| Système de secours + Reset | Le temps d'un redémarrage | Oui, toutes les VM | Monter les disques depuis un système minimal et réparer à la main |

Le filet de sécurité me permettra de récupérer l'accès en moins de 10 minutes, sans redémarrage, alors que les deux autres options dépendent soit du support, soit d'un redémarrage.

## Le principe

Avant toute modification susceptible de me couper l'accès au serveur, j'arme un retour arrière différé.

Dans le cas d'une modification des interfaces pour l'ajout de bridges internes par exemple, je programme la suppression du fichier de configuration et le rechargement de la configuration des interfaces. Ces deux actions se déclenchent 10 minutes après leur programmation.

Une fois le retour arrière armé, je crée ce fichier de configuration et j'applique mes modifications en rechargeant la configuration des interfaces réseau. Comme je crée mon propre fichier, je n'ai rien à sauvegarder : il suffit de le supprimer.

Je vérifie ensuite que la configuration a bien été prise en compte. Si tout fonctionne, j'annule le retour arrière. Sinon, le fichier que je viens de créer est supprimé et mes modifications avec lui, et la configuration des interfaces est rechargée.

J'arme le retour arrière avant de créer le fichier. Comme ça, j'évite de laisser un fichier actif et non protégé, qui serait chargé au prochain redémarrage de la machine. **Ordre à retenir** : armer, modifier, appliquer, vérifier, puis annuler le filet.