# OxeAI API - Quick Architecture Reference

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      CLIENT (Mobile/Web)                     │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTP
                               ▼
                    ┌──────────────────────┐
                    │   web/main.py       │  ← FastAPI App Entry
                    │   (FastAPI Setup)   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  interfaces/http/    │  ← HTTP Endpoints
                    │  routers/*.py        │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  application/        │  ← App Lifecycle
                    │  lifespan.py        │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  domain/            │  ← Business Logic
                    │  ├── agents/        │     (Pure Logic)
                    │  ├── memory/        │
                    │  ├── trackers/      │
                    │  └── ...            │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  infrastructure/     │  ← Technical Details
                    │  ├── database/      │     (MongoDB, LLM)
                    │  ├── functions/     │
                    │  └── llm/           │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  MongoDB / Redis /   │  ← External Services
                    │  Google Gemini       │
                    └─────────────────────┘
```

## 📁 Directory Structure (Simplified)

```
src/
│
├── 🌐 web/                    # Web Server Entry Point
│   └── main.py                # FastAPI app initialization
│
├── 🔌 interfaces/              # External Communication
│   └── http/routers/          # API endpoints
│       ├── prompt.py          # Main chat endpoint
│       ├── sessions.py        # Session management
│       ├── trackers.py        # Health trackers
│       └── ...                # Other endpoints
│
├── 🎯 application/             # Application Orchestration
│   ├── lifespan.py            # Startup/shutdown logic
│   └── models.py              # App state definition
│
├── 💼 domain/                  # Business Logic (Pure)
│   ├── agents/langgraph/      # AI Agent System
│   │   ├── graph.py          # Workflow definition
│   │   ├── agents/           # 7 specialized agents
│   │   └── nodes/            # Router, aggregator
│   │
│   ├── memory/                # Memory Systems
│   │   ├── long_term/        # Long-term memory
│   │   └── semantic/         # Knowledge base
│   │
│   ├── trackers/              # Tracker domain
│   ├── sessions/              # Session domain
│   ├── workouts/              # Workout domain
│   ├── nutrition/             # Nutrition domain
│   └── ...                    # Other domains
│
├── 🔧 infrastructure/          # Technical Implementation
│   ├── database/              # MongoDB connection
│   ├── llm/                   # Gemini API client
│   └── functions/             # AI Agent Tools
│       ├── core.py            # Tool registry
│       ├── workout_coach/     # Workout tools
│       ├── nutrition/         # Nutrition tools
│       └── ...                # Other tools
│
└── 🛠️ shared/                 # Shared Utilities
    ├── config/                # Configuration
    ├── utils/                 # Utilities
    ├── files/                 # File management
    └── usage/                 # Usage tracking
```

## 🔄 Request Flow (Chat Example)

```
1. Client → POST /Sessions/Current/Prompt
   │
   ▼
2. web/main.py (FastAPI)
   │
   ▼
3. interfaces/http/routers/prompt.py
   │
   ▼
4. domain/chat/services.py
   │
   ▼
5. domain/agents/langgraph/
   │   ├── Router → Selects agent
   │   ├── Agent → Processes query
   │   │   └── Calls tools
   │   └── Aggregator → Formats response
   │
   ▼
6. infrastructure/functions/ (Tools execute)
   │
   ▼
7. infrastructure/database/ (MongoDB queries)
   │
   ▼
8. Response flows back up
```

## 🎯 4-Layer Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Layer 4: INTERFACES                                      │
│  How external systems communicate (HTTP, MQ, etc.)       │
│  └── interfaces/http/routers/                            │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 3: APPLICATION                                    │
│  Orchestration & use cases                               │
│  └── application/lifespan.py                             │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 2: DOMAIN                                         │
│  Business logic (NO external dependencies)               │
│  └── domain/ (agents, memory, trackers, etc.)           │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 1: INFRASTRUCTURE                                 │
│  Technical implementation (DB, LLM, APIs)               │
│  └── infrastructure/ (database, llm, functions)         │
└─────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

### Domain-Driven Design (DDD)
- **Domain Layer**: Pure business logic, no external deps
- **Infrastructure Layer**: Implements domain interfaces
- **Interfaces Layer**: External communication
- **Application Layer**: Orchestration

### Dependency Direction
```
Interfaces → Application → Domain ← Infrastructure
```
- Domain defines interfaces
- Infrastructure implements them
- Interfaces/Application use domain

### Repository Pattern
```
Domain:     OxeAITrackerRepository (interface)
            ↓
Infrastructure: MongoOxeAITrackerRepository (implementation)
```

### Dependency Injection
```
app.state.tracker_repository = MongoOxeAITrackerRepository(db)
                              ↓
Router accesses via: app_state.tracker_repository
```

## 📊 Component Relationships

```
┌─────────────┐
│   Router    │  ← HTTP endpoint
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Service   │  ← Business logic
└──────┬──────┘
       │
       ▼
┌─────────────┐      ┌─────────────┐
│  Repository │◄─────┤  MongoDB    │
│ (Interface) │      │ (Database)  │
└─────────────┘      └─────────────┘
       ▲
       │
┌──────┴──────┐
│  Repository │  ← Implementation
│ (MongoDB)   │
└─────────────┘
```

## 🚀 Quick Navigation Guide

**Want to...**
- **Add a new API endpoint?** → `interfaces/http/routers/`
- **Add business logic?** → `domain/[feature]/services.py`
- **Add a database model?** → `domain/[feature]/models.py`
- **Add a new AI tool?** → `infrastructure/functions/[category]/`
- **Add a new agent?** → `domain/agents/langgraph/agents/`
- **Change database queries?** → `infrastructure/database/` or domain `persistence.py` implementations
- **Modify app startup?** → `application/lifespan.py`
- **Change settings?** → `config/models.py`

## 📝 Common File Patterns

Each domain module typically has:
```
domain/[feature]/
├── models.py        # Data models
├── persistence.py   # Repository interface
└── services.py      # Business logic
```

Each router typically:
```python
@router.post("/endpoint")
async def handler(
    request: RequestModel,
    ctx = Depends(get_auth_context)
):
    # Validate → Call service → Return response
```

## 🎓 Learning Path

1. **Start**: `web/main.py` (entry point)
2. **Follow**: `interfaces/http/routers/prompt.py` (chat endpoint)
3. **Understand**: `domain/agents/langgraph/` (AI system)
4. **Explore**: `domain/[feature]/` (business logic)
5. **See**: `infrastructure/` (technical implementation)

---

**See**: `CODEBASE_ARCHITECTURE_DIAGRAM.md` for detailed explanations
