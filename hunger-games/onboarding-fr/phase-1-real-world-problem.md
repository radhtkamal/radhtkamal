# Phase 1 — Quel problème concret Hunger Games résout-il ?

> **Open Food Facts Hunger Games — Onboarding contributeur**
>
> Ce document est la Phase 1 d'une série d'onboarding progressive.
> Il s'appuie sur les preuves du dépôt (README, `src/robotoff.ts`, `src/hooks/useQuestions.ts`, `src/const.ts`) et la [Robotoff API Reference](https://openfoodfacts.github.io/robotoff/references/api/) officielle.

---

## Avant d'ouvrir un seul fichier

Imaginez un rayon de supermarché photographié des milliers de fois par des bénévoles du monde entier. Chaque photo peut montrer un logo de marque, un badge Nutri-Score, une liste d'ingrédients, un symbole de recyclage ou un code-barres — mais la photo n'est que des pixels. Transformer ces pixels en faits structurés (« ce produit est bio », « la marque est Danone », « la catégorie est yaourt ») est difficile à grande échelle.

C'est le problème que Hunger Games existe pour aider à résoudre — non pas en faisant l'apprentissage automatique lui-même, mais en rendant la **validation humaine des suppositions de la machine** suffisamment rapide pour que des millions de produits puissent être améliorés.

---

## Le pipeline (modèle vérifié)

Voici l'histoire de bout en bout, avec chaque flèche étiquetée selon ce que nous pouvons **prouver** par rapport à ce que nous **inférons**.

```
Open Food Facts product database
        │
        │  volunteers upload product photos + some manual edits
        ▼
Product images stored on OFF infrastructure
        │
        │  FACT: README states OFF processes pictures with OCR and ML
        ▼
Robotoff generates predictions / insights
        │
        │  FACT: Robotoff API defines "insights" as facts inferred from pictures
        │  FACT: Robotoff exposes /questions/ for items needing human validation
        ▼
Hunger Games fetches a question and shows it as a simple UI
        │
        │  user clicks Yes / No / Skip
        ▼
Hunger Games sends an annotation to Robotoff
        │
        │  FACT: robotoff.annotate(..., update: 1) is called
        ▼
Robotoff records the vote and may update Open Food Facts
        │
        │  FACT (Robotoff API): update=1 sends the update to Open Food Facts
        │  INFERENCE: exact Product Opener field changes depend on insight_type
        ▼
Open Food Facts product record becomes more complete / accurate
```

### Corrections à un modèle mental naïf

| Hypothèse naïve | Réalité vérifiée |
|---|---|
| « Hunger Games exécute les modèles ML » | **FAIT :** Hunger Games est un frontend React. Le ML vit dans Robotoff. |
| « Une question et un insight sont le même objet » | **FAIT :** Une question est une **présentation** d'un insight qui nécessite encore une validation. Les questions référencent `insight_id`. |
| « Cliquer sur Oui écrit immédiatement dans Open Food Facts depuis le navigateur » | **FAIT :** Le navigateur appelle **le endpoint annotate de Robotoff**. Robotoff décide si/quand pousser vers OFF (`update=1`). |
| « Chaque clic modifie instantanément les données de production » | **FAIT (Robotoff API) :** Les utilisateurs anonymes contribuent des **votes** ; plusieurs votes anonymes identiques peuvent être requis. Les votes des utilisateurs OFF enregistrés peuvent être appliqués directement. |
| « Passer signifie que rien ne s'est passé » | **FAIT (Robotoff API) :** `-1` (skip) indique à Robotoff de ne plus afficher cet insight **à cet utilisateur** (authentifié ou non). |

---

## Qu'est-ce qu'Open Food Facts ?

**FAIT (README + contexte écosystème) :** Open Food Facts est une base de données collaborative et ouverte de produits alimentaires (et connexes). N'importe qui peut scanner un code-barres, photographier un produit et contribuer des données structurées — ingrédients, nutrition, labels, emballage, marques, et plus encore.

**INFÉRENCE :** L'échelle du projet (millions de produits, contributeurs mondiaux, langues multiples) signifie que l'édition manuelle de fiches complètes seule ne peut pas suivre le flux de photos et de données partielles.

**Pourquoi les données produit crowdsourcées comptent :** Les étiquettes produit diffèrent selon le pays, la langue et les reformulations. Une base propriétaire centrale ne peut pas couvrir chaque marque et variante locale. Le modèle OFF — données ouvertes, contributions bénévoles, code-barres comme clé universelle — permet d'agrégger une couverture mondiale à partir d'efforts distribués.

