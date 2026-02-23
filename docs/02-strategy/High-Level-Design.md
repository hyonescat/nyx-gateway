# NYX GATEWAY - High-Level Design (HLD)

**Version:** 1.0  
**Date:** 2026-02-23  
**Status:** Draft  

---

## 1. Executive Summary

This document defines the high-level architecture for Nyx Gateway, a complete AI agent orchestration platform. The architecture supports 1000+ concurrent agents, 10,000+ messages/minute, and 99.9% uptime SLA.

**Key Architectural Decisions:**
- Microservices architecture with domain-driven design
- Event-driven communication via message broker
- Multi-protocol API (REST, WebSocket, GraphQL)
- Cloud-native deployment on Kubernetes

---

## 2. System Architecture

### 2.1 Component Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                    │
├─────────────┬─────────────┬─────────────┬─────────────────────────────────────┤
│   Web UI    │  Mobile PWA │  CLI Tool   │         SDK Clients                 │
└──────┬──────┴──────┬──────┴──────┬──────┴───────────────┬─────────────────────┘
       │             │             │                      │
       └─────────────┴─────────────┴──────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │    API GATEWAY     │
                    │  (Kong/AWS API GW) │
                    └─────────┬──────────┘
                              │
┌─────────────────────────────┼─────────────────────────────────────────────────┐
│                     SERVICE LAYER (Kubernetes)                                │
│                                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Agent     │  │   Session   │  │    Tool     │  │   Channel   │          │
│  │   Service   │  │   Service   │  │   Service   │  │   Service   │          │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Voice     │  │  Scheduler  │  │  Security   │  │   Config    │          │
│  │    &        │  │   Service   │  │   Service   │  │   Service   │          │
│  │   Vision    │  └─────────────┘  └─────────────┘  └─────────────┘          │
│  │   Service   │  ┌─────────────┐                                             │
│  └─────────────┘  │  Analytics  │                                             │
│                   │   Service   │                                             │
│                   └─────────────┘                                             │
│         │                                                                     │
│         └────────────────┬────────────────────────────────────────────────────┘
│                          │
│              ┌───────────▼────────────┐
│              │      EVENT BUS         │
│              │    (Apache Kafka)      │
│              └───────────┬────────────┘
│                          │
└──────────────────────────┼────────────────────────────────────────────────────┘
                           │
┌──────────────────────────┼────────────────────────────────────────────────────┐
│                     DATA LAYER                                                 │
│                                                                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │ PostgreSQL  │  │    Redis    │  │Elasticsearch│  │  ClickHouse │           │
│  │ (Primary)   │  │   (Cache)   │  │  (Search)   │  │ (Analytics) │           │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘           │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Service Responsibilities

| Service | Responsibility | Key Technologies |
|---------|---------------|------------------|
| **Agent Service** | Agent lifecycle, registry, RPC | PostgreSQL, Redis, gRPC |
| **Session Service** | Conversation management, real-time | WebSocket, Redis Pub/Sub |
| **Tool Service** | Tool registry, sandboxed execution | Firecracker, Kafka |
| **Channel Service** | Multi-channel adapters, routing | Go, Webhooks |
| **Voice & Vision Service** | Voice calls, screen share, desktop vision | WebRTC, WebSocket, AI vision |
| **Scheduler Service** | Cron jobs, workflow orchestration | Temporal.io |
| **Security Service** | Auth, authorization, audit | Keycloak, OPA |
| **Config Service** | Settings, feature flags, secrets | Vault |
| **Analytics Service** | Metrics, tracing, logging | Prometheus, Jaeger |

### 2.3 Voice & Vision Service (NEW)

**Responsibility:** Real-time voice communication, screen sharing, and desktop vision for AI-agent presence

This service enables Nyx Gateway to have a **presence** — not just text, but voice calls and visual awareness of the user's desktop.

