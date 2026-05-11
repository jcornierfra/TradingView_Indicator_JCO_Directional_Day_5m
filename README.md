# JCO Directional Day

![JCO Directional Day](screenshot.png)

Indicateur TradingView Pine Script v6 d'**anticipation du risque d'explosion** sur la session NY AM du NQ Futures (Nasdaq 100), pour le scalp contrarien.

À partir des informations disponibles **avant l'ouverture NY**, il classe la journée selon son risque d'explosion ≥ 200 pts en **6 niveaux**, via un score composite à 5 leviers et 2 filtres extrêmes. Une alerte tardive à 15h40 permet de capter les explosions qui se révèlent dès les premières bougies AM.

Inclut un greffon **FVG (Fair Value Gap)** qui repeint en jaune les bougies au centre d'un gap, avec filtre configurable sur la taille minimum.

---

## Principe

Une **explosion ≥ 200 c2** est définie strictement (cf. PDF source) :

- Séquence de bougies M5 consécutives durant NY AM (15h30–17h30 Paris)
- `|close_dernière_bougie − open_première_bougie| ≥ 200 points`
- Au maximum 2 bougies contraires dans la séquence

L'étude statistique sous-jacente couvre 340 jours (jan. 2025 – mai 2026) avec **46 explosions** observées, soit un baseline de **13.5%** par jour. L'objectif du système est de concentrer ce risque sur quelques zones identifiables, plutôt que de le subir uniformément.

À 15h30 le verdict est posé (5 niveaux). À 15h40, une alerte tardive peut faire basculer un verdict Orange en ROUGE 15h45 — ce qui capte les explosions où les 2 premières bougies M5 sont anormalement larges.

---

## Les 5 leviers du score composite

Chaque condition activée vaut +1 point, sauf « lundi » qui retire 1 point. Score final entre **−1 et +5**.

| # | Levier | Condition | Effet | Justification |
|---|--------|-----------|-------|---------------|
| 1a | `pre_amp_pts` | ≥ 240 | +1 | Range pré-NY 14h00–15h30. Une pré-NY agitée annonce souvent la suite. |
| 1b | `pre_net_pts` | ≤ −110 | +1 | Net pré-NY baissier (effondrement annoncé, asymétrie observée). |
| 3+ | `weekday` | jeudi ou vendredi | +1 | Fins de semaine : ajustements institutionnels, news macro. |
| 3− | `weekday` | lundi | −1 | Lundi statistiquement très calme (4.3% P(expl)). |
| 4 | `expl_count_5d` | ≥ 2 sur 5 jours | +1 | Régime explosif récent — les explosions arrivent en cluster. |
| 5 | `position_in_14d_range` | ≤ 32% | +1 | NQ proche du plus bas 14j → zone de stops, panique potentielle. |

---

## Les 6 niveaux de risque

| Niveau | Règle | P(explosion) | Fréquence | Action |
|--------|-------|--------------|-----------|--------|
| 🟢 **VERT 15h30** | `pre_amp ≤ 130` ET `expl_count_5d = 0` | **0.0%** | 26% du temps | Scalp contrarian sans crainte |
| 🟩 **Orange-faible** | score ≤ 0 | 8.1% | 22% | Contrarian taille standard |
| 🟨 **Orange-moyen** | score 1–2 | 12.0% | 37% | Contrarian, gestion stricte |
| 🟧 **Orange-élevé** | score ≥ 3 | 19.4% | 9% | Taille réduite, stops serrés |
| 🟥 **ROUGE 15h45** | Orange ET `b12_range ≥ 120` (à 15h40) | **72.7%** | 3% | Sortir ou réduire, alerte tardive |
| 🔴 **ROUGE 15h30** | `pre_amp ≥ 280` ET `on_amp/h ≥ 30` ET `position_14d ≤ 32` | **91.7%** | 4% | Pas de contrarian, attendre |

**Recall combiné rouges 15h30+15h45** : 41.3% des explosions captées sur 7% du temps avec 82.6% de précision.

**Filtre VERT** : 0 explosion sur 87 jours testés (parfait en train ET test).

---

## Les 4 phases temporelles

L'indicateur évolue selon l'heure Paris :

