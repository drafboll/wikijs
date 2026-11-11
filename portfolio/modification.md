---
title: Modifier le contenu du site
description: 
published: 1
date: 2026-11-11T11:08:49.566Z
tags: 
editor: markdown
dateCreated: 2026-11-11T11:06:05.415Z
---

# MODIFIER LE CONTENU DU SITE ALEXANDRE-FAYE.FR

Le guide pas à pas pour changer ce qui s'affiche sur le site : ajouter une nouvelle sortie dans
une page de sport, des photos, une vidéo, un nouveau sport, ou changer un texte.

Tout se fait **sur le PC**, dans `/home/alexandre/alexandre-faye-3d/`. Le serveur n'est
touché qu'à la toute fin, pour mettre en ligne (voir la page *Mettre à jour le site en ligne*).

> **La règle d'or** : on ne modifie **jamais** le dossier `site/`. Il est effacé et recréé à
> chaque mise à jour. On modifie les **sources** (les `.json` et le dossier `template/`), puis on
> régénère.
{.is-warning}

---

## 1. Le principe en trois gestes

Quelle que soit la modification, c'est toujours la même chose :

1. **Modifier** un fichier source (un texte, une photo ajoutée…) ;
2. **Double-cliquer** sur `mettre-a-jour.bat` : il fabrique le site et l'ouvre dans le navigateur
   sur http://127.0.0.1:5180 ;
3. **Regarder** le résultat. Si c'est bien, on met en ligne. Sinon, on corrige et on recommence
   au geste 2.

Tant qu'on n'a pas mis en ligne, **rien ne change pour les visiteurs** : on peut essayer,
casser, recommencer sans risque.

> Ne jamais ouvrir une page en double-cliquant sur un fichier `.html` : la 3D ne démarre pas et
> les titres se superposent. Toujours passer par `mettre-a-jour.bat`.
{.is-info}

---

## 2. Les outils à avoir

| Outil | Pourquoi | Où le trouver |
|---|---|---|
| **Visual Studio Code** (recommandé) ou Notepad++ | Pour modifier les fichiers. Il colore le code et **souligne en rouge** les erreurs dans les `.json` (virgule oubliée…) | code.visualstudio.com (gratuit) |
| **Paint** | Pour réduire la taille des photos et faire les miniatures | Déjà dans Windows |
| **HandBrake** | Pour convertir les vidéos d'iPhone en MP4 | handbrake.fr (gratuit) |

Le Bloc-notes de Windows marche aussi, mais il ne signale aucune erreur.

**Ouvrir le projet dans VS Code** : clic droit sur le dossier `alexandre-faye-3d` → *Ouvrir avec
Code*. La liste des fichiers apparaît à gauche : on clique sur un fichier pour l'ouvrir, on
modifie, et **Ctrl+S** pour enregistrer.

---

## 3. Où se trouve quoi

```
alexandre-faye-3d\
├── site.json               ← textes de la page d'accueil (partie animée)
├── competences.json        ← textes de la page Compétences
├── sport.json              ← page Sport + la liste des sports (cartes, titres, photos de couverture)
├── site-en.json            ← accueil en anglais
├── competences-en.json     ← compétences en anglais
├── mettre-a-jour.bat       ← fabrique le site et l'ouvre (geste 2)
├── preparer-mise-en-ligne.bat ← prépare l'archive pour le serveur
└── template\
    ├── media\              ← TES PHOTOS ET VIDÉOS
    │   ├── tennis\
    │   ├── course\
    │   ├── velo\
    │   ├── natation\
    │   └── randonnee\
    └── sections\
        ├── parcours.html       ← expérience et formation (accueil)
        ├── competences.html    ← le détail des compétences
        └── sport\
            ├── tennis.html     ← les récits de la page Tennis
            ├── course.html     ← les récits de la page Course
            ├── velo.html
            ├── natation.html
            └── randonnee.html
```

**Pour les sports, retiens ceci :**

- le **contenu** d'une page de sport (récits, photos, vidéos) est dans
  `template\sections\sport\<sport>.html` ;
