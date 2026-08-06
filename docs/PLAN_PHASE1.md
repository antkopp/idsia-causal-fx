# Phase 1 — Données & ETL : spécification détaillée

Objectif : livrer un panel de **séries de force par devise**, synchrone, stationnarisé, standardisé,
avec assez d'échantillons (V3), stocké et partageable avec Mert (V12). Six étapes.

---

## 1.1 Univers de devises

**Nœuds = devises (pas les paires)** — décision actée 14/07.

Univers en **deux étages** (simplification actée le 05/08 — pas d'étage « témoins » séparé, la falsification
est une règle d'analyse, pas une structure de données) :

- **Étage 1 — Hard (10, G10)** : USD, EUR, JPY, GBP, CHF, AUD, NZD, CAD, SEK, NOK. Le cœur de l'étude.
- **Étage 2 — Soft (6 à 9, flotteurs)** : MXN, ZAR, PLN, HUF, CZK, ILS + **CLP, COP, PHP sous condition de
  sonde** (t0 et taux de cotes rassies, V25/V28 — si les données ne tiennent pas, elles sortent, décision
  factuelle). Rôle : élargir le réseau + porter le « check » de Mert.
- **Règle de falsification pré-enregistrée (V8, mail Mert 17/07)** : *toute flèche forte soft → hard est,
  par convention préalable, un signal d'erreur de la méthode — jamais une découverte.* Gravée avant de
  regarder les données ; c'est elle qui remplace l'ancien étage « témoins ».
- **Un seul panier NEER fixe** sur tout l'univers retenu (poids BIS) : les séries de force sont identiques
  dans tous les panels — comparabilité A/B garantie par construction.
- **Exclues avec motif** : CNY (flottement géré), INR/KRW/BRL/SGD (gérées/contrôles — candidates de 2ᵉ
  vague), TRY (inflation extrême), DKK (arrimée à l'EUR → colinéarité). Preuve d'exhaustivité : UNIVERS_FX.md.
- **K = 16 à 19 nœuds** ; 15 à 18 paires vs USD à ingérer (l'USD n'a pas de paire propre : sa force vient
  du panier).
- **Deux panels d'estimation** (annoncés à Mert le 29/07) : **panel A « hard-only »** (étage 1) et
  **panel B « hard+soft »** (tout l'univers). Résultats principaux = panel A ; panel B = check directionnel,
  falsification, comparaison de stabilité A vs B.
- Nœuds latents assumés (observabilité partielle du cadre AAAI) : toutes les devises hors univers + facteurs
  communs non modélisés.

**Décision à valider : composition exacte (10+6).**

## 1.2 Cotations sources & triangulation

- Source : EODHD, tickers `XXXUSD.FOREX` (ou `USDXXX.FOREX` selon la convention de cote du marché —
  à normaliser à l'ingestion en **log-prix « unités d'USD par unité de XXX »**, croissant = XXX se renforce).
- **Pivot USD** : tout taux croisé i/j se déduit par triangulation `log P_ij = log P_iUSD − log P_jUSD`.
  On n'ingère donc QUE les 15 paires USD ; les croisés sont dérivés (cohérents par construction, pas
  d'arbitrage triangulaire résiduel dans les données dérivées).
- **Tâche préalable obligatoire** : sonde de disponibilité par paire (la doc EODHD ne garantit pas 2009 pour
  toutes) — un script qui interroge la première barre 1m disponible par paire et fixe `t0(paire)`.
  Le panel commun démarre à `max_i t0(i)`.

## 1.3 Granularité & historique (répond à V3)

- **Ingestion : barres 1 minute, 2009 → aujourd'hui** (seule granularité intraday profonde chez EODHD ;
  5m/1h natifs ne commencent qu'en oct. 2020).
- **Agrégation datalayer : barres 15 min et 1 h** (close de la dernière minute de l'intervalle).
  La 1m brute n'entre jamais directement dans l'estimation (bruit de microstructure) — elle est la
  matière première.
- Budgets d'échantillons résultants (FX ≈ 24h × 5j/sem) :
  | Barre | échantillons/an | 2009→2026 (~17 ans) |
  |---|---|---|
  | 1 h | ~6 200 | **~105 000** — l'échelle des expériences du papier |
  | 15 min | ~25 000 | ~425 000 |
  | 1 jour (référence) | ~260 | ~4 400 |
- **Contrainte d'horizon stationnaire (Mert 17/07 : « we should be on a stationary horizon more or less »
  — l'algorithme suppose l'équilibre)** : les ~105k échantillons 2009→2026 ne forment PAS une fenêtre
  d'estimation valide — 17 ans de FX traversent des régimes distincts (QE, choc SNB 2015, COVID, hausses
  2022). **L'unité d'estimation = fenêtre calendaire courte, la densité intraday fournissant les échantillons
  À L'INTÉRIEUR de la fenêtre.** C'est la résolution de la tension de Mert : horizon stationnaire (court)
  × bruit (besoin d'échantillons) → l'intraday concilie les deux. Le backfill complet sert aux analyses
  roulantes et à la stabilité temporelle, jamais à une estimation poolée sur 17 ans.
- **Fenêtre cible annoncée à Mert (email 29/07) : ≈ 4 mois en barres 1 min** — ~125k échantillons
  (24h × 5j ≈ 7 200 min/sem × ~17 sem), soit le budget des expériences de Santos et al. concentré dans une
  fenêtre calendaire courte, plausiblement mono-régime.
- **La grille d'horizons de Mert (V9) reste bidimensionnelle : fréquence de barre × longueur de fenêtre.**
  Départ = 1 min × 4 mois (l'engagement email) ; balayage autour (15m/1h × 6 mois-2 ans) pour la
  sensibilité ; le test synthétique (Phase 4) confirme le minimum d'échantillons requis.
- Coût backfill : ~52 requêtes/paire (120 j/requête) × 15 paires × 5 crédits ≈ **~4 000 crédits one-shot**
  (plafond quotidien 90k — marge confortable). Incrément quotidien ensuite : 15 requêtes/jour, négligeable.

## 1.4 Série de force par devise (type NEER)

- **Formule** : pour la devise i, force en log :
  `s_i(t) = Σ_{j≠i} w_ij · [log P_iUSD(t) − log P_jUSD(t)]`, poids `Σ_j w_ij = 1`.
- **Poids : BIS effective exchange rate weights** (paniers larges publiés par la BIS), restreints à notre
  univers K et renormalisés. Motifs : (i) c'est la référence « standard and well documented » annoncée à Mert
  (email 14/07, esprit NEER/BIS) ; (ii) les poids commerciaux sont **asymétriques** (w_ij ≠ w_ji), ce qui
  évite la singularité exacte de V24. Repli si récupération BIS trop lourde : poids égaux + pseudo-inverse
  (V24 assumé).
- **Stationnarisation** : l'entrée du VAR = **log-rendements de la force** `Δs_i(t)` par barre
  (les niveaux sont des quasi-marches aléatoires — non stationnaires, cf. fiche K&S).
- **Standardisation (V2, homogénéité nodewise)** : `Δs_i` divisé par sa volatilité glissante
  (fenêtre à fixer, départ : 60 barres) → variances ~égales entre nœuds et atténuation du clustering de vol
  (rapproche de l'hypothèse i.i.d. temporelle).
- **Centrage : réconciliation V26** (Mert 03/08 : Santos suppose un système zéro-mean) — les séries entrant
  dans le pipeline Santos naïf (étape 1) sont **démoyennées par fenêtre** ; la moyenne retirée est **conservée
  à part** car l'intercept du VAR encode l'informativité d (V18/fiche K&S §6), estimée en Phase 3.

## 1.5 Calendrier & alignement du panel

- Timestamps **UTC** partout (le FX est 24/5, pas de fuseau « naturel » ; l'UTC évite les pièges DST).
- Semaine FX : du dimanche ~21h UTC au vendredi ~21h UTC ; **barres de week-end exclues**.
- **Panel synchrone obligatoire** (le VAR exige des observations simultanées) : une barre n'est retenue que
  si TOUTES les paires ont une cote ; trous courts comblés par report de la dernière cote (limite : 3 barres) ;
  au-delà, la barre est supprimée pour tout le panel. Journal des suppressions conservé (auditables).
- Jours fériés globaux (1er janvier, Noël) et journées à liquidité morte : détectées par un seuil de barres
  manquantes sur la journée → journée exclue entière.

## 1.6 Stockage, ETL, partage (V12)

Réutilise l'infrastructure existante (swiss-wealth-etl + R2 + Supabase) :

- **R2 parquet** (brut volumineux) : barres 1m par paire, partition hive `pair/year`
  (~95 M lignes, ~2-3 Go compressés). Pattern identique au datalayer prix existant.
- **Supabase** (dérivés consultables — c'est ce qu'on partage à Mert) :
  - `fx_bars_1h` / `fx_bars_15m` : barres agrégées par paire (~1,6 M lignes en 1h) ;
  - `fx_strength` : séries de force par devise × barre (s, Δs, Δs standardisé) + version des poids ;
  - `fx_panel_log` : journal des barres/journées exclues (auditabilité).
- **ETL dans swiss-wealth-etl** (registre `jobs_registry` + `etl_log`, conventions existantes) :
  1. `fx_intraday_probe` (one-shot) : sonde t0 par paire ;
  2. `fx_intraday_backfill` (one-shot, reprise sur erreur, respect du RateLimiter crédits) ;
  3. `fx_intraday_daily` (cron) : incrément J-1 + recalcul des agrégats/force du jour ;
  4. `fx_strength_build` : construction/reconstruction des séries de force (re-jouable si les poids changent).
- **Accès Mert** : rôle Postgres **lecture seule** limité aux tables `fx_*` (pas de clé service, pas d'accès
  aux autres schémas). Livré avec un README d'accès (dictionnaire des colonnes, conventions).

## Critère de sortie de la Phase 1

1. Panel `Δs` standardisé, synchrone, en barres 1h et 15m, de `max t0` à aujourd'hui, dans Supabase ;
2. Sonde de disponibilité documentée (t0 par paire) ;
3. Cron incrémental en place et vert 3 jours de suite ;
4. Accès lecture seule transmis à Mert ;
5. Les hypothèses introduites ici (poids BIS, standardisation, règles de trous) **déposées dans la matrice
   V23** (`docs/HYPOTHESES.md`).