**Pourquoi les photos seules ne suffisent pas :** Une photo prouve *qu'une chose était visible sur l'emballage au moment de la capture*, mais pas :

- si l'interprétation ML est correcte
- si la valeur appartient à un champ normalisé par taxonomie (`en:organic`, pas seulement le mot « Bio »)
- si des signaux contradictoires existent entre plusieurs photos du même produit
- si un logo détecté désigne réellement la marque revendiquée ou un label de certification

Les photos sont des preuves. Les champs structurés sont des affirmations. Robotoff propose des affirmations ; les humains les valident.

---

## Que fait Robotoff ?

**FAIT (intro Robotoff API) :** « Robotoff provides a simple API allowing consumers to fetch predictions and annotate them. »

Robotoff se situe entre les ressources brutes OFF (surtout les images) et les faits produit actionnables. Concrètement, il :

1. **Exécute des prédicteurs** — modèles ML, pipelines OCR, détecteurs de logos, etc. (**FAIT :** l'API a un filtre `predictor` ; le README mentionne OCR/ML.)
2. **Matérialise des insights** — **FAIT (Robotoff API) :** « An insight is a fact about a product that has been either extracted or inferred from the product pictures, characteristics,… If the insight is correct, the Openfoodfacts DB can be updated accordingly. »
3. **Expose des questions** — invites conviviales dérivées d'insights qui nécessitent encore une validation.
4. **Accepte des annotations** — Oui / Non / Passer / (parfois) données structurées.
5. **Orchestre les mises à jour OFF** — lorsque les annotations confirment un insight et que la politique le permet.

**INFÉRENCE :** Robotoff est le **système de référence pour les faits candidats dérivés du ML et leur état de validation**. Open Food Facts reste le **système de référence pour les données produit publiées**.

---

## Vocabulaire du domaine (niveau Phase 1)

Ces termes seront développés en Phase 3. Pour l'instant, retenez cette distinction :

| Terme | Signification en une ligne | Créé par | Rôle de Hunger Games |
|---|---|---|---|
| **Prediction** | Sortie ML brute d'un modèle | Prédicteurs Robotoff | Rarement affichée directement ; filtrée en insights |
| **Insight** | Fait candidat sur un produit (`insight_id`) | Robotoff | Annoté via l'API ; rarement rendu en JSON brut |
| **Question** | Invite prête pour l'UI liée à un `insight_id` | Robotoff (`/questions/`) | Récupérée, affichée, répondue |
| **Annotation** | Décision humaine : `1` oui, `0` non, `-1` passer, `2` oui+données | Humain via Hunger Games → Robotoff | Envoyée par `robotoff.annotate()` |
| **Product update** | Modification de la fiche produit OFF | Product Opener (via Robotoff si autorisé) | **Pas** directement depuis Hunger Games sauf via Robotoff |

**Ne confondez pas « prediction », « insight » et « question »** — l'API les traite comme des couches liées mais distinctes.

Exemple de forme de payload question (**FAIT** depuis `src/robotoff.ts`) :

```typescript
interface QuestionInterface {
  barcode: string;
  insight_id: string;
  insight_type: string;   // e.g. "label", "brand", "category"
  question: string;       // e.g. "Does the product have this label?"
  source_image_url?: string;
  ref_image_url?: string; // reference logo/badge when applicable
  type: string;           // e.g. "add-binary"
  value: string;          // human-readable value
  value_tag: string;      // taxonomy tag sent to Product Opener if accepted
}
```

---

## Pourquoi Robotoff a-t-il besoin d'humains ?

La confiance de la machine n'est pas la vérité terrain.

**INFÉRENCE (forte, standard industriel, alignée avec la conception Robotoff) :** Les modèles lisent mal les photos floues, confondent des logos similaires, hallucinent du texte à partir du bruit OCR, et ne peuvent pas connaître le contexte (autocollant promotionnel vs. label permanent).

**FAIT (Robotoff API) :** Le endpoint annotate implémente un **mécanisme de vote**. Les annotations anonymes sont des votes ; les utilisateurs enregistrés peuvent appliquer directement. Cela encode explicitement une méfiance envers les décisions automatisées ou anonymes en un seul coup pour les données de production.

Les humains apportent :

- **Précision** — rejeter les faux positifs avant qu'ils polluent OFF
- **Routage du rappel** — confirmer les vrais positifs pour que Robotoff applique les mises à jour
- **Résolution d'ambiguïté** — Passer retire les mauvaises questions de *votre* file sans rejeter faussement
- **Signal d'entraînement** — les annotations validées améliorent les futurs prédicteurs (**INFÉRENCE**, boucle ML standard ; le pipeline exact de réentraînement est **INCONNU** depuis le seul dépôt Hunger Games)

---

## Pourquoi Robotoff a-t-il besoin de Hunger Games spécifiquement ?

Robotoff expose des API HTTP. Il ne fournit pas d'UX d'annotation grand public à l'échelle dont OFF a besoin.

**FAIT (README) :**

> « Hunger Games is a series of mini-apps that let users contribute data to Open Food Facts, in a rather fun way using React. It relies heavily on Robotoff APIs, as well as Open Food Facts APIs. »

> « We process all pictures using top OCR and Machine earning techniques and get a lot of predictions about the products. We then need to let users leverage those predictions to easily complete products, in a fun way, on their desktop and/or mobile devices. »

Donc :

> **Robotoff a besoin d'humains.**
> **Les humains ont besoin d'une interface optimisée pour la vitesse.**
> **Hunger Games est cette interface.**

Sans Hunger Games (ou des clients équivalents), les insights s'accumuleraient dans Robotoff sans chemin scalable vers la validation.

**FAIT :** L'URL de production est `https://hunger.openfoodfacts.org`. Base API Robotoff dans le code : `https://robotoff.openfoodfacts.org/api/v1`.

Hunger Games **n'est pas** Robotoff et **n'est pas** le backend OFF. C'est un **participant frontend** dans un écosystème plus large. La Phase 2 cartographiera précisément les frontières.

---

## Pourquoi une interaction « ludique » plutôt que l'édition complète de produit ?

**FAIT (objectif README) :** « Every Open Food Facts user can annotate products in a few minutes. »

L'édition complète de produit (naviguer OFF, trouver le champ, saisir la valeur taxonomique, enregistrer) a une charge cognitive élevée. Hunger Games réduit chaque contribution à :

1. Lire une question en une phrase
2. Jeter un coup d'œil à une image recadrée
3. Appuyer sur Oui / Non / Passer

**INFÉRENCE :** Le cadrage ludique (plusieurs mini-apps, raccourcis clavier, UI playful selon README/liens Figma) augmente le débit et la participation répétée — critique pour vider les arriérés d'insights.

**FAIT (README outstanding issues) :** Les mainteneurs s'inquiètent explicitement de la « cognitive load on the users for the questions » — confirmant que l'efficacité UX est une préoccupation produit de premier ordre, pas une décoration.

---

## Que se passe-t-il quand vous cliquez Oui, Non ou Passer ?

C'est l'histoire runtime la plus importante de la Phase 1. Nous tracerons le mini-jeu **Questions** — le cœur architectural (parcours plus profond en Phase 7).

### Les trois boutons correspondent à trois entiers

**FAIT** (`src/const.ts`) :

```typescript
export const CORRECT_INSIGHT = 1;   // Yes
export const WRONG_INSIGHT = 0;     // No
export const SKIPPED_INSIGHT = -1;  // Skip
```

**FAIT** (`src/pages/questions/QuestionDisplay.tsx`) : les boutons appellent `answerQuestion({ question, answer })`.

### Étape par étape : ce que fait le frontend

**FAIT** (`src/hooks/useQuestions.ts`, `answerQuestion`) :

1. **Envoyer l'annotation à Robotoff (non bloquant)**

   ```typescript
   robotoff.annotate(question.insight_id, answer).catch((err) => {
     console.error("Error while answering question", err);
   });
   ```

   **FAIT** (`src/robotoff.ts`) :

   ```typescript
   annotate(insightId: string, annotation: -1 | 0 | 1) {
     return robotoffClient.annotate({
       insight_id: insightId,
       annotation: annotation,
       update: 1,
     });
   }
   ```

   `update: 1` signifie que Robotoff doit envoyer une mise à jour confirmée à Open Food Facts lorsque la politique le permet (**FAIT**, Robotoff API).

2. **Mettre à jour immédiatement le cache React Query local (UI optimiste)**

   La question répondue est **retirée de la file locale** avant la fin de l'appel réseau :

   ```typescript
   queryClient.setQueryData(keys, (data) => ({
     questions: data.questions.filter(
       (q) => q.insight_id !== question.insight_id,
     ),
     count: data.count !== 100 ? data.count - 1 : 100,
   }));
   ```

   Classification : **optimiste pour la progression de la file**, **fire-and-forget pour la mutation distante** (les erreurs vont seulement dans `console.error`).

3. **Peut-être précharger plus de questions**

   Si la file locale descend à ≤5 mais que le serveur signale qu'il y en a plus, une mutation en arrière-plan refetch et ajoute des questions non dupliquées.

4. **Mettre à jour la mémoire des réponses récentes (Oui/Non uniquement)**

   Passer est intentionnellement exclu de l'historique de la barre latérale des réponses récentes.

5. **Analytics**

   Événement Matomo suivi via `useMatomoTrackAnswerQuestion`.

6. **L'UI affiche la question suivante**

   `question = questions[0]` — le premier élément du tableau en cache après suppression.

### Étape par étape : ce que fait Robotoff (distant)

**FAIT (Robotoff API — Submit an annotation) :**

| Clic | Valeur `annotation` | Signification | Effet sur les données OFF |
|---|---|---|---|
| **Oui** | `1` | L'insight est correct | Avec `update=1`, Robotoff envoie la mise à jour à Open Food Facts si autorisé |
| **Non** | `0` | L'insight est incorrect | Ne **sera pas** appliqué ; signal négatif enregistré |
| **Passer** | `-1` | L'utilisateur ne peut/ne veut pas décider | Insight masqué pour cet utilisateur à l'avenir ; ce n'est pas un rejet du fait lui-même |

Règles supplémentaires (**FAIT**, Robotoff API) :

- **Utilisateur anonyme :** annotation comptée comme un **vote** ; plusieurs votes anonymes identiques peuvent être requis avant application.
- **Utilisateur OFF enregistré (cookie session / Basic Auth) :** le vote peut être **appliqué directement**.
- **Suivi des passes :** le mécanisme de vote mémorise les insights passés par utilisateur/appareil pour que la même question ne réapparaisse pas.

**INFÉRENCE :** Connecté à Open Food Facts dans le même navigateur (credentials inclus via `fetch(..., { credentials: "include" })` dans `robotoff.ts`), votre Oui/Non peut avoir un effet aval immédiat. Anonyme, vous contribuez à un consensus.

### Ce que Hunger Games ne fait *pas* sur Oui/Non/Passer

- N'appelle **pas** directement les API d'édition Product Opener OFF pour les questions binaires standard
- N'**attend pas** le succès de annotate avant d'avancer l'UI
- Ne **restaure pas** l'UI si annotate échoue (logue seulement l'erreur — à investiguer en Phase 7)

