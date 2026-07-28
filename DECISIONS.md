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

## À trancher plus tard (parqué, ne pas perdre)

- Estimateur des poids 1B : Granger restreint vs R̂₁−R̂₃ rescalé → tranché par le test synthétique (Phase 4).
- NBSL vs ASL et paramètres du SLM autres que A (V7) → Phase 3.
- Fréquence des données (journalier vs intraday) → Phase 1, dépend de V3.
- Instruments et moteur de backtest de la Phase 6 (spot/forwards vs ETF ; quant-portfolio-base vs standalone).
