# Rapport d'audit — prédicteur de séjour prolongé v1 (MediVox) — À COMPLÉTER

> 2 lectorats : 👩‍💻 Hélène (technique) · ⚖️ Marc (DPO). Renomme en rapport_audit.md.

## 1. Synthèse exécutive
_½ page, le « plus grave » d'abord, lisible en 5 min._
## 2. Contexte et périmètre
## 3. Volet éthique ⚖️
## 4. Volet technique 👩‍💻
## 5. Volet ressources
## 6. Tableau consolidé des risques
## 7. Questions ouvertes pour le client


# Rapport d'audit – Prédicteur de séjour prolongé v1 (MediVox)

---

# 1. Synthèse exécutive

L’audit réalisé sur le prédicteur DMS de MediVox met en évidence trois risques majeurs susceptibles d’affecter la fiabilité opérationnelle, l’équité et la conformité du système.

Le risque principal concerne l’équité du modèle. L’analyse du sexe montre un Disparate Impact de **0,653** sur les étiquettes historiques et de **0,291** sur les prédictions du modèle, alors que la référence construite à partir de la durée réelle des séjours présente un DI de **1,013**. Les femmes sont beaucoup moins souvent identifiées comme présentant un séjour prolongé alors que la fréquence réelle observée est comparable à celle des hommes. Le taux de faux négatifs atteint **68 % chez les femmes contre 16,9 % chez les hommes**. Sous l’hypothèse que le score sert à anticiper les séjours prolongés, les femmes apparaissent comme le groupe le plus exposé au risque de non-détection.

L’audit technique met en évidence une architecture monolithique peu industrialisée. Plusieurs risques critiques ont été identifiés : secret de production stocké en clair, absence de validation robuste du modèle, manque de traçabilité, absence de monitoring et présence de plusieurs points de défaillance uniques.

L’audit des ressources montre que le modèle historique Random Forest est plus coûteux qu’une régression logistique tout en obtenant des performances inférieures sur le jeu de test utilisé. La régression logistique obtient le meilleur ROC-AUC (**0,736**) tout en étant significativement plus rapide à entraîner et plus légère à stocker.

---

# 2. Contexte et périmètre

## Contexte

MediVox Cliniques exploite un système de prédiction des séjours prolongés développé par un prestataire externe aujourd’hui absent. Le système est utilisé en environnement de santé et ne dispose plus de documentation complète.

## Objectif

Identifier les risques :

- éthiques ;
- réglementaires ;
- techniques ;
- opérationnels ;
- liés à la consommation de ressources.

## Périmètre

L’audit couvre :

- le code source hérité ;
- le modèle de prédiction ;
- le dataset fourni ;
- les pratiques de développement observées ;
- la consommation de ressources.

## Hors périmètre

L’audit n’inclut pas :

- la correction du code ;
- la mitigation des biais ;
- la refonte de l’architecture ;
- la réalisation d’une AIPD ;
- les tests d’intrusion.

---

# 3. Volet éthique ⚖️

## Variables sensibles identifiées

### Variables sensibles directes

- sexe ;
- âge.

### Variables indirectes (proxys potentiels)

- département ;
- service ;
- type d’admission ;
- nombre de comorbidités.

## Analyse du biais selon le sexe

### Disparate Impact

| Mesure | DI F/M |
|----------|----------:|
| Référence DMS ≥ 7 jours | 1,013 |
| Étiquette historique | 0,653 |
| Prédiction du modèle | 0,291 |

La durée réelle observée est comparable entre femmes et hommes. Les étiquettes historiques introduisent un écart qui est ensuite amplifié par le modèle.

### Investigation

| Groupe | Recall | FNR | FPR |
|----------|----------:|----------:|----------:|
| Femmes | 32,0 % | 68,0 % | 6,7 % |
| Hommes | 83,1 % | 16,9 % | 34,5 % |

Le modèle détecte beaucoup moins souvent les séjours prolongés chez les femmes.

### Analyse selon l’âge

L’analyse des classes d’âge montre une augmentation progressive du taux de séjours prolongés et des prédictions positives avec l’âge.

Les patients de moins de 40 ans présentent un DI de 0,150 par rapport aux patients de 75 ans et plus. Cet écart reflète cependant en partie une différence réelle observée dans les durées de séjour.

## RGPD

Le système traite des données de santé.

Les éléments suivants ne sont pas documentés :

