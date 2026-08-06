# Plan du code — `fx_intraday_probe` (v1, revu et réévalué le 05/08)

Sonde de disponibilité/qualité EODHD, AVANT toute ingestion. Implémente les verdicts pré-enregistrés de
PLAN_PHASE1 §1.2. Aucune écriture Supabase (hors `etl_log`), aucune écriture R2 — lecture EODHD + artefacts
locaux + rapport.

## 0. Emplacement et intégration (frontière 1.6 : mécanique → swiss-wealth-etl)

- Module : `src/pipelines/fx_probe.py` (nouveau) dans swiss-wealth-etl.
- Entrée : `run_etl.py --fx-probe` (pattern de flags existant).
- Registre : entrée `fx-intraday-probe` dans `src/jobs_registry.py` (`schedule: None` = manuel,
  `pipelines: ["fx_probe"]`, `cost_credits_est: ~1400`, `active: True`) → l'audit v2 le suit d'office.
- Exécution locale (pas Railway — one-shot) ; artefacts → `reports/fx_probe/` (ETL repo, gitignoré sauf
  rapport) ; le `PROBE_REPORT.md` final est recopié/committé dans idsia-causal-fx/docs.

## 1. Configuration (en tête de module — AUCUN nombre magique dans le corps)

```python
FX_PAIRS = {  # devise -> spec
  "EUR": {"ticker": "EURUSD.FOREX", "usd_role": "price", "range": (0.8, 1.8), "tier": "hard"},
  "JPY": {"ticker": "USDJPY.FOREX", "usd_role": "base",  "range": (70, 170), "tier": "hard"},
  ... # 18 paires, plages plausibles discriminantes à 10^±1 près (PLAN §1.2 sous-unités)
}
SAMPLE_MONTHS = ["2010-02","2010-08","2015-02","2015-08","2020-02","2020-08","2025-02","2025-08"]
#   4 époques × (hiver, été) -> couvre DST (V31) et cohérence par époque ; + mois post-t0 si t0 > 2010-02.
THRESHOLDS = {  # verdicts pré-enregistrés §1.2 — NE PAS MODIFIER APRÈS EXÉCUTION
  "1m": {"coverage": 0.95, "stale": 0.30}, "15m": {"coverage": 0.97, "stale": 0.10},
  "1h": {"coverage": 0.98, "stale": 0.05}, "daily_coherence_bps": 10,
}
ROLLOVER_NY = (17, 18)   # zone exclue de la grille de verdict (17h-18h NY)
OUTLIER_ABS_DLOG = 0.05  # scan glitches ; + détection sauts ~log(10)/log(100) (tolérance ±5 %)
```

## 2. Algorithme, par paire

