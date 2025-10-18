# Chapitre 28 : ML Pipelines & Feature Engineering — scikit-learn, Spark MLlib

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Feature engineering** pour ML
- **scikit-learn pipelines**
- **Spark MLlib** distributed ML
- **Preprocessing** et validation
- **Model deployment** patterns

---

## 📖 Table des matières

1. [Feature Engineering](#feature-engineering)
2. [scikit-learn Pipelines](#scikit-learn-pipelines)
3. [Spark MLlib](#spark-mllib)
4. [Preprocessing & Validation](#preprocessing--validation)
5. [Model Serving](#model-serving)

---

## Feature Engineering

### Numerical features

```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler, MinMaxScaler, PolynomialFeatures

# Load data
df = pd.read_csv('customers.csv')

# Standardization (mean=0, std=1)
scaler = StandardScaler()
df['age_scaled'] = scaler.fit_transform(df[['age']])

# Normalization (0-1 range)
normalizer = MinMaxScaler()
df['salary_norm'] = normalizer.fit_transform(df[['salary']])

# Polynomial features
poly = PolynomialFeatures(degree=2)
age_poly = poly.fit_transform(df[['age']])
df['age_squared'] = age_poly[:, 1]

# Binning
df['age_bin'] = pd.cut(df['age'], bins=[0, 25, 45, 65, 100], labels=['young', 'adult', 'senior', 'retired'])
```

### Categorical features

```python
from sklearn.preprocessing import LabelEncoder, OneHotEncoder

# Label encoding (for ordinal)
le = LabelEncoder()
df['status_encoded'] = le.fit_transform(df['status'])  # ['high', 'low', 'med'] -> [1, 0, 2]

# One-hot encoding (for nominal)
df_encoded = pd.get_dummies(df, columns=['country'], drop_first=True)

# Target encoding (mean encoding)
target_encoding = df.groupby('category')['target'].mean()
df['category_target_enc'] = df['category'].map(target_encoding)
```

### Text features

```python
from sklearn.feature_extraction.text import TfidfVectorizer, CountVectorizer

# TF-IDF
vectorizer = TfidfVectorizer(max_features=100, ngram_range=(1, 2))
tfidf_features = vectorizer.fit_transform(df['description'])

# Word embeddings
from gensim.models import Word2Vec

sentences = [text.split() for text in df['description']]
model = Word2Vec(sentences, vector_size=100)
```

### Time-based features

```python
df['date'] = pd.to_datetime(df['date'])
df['year'] = df['date'].dt.year
df['month'] = df['date'].dt.month
df['day_of_week'] = df['date'].dt.dayofweek
df['is_weekend'] = df['day_of_week'].isin([5, 6]).astype(int)
df['days_since'] = (pd.Timestamp.now() - df['date']).dt.days
```

---

## scikit-learn Pipelines

### Build pipeline

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split, cross_val_score
import pandas as pd

# Load and prepare
df = pd.read_csv('data.csv')
X = df.drop('target', axis=1)
y = df['target']

# Create pipeline
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('pca', PCA(n_components=10)),
    ('classifier', LogisticRegression(max_iter=1000))
])

# Train/test split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

# Train
pipeline.fit(X_train, y_train)

# Evaluate
score = pipeline.score(X_test, y_test)
cv_scores = cross_val_score(pipeline, X, y, cv=5)
print(f"Test accuracy: {score:.3f}")
print(f"CV scores: {cv_scores.mean():.3f} (+/- {cv_scores.std():.3f})")
```

### Custom transformers

```python
from sklearn.base import BaseEstimator, TransformerMixin

class CustomFeatureTransformer(BaseEstimator, TransformerMixin):
    def __init__(self, columns=None):
        self.columns = columns
    
    def fit(self, X, y=None):
        return self
    
    def transform(self, X):
        X_copy = X.copy()
        # Add custom features
        X_copy['feature_1'] = X_copy['col1'] * X_copy['col2']
        X_copy['feature_2'] = X_copy['col1'] ** 2
        return X_copy

# Use in pipeline
pipeline = Pipeline([
    ('custom', CustomFeatureTransformer()),
    ('scaler', StandardScaler()),
    ('model', LogisticRegression())
])
```

---

## Spark MLlib

### Spark ML Pipeline

```python
from pyspark.sql import SparkSession
from pyspark.ml import Pipeline
from pyspark.ml.feature import StringIndexer, OneHotEncoder, VectorAssembler
from pyspark.ml.classification import LogisticRegression
from pyspark.ml.evaluation import BinaryClassificationEvaluator

spark = SparkSession.builder.appName("MLPipeline").getOrCreate()

# Load data
df = spark.read.csv("data.csv", header=True, inferSchema=True)

# Define stages
stages = [
    # Index categorical column
    StringIndexer(inputCol="category", outputCol="category_idx"),
    
    # One-hot encode
    OneHotEncoder(inputCol="category_idx", outputCol="category_encoded"),
    
    # Assemble features
    VectorAssembler(
        inputCols=["age", "salary", "category_encoded"],
        outputCol="features"
    ),
    
    # Train model
    LogisticRegression(maxIter=10, labelCol="target")
]

# Create pipeline
pipeline = Pipeline(stages=stages)

# Split data
train, test = df.randomSplit([0.7, 0.3])

# Train
model = pipeline.fit(train)

# Predict
predictions = model.transform(test)

# Evaluate
evaluator = BinaryClassificationEvaluator(labelCol="target")
accuracy = evaluator.evaluate(predictions)
print(f"AUC: {accuracy:.3f}")
```

---

## Preprocessing & Validation

### Train/test split strategies

```python
from sklearn.model_selection import StratifiedKFold, TimeSeriesSplit

# Stratified (for imbalanced data)
skf = StratifiedKFold(n_splits=5)
for train_idx, test_idx in skf.split(X, y):
    X_train, X_test = X.iloc[train_idx], X.iloc[test_idx]
    y_train, y_test = y.iloc[train_idx], y.iloc[test_idx]

# Time series (preserve order)
tscv = TimeSeriesSplit(n_splits=5)
for train_idx, test_idx in tscv.split(X):
    X_train, X_test = X.iloc[train_idx], X.iloc[test_idx]
    y_train, y_test = y.iloc[train_idx], y.iloc[test_idx]
```

### Handle missing values

```python
from sklearn.impute import SimpleImputer

# Numeric imputation
imputer = SimpleImputer(strategy='mean')
X_imputed = imputer.fit_transform(X)

# Categorical imputation
cat_imputer = SimpleImputer(strategy='most_frequent')
X_cat = cat_imputer.fit_transform(X_categorical)
```

### Feature selection

```python
from sklearn.feature_selection import SelectKBest, f_classif, RFE
from sklearn.ensemble import RandomForestClassifier

# SelectKBest
selector = SelectKBest(f_classif, k=10)
X_selected = selector.fit_transform(X, y)

# RFE (Recursive)
rf = RandomForestClassifier()
rfe = RFE(rf, n_features_to_select=10)
X_rfe = rfe.fit_transform(X, y)
```

---

## Model Serving

### Save and load models

```python
import joblib
import pickle

# Save scikit-learn model
joblib.dump(pipeline, 'model.pkl')
loaded_model = joblib.load('model.pkl')

# Spark ML
model.save("spark_model")
loaded_spark_model = PipelineModel.load("spark_model")
```

### REST API for predictions

```python
from flask import Flask, request, jsonify
import joblib
import pandas as pd

app = Flask(__name__)
model = joblib.load('model.pkl')

@app.route('/predict', methods=['POST'])
def predict():
    data = request.json
    df = pd.DataFrame([data])
    prediction = model.predict(df)[0]
    probability = model.predict_proba(df)[0]
    
    return jsonify({
        'prediction': int(prediction),
        'probability': probability.tolist()
    })

if __name__ == '__main__':
    app.run(port=5000)
```

---

## 🎓 Exercices pratiques

### Exercice 28.1 : Feature engineering
Créez 10 features from raw data.

### Exercice 28.2 : Pipeline
Construisez scikit-learn pipeline end-to-end.

### Exercice 28.3 : Spark MLlib
Entraînez modèle Spark sur 1GB data.

### Exercice 28.4 : Validation
Comparez stratégies cross-validation.

### Exercice 28.5 : API
Servez modèle via Flask REST API.

---

## 📚 Références

- **scikit-learn** : https://scikit-learn.org/
- **Spark MLlib** : https://spark.apache.org/mllib/
- **Feature Engineering** : https://www.deeplearningbook.org/

---

**Prêt pour NoSQL? → [Chapitre 29 : Graph & NoSQL](./29_Graph_NoSQL.md)**