- base légale du traitement ;
- durée de conservation ;
- politique de minimisation ;
- information des patients ;
- mesures de sécurité associées aux données.

## Article 22

Le système produit une décision automatisée à partir d’un seuil de probabilité.

À confirmer :

- existence d’une intervention humaine effective ;
- impact concret du score sur le parcours du patient.

## AI Act

La qualification du niveau de risque ne peut être déterminée à partir du seul code.

Elle dépend notamment :

- de l’usage réel du score ;
- de son influence sur la prise de décision ;
- d’un éventuel statut de dispositif médical.

---

# 4. Volet technique 👩‍💻

## Architecture

Le système repose sur une architecture monolithique :

```text
Dataset CSV
    ↓
Prétraitement manuel
    ↓
Random Forest
    ↓
Fichier .joblib
    ↓
Script de prédiction
```

## Constats

- absence de pipeline de prétraitement ;
- absence de modularisation ;
- absence de validation des données d’entrée ;
- absence de gestion des erreurs ;
- absence de journalisation ;
- absence de monitoring ;
- absence de versionnement du modèle.

## Sécurité

Le mot de passe de production est stocké directement dans le code :

```python
DB_PASSWORD = "medivox_prod_2024"
```

## SPOF identifiés

- modèle unique `.joblib` ;
- chemin de chargement codé en dur ;
- déploiement manuel via SCP ;
- absence de mécanisme de reprise.

---

# 5. Volet ressources

## Comparaison du modèle historique avec deux alternatives

| Modèle | Accuracy | Balanced Accuracy | ROC-AUC | Temps entraînement (ms) | Temps inférence (ms) | Taille modèle (MB) |
|---------|---------:|---------:|---------:|---------:|---------:|---------:|
| Régression logistique | **0,691** | **0,660** | **0,736** | **10,97** | **1,71** | **0,0013** |
| HistGradientBoosting | 0,678 | 0,647 | 0,720 | 755,78 | 10,29 | 0,3431 |
| Random Forest (historique) | 0,676 | 0,643 | 0,720 | 489,17 | 19,46 | 4,3748 |

## Analyse

La régression logistique :

- obtient les meilleures performances ;
- est la plus rapide à entraîner ;
- est la plus rapide en inférence ;
- est la plus légère en stockage.

Le modèle historique Random Forest est le plus coûteux des trois sans bénéfice de performance observable sur le jeu de test utilisé.

---

# 6. Tableau consolidé des risques

| Indicateur | Sévérité | Conséquence client |
|------------|----------|-------------------|
| Biais observé sur le sexe (DI prédictions = 0,291) | 🔴 | Sous-détection des séjours prolongés chez certaines patientes |
| FNR femmes = 68 % | 🔴 | Mauvaise anticipation des besoins hospitaliers |
| Secret stocké en clair | 🔴 | Risque d’accès non autorisé |
| Absence de validation du modèle | 🔴 | Fiabilité non démontrée |
| Décision automatisée non documentée | 🔴 | Risque réglementaire |
| Absence de monitoring | 🟠 | Dérive non détectée |
| Absence de journaux d’audit | 🟠 | Difficulté d’investigation |
| Déploiement manuel | 🟠 | Risque opérationnel |
| SPOF sur le modèle unique | 🟠 | Indisponibilité potentielle du service |
| Architecture monolithique | 🟠 | Maintenance difficile |
| Consommation supérieure à l’alternative | 🟡 | Coût d’exploitation accru |

---

# 7. Questions ouvertes pour le client

## Données et étiquettes

1. Quelle règle métier a été utilisée pour construire l’étiquette `sejour_prolonge` ?
2. Comment expliquer le déséquilibre observé entre les femmes et les hommes alors que la fréquence réelle des séjours prolongés est comparable ?

## Usage métier

3. Qui consulte le résultat du système ?
4. Le score influence-t-il directement une décision clinique ou organisationnelle ?
5. Existe-t-il une possibilité de contestation humaine ?

## RGPD

6. Quelle est la base légale autorisant le traitement des données de santé ?
7. Quelle est la durée de conservation du dataset et des modèles ?

## AI Act

8. Le système est-il considéré comme un dispositif médical ou intégré à un dispositif médical ?
9. Une qualification AI Act a-t-elle déjà été réalisée ?

## Exploitation

10. Existe-t-il un historique des versions du modèle ?
11. Quels incidents ont déjà été observés en production ?
12. Des outils de supervision ou d’alerte sont-ils actuellement déployés ?