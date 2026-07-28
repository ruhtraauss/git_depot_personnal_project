# Dimensionnement PV + Batterie + Réseau — Optimisation MILP (Pyomo)

Ce notebook détermine, pour une maison individuelle en France, **combien de panneaux photovoltaïques et de modules de batterie installer**, ainsi que le **dispatch horaire optimal** (production PV, charge/décharge batterie, achat/vente réseau) qui **minimise le coût total annualisé** tout en couvrant la demande électrique à chaque heure de l'année.

Le problème est formulé comme un **MILP (Mixed-Integer Linear Program)** résolu avec [Pyomo](https://www.pyomo.org/) et le solveur open-source [HiGHS](https://highs.dev/).

---

## Sommaire

- [Ce que fait le notebook](#ce-que-fait-le-notebook)
- [Prérequis](#prérequis)
- [Données nécessaires](#données-nécessaires)
- [Comment l'utiliser](#comment-lutiliser)
- [Modélisation mathématique](#modélisation-mathématique)
  - [Ensembles](#ensembles)
  - [Paramètres](#paramètres)
  - [Variables de décision](#variables-de-décision)
  - [Fonction objectif](#fonction-objectif)
  - [Contraintes](#contraintes)
- [Hypothèses et limites](#hypothèses-et-limites)
- [Résultats produits](#résultats-produits)
- [Licence / avertissement](#licence--avertissement)

---

## Ce que fait le notebook

1. Calcule la **production PV horaire d'un panneau** sur une année type (TMY) via `pvlib`, pour un panneau DualSun Flash 425 Wc à Grenoble.
2. Récupère la **demande électrique horaire d'un foyer** à partir des profils de consommation publics Enedis (courbe de charge type), normalisée puis mise à l'échelle d'une consommation annuelle choisie (4500 kWh/an par défaut: représentatif d'un foyer typique français).
3. Récupère les **prix de l'électricité** :
   - contrat à tarif fixe (heures pleines / heures creuses),
   - contrat à prix spot (marché day-ahead ENTSO-E).
   - contrat de vente (prix fixe)
4. Construit et résout un **MILP** qui choisit :
   - le nombre de panneaux PV,
   - le nombre de modules de batterie,
   - le flux horaire d'énergie entre PV, batterie, réseau et maison,
   de façon à **minimiser le coût total annualisé** (investissement + abonnement + achat réseau − vente du surplus), sous contrainte de couvrir la demande à chaque heure.
5. Affiche les résultats (dimensionnement optimal, courbes horaires d'achat/vente/production/état de charge).

---

## Prérequis

- Python ≥ 3.10
- Bibliothèques :
  ```bash
  pip install pyomo pvlib pandas matplotlib entsoe-py highspy
  ```
- Le solveur **HiGHS** est installé automatiquement via le paquet `highspy` (pas besoin d'installation système séparée).

## Données nécessaires

| Donnée | Source | Où la placer |
|---|---|---|
| Courbe de charge type (`coefficients-des-profils.csv`) | [data.enedis.fr](https://data.enedis.fr) — dataset *"Courbes de charge — Profils Enedis"* | Même dossier que le notebook |
| Clé API ENTSO-E (optionnelle, pour le tarif spot) | [transparency.entsoe.eu](https://transparency.entsoe.eu/) — créer un compte gratuit et générer une clé dans les paramètres | Variable d'environnement `ENTSOE_API_KEY` |
| Données météo (TMY Grenoble) | Récupérées automatiquement via `pvlib.iotools.get_pvgis_tmy` (API PVGIS, gratuite, pas de clé requise) | — |

## Comment l'utiliser

1. Cloner le dépôt et se placer dans le dossier du projet.
2. Installer les dépendances (voir ci-dessus).
3. Télécharger `coefficients-des-profils.csv` depuis data.enedis.fr et le placer à côté du notebook.
4. (Optionnel) Définir `ENTSOE_API_KEY` si vous voulez calculer le tarif spot.
5. Ouvrir le notebook et exécuter les cellules dans l'ordre.
6. Adapter les paramètres à votre situation si besoin (voir tableau ci-dessous) :

| Paramètre | Cellule | Valeur par défaut | À adapter selon |
|---|---|---|---|
| `latitude`, `longitude` | Production PV | Grenoble | Votre localisation |
| `prix_panneau_pv` | Panneaux | 1000 €/panneau posé | Devis réel |
| `prix_kw_batterie`, `prix_kwh_batterie` | Batterie | 120 €/kW, 350 €/kWh | Devis réel |
| `capacite_module_batterie`, `puissance_module_batterie` | Batterie | 2.5 kWh / 1.25 kW | Produit batterie visé |
| `prix_energie_heure_creuse/pleine` | Prix élec | 0.1579 / 0.2065 €/kWh | Votre contrat réel |
| `taux_actualisation`, `duree_vie_pv`, `duree_vie_batterie` | Annualisation (CRF) | 4 %, 25 ans, 12 ans | Vos hypothèses financières |
| `demande_annuelle_elec_kwh` | Demande | 4500 kWh/an | Votre consommation réelle (relevé de compteur/facture) |

## Modélisation mathématique

### Ensembles

- $t \in \mathcal{T} = \{1, \dots, 8760\}$ : les heures de l'année.

### Paramètres

| Symbole | Description | Unité |
|---|---|---|
| $D_t$ | Demande électrique du foyer à l'heure $t$ | kWh |
| $\pi_t$ | Production PV d'**un seul** panneau à l'heure $t$ | kWh |
| $c^{ach}_t$ | Prix d'achat de l'électricité à l'heure $t$ (tarif fixe HC/HP) | €/kWh |
| $c^{vte}_t$ | Prix de vente du surplus à l'heure $t$ (constant, 0.011) | €/kWh |
| $c_{abo}$ | Abonnement annuel | € |
| $c_{pv}$ | Coût d'un panneau installé | € |
| $c^{kW}_{b}$, $c^{kWh}_{b}$ | Coût unitaire de puissance / capacité batterie | €/kW, €/kWh |
| $\eta^{ch}$, $\eta^{dc}$ | Rendements de charge / décharge batterie | – |
| $e_{mod}$, $p_{mod}$ | Capacité / puissance d'un module de batterie standard | kWh, kW |
| $CRF_{pv}$, $CRF_{b}$ | Facteurs d'annualisation du capex (PV, batterie) | – |
| $M$ | Grand-M (borne haute des flux horaires) | kWh/h |

Le facteur d'annualisation (Capital Recovery Factor) répartit un investissement payé une seule fois sur sa durée de vie, à un taux d'actualisation $r$ :

$$CRF(r, n) = \frac{r(1+r)^n}{(1+r)^n - 1}$$

### Variables de décision

**Dimensionnement (entiers) :**
- $n_{pv} \in \mathbb{N}$ : nombre de panneaux PV installés
- $n_b \in \mathbb{N}$ : nombre de modules de batterie installés

**Dimensionnement dérivé (continues, liées aux entiers ci-dessus) :**
- $P^{max}_b, E^{max}_b \ge 0$ : puissance (kW) et capacité (kWh) résultantes de la batterie

**Flux horaires (continus, $\ge 0$, en kWh/h) pour tout $t$ :**
`achat`, `vente`, `pv_maison`, `pv_batterie`, `pv_grid`, `grid_batterie`, `grid_maison`, `batterie_maison`, `batterie_grid`, `echarge`, `edecharge`

**État de charge :** $E_t \ge 0$ (kWh)

**Binaires :** $b_t, y_t \in \{0,1\}$ (exclusivité charge/décharge, exclusivité achat/vente)

### Fonction objectif

Minimiser le coût total annuel = abonnement + capex annualisé (PV + batterie) + coût net d'électricité :

$$
\min \; c_{abo} \;+\; n_{pv}\, c_{pv}\, CRF_{pv} \;+\; \big(P^{max}_b\, c^{kW}_{b} + E^{max}_b\, c^{kWh}_{b}\big)\, CRF_{b} \;+\; \sum_{t \in \mathcal{T}} \Big( achat_t \cdot c^{ach}_t - vente_t \cdot c^{vte}_t \Big)
$$

### Contraintes

**1. Bilan de production PV** — la production totale (nombre de panneaux × production unitaire) se répartit entre autoconsommation directe, charge batterie et injection réseau :
$$\pi_t \cdot n_{pv} = pv\_maison_t + pv\_batterie_t + pv\_grid_t \qquad \forall t$$

**2. Bilan achat réseau** — l'électricité achetée sert à la maison et/ou à charger la batterie :
$$achat_t = grid\_batterie_t + grid\_maison_t \qquad \forall t$$

**3. Bilan vente réseau** — l'électricité vendue provient du PV et/ou de la batterie :
$$vente_t = batterie\_grid_t + pv\_grid_t \qquad \forall t$$

**4. Bilan charge/décharge batterie :**
$$echarge_t = grid\_batterie_t + pv\_batterie_t \qquad \forall t$$
$$edecharge_t = batterie\_maison_t + batterie\_grid_t \qquad \forall t$$

**5. Dynamique de l'état de charge** (avec pertes de rendement) :
$$E_t = E_{t-1} + echarge_t \cdot \eta^{ch} - \frac{edecharge_t}{\eta^{dc}} \qquad \forall t>1$$
$$E_1 = 0 \qquad E_{8760} = 0 \quad \text{(cycle annuel fermé)}$$

**6. Bornes de capacité et de puissance :**
$$0 \le E_t \le E^{max}_b \qquad \forall t$$
$$echarge_t \le P^{max}_b \qquad edecharge_t \le P^{max}_b \qquad \forall t$$

**7. Discrétisation de la batterie** — capacité et puissance sont des multiples entiers d'un module standard (et non des variables continues, pour rester réaliste commercialement) :
$$E^{max}_b = n_b \cdot e_{mod} \qquad P^{max}_b = n_b \cdot p_{mod}$$

**8. Exclusivité charge / décharge** (on ne charge pas et ne décharge pas la batterie à la même heure), via grand-M et une variable binaire $b_t$ :
$$echarge_t \le M \cdot b_t \qquad edecharge_t \le M \cdot (1-b_t) \qquad \forall t$$

**9. Exclusivité achat / vente** (on n'achète pas et ne vend pas au réseau à la même heure), via grand-M et une variable binaire $y_t$ :
$$achat_t \le M \cdot y_t \qquad vente_t \le M \cdot (1-y_t) \qquad \forall t$$

**10. Couverture de la demande** — contrainte centrale du problème, la demande est satisfaite à chaque heure par le PV direct, le réseau ou la batterie :
$$D_t = pv\_maison_t + grid\_maison_t + batterie\_maison_t \qquad \forall t$$

## Hypothèses et limites

- **Année type, pas une année réelle** : la météo (TMY, via PVGIS) et le profil de demande (courbe de charge Enedis d'une année donnée) sont des profils représentatifs, pas des mesures réelles du foyer.
- **Annualisation du capex (CRF)** : suppose un taux d'actualisation constant et une durée de vie fixe pour le PV (25 ans) et la batterie (12 ans), sans dégradation progressive de performance (le PV perd en réalité ~0.5 %/an de rendement, non modélisé ici).
- **Batterie discrétisée** mais le ratio puissance/capacité d'un module (0.5C par défaut) est figé ; changer de produit batterie implique de changer `capacite_module_batterie` et `puissance_module_batterie`.
- **Grand-M = 20 kWh/h** : suffisant pour une échelle résidentielle (~4500 kWh/an) mais à revérifier si vous simulez une consommation ou un dimensionnement PV beaucoup plus élevé (le pic horaire réel doit rester `<< M`).
- **Cycle batterie annuel fermé** ($E_1 = E_{8760} = 0$) : pas de report de stock d'une année sur l'autre (cohérent avec un modèle mono-année).
- **Le contrat à prix spot est calculé mais pas utilisé par le modèle** (le modèle utilise uniquement le tarif fixe). Pour comparer les deux contrats, il faut relancer manuellement le modèle en remplaçant `df_achat_elec = df_achat_elec_contrat_fixe` par `df_achat_elec = df_achat_elec_contrat_spot`.
- **Pas de contrainte de surface de toit ni de budget maximal** : rien n'empêche numériquement le solveur de proposer un grand nombre de panneaux/modules si c'est économiquement optimal ; ajoutez une contrainte si vous voulez plafonner `n_pv` ou `n_b`.
- **Prix de vente du surplus fixe (0.011 €/kWh)** : à ajuster selon que vous visez l'obligation d'achat réglementée ou une simple valorisation libre.

## Résultats produits

À l'exécution, le notebook affiche :
- le nombre optimal de panneaux PV et de modules de batterie (et la puissance/capacité résultantes),
- le coût total annualisé optimal,
- les courbes horaires sur l'année : achat réseau, vente réseau, autoconsommation PV directe, soutirage réseau vers la maison, état de charge de la batterie.

## Licence / avertissement

Projet personnel à but pédagogique. Les résultats dépendent fortement des hypothèses de coût, de durée de vie et de prix de l'électricité renseignées : ils ne constituent pas un conseil d'investissement ou d'ingénierie et doivent être vérifiés/adaptés avant toute décision réelle d'installation.
