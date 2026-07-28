# Audit du code open-source — Santos, Rente, Seabra, Moura (AAAI'24)

**Papier** : « Learning the Causal Structure of Networked Dynamical Systems under Latent Nodes and Structured Noise »
**Repo** : https://github.com/seabrapt/brain_underlying_structure_identification
**Clone audité** : HEAD `8fdd983` (2024-01-08), branche `main`, audit réalisé le 2026-07-28 (clone dans le scratchpad, non vendorisé — V11).

---

## 1. Inventaire

### Arborescence à HEAD

```
brain_underlying_structure_identification/
├── .gitignore                      (28 o)     — ignore __pycache__, logs/, *.pkl
├── README.md                       (438 o)    — 10 lignes, description du walkthrough
└── walkthrough/
    ├── helper_functions.py         (17 870 o) — Python ; TOUTE la logique (génération, features, estimateurs, architectures)
    ├── requirements.txt            (6 112 o)  — pip freeze encodé UTF-16 (Windows)
    ├── tutorial.ipynb              (16 086 o) — notebook Jupyter ; démo bout-en-bout + comparaison d'estimateurs
    ├── training.ipynb              (14 016 o) — notebook Jupyter ; entraînement du FFNN et du CNN baseline
    └── weights/
        ├── cnn.h5                  (1 221 352 o) — poids Keras du CNN baseline [Machado et al.], entrée 200 features
        ├── directed.h5             (1 245 944 o) — poids Keras du FFNN « our method », entrée 200 features
        └── undirected.h5           (1 245 944 o) — poids Keras du FFNN, entrée 200 features (md5 différent de directed.h5)
```

Total : **9 fichiers**, ~3,7 Mo (dont ~3,6 Mo de poids). Aucun fichier de données brutes (ni synthétiques ni cérébrales). Pas de tests, pas de CI, pas de packaging.

### Historique git

31 commits, du **2023-04-07** (« Initial commit ») au **2024-01-08** (« rentilt fix »). Auteurs : **Seabra** (Rui Seabra), **rentilt / Diogo Rente**. Aucun commit de Santos ou Moura.

Phases :
- **avr.–juin 2023** : gros repo de recherche exploratoire — dossiers `Linear/` (ColoredNoise, ShiftData, Feature_Inversion), `NonLinear/`, dizaines de modèles Keras sauvegardés, scripts `run_vm_probs_new_features_directed.py`, `estimators.py`, `functions.py`, `features_inversion.py`, notebooks divers.
- **2023-08-15** (`183bde4`) : ajout du dossier `walkthrough/` (« simple walkthrough / tutorial »).
- **2023-12-10** (`6e73ca1` « Changes ») : **purge massive** — tout `Linear/`, `NonLinear/` et les modèles sont supprimés ; ne survit que le walkthrough.
- **déc. 2023–jan. 2024** : polissage (barres d'erreur, `load_weights`, fix `.pkl`).

Conséquence : le code publié à HEAD est une **version pédagogique condensée**, pas le code des expériences du papier. Les anciens fichiers restent récupérables dans l'historique git (certains chemins illisibles sous Windows : « Filename too long »).

---

## 2. Correspondance papier ↔ code

| Brique du papier | Où | Statut |
|---|---|---|
| Graphe Erdős–Rényi dirigé/non dirigé | `helper_functions.py::get_adjacency` | Complet (mais voir bug directionnel §6) |
| Matrice A (règle laplacienne) | `helper_functions.py::get_A` | Complet |
| Bruit structuré (Σx, β) | `helper_functions.py::generate_noise` | Complet |
| Séries y(n+1)=A·y(n)+x(n+1) | `helper_functions.py::generate_timeseries` | Complet |
| Covariances empiriques R̂_k | inline dans `get_inverted_features` et les `get_*_preds` | Complet |
| Features F (lags) | `extract_cross_correlation_unscaled_features` | Complet (sert au CNN baseline) |
| Features T (inverses) + K combinées | `get_inverted_features` | Complet |
| Standard-scaler | `StandardScaler` dans les notebooks | Complet |
| FFNN + classification | `get_model_architecture` + `training.ipynb` + `cluster_predictions` | Complet |
| Estimateurs de référence | `get_granger_preds`, `get_r1_minus_r3_preds`, `get_r1_preds`, `get_r0_inv_preds` | Complet (mais precision matrix ≠ graphical lasso, §6) |
| Observabilité partielle | slicing `x[:, ss:se]` dans les notebooks | Rudimentaire (sous-ensemble contigu) |
| Cas dirigé de bout en bout | partiellement (`undirected=False`) | **Incomplet/buggé** (§6) |
| Données cérébrales (connectome) | — | **Absent** |

### Détail par brique

**Graphe et matrice A** — `get_adjacency(sz, p, undirected)` : matrice aléatoire uniforme, arête si `rand <= p` (docstring inversé — coquille), diagonale à 0 ; si non dirigé, `triu` puis symétrisation. `get_A(adj, c, rho)` implémente la règle laplacienne du papier :
```python
Dmax = max(degrés) ; L = D - adj
A = rho * (I - (c/Dmax) * L)
```
soit A_ij = rho·c·G_ij/Dmax hors diagonale et A_ii = rho·(1 − c·d_i/Dmax) ; rayon spectral < 1 garanti par 0 < c, rho < 1. Valeurs des notebooks : `c=0.9, rho=0.75`.

**Bruit structuré** — `generate_noise(N, n_samples, alpha, beta, perturbation=False)` :
```python
z[i] = alpha * x1 + x2 @ (beta/sqrt(N) * 1·1ᵀ)
```
⇒ Σ_x = α²·I + β²·1·1ᵀ : un **facteur commun scalaire** parfaitement corrélé entre tous les nœuds, d'intensité β (jusqu'à 100–200 dans le tutoriel), plus une option `perturbation` (jitter ±5 % de β hors diagonale, non utilisée dans les notebooks). La synthèse des séries est la récursion VAR(1) littérale du papier, x(0)=0.

