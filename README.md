# Moteur de recherche d'articles scientifiques
### Prototype d'assistance à l'estimation des tailles d'effet — Elements Impact

---

## Contexte

Ce prototype s'inscrit dans le projet **Boussole**, outil d'aide à la décision développé par Elements Impact pour calculer l'impact de projets sur le bien-être subjectif (SWB).

Le modèle mathématique de Boussole repose sur les travaux de Margolis *et al.*, qui fournissent les coefficients β d'un modèle de régression sur le SWB à partir de 79 prédicteurs. Pour utiliser ce modèle sur un projet réel, il faut estimer **la taille d'effet** de l'action menée sur chacun de ces prédicteurs — une taille d'effet s'exprimant généralement en **d de Cohen** ou **g de Hedge**.

Cette recherche doit être réalisée pour chaque groupe de bénéficiaires, pour chaque prédicteur pertinent, à partir de la littérature scientifique. C'est un exercice pénible, chronophage et soumis à de nombreuses incertitudes.

**Objectif du prototype** : automatiser une première étape de ce travail en proposant, à partir d'une description du projet, une liste classée d'articles scientifiques potentiellement exploitables pour estimer ces tailles d'effet.

---

## Pipeline

Le notebook suit une architecture en trois étapes séquentielles :

```
Inputs (5 champs textuels)
         │
         ▼
┌─────────────────────────────┐
│  1. COLLECTER               │
│  OpenAlex API               │
│  6 requêtes parallèles      │
│  → ~200 candidats bruts     │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  2. SCORER                  │
│  Scoring heuristique        │
│  12 signaux lexicaux        │
│  → Top N candidats classés  │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  3. ÉVALUER (optionnel)     │
│  Gemini 2.5 Flash           │
│  Prompt structuré JSON      │
│  → Shortlist avec labels    │
└─────────────────────────────┘
```

---

## Structure du notebook

| Section | Titre | Description |
|---------|-------|-------------|
| 1 | Secrets and configuration | Chargement de la clé API Gemini (Kaggle secret ou variable d'environnement) |
| 2 | Search inputs | Paramètres de la recherche à éditer par l'utilisateur |
| 3 | Text helpers | Normalisation, tokenisation, suppression des stopwords |
| 4 | Group expansion & effect signal | Expansion des termes de groupe, détection de signaux quantitatifs |
| 5 | Build OpenAlex queries | Construction des 6 requêtes parallèles |
| 6 | OpenAlex retrieval & deduplication | Appels API, pagination par curseur, déduplication |
| 7 | Heuristic ranking | Calcul du score heuristique sur 12 composantes |
| 8 | Retrieve and rank | Exécution du pipeline de collecte et classement |
| 9 | Optional Gemini reranking | Appel Gemini par batches sur le top N |
| 10 | Run Gemini & diagnostics | Exécution, affichage des comptages de labels, diagnostics d'erreur |
| 11 | Display helpers | Fonctions de rendu HTML du tableau final |
| 12 | Display final shortlist | Affichage du tableau de résultats stylisé |

---

## Entrées

Le notebook prend **5 champs textuels libres** à renseigner dans la section 2 :

```python
objective  = "socio emotional development"
context    = "parenting support program"
group      = "children"
predictor  = "family support"
tags       = "early childhood, parenting"
```

| Champ | Rôle |
|-------|------|
| `objective` | Outcome d'intérêt (ce que l'on cherche à mesurer) |
| `context` | Cadre de l'intervention (type de programme, setting) |
| `group` | Population cible |
| `predictor` | Prédicteur ou levier d'action (ce dont on cherche l'effet) |
| `tags` | Mots-clés complémentaires pour élargir la couverture |

---

## Paramètres configurables

```python
YEAR_MIN                    = 2000      # Année minimale de publication
OPENALEX_PER_PAGE           = 50        # Résultats par page OpenAlex
OPENALEX_MAX_PAGES_PER_QUERY = 2        # Pages maximum par requête
MAX_CANDIDATES_AFTER_DEDUP  = 250       # Candidats retenus après déduplication
TOP_N_FOR_LLM               = 24        # Articles envoyés à Gemini
GEMINI_BATCH_SIZE           = 8         # Articles par appel Gemini
OPENALEX_MAILTO             = "..."     # Email pour le mode "polite" OpenAlex
```

---

## Détail technique des étapes

### Étape 1 — Collecte via OpenAlex