- les **photos et vidéos** elles-mêmes sont dans `template\media\<sport>\` ;
- la **carte** du sport sur la page Sport (titre, petit texte, photo de couverture, « 3 récits ·
  18 photos ») est dans `sport.json`.

---

## 4. Ajouter une nouvelle expérience dans une page de sport

C'est le cas le plus courant. Exemple : tu as fait une **randonnée à Fontainebleau** le
samedi 6 décembre 2026, 12 km, et tu as 4 photos.

### Étape 1 : préparer les photos

**a. Les mettre au bon format**

- Format **JPG** (ou PNG). Si ton téléphone donne des `.HEIC` (iPhone) : ouvre la photo dans
  l'application **Photos** de Windows → `…` → *Enregistrer sous* → choisis **JPG**.
  (Sur l'iPhone, tu peux aussi régler *Réglages → Appareil photo → Formats → Le plus compatible*.)

**b. Leur donner un nom simple**

Le nom de fichier doit être **sans espace, sans accent, sans majuscule**, avec des tirets.
Le plus simple : le lieu, puis un numéro.

| ❌ À éviter | ✅ Bien |
|---|---|
| `IMG_4521.HEIC` | `fontainebleau-01.jpg` |
| `Forêt de Fontainebleau (2).jpg` | `fontainebleau-02.jpg` |
| `Photo été.JPG` | `fontainebleau-03.jpg` |

> Sur le serveur, `Fontainebleau-01.jpg` et `fontainebleau-01.jpg` sont **deux fichiers
> différents**. Une seule majuscule de travers et la photo ne s'affiche pas en ligne, alors
> qu'elle s'affiche sur le PC. D'où la règle : tout en minuscules.
{.is-warning}

**c. Réduire la grande photo (si elle vient d'un téléphone)**

Une photo de téléphone fait souvent 4000 pixels et 4 à 8 Mo : beaucoup trop lourd pour un site.
On la réduit à **1600 pixels** de large :

1. Clic droit sur la photo → *Ouvrir avec* → **Paint** ;
2. **Ctrl+W** (ou bouton *Redimensionner*) ;
3. Coche **Pixels**, laisse **Conserver les proportions** coché, mets **1600** dans *Horizontal* ;
4. **OK**, puis **Ctrl+S** pour enregistrer.

La photo doit maintenant peser moins de 1 Mo environ.

**d. Faire la miniature**

Chaque photo a une **petite version** (300 pixels) qui s'affiche dans la galerie. La grande ne se
charge que quand le visiteur clique. C'est ce qui rend les pages rapides.

1. Ouvre la grande photo dans **Paint** ;
2. **Ctrl+W** → **Pixels** → **300** dans *Horizontal* → **OK** ;
3. **Fichier → Enregistrer sous → Image JPEG**, et nomme-la avec **`-mini`** à la fin :
   `fontainebleau-01-mini.jpg` ;
4. Ferme Paint **sans** réenregistrer la grande.

Recommence pour chaque photo. Tu dois obtenir des paires :

```
fontainebleau-01.jpg        fontainebleau-01-mini.jpg
fontainebleau-02.jpg        fontainebleau-02-mini.jpg
fontainebleau-03.jpg        fontainebleau-03-mini.jpg
fontainebleau-04.jpg        fontainebleau-04-mini.jpg
```

> Pressé ? Tu peux sauter la miniature et utiliser la grande photo aux deux endroits du code
> (étape 3). Ça marche, la page est juste plus lente à charger.
{.is-info}

### Étape 2 : déposer les photos

Copie les 8 fichiers dans le dossier du sport :

```
template\media\randonnee\
```

(Pour la course : `template\media\course\`, pour le vélo : `template\media\velo\`, etc.)

### Étape 3 : écrire le récit

Ouvre `template\sections\sport\randonnee.html` dans VS Code. Le fichier ressemble à ceci :

```html
  <div class="cv" id="randonnee">
    <div class="cv-block">
      <div class="stories">
        <article class="tl-card story reveal">
          … le récit des Vaux de Cernay …
        </article>
        <article class="tl-card story reveal">
          … le récit de Thiéfosse …
        </article>
        …
      </div>
    </div>
  </div>
