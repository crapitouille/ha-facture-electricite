# Facture d'électricité pour Home Assistant

Estimation de la facture d'électricité **par jour, mois et année** à partir d'un compteur P1 (heures pleines / heures creuses), avec un **comparatif** entre le tarif DOUBLE (HP/HC) et un tarif dynamique de type VARIO (prix par créneau de 15 minutes).

Le compteur P1 et les capteurs de tarifs sont **choisis depuis le dashboard** : rien n'est écrit en dur dans le rapport.

## Contenu

| Fichier | Rôle |
|:--|:--|
| `packages/facture_electricite.yaml` | Tout le calcul : sélections, compteurs, capteurs, script de configuration |
| `dashboards/facture_electricite.yaml` | Dashboard à 2 vues : « Facture et comparatif » et « Paramètres » |

## Prérequis

- Home Assistant récent (syntaxe `triggers:` / `trigger:` des templates, version 2024.10 ou plus).
- Un compteur P1 qui expose, en kWh (les Wh et MWh sont convertis), l'énergie importée en **HP**, l'énergie importée en **HC** et l'énergie **injectée**, avec `state_class: total_increasing`.
- Un capteur de **tarif actif** dont l'état vaut `1` en heures pleines et `2` en heures creuses.
- Les prix du tarif DOUBLE (HP et HC) et, pour le comparatif, une **grille de prix VARIO** : un capteur dont l'attribut `prices` est une liste `[[heure ISO, prix], …]` par créneau de 15 minutes.

Valeurs par défaut (posées au premier démarrage, seulement si le champ est vide) : compteur **HomeWizard P1** et intégration **Groupe E Tariffs** (`sensor.p1_meter_*` et `sensor.groupe_e_tariffs_v2_*`). Avec un autre matériel, choisis tes entités dans la vue « Paramètres ».

## Installation

1. Active les packages dans `configuration.yaml` si ce n'est pas déjà fait :
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copie `packages/facture_electricite.yaml` dans `/config/packages/`.
3. *Outils de développement > YAML > Vérifier la configuration*, puis **redémarre** Home Assistant (les `input_*`, `utility_meter` et `script` en YAML demandent un redémarrage).
4. Crée un dashboard (*Paramètres > Tableaux de bord > Ajouter*), ouvre-le, puis *⋮ > Modifier > ⋮ > Éditeur de configuration brute*, et colle le contenu de `dashboards/facture_electricite.yaml`.
5. Dans la vue **Paramètres**, vérifie les entités lues (ou clique sur « Choisir les entités… »), puis renseigne **Prix reprise injection** et **Frais fixes / mois** dans la vue « Facture et comparatif ».

## Choisir ses entités

Sept sélections, stockées dans les `input_text.facture_source_*` :

| Sélection | Entité attendue |
|:--|:--|
| Import HP | énergie importée, tarif 1 (kWh) |
| Import HC | énergie importée, tarif 2 (kWh) |
| Injection | énergie injectée (kWh) |
| Tarif actif | état `1` = HP, `2` = HC |
| Prix DOUBLE HP / HC | prix du kWh en CHF/kWh |
| Grille VARIO | attribut `prices` = `[[heure, prix], …]` |

Elles se changent par le script **« Facture : choisir les sources »** (bouton de la vue Paramètres, listes de recherche) ou en tapant directement l'`entity_id` dans la vue.

## Comment ça marche

- **Miroirs** (`sensor.facture_p1_*`) : capteurs qui recopient les entités choisies. Tout le reste du package ne lit que ces miroirs.
- **Facture estimée** = kWh HP × prix HP + kWh HC × prix HC − kWh injectés × prix de reprise + frais fixes au prorata. « Aujourd'hui » court depuis minuit (compteurs à remise à zéro quotidienne). Les frais fixes sont comptés **depuis le début du suivi** (`Frais fixes comptés depuis`, posé à la date d'installation et modifiable dans le dashboard) : sans cela, l'estimation de l'année inclurait les mois précédant l'installation.
- **Coûts cumulés DOUBLE et VARIO** : à chaque lecture du compteur, les kWh consommés depuis la lecture précédente sont multipliés par le prix du moment (HP ou HC pour DOUBLE, créneau de 15 minutes pour VARIO). Les compteurs jour / mois / année en sont déduits. Le dernier relevé de chaque compteur est mémorisé : si le P1 devient indisponible puis revient, les kWh consommés pendant la coupure sont comptés au retour (au prix du moment). Le prix VARIO retient le dernier créneau de 15 min commencé (jusqu'à 1 h), un trou dans la grille du fournisseur ne donne donc pas un prix de 0.
- **Comparatif** : achat d'électricité uniquement (import réseau), sans reprise d'injection ni frais fixes.

Entités créées : `sensor.facture_p1_*`, `sensor.facture_prix_*`, `sensor.facture_estimee_{jour,mois,annee}`, `sensor.facture_conso_{hp,hc}_*`, `sensor.facture_injection_*`, `sensor.facture_cout_{double,vario}_*`, `sensor.facture_vario_{min,max}_du_jour`, `sensor.facture_tarif_actif`, plus les `input_*`, le script et une automatisation de démarrage (`facture_electricite_defauts`).

## À savoir

- **Rien n'est rétroactif.** Les compteurs commencent à zéro à l'installation (la première lecture sert de référence). La facture « année » ne couvre que la période suivie, et le comparatif DOUBLE / VARIO ne se calcule que vers l'avant.
- **Changer de compteur P1** : la première lecture du nouveau compteur crée un saut dans les compteurs jour / mois / année. Recalibre-les ensuite (`utility_meter.calibrate`). Les coûts cumulés ignorent ce saut.
- **Prix DOUBLE** : le prix HP est lu dans le capteur choisi pour « Prix DOUBLE HP » (avec Groupe E Tariffs : `…_double_max_price_today`) et le prix HC dans « Prix DOUBLE HC » (`…_double_min_price_today`). Les prix sont donc ceux du jour, pas un historique de tarifs.
- **Graphique « Prix du kWh (24 dernières heures) »** : seul élément glissant du dashboard (un graphique d'historique ne sait pas afficher « depuis minuit »). Tous les montants « Aujourd'hui » partent de minuit.
- **Précision** : le compteur P1 est lu à la résolution de ses mises à jour (quelques minutes). Le coût VARIO est une estimation, valorisée au prix du créneau en cours à chaque lecture.
- Estimation indicative : elle ne remplace pas la facture du fournisseur (taxes, abonnement réel, redevances).

## Désinstallation

Supprime `packages/facture_electricite.yaml`, redémarre, puis supprime le dashboard.