L'API OpenAlex est interrogée **sans clé**, via le [mode polite](https://docs.openalex.org/how-to-use-the-api/rate-limits-and-authentication) (paramètre `mailto`). Le notebook construit **6 requêtes parallèles** à partir des 5 champs d'entrée, en combinant différemment les termes :

| Requête | Combinaison |
|---------|-------------|
| Q1 | `predictor` + `context` |
| Q2 | `predictor` + `objective` |
| Q3 | `objective` + `context` + `group` |
| Q4 | `predictor` + `group` + `tags` |
| Q5 | `tags` + `objective` |
| Q6 | Tous les termes (union, max 10) |

Les résultats sont fusionnés et dédupliqués par `id` OpenAlex. Un champ `query_hit_count` comptabilise le nombre de requêtes distinctes ayant retourné chaque article — c'est un signal de robustesse inter-requêtes.

La pagination utilise le mécanisme de **curseur** d'OpenAlex. Les abstracts sont reconstruits depuis l'index inversé (`abstract_inverted_index`) fourni par l'API.

**Filtre appliqué** : `type:article|review`, `from_publication_date:{YEAR_MIN}-01-01`, tri par `relevance_score:desc`.

---

### Étape 2 — Scoring heuristique

Le score heuristique est calculé sur **12 composantes lexicales**, dont la somme des poids est exactement **1.0** :

| Composante | Poids | Description |
|------------|-------|-------------|
| `predictor_overlap` | 0.183 | Proportion de termes du prédicteur trouvés dans titre+abstract |
| `context_overlap` | 0.150 | Idem pour le contexte |
| `objective_overlap` | 0.133 | Idem pour l'objectif |
| `group_overlap` | 0.067 | Idem pour le groupe (avec expansion sémantique) |
| `tags_overlap` | 0.050 | Idem pour les tags |
| `predictor_phrase` | 0.083 | Bonus si le prédicteur apparaît comme phrase exacte |
| `context_phrase` | 0.050 | Idem pour le contexte |
| `objective_phrase` | 0.033 | Idem pour l'objectif |
| `title_pred_bonus` | 0.042 | Bonus si le prédicteur est dans le titre |
| `title_ctx_bonus` | 0.025 | Bonus si le contexte est dans le titre |
| `effect_signal` | 0.117 | Détection de termes quantitatifs (Cohen, SMD, RCT, OR…) |
| `metadata_signal` | 0.067 | Score composite : présence d'abstract, récence, citations |

