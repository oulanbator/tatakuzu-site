# tatakuzu-site

Site vitrine + fichier `app-ads.txt` de **Tatakuzu** (jeu Binairo, Mukeka Studio).

Hébergé sur **Firebase Hosting** → en ligne à https://tatakuzu.web.app

## À quoi sert ce repo (contexte)

Quatre rôles, un seul dossier :

1. **`public/app-ads.txt`** — obligatoire pour **AdMob**. AdMob lit ce fichier pour vérifier
   qu'on est autorisé à monétiser l'app. Le domaine `tatakuzu.web.app` est déclaré comme
   « Site Web » dans la fiche Play Store → AdMob crawle ce domaine → trouve `/app-ads.txt`.
   **Ne jamais supprimer ni renommer ce fichier**, sinon la diffusion des pubs casse.

2. **`public/index.html`** — landing page du jeu (ce même domaine est affiché publiquement
   sur la fiche Play Store, donc un joueur peut cliquer et atterrir ici).

3. **`public/d.html`** — page de repli des **liens de défi** (duel entre amis, T42 du jeu).
   Un lien de duel a la forme `https://tatakuzu.web.app/d#<données>`. Si Tatakuzu est
   installé, Android intercepte le lien et cette page n'est jamais vue ; sinon elle
   s'affiche et invite à installer le jeu. `firebase.json` la sert sur `/d` via une
   *rewrite* (sans quoi `/d` renverrait 404).

4. **`public/.well-known/assetlinks.json`** — déclare que ce domaine autorise
   `com.mukeka.tatakuzu` à ouvrir ses liens (**Android App Links**). Sans ce fichier,
   Android affiche un sélecteur d'app au lieu d'ouvrir Tatakuzu — **sans message
   d'erreur**. L'empreinte SHA-256 qu'il contient est celle du **certificat de signature
   d'application** (celui de Google, Play App Signing), *pas* celle du keystore d'upload.

> ⚠️ Ce repo ne contient QUE le site. Le code du jeu (projet Flutter) est un repo séparé.
> Le keystore Android n'a rien à faire ici (voir `.gitignore`).

## Structure

```
tatakuzu-site/
├── public/
│   ├── index.html      ← landing page
│   ├── d.html          ← page de repli des liens de défi, servie sur /d
│   ├── app-ads.txt     ← validation AdMob (NE PAS TOUCHER sans raison)
│   └── .well-known/
│       └── assetlinks.json  ← Android App Links (NE PAS TOUCHER sans raison)
├── firebase.json       ← config hosting (public, ignore, rewrites, headers)
├── .firebaserc         ← lien vers le projet Firebase
├── .gitignore
└── README.md
```

## Mettre à jour le site en ligne (le cas courant)

Après avoir modifié `index.html` ou `app-ads.txt` :

```bash
firebase deploy --only hosting
```

Vérifier ensuite dans le navigateur :
- Page : https://tatakuzu.web.app
- Fichier AdMob : https://tatakuzu.web.app/app-ads.txt
- Page de duel : https://tatakuzu.web.app/d
- App Links : https://tatakuzu.web.app/.well-known/assetlinks.json

Les quatre doivent répondre **200**. En particulier `assetlinks.json` : voir le piège
ci-dessous, il est déjà tombé une fois sur ce projet.

### ⚠️ Le piège du `ignore` (App Links)

Le `firebase.json` généré par défaut contient `"ignore": ["**/.*"]`, ce qui **exclut du
déploiement tout fichier ou dossier commençant par un point** — donc `.well-known/`.
Résultat : `assetlinks.json` renvoie 404 en ligne alors qu'il est bien dans le repo, et la
vérification App Links échoue **en silence**.

Ce motif a donc été remplacé par `"**/.DS_Store"` (l'intention réelle) ; **ne pas le
remettre**. Firebase ne sait pas faire de négation (`!.well-known/**`) dans `ignore`.

Autre piège adjacent : **ne pas** utiliser `"appAssociation": "AUTO"`. Ce mécanisme
reposait sur Firebase Dynamic Links, **arrêté en août 2025**. On sert le fichier à la main.

## Repartir de zéro sur une NOUVELLE machine

Firebase Hosting ne se « clone » pas : la source vit ici, sur Git. Sur un nouvel ordi :

```bash
# 1. Récupérer la source
git clone https://github.com/oulanbator/tatakuzu-site.git
cd tatakuzu-site

# 2. Installer les outils Firebase (une fois par machine)
npm install -g firebase-tools
firebase login

# 3. Déployer
firebase deploy --only hosting
```

Si `firebase deploy` se plaint de ne pas savoir quel projet utiliser :
```bash
firebase use --add        # sélectionner le projet Firebase de Tatakuzu
```

## Workflow Git habituel

```bash
git add .
git status                # TOUJOURS vérifier ce qui part avant de committer
git commit -m "Message"
git push
```

## Authentification GitHub (rappel Windows)

GitHub n'accepte plus le mot de passe de compte pour `git push`. Le plus simple, une fois
par machine :

```bash
winget install --id GitHub.cli
gh auth login             # GitHub.com → HTTPS → Login with a web browser
```

Après ça, `git push` fonctionne sans rien redemander.

## Rappels importants (à ne pas re-découvrir dans la douleur)

- **Jamais de secret dans ce repo** : keystore (`.jks`), `key.properties`, clés API…
  Ils sont exclus par `.gitignore`. Un secret poussé sur Git = à considérer comme
  compromis (le supprimer ne suffit pas, il faut le régénérer/réinitialiser).
- **Toujours poser le `.gitignore` AVANT le premier `git add`** sur un nouveau dépôt.
- Après un changement de `app-ads.txt`, AdMob peut mettre **jusqu'à 24 h** à re-crawler.
  Forcer la vérif : AdMob → l'app → « Valider l'application » → « Rechercher des mises à jour ».
- Le keystore Android (le vrai actif critique) se sauvegarde **ailleurs** que sur cette
  machine et hors de Git. Perte du keystore = plus aucune mise à jour possible sur le Play Store
  (sauf réinitialisation de l'upload key si Play App Signing est activé).