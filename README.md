# Dimensionnement PV + Batterie + Réseau — Optimisation MILP (Pyomo)

Ce notebook détermine, pour une maison individuelle en France, **combien de panneaux photovoltaïques et de modules de batterie installer**, ainsi que le **dispatch horaire optimal** (production PV, charge/décharge batterie, achat/vente réseau) qui **minimise le coût total annualisé** tout en couvrant la demande électrique à chaque heure de l'année.

Au-delà de l'optimum de marché pur, le notebook **chiffre le coût de la transition énergétique** : en plafonnant progressivement la part de la demande annuelle qui peut être couverte par des achats réseau, on force le système vers plus d'autonomie et on regarde combien ça coûte à chaque palier.

Le problème est formulé comme un **MILP (Mixed-Integer Linear Program)** résolu avec [Pyomo](https://www.pyomo.org/) et le solveur open-source [HiGHS](https://highs.dev/). La modélisation mathématique complète (ensembles, paramètres, variables, fonction objectif, contraintes) est dans [`modelisation.tex`](modelisation.tex) / [`modelisation.pdf`](modelisation.pdf) — ce README n'en donne qu'un résumé pratique.

---

## Sommaire

- [Ce que fait le notebook](#ce-que-fait-le-notebook)
- [Prérequis](#prérequis)
- [Données nécessaires](#données-nécessaires)
- [Comment l'utiliser](#comment-lutiliser)
- [Options du modèle](#options-du-modèle)
- [Analyse de sensibilité](#analyse-de-sensibilité)
- [Hypothèses et limites](#hypothèses-et-limites)
- [Licence / avertissement](#licence--avertissement)

---

## Ce que fait le notebook

1. Calcule la **production PV horaire d'un panneau** sur une année type (TMY) via `pvlib`, pour un panneau DualSun Flash 425 Wc à Grenoble.
2. Récupère la **demande électrique horaire d'un foyer** à partir des profils de consommation publics Enedis (courbe de charge type), normalisée puis mise à l'échelle d'une consommation annuelle choisie (4500 kWh/an par défaut : représentatif d'un foyer typique français).
3. Récupère les **prix de l'électricité** :
   - achat en tarif fixe (heures pleines / heures creuses),
   - achat au prix spot (marché day-ahead ENTSO-E),
   - vente du surplus à prix fixe (0.011 €/kWh).
4. Construit et résout un **MILP** qui choisit le nombre de panneaux PV, le nombre de modules de batterie, et le flux horaire d'énergie entre PV/batterie/réseau/maison, pour minimiser le coût total annualisé (investissement + abonnement + achat réseau − vente du surplus), sous contrainte de couvrir la demande à chaque heure.
5. Balaie un **plafond d'achat réseau** (100% = marché libre, jusqu'à 0% = autonomie totale imposée), pour les deux types de contrat séparément, et affiche le dimensionnement optimal, le coût total, et sa décomposition par poste à chaque palier.
6. Fait une **analyse de sensibilité** sur les hypothèses de coût (prix PV, prix batterie, taux d'actualisation, durée de vie batterie) pour juger de leur importance relative sur les conclusions.

---

## Prérequis

```bash
pip install pyomo pvlib pandas matplotlib entsoe-py highspy
```

Le solveur **HiGHS** s'installe automatiquement via `highspy`, pas besoin d'installation système séparée.

## Données nécessaires

| Donnée | Source | Où la placer |
|---|---|---|
| Courbe de charge type (`coefficients-des-profils.csv`) | [data.enedis.fr](https://data.enedis.fr) — dataset *"Courbes de charge — Profils Enedis"* | Même dossier que le notebook |
| Clé API ENTSO-E (optionnelle, pour le tarif spot) | [transparency.entsoe.eu](https://transparency.entsoe.eu/) — compte gratuit | Demandée de façon sécurisée au lancement (`getpass`), jamais codée en dur |
| Données météo (TMY Grenoble) | Récupérées automatiquement via `pvlib.iotools.get_pvgis_tmy` (API PVGIS, gratuite) | — |

## Comment l'utiliser

1. Cloner le dépôt, installer les dépendances.
2. Télécharger `coefficients-des-profils.csv` depuis data.enedis.fr/github et le placer à côté du notebook.
3. (Optionnel) Avoir une clé ENTSO-E sous la main si vous voulez le tarif spot.
4. Exécuter les cellules dans l'ordre. Résolution quasi-instantanée (quelques secondes par scénario)
5. Adapter les paramètres à votre situation si besoin :

| Paramètre | Valeur par défaut | À adapter selon |
|---|---|---|
| `latitude`, `longitude` | Grenoble | Votre localisation |
| `demande_annuelle_elec_kwh` | 4500 kWh/an | Votre consommation réelle |
| `prix_panneau_pv` | 1000 €/panneau posé | Devis réel |
| `prix_kw_batterie`, `prix_kwh_batterie` | (120 €/kW), 1000 €/kWh | Devis réel |
Info : dans le code, la batterie est discrétisée (ne peut prendre que 2.5 kwh/1.25 kw, donc 1 seul cout de la batterie (1000€/kwh))
| `capacite_module_batterie`, `puissance_module_batterie` | 2.5 kWh / 1.25 kW | Produit batterie visé |
| `prix_energie_heure_creuse/pleine` | 0.1579 / 0.2065 €/kWh | Votre contrat réel |
| `taux_actualisation`, `duree_vie_pv`, `duree_vie_batterie` | 4 %, 25 ans, 12 ans | Vos hypothèses financières |

## Options du modèle

La fonction `construire_et_resoudre_modele(...)` expose plusieurs leviers, tous désactivés/neutres par défaut :

| Option | Défaut | Effet |
|---|---|---|
| `taux_autoproduction_min` | `None` | Plafonne le **total annuel** des achats réseau à cette fraction de la demande (`1.0` = marché libre, `0.0` = autonomie totale imposée à chaque heure) |
| `autoriser_arbitrage_trading_batterie` | `False` | Si `True`, la batterie peut se charger depuis le réseau (pas seulement depuis le PV) — ouvre la porte à de l'arbitrage tarifaire, plus proche du trading que de l'autoconsommation |
| `autoriser_deconnexion_reseau` | `False` | Si `True`, un binaire rend l'abonnement réseau optionnel (le foyer peut choisir de se couper complètement du réseau: achat et vente) |
| `prix_panneau_pv_override`, `prix_kw_batterie_override`, `prix_kwh_batterie_override`, `taux_actualisation_override`, `duree_vie_pv_override`, `duree_vie_batterie_override` | `None` | Remplacent ponctuellement une hypothèse économique sans toucher aux valeurs globales du notebook — utilisé pour l'analyse de sensibilité |

## Analyse de sensibilité

Les hypothèses de coût (prix PV, prix batterie, taux d'actualisation, durée de vie de la batterie) sont incertaines. Le notebook fait varier chacune indépendamment (les autres fixées à leur valeur par défaut), puis combine les deux leviers les plus incertains (prix PV × prix batterie) dans une carte de chaleur, pour identifier lesquelles changent réellement les conclusions et lesquelles sont anecdotiques.

## Hypothèses et limites

- **Année météo et profil de demande représentatifs**, pas des mesures réelles du foyer.
- **Le plafond d'achat réseau est annuel, pas horaire** : le solveur reste libre de placer les achats autorisés au moment le plus économique de l'année.
- **Batterie discrétisée par modules entiers** de ratio puissance/capacité fixe (0.5C par défaut) — un modèle non discrétisé pourrait en théorie trouver un ratio économiquement optimal différent, mais ne correspondant à aucun produit commercial réel ; la discrétisation contraint volontairement le résultat à ce qui existe sur le marché.
- **Capex annualisé via CRF** (taux d'actualisation constant, durée de vie fixe), sans dégradation progressive de performance du PV (~0.5%/an en réalité, non modélisée).
- **Cycle de batterie circulaire** (l'état de fin d'année alimente le début), pas un stock figé à 0 à une heure arbitraire du calendrier — évite une infaisabilité artificielle plutôt qu'une hypothèse physique en soi.
- **Arbitrage batterie désactivé par défaut** : la batterie ne se charge qu'à partir du PV, pas du réseau (voir [Options du modèle](#options-du-modèle)).
- **Pas de plafond de surface de toit ni de budget maximal** : rien n'empêche numériquement un grand nombre de panneaux/modules si c'est économiquement optimal.
- **Deux contrats testés séparément** (tarif fixe, prix spot) : pas de comparaison automatique, il faut relancer le même balayage de scénarios avec l'autre série de prix.

## Licence / avertissement

Projet personnel à but pédagogique. Les résultats dépendent fortement des hypothèses de coût, de durée de vie et de prix de l'électricité renseignées : ils ne constituent pas un conseil d'investissement ou d'ingénierie et doivent être vérifiés/adaptés avant toute décision réelle d'installation.