```

Chaque `<article …> … </article>` est **une expérience** (une carte sur la page). Pour en ajouter
une, on colle un nouveau bloc **juste après la ligne `<div class="stories">`**, pour qu'il
apparaisse **en premier** (le plus récent en haut).

Voici le bloc à copier-coller, puis à modifier :

```html
        <article class="tl-card story reveal">
          <p class="story-meta">Samedi 6 décembre 2026 · Forêt de Fontainebleau · 12 km</p>
          <h4>Les rochers de Fontainebleau</h4>
          <p>Premier paragraphe : où, avec qui, comment ça s'est passé.</p>
          <p>Deuxième paragraphe : le moment fort, la difficulté, le ressenti.</p>
          <div class="gallery">
            <a href="./media/randonnee/fontainebleau-01.jpg"><img loading="lazy" src="./media/randonnee/fontainebleau-01-mini.jpg" alt="Fontainebleau"></a>
            <a href="./media/randonnee/fontainebleau-02.jpg"><img loading="lazy" src="./media/randonnee/fontainebleau-02-mini.jpg" alt="Fontainebleau"></a>
            <a href="./media/randonnee/fontainebleau-03.jpg"><img loading="lazy" src="./media/randonnee/fontainebleau-03-mini.jpg" alt="Fontainebleau"></a>
            <a href="./media/randonnee/fontainebleau-04.jpg"><img loading="lazy" src="./media/randonnee/fontainebleau-04-mini.jpg" alt="Fontainebleau"></a>
          </div>
        </article>
```

**Ce que veut dire chaque ligne :**

| Ligne | Ce qu'elle affiche | Ce que tu changes |
|---|---|---|
| `<article class="tl-card story reveal">` | le début de la carte | rien |
| `<p class="story-meta">…</p>` | la petite ligne bleue au-dessus du titre | la date, le lieu, la distance, séparés par ` · ` |
| `<h4>…</h4>` | le titre du récit | ton titre |
| `<p>…</p>` | un paragraphe de texte | ton texte ; un `<p>…</p>` par paragraphe, autant que tu veux |
| `<div class="gallery">` | le début de la galerie de photos | rien |
| `<a href="…grande…"><img … src="…mini…" alt="…"></a>` | **une photo** | le nom de la grande (après `href=`), celui de la miniature (après `src=`), et une description (après `alt=`) |
| `</div>` | la fin de la galerie | rien |
| `</article>` | la fin de la carte | rien |

**Pour une photo, le schéma est toujours le même** :

```html
<a href="./media/SPORT/GRANDE.jpg"><img loading="lazy" src="./media/SPORT/MINIATURE.jpg" alt="DESCRIPTION"></a>
```

- `href` = la photo qui s'ouvre en grand au clic ;
- `src` = la miniature affichée dans la galerie ;
- `alt` = une courte description (lue par les lecteurs d'écran et par Google).

**Variantes :**

- **Une légende visible** sous chaque photo : remplace `<div class="gallery">` par
  `<div class="gallery gallery-captions">` et ajoute `<span>Ta légende</span>` juste avant `</a>` :

```html
<a href="./media/velo/bordeaux.jpg"><img loading="lazy" src="./media/velo/bordeaux-mini.jpg" alt="Bordeaux"><span>Bordeaux</span></a>
```

- **Pas de photo** : supprime tout le bloc, de `<div class="gallery">` jusqu'à son `</div>`.
- **Pas de date** : supprime la ligne `<p class="story-meta">…</p>`.

**Enregistre** (Ctrl+S).

> **Trois pièges dans les fichiers `.html`** :
> - le caractère **`&`** doit s'écrire `&amp;` (« Tom &amp; Jerry ») ;
> - les signes **`<`** et **`>`** dans un texte s'écrivent `&lt;` et `&gt;` ;
> - chaque balise ouverte doit être fermée : un `<p>` → un `</p>`, un `<article …>` → un
>   `</article>`. VS Code surligne la balise correspondante quand tu cliques dessus.
{.is-warning}

### Étape 4 : mettre à jour la carte du sport

Sur la page **Sport**, la carte Randonnée affiche « 3 récits · 18 photos ». Avec ton nouveau
récit, ça devient « 4 récits · 22 photos ».

Ouvre `sport.json`, cherche le bloc de la randonnée (Ctrl+F → `randonnee`) :

```json
    {
      "file": "randonnee.html", "label": "Randonnée", "icon": "mountain",
      "title": ["Randonnée", "grand air"],
      "lead": "Forêts, montagnes et falaises : des sorties en famille et entre amis, du plat des Yvelines aux Gorges du Verdon.",
      "teaser": "Vaux de Cernay, Thiéfosse et les Gorges du Verdon en famille.",
      "meta": "3 récits · 18 photos", "content": "sections/sport/randonnee.html",
      "cover": "./media/randonnee/verdon-01.jpg"
    }
