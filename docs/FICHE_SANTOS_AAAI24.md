# Fiche de lecture — Santos, Rente, Seabra & Moura (AAAI'24)

**« Learning the Causal Structure of Networked Dynamical Systems under Latent Nodes and Structured Noise »**
[arXiv 2312.05974v3](https://arxiv.org/abs/2312.05974) · [AAAI proceedings](https://ojs.aaai.org/index.php/AAAI/article/view/29406) · code : [seabrapt/brain_underlying_structure_identification](https://github.com/seabrapt/brain_underlying_structure_identification)
Rôle dans notre pipeline : **étape 1 — identification de A** (support en 1A, poids en 1B).

---

## 1. Modèle (éq. 1)

```
y(n+1) = A·y(n) + x(n+1)
```

- `y(n) ∈ R^N` : état des N nœuds (chez nous : log-rendements standardisés des séries de force par devise).
- `x(n)` : bruit d'entrée, moyenne nulle, **i.i.d. dans le temps** mais **corrélé spatialement** — covariance
  `Σx` quelconque (aucune hypothèse sur la nature des corrélations, aucune connaissance de Σx requise).
- `A ∈ R+^{N×N}` : matrice d'interaction **non négative**, son support = le graphe causal.
  Normalisation : `A = ρ·Ā` avec `Ā` stochastique et `ρ < 1` (stabilité : rayon spectral ρ(A) < 1).
- **Observabilité partielle** : on n'observe qu'un sous-ensemble S de nœuds ; l'objet d'inférence est le
  support de la sous-matrice `A_S` (liens entre nœuds observés). Les nœuds latents biaisent l'estimation
  mais le papier caractérise ce biais.
- Justification du linéaire : discrétisation d'EDS linéaires ; ou système non linéaire **près d'un équilibre**
  avec bruit assez petit (Hartman-Grobman) ; extensions possibles via Koopman / changement de variables / Box-Cox.

## 2. Ce que la méthode produit : le SUPPORT (1A)

**Définition 1 (structural consistency)** : un estimateur matriciel `F_S^(n)` est structurellement consistant
s'il existe un seuil τ tel que `F_ij > τ ⟺ j → i` (avec probabilité → 1). C'est un problème de
**classification binaire par paire** — connecté / déconnecté — pas une estimation de valeurs.

Mécanisme :
- **Covariances empiriques de lag k** : `R̂_k(n) = (1/n) Σ_ℓ y(ℓ+k)·y(ℓ)^T`.
- **Features F** (éq. 11) : pour chaque paire ij, le vecteur `([R̂_D]_ij, …, [R̂_M]_ij)` avec D ≤ 1, M ≥ 3.
- **Features T** (nouvelles) : entrées des **inverses** des moments observés `([R̂_0]_S^{-1}, …, [R̂_M]_S^{-1})_ij`
  — atténuent empiriquement drift et oscillation dus au bruit corrélé.
- **Features K = (F, T)** combinées, normalisées par standard-scaler → **FFNN** entraîné sur données synthétiques
  (vérité terrain connue) classe chaque paire. Le théorème 2 garantit la séparabilité linéaire des features
  centrées sous la condition (12).

## 3. Le chemin vers les POIDS (1B) — pas le message principal du papier, mais dedans

- **Granger** : en observabilité **totale**, `R₁·R₀^{-1} = A` exactement (limite n→∞) — poids compris.
  En observabilité **partielle**, biais en forme close (éq. 5) : `E_S = A_SS'·([R₀^{-1}]_S')^{-1}·[A²]_S'S`
  (S' = nœuds latents). Performance dégradée sous bruit coloré.
- **Théorème 1** : `(1/σ²_gap)·(R̂₁(n) − R̂₃(n)) → A + E` avec
  `E = (1/σ²_gap)·[βρ·11^T + (I−A²)·Σᵢ A^{i+1}·Σ̄·Aⁱ]` où
  `σ²_gap = σ² − max_{i≠j} E[x_i x_j]`, `β` = moyenne des hors-diagonales de Σx, `Σ̄` = leur variabilité.
  Lecture : **β ne fait que translater** (drift uniforme, inoffensif pour le support, biais constant pour les
  poids), c'est **l'oscillation de Σ̄ qui déforme** les valeurs.
- **Notre 1B** : appliquer l'un de ces estimateurs, **restreint au support identifié en 1A** (zéros imposés hors
  support). Choix Granger vs R̂₁−R̂₃ tranché par le test synthétique (Phase 4). Attention : `σ²_gap` n'est pas
  observable directement → les poids sont identifiés **à une échelle près** ; la normalisation `A = ρ·Ā` peut
  servir de convention.

## 4. Condition d'identifiabilité (Théorème 2)

```
Osc(Off(Σx)) / σ²_gap  ≤  A⁺_min·(1−ρ²) / (2ρ(ρ²+1))
```

- `Osc` = max − min des hors-diagonales de Σx ; `A⁺_min` = plus petite entrée non nulle de A.
- Sens : la **variabilité** des corrélations de bruit doit être petite devant le plus petit lien réel.
  Si les corrélations sont fortes mais **homogènes**, tout va bien (le drift est plat).
- **Remarque 1 (interventions exogènes)** : ajouter un bruit d'intervention i.i.d. de variance σ²_ξ assez
  grande restaure l'identifiabilité quelle que soit Σx (le dénominateur devient σ²_gap + σ²_ξ). Inapplicable
  en FX observationnel — mais utile en simulation, et écho direct aux interventions do() de Kayaalp & Sayed.

## 5. Méthodologie expérimentale du papier (à répliquer/adapter)

- Génération : graphe Erdős–Rényi (non dirigé) ou Binomial (dirigé), poids par **règle laplacienne**
  `A_ij = α·G_ij/d_max(G)`, `A_ii = ρ − Σ_k A_ik`, avec 0 < α ≤ ρ < 1.
- Entraînement FFNN sur **une** réalisation N=50, p=0.5, β ∈ {0,5,…,50} ; généralise à d'autres régimes.
- Benchmarks : Granger, R̂₁ seul, R̂₁−R̂₃ (NIG), precision matrix / graphical lasso, Machado et al. 2023 —
  clustering par mélange gaussien pour ces baselines.
- Régimes testés : N = 80–160, S = 20–70 observés, β = 100–240, p = 0.3–0.7, jusqu'au connectome cérébral réel.
- **Échantillons : 10⁵ à 5·10⁵** — voir V3.

## 6. Hypothèses — statut pour notre cas FX

| Hypothèse | Où | Statut chez nous | Parade |
|---|---|---|---|
| Linéarité + stabilité ρ(A)<1 | éq. 1, Ass. 1 | À tester (assumée près de l'équilibre — email 27/07) | Log-rendements ; fenêtres stationnaires ; V10 |
| **A symétrique** (théorèmes) | Ass. 1 | **Violée** (on veut du dirigé) | Expériences dirigées du papier + aveu explicite de Mert (06/07) ; à écrire noir sur blanc (V1) |
| A non négative | Problem Formulation | **À tester** — des influences FX peuvent être négatives | Point à discuter avec Mert ; transformation éventuelle |
| Homogénéité nodewise σ² | Ass. 2 | Réalisable | Standardiser chaque série de force (V2) |
| Distinguabilité par paires σ² > E[x_i x_j] | Ass. 3 | Plausible | Vérifier empiriquement sur Σ̂x résiduelle |
| Bruit i.i.d. dans le temps | Problem Formulation | **Douteuse** (clustering de volatilité) | Standardisation par vol réalisée ; à tester en simulation |
| Condition (12) sur Osc(Off(Σx)) | Théorème 2 | Inconnue | Non vérifiable directement (Σx inconnue) ; le test synthétique avec bruit calqué FX en tient lieu |
| n ~ 10⁵ échantillons | Fig. 3 | **Violée en journalier** (~8k) | Intraday et/ou démonstration synthétique à notre échelle (V3) |

## 7. Points saillants pour la suite

1. La **force** du papier pour nous : bruit corrélé quelconque + observabilité partielle = exactement le régime
   FX (confondants force USD / taux / sentiment ; devises non modélisées et facteurs latents). (V5)
2. Le pipeline est **simulation-first** : le FFNN s'entraîne sur du synthétique avec vérité terrain — notre
   test de récupération (Phase 4) est donc structurellement le même code que l'entraînement (Phase 2).
3. La nouvelle hypothèse repérée à la rédaction de cette fiche : **A non négative** — les influences entre
   devises peuvent être de signe négatif. → ajouté au registre (V14).
4. Papier compagnon utile si besoin : Machado et al. 2023 (features d'origine, bruit diagonal) ;
   Matta, Santos & Sayed 2020 (analyse du biais Granger sous observabilité partielle) ;
   Santos et al. ICASSP 2024 (expériences supplémentaires non dirigées).
