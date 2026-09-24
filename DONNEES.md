# Main Levée — données : modèle et export

## Où vivent les données

Il n'y a **pas de back-end applicatif** dans ce projet. `index.html` contient toute l'application ;
ce qui tient lieu de serveur est la capability `db` de la plateforme Artifact : un magasin de
documents JSON hébergé par claude.ai, atteint à l'exécution par `claude.use("db")`.

Conséquences pratiques :

- une base par artifact, créée à la première écriture, supprimée avec l'artifact ;
- pas de schéma SQL, pas de fichier de base à télécharger, pas de serveur à démarrer ;
- les règles d'accès sont déclarées au moment de la publication (paramètre `capabilities`), pas dans
  du code ;
- ouverte hors artifact (fichier local, aperçu `data:`), la page ne trouve pas `db` et retombe en
  mode solo avec `localStorage`. Les données ci-dessous n'existent alors pas.

Artifact concerné : https://claude.ai/artifact/3tYZHT1x6SA4Eap3eLXT6L

## Modèle de données

Trois chemins, tous à la racine de la base.

### `quiz/current` — le quiz joué

```json
{
  "title": "Culture générale du vendredi",
  "duration": 15,
  "questions": [
    { "q": "Quelle est la capitale de l’Australie ?",
      "a": ["Sydney", "Melbourne", "Canberra", "Perth"],
      "correct": 2 }
  ]
}
```

- `title` : titre du quiz, au plus 60 caractères ; chaîne vide si l'organisateur n'en a pas saisi.
- `duration` : secondes par question, entre 5 et 120.
- `demo` : `true` si la partie commence par la question test (entraînement sans point). Absent ou
  `true` par défaut ; `false` quand l'organisateur a décoché l'option.
- `a` : toujours quatre réponses, dans l'ordre des symboles de cartes (pique, cœur, trèfle, carreau).
- `correct` : index dans `a`, de 0 à 3.

Écrit par l'organisateur à l'ouverture du salon (`openLobby` dans `index.html`). C'est une copie du
brouillon de l'éditeur, lequel vit dans le `localStorage` du navigateur de l'organisateur et n'est
donc **pas** dans cette base.

### `game/state` — l'état de la partie

```json
{
  "gameId": "g1790093081274",
  "phase": "ended",
  "index": 2,
  "startedAt": 1790093123995,
  "stopped": false
}
```

- `gameId` : `"g"` suivi de l'horodatage de création ; change à chaque nouvelle partie et sert à
  distinguer les joueurs de la partie en cours des reliquats de la précédente.
- `phase` : `lobby` (salon ouvert), `intro` (interstitiel annonçant le titre), `question` (temps de
  réponse), `reveal` (bonne réponse affichée), `ended` (terminée).
- `index` : question en cours, à partir de 0 ; vaut -1 dans le salon.
- `startedAt` : départ de la question en millisecondes epoch, qui sert à synchroniser le décompte de
  tout le monde et à mesurer la rapidité des réponses.
- `stopped` : `true` quand l'organisateur a arrêté la partie avant la fin.

Écrit uniquement par l'organisateur.

### `players/<id du viewer>` — un document par joueur

```json
{
  "gameId": "g1790093081274",
  "name": "Mat",
  "icon": 4,
  "score": 2680,
  "answeredIndex": 2,
  "choice": 0
}
```

- L'id du document est l'identifiant **opaque** du viewer renvoyé par `user.id()` (forme `u_…`),
  stable par personne et par organisation.
- `name` : le prénom saisi sur l'écran du nom, au plus 24 caractères.
- `icon` : index dans le tableau `AVATARS` de `index.html`, de 0 à 9.
- `score` : cumul de la partie. Une bonne réponse vaut de 500 à 1 000 points selon la rapidité
  (`1000 × (0,5 + 0,5 × temps_restant / durée)`, arrondi à la dizaine), une erreur ou une expiration
  vaut 0.
- `answeredIndex` : dernière question répondue ; -1 tant que le joueur n'a pas répondu.
- `choice` : index de la réponse cochée dans `a`, ou -1.

Chaque participant n'écrit que son propre document (`joinGame` et `writeMyProgress`).

## Règles d'accès

Déclarées à la publication :

- `quiz` et `game` : lecture par tous les lecteurs, écriture réservée aux éditeurs (`admin`).
- `players` : lecture par tous, écriture réservée aux éditeurs…
- …sauf `players/{self}`, où chaque participant écrit son propre document (niveau `interact`).

Un membre de l'organisation partagé en « Can view » n'atteint pas le niveau `interact` : il peut lire
la partie mais pas s'inscrire. Le bon réglage de partage est « Can interact ».

## Refaire l'export

L'export se demande depuis une session Claude Code ayant accès à l'artifact, avec l'outil
ArtifactData (action `list`, une collection à la fois, avec un dossier de sortie). Formulation
suffisante :

> Exporte les collections `quiz`, `game` et `players` de l'artifact
> https://claude.ai/artifact/3tYZHT1x6SA4Eap3eLXT6L dans `db-export/`

Arborescence produite :

```
db-export/
  quiz/current.json
  game/state.json
  players/<id du viewer>.json      (un fichier par joueur)
```

Les exports faits depuis l'éditeur (bouton « Export ») sont nommés `main-levee-AAAA-MM-JJ.json` et
contiennent le même objet que `quiz/current`.

C'est un **instantané**, pas une synchronisation : le contenu reflète l'état au moment de
l'extraction et ne se met pas à jour tout seul. Refaire l'export pendant une partie donne des scores
partiels.

## Ce que l'export ne contient pas

- **Aucun historique de parties.** Chaque nouvelle partie écrase `quiz/current` et `game/state`, et
  réutilise les documents joueurs (clé = identifiant du viewer). Pour garder le classement d'une
  session, exporter avant de lancer la suivante.
- **Aucune donnée personnelle** au-delà du prénom saisi par le joueur et de son identifiant opaque :
  ni e-mail, ni nom d'annuaire, ni horodatage de connexion.
- **Pas les questions en cours d'édition** : le brouillon de l'organisateur reste dans le
  `localStorage` de son navigateur tant qu'il n'a pas lancé de partie.
