# idsia-causal-fx

Application du cadre d'influence causale en apprentissage social (Kayaalp & Sayed, EPFL/IDSIA) aux devises :
identification du réseau causal entre devises à partir de leurs séries de force (type NEER), puis calcul de la
matrice d'influence causale et du CausalRank, et enfin exploitation dans un petit portefeuille backtesté.

Collaboration de recherche avec Mert Kayaalp (IDSIA/SUPSI), cc Oleg Szehr, Marco Zaffalon.

## Documents

- [docs/PLAN.md](docs/PLAN.md) — plan structuré en 7 phases (0 à 6), critères de passage.
- [docs/VIGILANCE.md](docs/VIGILANCE.md) — registre des points de vigilance, chacun avec son moment de rappel.
- [DECISIONS.md](DECISIONS.md) — journal daté des décisions.

## Références

1. Kayaalp & Sayed, *Causal Influences over Social Learning Networks*, 2023 — [arXiv 2307.09575](https://arxiv.org/abs/2307.09575)
2. Santos, Rente, Seabra & Moura, *Learning the Causal Structure of Networked Dynamical Systems under Latent Nodes and Structured Noise*, AAAI'24 — [arXiv 2312.05974](https://arxiv.org/abs/2312.05974) — code : [seabrapt/brain_underlying_structure_identification](https://github.com/seabrapt/brain_underlying_structure_identification)
3. Fil email « Literature causality » (mars–juillet 2026) — direction actée : causal discovery sur les devises, nœuds = devises (pas les paires), séries de force type NEER, majeures + mineures en sanity check, horizon = hyperparamètre critique.