**Covariances empiriques** — jamais isolées dans une fonction dédiée ; chaque estimateur recalcule ses R̂_k par produits de fenêtres décalées. **Aucun retrait de moyenne nulle part** (séries synthétiques centrées par construction).

**Features** — deux familles :
1. `extract_cross_correlation_unscaled_features` : features **F** — cross-corrélation par paire via `scipy.signal.correlate`, `n_features` lags centrés (±100 pour 200 features). Entrée du **CNN baseline [Machado et al.]**.
2. `get_inverted_features(timeseries, n=200, undirected=True)` : features **K** — pour `offset` 0..n//4−1 (**lags 0..49** pour 200 features), 4 colonnes par lag : `R̂_offset[paire]`, `inv(R̂_offset)[paire]`, `R̂_offsetᵀ[paire]`, `inv(R̂_offsetᵀ)[paire]` (fallback `lstsq` si singulier). Extraction par `get_upper_tr` (qui lit en réalité le triangle **inférieur** `M[j,i], j>i`) ou `get_off_diagonal` (dirigé). Puis `StandardScaler` colonne par colonne.

**FFNN** — Keras/TensorFlow, `get_model_architecture(n_features)` :
```
Dense 512 relu → Dense 256 relu → Dropout 0.2 → Dense 200 tanh → Dense 100 tanh → Dense 1 linear
optimizer=adam, loss=mse
```
~307k paramètres. **Régression MSE sur labels {1, 2}** (1=déconnecté, 2=connecté), classification finale par clustering k-means/GMM 2 groupes des prédictions (`cluster_predictions`). Entraînement (`training.ipynb`) : 10 datasets concaténés — β ∈ linspace(0,50,10), tsize = (β+1)·20 000 (20k → 1,02M échantillons), graphes 30–80 nœuds, p=0.5 — EarlyStopping(patience=7), validation_split=0.1, 10 runs dont on garde le **meilleur sur le test set** (leakage). Les features d'entraînement du notebook font **100** dimensions alors que les poids livrés en attendent **200** (§6).

**Estimateurs de référence** (candidats sous-étape 1B — poids de A) :

