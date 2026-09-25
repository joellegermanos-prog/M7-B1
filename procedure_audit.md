# Procédure d'audit IA — template 7 sections (MediVox)

> Procédure **fournie** : remplissez chaque section. Un audit **outillé**, pas
> improvisé. Périmètre = observer/documenter/hiérarchiser (≠ corriger, ≠ AIPD).

## 1. Périmètre et hors-périmètre
Ce qui est audité (modèle, code, dataset) ; ce qui est exclu (pen-test, AIPD,
refonte). Les 2 lectorats du rapport (technique / DPO).


## 2. Audit éthique
Variables sensibles (directes/indirectes) ; disparate impact chiffré sur ≥ 1
variable puis investigué (préjudice défini, erreurs par groupe, étiquette vs
réalité) ; RGPD santé (art. 9, minimisation, conservation) ; usage réel du score ;
AI Act (qualification raisonnée art. 6 → obligations si haut risque) ; art. 22
(2 conditions examinées).

### Variables sensibles identifiées

Variables sensibles directes :

- `sexe`
- `age`

Variables indirectes (proxys potentiels) :

- `departement`
- `service`
- `type_admission`
- `nb_comorbidites`

### Disparate Impact

L'analyse met en évidence un écart significatif selon le sexe :

| Mesure | DI F/M |
|----------|----------:|
| Référence basée sur la durée réelle | 1,025 |
| Étiquette historique | 0,653 |
| Prédiction du modèle | 0,292 |

La durée réelle des séjours est similaire entre femmes et hommes (DI ≈ 1), mais les étiquettes historiques présentent déjà un déséquilibre défavorable aux femmes. Le modèle amplifie fortement cet écart.

### Investigation

| Groupe | FNR | Recall |
|----------|----------:|----------:|
| Femmes | 68,00 % | 32 % |
| Hommes | 16,9 % | 83,1 % |

Le modèle détecte beaucoup moins souvent les séjours prolongés chez les femmes.

Par ailleurs :

- 917 incohérences entre l'étiquette historique et la durée réelle ont été identifiées ;
- ces incohérences concernent exclusivement des patientes.

### RGPD

Points à clarifier avec le DPO :

- base légale du traitement (article 9) ;
- politique de minimisation ;
- durée de conservation ;
- information des patients ;
- mesures de sécurité associées aux données de santé.

### Article 22

Le système produit une décision automatiquement à partir d’un seuil de probabilité.

À confirmer :

- existence d’une supervision humaine effective ;
- impact réel du score sur le parcours du patient.

### AI Act

Le niveau de risque ne peut pas être qualifié définitivement à partir du seul code.

La classification dépend notamment :

- de l’usage réel du score ;
- de son influence sur les décisions ;
- d’un éventuel statut de dispositif médical.

## 3. Audit technique
_Architecture (modularité, couplage) ; sécurité (secrets, validation, transport) ;
scalabilité ; **points de rupture** (SPOF)._

Le système repose sur une architecture monolithique composée de scripts Python exécutés localement. Les étapes de préparation des données, d'entraînement et de prédiction sont fortement couplées et ne reposent sur aucun pipeline de traitement structuré.

L'analyse de sécurité met en évidence la présence d'un secret applicatif stocké en clair dans le code (`DB_PASSWORD`), ainsi qu'une absence de validation des données d'entrée et de gestion des erreurs.

Plusieurs points de rupture (SPOF) ont été identifiés : un unique fichier modèle (`dms_predictor_v1.joblib`), un chemin de chargement codé en dur et un déploiement manuel via SCP.

Le système présente également un faible niveau d'industrialisation : absence de logs, de monitoring, de versionnement du modèle et de métadonnées associées.

Enfin, aucune preuve de montée en charge ou de capacité à supporter une augmentation du volume de données n'a été observée. L'architecture actuelle apparaît adaptée à un usage limité mais présente des risques significatifs en matière de sécurité, de maintenabilité, de traçabilité et de continuité de service.


## 4. Audit ressources
_Mesures **psutil** (temps train/inférence, RSS, taille modèle) ; comparaison à
**≤ 2 alternatives** ; lecture sobriété (chiffrée, honnête)._


L'audit des ressources a été réalisé à l'aide de la bibliothèque `psutil`.

Les mesures collectées sont :

- Temps d'entraînement
- Temps d'inférence
- Mémoire RSS consommée
- Taille du modèle sérialisé

Le modèle historique **Random Forest** a été comparé à une alternative plus sobre : **Logistic Regression**.

**Résultats**

| Modèle | Temps entraînement (s) | Temps inférence (s) | Mémoire RSS (MB) | Taille modèle (MB) |
|---------|----------------------:|--------------------:|-----------------:|-------------------:|
| Random Forest | 0,5953 | 0,0713 | 3,5898 | 4,7298 |
| Logistic Regression | 0,0729 | 0,0071 | 0,9766 | 0,0012 |