```

| Champ | Où ça s'affiche | Exemple de modification |
|---|---|---|
| `"meta"` | la petite ligne en bas de la carte | `"4 récits · 22 photos"` |
| `"teaser"` | le texte court de la carte | `"Fontainebleau, Vaux de Cernay, Thiéfosse et le Verdon."` |
| `"lead"` | le texte d'introduction en haut de la page Randonnée | à ton goût |
| `"cover"` | la photo de fond de la carte **et** du haut de la page | `"./media/randonnee/fontainebleau-01.jpg"` |
| `"title"` | le grand titre de la page, sur deux lignes | `["Randonnée", "grand air"]` |

**Les règles du `.json`** (c'est là que se font 90 % des erreurs) :

- un texte est **toujours entre guillemets droits** `"…"` ;
- entre deux champs, **une virgule** ; après le **dernier** champ d'un bloc, **pas de virgule** ;
- un guillemet **à l'intérieur** d'un texte s'écrit `\"` : `"Le \"petit\" Jon"` ;
- l'apostrophe `'` ne pose aucun problème.

Enregistre (Ctrl+S). Si VS Code souligne quelque chose en rouge, c'est qu'il y a une erreur à
cet endroit.

### Étape 5 : vérifier

1. Double-clic sur **`mettre-a-jour.bat`** ;
2. le navigateur s'ouvre : va sur **Sport** → carte **Randonnée** ;
3. vérifie : le nouveau récit est en haut, les photos s'affichent, un clic ouvre la grande, les
   flèches passent d'une photo à l'autre ;
4. si tu ne vois pas ta modification : **Ctrl+F5** (force le navigateur à recharger).

Tout est bon ? Il ne reste qu'à mettre en ligne : voir la page *Mettre à jour le site en ligne*.

---

## 5. Remplir la page Course (le premier récit)

La page Course affiche pour l'instant une carte « Les premières foulées arrivent ». Pour la
remplacer par ton premier récit :

1. prépare et dépose tes photos dans `template\media\course\` (section 4, étapes 1 et 2) ;
2. ouvre `template\sections\sport\course.html` ;
3. **supprime** tout le bloc de la carte d'attente : de `<article class="tl-card sub-empty reveal">`
   jusqu'à son `</article>` (y compris le dessin `<span class="sport-icon">…</span>` à l'intérieur) ;
4. à sa place, colle ceci (et remplis-le comme à la section 4, étape 3) :

```html
      <div class="stories">
        <article class="tl-card story reveal">
          <p class="story-meta">Dimanche 14 décembre 2026 · Parc de … · 5 km · 28 min</p>
          <h4>Ma première sortie</h4>
          <p>Le récit…</p>
          <div class="gallery">
            <a href="./media/course/premiere-sortie-01.jpg"><img loading="lazy" src="./media/course/premiere-sortie-01-mini.jpg" alt="Première sortie"></a>
          </div>
        </article>
      </div>
```

Le `<div class="stories">` est nécessaire ici, car la page Course n'en avait pas encore. Les
récits suivants se colleront juste après cette ligne, comme pour les autres sports.

5. dans `sport.json`, bloc `course.html`, mets à jour :
   - `"meta": "Bientôt"` → `"meta": "1 récit · 1 photo"` ;
   - `"teaser"` et `"lead"` : un vrai texte ;
   - ajoute une photo de couverture : `"cover": "./media/course/premiere-sortie-01.jpg"`
     (attention à la virgule sur la ligne d'avant) ;
6. dans `sport.json`, chapitre **« Mes disciplines »** (`"benefits"`), la ligne de la course dit
   « La nouvelle venue : premières foulées en cours. » : tu peux la changer aussi.

---

## 6. Ajouter seulement des photos à un récit existant

1. Prépare et dépose les photos (section 4, étapes 1 et 2).
2. Ouvre le fichier du sport, trouve le récit (Ctrl+F avec un mot du titre).
3. Dans son `<div class="gallery"> … </div>`, ajoute une ligne par photo, **avant** le `</div>` :

```html
            <a href="./media/randonnee/verdon-19.jpg"><img loading="lazy" src="./media/randonnee/verdon-19-mini.jpg" alt="Gorges du Verdon"></a>