---

## Comparaison avec votre parcours

Si vous avez construit des apps Firebase avec UI optimiste :

| Pattern que vous connaissez | Équivalent Hunger Games |
|---|---|
| Firestore `onSnapshot` live data | TanStack Query fetchant Robotoff `/questions/` |
| Optimistic `setDoc` puis reconcile | `setQueryData` retire la question immédiatement ; annotate est async |
| Firebase Auth UID on writes | Cookie session OFF / vote appareil anonyme sur Robotoff |
| Client writes product truth | **Non** — Robotoff possède le chemin validation → mise à jour OFF |

La différence importante : **le propriétaire de l'état serveur pour les annotations est Robotoff, pas Hunger Games**. Le cache frontend est une **file de travail**, pas un état d'annotation faisant autorité.

---

## Un modèle mental compact (gardez-le en tête)

> **Open Food Facts détient les produits et les photos.**
> **Robotoff transforme les photos en insights et questions.**
> **Hunger Games transforme les questions en annotations.**
> **Robotoff transforme les annotations confirmées en mises à jour produit.**

Ou plus court :

> **Photos → predictions → questions → annotations → faits produit**

Vous êtes l'étape humaine au milieu. Hunger Games est la borne d'arcade ; Robotoff est l'arbitre et le marqueur de score ; Open Food Facts est le registre de la ligue.

