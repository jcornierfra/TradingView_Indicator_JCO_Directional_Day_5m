# JCO Directional Day

![JCO Directional Day](screenshot.png)

Indicateur TradingView Pine Script v6 d'**anticipation du risque d'explosion** sur les sessions NY AM **et NY PM** du NQ Futures (Nasdaq 100), pour le scalp contrarien.

**Module AM** : à partir des informations disponibles **avant l'ouverture NY** (15h30 Paris), classe la journée selon son risque d'explosion ≥ 200 pts en **6 niveaux**, via un score composite à 5 leviers et 2 filtres extrêmes. Verdict posé à **15h20** (10 min avant l'ouverture). Alerte tardive au close de B2 (**15h40**) pour capter les explosions des 2 premières bougies.

**Module PM** (extension v2.2.0, activable) : à partir des informations disponibles **avant l'ouverture NY PM** (18h30 Paris), classe la session selon son risque d'explosion ≥ 150 pts en **7 niveaux**, via un score composite à 5 leviers (dont `am_expl` comme pont AM→PM) et 3 filtres extrêmes (VERT, ROUGE 18h20, ROUGE FOMC). Verdict posé à **18h20**, alerte tardive à **18h40**.

Le dashboard bascule automatiquement de AM à PM à **18h00 Paris**, avec rappel cross-session du verdict de l'autre côté.

Inclut un greffon **FVG (Fair Value Gap)** qui repeint en jaune les bougies au centre d'un gap, avec filtre configurable sur la taille minimum.

---

## Principe

Une **explosion ≥ 200 c2** est définie strictement (cf. PDF source) :

- Séquence de bougies M5 consécutives durant NY AM (15h30–17h30 Paris)
- `|close_dernière_bougie − open_première_bougie| ≥ 200 points`
- Au maximum 2 bougies contraires dans la séquence

L'étude statistique sous-jacente couvre 340 jours (jan. 2025 – mai 2026) avec **46 explosions** observées, soit un baseline de **13.5%** par jour. L'objectif du système est de concentrer ce risque sur quelques zones identifiables, plutôt que de le subir uniformément.

À 15h20 le verdict est posé (5 niveaux). À 15h40, une alerte tardive peut faire basculer un verdict Orange en ROUGE 15h40 — ce qui capte les explosions où les 2 premières bougies M5 sont anormalement larges.

---

## Les 5 leviers du score composite

Chaque condition activée vaut +1 point, sauf « lundi » qui retire 1 point. Score final entre **−1 et +5**.

| # | Levier | Condition | Effet | Justification |
|---|--------|-----------|-------|---------------|
| 1a | `pre_amp_pts` | ≥ 240 | +1 | Range pré-NY 14h00–15h20. Une pré-NY agitée annonce souvent la suite. |
| 1b | `pre_net_pts` | ≤ −110 | +1 | Net pré-NY baissier 14h00–15h20 (effondrement annoncé, asymétrie observée). |
| 3+ | `weekday` | jeudi ou vendredi | +1 | Fins de semaine : ajustements institutionnels, news macro. |
| 3− | `weekday` | lundi | −1 | Lundi statistiquement très calme (4.3% P(expl)). |
| 4 | `expl_count_5d` | ≥ 2 sur 5 jours | +1 | Régime explosif récent — les explosions arrivent en cluster. |
| 5 | `position_in_14d_range` | ≤ 32% | +1 | NQ proche du plus bas 14j → zone de stops, panique potentielle. |

---

## Les 6 niveaux de risque

| Niveau | Règle | P(explosion) | Fréquence | Action |
|--------|-------|--------------|-----------|--------|
| 🟢 **VERT 15h20** | `pre_amp ≤ 130` ET `expl_count_5d = 0` | **0.0%** | 27% du temps | Scalp contrarian sans crainte |
| 🟩 **Orange-faible** | score ≤ 0 | 7.6% | 23% | Contrarian taille standard |
| 🟨 **Orange-moyen** | score 1–2 | 13.9% | 36% | Contrarian, gestion stricte |
| 🟧 **Orange-élevé** | score ≥ 3 | 19.2% | 8% | Taille réduite, stops serrés |
| 🟥 **ROUGE 15h40** | Orange ET `b12_range ≥ 120` ET alignement B1+B2 (V1) | **88.9%** | ~3% | Sortir ou réduire, alerte tardive |
| 🔴 **ROUGE 15h20** | `pre_amp ≥ 280` ET `on_amp/h ≥ 30` ET `position_14d ≤ 32` | **90.9%** | 3% | Pas de contrarian, attendre |

**Précision combinée rouges 15h20+15h40** : **90.0%** (vs 86.4% en évaluation à 15h30).

**Filtre VERT** : 0 explosion sur 93 jours testés (parfait en train ET test).

### Alerte tardive V1 — condition d'alignement directionnel

L'alerte ROUGE 15h40 requiert que **les corps de B1 (15h30–15h35) et B2 (15h35–15h40) aillent dans le même sens**, en plus du critère `b12_range ≥ 120`. Cela filtre les faux positifs *yo-yo* type « spike + retour » où B1 et B2 s'opposent (range élevé mais marché qui range sans direction nette).

**Précision : 72.7% (V0, range seul) → 88.9% (V1, range + alignement)** sur 2025–2026, sans perte de vrais positifs.

Le réglage `require_b12_alignment` permet de revenir à V0 si besoin (par exemple si le régime de marché change).

---

## Les 4 phases temporelles

L'indicateur évolue selon l'heure Paris :

| Phase | Plage | État du score | État de la classification |
|-------|-------|---------------|---------------------------|
| 1. Structurel | 1h00 → 14h00 | 3 leviers stables (weekday, expl_5d, position_14d) | non calculée — fond gris |
| 2. Provisoire | 14h00 → 15h20 | + pre_amp/pre_net partiels, recalculé à chaque bougie M5 | "EN COURS" |
| 3. Verrouillé | 15h20 | score final, classification figée (VERT / ROUGE 15h20 / Orange-X) | "Etat" affiché |
| 4. Ajusté | 15h40+ | inchangé | "Etat ajusté" affiché — bascule éventuelle en ROUGE 15h40 |

Le dashboard distingue **Etat** (verdict 15h20, ne change jamais après) et **Etat ajusté** (classification courante, peut basculer à 15h40).

Reset journalier à **1h00 Paris** (heure fixe) : purge des verdicts de la veille avant que la phase structurelle ne reprenne le calcul du score.

---

## Module NY PM (v2.2.0)

Extension symétrique du module AM, calibrée sur l'étude `NQ_Anticipation_NY_PM_Score_v4.pdf` (334 jours 2025–2026, recall ROUGES = 51%). Activable / désactivable via l'input `Activer le module Risk Assessment NY PM`.

### Les 5 leviers PM (score 0–5)

| # | Levier | Condition | Effet |
|---|--------|-----------|-------|
| 1a | `pre_pm_amp_pts` | ≥ 140 | +1 (range pré-PM 17h20–18h20) |
| 1b | `pre_pm_net_pts` | ≤ −100 | +1 (net pré-PM 17h20–18h20) |
| 2 | `am_expl` | l'AM du jour J a explosé | +1 (pont AM→PM, seuil c2 200) |
| 3 | `expl_pm_count_5d` | ≥ 2 sur 5 jours | +1 (régime explosif PM, seuil c2 150) |
| 4 | `position_in_14d_pm` | ≤ 30% | +1 (proche du plus bas 14j à 18h20) |

Différences clés vs AM :

- Pas de levier calendaire `weekday` (donc score ≥ 0)
- Seuil c2 plus bas (150 vs 200 pts) car l'amplitude PM est mécaniquement plus faible
- `am_expl` remplace le levier `weekday` comme 3ᵉ contributeur

### Les 7 niveaux PM

| Niveau | Règle | Action |
|--------|-------|--------|
| 🔴🔴 **ROUGE FOMC** | `is_fomc == true` (prioritaire sur tout) | Pas de scalp, jour FOMC |
| 🟢 **VERT 18h20** | `pre_pm_amp ≤ 80` ET `expl_pm_5d = 0` | Scalp contrarian sans crainte |
| 🟩 **Orange-faible** | score ≤ 0 | Contrarian taille standard |
| 🟨 **Orange-moyen** | score 1–2 | Contrarian, gestion stricte |
| 🟧 **Orange-élevé** | score ≥ 3 | Taille réduite, stops serrés |
| 🟥 **ROUGE 18h40** | Orange ET `b12_range_pm ≥ 80` (**pas** d'alignement) | Sortir ou réduire, alerte tardive |
| 🔴 **ROUGE 18h20** | `pre_pm_amp ≥ 140` ET `pos_14d ≤ 30` ET `am_expl == true` | Pas de contrarian |

Différence importante vs AM : l'alerte tardive PM **n'exige pas l'alignement directionnel B1+B2** (PDF v4 Section 5). Le seul critère est `b12_range_pm ≥ 80 pts`. C'est plus permissif que la V1 AM, calibré sur le comportement PM observé.

### Liste FOMC

16 dates 2025–2026 codées en dur dans l'indicateur (extensible). Quand le jour J est un FOMC, la classification PM passe automatiquement en **ROUGE FOMC** (prioritaire sur tous les autres filtres).

### Workflow temporel PM

| Heure | Évènement | Affichage |
|-------|-----------|-----------|
| 17h20 | Début accumulation pré-PM | (calculs en arrière-plan) |
| **18h00** | **Bascule visuelle AM → PM** | Le dashboard montre désormais les leviers PM (encore partiels) |
| 18h20 | Décision préparatoire PM | Classification PM verrouillée, `Etat PM` figé |
| 18h30 | Ouverture session PM réelle | Push des bougies PM commence |
| 18h40 | Alerte tardive PM (close B2) | Bascule éventuelle vers **ROUGE 18h40** |
| 20h30 | Fin session PM | Détection c2 PM (seuil 150) + push dans historique 5j |

### Dashboard étendu

Le dashboard bascule à **18h00 Paris** entre layout AM (13 lignes + 1 ligne `Etat PM` cross-session) et layout PM (13 lignes + 1 ligne `Etat AM` cross-session figée). Trois modes inchangés (Masque / Complet 3-col / Complet 2-col / Simplifié).

**Exemple Complet 3-col mode PM** :

```text
Etat AM     │ Orange-moyen   │
Etat PM     │ ROUGE 18h20    │
Score       │ +4             │
P(expl)     │ —              │
─────────────────────────────────
pre_pm_amp  │ 253 pts        │ ✓ (>= 140)
pre_pm_net  │ −180 pts       │ ✓ (<= −100)
am_expl     │ True           │ ✓
expl_pm_5d  │ 2              │ ✓ (>= 2)
pos_14d_pm  │ −46.0%         │ ✓ (<= 30%)
is_fomc     │ False          │
─────────────────────────────────
b12 range   │ —              │
Etat ajuste │ —
```

Pour le module PM désactivé, le dashboard reste exactement comme en v2.1.0 (aucune bascule, aucune référence PM).

---

## Détection des explosions c2 (algorithme strict)

Pour chaque jour passé, on collecte les ~24 bougies M5 de la session AM (15h30–17h30), puis on cherche une paire `(i, j)` avec `j ≥ i+2` telle que :

```text
|close[j] − open[i]| ≥ 200 pts
ET nombre de bougies contraires dans [i..j] ≤ 2
```

Une **bougie contraire** est une bougie dont le corps va à l'opposé du mouvement net.

Cette définition stricte est implémentée en Pine v6 (fonction `f_detect_explosion_c2`) au lieu d'un proxy approximatif `amp_AM ≥ 200`, qui sur-compterait massivement les jours yo-yo violents sans direction nette (89 jours détectés vs 46 réels, casserait la calibration).

L'historique des 5 derniers jours est stocké dans un ring buffer (`recent_expl_days`). À chaque rechargement de l'indicateur, TradingView rejoue l'historique M5 disponible et reconstruit l'état — **pas de période de warm-up nécessaire**.

---

## Dashboard (bas à droite)

Trois modes via le dropdown **« Mode d'affichage »** :

### Mode `Simplifie` (2 colonnes × 4 lignes)

Le plus compact, pour lecture rapide pendant le scalp :

```text
NY AM        risk expl
Etat        │ Orange-eleve
P(expl)     │ 19.2%
Etat ajuste │ ROUGE (apres 15h40, sinon —)
```

### Mode `Complet` — 3 colonnes (par défaut, seuils visibles)

```text
Etat        │ Orange-eleve
Score       │ +5
P(expl)     │ 19.2%
─────────────────────────────────
pre_amp     │ 268 pts     │ ✓ (>= 240)
pre_net     │ −198 pts    │ ✓ (<= −110)
weekday     │ Vendredi    │ +1
expl_5d     │ 5           │ ✓ (>= 2)
pos_14d     │ −1.3%       │ ✓ (<= 32%)
─────────────────────────────────
b12 range   │ 158 pts     │ ✓ (>= 120)
b12 align   │ B1↑ B2↑     │ ✓
Etat ajuste │ ROUGE
```

Les deux lignes `b12 range` / `b12 align` rendent visible le pourquoi de l'alerte tardive : range cumulé B1+B2 et alignement directionnel. Avant 15h40 elles affichent `—`.

### Mode `Complet` — 2 colonnes (compact, seuils masqués)

Identique au précédent mais sans la 3e colonne. La couleur de chaque valeur de levier indique son statut :

- **Vert** : levier activé (contribue au score)
- **Rouge** : weekday Jeu/Ven (lever +1, augmente le risque) — sémantique inverse
- **Vert** : weekday Lundi (lever −1, réduit le risque)
- **Gris** : levier non activé ou neutre
- Pour `b12 range` : vert si ≥ seuil
- Pour `b12 align` : vert si aligné, rouge si yo-yo

### Couleur de fond

Le fond du panneau reflète la **classification courante** (incluant ROUGE 15h40 si l'alerte se déclenche), pour une lecture visuelle immédiate de l'urgence.

| Classification | Couleur de fond |
|----------------|-----------------|
| VERT 15h20 | vert pâle |
| Orange-faible | vert-jaune pâle |
| Orange-moyen | jaune pâle |
| Orange-élevé | orange pâle |
| ROUGE 15h40 | rouge clair (coral) |
| ROUGE 15h20 | rouge foncé |

---

## Source statistique

PDF *« Anticiper les explosions sur la session NY AM »* (Mai 2026) :

- 340 jours analysés · NQ M5 · 1 jan. 2025 – 8 mai 2026
- 46 explosions ≥ 200 pts c2 détectées (baseline 13.5%)
- Calibration : train 2025 / test 2026 (hold-out)
- 9 pistes testées, 5 retenues dans le score composite

Toutes les valeurs en dur dans le code (seuils, bandes du score, mapping des couleurs) proviennent directement du PDF et sont surchargeables via inputs.

---

## Paramètres

### Session NY AM

- **Heure debut NY AM (Paris)** : 15 (défaut)
- **Minute debut NY AM** : 30 (défaut)

Toutes les autres bornes temporelles (reset 1h00, pré-NY 14h00, évaluation 15h20, B2 close 15h40, fin AM 17h30) sont dérivées de ce point, et le DST est géré automatiquement.

### Dashboard

- **Mode d'affichage** : `Masque` / `Complet` / `Simplifie`
- **Afficher les seuils en 3e colonne, mode Complet** : on/off (défaut on)
- **Lignes vides en bas** : 0–20 (défaut 7) — pour empiler un dashboard secondaire

### Score composite

- **Seuil pre_amp pour +1 au score** : 240 pts
- **Seuil pre_net pour +1 au score** : −110 pts
- **Nb explosions sur 5j pour +1** : 2
- **Position dans range 14j (%) pour +1** : 32

### Filtres extrêmes

- **Seuil pre_amp max pour filtre VERT** : 130 pts
- **Seuil pre_amp min pour filtre ROUGE 15h20** : 280 pts
- **Seuil on_amp/h min pour ROUGE 15h20** : 30 pts/heure
- **Seuil position_14d max pour ROUGE 15h20** : 32%
- **Seuil b12_range pour ROUGE 15h40** : 120 pts
- **Alerte 15h40 : exiger l'alignement B1+B2** : on/off (défaut on, recommandé). V1 du filtre — décocher pour revenir au comportement V0 (range seul).

### Paramètres techniques

- **Seuil pts pour explosion c2 AM** : 200
- **Max bougies contraires dans une séquence c2** : 2
- **Fenêtre du range structurel** : 14 jours
- **Fenêtre du compteur d'explosions** : 5 jours

### Module PM (v2.2.0)

- **Activer le module Risk Assessment NY PM** : on/off (défaut on). Si décoché, le dashboard reste en comportement v2.1.0.
- **Seuil pre_pm_amp pour +1 au score PM** : 140 pts
- **Seuil pre_pm_net pour +1 au score PM** : −100 pts
- **Nb explosions PM sur 5j pour +1 au score PM** : 2
- **Position dans range 14j (%) pour +1 au score PM** : 30
- **Seuil pre_pm_amp max pour filtre VERT PM** : 80 pts
- **Seuil pre_pm_amp min pour filtre ROUGE 18h20** : 140 pts
- **Seuil position_14d_pm max pour ROUGE 18h20** : 30%
- **Seuil b12_range_pm pour ROUGE 18h40** : 80 pts (pas d'alignement requis)
- **Seuil pts pour explosion c2 PM** : 150

### FVG

- **Afficher les bougies FVG** : on (défaut)
- **Couleur du corps** : jaune
- **Écart minimum (pts)** : 1

---

## Installation

1. Ouvrir TradingView → Pine Script Editor
2. Coller le contenu de [`Indicator_JCO_Directional_Day_5m.pine`](Indicator_JCO_Directional_Day_5m.pine)
3. Cliquer sur **Ajouter au graphique**
4. Appliquer sur un graphique NQ Futures (CME) en **M5**

---

## Compatibilité

- **Symbole recommandé** : `NQ1!` / `MNQ1!` (CME Globex)
- **Timeframe requis** : **M5** (les bornes 15h30/15h35/15h40 sont calibrées sur l'ouverture des bougies M5)
- **Pine Script** : v6
- **DST** : géré automatiquement (CEST / CET Paris)

---

## Limites à connaître

1. **Calibration sur 340 jours** : échantillon raisonnable mais limité. Le filtre ROUGE 15h20 perd un peu de précision sur le hold-out 2026 (75% vs 100% train).
2. **~12% des explosions restent imprévisibles** : aucun levier ne signale parfois, et le système le sait. C'est la limite irréductible.
3. **Marché qui change** : la calibration est faite sur le régime 2025–mi 2026. Si le marché redevient calme façon 2023–2024, les seuils ne seront plus pertinents et il faudra recalibrer.
4. **Détection c2 stricte** : la fonction tourne en O(n³) mais sur n=24, ce qui prend quelques millisecondes par jour à 17h30. Pas d'impact en historique ni en temps réel.
5. **Pas d'oracle** : le système transforme un risque diffus en risque concentré et hiérarchisé. Il aligne la taille des positions sur la probabilité d'explosion, il n'élimine pas le risque.

---

## Changelog

### v2.2.0 - 2026-05-12

**Extension : module Risk Assessment NY PM**. Le module AM est inchangé fonctionnellement.

- Source : étude `NQ_Anticipation_NY_PM_Score_v4.pdf` (334 jours, recall ROUGES 51%).
- Session PM 18h30–20h30 Paris : évaluation à **18h20**, alerte tardive à **18h40**, fin à 20h30.
- **Score composite PM à 5 leviers**, plage 0–5 :
  - `pre_pm_amp_pts ≥ 140` (range 17h20–18h20)
  - `pre_pm_net_pts ≤ −100`
  - `am_expl == true` (pont AM→PM, levier inédit)
  - `expl_pm_count_5d ≥ 2` (régime explosif PM, seuil c2 PM = 150 pts)
  - `position_14d_pm ≤ 30%`
- **7 niveaux de risque PM** : ROUGE FOMC (prioritaire), VERT 18h20, ROUGE 18h20, Orange-faible/moyen/élevé, ROUGE 18h40 (alerte tardive, **sans alignement** — différence clé vs AM).
- **Détection c2 PM** avec seuil 150 pts (vs 200 AM), ring buffer `recent_pm_expl_days` séparé.
- **Liste FOMC 2025–2026** codée en dur (16 dates). Extensible.
- **Dashboard étendu** : bascule visuelle AM → PM à 18h00 Paris. Trois modes étendus (Complet 3-col, Complet 2-col, Simplifié), chacun avec un layout PM dédié de 14 lignes. Rappel cross-session : ligne `Etat AM` (figé) sur le panneau PM, ligne `Etat PM` (à venir) sur le panneau AM.
- **Module désactivable** via input `Activer le module Risk Assessment NY PM` (défaut on). Si décoché, comportement strictement identique à v2.1.0.

### v2.1.0 - 2026-05-12

**Évaluation décalée de 15h30 à 15h20** (10 min avant l'ouverture NY AM), suite à la calibration PDF v2.

- `pre_amp` / `pre_net` accumulés sur **14h00–15h20** (16 bougies M5 au lieu de 18). Corrélation 0.98 avec les valeurs 15h30 — diagnostic identique en pratique.
- `on_amp_per_hour` : durée overnight passe de 17.4167h à **17.25h** (22h05 J−1 → 15h20 J).
- `position_14d` : basé sur l'open de la bougie 15h20–15h25.
- Alerte tardive renommée **15h45 → 15h40** (instant inchangé : close de B2). L'attente jusqu'à 15h45 n'apportait aucune information supplémentaire.
- Renommages classification : VERT/ROUGE 15h30 → VERT/ROUGE **15h20**, ROUGE 15h45 → ROUGE **15h40**.
- Probabilités par classe recalibrées (PDF v2) : Orange-faible 8.1% → 7.6%, Orange-moyen 12.0% → 13.9%, Orange-élevé 19.4% → 19.2%, ROUGE 15h40 72.7% → **88.9%**, ROUGE 15h20 91.7% → 90.9%. VERT inchangé à 0.0%.
- **Gains opérationnels** : +10 min de préparation avant ouverture, +6 jours VERT (93 vs 87, toujours 0% explosion), précision combinée ROUGE 90.0% vs 86.4%, alerte tardive 88.9% vs 80.0%.

Le push des bougies AM (15h30 → 17h30) et la détection c2 restent inchangés.

### v2.0.2 - 2026-05-11

- **Phase 1 (Structurel) implémentée**. Avant cette version, le dashboard affichait le verdict de la veille jusqu'à 14h00 (fond coloré avec une classe héritée). Le nouveau trigger `fire_daily_reset` à **1h00 Paris** (heure fixe pour éviter les problèmes de TZ) purge tous les verdicts et données de la veille.
- **Score structurel** entre 1h00 et 14h00 : recalcul continu avec les 3 leviers stables (`weekday`, `expl_count_5d`, `position_14d` estimée via close). Fond gris du panneau, leviers `pre_amp` / `pre_net` affichent `—`.
- **Dashboard** : label `Structurel` pour l'Etat entre 1h00 et 14h00, puis `EN COURS` entre 14h00 et 15h20 (ex-15h30), puis classification finale.
- **Fix** helper `f_paris_to_utc_min` : wrap modulo 1440 quand le résultat est négatif. Sans ce wrap, 01h00 Paris en CEST (été) donnait `60 − 120 = −60`, valeur jamais atteinte par `utc_total` ∈ [0, 1439], et le trigger `fire_daily_reset` ne s'activait jamais en été.

### v2.0.1 - 2026-05-11

- **Fix** : `classification_15h30` n'était reset qu'à 15h30 et pas à 14h00. Conséquence : pendant la pré-NY le dashboard affichait le verdict de la veille avec un fond gris (incohérent). Reset ajouté au trigger pré-NY start.
- **Alerte tardive 15h45 V1** : ajout d'une condition d'alignement directionnel sur B1 et B2. L'alerte ne se déclenche que si les corps de B1 et B2 vont dans le même sens, en plus du critère `b12_range ≥ 120`. Filtre les faux positifs *yo-yo* (spike + retour).
  - Sur les 11 alertes V0 (2025–2026) : 8 vraies explosions, 3 faux positifs dont 1 yo-yo filtré par V1.
  - Précision : 72.7% (V0) → **80.0% (V1)** sans perte de vrai positif.
  - Nouvel input `Alerte 15h45 : exiger l'alignement B1+B2` (défaut on) pour pouvoir revenir au comportement V0.
- **Dashboard Complet** : 2 nouvelles lignes `b12 range` et `b12 align` (avant `Etat ajusté`) pour visualiser le détail de la condition d'alerte. Table passe de 11 à 13 lignes.

### v2.0.0 - 2026-05-11

**REFONTE MAJEURE** — remplacement intégral du module statistique v1.x par le module Risk Assessment basé sur l'étude *« Anticipation des explosions NY AM »*.

- Score composite à 5 leviers + 2 filtres extrêmes + alerte tardive 15h45
- Détection exacte des explosions c2 (pas de proxy approximatif)
- 6 niveaux de risque avec couleurs de fond dédiées
- 4 phases d'affichage : structurel / provisoire / verrouillé / ajusté
- Dashboard refondu :
  - Complet 3 colonnes (Label / Valeur / Indicateur seuil)
  - Complet 2 colonnes compact (couleur des leviers indiquant validation)
  - Simplifié 2 colonnes × 4 lignes (titre / Etat / P(expl) / Etat ajusté)
- Conservés de v1.3.1 : style header, helpers DST, greffon FVG, palette de couleurs (vert / gold / rouge / bleu), structure dashboard (table transparente + padding configurable), dropdown Mode d'affichage Masque/Complet/Simplifie

### v1.3.1 - 2026-05-05

- Dashboard simplifié : suppression du préfixe `Retr` sur les lignes de pourcentage.

### v1.3 - 2026-05-05

- Palette de couleurs uniformisée avec *JCO NY Amplitude Levels* (jaune assombri `#c99a2c`).
- Dropdown **« Mode d'affichage »** à 3 options remplace le toggle on/off.
- Nouveau dashboard simplifié pour lecture rapide.

### v1.2 - 2026-05-04

- Ajout du greffon FVG (Fair Value Gap).
- Dashboard repositionné en `bottom_right` avec padding configurable.
- Refactor du fond du dashboard pour empilage propre avec un dashboard secondaire.

### v1.1 - 2026-05-04

- Snapshot supplémentaire à 17h30, capture de l'amplitude finale NY AM.
- Bloc PM dans le dashboard (continuation + retracement Section 3).

### v1.0 - 2026-05-04

- Initial release : module statistique CAT1A/Timing avec pondération log-odds par jour-de-semaine et mois.

---

## Licence

[Mozilla Public License 2.0](https://mozilla.org/MPL/2.0/)

© jcornier — [GitHub](https://github.com/jcornierfra/TradingView_Indicator_JCO_Directional_Day_5m)
