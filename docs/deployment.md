# Déploiement : MongoDB Atlas + Render (API) + Vercel (client)

**État** : le relais Vercel est configuré pour `https://sunuadmin2.onrender.com`. À réserver à une démonstration avec des données fictives. Les offres, limites et prix des services changent : vérifiez-les sur leurs sites.

```
Navigateur -> Vercel (fichiers du client + relais /api) -> Render (API Express) -> Atlas (MongoDB)
                                                                      \-> stockage S3 (pièces)
```
Le relais `/api` de Vercel garde client et API sur la **même origine** : le cookie de connexion et la protection CSRF fonctionnent comme en local.

## Étapes
1. **Atlas** : créer un cluster gratuit (M0), un utilisateur de base avec un mot de passe fort, puis dans *Network Access* autoriser Render. Les adresses de sortie du plan gratuit de Render ne sont pas fixes : sans plage fixe confirmée dans la documentation Render, il faut `0.0.0.0/0` (risque accepté, d'où le mot de passe fort). Récupérer l'URI `mongodb+srv://...` et ajouter le nom de base : `.../sunuadmin?retryWrites=true&w=majority`.
2. **Pièces justificatives** (au choix) :
   - *Stockage S3* (recommandé) : créer un bucket **privé** chez un fournisseur compatible S3, avec une clé d'accès limitée à ce bucket ; renseigner `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, et `S3_ENDPOINT` (hors AWS) ou `S3_REGION` (AWS).
   - *Disque Render payant* : voir le bloc commenté de `render.yaml` et `UPLOAD_DIR=/data/uploads`.
   - *Rien* : les pièces disparaissent à chaque redémarrage (démonstration uniquement).
3. **GitHub** : pousser le dépôt (`git status` ne doit lister aucun `.env`).
4. **Render** : déployer l'API depuis `render.yaml`. Renseigner `MONGODB_URI` et les variables `S3_*` si le stockage objet est utilisé. `JWT_SECRET` est généré. Vérifier `https://sunuadmin2.onrender.com/api/v1/health` : la réponse doit contenir `"db": true`.
5. **Vercel** : dans [votre tableau de bord](https://vercel.com/souleymane312s-projects), choisir *Add New > Project*, importer `souleymane312/sunuadmin2`, puis définir **Root Directory = `client`**, preset Vite, commande de build `npm run build` et dossier de sortie `dist`. Aucun secret ni variable d'environnement client n'est nécessaire. Déployer.
6. **Retour sur Render** : mettre `CLIENT_ORIGIN` à l'URL de production exacte fournie par Vercel (par exemple `https://<votre-site>.vercel.app`, sans `/` final) ; le service redémarre.
7. **Premier administrateur**, depuis votre PC, contre la base Atlas :
```powershell
cd server
$env:MONGODB_URI = Read-Host "URI Atlas"
$env:ADMIN_EMAIL = Read-Host "E-mail"
$env:ADMIN_PASSWORD = Read-Host "Mot de passe (12 caracteres minimum)"
npm run create-admin
Remove-Item Env:ADMIN_PASSWORD, Env:MONGODB_URI
```

## Vérifications après déploiement
- `https://<site-vercel>/api/v1/health` : `db: true`, et `ip` = **votre** adresse publique (sinon régler `TRUST_PROXY` : sans cela, la limitation de débit voit une seule adresse pour tous).
- Inscription, connexion, rechargement de la page (la session doit tenir), déconnexion.
- Envoi d'un PDF de 3 Mo puis d'un PDF de 4 Mo : la limite est de 4 Mo car Vercel limite le corps des requêtes de ses fonctions à 4,5 Mo ; non vérifié pour les relais vers un serveur externe.
- Faire un redéploiement manuel sur Render, puis télécharger la pièce : elle doit être toujours là.

## Pièges connus
- **Render gratuit s'endort** après 15 minutes sans trafic ; le réveil peut dépasser les 30 secondes de délai des relais Vercel, la première requête peut donc échouer. Un plan payant qui ne s'endort pas règle le problème.
- Atlas M0 : 512 Mo et 500 connexions ; vérifier les sauvegardes réellement proposées sur le plan gratuit.
- Le CORS et le cookie sont réglés pour la même origine ; un appel direct du client vers l'API (sans relais) demanderait `SameSite=None` et serait bloqué par certains navigateurs.

## Variables du serveur
`MONGODB_URI`, `JWT_SECRET` (32 caractères minimum, sinon l'API refuse de démarrer), `CLIENT_ORIGIN`, `NODE_ENV=production`, `TRUST_PROXY`, `PORT` (fourni par Render), `UPLOAD_DIR`, `S3_*`. Aucun secret côté client.

## Sauvegarde et restauration
```powershell
mongodump --uri "<URI Atlas>" --archive=sunuadmin.archive --gzip
mongorestore --uri "<URI Atlas>" --archive=sunuadmin.archive --gzip --drop     # écrase la base
```
Sauvegarder aussi le bucket S3 (ou `UPLOAD_DIR`) : les pièces ne sont pas dans MongoDB. Tester une restauration.