```

4. Mets à jour le nombre de photos dans `"meta"` de `sport.json`.

---

## 7. Ajouter une vidéo

### a. Préparer la vidéo

- Format **MP4 (H.264)**. Les vidéos d'iPhone (`.mov`, souvent en HEVC) ne se lisent pas partout :
  ouvre-les dans **HandBrake**, préréglage **« Fast 1080p30 »**, format **MP4**, *Start Encode*.
- Poids : **moins de 50 Mo** (1 à 2 minutes). Au-delà, mets-la plutôt sur YouTube (voir c).
- Nom : sans espace ni accent, ex. `match-moustafa.mp4`.
- Dépose-la dans `template\media\<sport>\`.

### b. L'afficher

Dans le fichier du sport, une galerie de vidéos s'écrit `<div class="gallery gallery-videos">`
(vignettes larges), et chaque vidéo :

```html
<a href="./media/tennis/match-moustafa.mp4">
  <video src="./media/tennis/match-moustafa.mp4#t=0.5" muted loop playsinline preload="metadata"></video>
  <span>Échange avec Moustafa</span>
</a>
```

- Le chemin apparaît **deux fois** : change les deux.
- `#t=0.5` = l'image utilisée comme vignette (ici à 0,5 seconde). `#t=3` prend l'image à 3 secondes.
- Au survol, la vignette joue la vidéo sans le son ; au clic, elle s'ouvre en grand avec le son.

**Sur la page Tennis**, la galerie de vidéos existe déjà : c'est la carte « Sur le court », en bas
de `template\sections\sport\tennis.html`. Ajoute simplement ton bloc `<a> … </a>` après la dernière
vidéo (« Au coucher du soleil »), avant le `</div>` qui ferme la galerie.

> La page Tennis est un peu différente des autres : ses récits sont des **cartes courtes** côte à
> côte (`<article class="tl-card skill-card reveal"><h4>Titre</h4><p>Texte</p></article>`, en
> haut du fichier). Pour une nouvelle carte courte, copie une de ces lignes. Pour un vrai récit
> avec photos, colle le bloc de la section 4 juste au-dessus de la ligne `<!-- VIDÉOS`, entouré
> de `<div class="cv-block">` et `</div>`.
{.is-info}

### c. Une vidéo YouTube

Prends l'identifiant dans l'adresse : `https://youtu.be/dQw4w9WgXcQ` → `dQw4w9WgXcQ`.

```html
<a href="https://youtu.be/dQw4w9WgXcQ" data-youtube="dQw4w9WgXcQ">
  <img loading="lazy" src="https://img.youtube.com/vi/dQw4w9WgXcQ/hqdefault.jpg" alt="">
  <span>Ma vidéo YouTube</span>
</a>
```

L'identifiant apparaît **trois fois**. Pratique pour les vidéos longues : elles ne pèsent rien sur
le serveur. Mets-la en « non répertoriée » sur YouTube si elle ne doit pas être publique ailleurs.

---

## 8. Ajouter un nouveau sport (exemple : escalade)

1. **La page** : copie `template\sections\sport\course.html` et renomme la copie `escalade.html`.
   Ouvre-la, remplace `id="course"` par `id="escalade"` sur la première ligne, puis mets tes récits
   (section 5).
2. **Les photos** : crée le dossier `template\media\escalade\` et dépose-les.
3. **La carte** : dans `sport.json`, dans la liste `"subpages"`, ajoute un bloc. Le plus simple :
   copie le bloc d'un autre sport, colle-le, et modifie :

```json
    {
      "file": "escalade.html", "label": "Escalade", "icon": "mountain",
      "title": ["Escalade", "prise par prise"],
      "lead": "Le texte d'introduction en haut de la page.",
      "teaser": "Le texte court de la carte.",
      "meta": "1 récit · 5 photos", "content": "sections/sport/escalade.html",
      "cover": "./media/escalade/bloc-01.jpg"
    }