1. **t0 (dichotomie)** : bornes [2009-01-01, aujourd'hui] ; requête 1m sur fenêtre de 120 j au point médian ;
   vide → droite, non-vide → gauche ; ~6 requêtes → fenêtre contenant t0 ; t0 exact = timestamp de la
   première barre. Paire vide partout → verdict `ABSENT`. Garde-fou : vérifier que la réponse couvre bien
   la fenêtre demandée (l'API tronque silencieusement) via premier/dernier timestamp retournés.
2. **Mois-échantillons** : fetch 1m sur chaque mois de SAMPLE_MONTHS ≥ t0 (+ mois post-t0) — 1 requête/mois.
3. **Intégrité (classe temps)** : timestamps uniques, croissants, alignés à la minute ; compte des doublons
   et désordres (verdict `INTEGRITY_FAIL` si > 0,1 %).
4. **Convention (classe valeur, blocage dur V17)** : médiane du close dans la plage plausible ; sinon test
   du décalage 10^k (log10(médiane/centre de plage) arrondi) → verdict `CONVENTION_FAIL` ou `SCALE_10^k`.
5. **Grille de verdict** : minutes de la semaine FX (dim 17h NY → ven 17h NY, DST-conscient via
   zoneinfo America/New_York) MOINS la zone de rollover — la grille sur laquelle couverture et rassies
   sont jugées est celle que le panel utilisera (1.5).
6. **Métriques par fréquence** (1m natif ; 15m/1h agrégés depuis le 1m par close de fin d'intervalle) :
   couverture, taux de rassies — **globales ET par tranche horaire NY** (24 buckets, V31).
7. **Scan d'outliers** : max |Δlog|, compte > seuil, détection de sauts ≈ log(10)/log(100) (ruptures
   d'échelle mi-série).
8. **Empreinte week-end** : barres dans [ven 17h NY, dim 17h NY] — attendu ≈ 0 ; sinon compte + flag.
9. **Cohérence journalière** : close daily reconstruit (convention 17h NY) vs EOD officiel EODHD
   (`client.get_eod`, 1 cr/paire, tout l'historique en 1 requête) — médiane et p95 en bps, **par
   mois-échantillon** (détecte un changement de nature du prix par époque).
10. **Cross-check d'agrégation** (mois ≥ oct. 2020) : nos 15m/1h agrégés vs barres **natives** EODHD
    15m/1h sur le même mois (2 requêtes) — écart attendu nul sur les closes.
11. **Triangulation (V17)** : sur un mois-échantillon récent, fetch `EURJPY.FOREX` 1m et comparer au
    croisé triangulé `p_EUR − p_JPY` — médiane de l'écart en bps ; échec = erreur de normalisation.
12. **Verdicts par paire** : `ELIGIBLE_1M` / `ELIGIBLE_15M` / `ELIGIBLE_1H` / `EXCLUE` + t0 effectif
    (t0 + 1 mois plein) + drapeaux (SCALE, INTEGRITY, WEEKEND, COHERENCE).

## 3. Mécanique d'exécution

- **Cache disque** : chaque réponse brute sauvée en JSON gzip `reports/fx_probe/raw/{pair}_{interval}_{from}_{to}.json.gz`
  AVANT tout traitement ; relance = skip des fenêtres déjà en cache (idempotent, reprise sur erreur).
  Ces bruts servent aussi au **diff de révision** au backfill (V29).
- **Budget** : t0 ~6 req + 8-9 mois 1m + 1 EOD + 2 natifs + part triangulation ≈ 16-18 req/paire
  × 18 paires ≈ **300 requêtes ≈ 1 500 crédits** (marge sur l'estimation initiale de 1 000).
- **Rate limiting/instrumentation** : via `client.EODHDClient` existant — rien à écrire.
- **Journal** : une entrée `etl_log` (pipeline `fx_probe`) en fin de run (pattern `log_pipeline_run`).
- **Sorties** : `reports/fx_probe/metrics.parquet` (toutes métriques, longues) +
  `PROBE_REPORT.md` (tableau par paire : t0, verdicts par fréquence, drapeaux ; tableau par tranche
  horaire ; sections convention/cohérence/triangulation ; rappel des seuils et de leur statut
  pré-enregistré).
- **Tests** (dans swiss-wealth-etl, pytest, sans réseau) : sur fixtures synthétiques — dichotomie t0,
  détection 10^k, agrégation 15m/1h, grille DST hiver/été, rassies/couverture par bucket, détection de
  troncature API. Un fixture = un mois de barres générées avec défauts injectés connus.

## 4. Revue du plan (ce que la relecture a corrigé)

- **Troncature silencieuse de l'API** ajoutée en garde-fou (1) — sinon la dichotomie t0 peut converger faux.
- **La grille de verdict exclut rollover et week-end** (5) — sinon les seuils de couverture sont
  mécaniquement inatteignables (~4 % de barres mortes structurelles) et les verdicts faussés.
- **Cross-check natif et triangulation** intégrés à la sonde (10-11) plutôt que reportés à l'audit —
  ils coûtent 3 requêtes et bloquent des erreurs de fond AVANT le backfill.
- **Cache brut gzip systématique** avant traitement — réponse au double besoin idempotence + diff de
  révision (V29), et permet de re-calculer les métriques sans re-consommer de crédits si un seuil
  d'analyse (pas de verdict) doit être corrigé.
- **EOD daily par `get_eod` (1 cr)** plutôt qu'ingestion daily préalable — la référence journalière du
  check de cohérence ne nécessite PAS le job d'ingestion daily (qui reste optionnel, plus tard).

## 5. Preuve d'optimalité vs swiss-wealth-etl

Chaque besoin de la sonde correspond soit à un composant existant réutilisé tel quel, soit à du neuf
justifié par une absence vérifiée dans le repo :

| Besoin | Composant | Statut |
|---|---|---|
| HTTP EODHD, auth, retries | `src/client.py` (`EODHDClient._request`) | **Réutilisé** — zéro code HTTP neuf |
| Endpoint intraday | `client.get_intraday(symbol, interval, from_ts, to_ts)` | **Réutilisé tel quel** (signature vérifiée, timestamps unix) |
| Endpoint daily de référence | `client.get_eod` | **Réutilisé** |
| Comptage crédits + quota | RateLimiter en crédits + `eodhd_credits.py` (intraday = 5 cr déjà tarifé) | **Réutilisé** — l'incident quota de juillet est structurellement impossible à reproduire |
| Suivi du job par l'audit v2 | entrée `jobs_registry.py` | **Réutilisé** (une entrée à ajouter) |
| Convention d'entrée | flag `run_etl.py` | **Réutilisé** (pattern) |
| Journal des runs | `etl_log` (pattern `log_pipeline_run` de `intraday.py`) | **Réutilisé** (pattern) |
| Logique FX 24/5, grille NY, métriques, verdicts | — absente du repo (l'`intraday.py` existant est actions : heures de bourse par exchange, ticker_id UUID, table `intraday_prices` 7 j glissants — **vérifié inadapté** : le FX n'a ni exchange calendar ni ticker_id) | **Neuf, dans `fx_probe.py`** |
| Écriture R2 | `quant-portfolio-parquet` (autre repo) | **Hors périmètre sonde** (rien à écrire) |

Alternatives rejetées : (i) *script standalone hors ETL* → perdrait RateLimiter crédits, instrumentation
quota, etl_log et suivi d'audit — inacceptable après l'incident de juillet ; (ii) *étendre `intraday.py`* →
mélangerait deux domaines (calendriers de bourse vs 24/5, ticker_id vs paires) pour ~10 lignes partagées ;
(iii) *coder dans idsia-causal-fx* → violerait la frontière mécanique/science actée au 1.6.

## 6. Réévaluation (risques résiduels et parades)

1. **Taille des réponses** : 120 j de 1m ≈ 170k barres ≈ 15-20 Mo JSON — OK en mémoire, gzip sur disque ;
   parser en streaming inutile à cette échelle.
2. **t0 flou** (données éparses au démarrage d'une paire) : couvert par t0 effectif = t0 + 1 mois plein +
   le mois post-t0 ajouté aux échantillons.
3. **Barres natives 15m/1h absentes avant oct. 2020** : le cross-check (10) n'est possible que post-2020 —
   assumé, documenté dans le rapport.
4. **8 mois-échantillons peuvent rater un trou localisé** : assumé — c'est le rôle de l'audit post-backfill
   (V29, QC à deux étages) ; la sonde décide l'ÉLIGIBILITÉ, pas la complétude.
5. **Sensibilité des seuils** : le rapport publie les métriques CONTINUES (pas seulement les verdicts) —
   si une paire échoue à 31 % de rassies vs seuil 30 %, on le verra et la discussion sera factuelle
   (amendement de seuil = décision documentée AVANT tout usage aval, jamais après).
6. **Fuseau** : zoneinfo(America/New_York) sur les timestamps unix UTC d'EODHD — le seul point où une
   erreur serait silencieuse ; couvert par le test DST hiver/été et l'empreinte week-end.
7. Budget final : **~300 requêtes ≈ 1 500 crédits** (< 2 % du quota quotidien).
