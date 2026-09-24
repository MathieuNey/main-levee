# Lancer l'émulateur Firebase de Main Levée

## À quoi sert l'émulateur

L'émulateur fait tourner Firebase (Auth, Firestore, Hosting) sur ton poste : tu joues une vraie partie
sans jamais toucher à la base réelle.

- Servie depuis `127.0.0.1` ou `localhost`, la page se branche d'elle-même sur l'émulateur. Sur
  l'hébergement réel, rien ne change.
- Les règles de `firestore.rules` s'appliquent : un test vérifie aussi les droits d'accès.
- Chaque onglet a son propre compte anonyme : un onglet organisateur et plusieurs onglets joueurs
  simulent une vraie partie.
- Les données disparaissent à l'arrêt de l'émulateur.

## Prérequis

Il faut quatre choses sur la machine ; sur ton poste, tout est déjà en place.

| Élément | Détail | Comment vérifier |
| --- | --- | --- |
| Node.js | version récente (installée : 26) | `node --version` |
| Java | version 21 ou plus, ici dans `C:\Users\mney\tools\jdk-21` | `C:\Users\mney\tools\jdk-21\bin\java -version` |
| Ports libres | 4000, 4400, 5000, 8080, 9099 | voir la section [Arrêter et dépanner](#arrêter-et-dépanner) |
| Internet | seulement au premier lancement, pour télécharger les émulateurs (déjà en cache) | — |

Le projet contient déjà la configuration nécessaire : les ports dans `firebase.json`, le projet dans
`.firebaserc`. Aucun compte Firebase ni `firebase login` n'est requis pour l'émulateur.

## Installer Java 21 sans droits administrateur

La version portable de Microsoft s'installe par simple décompression, sans invite administrateur. Sur
ton poste, c'est déjà fait.

1. Télécharger le zip [Microsoft Build of OpenJDK 21](https://aka.ms/download-jdk/microsoft-jdk-21-windows-x64.zip)
   (environ 190 Mo).
2. Le décompresser, puis renommer le dossier obtenu en `jdk-21` et le placer dans `C:\Users\mney\tools\`.
3. Vérifier que `C:\Users\mney\tools\jdk-21\bin\java.exe` existe, puis lancer
   `C:\Users\mney\tools\jdk-21\bin\java -version` : la réponse doit commencer par `openjdk version "21`.

Ailleurs que dans ce dossier, il faut adapter le chemin dans les commandes ci-dessous et dans
`.claude/launch.json`.

## Lancer les émulateurs

Une seule commande, dans un terminal PowerShell ouvert dans le dossier du projet. Elle met Java dans le
`PATH` de ce terminal seulement, sans toucher à ta configuration Windows.

1. Ouvrir PowerShell dans le dossier `main-levee`.
2. Coller la commande :

   ```powershell
   $env:JAVA_HOME = "C:\Users\mney\tools\jdk-21"; $env:Path = "$env:JAVA_HOME\bin;$env:Path"; npx firebase-tools emulators:start --only auth,firestore,hosting
   ```

3. Attendre le message `All emulators ready! It is now safe to connect your app.`, en général en 10 à
   20 secondes.
4. Laisser ce terminal ouvert : fermer le terminal arrête les émulateurs.

| Adresse | Ce qu'on y trouve |
| --- | --- |
| http://127.0.0.1:5000 | la page Main Levée, branchée sur l'émulateur |
| http://127.0.0.1:4000 | l'interface des émulateurs : comptes créés, documents de la base |

Pour vérifier que la page parle bien à l'émulateur : ouvrir la console du navigateur (F12). Le message
`Main Levée : émulateurs Firebase (auth :9099, firestore :8080)` doit s'afficher.

Depuis Claude, pas besoin de terminal : la configuration `emulateurs` de `.claude/launch.json` lance la
même chose.

## Tester une partie

Un onglet par personne suffit : chaque onglet est un joueur distinct.

1. Onglet 1, l'organisateur : ouvrir http://127.0.0.1:5000, cliquer sur « Organiser un quiz », puis sur
   « Lancer la partie → ».
2. Onglets 2 et 3, les joueurs : ouvrir la même adresse, cliquer sur « Participer au quiz » et saisir un
   nom différent dans chacun.
3. Revenir sur l'organisateur : les joueurs apparaissent dans le salon. Cliquer sur « Démarrer avec
   2 joueurs → ».
4. Répondre depuis chaque onglet joueur, puis faire avancer la partie depuis l'organisateur jusqu'au
   classement final.

Points d'attention :

- Un joueur qui arrive avant l'ouverture du salon voit « Aucune partie ouverte ». Il entre tout seul dans
  la partie dès que l'organisateur ouvre le salon.
- Recharger un onglet joueur en fait un nouveau joueur : l'ancien reste inscrit.
- Le nom proposé est celui du dernier joueur saisi dans ce navigateur : le remplacer dans chaque onglet.
- Le contenu de la base se lit en direct sur http://127.0.0.1:4000/firestore (`game/state`, `players`,
  `quiz/current`).

## Arrêter et dépanner

Pour arrêter : **Ctrl+C** dans le terminal des émulateurs, puis attendre la fin de l'arrêt. Les données
de test disparaissent.

| Symptôme | Cause | Solution |
| --- | --- | --- |
| `Could not spawn java -version` ou `java` introuvable | Java n'est pas dans le `PATH` du terminal | Relancer la commande complète, avec les deux `$env:` |
| `Port 8080 is not open` ou un port déjà utilisé | Un émulateur précédent tourne encore | Voir les commandes ci-dessous |
| La page ne charge pas sur le port 5000 | Les émulateurs ne sont pas encore prêts | Attendre `All emulators ready!` puis recharger |
| Pas de message « émulateurs Firebase » dans la console | La page n'est pas servie depuis `127.0.0.1` ou `localhost` | Passer par http://127.0.0.1:5000, pas par le fichier ouvert directement |
| Les joueurs ne voient pas la partie | La page a été ouverte avant le lancement des émulateurs | Recharger les onglets |

Pour trouver et arrêter un émulateur resté en mémoire, dans PowerShell :

```powershell
Get-NetTCPConnection -State Listen -LocalPort 4000,4400,5000,8080,9099 | Select-Object LocalPort, OwningProcess
```

Puis arrêter chaque processus listé (normalement `node` et `java`) avec `Stop-Process -Id <numéro>`.

Les journaux `firestore-debug.log` et `firebase-debug.log` apparaissent dans le dossier du projet : ils
sont ignorés par git et par le déploiement.
