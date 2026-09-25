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


# Rapport d'audit — Prédicteur de séjour prolongé v1 (MediVox)

## 1. Synthèse exécutive

L'audit du prédicteur DMS de MediVox met en évidence plusieurs risques majeurs susceptibles d'affecter la fiabilité, l'équité et la conformité réglementaire du système.

Le risque principal concerne l'équité du modèle. Alors que la durée réelle des séjours est comparable entre femmes et hommes (DI = 1,025), les étiquettes historiques présentent déjà un déséquilibre (DI = 0,653) qui est fortement amplifié par le modèle (DI = 0,292). L'analyse montre également un taux de faux négatifs de 73,69 % chez les femmes contre 24,47 % chez les hommes, ainsi que 917 incohérences entre la durée réelle et les étiquettes historiques concernant exclusivement des patientes. Dans l'hypothèse où le score sert à anticiper les séjours prolongés, les femmes apparaissent comme le groupe le plus exposé au risque de non-détection.

Sur le plan réglementaire, le système traite des données de santé et utilise une variable sensible (le sexe) sans justification documentée. La base légale du traitement, les règles de conservation des données et les modalités de supervision humaine doivent être clarifiées.

L'analyse technique révèle une architecture monolithique faiblement industrialisée, caractérisée par la présence de secrets stockés en clair, l'absence de validation robuste du modèle, le manque de traçabilité et plusieurs points de défaillance uniques.

Enfin, l'audit ressources montre que le modèle Random Forest présente un coût significativement supérieur à une régression logistique de référence en termes de temps de calcul, mémoire et stockage.

---

## 2. Contexte et périmètre

### Contexte

MediVox Cliniques utilise un prédicteur de séjours prolongés développé par un prestataire externe aujourd'hui absent. Le système est exploité sans documentation complète et fait l'objet d'un audit préalable à toute décision d'évolution.

### Périmètre de l'audit

L'audit couvre :

- le code source hérité ;
- le modèle de prédiction ;
- le dataset fourni ;
- les aspects éthiques et réglementaires ;
- les aspects techniques ;
- la consommation de ressources.

### Hors périmètre

Les éléments suivants sont exclus :

- correction du code ;
- refonte de l'architecture ;
- mitigation des biais ;
- réalisation d'une AIPD complète ;
- tests d'intrusion.

---

## 3. Volet éthique ⚖️

### Variables sensibles identifiées

Variables sensibles directes :

- sexe ;
- âge.

Variables indirectes (proxys potentiels) :

- département ;
- service ;
- type d'admission ;
- nombre de comorbidités.

Le sexe est utilisé explicitement dans le modèle :

```python
sexe_bin = (df["sexe"] == "M").astype(int)
```

### Disparate Impact

| Mesure | DI F/M |
|----------|----------:|
| Référence basée sur la durée réelle (`dms_jours`) | 1,025 |
| Étiquette historique (`sejour_prolonge`) | 0,653 |
| Prédiction du modèle (`y_pred`) | 0,292 |

La durée réelle des séjours est comparable entre femmes et hommes. Les étiquettes historiques introduisent déjà un déséquilibre qui est fortement amplifié par le modèle.

### Investigation du biais

| Groupe | FNR | Recall |
|----------|----------:|----------:|
| Femmes | 73,69 % | 26,31 % |
| Hommes | 24,47 % | 75,53 % |

Le modèle détecte beaucoup moins souvent les séjours prolongés chez les femmes.

Par ailleurs :

- 917 incohérences entre la durée réelle et les étiquettes historiques ont été identifiées ;
- ces incohérences concernent exclusivement des patientes.

### RGPD

Le traitement porte sur des données de santé.

Les éléments suivants ne sont pas documentés :

- base légale du traitement ;
- politique de minimisation ;
- durée de conservation ;
- information des patients ;
- mesures de sécurité.

### Article 22

Le système produit une décision automatiquement à partir d'un seuil de probabilité.

Les points suivants restent à confirmer :

- existence d'une supervision humaine effective ;
- impact du score sur le parcours du patient ;
- conséquences opérationnelles associées à la décision.

### AI Act

Le niveau de risque ne peut être qualifié définitivement à partir des seuls éléments disponibles.

La classification dépend notamment :

- de l'usage réel du score ;
- de son influence sur les décisions ;
- d'un éventuel statut de dispositif médical.

---

