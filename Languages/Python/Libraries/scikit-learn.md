---
tags: [python, library, sklearn, machine-learning]
---

# scikit-learn

Classical ML, preprocessing and metrics. **The right default for tabular data.**

Index: [[Python libraries]] · Choosing: [[Choosing your approach]] · Deep learning: [[Tensors, autograd and the training loop]]

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add scikit-learn                      # in a uv project - PREFERRED
uv pip install scikit-learn              # into the active venv, no project file
pip install scikit-learn                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add scikit-learn
uv run python -c "import sklearn; print(sklearn.__version__)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install scikit-learn` | `pip install scikit-learn` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import sklearn; print(sklearn.__version__)"
```

> ⚠️ **The package is `scikit-learn`, the import is `sklearn`.** `uv add sklearn` installs a deprecated dummy package that does nothing useful.

```bash
uv add lightgbm xgboost           # the gradient boosting libraries - both ship wheels
```

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

---

## The one API

Every model in sklearn has the same three methods. Learn it once.

```python
model.fit(X_train, y_train)      # learn
model.predict(X_test)            # predict
model.score(X_test, y_test)      # evaluate
```

Transformers add two more:
```python
scaler.fit(X_train)              # learn the parameters
scaler.transform(X)              # apply them
scaler.fit_transform(X_train)    # both, TRAIN ONLY
```

> ⚠️ **`fit_transform` on train, `transform` on everything else.** Fitting the scaler on all your data leaks the test set's mean and standard deviation into training. Quiet, and a real leak ([[Problem framing]]).

## Splitting

```python
from sklearn.model_selection import train_test_split, TimeSeriesSplit, StratifiedKFold

X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42)
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, stratify=y)   # keep class balance
```

> ⚠️ **Never `train_test_split` on time series.** It puts the future in the training set, your score looks brilliant, and the model is worthless. Use `TimeSeriesSplit`, or split by date manually ([[Problem framing]]).

```python
from sklearn.model_selection import TimeSeriesSplit

tscv = TimeSeriesSplit(n_splits=5)
for train_idx, test_idx in tscv.split(X): ...
```

## Preprocessing

```python
from sklearn.preprocessing import LabelEncoder, OneHotEncoder, OrdinalEncoder
from sklearn.preprocessing import (StandardScaler, MinMaxScaler, RobustScaler,
                                   OneHotEncoder, OrdinalEncoder, LabelEncoder)
from sklearn.impute import SimpleImputer, KNNImputer

StandardScaler()          # mean 0, std 1 - the default
MinMaxScaler()            # scale to [0,1]
RobustScaler()            # median/IQR - use when there are outliers

OneHotEncoder(handle_unknown="ignore", sparse_output=False)   # categories -> columns
OrdinalEncoder()          # categories -> integers (ONLY if genuinely ordered)
LabelEncoder()            # for the TARGET only, not features

SimpleImputer(strategy="median")     # mean | median | most_frequent | constant
```

> **`handle_unknown="ignore"` on `OneHotEncoder`.** Without it, a category that appears in production but not in training crashes at inference.

## Pipelines — always use one

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer

numeric = ["recency", "frequency", "monetary"]
categorical = ["country", "segment"]

pre = ColumnTransformer([
    ("num", Pipeline([("impute", SimpleImputer(strategy="median")),
                      ("scale", StandardScaler())]), numeric),
    ("cat", Pipeline([("impute", SimpleImputer(strategy="most_frequent")),
                      ("onehot", OneHotEncoder(handle_unknown="ignore"))]), categorical),
])

model = Pipeline([("pre", pre), ("clf", RandomForestClassifier(random_state=42))])
model.fit(X_train, y_train)
```

> **A Pipeline is not a convenience — it's correctness.** It guarantees the scaler is fitted only on the training fold during cross-validation. Doing it manually is how leakage creeps in.

```python
import joblib
joblib.dump(model, "model.joblib")       # saves the WHOLE pipeline
```

> ⚠️ `joblib.load` executes arbitrary code. Only load artifacts **you** produced ([[26 — SECURITY]]).

## Models

### Regression

```python
from sklearn.linear_model import LinearRegression, Ridge, Lasso, ElasticNet
from sklearn.ensemble import RandomForestRegressor, GradientBoostingRegressor
from sklearn.svm import SVR

LinearRegression()                      # the BASELINE - always run this first
Ridge(alpha=1.0)                        # L2 regularisation
Lasso(alpha=0.1)                        # L1 - drives weights to exactly zero
RandomForestRegressor(n_estimators=100, random_state=42, n_jobs=-1)
```