- **Granger** `get_granger_preds` : exactement **R̂₁·R̂₀⁻¹** —
  ```python
  R0 = z·zᵀ/tsize
  R1 = z[:,2:]·z[:,1:-1]ᵀ/(tsize-1)
  pred = R1 @ inv(R0)     # lstsq(R0, I) si singulière
  ```
  Estimateur moindres carrés du VAR(1) restreint aux nœuds observés ; en observabilité totale et bruit diagonal, **estime les poids de A** de façon consistante. Aucune régularisation, aucun centrage.
- **R̂₁−R̂₃ (NIG)** `get_r1_minus_r3_preds` : différence brute des moments lag 1 et lag 3, sans facteur correctif ni inversion. Proportionnel à A au premier ordre seulement : bon pour la **structure**, à recalibrer pour l'**amplitude** en 1B.
- **R̂₁** seul ; **« precision matrix »** = `inv(R̂₀)` brut (pas un graphical lasso, malgré `gglasso`/`regain` dans le freeze — utilisés dans le code supprimé).

**Observabilité partielle** — simple slicing des **premiers nœuds contigus** (`x[:, ss:se]`). Un pickle nommé `comparison_data_random_idx.pkl` suggère une variante à indices aléatoires, absente du code. L'entraînement se fait **sans** observabilité partielle ; seuls test et tutoriel masquent des nœuds.

---

## 3. Dépendances et exécutabilité

- **requirements.txt** : pip freeze de 161 paquets **encodé UTF-16-LE** — `pip install -r` échoue tel quel sous Linux. Réellement utilisés : `numpy==1.23.5`, `scipy==1.10.0`, `scikit-learn==1.3.0`, `keras==2.10.0`/`tensorflow==2.10.1` (API Sequential + `.h5`), `matplotlib==3.6.3`, `tqdm`. Le reste (torch, librosa, opencv, dgl, geopandas, pyEDFlib, gglasso, regain…) = bruit de l'environnement de recherche.
- **Python** : non spécifié ; `pywin32==305` + TF 2.10 ⇒ vraisemblablement **Python 3.10 sous Windows**.
- **Exécutabilité** : tutoriel autonome (tout synthétique, poids fournis). Effort ~**1 heure** (venv 3.10, ~8 paquets à la main). Pièges : TF 2.10 incompatible Python ≥3.11 ; la cellule « Multiple Runs » génère 3×500 000 échantillons sur 120 nœuds (minutes/Go) ; la cellule de plot finale lit un pickle absent et plantera ; `training.ipynb` produit un modèle 100 features **incompatible** avec les poids fournis (200).

---

## 4. Réutilisable vs à réécrire pour notre cas FX

Rappel : N = 10–30 devises, ~5 000–8 000 échantillons journaliers (vs 10⁵–10⁶), bruit très corrélé, graphes **dirigés**, ré-entraînement FFNN sur nos régimes.

| Brique | Verdict | Justification |
|---|---|---|
| `get_adjacency`/`get_A` | **Adaptable** | 40 lignes triviales ; corriger le cas dirigé ; calquer la distribution des poids sur des A estimés FX plutôt que la règle laplacienne pure. |
| `generate_noise` | **Adaptable** | α²I + β²·11ᵀ = bonne maquette du confondant USD ; étendre à plusieurs facteurs (Σ = α²I + B·Bᵀ) et à des covariances estimées FX. |
| `generate_timeseries` | **Réutilisable tel quel** | 8 lignes de récursion VAR(1). |
| `get_inverted_features` (features K) | **Adaptable — cœur de la méthode** | Logique transposable directement. À adapter : **centrage** (nos données ne sont pas de moyenne nulle), nombre de lags (50 lags sur 5 000 points ≠ sur 500 000), **inversion régularisée** (ridge/pinv à N=30, T=5 000). |
| Features F / CNN baseline | **Réutilisable** si on reproduit la baseline ; sinon inutile. |
| FFNN + recette d'entraînement | **À réécrire (en s'inspirant)** | Architecture 512-256-200-100-1 = bon départ ; refaire en PyTorch/TF récent ; remplacer labels {1,2}+MSE+clustering par sigmoïde+BCE ; grille d'entraînement recalquée sur nos régimes ; bannir la sélection best-of-10 sur le test (leakage). |
| `cluster_predictions` | **Réutilisable** | Astuce simple pour seuiller sans seuil arbitraire ; garder aussi l'option seuil calibré. |
| Estimateurs Granger / R̂₁−R̂₃ / R̂₁ / inv(R̂₀) | **Adaptable — candidats 1B** | 15 lignes chacun ; réécrire avec centrage, régularisation, et garder la **matrice complète** (pour les poids dirigés Â = R̂₁·R̂₀⁻¹ entière). Re-vérifier l'orientation (piège `get_upper_tr`). |
| Observabilité partielle | **À réécrire** | Slicing contigu ; nous voudrons des sous-ensembles arbitraires et une réflexion sur qui sont les latents. |
| Métriques (`get_acc_idgap`, sensitivity/specificity) | **Réutilisable** | L'« identifiability gap » diagnostique la séparabilité. |
| Poids .h5 fournis | **Inutilisables** | Régimes trop éloignés du nôtre ; provenance/licence non clarifiées ; on ré-entraîne de toute façon. |