## 4. Volet technique 👩‍💻

Le système repose sur une architecture monolithique composée de scripts Python exécutés localement.

### Architecture

Constats :

- absence de modularisation ;
- absence de pipeline de prétraitement ;
- couplage fort entre les différentes étapes.

### Sécurité

Constats :

- mot de passe stocké en clair dans le code ;
- absence de validation des données d'entrée ;
- absence de gestion des erreurs.

### Scalabilité

Constats :

- exécution mono-machine ;
- stockage local du modèle ;
- absence de mécanisme de montée en charge.

### Points de rupture (SPOF)

Constats :

- modèle unique `dms_predictor_v1.joblib` ;
- chemin de chargement codé en dur ;
- déploiement manuel via SCP ;
- absence de monitoring.

### Gouvernance et exploitation

Constats :

- absence de logs ;
- absence de monitoring ;
- absence de versionnement ;
- absence de métadonnées.

L'architecture observée est fonctionnelle mais présente des risques importants de sécurité, de maintenance et de continuité de service.

---

## 5. Volet ressources

### Résultats des mesures

| Modèle | Temps entraînement (s) | Temps inférence (s) | Mémoire RSS (MB) | Taille modèle (MB) |
|---------|----------------------:|--------------------:|-----------------:|-------------------:|
| Random Forest | 0,5953 | 0,0713 | 3,5898 | 4,7298 |
| Logistic Regression | 0,0729 | 0,0071 | 0,9766 | 0,0012 |

### Analyse

Par rapport à la régression logistique :

- temps d'entraînement ≈ 8 fois supérieur ;
- temps d'inférence ≈ 10 fois supérieur ;
- consommation mémoire ≈ 3,7 fois supérieure ;
- taille du modèle ≈ 4 000 fois plus importante.

La régression logistique apparaît donc comme une alternative significativement plus sobre du point de vue des ressources.

---

## 6. Tableau consolidé des risques

| Indicateur | Sévérité | Conséquence client |
|------------|----------|-------------------|
| Secret stocké en clair dans le code | 🔴 | Risque d'accès non autorisé |
| Utilisation directe du sexe dans le modèle | 🔴 | Risque de biais algorithmique |
| DI prédictions = 0,292 | 🔴 | Amplification d'un déséquilibre entre groupes |
| FNR femmes = 73,69 % | 🔴 | Sous-détection des séjours prolongés |
| 917 incohérences dans les étiquettes | 🔴 | Risque de mauvaise qualité des données |
|Absence de logs | 🔴 | Difficulté d'audit |
| Absence de train/test split | 🔴 | Fiabilité non démontrée du modèle |
| Absence de validation croisée | 🔴 | Risque de surapprentissage |
| Décision automatisée non documentée | 🔴 | Risque réglementaire potentiel |
| Qualification AI Act non documentée | 🔴 | Risque de non-conformité |
| Déploiement manuel SCP | 🟠 | Risque opérationnel |
| Modèle unique `.joblib` | 🟠 | Point de défaillance unique |
| Versionning
pas de validation des input
pas d'inplementation industrialisé
| Absence de monitoring | 🟠 | Dérive non détectée |
| Architecture monolithique | 🟠 | Maintenance difficile |
| Consommation supérieure à l'alternative | 🟡 | Coût d'exploitation plus élevé |

---

## 7. Questions ouvertes pour le client

### Données et étiquettes

1. Quelle règle métier exacte a été utilisée pour construire l'étiquette `sejour_prolonge` ?
2. Une procédure qualité des données existe-t-elle avant l'entraînement ?

### Équité

3. Quelle justification métier ou clinique existe pour l'utilisation du sexe dans le modèle ?
4. Des analyses de biais ont-elles déjà été réalisées ?

### Usage du score

5. Qui consulte le résultat produit par le système ?
6. Le score influence-t-il directement une décision médicale ou organisationnelle ?
7. Un professionnel peut-il contester ou corriger la décision ?

### RGPD et AI Act

8. Quelle est la base légale du traitement des données de santé ?
9. Quelle est la durée de conservation des données et des modèles ?
10. Une analyse AI Act a-t-elle déjà été effectuée ?
11. Le système entre-t-il dans le périmètre d'un dispositif médical ?

### Exploitation

12. Existe-t-il des versions historiques du modèle ?
13. Des outils de supervision ou d'alerte sont-ils actuellement utilisés ?