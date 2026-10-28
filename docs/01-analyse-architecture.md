# LinguaMind — Étapes 1 et 2 : analyse technique et proposition d'architecture

> Statut : **proposition à valider** avant la création de la structure du projet (étape 4).
> Date de l'analyse : 28 octobre 2026.
>
> Convention : ✅ = vérifié pendant l'analyse (commande exécutée, index consulté) ;
> 🔎 = à vérifier à l'étape 3 (source inaccessible depuis l'environnement d'analyse) ;
> ⚠️ = limite ou risque connu.

---

## 0. Synthèse en une page

| Question | Réponse recommandée |
|---|---|
| Plateforme V1 | **Application desktop Python** (Windows, Linux, macOS) avec interface **PySide6 (Qt 6)** |
| Moteur de traduction | **CTranslate2** exécutant des modèles **Marian/OPUS-MT**, en lisant directement le format de paquets **Argos Translate** (sans dépendre de la bibliothèque `argostranslate` elle-même, voir §3.3) |
| Moteur optionnel | **NLLB-200 distillé 600M** (converti CTranslate2, int8), uniquement en pack optionnel car sa licence interdit l'usage commercial |
| Malagasy | Paquets **OPUS-MT en↔mg** convertis en CTranslate2 (🔎 disponibilité exacte à confirmer), sinon NLLB-200 en usage non commercial |
| Intelligence idiomatique | Base locale **SQLite + FTS5** d'expressions vérifiées, détection des expressions dans la phrase source avant la traduction neuronale |
| Détection de langue | **langid.py** (BSD, 97 langues dont le malagasy ✅) + détection d'alphabet Unicode + seuil de confiance |
| Langues V1 réalistes | ~49 langues avec l'anglais, via les paquets Argos existants ✅ ; malagasy en paquet dédié |
| Taille d'une paire | ≈ 60 à 120 Mo par direction (ordre de grandeur) ; NLLB-600M int8 ≈ 600 à 700 Mo pour toutes ses langues |
| Matériel minimal visé | CPU x86-64 ou ARM64, 4 Go de RAM, 1 Go de disque libre + packs |
| Android | Phase ultérieure en **Kotlin**, en réutilisant le format des packs et la base SQLite |

