# Journal des décisions

Format : date — décision — motif/source. Les points de vigilance associés vivent dans [docs/VIGILANCE.md](docs/VIGILANCE.md).

## 2026-07 (échange email avec Mert, fil « Literature causality »)

- **14/07** — Nœuds du réseau = **devises** (pas les paires) ; entrée = **série de force par devise** type NEER
  (agrégation des cotes bilatérales). Motif : objectif = quelles devises mènent les autres ; entrée standard et
  documentée (esprit BIS NEER), la couche causale est notre ajout.
- **14/07** — Confondants (force USD, différentiels de taux, risk sentiment) traités comme **bruit exogène
  corrélé** du modèle — pas comme nœuds. Le papier AAAI'24 le permet sans hypothèse sur la nature des corrélations.
- **14/07** — ETL sur **Supabase**, accès partagé à Mert ; implémentation via le code open-source des auteurs.
- **17/07 (Mert)** — Univers = majeures **+ mineures en sanity check** ; **horizon = hyperparamètre critique**
  (grille systématique).
- **27/07** — Limitation principale assumée : **stationnarité + linéarité**. Piste complémentaire hors-équilibre
  parquée : « Causal tail dependence coefficient » (envoyé à Mert).

## 2026-07-28 (session de cadrage)

- Pipeline en deux étapes scientifiques validé : **1) identification de A** (support 1A + poids 1B, Santos et al.
  AAAI'24) → **2) injection de Â dans les formules closes de Kayaalp & Sayed** (C(m→k) + CausalRank). L'étape 1
  remplace la partie estimation du GCL.
- Plan en 7 phases acté (docs/PLAN.md), dont **Phase 6 ajoutée : petit portefeuille de trading + backtest** en
  dernière phase.
- Code des auteurs localisé : [seabrapt/brain_underlying_structure_identification](https://github.com/seabrapt/brain_underlying_structure_identification)
  (Python, walkthrough complet, pas de licence → V11).
- Registre de vigilance créé (V1-V13) avec déclencheur par phase : rien ne se ressort « de mémoire », tout est tracé.

## 2026-07-28 (exécution Phase 0)

- **Fiches de lecture produites** : `docs/FICHE_SANTOS_AAAI24.md` et `docs/FICHE_KAYAALP_SAYED_2023.md`
  (version lue : v2/JMLR 27:1-54, 2026 — à citer ainsi auprès de Mert). Audit code : `docs/AUDIT_CODE_SANTOS.md`.
- **V7 résolu sur le principe** : à A donnée, C(m→k) ne demande que H=2, dose uniforme, un scalaire
  d'informativité d̄>0 (homogénéité — l'ordre des influences n'en dépend pas) + (δ, β=1) en ASL.
  d estimable via l'intercept du VAR ou GCL éq. (82). Valeurs à trancher en Phase 3.
- **Variante retenue de fait : ASL** (V16) — NBSL est non-stationnaire, incompatible avec un fit VAR sur séries
  standardisées. NBSL reste la limite δ→0 pour les formules.
- **Code des auteurs : ré-implémentation from scratch, zéro vendoring** (V11 statué, V20-V21) — le code publié
  est un tutoriel buggé sur le cas dirigé, sans licence ; nos briques seront réécrites et testées (orientation
  des flèches vérifiée par test unitaire, V17).
- **Découverte structurante** (fiche K&S §4) : le pipeline du papier de Mert sépare déjà « A d'une source
  externe » (éq. 81, adjacence Twitter) et « d estimé des résidus » (éq. 82). Notre substitution
  (AAAI'24 → A) remplace l'éq. (81) et conserve l'éq. (82) : **argument de cohérence fort pour la note à Mert**.

- **Jalon bloquant ajouté (demande du 28/07)** : revue exhaustive finale de toutes les hypothèses de toutes
  les méthodes du pipeline tel que construit (PLAN §5.0, V23, futur `docs/HYPOTHESES.md`). Verdict tripartite
  par hypothèse ; alimentée en continu à chaque méthode adoptée ; conditionne la note à Mert ET la Phase 6.

- **Données : cap sur l'intraday FX (28/07)** — parade principale à V3 (déficit d'échantillons en journalier).
  Chantier ETL de la Phase 1 : recharger l'historique intraday FX (EODHD) ; granularité à trancher en Phase 1,
  dimensionnée par le test synthétique (besoin réel en échantillons).
- **Signe des influences (28/07)** — architecture actée : **A = magnitude** (valeur absolue de l'influence,
  compatible avec l'hypothèse A ≥ 0 du cadre AAAI'24) + **couche de signe séparée**. Candidat naturel :
  `S = signe(Â_VAR)` issue des coefficients Granger/VAR de la sous-étape 1B (gratuite, même estimation).
  À valider avec Mert ; fournit le sens des trades en Phase 6 (V19).

- **Relecture du mail de Mert 17/07 → deux amendements Phase 1 (28/07)** :
  (i) univers en **3 étages** — majeures / mineures flottantes / **contrôles négatifs purs** (2-3 exotiques
  « random », candidates CLP/COP/PHP, hors poids NEER et hors résultats, rôle exclusif de falsification) ;
  (ii) **l'unité d'estimation est une fenêtre calendaire courte (1-3 ans, régime stationnaire)** — jamais
  2009→2026 poolé ; l'intraday sert à densifier l'intérieur de la fenêtre, ce qui concilie les deux
  contraintes de Mert (stationnarité vs bruit).

## 2026-07-29 → 2026-08-03 (échange design complet avec Mert)

**Emails Antoine 29/07** (design synthétisé et envoyé — engagements pris) :
- Two-stage : support via classifieur + **poids via l'estimateur de moments rescalé** (masquage de l'estimation
  pleine matrice OU moindres carrés sur les parents identifiés) ; C_m→k via GCL, **informativité estimée
  depuis la récursion des log-croyances**.
- ASL (stationnaire, aligné Santos), **H=2** (« currency = strong or weak »).
- Données : **fenêtres ≈ 4 mois en barres 1 min** (≈ budget d'échantillons de Santos et al.) ; indices de
  force maison, méthodologie BIS NEER ; **deux panels** — hard+soft (check) et hard-only.
- Signes/échelles de temps : **M signée, A = |M|, M = estimateur de différence de moments (R̂₁−R̂₃), pas
  Granger** ; matrice des signes = signe de la **réponse impulsionnelle cumulée de M sur le holding period** ;
  le signe dépend du hp. + 3 papiers non-linéaires partagés (Liang-Kleeman, Sugihara CCM, review Runge) — parqués.

**Réponse Mert 03/08** :
- **Directive « naive first »** : Santos suppose un système **zéro-mean** (pas d'état → pas de confondant chez
  eux) ; les extensions (confondants, non-linéaire) améliorent mais ne sauvent pas une méthode qui ne marche
  pas — **d'abord l'apprentissage naïf de A + validation par simulations** (les simulations sont l'arbitre) ;
  **hp/signes incorporés APRÈS l'analyse initiale** (le mécanisme de signes n'est ni validé ni rejeté — différé).
- **A dirigée confirmée** : symétrie requise pour les théorèmes seulement, pas pour la stratégie (V1 statué).
- Papier heavy-tailed : statique (pas séries temporelles), §5.2 intéressant ; deux axes laissés **ouverts** :
  journalier/long terme vs causal court terme ; transitoire-depuis-équilibre vs transitoire-depuis-transitoire.

**Conséquences plan** : la Phase 2 commence par le pipeline naïf zéro-mean strict ; le test synthétique
(Phase 4) remonte en priorité absolue (c'est l'arbitre annoncé à Mert) ; couche de signes/hp = après.

## 2026-08-05 (revue 1.1 — univers)

- **Étage 1 validé** : G10 complet, prouvé fermé (UNIVERS_FX.md, recensement BIS 39 devises) ; 10 devises
  jugées suffisantes ET maximales (T/N ~100× plus favorable que le papier ; « hard flottant » structurellement
  épuisé par le G10). Flags de régime : CHF 2011-15 inéligible / 2015-22 flagué ; événements JPY/GBP.
- **Étage 2 validé** : MXN, ZAR, PLN, HUF, CZK, ILS — bande 0,3-1,5 % prouvée fermée ; CZK et ILS gardées
  avec flags (CZK : plancher 2013-17 inéligible + défense 2022 ; ILS : ≤2022 flagué + oct. 2023-24 inéligible).
- **Filtre de liquidité requalifié** (V28) : filtre de validité de mesure dépendant de la fréquence, pas
  d'importance économique ; seuil réel = taux de cotes rassies mesuré par la sonde.
- **Simplification : pas d'étage « témoins » séparé** (proposition utilisateur) — CLP, COP, PHP = nœuds soft
  ordinaires sous condition de sonde ; panier NEER unique et fixe sur tout l'univers (comparabilité A/B par
  construction) ; la falsification survit comme **règle pré-enregistrée** (V8) : toute flèche forte
  soft → hard = signal d'erreur, jamais une découverte.
- Cas-limites documentés : CLP (admissible, intégrée soft), RUB (exclu même en historique, cohérence du
  panel), ISK (seule flottante exclue par pure liquidité), SGD (hard au sens qualité mais NEER administré).

## À trancher plus tard (parqué, ne pas perdre)

- Estimateur des poids 1B : Granger restreint vs R̂₁−R̂₃ rescalé → tranché par le test synthétique (Phase 4).
- NBSL vs ASL et paramètres du SLM autres que A (V7) → Phase 3.
- Fréquence des données (journalier vs intraday) → Phase 1, dépend de V3.
- Instruments et moteur de backtest de la Phase 6 (spot/forwards vs ETF ; quant-portfolio-base vs standalone).