#### 2.3.1 Component Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         VOICE & VISION SERVICE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        VOICE GATEWAY                                 │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │    │
│  │  │ WebRTC       │  │ Discord      │  │ Streaming    │              │    │
│  │  │ Handler      │  │ Voice GW     │  │ SFU          │              │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │    │
│  │                                                                      │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │    │
│  │  │ Audio Ingest │  │ Audio Output │  │ Call State   │              │    │
│  │  │ (Opus)       │  │ (Opus)       │  │ Manager      │              │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    REAL-TIME AI PIPELINE                             │    │
│  │                                                                      │    │
│  │   User Voice ──► Whisper STT ──► LLM ──► ElevenLabs TTS ──► Output │    │
│  │       │                                              │               │    │
│  │       │              Latency Target: <500ms          │               │    │
│  │       ▼                                              ▼               │    │
│  │   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐         │    │
│  │   │ Stream  │───►│ Context │───►│ Generate│───►│ Stream  │         │    │
│  │   │ Buffer  │    │ Inject  │    │ Response│    │ Audio   │         │    │
│  │   └─────────┘    └─────────┘    └─────────┘    └─────────┘         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      DESKTOP VISION                                  │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │    │
│  │  │ Screen       │  │ VNC/Remote   │  │ AI Vision    │              │    │
│  │  │ Capture      │  │ Desktop      │  │ Analysis     │              │    │
│  │  │ (WebRTC)     │  │ Protocol     │  │ (GPT-4V)     │              │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │    │
│  │                                                                      │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │    │
│  │  │ Snapshot     │  │ Video Stream │  │ Object       │              │    │
│  │  │ Scheduler    │  │ Ingestion    │  │ Detection    │              │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.3.2 Capabilities

| Feature | Description | Technology |
|---------|-------------|------------|
| **Voice Calls** | Join Discord voice channels, bi-directional audio | WebRTC, Discord Voice Gateway |
| **Screen Share** | Receive user's screen share in real-time | WebRTC screen capture |
| **Desktop Vision** | Periodic screenshots + AI analysis of user's desktop | VNC/SCF, GPT-4V, Claude |
| **Real-time STT** | Streaming speech-to-text with low latency | Whisper Live, Faster-Whisper |
| **Real-time TTS** | Streaming text-to-speech for responses | ElevenLabs Turbo v2.5 |
| **Interruption** | Detect when user speaks over AI, pause/resume | Voice Activity Detection (VAD) |

#### 2.3.3 Voice Call Flow

```
┌─────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  User   │───►│ Discord      │───►│ Voice        │───►│  Whisper     │
│  Speech │    │ Voice Channel│    │ Gateway      │    │  STT Stream  │
└─────────┘    └──────────────┘    └──────────────┘    └──────┬───────┘
                                                              │
                                                              ▼
┌─────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  User   │◄───│  Discord     │◄───│  ElevenLabs  │◄───│    LLM       │
│  Hears  │    │  Voice Out   │    │  TTS Stream  │    │  Response    │
└─────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                              ▲
                                                              │
┌─────────┐    ┌──────────────┐    ┌──────────────┐          │
│ Desktop │───►│  AI Vision   │───►│  Context     │──────────┘
│ Screen  │    │  Analysis    │    │  Injection   │
└─────────┘    └──────────────┘    └──────────────┘
```

**Flow:**
1. User speaks in Discord voice channel
2. Voice Gateway receives audio stream (Opus)
3. Audio streamed to Whisper STT in real-time
4. Transcription sent to LLM with desktop context
5. LLM generates response
6. Response streamed to ElevenLabs TTS
7. Audio sent back to Discord voice channel
8. Desktop screenshots provide visual context (optional)

#### 2.3.4 Desktop Vision Modes

| Mode | Frequency | Use Case |
|------|-----------|----------|
| **Passive** | Every 30s | General awareness, idle monitoring |
| **Active** | Every 5s | During voice calls, focused work |
| **On-Demand** | Triggered | When AI needs to "look" at something |
| **Stream** | Real-time | Screen sharing, pair programming |

#### 2.3.5 API Endpoints

```
# Voice Session Management
POST /api/v1/voice/calls           # Initiate voice call
GET  /api/v1/voice/calls/{id}      # Get call status
POST /api/v1/voice/calls/{id}/join # Join voice channel
POST /api/v1/voice/calls/{id}/leave# Leave voice channel

# Desktop Vision
POST /api/v1/vision/capture        # Capture screenshot
GET  /api/v1/vision/stream         # WebSocket stream
POST /api/v1/vision/analyze        # Analyze current view

# Screen Share (WebRTC)
POST /api/v1/webrtc/offer          # Send SDP offer
POST /api/v1/webrtc/answer         # Receive SDP answer
POST /api/v1/webrtc/ice            # Exchange ICE candidates
```

---

## 3. Data Flow

### 3.1 Message Processing

```
Channel Adapter → Router → Session Service → Agent Runtime
                                                  │
                    ┌─────────────────────────────┼─────────────┐
                    │                             │             │
                    ▼                             ▼             ▼
              Tool Service                   LLM Gateway    Event Bus
```

1. Message received via channel adapter
2. Router determines target session/agent
3. Session Service retrieves context
4. Agent Runtime processes message
5. Tool/LLM calls routed to respective services
6. Events published to Kafka
7. Response sent back through channel

---

## 4. API Design

### 4.1 Protocols

