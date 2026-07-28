# Plan — Causal FX (7 phases)

Pipeline global : données FX → identification du réseau causal (support **1A** puis poids **1B**, Santos et al. AAAI'24)
→ matrice d'influence causale C(m→k) + CausalRank (Kayaalp & Sayed) → validation → livrables Mert → portefeuille backtesté.

Les deux étapes scientifiques successives (validées) :
1. **Identification de A** — l'algorithme AAAI'24 classe chaque paire connectée/déconnectée (= support de A) ;
   les poids s'obtiennent en sous-étape 1B via les estimateurs du même papier (Granger `R̂₁·R̂₀⁻¹` ou `R̂₁−R̂₃`
   rescalé par `σ²_gap`, Théorème 1), restreints au support identifié.
2. **Influence causale** — injection de Â dans les formules closes de Kayaalp & Sayed (l'étape 1 remplace la partie
   estimation du GCL) → C(m→k) interventionnelle + CausalRank. Reste à trancher : paramètres du SLM autres que A
   (informativité des signaux, β/δ de l'ASL).

---

## Phase 0 — Cadrage & matériel

- **0.1 Fiches de lecture opérationnelles** (`docs/`) : AAAI'24 (équations 1, 7-9, hypothèses 1-3, condition 12,
  features F/T/K, méthodologie FFNN) et Kayaalp & Sayed (C(m→k), NBSL vs ASL, β/δ, CausalRank, GCL).
  Chaque hypothèse avec statut : satisfaite / violée / à tester chez nous.
- **0.2 Audit du code des auteurs** : cloner le walkthrough, cartographier réutilisable vs à réécrire, dépendances.
  Licence absente → question à poser à Mert.
- **0.3 Squelette du repo** : `docs/`, `DECISIONS.md`, `references/`, puis `etl/`, `simulation/`, `estimation/`,
  `influence/`, `validation/`, `portfolio/`.
- **0.4 Registre des hypothèses assumées** (v1 dans `DECISIONS.md`).

**Critère de sortie** : livrables committés ; question « 1B : Granger restreint ou R̂₁−R̂₃ rescalé ? » posée
(tranchée en Phase 4 par le test synthétique).

## Phase 1 — Données & ETL (Supabase, accès partagé à Mert)

- Univers : devises flottantes majeures + **mineures pour le sanity check** (conseil Mert 17/07).
- Source EODHD ; construction de la **série de force par devise** (type NEER : agrégation des cotes bilatérales,
  choix des pondérations à documenter).
- Stationnarisation (log-niveaux vs log-rendements — l'algorithme suppose des données d'équilibre) ;
  standardisation des variances (hypothèse d'homogénéité nodewise).
- Fréquence et historique : tension budget d'échantillons (V3) — journalier ~7 800 points/30 ans vs intraday.
- ETL + partage d'accès Supabase à Mert (engagement email 14/07).

## Phase 2 — Identification de A (Santos et al.)

- Portage/adaptation du code walkthrough ; ré-entraînement FFNN sur régimes calqués sur le nôtre
  (N petit 10-30, β élevé — V4).
- **Grille d'horizons** (hyperparamètre critique, Mert 17/07) ; nœuds latents assumés (devises non modélisées,
  facteurs latents) ; bruit corrélé toléré sans hypothèse sur sa nature (confondants FX : force USD,
  différentiels de taux, risk sentiment).
- Sous-étape **1B poids** : estimateur choisi restreint au support.

## Phase 3 — Influence causale (Kayaalp & Sayed)

- Formalisation du pont série de force → croyances du SLM ; choix NBSL vs ASL (email 11/06 penche ASL).
- Traitement des paramètres non identifiés (hypothèse simplificatrice ou estimation).
- Calcul C(m→k) + **CausalRank** des devises ; évolution en fenêtres glissantes.

## Phase 4 — Validation & robustesse

- **Test de récupération synthétique end-to-end à notre échelle** (N petit, n ≈ 5-8k échantillons, β élevé) —
  critique vu V3 ; tranche aussi le choix de l'estimateur 1B.
- **Sanity check devises mineures** : une mineure qui « cause » l'USD = signal d'erreur (Mert 17/07).
- Robustesse fenêtre/fréquence ; stabilité temporelle (rolling) ; sensibilité aux confondants.

## Phase 5 — Livrables recherche

- Note de résultats pour Mert : graphe estimé, poids, CausalRank, sensibilité à l'horizon, sanity checks.
- Proposition de réunion (engagement email 17/07 : « looking forward to hearing results »).

## Phase 6 — Portefeuille & backtest (dernière phase)

Objectif : un **petit portefeuille de trading** exploitant la structure causale estimée, backtesté proprement.

- **Signaux candidats** (à trancher le moment venu, après les résultats de Phase 3-4) :
  a) *lead-lag* : les mouvements des devises meneuses (CausalRank élevé) prédisent les suiveuses → positions sur
  les suiveuses conditionnées aux mouvements récents des meneuses ;
  b) *régimes* : bascules de concentration d'influence (montée du CausalRank d'une devise) comme signal
  d'entrée/sortie ou de dé-risquage ;
  c) *paniers* : long/short meneurs vs suiveurs.
- **Discipline** : estimation **walk-forward** de A et C (fenêtre = l'horizon retenu), signaux sur info passée
  uniquement (PIT strict), coûts de transaction, pas de calibration in-sample des seuils.
- **Benchmarks obligatoires** : equal-weight, momentum FX, carry — le signal causal doit battre ce qu'un signal
  naïf obtient déjà.
- **À trancher** : instruments (spot/forwards vs ETF devises), taille, devise de base, moteur de backtest
  (réutilisation de quant-portfolio-base vs backtest FX standalone dans ce repo).
- **Garde-fou** : la matrice estimée porte beaucoup de variance — test de significativité vs benchmarks et
  sensibilité du P&L à l'horizon d'estimation avant toute conclusion.
