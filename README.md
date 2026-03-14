# vlm-rag-service-robot
### RAG-Based Personalized Home Service Robot

> *"We augment a RAG-based robot service pipeline with a manifold learning layer that converts passively observed behaviour into proactive, personalized service proposals — forming a closed loop between perception, memory, retrieval, and action."*
>
> — M.Sc. Thesis, National Cheng Kung University (Expected Jul 2026)

---

![Python](https://img.shields.io/badge/Python-3.10-3776AB?style=flat-square&logo=python&logoColor=white)
![Unity](https://img.shields.io/badge/Unity-2022-000000?style=flat-square&logo=unity&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-REST_API-000000?style=flat-square&logo=flask&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-Atlas-47A248?style=flat-square&logo=mongodb&logoColor=white)
![FAISS](https://img.shields.io/badge/FAISS-Vector_DB-0467DF?style=flat-square)
![ROS2](https://img.shields.io/badge/ROS2-Humble-22314E?style=flat-square&logo=ros&logoColor=white)
![LLM](https://img.shields.io/badge/LLM-Gemma3_4b-FF6F00?style=flat-square)
![VLM](https://img.shields.io/badge/VLM-LLaVA--phi3-7C3AED?style=flat-square)

> **Repository Status:** Core AI modules are currently private pending thesis defence and publication (Expected Jul 2026). This repository contains the Unity frontend, system architecture overview, and API specifications.

---

## Overview

A home service robot that understands **who you are** and **what you need** — without being manually programmed with rules.

| Mode | Trigger | Mechanism |
|------|---------|-----------|
| **Reactive (RAG)** | User speaks a need | LLM intent analysis → FAISS retrieval → personalized answer + navigation |
| **Proactive (Manifold)** | User behaviour observed silently | Behavioural embedding → manifold trajectory → service proposal |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Unity 3D Frontend                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ UserEntity  │  │StaticCamera  │  │ExperimentRunner│  │
│  │ (NavMesh +  │  │   Manager    │  │  / DemoRunner  │  │
│  │  Animator)  │  │  (Scoring)   │  │                │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬────────┘  │
│         └────────────────┼──────────────────┘           │
│                    POST /predict                         │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  Python / Flask Backend                  │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐   ┌─────────────┐  │
│  │  Perception │    │     RAG     │   │  Manifold   │  │
│  │   Engine    │───▶│   Pipeline  │   │   Engine    │  │
│  │ (VLM:llava) │    │  (8-step)   │   │(UMAP+HDBSCAN│  │
│  └──────┬──────┘    └──────┬──────┘   └──────┬──────┘  │
│         │                 │                  │         │
│  ┌──────▼─────────────────▼──────────────────▼──────┐  │
│  │              Knowledge Base (MongoDB)             │  │
│  │  observation_logs │ dynamic_objects │ scene_snap  │  │
│  └──────────────────────────┬────────────────────────┘  │
│                             │                           │
│  ┌──────────────────────────▼────────────────────────┐  │
│  │           Dual FAISS Vector Index                 │  │
│  │    habit_index (SBERT)  │  dynamic_index (SBERT)  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Core Components

### Perception Engine
Continuously observes the home environment via VLM (LLaVA-phi3). When a user is active in a room, the Unity frontend selects optimal camera viewpoints, renders off-screen images, and posts them to the backend. The perception engine extracts actions, interacting objects, and spatial relations — automatically building the knowledge base without manual annotation.

### RAG Pipeline
When a user expresses a need, an 8-step pipeline runs: LLM intent analysis → dual FAISS retrieval (personal habit memory + real-time object locations) → cross-match scoring → navigation target resolution → personalized LLM answer. The system differentiates between users — the same query from two different people yields different, personalized responses.

### Manifold Engine
Operates silently in the background. Behavioural observations are encoded into high-dimensional feature vectors and projected via UMAP into a 2D manifold. HDBSCAN clusters represent recurring behaviour patterns. Trajectory inertia predicts where a user is heading in behaviour-space — triggering proactive service proposals before the user asks.

### Memory System (Dual FAISS)
Two separate vector indices power retrieval:
- **Habit index** — encodes personal interaction history per user (what they do, near which furniture, with which objects)
- **Dynamic index** — encodes real-time object locations in the home

Both indices are updated continuously as new observations arrive.

---

## Unity Frontend

### Key Scripts

| Script | Responsibility |
|--------|---------------|
| `UserEntity.cs` | NavMesh movement + Animator state machine (Idle / Walking / Sitting / Eating / StandingUp / Nodding) |
| `StaticCameraManager.cs` | Scores CameraNodes by visibility, angle, and distance; decides view count |
| `VirtualCameraBrain.cs` | Off-screen render → Base64 JPEG → POST /predict |
| `ExperimentRunner.cs` | Fully automated experiment pipeline (Exp1 / Exp3A / Exp4 / Exp5) |
| `DemoRunner.cs` | Manual scenario trigger for thesis defence `[1][2][3][4]` |
| `ProactiveServiceManager.cs` | Polls GET /service_proposal every 3s, renders proposal UI |
| `RoomArea.cs` | Collision trigger → snapshot request (Demo / real-sim mode) |

### Demo Scenarios (Thesis Defence)

| Key | Scenario | Demonstrates |
|-----|---------|-------------|
| `[1]` | Mom: "我餓了" → kitchen apple | RAG personalization |
| `[2]` | Dad: "我餓了" → sofa banana | Same query, different user → different result |
| `[3]` | Mom sits silently → robot proactively offers | Manifold proactive prediction |
| `[4]` | Banana removed → robot suggests apple instead | Dynamic environment awareness |
| `[R]` | Reset all to initial state | — |

---

## API Reference

| Route | Direction | Purpose |
|-------|-----------|---------|
| `POST /predict` | Unity → Python | VLM perception + memory update + manifold record |
| `POST /interact` | Unity → Python | RAG pipeline: user query → personalized answer + nav |
| `POST /interact/confirm` | Unity → Python | User nav choice → robot moves |
| `GET /service_proposal` | Unity polls | Pending proactive proposals |
| `POST /service_response` | Unity → Python | accepted / rejected / ignored → manifold feedback |
| `POST /scene` | Unity → Python | Static furniture sync → scene_snapshots |
| `POST /dynamic_sync` | Unity → Python | Sensor object positions → dynamic_objects |

---

## Quick Start

```bash
# 1. Start MongoDB
mongod --dbpath ./data/db

# 2. Pull LLM/VLM models via Ollama
ollama pull llava-phi3
ollama pull gemma3:4b

# 3. Install Python dependencies
pip install flask pymongo sentence-transformers faiss-cpu umap-learn hdbscan scikit-learn

# 4. Run Flask server
python app.py   # serves on :5000

# 5. Unity: Play → ExperimentRunner → Exp4_ManifoldWarmup → Exp5_EndToEnd
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Simulation | Unity 2022, NavMesh, Animator, ReadyPlayerMe, Mixamo |
| Backend | Python 3.10, Flask |
| LLM / VLM | Gemma3:4b (intent + answer generation), LLaVA-phi3 (visual perception) |
| Vector Search | FAISS (dual index), SBERT (sentence-transformers) |
| Manifold Learning | UMAP, HDBSCAN, StandardScaler (scikit-learn) |
| Database | MongoDB (4 collections) |
| Robotics | ROS2 Humble, Linux |

---

## Author

**Hui-Hsin Huang**
M.S. Candidate, Computer Science — National Cheng Kung University
Email: wenny2377@gmail.com

---

*Core AI source code (RAG pipeline, Manifold engine, Perception engine, Memory modules) is private pending thesis defence and publication. Experimental results and technical details will be released upon publication — Expected: August 2026.*