| Protocol | Use Case | Endpoint |
|----------|----------|----------|
| **REST** | CRUD operations | `/api/v1/*` |
| **WebSocket** | Real-time events | `/ws/v1/events` |
| **GraphQL** | Complex queries | `/graphql` |
| **gRPC** | Service-to-service | Internal |

### 4.2 Key Endpoints

**Agents:**
- `GET /api/v1/agents` - List agents
- `POST /api/v1/agents` - Create agent
- `POST /api/v1/agents/{id}/spawn` - Spawn instance

**Sessions:**
- `GET /api/v1/sessions` - List sessions
- `GET /api/v1/sessions/{id}` - Get session
- `DELETE /api/v1/sessions/{id}` - Terminate

**Tools:**
- `GET /api/v1/tools` - List tools
- `POST /api/v1/tools/execute` - Execute tool

---

## 5. Security Model

### 5.1 Authentication
- OAuth2/OIDC for web clients
- API keys for service accounts
- mTLS for service-to-service

### 5.2 Authorization
- RBAC with roles: admin, operator, developer, viewer
- Resource-level permissions
- Approval workflows for dangerous operations

### 5.3 Audit
- All actions logged to ClickHouse
- Immutable audit trail
- 90-day retention minimum

---

## 6. Deployment Strategy

### 6.1 Infrastructure
- **Platform:** Kubernetes (EKS/GKE/AKS)
- **Service Mesh:** Istio or Linkerd
- **Ingress:** Kong or AWS API Gateway
- **GitOps:** ArgoCD or Flux

### 6.2 Deployment Pattern
- Blue-green deployments
- Feature flags for gradual rollout
- Automated rollback on failure
- Zero-downtime updates

### 6.3 Scaling
- Horizontal Pod Autoscaler (HPA)
- Cluster Autoscaler for nodes
- Redis Cluster for cache
- PostgreSQL read replicas

---

## 7. Technology Stack

| Layer | Technology |
|-------|------------|
| **API Gateway** | Kong or AWS API Gateway |
| **Services** | Go, Python |
| **Databases** | PostgreSQL, Redis, ClickHouse |
| **Message Bus** | Apache Kafka |
| **Search** | Elasticsearch |
| **Storage** | MinIO (S3-compatible) |
| **Monitoring** | Prometheus, Grafana, Jaeger |
| **Security** | Keycloak, Vault, OPA |
| **Workflows** | Temporal.io |
| **Frontend** | Lit (Web Components) |

### 7.1 Voice & Vision Stack

| Component | Technology |
|-----------|------------|
| **WebRTC** | Pion (Go) or aiortc (Python) |
| **STT** | OpenAI Whisper Live, Faster-Whisper |
| **TTS** | ElevenLabs Turbo v2.5 |
| **Voice Gateway** | Discord Voice Gateway API |
| **Screen Capture** | WebRTC getDisplayMedia |
| **Desktop Protocol** | VNC, RDP, or SCF |
| **AI Vision** | GPT-4V, Claude 3.5 Sonnet |
| **Audio Codec** | Opus |
| **VAD** | Silero VAD or WebRTC VAD |

---

## 8. Performance Targets

| Metric | Target |
|--------|--------|
| API Response Time (p95) | < 100ms |
| WebSocket Latency | < 50ms |
| Agent Spawn Time (warm) | < 100ms |
| Dashboard Load | < 2s |
| Concurrent Agents | 1000+ |
| Messages/Minute | 10,000+ |
| Uptime SLA | 99.9% |

### 8.1 Voice & Vision Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Voice Latency (end-to-end) | < 500ms | STT + LLM + TTS pipeline |
| First Audio Byte | < 200ms | From speech end to AI voice start |
| STT Accuracy | > 95% | Real-world conditions |
| TTS Quality | Natural | ElevenLabs Turbo v2.5 |
| Screen Share Latency | < 100ms | WebRTC peer-to-peer |
| Vision Analysis Time | < 2s | Screenshot to description |
| Concurrent Voice Calls | 10+ | Per Gateway instance |
| Desktop FPS | 1-30 | Configurable capture rate |

---

## 9. Roadmap

### Phase 1: Core Platform (Months 1-4)
- Agent, Session, Tool, Channel Services
- Dashboard with Matrix theme
- OpenClaw feature parity

### Phase 2: Voice & Vision (Months 5-6) ⭐
- Voice Gateway Service
- Discord voice integration
- Real-time AI audio pipeline
- Desktop vision MVP

### Phase 3: Advanced Features (Months 7-10)
- Operations Center
- Visual workflow builder
- Multi-tenancy
- Advanced observability

### Phase 4: Scale & Polish (Months 11-12)
- Performance optimization
- Security hardening
- Documentation
- Production launch

---

*Next: Detailed component designs and implementation planning*