| Phase | Plage | État du score | État de la classification |
|-------|-------|---------------|---------------------------|
| 1. Structurel | avant 14h00 | 3 leviers stables (weekday, expl_5d, position_14d) | non calculée |
| 2. Provisoire | 14h00 → 15h30 | + pre_amp/pre_net partiels, recalculé à chaque bougie M5 | "EN COURS" |
| 3. Verrouillé | 15h30 | score final, classification figée (VERT / ROUGE 15h30 / Orange-X) | "Etat" affiché |
| 4. Ajusté | 15h40+ | inchangé | "Etat ajusté" affiché — bascule éventuelle en ROUGE 15h45 |

Le dashboard distingue **Etat** (verdict 15h30, ne change jamais après) et **Etat ajusté** (classification courante, peut basculer à 15h40).

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
P(expl)     │ 19.4%
Etat ajuste │ ROUGE (apres 15h40, sinon —)
```

### Mode `Complet` — 3 colonnes (par défaut, seuils visibles)

```text
Etat        │ Orange-eleve
Score       │ +5
P(expl)     │ 19.4%
─────────────────────────────────
pre_amp     │ 268 pts     │ ✓ (>= 240)
pre_net     │ −198 pts    │ ✓ (<= −110)
weekday     │ Vendredi    │ +1
expl_5d     │ 5           │ ✓ (>= 2)
pos_14d     │ −1.3%       │ ✓ (<= 32%)
─────────────────────────────────
Etat ajuste │ ROUGE
```

### Mode `Complet` — 2 colonnes (compact, seuils masqués)

Identique au précédent mais sans la 3e colonne. La couleur de chaque valeur de levier indique son statut :

- **Vert** : levier activé (contribue au score)
- **Rouge** : weekday Jeu/Ven (lever +1, augmente le risque) — sémantique inverse
- **Vert** : weekday Lundi (lever −1, réduit le risque)
- **Gris** : levier non activé ou neutre

### Couleur de fond

Le fond du panneau reflète la **classification courante** (incluant ROUGE 15h45 si l'alerte se déclenche), pour une lecture visuelle immédiate de l'urgence.

| Classification | Couleur de fond |
|----------------|-----------------|
| VERT 15h30 | vert pâle |
| Orange-faible | vert-jaune pâle |
| Orange-moyen | jaune pâle |
| Orange-élevé | orange pâle |
| ROUGE 15h45 | rouge clair (coral) |
| ROUGE 15h30 | rouge foncé |

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

Toutes les autres bornes temporelles (pré-NY 14h00, B2 close 15h40, fin AM 17h30) sont dérivées de ce point, et le DST est géré automatiquement.

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
- **Seuil pre_amp min pour filtre ROUGE 15h30** : 280 pts
- **Seuil on_amp/h min pour ROUGE 15h30** : 30 pts/heure
- **Seuil position_14d max pour ROUGE 15h30** : 32%
- **Seuil b12_range pour ROUGE 15h45** : 120 pts

### Paramètres techniques

- **Seuil pts pour explosion c2** : 200
- **Max bougies contraires dans une séquence c2** : 2
- **Fenêtre du range structurel** : 14 jours
- **Fenêtre du compteur d'explosions** : 5 jours

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

1. **Calibration sur 340 jours** : échantillon raisonnable mais limité. Le filtre ROUGE 15h30 perd un peu de précision sur le hold-out 2026 (75% vs 100% train).
2. **~12% des explosions restent imprévisibles** : aucun levier ne signale parfois, et le système le sait. C'est la limite irréductible.
3. **Marché qui change** : la calibration est faite sur le régime 2025–mi 2026. Si le marché redevient calme façon 2023–2024, les seuils ne seront plus pertinents et il faudra recalibrer.
4. **Détection c2 stricte** : la fonction tourne en O(n³) mais sur n=24, ce qui prend quelques millisecondes par jour à 17h30. Pas d'impact en historique ni en temps réel.
5. **Pas d'oracle** : le système transforme un risque diffus en risque concentré et hiérarchisé. Il aligne la taille des positions sur la probabilité d'explosion, il n'élimine pas le risque.

---

## Changelog

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