```

   - **L'ordre des blocs = l'ordre des cartes** et des boutons « Précédent / Suivant ».
   - **Virgule** entre deux blocs `{ … }`, **aucune** après le dernier.
   - `"icon"` au choix : `tennis run bike swim mountain spark shield clock chart check people
     target leaf wrench lock globe code phone star truck euro server terminal`.
4. **Optionnel** : dans `sport.json`, ajoute-le au chapitre « Mes disciplines » (`"benefits"` →
   `"items"`), au texte du haut (`"hero"` → `"lead"`), au pied de page (`"footer"` → `"spec"`), et
   change le chiffre « 5 disciplines pratiquées » dans `"figures"` (`"value": 5` → `6`).

---

## 9. Changer d'autres textes

| Je veux changer… | Fichier | Où dans le fichier |
|---|---|---|
| Le titre et le texte d'un chapitre animé de l'accueil | `site.json` | `"hero"`, `"about"`, `"benefits"`, `"results"`, `"contact"` |
| Les chiffres animés (« 15,59 km »…) | le `.json` de la page | `"figures"` : `"value"`, `"suffix"`, `"label"` |
| Mon expérience, ma formation | `template\sections\parcours.html` | les blocs `<article>` |
| Mes compétences | `template\sections\competences.html` | les listes de chaque catégorie |
| La version anglaise | `site-en.json`, `competences-en.json`, `template\sections\en\` | mêmes endroits que le français |
| Les textes du formulaire de contact | le `.json` de la page | `"form"` |

> Un titre de chapitre animé tient sur **deux lignes de 18 caractères maximum** :
> `"title": ["Du STAPS", "à l'informatique"]`. Au-delà, il déborde.
{.is-info}

> Les pages de sport n'existent **qu'en français**. Si tu changes l'accueil ou les compétences,
> pense à faire la même modification dans la version anglaise.
{.is-info}

---

## 10. Demander à Claude

Pour une modification plus compliquée (nouvelle page, nouvelle mise en forme), le plus simple est
de demander à Claude :

1. ouvre l'application Claude, onglet **Code**, sur le dossier `D:\Documents\Claude` ;
2. donne-lui le fichier `RECAP.md` du projet, puis décris ce que tu veux, par exemple :
   *« Ajoute dans la page Randonnée une sortie à Fontainebleau du 6 décembre, 12 km, avec les
   photos fontainebleau-01 à 04 que j'ai mises dans template/media/randonnee, et fais les
   miniatures. »* ;
3. il modifie les fichiers et régénère le site ; tu vérifies sur http://127.0.0.1:5180.

---

## 11. En cas de souci

| Symptôme | Cause et solution |
|---|---|
| `mettre-a-jour.bat` affiche **ERREUR** | Lis les dernières lignes : elles donnent le fichier et souvent la ligne. Le plus fréquent : une virgule en trop ou manquante dans un `.json`, ou un `"` non échappé (écris `\"`) |
| La fenêtre de `mettre-a-jour.bat` se ferme toute seule sans rien ouvrir | Ouvre une invite de commandes dans le dossier et tape `mettre-a-jour.bat` pour lire le message |
| Titres superposés, pas de 3D | Page ouverte par double-clic sur un `.html` : passe par `mettre-a-jour.bat` |
| Une photo n'apparaît pas | Vérifie le nom **exact** (minuscules, extension `.jpg` ou `.jpeg`, `-mini`) et le dossier |
| La photo s'affiche sur le PC mais pas en ligne | Différence de majuscules dans le nom (Windows ne les distingue pas, le serveur si) |
| Une vidéo ne se lit pas | Vidéo HEVC / `.mov` : convertis-la en MP4 H.264 avec HandBrake |
| Le récit apparaît, mais la mise en page est cassée | Une balise mal fermée : vérifie les `</p>`, `</div>`, `</article>` |
| Un `&` casse l'affichage | Écris `&amp;` |
| La modification n'apparaît pas | **Ctrl+F5** pour forcer le rechargement |
| En ligne, l'ancienne version reste affichée | Purge du cache Cloudflare, puis Ctrl+F5 (voir *Mettre à jour le site en ligne*) |
