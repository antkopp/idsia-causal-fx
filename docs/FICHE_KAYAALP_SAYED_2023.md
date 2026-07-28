# Fiche de lecture — Kayaalp & Sayed (2023/2026), « Causal Influences over Social Learning Networks » (arXiv 2307.09575v2, JMLR 27:1–54, 2026)

> Version lue : v2 (17 mai 2026, version JMLR, 54 p.). La numérotation des équations/sections ci-dessous suit cette version.
> Rôle dans notre pipeline : **étape 2 — influence causale C(m→k) + CausalRank** à partir de la matrice A estimée en amont (Santos et al. AAAI'24).

## 1. Modèle

### 1.1 NBSL (non-Bayesian social learning) — Sec. 3.3

Réseau de K agents, état du monde vrai `θ°` dans un ensemble fini `Θ = {θ_1,…,θ_H}` de H hypothèses. À chaque instant i, l'agent k reçoit un signal privé `ξ_{k,i}` de vraisemblance `L_k(ξ|θ)` (connue de l'agent pour tout θ).

**Adaptation (Bayes local), éq. (3) :**
```
ψ_{k,i}(θ) ∝ L_k(ξ_{k,i}|θ) · μ_{k,i−1}(θ)
```
**Combinaison (moyenne géométrique des croyances des voisins), éq. (4) :**
```
μ_{k,i}(θ) ∝ ∏_{ℓ∈N_k} [ψ_{ℓ,i}(θ)]^{a_ℓk}
```
- `μ_{k,i}` : croyance (pmf sur Θ) de l'agent k ; `ψ_{k,i}` : croyance intermédiaire (celle qui est partagée/publiée) ; `N_k` : voisinage de k.
- `A = [a_ℓk]` : matrice de combinaison ; **`a_ℓk` = poids que l'agent k accorde à l'agent ℓ** (attention à l'ordre des indices). `a_ℓk > 0 ⟺ ℓ ∈ N_k`.
- **A est left-stochastic : chaque COLONNE somme à 1** (`A^T 1_K = 1_K`). Graphe dirigé, A non symétrique en général. Vecteur de Perron v : `Av = v`, `1^T v = 1`, entrées > 0 (centralité réseau).

**Hypothèses sur signaux et graphe (Sec. 3.1–3.2) :** signaux i.i.d. **dans le temps**, vraisemblances invariantes dans le temps ; **la corrélation spatiale (entre agents) est autorisée** — le cadre tolère donc des confondeurs latents dans les observations ; graphe fortement connexe + au moins un self-loop ; croyances initiales strictement positives ; `d_k(θ) < ∞`.

**Informativité (éq. (1))** — LA quantité clé hors-A :
```
d_k(θ) ≜ D_KL( L_k(·|θ°) || L_k(·|θ) )   ≥ 0,  définie pour chaque θ ≠ θ°
```
C'est le « pouvoir discriminant » du signal privé de k contre l'hypothèse fausse θ. Vecteur `d(θ) = [d_1(θ),…,d_K(θ)]^T`.

### 1.2 Log-belief ratios λ — le pont linéaire (éq. (5)–(7))

```
λ_{k,i}(θ) ≜ log[ μ_{k,i}(θ°) / μ_{k,i}(θ) ]        (un vecteur λ_i(θ) ∈ R^K par hypothèse fausse θ)
x_{k,i}(θ) ≜ log[ L_k(ξ_{k,i}|θ°) / L_k(ξ_{k,i}|θ) ]  (log-likelihood ratio, E[x_{k,i}(θ)] = d_k(θ))
```
**Dynamique NBSL exacte (éq. (6)) — récursion linéaire :**
```
λ_i(θ) = A^T ( λ_{i−1}(θ) + x_i(θ) )
```
Théorème 1 : `(1/i)·λ_{k,i}(θ) → Σ_ℓ v_ℓ d_ℓ(θ)` p.s., identique pour tous les agents ; sous identifiabilité globale (au moins un agent avec `d_k(θ)>0` pour chaque θ faux), `μ_{k,i}(θ°) → 1` p.s. **Donc λ diverge linéairement : le processus NBSL n'est PAS stationnaire.**

### 1.3 ASL (adaptive social learning) — Sec. 3.4

**Adaptation modifiée (éq. (10)) :** `ψ_{k,i}(θ) ∝ L_k^β(ξ_{k,i}|θ) · μ_{k,i−1}^{1−δ}(θ)`, combinaison (4) inchangée. Paramètres : `0 < δ < 1` (facteur d'oubli : grand δ = priorité aux observations récentes) et `β > 0` (poids des observations propres vs croyances passées ; Bordignon et al. 2021 ne traitent que β=δ et β=1, la généralisation β>0 est directe).

**Dynamique ASL exacte (éq. (11)) — AR(1) vectoriel stable :**
```
λ_i(θ) = A^T ( (1−δ)·λ_{i−1}(θ) + β·x_i(θ) )
```
**Asymptotique (Théorème 2, éq. (12)) :** contrairement à NBSL, pas de convergence vers la vérité mais convergence **en distribution** vers une série absolument convergente `λ_i(θ) →dist β·Σ_{j≥1} (1−δ)^{j−1} (A^T)^j x_j(θ)` — fluctuations permanentes qui maintiennent l'adaptation. **Moyenne limite (Corollaire 1, éq. (13)) :**
```
lim E[λ_i(θ)] = (β/(1−δ)) · [ (I − (1−δ)A^T)^{−1} − I ] · d(θ)
```
NBSL est le cas limite `δ→0, β=1` des formules ASL.

## 2. Définition de l'influence causale (Sec. 4–5)

### 2.1 Intervention

Intervention **atomique et persistante à la Pearl** : `do(μ_{m,i} := μ_m)` — **on fige la CROYANCE de l'agent m** à un vecteur constant `μ_m` pour tous les instants (pas les signaux : l'agent m cesse d'utiliser ses signaux ET les messages de ses voisins ; il continue d'émettre `μ_m`). Effet mécanique : la matrice devient réductible (éq. (23)–(25), pour m=1) :
```
A = [ a_mm  r^T ]        Ã = [ 1  r^T ]
    [  *    R   ]   ⟹        [ 0  R   ]
```
- `r = [a_{m2},…,a_{mK}]^T` (K−1)×1 : **poids que les AUTRES accordent à m** ;
- `R` (K−1)×(K−1) : sous-matrice de A entre les autres agents (on supprime ligne et colonne m). `ρ(R) < 1` (garanti par la forte connexité).

**Mesure d'influence (éq. (15)), horizon = ∞, moyennage = espérance :**
```
C_{m→k} ≜ μ̄_{k,∞}(θ°) − μ̃_{k,∞}(θ°)
```
où `μ̄` (pré-intervention) et `μ̃` (post-intervention) sont des croyances asymptotiques **reconstruites à partir des log-belief ratios moyens** (éq. (16)–(17)) :
```
μ_{k,∞}(θ°) ≜ 1 / ( 1 + Σ_{θ≠θ°} exp{−λ_{k,∞}(θ)} ) ,   λ_{k,∞}(θ) ≜ lim_i E[λ_{k,i}(θ)]
```
(subtilité : espérance DANS l'exponentielle, pas `lim E[μ]` — c'est ce qui permet les formes closes). Un variant à horizon fini n existe (Remark 3, éq. (39)–(40)) et une généralisation à l'intervention sur un sous-réseau (Remark 2, éq. (38)).

### 2.2 Formes closes

**NBSL (éq. (35) + (37)).** Pré-intervention : `μ̄_{k,∞}(θ°) = 1` (Théorème 1), donc `C^NB_{m→k} = 1 − μ̃_{k,∞}(θ°) ∈ [0,1]`. Post-intervention :
```
λ̃_{−m,∞}(θ) = [ (I − R^T)^{−1} − I ] · d_{−m}(θ)  +  log[μ_m(θ°)/μ_m(θ)] · 1_{K−1}

C^NB_{m→k} = 1 − 1 / ( 1 + Σ_{θ≠θ°} (μ_m(θ)/μ_m(θ°)) · exp{ −[ ((I−R^T)^{−1} − I)·d_{−m}(θ) ]_k } )
```
Ingrédients exhaustifs : **(i)** entrées de A hors agent m (via R uniquement — r n'apparaît qu'implicitement par la contrainte left-stochastic, éq. (62) : `(I−R^T)^{−1} r = 1`), **(ii)** informativités KL `d_ℓ(θ)` des K−1 autres agents pour chaque θ faux, **(iii)** la dose d'intervention `μ_m`, **(iv)** H (via la somme sur Θ\{θ°}). Horizon infini.

**ASL (éq. (61) + (22)).** Post-intervention :
```
λ̃_{−m,∞}(θ) = log[μ_m(θ°)/μ_m(θ)] · (I − (1−δ)R^T)^{−1} r
             + (β/(1−δ)) · [ (I − (1−δ)R^T)^{−1} − I ] · d_{−m}(θ)
```
et `C^ASL_{m→k} = μ̄_{k,∞}(θ°) − μ̃_{k,∞}(θ°)` avec le pré-intervention donné par (13) injecté dans (16). Ingrédients exhaustifs : **R ET r explicitement** (contrairement à NBSL), `d_{−m}(θ)` (et `d(θ)` complet pour le terme pré-intervention), **δ**, **β**, dose `μ_m`, H. Interprétation en série (éq. (63)–(64)) : chaque hop mélange (R^T) et oublie ((1−δ)) ; le terme `log μ_m(θ°)/μ_m(θ)` agit comme une « pseudo-informativité » injectée par m.

**Version dose-indépendante (éq. (41)/(70)) :** on fixe `μ_m(θ) = 1/H` uniforme → `C̄_{m→k}`. L'annexe A montre que c'est équivalent à l'**average causal derivative effect** (dérivée de la dose-réponse). Pour H=2, la formule NBSL devient simplement :
```
C̄^NB_{m→k} = σ( −z_k )   avec  z = [ (I−R^T)^{−1} − I ]·d_{−m}(θ′),  σ = sigmoïde logistique
```

### 2.3 Point central (notre V7) : quels paramètres AUTRES que A ?

| Paramètre | Rôle | Comment le papier l'obtient | Options si on ne connaît que A |
|---|---|---|---|
| `d_k(θ)` (K×(H−1) divergences KL, « informativité ») | Pondère l'écrantage : les agents informés sont moins influençables et « blindent » leurs voisins | Soit supposé connu (simulations : gaussiennes, `d_k = ν_k²/2`) ; soit **estimé par GCL** (éq. (82)) à partir des croyances partagées | (a) **hypothèse d'homogénéité** `d_k(θ) = d̄` ∀k → un seul scalaire à choisir/sweeper (voir Sec. 6 : sous NBSL homogène, l'ORDRE des C̄ ne dépend pas de d̄>0) ; (b) estimation séparée depuis les résidus/intercept du VAR (E[bruit] = A^T d, cf. Sec. 6) ; (c) proxy économique (ex. variance idiosyncratique de la série de force). **Attention : d = 0 exactement rend C dégénérée (cf. vigilance #4)** |
| Dose `μ_m` | C est une courbe dose-réponse | Convention `μ_m = 1/H` (uniforme) pour un score unique | Prendre la convention uniforme — aucun coût |
| H (nb d'hypothèses) | Taille de Θ, diagonale de C = 1−1/H | Choix de modélisation (H=2 dans les 2 applications du papier) | Prendre **H=2** (« devise forte » vs « faible ») : d devient un scalaire par agent, formules sigmoïdes |
| δ (ASL) | Facteur d'oubli ; atténue l'influence des agents lointains | Paramètre de design supposé connu (GCL le prend en entrée) | Récupérable des sommes de lignes du A estimé si mapping ASL (cf. Sec. 6) ; sinon hyperparamètre à balayer |
| β (ASL) | Poids des observations propres ; multiplie d | Paramètre de design supposé connu | Non identifiable séparément de l'échelle de d (β·d apparaît en produit dans (61) à dose uniforme) → fixer β=1 et absorber dans d̄ |
| Horizon | ∞ par défaut ; variante finie éq. (40) | — | ∞ (ou n fini si on veut une « influence à n jours ») |

**Résumé V7 : à A donnée, il suffit de choisir H=2, dose uniforme, et UN vecteur d'informativité `d` (K scalaires, ou un seul scalaire d̄ sous homogénéité) — plus (δ, β) si variante ASL.**

## 3. CausalRank (Sec. 6, Algorithme 1)

- On construit la **matrice d'influence K×K** `[C]_mk = C̄_{m→k}` (dose uniforme), avec **diagonale posée à `1 − 1/H`** (éq. (75), valeur de l'auto-intervention).
- Toutes les entrées sont **strictement positives** (car ρ(R)<1 et d fini) ⟹ C est primitive ⟹ théorème de Perron : valeur propre dominante ρ réelle positive unique.
- **CausalRank = vecteur propre droit de Perron q de C : `C q = ρ q`** (normalisé q ≥ 0, Σq = 1 pour l'affichage). `q_m` = influence globale de m, pondérée récursivement : influencer des agents eux-mêmes influents compte plus (analogie PageRank explicite dans le papier).
- Alternative naïve comparée : AIR(m) = moyenne de la ligne m (`(1/(K−1))Σ_{k≠m} C_{m→k}`). CausalRank ≠ centralité de Perron v de A : q dépend aussi des informativités. Expériences : CausalRank robuste aux attaques d'inflation de followers (bots), contrairement au follower count et partiellement à AIR.
- Coût : O(K³) au total pour toute la matrice C + le vecteur propre (annexe F, avec mutualisation Schur des inverses `(I−(1−δ)R_m^T)^{−1}` entre agents ; naïf = O(K⁴)).

## 4. GCL (Graph Causality Learning) (Sec. 7, Algorithme 2)

Ce qu'il faut retenir pour la comparaison avec notre remplacement :

- **GCL n'estime PAS A à partir des séries temporelles.** A est formée par une **heuristique sur la matrice d'adjacence supposée connue** (règle d'averaging, éq. (81) : `Â_ℓk = 1/|N_k|` si ℓ∈N_k) — dans l'application Twitter, l'adjacence = qui suit qui.
- **GCL estime l'informativité D** (`[D]_kj ≈ d_k(θ_j)`) à partir des **croyances intermédiaires partagées ψ_{k,i}** (les « actions » publiques, ex. scores de sentiment de tweets), via la récursion linéaire des log-ratios (éq. (78)) `Λ_i = (1−δ)A^T Λ_{i−1} + β X_i` :
```
D̂ = (1/(βM)) · Σ_{i=1}^M ( Λ_i − (1−δ)·Â^T·Λ_{i−1} )      (éq. (82), une moyenne de résidus)
```
avec `θ°` estimé par `θ̂° = argmax_θ Σ_k ψ_{k,M}(θ)` (éq. (80)). δ et β sont des ENTRÉES (pas estimés).
- Hypothèses : modèle ASL/NBSL correct, adjacence connue, δ petit, erreur de Â petite. Garantie (Théorème 3) : `E|C_{m→k} − Ĉ_{m→k}| = O(1/√M)`.
- **Donc notre substitution (Santos et al. AAAI'24 pour A) remplace l'étape (81), pas l'étape (82)** — la structure du pipeline du papier (A d'une source, d des résidus de la récursion) est exactement conservée, et l'étape (82) nous donne gratuitement un estimateur de d une fois A branchée.

## 5. Hypothèses du papier — statut pour notre cas FX

| Hypothèse | Où | Statut chez nous | Parade |
|---|---|---|---|
| Signaux `ξ_{k,i}` i.i.d. dans le temps, vraisemblances time-invariantes | Sec. 3.1 (p. 5–6) | **À tester** : log-rendements FX ≈ peu autocorrélés mais hétéroscédastiques et à régimes | Standardisation par vol (déjà prévu), fenêtres roulantes ; le papier reconnaît le drift des distributions comme travail futur |
| Corrélation spatiale des signaux autorisée (confondeurs latents OK) | Sec. 3.1 + Fig. 13 | **Satisfaite** — crucial pour le FX (chocs USD communs) ; l'expérience Fig. 13 montre C invariante à la corrélation des données | — |
| Graphe fortement connexe + ≥1 self-loop | Sec. 3.2 | **À tester** sur la A estimée en amont | Seuiller/régulariser A pour garantir l'irréductibilité ; ajouter des self-loops ε |
| **A left-stochastic (colonnes somment à 1 ; a_ℓk = poids de k sur ℓ)** | Sec. 3.2, éq. (2) | **Violée a priori** : une A de régression n'est pas stochastique | Normaliser les colonnes de \|A\| (ou projeter sur le simplexe colonne par colonne) après transposition correcte — cf. Sec. 6 |
| État du monde θ° fixe/stationnaire | Sec. 3.1 | **Violée** en FX sur longue période | Utiliser la variante **ASL** (δ>0, conçue pour la non-stationnarité) et/ou fenêtres |
| Identifiabilité globale (∃k : d_k(θ)>0) | Def. 1 | Satisfaite dès que d̄>0 | Ne pas poser d=0 |
| H fini ; H=2 vs H>2 | Sec. 3.1 ; les 2 applications du papier utilisent H=2 | **Choisir H=2** (formules sigmoïdes, d scalaire par nœud) | H>2 possible mais multiplie les d(θ) à fournir sans bénéfice clair |
| `d_k(θ) < ∞`, croyances initiales > 0 | Sec. 3.1 | Techniques, satisfaites par construction | — |
| δ∈(0,1), β>0 connus | Sec. 3.4, Alg. 2 | Hyperparamètres | δ des sommes de lignes de la A estimée (cf. Sec. 6) ; β:=1 |

## 6. Pont avec l'estimation externe de A (Santos et al. AAAI'24)

**Mapping exact.** La dynamique observable est celle des log-belief ratios (nos « séries de force » jouent le rôle de λ ; pour H=2 c'est un scalaire par devise) :

- NBSL (éq. 6) : `λ_i = A^T λ_{i−1} + A^T x_i` → forme `y(n+1) = M·y(n) + w(n)` avec **`M = A^T`** (transposée !) et **bruit `w(n) = A^T x_n`** (le bruit est LUI AUSSI prémultiplié par A^T, donc corrélé entre nœuds même si les x le sont peu — et les x peuvent déjà être corrélés spatialement). **Mais NBSL est non-stationnaire (λ diverge linéairement, E[w] = A^T d ≠ 0)** : le fit d'un VAR stationnaire sur nos séries ne peut PAS correspondre à NBSL.
- **ASL (éq. 11) : `λ_i = (1−δ)A^T λ_{i−1} + β A^T x_i` → `M = (1−δ)A^T`, `w(n) = β A^T x_n`, stable (ρ(M)<1), stationnaire — c'est LE bon mapping pour un algorithme type y(n+1)=A_est·y(n)+bruit corrélé.**

Conséquences pratiques :
1. **`A_est ≈ (1−δ)·A_SLM^T`.** Donc `A_SLM = A_est^T/(1−δ)`. A_SLM left-stochastic (colonnes→1) ⟺ A_est a des **lignes** qui somment à `1−δ`. D'où : **δ̂ = 1 − (somme de ligne moyenne de A_est)**, et la dispersion des sommes de lignes est un diagnostic de mauvaise spécification. δ et A ne sont identifiables qu'via cette normalisation (seul le produit (1−δ)A^T est identifié par le VAR).
2. **Incohérences à signaler** : (i) transposition — la convention d'indices du papier (`a_ℓk` = poids de k sur ℓ, colonnes somment à 1) est l'inverse de la convention ligne-stochastique usuelle des VAR ; toute confusion inverse le sens des flèches d'influence ; (ii) les théorèmes AAAI'24 supposant A symétrique sont en tension avec une A_SLM généralement non symétrique (le papier insiste : graphe dirigé) ; (iii) le bruit du VAR n'est pas `x` mais `βA^T x` : sa moyenne `βA^T d ≠ 0` — l'intercept du VAR encode l'informativité (`d̂ = (A_est^T)^{−1}·ĉ/β` à un facteur près), piste alternative pour estimer d sans GCL.
3. **Cas limite « A seule » (point de départ Phase 3)** : NBSL, H=2, dose uniforme, **informativité homogène `d_k(θ′) = d̄` ∀k** :
```
C̄^NB_{m→k} = σ( −d̄ · w_{m,k} )   où  w_{m,k} = [ ((I−R_m^T)^{−1} − I)·1_{K−1} ]_k = Σ_{j≥1} [1^T R_m^j]_k
```
avec R_m = A privée de la ligne/colonne m. **Tout vient de A, plus UN scalaire d̄ > 0.** Comme z = d̄·w est globalement monotone en w, l'ordre des influences bipartites est indépendant de d̄ ; seul le contraste (et marginalement le classement CausalRank, via la non-linéarité sigmoïde) en dépend → balayer d̄ comme unique hyperparamètre. Garde-fou : **ne pas prendre d̄ = 0** (cf. vigilance #4) ; version ASL homogène = même chose avec `(I−(1−δ)R^T)` et le terme en r, au prix de δ.

## 7. Citations utiles pour la note à Mert

1. « Equation (35) is a general result which shows that C^NB_{m→k} is a function of (i) the combination weights (via R), and (ii) the individual informativeness of each agent (via d_{−m}(θ)). » — Sec. 5.1, p. 14.
2. « The graph causality learning (GCL) algorithm (Alg. 2) only requires a sequence of shared intermediate beliefs (actions) and the knowledge of adjacency matrix. » — Sec. 7, p. 25–26 (appuie l'idée que A vient d'une source externe au fit, donc que la substitution par une A estimée est dans l'esprit du papier).
3. « It is worth mentioning that in (65), the intervened log-belief ratio log μ_m(θ°)/μ_m(θ) behaves as "pseudo-informativeness". » — Sec. 5.2.1, p. 21 (utile pour discuter l'interprétation d'un choc de force d'une devise comme intervention).
4. (bonus, robustesse aux confondeurs) « Yet, our method maintains consistent results, which shows its robustness against non-causal factors. » — Sec. 8, p. 31, à propos de la Fig. 13.

## Points de vigilance nouveaux

1. **Le bruit du système linéaire n'est pas x mais A^T·x** (éq. (6)/(11)) : même des signaux spatialement indépendants produisent un bruit de VAR corrélé entre nœuds via A^T — cohérent avec l'hypothèse « bruit corrélé » d'AAAI'24, mais cela signifie que la covariance du bruit contient elle aussi de l'information sur A (redondance exploitable en validation).
2. **NBSL est non-stationnaire** (λ croît linéairement, Théorème 1) : seul le mapping **ASL** correspond à un VAR stationnaire fitté sur des séries standardisées. Corollaire : les formules NBSL restent utiles comme limite δ→0 des formules ASL, mais le régime des données est ASL.
3. **Identifiabilité (1−δ) vs A** : le VAR n'identifie que le produit (1−δ)A^T ; la séparation repose entièrement sur la contrainte left-stochastic de A. Les sommes de lignes de A_est donnent δ̂ ET un test de spécification gratuit (elles doivent être ~constantes entre lignes).
4. **Piège du cas d = 0 exactement** : sous NBSL, si tous les autres agents ont informativité nulle, `λ̃_k,∞ = log μ_m(θ°)/μ_m(θ)` pour TOUT k (éq. après (66)) ⟹ `C̄_{m→k} = 1 − 1/H` identique pour toutes les paires : **la matrice C devient constante et A disparaît complètement du résultat**. L'homogénéité doit donc être `d̄ > 0` strict, pas « on ignore d ». (En ASL, d=0 laisse subsister une dépendance en A via (66)/(68), mais dégénère aussi partiellement.)
5. **C n'est pas symétrisable ni signée** : `C_{m→k} ∈ (0,1)`, mesure une *magnitude* de contrôlabilité, pas un signe (pas de distinction influence « appréciative » vs « dépréciative »). Si la Phase 3 veut des influences signées, c'est hors du cadre du papier.
6. **Diagonale conventionnelle** `[C]_mm = 1 − 1/H` (auto-influence) : nécessaire à la primitivité de C pour Perron, mais c'est une convention — ne pas l'interpréter économiquement, et la garder telle quelle si on veut le CausalRank du papier.
7. **La dépendance en m passe par R_m** = A amputée de la ligne ET de la colonne m : le calcul de C est O(K³) au total seulement si on mutualise les inverses par compléments de Schur (annexe F) — trivial pour K≈10–30 devises, mais l'implémentation naïve par paire est O(K⁴).
8. **Ce que GCL prend en entrée, ce sont les croyances intermédiaires ψ (les « actions » publiées), pas μ** : dans notre analogie, la série de force observée doit être assimilée au log-ratio de ψ (ou de μ — les deux suivent la même récursion à un décalage près), il faudra fixer cette convention une fois pour toutes.
9. La définition de μ_{k,∞} passe l'espérance **à l'intérieur** de l'exponentielle (éq. (16)–(17)) : reproduire les C par simulation Monte-Carlo exige de moyenner les λ puis d'appliquer la sigmoïde, pas de moyenner les croyances (sinon écart de Jensen).
10. Le papier fournit une **variante à horizon fini** (éq. (40)) et une **intervention de sous-réseau** (éq. (38)) : directement utiles si on veut « influence du bloc dollar » ou « influence à 20 jours » sans travail théorique supplémentaire.
11. La v2 (JMLR 2026) diffère de la v1 2023 : expériences ajoutées (robustesse aux bots, CPU), rédaction remaniée — **citer la version JMLR 27(2026):1–54** dans la note à Mert ; les numéros d'équations ci-dessus suivent la v2.