### Analyse de sobriété

Les mesures mettent en évidence une consommation de ressources significativement plus élevée pour le modèle Random Forest.

Par rapport à la régression logistique :

- le temps d'entraînement est environ **8 fois supérieur** ;
- le temps d'inférence est environ **10 fois supérieur** ;
- la consommation mémoire est environ **3,7 fois supérieure** ;
- la taille du modèle est près de **4 000 fois supérieure**.

La régression logistique constitue donc une alternative beaucoup plus sobre en termes de calcul, de mémoire et de stockage.


Le modèle historique Random Forest présente un coût technique plus important que l'alternative examinée.

Toutefois, cet audit ne vise pas à recommander un remplacement du modèle existant. L'objectif est de documenter les ressources consommées afin d'évaluer le compromis entre performance et coût technique.


## 5. Tableau d'indicateurs consolidé
_12-18 lignes : indicateur / sévérité (🔴🟠🟡) / conséquence client. Hiérarchisé,
pas tout au même niveau._

| Risque | Niveau | Conséquence client |
|----------|----------|----------|
| Biais lié au sexe | 🔴 | Sous-détection des séjours prolongés chez certaines patientes |
| Incohérences entre étiquettes et durée réelle | 🔴 | Mauvaise qualité des données d’apprentissage |
| Absence de validation du modèle | 🔴 | Fiabilité non démontrée des performances |
| Secret stocké en clair | 🔴 | Risque de compromission du système |
| Décision automatisée non documentée | 🔴 | Risque réglementaire (RGPD art. 22) |
| Absence de monitoring et de traçabilité | 🟠 | Difficulté à détecter incidents et dérives |
| Déploiement manuel | 🟠 | Risque opérationnel |
| Fichier modèle unique (SPOF) | 🟠 | Arrêt potentiel du service |
| Architecture monolithique | 🟠 | Complexité de maintenance |
| Coût ressources supérieur à l’alternative | 🟡 | Coût d’exploitation plus élevé |

## 6. Synthèse exécutive
_½ page lisible en 5 min par un décideur non-ML (le « plus grave » d'abord)._

L’audit du prédicteur DMS de MediVox met en évidence un risque éthique majeur ainsi que plusieurs risques techniques et opérationnels susceptibles d’affecter la fiabilité et la conformité du système.

Le principal constat concerne un écart significatif entre les femmes et les hommes. Alors que la durée réelle des séjours est comparable entre les deux groupes (DI = 1,013), les étiquettes historiques montrent déjà un déséquilibre (DI = 0,653) qui est fortement amplifié par le modèle (DI = 0,291). L’analyse révèle également un taux de faux négatifs de 68,00 % chez les femmes contre 16,90 % chez les hommes, ainsi que 917 incohérences entre la durée réelle et les étiquettes historiques concernant exclusivement des patientes. Dans l’hypothèse où le score sert à anticiper les séjours prolongés, les femmes apparaissent comme le groupe le plus exposé au risque de non-détection.

Sur le plan réglementaire, le système traite des données de santé et utilise une variable sensible (le sexe) sans justification documentée. La base légale du traitement, les règles de conservation des données et les modalités d’intervention humaine dans le processus décisionnel ne sont pas documentées et doivent être clarifiées avec le DPO. L’applicabilité de l’article 22 du RGPD ainsi que la qualification du niveau de risque au sens de l’AI Act ne peuvent pas être déterminées définitivement à partir des seuls éléments techniques disponibles.

L’audit technique met en évidence une architecture monolithique faiblement industrialisée. Plusieurs risques critiques ont été identifiés : 
* la présence d’un secret stocké en clair dans le code (`DB_PASSWORD`);
* l’absence de validation des données d’entrée;
* l’absence de protocole robuste d’évaluation du modèle;
* l’absence de logs, de monitoring et de versionnement. 

Plusieurs points de rupture uniques (SPOF) ont également été identifiés, notamment le modèle unique stocké localement et le déploiement manuel.

Enfin, l’audit ressources montre que le modèle Random Forest consomme davantage de ressources qu’une régression logistique de référence : un temps d’entraînement environ 8 fois supérieur, un temps d’inférence environ 10 fois supérieur, une consommation mémoire plus élevée et une taille de modèle significativement plus importante.

Au regard de ces constats, les risques prioritaires concernent:
1. Le biais observé sur la variable sexe;
2. La qualité et la cohérence des données d’apprentissage;
3. L’absence de validation fiable du modèle;
4. Les lacunes de sécurité et de traçabilité.

Ces éléments doivent impérativement être clarifiés avant toute décision d’évolution du système.


## 7. Questions ouvertes
_Ce qu'il faut clarifier avec le client avant toute évolution._

1. Quelle est la durée de conservation des données patients et des modèles historiques ?
2. Quelle est la base légale justifiant l’utilisation des données de santé dans ce système ?
3. Existe-t-il un historique des versions précédentes du modèle ?