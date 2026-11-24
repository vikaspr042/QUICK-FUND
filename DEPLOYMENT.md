# QuickFund-X — Cloud-Native Staging & Production Deployment Guide

This guide covers deploying **QuickFund-X** to a cloud staging cluster (**Render**, **AWS ECS / EC2**, **Google Cloud Run**, or **Docker Compose**) for live demonstrations, external viva evaluation, or portfolio hosting.

---

## 1. One-Click Cloud Staging (Render Blueprint)

The repository includes a ready-to-deploy [`render.yaml`](render.yaml) blueprint that provisions:
1. **`quickfund-x-central-intelligence`** (FastAPI + GNN Inference + Interactive Cytoscape Admin Portal)
2. **`quickfund-x-postgres`** (Managed PostgreSQL 16 instance for the persistent fraud graph, incidents, and audit log)
3. **`quickfund-x-redis-broker`** (Managed Redis 7 instance for `XADD`/`XREAD` event streams and Pub/Sub telemetry)

### Steps:
1. Push this repository to GitHub.
2. In the [Render Dashboard](https://dashboard.render.com/blueprints), click **New Blueprint Instance** and connect your repository.
3. Render automatically reads `render.yaml`, builds the Docker container, wires `DATABASE_URL` and `REDIS_URL`, and exposes a public HTTPS URL (e.g., `https://quickfund-x-central-intelligence.onrender.com/dashboard`).
4. Once live, sign in to `/dashboard/login` using the seeded credentials (`admin@quickfund-x.example`) and click **"Simulate Syndicate Attack"** on `/dashboard/graph` to trigger a live 4-platform fraud ring demonstration in the cloud.

---

## 2. Full 11-Container Distributed Deployment (Docker Compose / AWS EC2 / GCP Compute Engine)

To run all **4 independent Lending Platforms** (`QuickFund`, `TrustLend`, `SecureCredit`, `FinBridge`), their **4 isolated MongoDB instances**, the **Redis Streams Event Broker**, **PostgreSQL 16**, and **Central Fraud Intelligence** on an AWS EC2 (`t3.medium`), GCP e2-standard-2 VM, or local machine:

```bash
# Build and start all 11 services in detached mode
docker compose up --build -d

# Verify health of all services
docker compose ps
```

### Exposed Service Endpoints

| Service | Port | URL | Role |
| :--- | :--- | :--- | :--- |
| **Central Fraud Intelligence & Admin Portal** | `8000` | `http://localhost:8000/dashboard` | Interactive Cytoscape Graph, Attack Simulator, 60/40 Explainability Drawer, Model Debugger |
| **Streamlit ML Model Debugger** | `8501` | `http://localhost:8501` | Standalone interactive ML hyperparameter & ROC/confusion matrix laboratory |
| **QuickFund (`loan-platform-a`)** | `4001` | `http://localhost:4001` | Consumer & personal loans platform + transactional outbox + Redis broker publisher |
| **TrustLend (`loan-platform-b`)** | `4002` | `http://localhost:4002` | Education & personal loans platform + transactional outbox + Redis broker publisher |
| **SecureCredit (`loan-platform-c`)** | `4003` | `http://localhost:4003` | Collateralized & personal loans platform + transactional outbox + Redis broker publisher |
| **FinBridge (`loan-platform-d`)** | `4004` | `http://localhost:4004` | Micro-enterprise & bridge loans platform + transactional outbox + Redis broker publisher |
| **Redis Event Broker (`redis-broker`)** | `6379` | `redis://localhost:6379` | Asynchronous `XADD`/`XREAD` event stream (`quickfund:application_events`) + Pub/Sub |
| **Central PostgreSQL (`central-postgres`)** | `5432` | `postgresql://localhost:5432` | Persistent heterogeneous graph store (`graph_nodes`, `graph_edges`), incidents, notifications |

---

## 3. Viva & Live Demo Recovery Commands

During live evaluations or defense presentations, use the root CLI commands to reset state or launch a multi-platform syndicate attack in one command:

```bash
# 1. Wipe all 4 MongoDB databases + Central PostgreSQL + Redis Stream and seed a clean baseline
npm run demo:reset

# 2. Fire 16 synthetic cross-platform syndicate applications over 30 seconds
npm run demo:attack

# 3. Launch the standalone Streamlit ML Hyperparameter Debugger
npm run demo:debugger
```
