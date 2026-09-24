# Main Levée — données : modèle et export

## Où vivent les données

Il n'y a **pas de back-end applicatif** dans ce projet. `index.html` contient toute l'application ;
ce qui tient lieu de serveur est **Firebase** : Firestore pour les documents JSON, Firebase Auth
(mode anonyme) pour identifier chaque visiteur, atteints via le SDK compat chargé en CDN dans
`index.html` et configurés par la constante `FIREBASE_CONFIG`.

Conséquences pratiques :

- un seul projet Firebase pour toute l'appli, à créer une fois dans la console Firebase ;
- les règles d'accès sont déclarées dans [`firestore.rules`](firestore.rules), à coller dans la
  console (ou déployer via `firebase deploy --only firestore:rules`) ;
- toute partie passe par Firebase : sans connexion à la base, on ne peut ni organiser ni jouer.
  `localStorage` ne garde que des préférences locales (brouillon de l'éditeur, nom, icône, thème) ;
- **choix assumé : pas de compte organisateur.** N'importe quel visiteur authentifié anonymement
  (donc n'importe qui ouvrant la page) peut ouvrir un salon. Adapté à un usage entre personnes de
  confiance — voir `firestore.rules` pour durcir si besoin.

Front hébergé sur GitHub Pages, base sur Firebase (projet Google Cloud séparé).

## Tester en local, sans toucher à la base réelle

Servie depuis `localhost` ou `127.0.0.1`, la page se branche d'elle-même sur les émulateurs
Firebase (Auth sur 9099, Firestore sur 8080) au lieu du vrai projet. Il faut Java 21 ou plus. Pas à pas
complet : [EMULATEUR.md](EMULATEUR.md).

```bash
npx firebase-tools emulators:start --only auth,firestore,hosting
```

- page : <http://localhost:5000>, interface des émulateurs : <http://localhost:4000> ;
- les émulateurs appliquent `firestore.rules` : les tests vérifient aussi les règles ;
- en local, chaque onglet a son propre compte anonyme : un onglet organisateur et plusieurs
  onglets joueurs simulent une vraie partie. Recharger un onglet en fait un nouveau joueur ;
- les données de l'émulateur disparaissent à l'arrêt.

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

- L'id du document est l'`uid` Firebase Auth du visiteur (connexion anonyme), stable pour ce
  navigateur tant que ses données de site ne sont pas effacées — mais recréé si le visiteur revient
  depuis un autre navigateur ou après avoir vidé ses cookies.
- `name` : le prénom saisi sur l'écran du nom, au plus 24 caractères.
- `icon` : index dans le tableau `AVATARS` de `index.html`, de 0 à 9.
- `score` : cumul de la partie. Une bonne réponse vaut de 500 à 1 000 points selon la rapidité
  (`1000 × (0,5 + 0,5 × temps_restant / durée)`, arrondi à la dizaine), une erreur ou une expiration
  vaut 0.
- `answeredIndex` : dernière question répondue ; -1 tant que le joueur n'a pas répondu.
- `choice` : index de la réponse cochée dans `a`, ou -1.

Chaque participant n'écrit que son propre document (`joinGame` et `writeMyProgress`).

## Règles d'accès

Déclarées dans [`firestore.rules`](firestore.rules) :

- `quiz/current` et `game/state` : lecture et écriture pour tout visiteur authentifié (anonyme
  compris) — pas de compte organisateur distinct.
- `players/{id}` : lecture pour tout visiteur authentifié, écriture réservée au propriétaire du
  document (`request.auth.uid == id`), c'est-à-dire chaque joueur pour lui-même.

## Refaire l'export

Depuis la console Firebase (Firestore > onglet Données), ou via le CLI :

```bash
firebase firestore:export gs://<bucket-export> --collection-ids=quiz,game,players
```

(nécessite le plan Blaze pour l'export vers Cloud Storage ; pour un export ponctuel en JSON, il est
plus simple de copier les documents à la main depuis la console).

Arborescence produite dans `db-export/` (manuelle, pas générée automatiquement par la commande
ci-dessus) :

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
