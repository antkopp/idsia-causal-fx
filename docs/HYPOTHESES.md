# Matrice des hypothèses (V23 — jalon bloquant 5.0)

Règle (PLAN §5.0) : TOUTES les hypothèses de TOUTES les méthodes du pipeline **tel que construit**,
déposées ici dès l'adoption de la méthode. Verdict tripartite obligatoire en Phase 5.0 :
**satisfaite** (preuve/test) / **violée mais justifiée** (argument écrit + test de sensibilité) /
**violée sans parade** (limite déclarée). Aucune note à Mert ni backtest Phase 6 sans revue complète.

Statut provisoire = état courant du raisonnement ; il ne vaut pas verdict. Le verdict final se prononce
en Phase 5.0 sur le système réellement construit.

## 1. Santos et al. AAAI'24 — identification de A (adoptée Phase 0, revue 04/08)

| Hypothèse | Où (papier) | Statut provisoire | Parade / test | Vigilance | Verdict 5.0 |
|---|---|---|---|---|---|
| Linéarité `y(n+1)=Ay(n)+x` + stabilité ρ(A)<1 | éq. 1, Ass. 1 | Assumée (linéarisation près de l'équilibre, log-rendements standardisés, fenêtres courtes) | Fenêtres stationnaires (V9) ; test synthétique avec non-linéarité contrôlée si besoin | V10 | — |
| A symétrique (théorèmes seulement) | Ass. 1 | **Violée** — notre objet est le cas dirigé | Expériences dirigées du papier + confirmation Mert 06/07 ; à assumer par écrit dans toute communication | V1 | — |
| A non négative | Problem Formulation | **Violée a priori** (influences FX négatives possibles) | A = magnitude ; signe porté par couche séparée (signes des coefficients VAR 1B) ; à valider avec Mert | V14 | — |
| Homogénéité nodewise des variances de bruit σ² | Ass. 2 | Réalisable | Standardisation de chaque série de force avant estimation | V2 | — |
| Distinguabilité σ² > E[x_i x_j] par paires | Ass. 3 | Plausible | Vérification empirique sur Σ̂x résiduelle après estimation | — | — |
| Bruit i.i.d. dans le temps | Problem Formulation | **Douteuse** (clustering de volatilité FX) | Standardisation par vol locale ; test synthétique avec bruit violant l'hypothèse pour chiffrer la casse | — | — |
| Bruit corrélé spatialement : autorisé sans hypothèse sur Σx | Problem Formulation | Satisfaite par design (c'est la force du papier pour les confondants FX) | — | V5 | — |
| Condition d'identifiabilité (12) : Osc(Off(Σx))/σ²_gap ≤ A⁺_min(1−ρ²)/(2ρ(ρ²+1)) | Théorème 2 | **Invérifiable directement** (Σx inobservable) | Substitut assumé : test synthétique Phase 4 avec bruit calqué FX, à notre échelle | — | — |
| Budget d'échantillons n ~ 10⁵–5·10⁵ | Fig. 3 | **Violée en journalier** (~8k points) | Historique intraday (V3, orientation actée) ; test synthétique = combien il en faut vraiment | V3 | — |
| FFNN entraîné sur régime représentatif du régime réel | Sec. expérimentale | **Violée si on réutilise leur réseau** (ER N=50 ≠ FX N~10-30) | Ré-entraîner NOTRE FFNN sur simulations calquées FX ; même code que le test Phase 4 | V4, V20, V21 | — |
| Moments empiriques : processus centré | implicite (éq. R̂_k) | **Fausse sur FX brut** | Centrage obligatoire des moments dans notre implémentation | V21 | — |
| Inversibilité / conditionnement de R̂₀ (features T, Granger) | éq. features T | **Menacée** (coupe transversale des forces quasi-colinéaire) | Poids BIS asymétriques ; surveillance du conditionnement ; ridge/pinv prêts | V24 | — |
| Poids 1B identifiés à échelle près (σ²_gap inobservable) | Théorème 1 | Structurelle | Convention de normalisation ; fixée naturellement par la contrainte left-stochastic du mapping SLM (donne δ̂) | V15, V16 | — |

## 2. Kayaalp & Sayed (JMLR 2026) — influence causale C(m→k) + CausalRank

*(À déposer — revue des blocs 5-8 en cours, 04/08. Socle : tableau §5 de FICHE_KAYAALP_SAYED_2023.md :
i.i.d. temporel des signaux, forte connexité + self-loop, A left-stochastic, θ° stationnaire → variante ASL,
identifiabilité globale d̄>0, H=2, δ/β.)*

## 3. Construction des données (Phase 1)

*(À déposer à l'adoption : formule NEER, poids BIS, standardisation, stationnarisation, fenêtres.)*

## 4. Estimation VAR / implémentation (Phase 2)

*(À déposer à l'adoption : régularisations, fenêtres, walk-forward.)*
