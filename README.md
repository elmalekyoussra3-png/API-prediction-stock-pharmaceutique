# 💊 API de Prédiction de Ventes \& Recommandation de Stock Pharmaceutique

Ce projet prédit la demande journalière de 8 catégories de médicaments à partir de l'historique des ventes, et recommande un réapprovisionnement lorsque le stock actuel risque d'être insuffisant. Le modèle est exposé via une API REST construite avec **FastAPI**.

## 📋 Sommaire

* [Aperçu du projet](#-aperçu-du-projet)
* [Architecture \& démarche](#-architecture--démarche)
* [Structure du dépôt](#-structure-du-dépôt)
* [Configuration de l'environnement](#-configuration-de-lenvironnement)
* [Lancer le projet](#-lancer-le-projet)
* [Endpoints de l'API](#-endpoints-de-lapi)
* [Exemples d'utilisation](#-exemples-dutilisation)
* [Résultats](#-résultats)
* [Auteurs](#-auteurs)

## 🔎 Aperçu du projet

Le jeu de données contient les ventes journalières (2014 et au-delà) de 8 catégories de médicaments (`M01AB`, `M01AE`, `N02BA`, `N02BE`, `N05B`, `N05C`, `R03`, `R06`), ainsi que le niveau de stock courant pour chacune.

Pour chaque médicament, deux modèles sont entraînés et comparés :

* **Random Forest Regressor**
* **XGBoost Regressor**

Le meilleur modèle (selon le R²) est conservé pour les prédictions. L'API expose ensuite 4 endpoints permettant de prédire la demande future, de comparer cette demande au stock actuel, et de recommander un réapprovisionnement si nécessaire.

## 🏗 Architecture \& démarche

1. **Nettoyage \& exploration des données** (`final\_salesdaily\_cleaned.csv`)
2. **Feature engineering** :

   * Encodage cyclique du temps (jour de semaine, mois, semaine de l'année via sin/cos)
   * Variables calendaires (week-end, début/fin de mois, trimestre)
   * Lags (J-1, J-7, J-14, J-30) et moyennes mobiles (7, 14, 30 jours)
   * Écart-type glissant et différenciation (J vs J-1, J vs J-7)
3. **Entraînement** : un Random Forest et un XGBoost par médicament, split temporel train/test (80/20)
4. **Sélection du meilleur modèle** par médicament (comparaison du R²)
5. **Prédiction récursive** : pour une date future au-delà des données disponibles, le modèle reconstruit les features pas à pas, jour après jour
6. **Sérialisation** des modèles avec `joblib`
7. **API FastAPI** exposant les prédictions et recommandations, testée via un tunnel `ngrok` depuis Google Colab

## 📁 Structure du dépôt

```
.
├── projet\_prediction\_stock.ipynb   # Notebook complet (EDA, feature engineering, entraînement, API)
├── README.md                       # Documentation (ce fichier)
├── requirements.txt                # Dépendances Python
├── .gitignore                      # Fichiers à exclure du dépôt
└── data/
    └── final\_salesdaily\_cleaned.csv  # Jeu de données (à ajouter, voir ci-dessous)
```

> ℹ️ Les modèles entraînés (`.pkl`) et le zip d'export ne sont pas versionnés sur GitHub (voir `.gitignore`) car ce sont des fichiers binaires volumineux et reproductibles en ré-exécutant le notebook. Voir \[Lancer le projet](#-lancer-le-projet) pour les régénérer.

## ⚙️ Configuration de l'environnement

### Option 1 — Google Colab (recommandé, identique à l'environnement de développement original)

1. Ouvrez [Google Colab](https://colab.research.google.com/)
2. Importez `projet\_prediction\_stock.ipynb` (`Fichier > Importer un notebook`)
3. Importez le fichier de données `final\_salesdaily\_cleaned.csv` dans l'environnement Colab (panneau de gauche > icône dossier > glisser-déposer)
4. Exécutez les cellules dans l'ordre

### Option 2 — Environnement local

**Prérequis** : Python 3.10+

```bash
# 1. Cloner le dépôt
git clone https://github.com/<votre-nom-utilisateur>/<nom-du-repo>.git
cd <nom-du-repo>

# 2. Créer un environnement virtuel
python -m venv venv
source venv/bin/activate      # Sur Windows : venv\\Scripts\\activate

# 3. Installer les dépendances
pip install -r requirements.txt
```

### 🔐 Variable d'environnement pour ngrok

L'API utilise [ngrok](https://ngrok.com/) pour exposer le serveur local publiquement (utile en environnement Colab). Le token **ne doit jamais être codé en dur dans le code**. Créez un compte gratuit sur ngrok.com, récupérez votre auth token, puis définissez-le comme variable d'environnement :

```bash
# Linux / macOS
export NGROK\_AUTH\_TOKEN="votre\_token\_ici"

# Windows (PowerShell)
$env:NGROK\_AUTH\_TOKEN="votre\_token\_ici"
```

En local (hors Colab), l'utilisation de ngrok n'est pas nécessaire : vous pouvez lancer l'API directement avec `uvicorn` (voir ci-dessous) et y accéder via `http://localhost:8000`.

## ▶️ Lancer le projet

### Dans le notebook (Colab ou Jupyter)

Exécutez les cellules dans l'ordre : chargement des données → feature engineering → entraînement des modèles → sauvegarde → lancement de l'API.

### En local, sans Colab

Après avoir exécuté les cellules d'entraînement et sauvegardé les modèles (`joblib.dump`), vous pouvez lancer l'API directement :

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

Puis ouvrez `http://localhost:8000/docs` pour la documentation interactive Swagger.

## 🔌 Endpoints de l'API

|Méthode|Endpoint|Description|
|-|-|-|
|`GET`|`/`|Page d'accueil, liste des endpoints disponibles|
|`GET`|`/predict?medicament=N05C\&date=2027-08-15`|Prédit la demande pour un médicament à une date donnée|
|`GET`|`/recommandation?medicament=N05C\&date=2027-08-15`|Compare la prédiction au stock actuel et recommande un réapprovisionnement|
|`GET`|`/predict-most-demanded?date=2027-08-15`|Prédit le médicament le plus demandé à une date donnée|
|`GET`|`/stock`|Retourne les niveaux de stock actuels de tous les médicaments|

Médicaments disponibles : `M01AB`, `M01AE`, `N02BA`, `N02BE`, `N05B`, `N05C`, `R03`, `R06`

## 💡 Exemples d'utilisation

```bash
curl "http://localhost:8000/predict?medicament=N05C\&date=2027-08-15"
# {"medicament": "N05C", "date": "2027-08-15", "valeur\_predite": 0.06}

curl "http://localhost:8000/recommandation?medicament=N05C\&date=2027-08-15"
# {"medicament": "N05C", "date": "2027-08-15", "stock\_actuel": 177.0, "valeur\_predite": 0.06, "statut": "STOCK SUFFISANT"}

curl "http://localhost:8000/predict-most-demanded?date=2027-08-15"
# {"date": "2027-08-15", "medicament\_plus\_demande": "N02BE", "valeur\_predite": 10.17, ...}
```

## 📊 Résultats

Les performances (MAE, MAPE, R²) de chaque modèle par médicament sont calculées et comparées dans le notebook (cellules d'évaluation). Le meilleur modèle entre Random Forest et XGBoost est sélectionné automatiquement par médicament selon le R² obtenu sur l'ensemble de test.

## 👤 Auteurs

* El Malek Youssra
* Wahi Hassna

\---

\*\*Encadrement :\*\*

\- Encadrant : Abdelhak Mahmoudi

\- Co-encadrant : Saad Frihi



\---

*Projet réalisé dans le cadre d'un projet académique de Machine Learning / Data Science.*