**En résumé : rien à vendoriser, tout à ré-implémenter** — ~700 lignes utiles, très lisibles ; une ré-implémentation propre (centrage, régularisation, PyTorch, seeds, tests) est plus rapide et plus sûre que d'adapter.

---

## 5. Licence et provenance

- **Aucun fichier LICENSE / COPYING** (vérifié à HEAD et dans l'historique). Aucun en-tête de copyright. Le README ne cite pas le papier ; l'attribution repose sur les noms des committers (Seabra/Rente, co-auteurs).
- Sans licence, régime par défaut « tous droits réservés » : **lire, auditer et ré-implémenter les idées** (publiées dans le papier ; les algorithmes ne sont pas protégés par le droit d'auteur) : **oui**. **Copier/vendoriser le code ou les poids** : **non** sans accord écrit.
- **Recommandation : ré-implémentation from scratch en citant le papier** — propre et de toute façon nécessaire (§4). Question licence à Mert : optionnelle.

---

## 6. Écarts et surprises

1. **Le code des expériences du papier n'est pas publié.** HEAD = tutoriel d'août 2023 ; tout le code de recherche a été supprimé le 2023-12-10. Les figures du papier ne sont pas reproductibles depuis HEAD (grilles, seeds et pickles absents).
2. **Le cas dirigé est bancal dans le code publié** : `tutorial.ipynb` charge `directed.h5` tout en générant un graphe non dirigé ; **bug** dans `get_target(A, undirected=False)` (la boucle ne remplit que le triangle supérieur — la moitié des labels reste « déconnecté ») ; dans la version historique, `get_adjacency` faisait `triu` avant le test `undirected` (graphe « dirigé » en fait triangulaire) ; `get_upper_tr` lit le triangle **inférieur** malgré son nom — piège d'orientation majeur en dirigé.
3. **Aucune donnée cérébrale** malgré le nom du repo — que du synthétique.
4. **Incohérence poids fournis (200 features) ↔ notebook d'entraînement (100 features).**
5. **« Precision matrix » = inv(R̂₀) brut**, pas graphical lasso.
6. **Méthodologie d'entraînement discutable** : sélection du meilleur modèle sur le jeu de **test** (leakage) ; labels {1,2}+MSE+clustering ; entraînement en observabilité totale vs test partiel ; **aucun seed** ; seuil 0.5 vs 1.5 dupliqué et divergent entre `helper_functions.py` et `training.ipynb`.
7. **Détails non documentés dans le papier**, à retenir : lags des features K = 0..49 (l'offset 0 inclut R̂₀ et inv(R̂₀) — la precision matrix est déjà une feature du FFNN) ; conventions d'indexation qui sautent l'échantillon 0 ; **aucune soustraction de moyenne** dans les moments empiriques (à corriger absolument sur FX) ; StandardScaler **re-fitté sur chaque jeu de test** (invariance d'échelle implicite, à assumer ou remplacer).
8. **requirements.txt en UTF-16** : développement Windows sans revue ; ré-encoder avant tout `pip install`.
