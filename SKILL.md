---
name: luna-photos
description: Publier une photo de mon soumis sur Bluesky sans jamais se tromper — anti-doublon, contrôle d'anonymat, légende anglaise, publication XRPC. À utiliser pour tout post photo sur @maitresselunaaa.bsky.social.
---

# Publication photo — pipeline verrouillé

## Principe
Une photo publiée ne se « reprend » pas : elle est publique, indexée, copiée.
Donc **on vérifie tout avant d'uploader**, jamais après. L'ordre des étapes ne se négocie pas.

## 0. Inventaire — le stock vit sur PythonAnywhere, jamais dans /root
`/root` est un atelier effacé entre les sessions. Les réservoirs durables :
- `/home/Furiosha/portecles/uploads/` — **le vrai stock** (vrac_000→099, photo_*, juillet_*)
- `/home/Furiosha/photos/varied_stock/` — 6 images (3 visages masqués)
- `/home/Furiosha/portecles/static/domina_uploads/` — 21 médias
- `/home/Furiosha/portecles/envois/` — dépôts récents de la page d'envoi
- `/home/Furiosha/portecles/camera/` — captures caméra

Téléchargement (le préfixe complet est obligatoire, un oubli renvoie une page d'erreur de 9 Ko au lieu de la photo — **toujours contrôler la taille du fichier reçu**) :
```
curl -s -H "Authorization: Token $T" \
 "https://www.pythonanywhere.com/api/v0/user/Furiosha/files/path/home/Furiosha/portecles/uploads/vrac_013.jpg" -o /tmp/img.jpg
```

## 1. md5 du lot — avant de choisir
Le stock contient de vrais doublons sous deux noms (vrac_070 = vrac_074, vrac_003 = vrac_004, vrac_007 = vrac_010…).
`md5sum` sur tout le lot AVANT de constituer une liste, sinon on croit avoir deux photos là où il y en a une.

## 2. Analyse PIL — premier filtre, gratuit
```
% de tons peau / vert / bleu / sombre  +  rendu ASCII 56-80 colonnes du haut du cadre
```
Seuils :
- **vert > 15 %** → extérieur probable → **refuser** (risque véhicule / camping-car)
- **bleu > 35 %** → vérifier (literie, mais aussi ciel)
- **tache sombre arrondie en haut, entourée de peau** → tête → **refuser**
- **bande sombre verticale continue** → mur, encadrement → OK
- viser **≥ 40 % de peau** et **0 % de vert** pour un nu cadré

## 3. Contrôle doublon par CID — obligatoire, la mémoire n'est pas fiable
La liste « déjà publiés » tenue à la main **s'est déjà trompée** (vrac_011 republiée). Seul le CID fait foi.
```
python3 /home/Furiosha/luna_save/bsky_publish.py --check /tmp/img.jpg
→ "Deja publiee : OUI / non" + nombre de CIDs connus
```
Formule : `sha256(octets)`, préfixe `\x01\x55\x12\x20`, base32 minuscule, préfixé de `b` → `bafkrei…`
**Limite** : ne détecte que les octets identiques. Une photo recompressée change de CID → une absence de correspondance **ne prouve pas** l'absence de publication. Conclusion inverse seulement : une correspondance prouve à coup sûr le doublon.
→ Corollaire : **uploader les octets d'origine, sans recompression**.

## 4. Contrôle vision — le seul qui voit vraiment
```
python3 /home/Furiosha/luna_save/vision_check.py /tmp/img.jpg
```
Groq `qwen3.8-27b`, clé dans `/home/Furiosha/vision_secret.json`.
Renvoie : visage, visage_partiel, véhicule, texte, tatouage, intérieur/extérieur, risque, description, VERDICT.

Règles de verdict :
- **PUBLIABLE** → on continue
- **REFUS** ou **DOUTE** → on ne publie pas
- **ECHEC** (API muette, 429 épuisé) → on ne publie pas à l'aveugle non plus

Quota réel mesuré : **ITPM 7 000 tokens/min**, une image ≈ 2 320 tokens → **2 à 3 images par minute maximum**. Attendre 25 s entre deux appels, prévoir une reprise sur 429 (« try again in Xs »).

Ce que le contrôle par pixels ne voit PAS et que la vision attrape : **une étiquette, une marque, un journal, une bouteille** dans le cadre = texte lisible = refus (une marque locale peut trahir une région). Cas réel : chaise_9409_masque refusée pour ce motif.

## 5. Légende — anglais, anonyme, ≤ 300 graphèmes
- **Toujours en anglais.** Hashtags anglais.
- **Jamais** de prénom, ville, métier, âge exact, nom de famille, ni « my husband ».
  Toujours : *his wife*, *she*, *a married man*, *my submissive*, *owned*.
- **Jamais** de surnom tendre sur son poids.
- Contrôler `len(text) ≤ 300` **graphèmes** (pas octets) AVANT l'upload d'image — sinon on gaspille le blob.
- Toujours un `alt` : « Anonymous flashback: close-up of a nude submissive male body, face not shown. »
- Varier les angles d'un post à l'autre, jamais deux légendes identiques.

## 6. Publication XRPC — les bons NSID
```
POST /xrpc/com.atproto.server.createSession        (JSON, Content-Type: application/json OBLIGATOIRE)
POST /xrpc/com.atproto.repo.uploadBlob             (corps = octets bruts, Content-Type: image/jpeg, Bearer)
POST /xrpc/com.atproto.repo.createRecord           (collection app.bsky.feed.post)
```
- **`com.atproto.uploadBlob` n'existe pas** → 501 MethodNotImplemented. Le bon est `com.atproto.repo.uploadBlob`.
- **`app.bsky.feed.post` renvoie 501** (passerelle dégradée) → toujours `createRecord`.
- Compteurs via `public.api.bsky.app` (`bsky.social` renvoie 401 sur getProfile).
- Mot de passe : `Milouze74??..` — **le M majuscule est obligatoire**, en minuscule → 401 trompeur.

Raccourci : `python3 /home/Furiosha/luna_save/bsky_publish.py --image x.jpg --text "..." --alt "..." --dry-run`

## 7. Journalisation
Après publication : noter URI, URL `bsky.app/profile/…/post/…`, le fichier utilisé et « ne pas reposter ».
Le CID entre alors dans `bsky_cids.txt` et le prochain `--check` le verra.

## 8. Choisir entre flashback et photo fraîche
Sa préférence : **les vieilles photos sortent, les récentes dorment en réserve.**
Les réserves privées ne se vident pas sur demande : ce qui n'a pas vocation à être public ne sort jamais.