L'idée directrice : **un cœur de traduction indépendant de l'interface**, piloté par des
données (packs, base d'expressions, manifestes JSON), testable sans interface et portable plus tard
vers Android.

---

## 1. Analyse des besoins

### 1.1 Besoins fonctionnels (résumés)

1. Traduire un texte entre deux langues, sans réseau, après installation des packs.
2. Détecter automatiquement la langue source, permettre le choix manuel et l'inversion.
3. Produire plusieurs sorties : naturelle, littérale, familière, formelle, équivalents idiomatiques,
   explication.
4. Reconnaître expressions idiomatiques, proverbes, argot, registres, et **signaler l'incertitude**
   au lieu d'inventer.
5. Gérer les packs linguistiques : liste, taille, état, import, suppression.
6. Dictionnaire d'expressions consultable (par expression, par sens, par thème, par registre).
7. Historique, favoris, paramètres, thèmes, taille du texte.
8. Fonctions complémentaires modulaires : documents texte, synthèse vocale, reconnaissance vocale,
   OCR.

### 1.2 Besoins non fonctionnels

- **Hors ligne strict** pour la traduction : aucun appel réseau dans le chemin de traduction.
- **Confidentialité** : aucun texte transmis, aucune télémétrie.
- **Légèreté** : chargement des modèles à la demande, un seul modèle actif en mémoire en mode
  économie.
- **Honnêteté** : ne jamais afficher une langue comme disponible si son modèle n'est pas installé ;
  ne jamais présenter une suggestion générée comme une expression attestée.
- **Extensibilité** : ajouter une langue = ajouter un pack (données), pas modifier du code.

### 1.3 Le point le plus difficile : « au-delà de la traduction littérale »

Un modèle de traduction neuronale (Marian, NLLB) produit **une** traduction, généralement correcte
pour les phrases courantes, souvent correcte pour les idiomes très fréquents (car présents dans les
corpus d'entraînement), mais :

- il ne sait **pas** produire sur demande une variante « formelle » ou « familière » ;
- il ne sait **pas** expliquer un idiome ;
- il traduit souvent littéralement les idiomes rares, régionaux, récents (argot des jeunes) ;
- sa qualité chute fortement pour les langues peu dotées (dont le malagasy).

La réponse réaliste est donc **hybride** (détaillée en §5) :

| Sortie demandée | Source réaliste hors ligne | Fiabilité |
|---|---|---|
| Traduction naturelle | Modèle neuronal + substitution des expressions reconnues par leur équivalent vérifié | Bonne pour les langues bien dotées |
| Traduction littérale | Sortie brute du modèle **sans** substitution, et glose mot à mot si un lexique bilingue est installé | Indicative |
| Familière / formelle | (a) entrées de la base d'expressions classées par registre ; (b) règles de registre limitées (ex. tu/vous en français) ; (c) hypothèses alternatives du modèle, affichées comme « autres formulations » et **non** étiquetées par registre | Partielle, signalée |
| Expressions équivalentes | Base d'expressions uniquement | Indiquée entrée par entrée |
| Explication du sens | Base d'expressions uniquement ; sinon « aucune explication disponible hors ligne » | Indiquée entrée par entrée |
| Niveau de confiance | Score du modèle, pivot, détection de langue, couverture du vocabulaire | Indicateur, pas une garantie |

Un **module génératif optionnel** (petit modèle de langage local via llama.cpp) pourrait plus tard
proposer des reformulations de registre et des explications. Il est volontairement **hors V1** : il
pèse plusieurs Go, il peut halluciner, et ses sorties devront être étiquetées « suggestion générée,
non vérifiée ».

---

## 2. Comparaison des plateformes

| Critère | Android natif (Kotlin) | Desktop Python | Flutter multiplateforme |
|---|---|---|---|
| Exécution de CTranslate2 | Pas de paquet officiel Android : compilation C++ avec le NDK + pont JNI à maintenir | Roues `pip` officielles Windows / Linux / macOS (✅ `ctranslate2 4.8.4` installé sous Python 3.13) | Même problème qu'Android, plus un pont FFI Dart ↔ C++ par plateforme |
| Réutilisation des paquets Argos | Possible, mais chargeur à réécrire | Immédiate | Chargeur à réécrire en Dart ou en C++ |
| Alternatives de moteur | Google ML Kit (hors ligne, gratuit, mais **pas de malagasy**, modèles et conditions propriétaires) ; Bergamot (Mozilla) à porter | Toutes les options étudiées | Plugins ML Kit (mêmes limites) |
| Mémoire disponible | Souvent 2 à 4 Go partagés : NLLB difficile | Confortable | Selon la cible |
| Vitesse de développement d'une V1 réellement fonctionnelle | Lente | **Rapide** | Moyenne (UI rapide, moteur lent à intégrer) |
| Tests automatisés du moteur | Instrumentés, plus lourds | `pytest`, simples, exécutables en intégration continue | Mixte |
| Distribution | APK / AAB | PyInstaller (exécutable autonome) | Binaires natifs |

**Conclusion** : la seule voie qui permet de livrer rapidement un moteur hors ligne **réellement
exécutable et testé** est le desktop Python. Android reste un objectif (phase 7), facilité par le
fait que les packs (modèles CTranslate2 + SentencePiece + manifeste JSON) et la base d'expressions
(SQLite) sont des données portables.

### 2.1 Choix de la bibliothèque d'interface desktop

| Option | Avantages | Inconvénients |
|---|---|---|
| **PySide6 (Qt 6)** — recommandé | Widgets natifs, accessibilité (Qt Accessibility, lecteurs d'écran), thèmes clair/sombre, synthèse vocale native via `QtTextToSpeech`, licence LGPL-3.0 ✅ | Taille (~100 à 150 Mo avec `PySide6-Essentials` seul, sans WebEngine) |
| Flet (Flutter piloté par Python) | Visuel moderne | Empaquetage et accessibilité moins maîtrisés, couche supplémentaire |
| Tkinter / CustomTkinter | Très léger | Accessibilité et rendu moderne limités |

On installera **`PySide6-Essentials`** (et non le méta-paquet `PySide6`, qui tire WebEngine et Pdf,
✅ vérifié dans ses dépendances).

---

## 3. Comparaison des moteurs de traduction hors ligne

### 3.1 Tableau comparatif

| Moteur / modèles | Langues | Qualité | Taille | Licence | Plateformes | Verdict |
|---|---|---|---|---|---|---|
| **Argos Translate** (paquets) | 50 codes dans l'index officiel ✅ ; quasiment tout passe par l'anglais (seule paire directe hors anglais : pt↔es ✅) ; **pas de malagasy** ✅ | Correcte à bonne pour les grandes langues ; pivot = erreurs cumulées | ≈ 60–120 Mo par direction | Code MIT ; licence des modèles à lire dans chaque paquet 🔎 | Tout ce qui fait tourner CTranslate2 | **Retenu comme source de paquets** |
| **CTranslate2** (moteur d'inférence) | Dépend du modèle | Identique au modèle d'origine, quantification int8 quasi sans perte | Divise la taille par ~4 en int8 | MIT | Windows, Linux, macOS (x86-64, ARM64) ✅ | **Retenu comme moteur d'exécution** |
| **OPUS-MT / Marian** (Helsinki-NLP) | Plus d'un millier de paires, dont en↔mg attendues 🔎 | Variable selon la paire ; correct pour les langues bien dotées | ≈ 300 Mo PyTorch, ≈ 75–80 Mo en CTranslate2 int8 | CC-BY 4.0 (attribution obligatoire) | Conversion via `ct2-transformers-converter` (outil de préparation, pas à l'exécution) | **Retenu pour le malagasy et les paires manquantes** |
| **NLLB-200** (Meta) | 200 langues dont malagasy (`plt_Latn`), wolof, yoruba, haoussa, zoulou… | Bonne couverture des langues peu dotées, qualité inégale | 600M distillé ≈ 600–700 Mo int8 ; 1,3B ≈ 1,3–1,5 Go int8 | **CC-BY-NC 4.0 : usage non commercial uniquement** | CTranslate2 | **Optionnel**, jamais installé par défaut, avertissement de licence |
| Bergamot / Firefox Translations | ~30 langues européennes | Bonne, très rapide | ≈ 15–40 Mo par direction | MPL-2.0 (code) | Moteur Marian/C++ ou WASM | Écarté en V1 (pas de malagasy, second moteur à intégrer) ; candidat pour Android |
| Google ML Kit | ~59 langues | Moyenne | ≈ 30 Mo par langue | Propriétaire | Android / iOS uniquement | Écarté (pas de malagasy, pas de desktop, conditions Google) |
| Petit LLM local (llama.cpp) | Variable | Fluide mais peut halluciner | 1 à 5 Go | Selon le modèle | Toutes | Hors V1 ; module « suggestions » éventuel |

### 3.2 Recommandation

- **Moteur d'exécution unique : CTranslate2**, qui sait exécuter à la fois les modèles Argos, les
  modèles OPUS-MT convertis et NLLB. Un seul moteur C++ = une seule dépendance lourde à valider.
- **Sources de modèles** : paquets Argos (langues principales), OPUS-MT convertis (malagasy et
  paires manquantes), NLLB en option non commerciale.
- **Pivot par l'anglais** lorsque la paire directe n'existe pas (ex. fr→ja = fr→en→ja), avec une
  pénalité affichée sur le niveau de confiance.

### 3.3 Décision importante : ne pas dépendre de la bibliothèque `argostranslate`

Constat ✅ (métadonnées PyPI, `argostranslate 1.11.0`) : elle dépend de `stanza==1.10.1`, qui dépend
de **PyTorch** (plusieurs centaines de Mo installés), de `spacy`, et de `minisbd`, publié sous
**licence AGPL**.

Or un paquet `.argosmodel` est une archive contenant un modèle CTranslate2, un modèle SentencePiece et
un fichier `metadata.json`. LinguaMind lira donc ces paquets **directement** avec `ctranslate2` et
`sentencepiece`, et fournira son propre découpeur de phrases (règles + liste d'abréviations, gestion
de la ponctuation non latine `。！？`, `؟`, `।`). Bénéfices : installation bien plus légère, pas
d'AGPL, contrôle total sur les accès réseau. Le format exact des paquets (préfixes de langue cible,
variantes de structure entre versions) sera **vérifié sur de vrais paquets** à l'étape 5 🔎.

---

## 4. Architecture technique recommandée

### 4.1 Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────────────┐
│ linguamind.ui  (PySide6)                                             │
│  Accueil · Traduction détaillée · Expressions · Historique ·         │
│  Langues · Paramètres                                                │
└───────────────▲──────────────────────────────────────────────────────┘
                │ appelle uniquement la couche services (pas d'accès direct aux modèles)
┌───────────────┴──────────────────────────────────────────────────────┐
│ linguamind.services  — façade applicative                            │
│  TranslationService · PackService · ExpressionService ·              │
│  HistoryService · SettingsService   (exécution en tâche de fond)     │
└──┬───────────┬──────────────┬──────────────┬──────────────┬──────────┘
   │           │              │              │              │
┌──▼─────┐ ┌───▼────────┐ ┌───▼─────────┐ ┌──▼──────────┐ ┌─▼──────────┐
│ engine │ │ packs      │ │ detection   │ │ expressions │ │ storage    │
│ pipeline│ │ registre,  │ │ langid +    │ │ SQLite FTS5 │ │ historique,│
│ backends│ │ import,    │ │ alphabets + │ │ recherche,  │ │ favoris,   │
│ CT2,   │ │ vérification│ │ seuils      │ │ appariement │ │ paramètres,│
│ pivot, │ │ SHA-256,   │ │             │ │ dans le     │ │ cache      │
│ cache  │ │ suppression │ │             │ │ texte       │ │            │
└────────┘ └────────────┘ └─────────────┘ └─────────────┘ └────────────┘
   │
┌──▼──────────────────────────────┐   ┌──────────────────────────────────┐
│ linguamind.netguard             │   │ linguamind.extras (optionnels)   │
│ mode hors ligne strict : bloque │   │ tts (QtTextToSpeech), documents, │
│ toute ouverture de socket       │   │ ocr (Tesseract), stt (Vosk)      │
└─────────────────────────────────┘   └──────────────────────────────────┘
```

Règles de dépendance : `ui → services → (engine, packs, detection, expressions, storage)`.
Aucun module du cœur n'importe Qt : le cœur est utilisable en ligne de commande et dans les tests.

### 4.2 Modules

| Module | Responsabilité | Interfaces principales |
|---|---|---|
| `engine.backends` | Abstraction d'un moteur : `CTranslate2Backend` (Argos, OPUS-MT, NLLB), `FakeBackend` (tests uniquement, jamais visible dans l'application) | `translate(segments, src, tgt, n_best) -> list[Hypothesis]` |
| `engine.router` | Choisit le chemin : paire directe, pivot via l'anglais, ou erreur « pack manquant » explicite | `plan(src, tgt) -> Route` |
| `engine.pipeline` | Normalisation → segmentation → repérage des expressions → traduction → réinsertion → variantes → confiance | `translate(request) -> TranslationResult` |
| `engine.segmenter` | Découpage en phrases multi-alphabets, textes longs par lots | `split(text, lang)` |
| `engine.confidence` | Score indicatif : log-probabilité du modèle, pivot, confiance de détection, ratio de longueur, jetons inconnus | `score(...) -> Confidence(level, reasons)` |
| `engine.cache` | Cache LRU en mémoire + cache SQLite optionnel (clé = hachage texte + paire + version du pack) | |
| `packs` | Manifestes, registre des packs installés, import d'archive, contrôle d'intégrité, suppression, calcul de taille | `list()`, `install(path)`, `remove(id)` |
| `detection` | `langid.py` restreint aux langues pertinentes, détection d'alphabet, seuil → « incertain » | `detect(text) -> Detection(lang, confidence, alternatives)` |
| `expressions` | Base SQLite + FTS5, appariement dans une phrase (normalisation, accents, variantes à emplacements `{qqn}`), import de jeux de données | `match(text, lang)`, `search(query, ...)` |
| `storage` | Historique, favoris, collections, paramètres (fichiers dans le dossier utilisateur via `platformdirs`) | |
| `netguard` | En mode hors ligne strict, remplace la création de socket par une erreur : garantit et **teste** l'absence d'accès réseau | `enable()`, `disable()` |
| `cli` | `linguamind translate`, `linguamind packs list/install/remove`, `linguamind expressions search` | |

### 4.3 Pipeline d'une traduction

```
texte source
  │ 1. normalisation Unicode (NFC), espaces, guillemets ; conservation du texte original
  │ 2. détection de langue (si « auto ») → si confiance faible : demander confirmation
  │ 3. segmentation en phrases
  │ 4. repérage des expressions connues (base locale) dans chaque phrase
  │ 5. traduction neuronale (directe ou pivot), n hypothèses
  │ 6. construction des variantes :
  │      naturelle  = meilleure hypothèse + équivalents vérifiés des expressions repérées
  │      littérale  = meilleure hypothèse sans substitution (+ glose si lexique installé)
  │      formelle / familière = entrées de registre de la base + règles limitées
  │      autres formulations = hypothèses 2..n (sans étiquette de registre)
  │      explication = signification et note culturelle des expressions repérées
  │ 7. niveau de confiance + avertissements (pivot, langue peu dotée, expression sans
  │    équivalent vérifié, détection incertaine)
  ▼
TranslationResult (sérialisable, enregistré dans l'historique si activé)
```

Exemple attendu avec l'entrée « It's raining cats and dogs. » (en→fr), si l'expression est dans la
base avec un équivalent vérifié :

- naturelle : « Il pleut des cordes. » (substitution depuis la base, source indiquée) ;
- littérale : sortie brute du modèle, par exemple « Il pleut des chats et des chiens. » si le modèle
  traduit mot à mot (le résultat réel dépendra du modèle) ;
- explication : « Signifie qu'il pleut très fort. » ;
- fiabilité de l'équivalence : « vérifiée ».

Si l'expression n'est **pas** dans la base : la sortie du modèle est affichée telle quelle, sans
explication inventée, avec la mention « aucune expression connue détectée ».

### 4.4 Format d'un pack LinguaMind

```
<pack-id>/
  manifest.json      # id, type (translation|expressions|lexicon|detection), langues, direction,
                     # moteur, version, taille, sha256 de chaque fichier, licence, source,
                     # limites connues, qualité indicative
  model/             # modèle CTranslate2 (model.bin, config.json, vocabulaires)
  sentencepiece.model
  LICENSE / ATTRIBUTION.txt
```

Les paquets `.argosmodel` sont importés et convertis en pack LinguaMind (copie + manifeste), sans
exécution de code. Une langue est affichée « disponible hors ligne » **uniquement** si un pack de
traduction valide (intégrité vérifiée, chargement réussi au moins une fois) couvre la paire, ou si une
route pivot complète existe.

### 4.5 Base d'expressions (schéma prévisionnel)

```
expression(id, text, lang, normalized, pattern, meaning, register, theme, example,
           cultural_note, regional_variants, source, license, verified_by, created_at)
equivalence(expression_id, target_expression_id | target_text, target_lang,
            reliability  -- 'verifiee' | 'approchee' | 'litterale_seulement' | 'incertaine',
            note)
similar(expression_id, similar_id)
expression_fts  -- index plein texte FTS5 sur text, meaning, example
```

Registres : familier, neutre, soutenu, vulgaire, professionnel (+ argot, jeunes, technique,
académique comme thèmes).

---

## 5. Langues réellement prises en charge par la solution envisagée

### 5.1 Niveau A — paquets Argos existants (✅ vérifié dans l'index officiel le 28/10/2026)

Directions **anglais ↔ X** disponibles pour : albanais, allemand, arabe, azéri, basque, bengali,
bulgare, catalan, chinois (simplifié `zh` et traditionnel `zt`), coréen, danois, espagnol, espéranto,
estonien, finnois, **français**, galicien, grec, hébreu, hindi, hongrois, indonésien, irlandais,
italien, japonais, kirghiz, letton, lituanien, malais, néerlandais, norvégien bokmål, ourdou, persan,
polonais, portugais (et portugais brésilien `pb`), roumain, russe, slovaque, slovène, suédois,
**swahili**, tagalog, thaï, tchèque, turc, ukrainien, vietnamien.

- Paire directe hors anglais : uniquement portugais ↔ espagnol.
- Toutes les autres combinaisons (ex. français → japonais) passent par l'anglais : qualité réduite,
  temps doublé, signalé à l'utilisateur.
- Toutes les langues prioritaires de la demande sont couvertes **sauf le malagasy**.

### 5.2 Niveau B — malagasy

- **Argos : aucun paquet malagasy** ✅.
- **OPUS-MT** : modèles en↔mg attendus chez Helsinki-NLP 🔎 (l'accès à huggingface.co était bloqué
  depuis l'environnement d'analyse). Une paire directe fr↔mg n'est pas garantie : sinon pivot
  fr→en→mg, avec cumul d'erreurs.
- **NLLB-200** : couvre le malagasy (`plt_Latn`) avec une paire directe fr↔mg, mais licence non
  commerciale.
- Un script de préparation (exécuté une fois, avec Internet, sur la machine du développeur ou de
  l'utilisateur) convertira ces modèles en packs LinguaMind.
- ⚠️ La qualité du malagasy sera **mesurée** sur un jeu de référence (§9) avant d'être annoncée ; elle
  sera affichée comme « langue peu dotée — vérifiez les résultats importants ».

### 5.3 Niveau C — autres langues africaines et rares

Wolof, yoruba, haoussa, igbo, zoulou, amharique, lingala, kinyarwanda… : couverture via NLLB (non
commercial) et, au cas par cas, via OPUS-MT 🔎. Aucune ne sera affichée tant que son pack n'est pas
installé et évalué.

### 5.4 Détection de langue

- `langid.py` : 97 langues, **inclut le malagasy** ✅. Test réel : « Manao ahoana ianao ? Tsara be ny
  andro anio. » → `mg` (1,0) ✅ ; « Il pleut des cordes aujourd'hui » → `fr` ✅.
- ⚠️ Limite constatée ✅ : « salut ça va » → `tr` (0,52). Les textes courts et familiers sont mal
  détectés. Parades : détection d'alphabet (arabe, cyrillique, CJK, hangul, devanagari…), restriction
  aux langues installées, petit lexique de mots fréquents pour fr/mg/en, et seuil sous lequel
  l'application affiche « détection incertaine » avec les alternatives.
- Option ultérieure : fastText `lid.176.ftz` (176 langues, ≈ 1 Mo, licence CC-BY-SA 3.0).

---

## 6. Limites concernant les expressions idiomatiques et les langues rares

1. **Pas de compréhension « humaine »** : l'ironie, le sarcasme, les doubles sens et les jeux de mots
   ne sont pas détectables de façon fiable par ces modèles. L'application pourra signaler une
   expression connue, pas deviner une intention.
2. **La base d'expressions est la seule source d'équivalents vérifiés.** Sa couverture initiale sera
   modeste (quelques centaines d'entrées rédigées et vérifiées pour fr, en, es, mg), extensible par
   import. Ce n'est pas et ne sera pas présenté comme exhaustif.
3. **Malagasy** : les proverbes (*ohabolana*) et les expressions familières demandent une validation
   par des locuteurs natifs. Les entrées non validées porteront la mention « à vérifier par un
   locuteur natif », et les variantes régionales (merina, côtières) seront indiquées lorsqu'elles
   sont connues.
4. **Argot des jeunes et expressions récentes** : évoluent vite, absents des corpus d'entraînement ;
   couverture uniquement via la base, mise à jour par import.
5. **Registres** : seules les variantes issues de la base ou de règles explicites porteront une
   étiquette « formel » ou « familier ». Les hypothèses alternatives du modèle seront présentées comme
   « autres formulations ».
6. **Pivot par l'anglais** : perte de nuances (tu/vous, genre grammatical, politesse japonaise ou
   coréenne). Signalé systématiquement.
7. **Niveau de confiance** : indicateur heuristique, pas une probabilité que la traduction soit
   juste ; un texte fluide peut être faux.

---

## 7. Taille des modèles et besoins matériels (ordres de grandeur)

| Élément | Disque | RAM lors de l'utilisation |
|---|---|---|
| Application sans modèle (Python + PySide6-Essentials + CTranslate2), empaquetée | ≈ 150 à 250 Mo | ≈ 150 à 250 Mo |
| Une direction Argos (ex. en→fr) | ≈ 60 à 120 Mo | ≈ 150 à 300 Mo par modèle chargé |
| Une direction OPUS-MT convertie int8 | ≈ 75 à 80 Mo | ≈ 150 à 300 Mo |
| Paire complète fr↔en + en↔mg (4 directions) | ≈ 300 à 450 Mo | Un ou deux modèles chargés à la fois |
| NLLB-200 distillé 600M int8 | ≈ 600 à 700 Mo | ≈ 1 à 1,5 Go |
| NLLB-200 distillé 1,3B int8 | ≈ 1,3 à 1,5 Go | ≈ 2 à 3 Go |
| Base d'expressions initiale | < 5 Mo | négligeable |

Configuration minimale visée : processeur 64 bits récent (x86-64 ou ARM64), 2 cœurs, **4 Go de RAM**,
1 Go de disque + packs choisis. Recommandée : 4 cœurs, 8 Go de RAM (indispensable pour NLLB 1,3B).
Sur la machine d'analyse (4 cœurs, 15 Go), un modèle Marian int8 devrait traduire une phrase courte en
bien moins d'une seconde ; ce chiffre sera **mesuré** par un banc d'essai et non supposé.

Mode économie de mémoire : un seul modèle chargé à la fois, déchargement après inactivité,
`inter_threads=1`, quantification int8 imposée.

---

## 8. Dépendances et licences prévues (détaillées et validées à l'étape 3)

| Dépendance | Rôle | Licence | Remarque |
|---|---|---|---|
| Python ≥ 3.11 | Langage | PSF | Testé avec 3.13 |
| `ctranslate2` 4.x | Inférence | MIT | Roues officielles ✅ |
| `sentencepiece` | Tokenisation | Apache-2.0 | |
| `langid` 1.1.6 | Détection de langue | BSD | ✅ installé et testé |
| `PySide6-Essentials` 6.x | Interface | LGPL-3.0 | Liaison dynamique ; obligations LGPL documentées pour la distribution |
| `platformdirs` | Dossiers utilisateur | MIT | |
| `pytest`, `sacrebleu` (dév.) | Tests, métrique chrF | MIT, Apache-2.0 | |
| `PyInstaller` (dév.) | Exécutable final | GPL-2.0 avec exception du chargeur | Autorise la distribution de l'exécutable |
| Paquets Argos | Modèles | Selon paquet 🔎 | Licence copiée dans chaque pack |
| OPUS-MT | Modèles | CC-BY 4.0 | Attribution affichée dans l'écran « Langues » |
| NLLB-200 | Modèle optionnel | CC-BY-NC 4.0 | Non commercial, jamais par défaut |
| Wiktionary via Kaikki.org (import optionnel) | Expressions | CC-BY-SA 4.0 | Partage à l'identique : reste dans un pack séparé |
| Tatoeba (import optionnel) | Exemples | CC-BY 2.0 FR | |
| Tesseract, Vosk, Piper (phase ultérieure) | OCR, voix | Apache-2.0, Apache-2.0, MIT (voix : selon modèle) | Non intégrés en V1 |

Exclus volontairement : `argostranslate` (PyTorch + AGPL via ses dépendances), `transformers`/PyTorch
à l'exécution (trop lourds ; utilisés seulement par le script de conversion de modèles).

---

## 9. Sécurité, confidentialité et stratégie de test

### 9.1 Sécurité et confidentialité

- Aucun appel réseau dans le cœur ; le téléchargement de packs est une action explicite, séparée, et
  désactivable par le « mode entièrement hors ligne » (qui active `netguard`).
- Import d'archives : refus des chemins absolus ou `..` (traversée de répertoires), limite de taille et
  de taux de compression (bombes zip), vérification SHA-256 par rapport au manifeste, aucun code
  exécutable dans un pack, aucun chargement `pickle`/`torch.load`.
- Historique local, désactivable, suppression individuelle et globale ; pas de télémétrie.
- Chiffrement de la base locale : non prévu en V1 (dépendance SQLCipher) ; recommandation de
  chiffrement du disque par le système, documentée.

### 9.2 Tests (étape 9, mais conçus dès l'étape 5)

- **Tests unitaires rapides** avec `FakeBackend` déterministe : routage, pivot, packs manquants,
  segmentation, caractères spéciaux et alphabets non latins, appariement des expressions, historique,
  erreurs de chargement (pack corrompu, SHA-256 faux, fichier manquant).
- **Test hors réseau** : `netguard` activé pendant toute la suite ; toute tentative de connexion fait
  échouer le test.
- **Tests d'intégration** avec de vrais packs : exécutés quand les packs sont présents, signalés comme
  « ignorés, pack absent » sinon (jamais comptés comme réussis).
- **Jeu de référence** (JSONL par paire) : catégories simple, idiome, ambiguïté, fautes de frappe,
  familier, texte long, caractères spéciaux ; métriques chrF + vérifications ciblées (« doit contenir
  l'équivalent X », « ne doit pas être la traduction littérale Y »). Les références fr/en seront
  rédigées et vérifiées ; les références malagasy seront marquées « à valider par un locuteur natif ».

---

## 10. Plan de développement progressif

| Jalon | Étapes | Contenu | Critère de sortie |
|---|---|---|---|
| J0 | 1–3 | Ce document, puis liste figée des dépendances et versions, vérification réelle des modèles et licences | Décisions validées |
| J1 | 4–5 | Structure du projet, cœur : packs, chargeur Argos direct, CTranslate2, pivot, détection, segmentation, cache, CLI, `netguard` | `linguamind translate --from en --to fr "…"` fonctionne réseau coupé ; tests verts |
| J2 | 7 (partie 1) | Base d'expressions SQLite/FTS5, appariement, variantes, explication, confiance ; collection initiale fr/en/es/mg | Exemple « raining cats and dogs » correct de bout en bout ; tests idiomes verts |
| J3 | 6 | Interface PySide6 : 6 écrans, thèmes, taille du texte, accessibilité clavier | Parcours complet utilisable sans terminal |
| J4 | 7 (partie 2) | Packs malagasy (script de conversion OPUS-MT, option NLLB), import/suppression dans l'interface, évaluation fr/mg | Score chrF publié, limites affichées |
| J5 | 8 | Textes longs, fichiers `.txt`/`.md`, synthèse vocale via `QtTextToSpeech` (voix du système), copie/partage ; OCR et reconnaissance vocale en modules optionnels seulement s'ils sont intégrés et testés | Chaque fonction listée comme « disponible » a son test |
| J6 | 9–10 | Correction, banc d'essai sur machine modeste, empaquetage PyInstaller, README et guides en français | Exécutable généré et lancé sur au moins une plateforme |
| J7 | (futur) | Application Android Kotlin réutilisant packs et base d'expressions | Étude de faisabilité CTranslate2 NDK vs Bergamot |

---

## 11. Contraintes constatées dans l'environnement de développement actuel

- L'environnement cloud utilisé pour l'analyse **refuse les connexions vers `argos-net.com` et
  `huggingface.co`** (politique réseau). L'index Argos a pu être lu depuis GitHub, mais les modèles
  eux-mêmes ne peuvent pas y être téléchargés.
- Conséquence : tant que ces domaines ne sont pas autorisés, les tests avec de **vrais modèles**
  devront être exécutés sur votre machine ; ceux de l'environnement cloud utiliseront le
  `FakeBackend` et seront présentés comme tels, jamais comme des traductions réelles.

---

## 12. Décisions à valider avant l'étape 3

1. **Plateforme V1** : desktop Python + PySide6 (Windows / Linux / macOS), Android plus tard ?
2. **Usage commercial envisagé ?** Si oui, NLLB-200 est exclu (y compris pour le malagasy) et l'on
   s'appuie uniquement sur OPUS-MT / Argos.
3. **Système d'exploitation principal** à cibler en premier pour l'empaquetage (Windows ?).
4. **Accès réseau de l'environnement** : autoriser `argos-net.com` et `huggingface.co` pour que les
   tests avec de vrais modèles puissent aussi s'exécuter ici.
