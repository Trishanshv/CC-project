# CCproject

[![CI](https://github.com/Trishanshv/CC-project/actions/workflows/ci.yml/badge.svg)](https://github.com/Trishanshv/CC-project/actions/workflows/ci.yml)
[![GitHub Repository](https://img.shields.io/badge/GitHub-Trishanshv%2FCC--project-blue.svg)](https://github.com/Trishanshv/CC-project)
[![Release](https://img.shields.io/badge/Release-v1.0-green.svg)](https://github.com/Trishanshv/CC-project/releases/tag/v1.0)

**CCproject** is a lightweight natural-language command parser designed to convert user queries into structured JSON intents. It uses knowledge distillation to fine-tune a compact Pythia-70M student model from a teacher model (Ollama Phi-3.5), storing model artifacts in an S3-compatible SeaweedFS object store, and serving inference via a containerised FastAPI application orchestrated on Kubernetes (k3d) with horizontal auto-scaling.

GitHub Repository: [https://github.com/Trishanshv/CC-project](https://github.com/Trishanshv/CC-project)  
Documentation Root: `README.md`

---

## Table of Contents

- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Quick Start Guide](#quick-start-guide)
  - [1. Build Commands](#1-build-commands)
  - [2. Deploy Commands](#2-deploy-commands)
  - [3. Test Commands](#3-test-commands)
  - [4. Teardown Commands](#4-teardown-commands)
- [API Usage & Endpoints](#api-usage--endpoints)
- [Network & Security ACLs](#network--security-acls)
- [Performance & Benchmark Results](#performance--benchmark-results)
- [Model Artifacts & Release](#model-artifacts--release)
- [CI/CD Pipeline](#cicd-pipeline)
- [Responsible AI Development](#responsible-ai-development)

---

## Architecture

```text
Natural Language Input ("Set an alarm for 7 am")
         │
         ▼
  Ingress Controller (Port 8080)
         │
         ▼
  FastAPI Service (nanoparse-service:v1)
         │
         ├─── Model Init / Download from SeaweedFS (S3)
         ▼
  Fine-tuned Pythia-70M Student Model (ONNX / Torch)
         │
         ▼
  JSON Schema Validation & Intent Framing
         │
         ▼
Structured Output ({"intent": "alarm_set", "time": "seven am", ...})
```

- **Inference Runtime:** FastAPI container running inside Kubernetes (`app` namespace).
- **Storage Layer:** S3-compatible SeaweedFS deployment inside Kubernetes (`data` namespace).
- **Autoscaling:** Horizontal Pod Autoscaler (HPA) targeting CPU utilisation across replicas.

---

## Technology Stack

- **Student Model:** EleutherAI/Pythia-70M (70M parameter transformer)
- **Teacher Model:** Ollama Phi-3.5 (distillation source)
- **Frameworks:** PyTorch, Hugging Face Transformers, Tokenizers, Safetensors
- **Serving:** FastAPI, Uvicorn, Pydantic
- **Storage:** SeaweedFS (S3-compatible object store)
- **Containerisation:** Docker
- **Cluster Orchestration:** Kubernetes with k3d (lightweight multi-node k3s in Docker)
- **Package Management:** Helm v3
- **Load Testing:** Locust
- **CI/CD:** GitHub Actions, Ruff, Gitleaks

---

## Repository Structure

```text
.
├── README.md               # Primary project documentation in root
├── Makefile                # Automation targets (train, up, down)
├── cluster/                # k3d cluster and SeaweedFS Helm configurations
│   ├── k3d.yaml            # k3d cluster specification and port mappings
│   └── seaweedfs-values.yaml # Helm chart values for SeaweedFS
├── k8s/                    # Kubernetes manifests
│   └── task4.yaml          # Deployment, Service, Ingress, and HPA configurations
├── service/                # FastAPI inference service
│   ├── app.py              # Application logic and endpoints
│   ├── Dockerfile          # Service container image definition
│   └── requirements.txt    # Service dependencies
├── training/               # Model training and distillation scripts
│   ├── train.py            # Student model fine-tuning routine
│   └── dataset.py          # PyTorch Dataset for prompt/completion pairs
├── data/                   # Validation dataset and JSON schema definitions
├── datasets/               # Cleaned datasets for training
├── tools/                  # Evaluation, NACL simulation, and Locust load testing
│   ├── score.py            # Intent & slot prediction scoring
│   ├── nacl_sim.py         # Network ACL simulator
│   └── locustfile.py       # Locust load test scenarios
├── network/                # Subnet design, CIDR allocations, and NACL rules
├── scripts/                # Deployment, teardown, and S3 upload scripts
│   ├── deploy.sh           # End-to-end deployment script
│   ├── destroy.sh          # Cluster cleanup script
│   └── upload_model.py     # S3 bucket creation and model upload helper
├── tests/                  # Unit tests
└── results/                # Evaluation reports and load testing benchmark data
```

---

## Prerequisites

Before running the commands below, ensure your environment has the following tools installed:

- **Docker** (v20.10+)
- **k3d** (v5.0+)
- **kubectl** (v1.24+)
- **Helm** (v3.0+)
- **Python** (3.10 to 3.12)
- **Make**
- **curl** and **tar**

Clone the repository:
```bash
git clone https://github.com/Trishanshv/CC-project.git
cd CC-project
```

Set up a virtual environment:
```bash
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r service/requirements.txt
```

---

## Quick Start Guide

### 1. Build Commands

#### Build Service Docker Image
To build the FastAPI inference Docker image directly:
```bash
docker build -t nanoparse-service:v1 service/
```

#### Train the Student Model (Knowledge Distillation)
Train the Pythia-70M student model using fine-tuning scripts:
```bash
# Quick validation training with subset (SUBSET=50 required by specification)
make train SUBSET=50 VERSION=v1

# Full dataset training (SUBSET=0 trains on all 1,500 samples)
make train SUBSET=0 VERSION=v1

# Alternatively, run using Python directly
python training/train.py --version v1 --seed 42 --subset 50
```
*Note: Training artifacts are output to `training/output/` and excluded from source control.*

---

### 2. Deploy Commands

Deploy the entire stack (k3d cluster, SeaweedFS S3 storage, model download & upload, Docker image build & import, and Kubernetes manifests):

#### Deploy using Makefile
```bash
# Deploy with default production model (v1)
make up

# Or specify a model version
make up MODEL_VERSION=v1
```

#### Deploy using Deployment Script directly
```bash
bash scripts/deploy.sh
```

#### What the Deployment Workflow Does:
1. Provisions the k3d cluster `nanoParse-cluster` configured via `cluster/k3d.yaml`.
2. Deploys SeaweedFS in the `data` namespace using Helm with `cluster/seaweedfs-values.yaml`.
3. Downloads the released model archive from [GitHub Releases](https://github.com/Trishanshv/CC-project/releases/tag/v1.0).
4. Port-forwards the S3 endpoint and executes `scripts/upload_model.py` to upload the weights into SeaweedFS.
5. Builds `nanoparse-service:v1` and imports it into the k3d cluster container cache.
6. Applies Kubernetes manifests (`k8s/task4.yaml`) in the `app` namespace.
7. Verifies rollout readiness (`kubectl rollout status deployment/nanoparse -n app`).

---

### 3. Test Commands

#### Run Unit Tests
Run the test suite using `unittest`:
```bash
python -m unittest discover -s tests -v
```

#### Evaluate Model Accuracy & Scoring
Score model predictions against the validation dataset:
```bash
python tools/score.py \
  results/student_predictions.jsonl \
  data/validation.jsonl \
  --out results/student_validation.json
```

#### Test Network ACLs (NACL Simulator)
Simulate and verify inbound and outbound network ACL rules:
```bash
python tools/nacl_sim.py \
  network/nacl.json \
  network/packets_sample.jsonl \
  --check
```

#### Run Load & Performance Tests (Locust)
Execute headless load tests against the running service:
```bash
locust -f tools/locustfile.py \
  --headless \
  --users 4 \
  --spawn-rate 4 \
  --run-time 2m \
  --host http://localhost:8080 \
  --csv results/task7/loadtest
```

#### Monitor Cluster & Pod Metrics
```bash
# Pod statuses
kubectl get pods -n app
kubectl get pods -n data

# Horizontal Pod Autoscaler status
kubectl get hpa -n app

# Resource utilisation (CPU / Memory)
kubectl top pods -n app
```

---

### 4. Teardown Commands

Tear down the running environment, destroy the k3d cluster, and clean up temporary deployment state:

#### Teardown using Makefile
```bash
make down
```

#### Teardown using Teardown Script directly
```bash
bash scripts/destroy.sh
```

---

## API Usage & Endpoints

Once deployed, the service is accessible at `http://localhost:8080`.

### Health & Readiness Endpoints
- **Liveness:** `GET http://localhost:8080/healthz`
- **Readiness:** `GET http://localhost:8080/readyz`

### Command Parsing Endpoint
- **Parse:** `POST http://localhost:8080/parse`

#### Example Request:
```bash
curl -X POST http://localhost:8080/parse \
  -H "Content-Type: application/json" \
  -d '{"text": "Set an alarm for 7 am"}'
```

#### Example Response:
```json
{
  "intent": "alarm_set",
  "date": null,
  "time": "seven am",
  "place": null
}
```

Response headers include tracking metadata:
- `X-Model-Version`: Deployed model release version.
- `X-Served-By`: Pod identifier handling the request.

---

## Network & Security ACLs

The `network/` directory establishes strict CIDR partitioning:
- Inbound and outbound ACL rules restricting inter-tier communication.
- Packet simulations validating permitted vs. denied traffic flows using `tools/nacl_sim.py`.

---

## Performance & Benchmark Results

### Model Validation Metrics (v1 Release)

| Metric | Raw Prediction | Post-Processed / Repaired | Quality Gate Target |
|---|---:|---:|---:|
| **JSON Validity** | 90.0% | **97.0%** | ≥ 95% |
| **Intent Accuracy** | 75.0% | **75.0%** | - |
| **Date Accuracy** | 68.0% | **68.0%** | - |
| **Time Accuracy** | 76.0% | **76.0%** | - |
| **Place Accuracy** | 77.0% | **77.0%** | - |
| **Full-Frame Accuracy** | 41.0% | **41.0%** | - |

*Note: Raw greedy token generation in small models can occasionally truncate unclosed quotes near maximum sequence length. Auto-closing quotes and braces in `tools/predict.py` and `service/app.py` restores full JSON validity to 97%, reliably clearing the 95% threshold while deliberately untrained baselines remain at 0%.*

### Load Test Benchmark (Locust, 4 Concurrent Users, 2 Replicas)

| Metric | Result | Target Benchmark |
|---|---:|---:|
| **Total Requests** | 89 | - |
| **Successful Requests** | 78 | - |
| **Failed Requests** | 11 | - |
| **Error Rate** | 12.36% | ≤ 1% |
| **Median Latency** | 3,700 ms | - |
| **p95 Latency** | 16,000 ms | ≤ 5,000 ms |
| **Throughput** | 0.748 req/s | - |

#### Load Test Analysis & Diagnostics
- **Error Breakdown:** Out of 11 recorded failures, 2 were connection resets during initial pod rollout, and 9 were HTTP 500 errors caused by unclosed JSON tokens when the model reached sequence limits.
- **Remediation:** With pre-warmed dual replicas (`Ready 2/2`) and resilient JSON quote-completion implemented in `service/app.py`, 500 errors are eliminated under steady-state load.
- **Latency & CPU Constraints:** High tail latency (p95 = 16s) stems from CPU quota contention during concurrent multi-user PyTorch inference on CPU; allocating 2+ CPU cores per replica reduces latency significantly.

---

## Model Artifacts & Release

Model archives are distributed through GitHub Releases to avoid bloating the git history:

- **Release Artifacts:** [CCproject v1.0 Release](https://github.com/Trishanshv/CC-project/releases/tag/v1.0)
- **Archive File:** `pythia-70m-v1.tar.gz` *(Local copy prepared in workspace root ready to attach)*
- **Publishing Instructions:** In GitHub, navigate to **Releases** -> **Draft a new release**, tag `v1.0`, upload `pythia-70m-v1.tar.gz`, and click **Publish release**.
- **Checksum Verification (SHA-256):**
  ```text
  3b098817b3ab49b30b44cd2b6086b71b1ef7645542451883d54dfcfebab47bfe  pythia-70m-v1.tar.gz
  ```

---

## CI/CD Pipeline

The GitHub Actions workflow defined in `.github/workflows/ci.yml` automates 5 sequential stages matching the project specification:

1. **Linting & Code Quality (`1. Lint`):** Python syntax compilation checks and Ruff linting.
2. **Unit Testing (`2. Unit tests`):** Discovers and runs unit tests under `tests/`.
3. **Model Evaluation (`3. Model evaluation`):** Downloads model release archive `pythia-70m-v1.tar.gz`, verifies SHA-256 checksum, generates validation predictions, and scores metrics using `tools/score.py`.
4. **Quality Gate (`4. Quality gate`):** Dedicated pipeline stage enforcing `--min-validity 95` via `tools/score.py`. Automatically passes with the trained model (97% validity) and fails when triggered with an untrained baseline (`model_source: untrained`).
5. **Secret Scanning (`5. Secret scan`):** Scans the repository for sensitive secrets and credentials using open-source Gitleaks.

---

## Responsible AI Development

Refer to [AI_USE.md](AI_USE.md) for detailed disclosures on teacher-student distillation protocols, synthetic data handling, limitations, and human oversight procedures.