### Classification

```python
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.naive_bayes import GaussianNB
from sklearn.neighbors import KNeighborsClassifier

LogisticRegression(max_iter=1000, class_weight="balanced")   # BASELINE
RandomForestClassifier(n_estimators=200, n_jobs=-1, random_state=42)
```

> **`class_weight="balanced"` for imbalanced data.** Otherwise the model learns "always predict the majority class" and scores 95% accuracy while being useless.

### Clustering and dimensionality reduction

```python
from sklearn.cluster import KMeans, DBSCAN, AgglomerativeClustering
from sklearn.decomposition import PCA, TruncatedSVD
from sklearn.manifold import TSNE

KMeans(n_clusters=4, random_state=42, n_init="auto")
DBSCAN(eps=0.5, min_samples=5)          # finds clusters of any shape, no k needed
PCA(n_components=0.95)                  # keep 95% of variance
```

> **Scale before KMeans and PCA.** Both use Euclidean distance, so an unscaled feature with a large range dominates everything.

## Gradient boosting — usually the winner on tabular data

```python
import lightgbm as lgb                          # uv add lightgbm

model = lgb.LGBMClassifier(n_estimators=500, learning_rate=0.05, num_leaves=31)
model.fit(X_tr, y_tr, eval_set=[(X_val, y_val)],
          callbacks=[lgb.early_stopping(50), lgb.log_evaluation(100)])
```

> **On tabular data, LightGBM/XGBoost beats a neural network in the large majority of real cases** — faster to train, less tuning, handles missing values and mixed types natively, and tells you feature importance ([[Choosing your approach]]).

## Metrics

```python
from sklearn.metrics import classification_report, confusion_matrix, mean_absolute_error, mean_squared_error, r2_score, roc_auc_score
from sklearn.metrics import (accuracy_score, precision_score, recall_score, f1_score,
                             roc_auc_score, confusion_matrix, classification_report,
                             mean_absolute_error, mean_squared_error, r2_score)

print(classification_report(y_true, y_pred))     # precision/recall/F1 per class
confusion_matrix(y_true, y_pred)
roc_auc_score(y_true, model.predict_proba(X)[:, 1])

mean_absolute_error(y_true, y_pred)              # MAE - interpretable
mean_squared_error(y_true, y_pred) ** 0.5        # RMSE
r2_score(y_true, y_pred)
```

> **Accuracy is misleading with imbalanced classes.** If 95% of rows are one class, predicting it always scores 95%. **Use F1 and look at the confusion matrix** ([[Mathematics reference]]).

## Cross-validation and tuning

```python
from sklearn.model_selection import cross_val_score, GridSearchCV, RandomizedSearchCV

cross_val_score(model, X, y, cv=5, scoring="f1_weighted")

grid = GridSearchCV(model, {"clf__n_estimators": [100, 300],
                            "clf__max_depth": [5, 10, None]},
                    cv=5, scoring="f1_weighted", n_jobs=-1)
grid.fit(X_train, y_train)
grid.best_params_ · grid.best_score_ · grid.best_estimator_
```

> **Note `clf__n_estimators`** — double underscore addresses a step inside a Pipeline.

> **`RandomizedSearchCV` beats `GridSearchCV`** for more than ~3 hyperparameters — it finds comparable results in a fraction of the time.

## Feature importance

```python
model.feature_importances_                        # trees
model.coef_                                       # linear

from sklearn.inspection import permutation_importance
r = permutation_importance(model, X_test, y_test, n_repeats=10, random_state=42)
```

> **Permutation importance is more trustworthy** than built-in tree importances, which are biased toward high-cardinality features.

## When to use sklearn vs the alternatives

| Use | When |
|---|---|
| **sklearn** | Tabular data, classical models, preprocessing pipelines |
| **LightGBM/XGBoost** | Tabular data where accuracy matters most |
| **PyTorch** — [[Tensors, autograd and the training loop]] | Images, text, audio, sequences |
| **Spark MLlib** — [[PySpark reference]] | Data genuinely bigger than one machine |

> **Always run `LogisticRegression` or `LinearRegression` first.** It takes ten seconds and is your baseline. If your complicated model can't beat it, that's the result ([[Problem framing]]).

## Related

[[Python libraries]] · [[08 — MACHINE LEARNING]] · [[Choosing your approach]] · [[Problem framing]] · [[pandas]] · [[NumPy]] · [[MLflow experiment tracking]]
