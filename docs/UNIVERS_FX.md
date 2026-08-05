# Univers FX — preuve d'exhaustivité par recensement

**Question** : l'univers (étages 1-3) omet-il une devise qui aurait dû y être ?
**Méthode de preuve** : partir du recensement complet du marché des changes — l'enquête triennale BIS 2022
([rpfx22](https://www.bis.org/statistics/rpfx22_fx.htm), 52 juridictions, ~1 200 dealers), qui rapporte
individuellement les **39 devises** représentant la quasi-totalité du turnover mondial — et **disposer de
chacune** contre nos critères. Toute devise hors de ce tableau fait < 0,1 % du turnover.

## Critères (rappel, PLAN_PHASE1 §1.1)

R1 flottement libre (FMI AREAER, de facto) · R2 liquidité suffisante pour un prix 1-min continu (~≥ 0,2 %
du turnover BIS) · R3 pas de contrôle de capitaux significatif · R4 pas d'ancrage à une devise de l'univers ·
R5 marché fonctionnel · (R6 profondeur EODHD — arbitré par la sonde, V25).

## Disposition des 39 devises BIS (part 2022 approximative, sur 200 %)

| Devise | Part | Disposition | Motif |
|---|---|---|---|
| USD ~88 % | | **Étage 1** | — |
| EUR ~31 % | | **Étage 1** | — |
| JPY ~17 % | | **Étage 1** | flags d'événements (interventions MoF 2022, 2024) |
| GBP ~13 % | | **Étage 1** | flag événement sept.-oct. 2022 |
| CNY ~7 % | | Exclue | R1 : flottement géré (bande PBoC) + R3 contrôles |
| AUD ~6,4 % | | **Étage 1** | — |
| CAD ~6,2 % | | **Étage 1** | — |
| CHF ~5,2 % | | **Étage 1** | fenêtres 2011-2015 inéligibles (plancher BNS), 2015-2022 flaguées |
| HKD ~2,6 % | | Exclue | R1/R4 : currency board USD |
| SGD ~2,4 % | | Exclue (2ᵉ vague) | R1 : NEER administré par la MAS (BBC) — l'objet même qu'on mesure |
| SEK ~2,2 % | | **Étage 1** | — |
| KRW ~1,9 % | | Exclue (2ᵉ vague) | R1/R3 : interventions BoK, mesures de flux |
| NOK ~1,7 % | | **Étage 1** | — |
| NZD ~1,7 % | | **Étage 1** | — |
| INR ~1,6 % | | Exclue (2ᵉ vague) | R1 : fortement gérée (RBI) |
| MXN ~1,5 % | | **Étage 2** | — |
| TWD ~1,1 % | | Exclue | R1 : gérée (CBC) |
| ZAR ~1,0 % | | **Étage 2** | — |
| BRL ~0,9 % | | Exclue (2ᵉ vague) | R3 : contrôles (IOF), segmentation onshore/offshore |
| DKK ~0,7 % | | Exclue | R4 : ERM2, arrimée EUR (copie du nœud EUR) |
| PLN ~0,7 % | | **Étage 2** | — |
| THB ~0,4 % | | Exclue | R1 : interventions BoT substantielles |
| ILS ~0,4 % | | **Étage 2** | flag interventions BoI ≤ ~2022 |
| IDR ~0,4 % | | Exclue | R1 : gérée (BI) |
| CZK ~0,4 % | | **Étage 2** | flag plancher CNB nov. 2013 - avr. 2017 (fenêtres inéligibles) |
| AED ~0,4 % | | Exclue | R1 : peg USD |
| TRY ~0,4 % | | Exclue | R1/R5 : inflation extrême, interventions, non-stationnarité structurelle |
| HUF ~0,3 % | | **Étage 2** | — |
| CLP ~0,3 % | | **Étage 3 (témoin)** | free floating AREAER ; interventions exceptionnelles annoncées (2019, 2022) → flags |
| SAR ~0,2 % | | Exclue | R1 : peg USD |
| PHP ~0,2 % | | **Étage 3 (témoin, avec réserve)** | flottante mais lissage BSP — témoin optionnel |
| MYR ~0,2 % | | Exclue | R1/R3 : gérée, non délivrable offshore |
| COP ~0,2 % | | **Étage 3 (témoin)** | free floating AREAER |
| RON ~0,1 % | | Exclue | R1/R2 : flottement géré (BNR), illiquide |
| RUB | | Exclue | R5 : marché cassé depuis 2022 (sanctions, contrôles) |
| PEN ~0,1 % | | Exclue | R1 : interventions BCRP permanentes |
| ARS | | Exclue | R3 : contrôles de change massifs |
| Autres rapportées (BGN arrimée EUR, devises du Golfe peggées, HRK entrée euro 2023…) | <0,1 % | Exclues | R1/R2/R4 |

**Cas limite honnête** : ISK (couronne islandaise) — authentiquement flottante depuis 2017, mais < 0,05 % du
turnover : exclue par R2 seul. C'est la seule devise flottante au monde écartée uniquement pour liquidité.

## Conclusion (la preuve)

1. **Étage 1 = exactement l'ensemble {flottante ∧ part ≥ 1,5 % ∧ non gérée} = le G10.** Entre NZD/NOK
   (~1,7 %) et MXN (~1,5 %), toutes les devises (HKD, SGD, KRW, TWD, INR) échouent à R1/R4. Aucune majeure
   flottante n'est omise — le G10 est un plafond structurel, pas un choix.
2. **Étage 2 = exactement les flotteurs de la bande 0,3-1,5 % sans réserve rédhibitoire** : MXN, ZAR, PLN,
   HUF, CZK, ILS. Les autres flotteurs de la bande échouent tous à un critère (BRL contrôles, THB/IDR
   interventions, TRY inflation) ou sont affectés aux témoins.
3. **Étage 3 = les flotteurs sous ~0,3 %** utilisables comme contrôles négatifs : CLP, COP (+ PHP en option).
4. **Tout le reste du monde** (~140 devises) est sous le plancher de liquidité R2 : pas de prix 1-min de
   marché continu, pas de couverture données fiable, influence plausible sur une majeure nulle.
5. **Argument théorique décisif** : l'omission d'un nœud n'introduit PAS de biais non maîtrisé — le cadre
   AAAI'24 est précisément construit pour l'**observabilité partielle** : toute devise exclue est un nœud
   latent, son effet est absorbé par le bruit corrélé et le biais E_S caractérisé (éq. 5 du papier).
   L'exhaustivité requise n'est donc pas celle des nœuds, mais celle de la **disposition** — établie ci-dessus.

Sources : [BIS Triennial 2022 — FX turnover](https://www.bis.org/statistics/rpfx22_fx.htm) ·
[BIS rpfx22](https://www.bis.org/statistics/rpfx22.htm) · classifications de régime : FMI AREAER (de facto),
millésimes 2022-2023. Révision prévue quand l'enquête 2025 ([rpfx25](https://www.bis.org/statistics/rpfx25_fx.htm))
sera intégrée — aucun changement de régime attendu qui modifierait la disposition.