**Détail du signal taille d'effet (`effect_signal`)** : 0.0 si aucun terme détecté, 0.4 pour 1 terme, 0.7 pour 2 termes, 1.0 pour 3 termes ou plus. Les termes surveillés incluent : `cohen's d`, `hedges g`, `standardized mean difference`, `smd`, `odds ratio`, `meta analysis`, `randomized trial`, `regression coefficient`, etc.

**Détail du `metadata_signal`** : +0.4 si un abstract est présent, +0.4 modulé par la récence (décroissance linéaire sur 20 ans), +0.2 plafonné par `log1p(citations) / 6`.

**Expansion du groupe** : le champ `group` est étendu via un dictionnaire de synonymes (ex. `"children"` → `["child", "youth", "adolescent", "pediatric", "school-age"]`).

---

### Étape 3 — Évaluation Gemini (optionnelle)

Le top `TOP_N_FOR_LLM` articles issus du classement heuristique est soumis à **Gemini 2.5 Flash** par batches de `GEMINI_BATCH_SIZE` articles. Une pause de **15 secondes** est respectée entre chaque batch pour ne pas dépasser le rate limit du free tier.

Le prompt demande à Gemini d'évaluer chaque article sur **7 dimensions** et de répondre en JSON strict :

```json
{
  "article_index": 0,
  "relevance_label": "highly_relevant | somewhat_relevant | background_only | irrelevant",
  "usable_for_effect_estimation": true,
  "study_type": "trial | observational | meta_analysis | review | commentary | unclear",
  "action_match": 0,
  "group_match": 1,
  "quantitative_signal": 2,
  "short_reason": "Max 30 mots."
}
```

| Champ | Valeurs | Signification |
|-------|---------|---------------|
| `relevance_label` | 4 niveaux | Pertinence globale au regard du predictor, context et group |
| `usable_for_effect_estimation` | `true/false` | L'article contient-il une comparaison quantitative exploitable ? |
| `study_type` | 6 types | Catégorie méthodologique de l'étude |
| `action_match` | 0 / 1 / 2 | Correspondance entre le predictor et l'action étudiée |
| `group_match` | 0 / 1 / 2 | Correspondance entre le group et la population de l'étude |
| `quantitative_signal` | 0 / 1 / 2 | Présence de données quantitatives empiriques |
| `short_reason` | ≤ 30 mots | Justification libre |

**Principe clé du prompt** : un article est jugé exploitable même sans mention explicite de "taille d'effet" — un RCT, une régression, un odds ratio, ou une méta-analyse avec résultats poolés sont tous jugés pertinents.

**Mode dégradé** : si Gemini est indisponible (quota épuisé ou clé absente), tous les champs Gemini sont remplis avec des valeurs par défaut (`not_scored`, `False`, `0`) et le classement final repose uniquement sur le score heuristique.

---

## Sortie

Le notebook affiche un tableau HTML stylisé (code couleur par label de pertinence) contenant, pour chaque article du shortlist final :

| Colonne | Description |
|---------|-------------|
| `Rank` | Rang dans le classement final |
| `Title` | Titre de l'article |
| `Year` | Année de publication |
| `Journal` | Revue |
| `Heuristic` | Score heuristique (0–1) |
| `Final` | Score final combiné (0–1) |
| `Gemini label` | Label de pertinence Gemini |
| `Effect estimation?` | Exploitable pour estimer un effet ? |
| `Action` | Score correspondance action (0–2) |
| `Group` | Score correspondance groupe (0–2) |
| `QuantSignal` | Signal quantitatif (0–2) |
| `Reason` | Justification Gemini (≤ 30 mots) |
| `URL` | Lien vers l'article |

**Code couleur** :
- 🟢 `highly_relevant`
- 🟡 `somewhat_relevant`
- 🔵 `background_only`
- 🔴 `irrelevant`
- ⚫ `not_scored` / `error`

---

## Dépendances

```
requests
pandas
google-genai
```

L'installation de `google-genai` est effectuée directement dans le notebook via `pip install -q -U google-genai`.

Aucune autre dépendance externe n'est requise. L'API OpenAlex ne nécessite pas de clé.

---

## Configuration de la clé Gemini

Le notebook cherche la clé dans l'ordre suivant :

1. Secret Kaggle nommé `GEMINI_API_KEY` (via `kaggle_secrets.UserSecretsClient`)
2. Variable d'environnement `GEMINI_API_KEY`
3. Si aucune clé n'est trouvée : mode heuristique seul, sans interruption

```python
try:
    from kaggle_secrets import UserSecretsClient
    GEMINI_API_KEY = UserSecretsClient().get_secret("GEMINI_API_KEY")
except Exception:
    GEMINI_API_KEY = os.environ.get("GEMINI_API_KEY", None)
```

---

## Limites connues

- **Couverture** : limitée aux articles indexés dans OpenAlex avec abstract disponible. Les articles sans abstract sont collectés mais pénalisés dans le score heuristique.
- **Troncature des abstracts** : seuls les 300 premiers caractères de l'abstract sont transmis à Gemini, pour des raisons de quota de tokens.
- **Quota Gemini** : le free tier de Gemini 2.5 Flash impose des limites de requêtes par minute et par jour. La pause de 15 secondes entre batches est un paliatif, non une garantie.
- **Absence de gold standard** : le prototype n'a pas été évalué sur un corpus d'articles annotés manuellement. La qualité du classement n'est pas mesurée quantitativement.
- **Extraction des tailles d'effet** : le prototype identifie des articles potentiellement exploitables mais n'extrait pas automatiquement les valeurs numériques de Cohen's d ou Hedge's g.
- **Biais de publication** : OpenAlex surreprésente les articles en accès ouvert et les études aux résultats positifs.

---

## Exemple d'utilisation

```
objective  = "socio emotional development"
context    = "parenting support program"
group      = "children"
predictor  = "family support"
tags       = "early childhood, parenting"
```

Résultat attendu : liste d'articles classés, dont les plus pertinents sont des RCT ou études observationnelles évaluant l'effet d'un soutien parental sur le développement socio-émotionnel de l'enfant, avec un label `highly_relevant` et `usable_for_effect_estimation = true`.

---

## Références

- [OpenAlex API documentation](https://docs.openalex.org/)
- [Google Gemini API (google-genai)](https://ai.google.dev/gemini-api/docs)
- Margolis *et al.* — modèle de régression SWB (base théorique de Boussole)
- Frijters *et al.* (2024) — *The WELLBY: a new measure of social value and progress*
