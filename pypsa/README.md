# Modélisation du dispatch électrique français avec PyPSA

Reconstitution du mix électrique français (2015-2026) sous forme de modèle de
dispatch économique, comparaison au réel observé, puis scénarios prospectifs
(prix du carbone, indisponibilité massive du nucléaire).

## Objectif

Deux questions guident ce projet :
- Un modèle de dispatch purement économique reproduit-il le comportement
  réel du système électrique français ?
- Là où il diverge, pourquoi — et qu'est-ce que ça révèle sur les
  contraintes qui pèsent réellement sur le système ?

## Résultats clés

- **Le nucléaire est reproduit à 2% près** par le solveur — bonne validation
  de la construction générale du modèle (capacités, disponibilités, merit
  order).
- **Le thermique pilotable (gaz, charbon, fioul) est massivement sous-utilisé**
  par rapport au réel (-90% à -99% selon la filière). L'analyse de
  sensibilité et le scan de prix du carbone montrent que ce n'est **pas**
  un problème de calibration de coûts : même un carbone à 5 fois son
  niveau observé ne referme pas l'écart. La cause la plus probable est
  l'absence de contraintes physiques sur le nucléaire (rampes, minimum
  technique) dans le modèle actuel, qui lui permet de moduler
  instantanément sa production — un comportement économiquement optimal
  mais physiquement irréaliste.
- **Le scénario de crise nucléaire** (indisponibilité ×1,5 à ×3) montre une
  cascade de substitution cohérente (gaz → charbon/fioul → renouvelables
  thermiques), et révèle la limite du bus unique : au-delà d'un facteur
  ~2, le modèle ne peut plus couvrir la demande sans recourir massivement
  à un générateur de secours fictif — en écho direct à la mobilisation
  réelle des interconnexions européennes lors de la crise de disponibilité
  nucléaire de 2022.

## Sources de données

| Donnée | Source | Granularité |
|---|---|---|
| Consommation réalisée | API RTE (éCO2mix) | horaire |
| Production éolien/solaire | API RTE (éCO2mix) | horaire |
| Production par filière (nucléaire, thermique, hydro, biomasse, déchets) | API RTE (Actual Generation) | ~30 min, agrégée à l'heure |
| Indisponibilités de production | API RTE (Generation Unavailabilities) | événementiel, étalé sur l'heure |
| Capacités installées par filière | API RTE (Installed Capacities) | annuelle, étalée sur l'heure |
| Solde import/export | API RTE (interconnexions) | horaire |
| Prix du gaz (TTF) et du carbone (EUA) | yfinance | journalier, forward-fillé |
| Température | Open-Meteo | journalière |

Toutes les sources sont publiques et gratuites (compte développeur RTE
requis pour les endpoints API, sans coût).

## Notebook PyPSA

Le run de référence (`build_and_solve_network()`) prend quelques minutes ;
les scénarios de sensibilité et de crise nucléaire relancent le solveur
plusieurs fois et peuvent prendre de quelques minutes à plusieurs heures
selon le facteur testé (le temps de résolution se dégrade fortement quand
le délestage devient actif sur beaucoup d'heures).

## Modélisation

- **Bus unique** (pas de contraintes réseau internes, échanges avec
  l'étranger représentés par un solde agrégé).
- **Non pilotables** (production réelle imposée comme plafond) : solaire,
  éolien, hydraulique fil de l'eau, déchets.
- **Pilotables** (le solveur choisit la production sous contrainte de
  disponibilité) : nucléaire, gaz, charbon, fioul, biomasse.
- **Stockage** : réservoirs hydrauliques et STEP modélisés en
  `StorageUnit`, avec un `max_hours` supposé (saisonnier pour les
  réservoirs, journalier pour les STEP) faute de donnée mesurée.
- **Générateur de secours** à coût prohibitif (délestage), pour garantir
  la faisabilité du solveur sur les heures où la capacité française seule
  ne suffit pas — son usage est lui-même un résultat (négligeable en
  situation normale, significatif sous crise nucléaire sévère).

## Limites assumées

- Pas de contrainte de rampe ni de minimum technique sur le nucléaire et
  le thermique — c'est la simplification qui pèse le plus sur l'écart
  entre modèle et réel.
- Prix du charbon et du fioul fixés par hypothèse de littérature, faute
  de série de marché collectée (contrairement au gaz et au carbone, basés
  sur des séries réelles TTF/EUA).
- `max_hours` des stockages hydrauliques non mesuré, posé par hypothèse.
- Bus unique national : pas de détail des échanges pays par pays.
- Pas d'engagement d'unité (unit commitment) : aucun coût de démarrage ni
  durée minimale de marche/arrêt.

## Perspectives

- Ajouter les contraintes de rampe manquantes sur le nucléaire et le
  thermique.
- Affiner la modélisation du stockage hydraulique (données de réservoir
  réelles si disponibles).
- Passer à une topologie multi-nœuds pour représenter finement le rôle
  des interconnexions, en particulier pour approfondir le scénario de
  crise nucléaire.
- Approfondir la contrainte CO2 en volume (plafond d'émissions plutôt que
  scan de prix), calibrée sur le dispatch optimisé de référence plutôt
  que sur les émissions réelles historiques.

## Auteur

Arthur Aussenac — IMT Atlantique, spécialisation mathématiques appliquées
et énergies renouvelables.