---

## Documentation vs réalité du code (notes Phase 1)

| Sujet | Documenté | Réalité du code | Classification |
|---|---|---|---|
| Gestionnaire de paquets | README : « Install yarn » (lien classic) | `package.json` : `"packageManager": "yarn@4.18.0"` | **DOCUMENTATION POSSIBLEMENT OBSOLÈTE** — utiliser Yarn 4 / Corepack en pratique |
| Branche par défaut dans CONTRIBUTING | `upstream/master` | Vérifier au clone en Phase 17 | **WORKFLOW DOCUMENTÉ** — confirmer qu'il est encore actuel |
| Contournement CORS | README suggère une extension navigateur | Pas encore vérifié | **WORKFLOW DOCUMENTÉ** — investiguer prudemment en Phase 12 |
| Faute ML dans README | « Machine earning » | — | Obsolescence inoffensive |

---

## Protection de l'espace mental

### À COMPRENDRE MAINTENANT

1. OFF = base produit ouverte ; les photos sont des preuves, pas des données structurées.
2. Robotoff = ML + API insight/question/annotation ; possède l'orchestration de validation.
3. Hunger Games = frontend React ; récupère les questions, envoie les annotations.
4. Insight ≠ question ≠ annotation — couches liées, identifiants/objets API différents.
5. Oui=`1`, Non=`0`, Passer=`-1` ; annotate va vers Robotoff avec `update: 1`.
6. L'UI avance de façon optimiste ; l'échec distant ne logue actuellement que dans la console.

