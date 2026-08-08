# 🎬 Recommendation System

> **A production-style, end-to-end recommendation platform combining classical recommender systems, deep retrieval, approximate nearest-neighbor search, learning-to-rank, and hybrid recommendation strategies.**

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat-square\&logo=python\&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-Deep%20Learning-EE4C2C?style=flat-square\&logo=pytorch\&logoColor=white)
![FAISS](https://img.shields.io/badge/FAISS-Vector%20Search-0467DF?style=flat-square)
![FastAPI](https://img.shields.io/badge/FastAPI-Serving-009688?style=flat-square\&logo=fastapi\&logoColor=white)
![Streamlit](https://img.shields.io/badge/Streamlit-Dashboard-FF4B4B?style=flat-square\&logo=streamlit\&logoColor=white)
![MLflow](https://img.shields.io/badge/MLflow-MLOps-0194E2?style=flat-square\&logo=mlflow\&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=flat-square\&logo=docker\&logoColor=white)

---

## 🚀 What Is This?

Modern recommendation engines rarely rely on a single algorithm.

A production system typically combines:

**Candidate Generation → Retrieval → Ranking → Re-ranking → Serving → Monitoring**

This project implements that architecture in a modular Python codebase and brings together multiple recommendation approaches:

* 🔥 Popularity-based recommendations
* 📝 Content-based filtering
* 👥 Collaborative filtering
* 🧮 Matrix factorization
* 🧠 Two-tower neural retrieval
* ⚡ FAISS approximate nearest-neighbor search
* 🏆 Learning-to-rank
* 🔀 Hybrid recommendation
* 🌐 FastAPI inference service
* 📊 Streamlit analytics dashboard
* 📈 MLflow experiment tracking
* 🔍 Monitoring and observability

The goal is to demonstrate how a recommender can evolve from a simple baseline into a **production-oriented recommendation platform**.

---

# 🧠 Recommendation Architecture

```text
                         ┌───────────────────┐
                         │   User / Item     │
                         │      Data         │
                         └─────────┬─────────┘
                                   │
                                   ▼
                       ┌───────────────────────┐
                       │   Data Processing     │
                       │ Cleaning / Features   │
                       └───────────┬───────────┘
                                   │
             ┌─────────────────────┼─────────────────────┐
             │                     │                     │
             ▼                     ▼                     ▼
      ┌─────────────┐      ┌──────────────┐      ┌───────────────┐
      │ Popularity  │      │ Content      │      │ Collaborative │
      │             │      │ Based        │      │ Filtering     │
      └──────┬──────┘      └──────┬───────┘      └───────┬───────┘
             │                    │                      │
             │                    │                      │
             └────────────┬───────┴───────────┬──────────┘
                          │                   │
                          ▼                   ▼
                  ┌──────────────┐    ┌────────────────┐
                  │ Matrix       │    │ Two-Tower      │
                  │ Factorization│    │ Retrieval      │
                  └──────┬───────┘    └───────┬────────┘
                         │                    │
                         │                    ▼
                         │            ┌────────────────┐
                         │            │ User / Item    │
                         │            │ Embeddings     │
                         │            └───────┬────────┘
                         │                    │
                         │                    ▼
                         │            ┌────────────────┐
                         │            │ FAISS ANN      │
                         │            │ Retrieval      │
                         │            └───────┬────────┘
                         │                    │
                         └──────────┬─────────┘
                                    ▼
                          ┌────────────────────┐
                          │ Candidate Fusion   │
                          └─────────┬──────────┘
                                    ▼
                          ┌────────────────────┐
                          │ Learning-to-Rank   │
                          │ LightGBM / XGBoost │
                          └─────────┬──────────┘
                                    ▼
                          ┌────────────────────┐
                          │ Hybrid Recommender │
                          └─────────┬──────────┘
                                    ▼
                    ┌───────────────┴────────────────┐
                    ▼                                ▼
             ┌─────────────┐                  ┌─────────────┐
             │   FastAPI   │                  │ Streamlit   │
             │     API     │                  │  Dashboard  │
             └──────┬──────┘                  └──────┬──────┘
                    │                                │
                    └──────────────┬─────────────────┘
                                   ▼
                         ┌────────────────────┐
                         │ Monitoring / MLflow│
                         │ Metrics / Logging  │
                         └────────────────────┘
```

---

# ⭐ Recommendation Strategies

## 1. 🔥 Popularity-Based

A strong baseline and cold-start fallback.

Supports signals such as:

* Most viewed items
* Most rated items
* Highest-rated items
* Trending items
* Recently popular items

Useful when a user has little or no interaction history.

---

## 2. 📝 Content-Based Filtering

Recommends items based on their attributes and textual similarity.

The implementation supports:

* TF-IDF representations
* Genre similarity
* Keyword similarity
* Cosine similarity
* Sentence-transformer extension

Content signals are configurable through the project configuration.

Example score fusion:

```text
Content Score
     │
     ├── TF-IDF similarity   → 50%
     ├── Genre similarity    → 30%
     └── Keyword similarity  → 20%
```

---

## 3. 👥 Collaborative Filtering

Learns recommendations from user-item interaction patterns.

Implemented approaches include:

* User-user similarity
* Item-item similarity
* Cosine similarity
* Baseline-centered scoring
* Top-K neighborhood recommendation

This allows the system to discover:

> "Users who interacted with similar items also liked these items."

---

## 4. 🧮 Matrix Factorization

The project supports latent-factor recommendation models including:

### SVD

```text
User → Latent User Vector
Item → Latent Item Vector
              ↓
       Interaction Score
```

### ALS

The configuration also supports implicit-feedback matrix factorization through `implicit`.

Optional extensions include:

* LightFM
* Neural Matrix Factorization

---

# 🧠 Two-Tower Retrieval

The two-tower architecture learns separate representations for users and items.

```text
                 USER
                   │
                   ▼
            ┌─────────────┐
            │ User Tower  │
            └──────┬──────┘
                   │
                   ▼
             User Embedding
                   │
                   │
                   │ Similarity
                   │
                   ▼
             Item Embeddings
                   ▲
                   │
            ┌──────┴──────┐
            │ Item Tower  │
            └─────────────┘
                   ▲
                   │
                  ITEM
```

### User features

* User ID
* Interaction statistics
* Genre preferences

### Item features

* Item ID
* Genres
* Release year
* Popularity statistics

The configured architecture uses:

* 64-dimensional embeddings
* Two hidden layers per tower
* ReLU activation
* Batch normalization
* Dropout
* L2-normalized output embeddings
* In-batch softmax loss
* Negative sampling
* Early stopping
* Cosine learning-rate scheduling

---

# ⚡ FAISS Vector Retrieval

Generated item embeddings can be indexed using **FAISS** for fast approximate nearest-neighbor retrieval.

```text
User
 │
 ▼
User Embedding
 │
 ▼
FAISS Index
 │
 ├── Candidate 1
 ├── Candidate 2
 ├── Candidate 3
 ├── ...
 └── Candidate K
```

This creates the retrieval layer required for scaling recommendation systems beyond brute-force similarity calculations.

---

# 🏆 Learning-to-Rank

Retrieved candidates are passed to a ranking model for final scoring.

Ranking features can include:

* User embeddings
* Item embeddings
* Embedding dot product
* Embedding cosine similarity
* User statistics
* Item statistics
* Temporal features
* Popularity features
* Content similarity
* Collaborative-filtering scores
* Matrix-factorization scores
* ANN retrieval scores

Supported ranking approaches include:

* **LightGBM**
* **XGBoost**
* Neural ranking extension

This creates the classic:

```text
Retrieval
   ↓
Hundreds of Candidates
   ↓
Feature Generation
   ↓
Learning-to-Rank
   ↓
Top-K Recommendations
```

---

# 🔀 Hybrid Recommendation

The hybrid layer combines multiple recommendation engines.

Configured strategy:

```text
Popularity              10%
Content-Based           15%
Collaborative           20%
Matrix Factorization    20%
Two-Tower               15%
Ranking                 20%
```

Scores can be normalized before fusion.

The system also includes intelligent fallbacks:

```text
New User
   ↓
Popularity Model

New Item
   ↓
Content-Based Model

Sparse User History
   ↓
Fallback Strategy

Active User
   ↓
Hybrid Recommendation
```

This makes the system more robust to the **cold-start problem**.

---

# 📊 Evaluation

A recommendation system should not be evaluated using classification accuracy alone.

The project is designed around ranking-oriented evaluation, including metrics such as:

* Precision@K
* Recall@K
* Hit Rate@K
* NDCG@K
* MAP@K
* MRR
* Coverage
* Diversity
* Novelty

The evaluation layer is separated from model implementation so different recommenders can be benchmarked consistently.

---

# 🧪 Experiment Tracking

**MLflow** is integrated into the project.

The configuration supports:

```text
Experiment
    ↓
Training Run
    ├── Parameters
    ├── Metrics
    ├── Configuration
    └── Model Artifacts
```

Configured experiment:

```text
recommendation-system
```

This makes it easier to compare experiments across:

* Model architectures
* Hyperparameters
* Datasets
* Retrieval strategies
* Ranking configurations

---

# 📡 Monitoring

The project includes a dedicated monitoring layer designed for production-style observability.

Monitoring capabilities include:

* Recommendation metrics
* System metrics
* Data quality checks
* Drift detection
* API/application logging
* Prometheus-compatible metrics
* Evidently integration

The architecture is designed around:

```text
Model
  │
  ▼
Predictions
  │
  ├── Quality Metrics
  ├── Data Monitoring
  ├── Performance Monitoring
  └── Drift Detection
             │
             ▼
       Monitoring Layer
```

---

# 🌐 FastAPI Serving

The recommendation engine is designed to be exposed through a REST API.

Example workflow:

```text
Client
  │
  │ user_id
  ▼
FastAPI
  │
  ▼
Recommendation Service
  │
  ├── Retrieve candidates
  ├── Generate features
  ├── Rank candidates
  ├── Apply hybrid strategy
  └── Return Top-K
  │
  ▼
JSON Response
```

Run the service with:

```bash
uvicorn api.main:app --reload
```

---

# 📊 Streamlit Dashboard

The project also includes a dashboard layer for interactive exploration.

Potential dashboard workflows include:

* User recommendation demos
* Item similarity exploration
* Model comparison
* Recommendation analysis
* Embedding exploration
* Dataset analytics
* System monitoring

Run:

```bash
streamlit run dashboard/app.py
```

---

# 🗂️ Project Structure

```text
recommendation_system/
│
├── api/
│   └── API application
│
├── configs/
│   ├── config.yaml
│   ├── data.yaml
│   ├── model.yaml
│   ├── api.yaml
│   └── monitoring.yaml
│
├── datasets/
│   ├── raw/
│   ├── processed/
│   └── artifacts/
│       ├── embeddings/
│       ├── encoders/
│       ├── faiss/
│       ├── features/
│       ├── figures/
│       ├── reports/
│       └── splits/
│
├── evaluation/
│
├── experiments/
│
├── models/
│   └── artifacts/
│       ├── popularity/
│       ├── content_based/
│       ├── collaborative/
│       ├── matrix_factorization/
│       ├── two_tower/
│       ├── ranking/
│       └── hybrid/
│
├── monitoring/
│
├── notebooks/
│
├── preprocessing/
│
├── ranking/
│
├── retrieval/
│
├── serving/
│
├── dashboard/
│
├── tests/
│
├── utils/
│
├── logs/
│
├── environment.yml
├── requirements.txt
└── README.md
```

---

# 🛠️ Tech Stack

| Category                | Technologies                  |
| ----------------------- | ----------------------------- |
| Language                | Python                        |
| Data                    | NumPy, Pandas, SciPy, PyArrow |
| Classical ML            | Scikit-learn                  |
| Collaborative Filtering | Surprise, implicit, LightFM   |
| Deep Learning           | PyTorch, TensorFlow           |
| Retrieval               | FAISS                         |
| NLP                     | TF-IDF, Sentence Transformers |
| Ranking                 | LightGBM, XGBoost             |
| API                     | FastAPI, Pydantic             |
| Dashboard               | Streamlit, Plotly             |
| MLOps                   | MLflow, DVC                   |
| Monitoring              | Evidently, Prometheus         |
| Testing                 | Pytest                        |
| Quality                 | Ruff, MyPy                    |
| Deployment              | Docker                        |

---

# ⚙️ Configuration-Driven Design

Instead of hard-coding model parameters throughout the project, the system centralizes configuration.

```text
configs/
│
├── config.yaml
├── data.yaml
├── model.yaml
├── api.yaml
└── monitoring.yaml
```

Model settings include:

* Embedding dimensions
* Learning rates
* Batch sizes
* Number of epochs
* Negative sampling
* Ranking parameters
* Hybrid weights
* Cold-start rules
* Artifact locations
* MLflow settings

This makes experimentation significantly easier.

---

# 🚀 Getting Started

## 1. Clone

```bash
git clone <your-repository-url>
cd recommendation_system
```

## 2. Create Environment

### Conda

```bash
conda env create -f environment.yml
conda activate recommendation-system
```

### Python Virtual Environment

```bash
python -m venv .venv
```

Windows:

```bash
.venv\Scripts\activate
```

Linux/macOS:

```bash
source .venv/bin/activate
```

## 3. Install Dependencies

```bash
pip install -r requirements.txt
```

---

# 🧪 Development Workflow

A typical development workflow looks like:

```text
1. Prepare Dataset
       ↓
2. Preprocess Data
       ↓
3. Generate Features
       ↓
4. Train Baseline Models
       ↓
5. Train Retrieval Models
       ↓
6. Build FAISS Index
       ↓
7. Generate Candidates
       ↓
8. Train Ranking Model
       ↓
9. Build Hybrid Recommender
       ↓
10. Evaluate
       ↓
11. Track with MLflow
       ↓
12. Serve through API
       ↓
13. Monitor in Production
```

---

# 📦 Model Artifacts

The repository is organized to keep model artifacts separated by recommendation strategy:

```text
models/artifacts/

├── popularity/
├── content_based/
├── collaborative/
├── matrix_factorization/
├── two_tower/
├── ranking/
└── hybrid/
```

Examples include:

```text
content_based/
├── tfidf_vectorizer.joblib
├── item_content_vectors.npz
└── content_similarity.npz

two_tower/
├── two_tower.pt
├── user_id_map.json
└── item_id_map.json

matrix_factorization/
├── svd_model.joblib
└── als_model.joblib
```

---

# 🐳 Docker

The project is structured for containerized deployment.

Example:

```bash
docker build -t recommendation-system .
```

Then:

```bash
docker run -p 8000:8000 recommendation-system
```

For a production deployment, the API, dashboard, model artifacts, monitoring stack, and supporting services can be separated into individual containers.

---

# 🧪 Testing & Code Quality

Testing is included as part of the engineering workflow.

Run:

```bash
pytest
```

With coverage:

```bash
pytest --cov=.
```

Linting:

```bash
ruff check .
```

Type checking:

```bash
mypy .
```

The goal is to keep the recommendation pipeline maintainable as additional algorithms and datasets are introduced.

---

# 📈 Scalability Design

The project follows a scalable recommendation architecture rather than relying exclusively on brute-force item scoring.

### Candidate Generation

Efficiently narrow a large item catalog to a manageable candidate set.

### ANN Retrieval

FAISS enables approximate vector search over item embeddings.

### Ranking

A dedicated ranking layer scores the retrieved candidates.

### Hybrid Fusion

Multiple independent recommendation signals can be combined.

```text
1,000,000+ Items
       │
       ▼
Candidate Retrieval
       │
       ▼
~1,000 Candidates
       │
       ▼
Feature Generation
       │
       ▼
Ranking Model
       │
       ▼
Top 10 / Top 20
```

This separation allows retrieval and ranking to evolve independently.

---

# 🔮 Roadmap

* [ ] Add online feature store
* [ ] Add real-time user-event ingestion
* [ ] Add Redis caching
* [ ] Add Kafka event pipeline
* [ ] Add model registry promotion workflow
* [ ] Add automated retraining
* [ ] Add A/B testing framework
* [ ] Add contextual bandits
* [ ] Add session-based recommendations
* [ ] Add real-time personalization
* [ ] Add GPU-accelerated ANN retrieval
* [ ] Add distributed training
* [ ] Add Kubernetes deployment
* [ ] Add recommendation explanations
* [ ] Add automated recommendation-quality monitoring

---

# 💡 What This Project Demonstrates

This repository is intentionally broader than a traditional recommendation-system notebook.

It demonstrates practical skills across the complete ML lifecycle:

```text
                DATA
                 │
                 ▼
        Feature Engineering
                 │
                 ▼
       Classical Recommenders
                 │
                 ▼
        Deep Representation
                 │
                 ▼
       Candidate Generation
                 │
                 ▼
        ANN Vector Search
                 │
                 ▼
          Learning-to-Rank
                 │
                 ▼
       Hybrid Recommendation
                 │
                 ▼
             Serving
                 │
                 ▼
          Monitoring
                 │
                 ▼
              MLOps
```

### Core ML Skills

* Recommendation algorithms
* Representation learning
* Embeddings
* Similarity search
* Collaborative filtering
* Matrix factorization
* Learning-to-rank
* Hybrid modeling

### ML Engineering Skills

* Modular architecture
* Configuration management
* Model artifacts
* API serving
* Experiment tracking
* Monitoring
* Testing
* Containerization

### Production Concepts

* Candidate generation
* Retrieval/ranking separation
* Cold-start handling
* ANN search
* Model versioning
* Observability
* Scalable inference

---

# 🎯 Project Highlights

| Area                | Implementation                 |
| ------------------- | ------------------------------ |
| Recommendation      | Multi-model architecture       |
| Retrieval           | Two-Tower + FAISS              |
| Ranking             | LightGBM / XGBoost             |
| Personalization     | Collaborative + latent factors |
| Content             | TF-IDF + metadata similarity   |
| Cold Start          | Popularity + content fallbacks |
| Hybridization       | Weighted score fusion          |
| Serving             | FastAPI                        |
| Visualization       | Streamlit                      |
| Experiment Tracking | MLflow                         |
| Data Versioning     | DVC-ready                      |
| Monitoring          | Evidently + Prometheus         |
| Testing             | Pytest                         |
| Deployment          | Docker                         |

---

## 👨‍💻 About

This project was built as an **ML Engineering portfolio project** to demonstrate the design of a modern recommendation platform from data processing through model training, retrieval, ranking, serving, and monitoring.

It is intended for **learning, experimentation, research, and portfolio demonstration**.

---

## ⭐ Support

If you find this project useful or interesting, consider giving the repository a ⭐ on GitHub.
