# mhikes-basecamp 🏔️

Basecamp: the authorized SSH keys for mhikes infra — your pass to the summit.

Ce dépôt contient la liste des clés publiques SSH autorisées à se connecter
aux nœuds d'infrastructure mhikes/EasyMountain (cluster Docker Swarm sur
AWS EC2). Il est consulté **dynamiquement** par chaque serveur à chaque
tentative de connexion SSH, via le mécanisme `AuthorizedKeysCommand` de
sshd — aucun redéploiement n'est nécessaire pour ajouter ou révoquer un
accès.

## Pourquoi ce dépôt est public

Le contenu de ce dépôt (des clés **publiques** SSH) n'est pas sensible par
nature — une clé publique est faite pour être partagée, exactement comme
le fait déjà GitHub via `github.com/<login>.keys`. La visibilité publique
permet aux serveurs de récupérer le fichier sans authentification
(`raw.githubusercontent.com`), simplifiant le mécanisme.

**Ne jamais committer de clé privée dans ce dépôt.** Une validation
automatique (voir plus bas) bloque toute PR contenant du matériel privé,
mais elle ne remplace pas la vigilance de chacun.

## Comment ça fonctionne

1. Un développeur se connecte en SSH à un nœud (utilisateur partagé `ubuntu`)
2. sshd exécute `/usr/local/bin/basecamp-authorized-keys.sh`, qui récupère
   le fichier [`authorized_keys_list`](./authorized_keys_list) de ce dépôt
3. sshd compare la clé publique présentée par le client à cette liste
4. Si elle correspond, et que le client prouve la possession de la clé
   privée associée, la connexion est acceptée

Un cache local (`/var/cache/authorized_keys_list`) sert de secours sur
chaque nœud si GitHub est temporairement injoignable.

L'accès CI/CD (déploiements automatisés) ne passe **pas** par ce
mécanisme : il utilise la paire de clés EC2 classique, injectée au
provisioning et indépendante de ce dépôt.

## Format du fichier `authorized_keys_list`

Une clé publique par ligne, précédée d'un commentaire identifiant son
propriétaire :

```
# David
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... david-mhikes-basecamp

# Francois
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... francois-mhikes-basecamp
```

- Les lignes vides et celles commençant par `#` sont ignorées par le
  script de résolution — utilisez-les librement pour annoter ou
  désactiver temporairement une clé (en la commentant).
- Le commentaire en fin de ligne de clé (ex. `francois-mhikes-basecamp`)
  n'a pas de valeur fonctionnelle pour sshd, mais facilite l'identification
  dans les logs de connexion.
- Le commentaire précédant la clé (ex. `# Francois`) ne doit pas contenir
  d'accents, afin de rester cohérent avec le commentaire de fin de ligne
  de la clé (`francois-mhikes-basecamp`) — cette cohérence est vérifiée
  par le CI.
- Les entrées (commentaire + clé) doivent être triées par ordre alphabétique
  du prénom du propriétaire — cet ordre est vérifié par le CI.

## Ajouter sa clé (nouveau membre de l'équipe)

1. Générer une paire de clés **dédiée à cet usage** (ne pas réutiliser une
   clé personnelle existante) :

   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/prenom-mhikes-basecamp -C "prenom-mhikes-basecamp"
   ```

2. Ouvrir une pull request ajoutant votre clé **publique**
   (`~/.ssh/prenom-mhikes-basecamp.pub`) au fichier `authorized_keys_list`,
   avec un commentaire `# prenom` au-dessus.
3. Une fois la PR mergée (après review), l'accès est effectif à la
   prochaine tentative de connexion — aucun redéploiement nécessaire.
4. Configurer votre client SSH (`~/.ssh/config`) pour utiliser cette clé
   dédiée sur les hôtes concernés :

   ```
   Host node1.mhikes.internal
       User ubuntu
       IdentityFile ~/.ssh/prenom-mhikes-basecamp
   ```

## Révoquer un accès (départ d'un membre de l'équipe)

Ouvrir une PR qui supprime (ou commente) la ligne correspondante dans
`authorized_keys_list`, et la merger. L'accès est révoqué à la prochaine
tentative de connexion sur chaque nœud — aucune action manuelle sur les
serveurs n'est nécessaire.

## Validation automatique

Chaque PR modifiant `authorized_keys_list` déclenche un contrôle CI qui :

- rejette toute présence du texte `PRIVATE KEY` dans le fichier ;
- vérifie que chaque ligne non vide et non commentée est une clé publique
  syntaxiquement valide (`ssh-keygen -lf`) ;
- rejette toute clé publique dupliquée ;
- vérifie que les entrées sont triées par ordre alphabétique du prénom ;
- vérifie que le commentaire précédant chaque clé correspond au commentaire
  de fin de ligne de cette clé.

Ce contrôle est un check obligatoire avant merge sur `main`.

## Sécurité du dépôt

- Toute modification passe par une pull request avec review obligatoire
  (protection de branche sur `main`).
- L'accès en écriture est limité aux membres actuels de l'équipe `ops-team`.
- Le dépôt est public en lecture, mais ce n'est pas un projet open source :
  aucune licence n'est attribuée, la réutilisation du contenu n'est pas
  autorisée.