### UTILE PLUS TARD

- Types d'insight exacts (`label`, `brand`, `category`, …) et mapping par type vers les champs OFF
- Flux d'annotation logo (méthodes API séparées dans `robotoff.ts`)
- Seuils de vote pour utilisateurs anonymes
- Cas limites de la logique de remplissage TanStack Query
- Configuration auth/session pour le dev local

### À IGNORER POUR L'INSTANT

- Mini-jeux individuels au-delà du flux Questions central (logos, nutrition, emballage, …)
- Scripts de génération de fichiers taxonomie (`yarn countries`, `yarn nutriments`)
- Workflows CI/CD, Netlify, Crowdin
- Frontières de migration TypeScript entre `.jsx` / `.tsx`

---

## Résumé Phase 1 — retenez ces cinq points

1. **Le problème :** l'échelle — des millions de photos produit nécessitent une validation humaine de faits extraits par ML.
2. **La répartition des rôles :** Robotoff pense ; les humains confirment ; OFF stocke la vérité publiée.
3. **Le rôle de Hunger Games :** minimiser la friction de l'insight à l'annotation via une UX ludique.
4. **Le chemin du clic :** retrait optimiste UI → `robotoff.annotate(insight_id, ±1|0, update=1)` → Robotoff peut mettre à jour OFF.
5. **La frontière :** Hunger Games ne remplace jamais Robotoff ou OFF ; il consomme leurs API.

### Incertitudes à résoudre dans les phases ultérieures

- **INCONNU :** Seuil exact de vote anonyme avant mise à jour OFF (config serveur Robotoff).
- **INCONNU :** Mapping complet de chaque `insight_type` vers les champs Product Opener (nécessite la source ou la doc Robotoff).
- **INCONNU :** Si le dev local fonctionne sans contournements CORS pour tous les endpoints (Phase 12).
- **INFÉRENCE :** Fréquence à laquelle les insights deviennent des questions vs. auto-appliqués — nécessite une investigation côté Robotoff.

---

## Suite

**Phase 2 — L'écosystème Open Food Facts**

Nous placerons Hunger Games dans un diagramme de contexte système aux côtés de :

- Robotoff
- API Open Food Facts / base produit
- `@openfoodfacts/openfoodfacts-nodejs`
- `openfoodfacts-webcomponents`
- services de taxonomie statique

Pour chaque dépendance : qui possède la source de vérité, qui appelle qui, et ce qui traverse la frontière.

---

## Sources primaires consultées

| Source | Ce que nous avons vérifié |
|---|---|
| `README.md` | Objectif du projet, liens écosystème, affirmations setup dev |
| `src/robotoff.ts` | Forme question, appel annotate, URL API, credentials |
| `src/hooks/useQuestions.ts` | File optimiste, annotate fire-and-forget, remplissage |
| `src/const.ts` | Constantes entières annotation, URLs API |
| `src/pages/questions/QuestionDisplay.tsx` | Câblage Oui/Non/Passer |
| [Robotoff API Reference](https://openfoodfacts.github.io/robotoff/references/api/) | Définition insight, sémantique annotate, vote, flag `update` |

---

*Arrêtez-vous ici. Lisez une fois, questionnez tout ce qui semble faux, puis nous passons à la Phase 2.*
