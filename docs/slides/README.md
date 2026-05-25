# Slides soutenance — Recipe Shelter

> Support visuel pour le **oral de soutenance** RNCP d'Arthur.
> Le deck est écrit en **[Marp](https://marp.app/)** — du Markdown pur qui se rend en HTML / PDF / PPTX avec une commande.

---

## Pourquoi Marp (et pas PowerPoint / Google Slides) ?

| Critère | Marp | PowerPoint |
|---|---|---|
| Versionnable (git, diff lisible) | ✅ | ❌ |
| Cohérent avec le reste du projet (Markdown partout) | ✅ | ❌ |
| Schémas Mermaid intégrables directement | ✅ | ❌ |
| Speaker notes en commentaires HTML | ✅ | ✅ |
| Rendu PDF / HTML / PPTX | ✅ | ✅ (PPTX natif) |
| Édition sans logiciel propriétaire | ✅ | ❌ |

> Si Arthur préfère présenter depuis PowerPoint, Marp peut **exporter en .pptx** — il aura juste à retoucher la mise en forme finale.

---

## Comment rendre le deck

### Option 1 — VS Code (recommandée pour itérer)

1. Installer l'extension **« Marp for VS Code »** (`marp-team.marp-vscode`).
2. Ouvrir [`soutenance-slides.md`](soutenance-slides.md).
3. Bouton **« Open Marp Preview to the side »** en haut à droite.
4. Pour exporter : palette de commandes (`Ctrl+Shift+P`) → `Marp: Export slide deck...` → choisir PDF ou PPTX.

### Option 2 — CLI (pour le rendu final)

```powershell
# Installer une fois
npm install -g @marp-team/marp-cli

# Rendre en PDF (ce qu'on présente)
marp soutenance-slides.md --pdf --allow-local-files -o soutenance.pdf

# Ou en PPTX si Arthur veut retoucher dans PowerPoint
marp soutenance-slides.md --pptx --allow-local-files -o soutenance.pptx

# Ou en HTML standalone (présentation depuis un navigateur)
marp soutenance-slides.md --html --allow-local-files -o soutenance.html
```

### Mermaid

Les diagrammes Mermaid intégrés (3 dans ce deck) sont rendus à condition d'utiliser :
- l'extension VS Code Marp **+** l'extension Mermaid, OU
- la CLI Marp avec `--theme-set` pointant sur un thème supportant Mermaid (le thème par défaut de ce deck l'inclut via `<script type="module">`).

**Alternative simple** : exporter les Mermaid en PNG depuis [mermaid.live](https://mermaid.live), et remplacer les blocs ` ```mermaid ` par `![](chemin/vers/image.png)`. Plus de souci de rendu.

---

## Structure du deck

| Section | Slides | Durée cible |
|---|---|---|
| 1. Intro + contexte | 1-3 | 1 min |
| 2. Cahier des charges + démarche | 4-5 | 2 min |
| 3. Bloc 1 — Frontend statique | 6-8 | 2 min |
| 4. Bloc 2 — Backend from scratch | 9-18 | **8 min** (cœur de la défense) |
| 5. Bloc 3 — Angular | 19-22 | 3 min |
| 6. Démo en ligne | 23-24 | 2 min (transition vers démo live) |
| 7. Améliorations + perspectives | 25-26 | 1 min |
| 8. Conclusion + Q&A | 27-28 | 1 min |
| **Total** | **28 slides** | **~20 min** |

Le créneau total de soutenance est typiquement 30-45 min : ~20 min de présentation + démo, puis ~15 min de questions du jury + adaptation de code en direct.

---

## Philosophie du deck

### 1. **Le deck n'est pas le script**

Les slides sont des **points d'ancrage visuels** — diagrammes, captures, tableaux récap. Arthur parle ; le slide rassure le jury et soutient ce qu'il dit. **Aucun slide ne doit être lu à voix haute mot pour mot.**

Les speaker notes (commentaires `<!-- _footer: -->` ou texte entre `<!-- -->`) contiennent la trame parlée, **pas le verbatim**.

### 2. **Densité = ennemi**

Règle de fer : **maximum 6 lignes par slide**. Si Arthur a envie d'en mettre plus, c'est qu'il a peur d'oublier — la solution est de **réviser** plus, pas de bourrer le slide. Un slide chargé = jury qui lit au lieu d'écouter.

### 3. **Tout slide doit être justifiable**

Pour chaque slide, Arthur doit pouvoir répondre à la question : « **pourquoi ce slide ?** ». S'il n'a pas de réponse en 5 secondes, le slide saute.

### 4. **Les ADR et UML sont des renforts, pas des armes principales**

Les diagrammes UML et les ADRs livrés dans `_draft_uml/` et `_draft_adr/` sont là **au cas où** un membre du jury demande un détail. Arthur ne les ouvre pas systématiquement — il les **mentionne** (« j'ai formalisé ce choix dans une ADR, je peux l'ouvrir si vous voulez »), et le jury décide.

---

## Walkthrough — comment t'approprier ce deck (Arthur)

### Étape 1 — Lecture critique (30 min)

Ouvre [`soutenance-slides.md`](soutenance-slides.md). Pour **chaque** slide :

1. Lis le titre + le contenu.
2. Pose-toi la question : « **est-ce que je peux expliquer ce slide pendant 30-45 secondes sans regarder les speaker notes ?** »
3. Si **non** : marque le slide d'un commentaire `<!-- TODO: réviser X -->` et passe à la suite.
4. Si **oui mais avec des nuances** : édite-le. C'est ton deck, pas le mien.

À la fin de cette passe : tu sais combien de chantiers de révision tu as.

### Étape 2 — Personnalisation (1 h)

Les zones à **obligatoirement** retoucher (marquées `[À COMPLÉTER]` dans le deck) :
- Slide 1 : date de soutenance, nom des jurés (si tu les connais à l'avance)
- Slide 7 : score Lighthouse réel (à mesurer)
- Slide 18 : capture ou lien du dashboard de modération (si tu as)
- Slide 22 : capture Angular DevTools ou Lighthouse côté Angular
- Slide 23-24 : scénario de démo précis (s'aligner avec `documentation/soutenance/demo/scenario-demo.md` côté backend repo)
- Slide 28 : tes coordonnées (LinkedIn, GitHub) si tu veux les afficher

### Étape 3 — Drill (≥ 3 passes complètes avant la soutenance)

1. **Passe 1 — chronométrée** : présente le deck **à toi-même à voix haute**, debout, comme si le jury était là. Chronomètre slide par slide. Cible : **20 min total**. Si tu dépasses de plus de 3 min : il faut couper.
2. **Passe 2 — interruptions** : Quentin ou un pote dev t'interrompt **3-5 fois pendant la présentation** avec une question imprévue. Tu réponds **sans perdre ton fil** (« bonne question, je reviens dessus après le slide suivant, sinon je peux ouvrir le fichier maintenant — vous préférez ? »). C'est le réflexe à entraîner.
3. **Passe 3 — sans deck** : tu présentes les **10 messages-clés** du deck **sans ouvrir le deck**, juste avec un papier blanc. Si tu y arrives, le deck est devenu un soutien et plus une béquille.

### Étape 4 — Préparer la projection (J-1)

- Tester le rendu **sur un autre écran que ton portable** (différence de couleurs / contraste).
- Tester en **mode présentateur** : speaker notes lisibles sur ton écran, slide propre sur le projecteur.
- Avoir le **PDF en backup local** (ne pas dépendre du wifi de la salle).
- Avoir le **deck ouvert dans un onglet, le repo code ouvert dans un autre, la démo en ligne dans un troisième**.

---

## Lien avec les autres artefacts

| Artefact | Lien | Quand le mentionner pendant la soutenance |
|---|---|---|
| Diagrammes UML | [Diagrammes UML](../soutenance/backend/uml/README.md) | Slides 13, 18 — référencer si jury demande la modélisation |
| ADRs | [ADRs](../soutenance/backend/adr/README.md) | Slides 10-12, 15, 17 — chaque décision techn. = une ADR à mentionner |
| Narration from-scratch | [Narration from-scratch](../soutenance/backend/adr/00-narration-from-scratch.md) | Slide 9 — la pièce maîtresse pour défendre le Bloc 2 |
| Banque de questions jury | [Banque de questions jury](../jury/README.md) | Préparation **hors soutenance** — drill avant le jour J |
| Cahier des charges | `CAHIER_DES_CHARGES.md` (hors site, racine du workspace) | Slide 4 — citer textuellement la contrainte « from scratch » |

---

## Ne pas copier ce dossier dans le repo `documentation`

Comme les autres `_draft_*`, ce dossier reste au workspace, **hors des 3 repos git**.

> Si Arthur souhaite livrer le PDF final du deck dans son repo `documentation` (recommandé), il peut :
> ```powershell
> # Rendre le PDF
> marp _draft_slides/soutenance-slides.md --pdf --allow-local-files -o documentation/soutenance/soutenance.pdf
> # Puis commit côté Arthur (pas Quentin) avec son identité
> ```
> Le `.md` source du deck n'a **pas besoin** d'être committé — c'est un brouillon de travail. Seul le PDF final intéresse le jury.
