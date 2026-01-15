# OxeAI API - Codebase Architecture Diagram & Explanation

## Table of Contents
1. [High-Level Architecture Overview](#high-level-architecture-overview)
2. [4-Layer DDD Structure](#4-layer-ddd-structure)
3. [Request Flow Diagram](#request-flow-diagram)
4. [Directory Structure Map](#directory-structure-map)
5. [Component Interactions](#component-interactions)
6. [Data Flow](#data-flow)

---

## High-Level Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT APPLICATIONS                          │
│              (Mobile Apps, Web Browsers, API Clients)                │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTP Requests
                               │ (REST API)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         WEB LAYER (FastAPI)                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  src/web/main.py                                             │  │
│  │  - FastAPI app initialization                                │  │
│  │  - Middleware (CORS, Logging)                                │  │
│  │  - Router registration                                       │  │
│  │  - Error handling                                            │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    INTERFACES LAYER (HTTP Routers)                   │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐        │
│  │ Sessions │  Files   │ Journal  │ Trackers │  Goals   │        │
│  │ Surveys  │  Plans   │  Meals   │  Prompt  │ LangGraph │        │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘        │
│                                                                      │
│  Purpose: HTTP endpoint definitions, request/response handling       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER (Orchestration)                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  src/application/lifespan.py                                │  │
│  │  - App startup/shutdown                                      │  │
│  │  - Database connection initialization                        │  │
│  │  - Repository/service initialization                         │  │
│  │  - Dependency injection setup                                │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  src/application/models.py                                  │  │
│  │  - AppState protocol definition                             │  │
│  │  - Type definitions for app state                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    DOMAIN LAYER (Business Logic)                     │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  AGENTS SYSTEM (LangGraph)                                   │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │  src/domain/agents/langgraph/                         │  │  │
│  │  │  ├── graph.py          (Workflow definition)           │  │  │
│  │  │  ├── state.py          (Graph state models)           │  │  │
│  │  │  ├── chat_service.py   (Main entry point)              │  │  │
│  │  │  ├── agents/           (7 specialized agents)          │  │  │
│  │  │  │   ├── companion.py                                 │  │  │
│  │  │  │   ├── workout_coach.py                             │  │  │
│  │  │  │   ├── workout_plan.py                              │  │  │
│  │  │  │   ├── nutrition.py                                 │  │  │
│  │  │  │   ├── sleep.py                                      │  │  │
│  │  │  │   ├── mental_wellbeing.py                          │  │  │
│  │  │  │   └── physical_health.py                           │  │  │
│  │  │  └── nodes/            (Router, Aggregator)             │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                                                               │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │  MEMORY SYSTEM                                         │  │  │
│  │  │  src/domain/memory/                                   │  │  │
│  │  │  ├── long_term/        (Long-term memory with TTL)    │  │  │
│  │  │  │   ├── facade.py     (Unified API)                  │  │  │
│  │  │  │   ├── services/     (Search, Extract, Hit Track)   │  │  │
│  │  │  │   └── jobs/          (Daily extraction job)          │  │  │
│  │  │  └── semantic/         (General knowledge base)        │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                                                               │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │  DOMAIN MODULES (Business Entities)                    │  │  │
│  │  │  ├── conversations/    (Chat conversations)            │  │  │
│  │  │  ├── sessions/         (User sessions)                 │  │  │
│  │  │  ├── profiles/        (User profiles)                 │  │  │
│  │  │  ├── workouts/         (Workout coach & plans)         │  │  │
│  │  │  ├── nutrition/        (Nutrition tracking)            │  │  │
│  │  │  ├── meals/            (Meal logging)                 │  │  │
│  │  │  ├── journal/         (Journal entries)               │  │  │
│  │  │  ├── trackers/        (Health trackers)               │  │  │
│  │  │  ├── goals/            (User goals)                   │  │  │
│  │  │  ├── plans/            (Workout/nutrition plans)      │  │  │
│  │  │  ├── surveys/          (User surveys)                 │  │  │
│  │  │  ├── mental_wellbeing/ (Mental health)                │  │  │
│  │  │  └── physical_health/  (Health symptoms)              │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  Each domain module typically contains:                              │
│  - models.py      (Domain models/entities)                          │
│  - persistence.py (Repository interfaces)                           │
│  - services.py    (Business logic services)                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│              INFRASTRUCTURE LAYER (Technical Implementation)         │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  DATABASE                                                     │  │
│  │  src/infrastructure/database/                                │  │
│  │  └── db.py          (MongoDB connection manager)             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  LLM INFRASTRUCTURE                                           │  │
│  │  src/infrastructure/llm/                                     │  │
│  │  └── gemini.py      (Google Gemini client)                   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  TOOL/FUNCTION SYSTEM                                         │  │
│  │  src/infrastructure/functions/                               │  │
│  │  ├── core.py         (Function host & registry)              │  │
│  │  ├── registration.py  (Tool registration factory)             │  │
│  │  ├── profile/        (Profile tools: get/update biometrics)  │  │
│  │  ├── workout_coach/  (Workout logging tools)                 │  │
│  │  ├── workout_plan/   (Workout plan tools)                    │  │
│  │  ├── nutrition/     (Nutrition tools)                        │  │
│  │  ├── mental_wellbeing/ (Mental health tools)                 │  │
│  │  ├── physical_health/ (Health tracking tools)                │  │
│  │  ├── companion/     (General companion tools)                 │  │
│  │  ├── memory/        (Memory search tools)                    │  │
│  │  └── misc/          (Utility tools: time, etc.)              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  AUTHENTICATION                                              │  │
│  │  src/infrastructure/auth/                                    │  │
│  │  └── authorization.py  (Auth middleware)                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    SHARED MODULES (Cross-cutting)                    │
│                                                                      │
│  ┌──────────────┬──────────────┬──────────────┬──────────────┐    │
│  │   config/    │  constants/  │   shared/    │    utils/    │    │
│  │              │              │              │              │    │
│  │ Settings     │ Time zones   │ Shared       │ Logging      │    │
│  │ Environment  │ Constants    │ models       │ Serialization│    │
│  │ variables    │              │ Types        │ UUID utils   │    │
│  └──────────────┴──────────────┴──────────────┴──────────────┘    │
│                                                                      │
│  ┌──────────────┬──────────────┬──────────────┐                   │
│  │   files/     │   usage/     │    demo/     │                   │
│  │              │              │              │                   │
│  │ File upload  │ Usage        │ Demo data    │                   │
│  │ management   │ tracking     │ & scripts    │                   │
│  │              │ Pricing      │              │                   │
│  └──────────────┴──────────────┴──────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SERVICES & DATABASES                    │
│                                                                      │
│  ┌──────────────┬──────────────┬──────────────┐                   │
│  │   MongoDB    │    Redis     │ Google Gemini│                   │
│  │   (Atlas)    │   (Cache)    │   (LLM API)  │                   │
│  │              │              │              │                   │
│  │ - Conversations│ - Session   │ - Chat       │                   │
│  │ - Sessions    │   cache     │   completion │                   │
│  │ - Memories    │ - Rate      │ - Embeddings │                   │
│  │ - Profiles    │   limiting  │              │                   │
│  │ - Workouts    │              │              │                   │
│  │ - etc.        │              │              │                   │
│  └──────────────┴──────────────┴──────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4-Layer DDD Structure

### Layer 1: Domain Layer (Business Logic)
**Location**: `src/domain/`

**Purpose**: Contains pure business logic with NO external dependencies (no database, no HTTP, no external APIs).

**Key Principles**:
- Defines **what** the business does, not **how** it's implemented
- Contains domain models, business rules, and domain services
- Should be testable without any infrastructure

**Structure**:
```
domain/
├── agents/              # AI agent system (LangGraph)
│   └── langgraph/       # Single-agent with multi-topic routing
│       ├── graph.py     # Workflow definition
│       ├── state.py     # State models
│       ├── agents/      # 7 specialized agents
│       └── nodes/       # Router, aggregator nodes
│
├── memory/              # Memory systems
│   ├── long_term/       # Long-term memory with TTL
│   └── semantic/        # Knowledge base
│
├── conversations/        # Conversation domain
├── sessions/            # Session management
├── profiles/            # User profiles
├── workouts/            # Workout domain
├── nutrition/           # Nutrition domain
├── meals/               # Meal tracking
├── journal/             # Journal entries
├── trackers/            # Health trackers
├── goals/               # User goals
├── plans/               # Plans (workout/nutrition)
├── surveys/             # Surveys
├── mental_wellbeing/    # Mental health
└── physical_health/    # Physical health tracking
```

**Example Domain Module Structure**:
```
domain/trackers/
├── models.py           # Tracker domain models (pure Python classes)
├── persistence.py      # Repository INTERFACE (abstract, no implementation)
└── services.py         # Business logic (e.g., tracker validation, calculations)
```

---

### Layer 2: Infrastructure Layer (Technical Implementation)
**Location**: `src/infrastructure/`

**Purpose**: Implements technical concerns - database access, external APIs, file storage, etc.

**Key Principles**:
- Implements interfaces defined in Domain layer
- Handles all technical details (MongoDB queries, HTTP clients, etc.)
- Can be swapped out without changing domain logic

**Structure**:
```
infrastructure/
├── database/
│   └── db.py           # MongoDB connection manager (OxeAIDb)
│
├── llm/
│   └── gemini.py       # Google Gemini API client
│
├── functions/          # Tool/function system (AI agent tools)
│   ├── core.py         # Function host & registry
│   ├── registration.py # Tool registration
│   ├── profile/        # Profile tools
│   ├── workout_coach/  # Workout tools
│   ├── nutrition/      # Nutrition tools
│   └── ...             # Other tool categories
│
└── auth/
    └── authorization.py # Authentication/authorization
```

**Example**: 
- Domain defines `OxeAITrackerRepository` (interface)
- Infrastructure implements `MongoOxeAITrackerRepository` (MongoDB-specific)

---

### Layer 3: Interfaces Layer (External Communication)
**Location**: `src/interfaces/`

**Purpose**: Defines how external systems communicate with the application (HTTP API, message queues, etc.).

**Structure**:
```
interfaces/
├── http/
│   └── routers/        # FastAPI route handlers
│       ├── sessions.py      # Session endpoints
│       ├── prompt.py         # Main chat endpoint
│       ├── files.py          # File upload endpoints
│       ├── journal.py        # Journal endpoints
│       ├── trackers.py       # Tracker endpoints
│       ├── goals.py          # Goal endpoints
│       ├── plans.py          # Plan endpoints
│       ├── surveys.py        # Survey endpoints
│       ├── meals.py          # Meal endpoints
│       ├── langgraph_chat.py # LangGraph chat endpoint
│       └── memory_debug.py   # Memory testing endpoints
│
└── dependencies/
    └── dependencies.py  # FastAPI dependency injection
```

**Responsibilities**:
- Parse HTTP requests
- Validate input
- Call domain services
- Format HTTP responses
- Handle HTTP-specific errors

---

### Layer 4: Application Layer (Orchestration)
**Location**: `src/application/`

**Purpose**: Coordinates domain services and use cases. Handles application lifecycle.

**Structure**:
```
application/
├── lifespan.py    # App startup/shutdown logic
│                  # - Database connection setup
│                  # - Repository initialization
│                  # - Service initialization
│                  # - Dependency injection
│
└── models.py      # App state protocol definition
                   # - Defines what's available in app.state
```

**Key Responsibilities**:
- Application startup (connect to DB, initialize services)
- Application shutdown (cleanup, close connections)
- Dependency injection setup
- Use case orchestration

---

## Request Flow Diagram

### Example: User sends a chat message

```
1. CLIENT REQUEST
   POST /Sessions/Current/Prompt
   {
     "user_message": "I just did 3x8 bench press at 135 lbs",
     "session_id": "...",
     "user_id": "..."
   }
   │
   ▼
   
2. WEB LAYER (src/web/main.py)
   - FastAPI receives request
   - CORS middleware processes
   - Logging middleware logs request
   - Routes to appropriate router
   │
   ▼
   
3. INTERFACES LAYER (src/interfaces/http/routers/prompt.py)
   - Router handler receives request
   - Validates input (Pydantic models)
   - Extracts user_id from auth context
   - Calls domain service
   │
   ▼
   
4. APPLICATION LAYER (orchestration)
   - Gets dependencies from app.state
   - Passes to domain service
   │
   ▼
   
5. DOMAIN LAYER - CHAT SERVICE
   src/domain/chat/services.py
   - Creates LangGraph chat service
   - Prepares conversation context
   │
   ▼
   
6. DOMAIN LAYER - LANGGRAPH SYSTEM
   src/domain/agents/langgraph/
   │
   ├─► graph.py (Workflow)
   │   │
   │   ├─► nodes/router.py
   │   │   - Analyzes user message
   │   │   - Selects appropriate agent (WorkoutCoachAgent)
   │   │
   │   ├─► agents/workout_coach.py
   │   │   - Gets user's workout session
   │   │   - Calls tools to log exercise
   │   │   - Generates response
   │   │
   │   └─► nodes/aggregator.py
   │       - Formats final response
   │
   ▼
   
7. INFRASTRUCTURE LAYER - TOOLS
   src/infrastructure/functions/workout_coach/
   - update_workout_session tool executes
   - Calls domain repository
   │
   ▼
   
8. INFRASTRUCTURE LAYER - DATABASE
   src/infrastructure/database/
   - MongoOxeAITrackerRepository
   - Executes MongoDB query
   - Saves workout data
   │
   ▼
   
9. RESPONSE FLOW (back up)
   Database → Repository → Tool → Agent → Aggregator → 
   Chat Service → Router → FastAPI → Client
   
   Response:
   {
     "response": "Excellent work! Logged 3 sets of bench press...",
     "agent_type": "workout_coach",
     "topic": "workout"
   }
```

---

## Directory Structure Map

### Complete `src/` Directory Tree

```
src/
│
├── application/              # Layer 4: Application orchestration
│   ├── lifespan.py           # App startup/shutdown
│   └── models.py             # App state protocol
│
├── config/                   # Shared: Configuration
│   ├── models.py             # Settings models
│   └── sources.py           # Config sources
│
├── constants/                # Shared: Constants
│   ├── persistence.py       # DB constants
│   └── time.py              # Time zone constants
│
├── domain/                   # Layer 1: Business logic
│   ├── agents/               # AI agent system
│   │   └── langgraph/       # LangGraph implementation
│   │       ├── graph.py      # Workflow
│   │       ├── state.py      # State models
│   │       ├── chat_service.py
│   │       ├── agents/       # 7 specialized agents
│   │       └── nodes/        # Router, aggregator
│   │
│   ├── memory/               # Memory systems
│   │   ├── long_term/        # Long-term memory
│   │   └── semantic/         # Knowledge base
│   │
│   ├── conversations/         # Conversation domain
│   ├── sessions/             # Session domain
│   ├── profiles/             # Profile domain
│   ├── workouts/             # Workout domain
│   ├── nutrition/            # Nutrition domain
│   ├── meals/                # Meal domain
│   ├── journal/              # Journal domain
│   ├── trackers/             # Tracker domain
│   ├── goals/                # Goal domain
│   ├── plans/                # Plan domain
│   ├── surveys/              # Survey domain
│   ├── mental_wellbeing/     # Mental health domain
│   └── physical_health/      # Physical health domain
│
├── infrastructure/            # Layer 2: Technical implementation
│   ├── database/
│   │   └── db.py            # MongoDB connection
│   ├── llm/
│   │   └── gemini.py        # Gemini client
│   ├── functions/           # Tool/function system
│   │   ├── core.py          # Function host
│   │   ├── registration.py  # Tool registration
│   │   ├── profile/         # Profile tools
│   │   ├── workout_coach/   # Workout tools
│   │   ├── nutrition/       # Nutrition tools
│   │   ├── mental_wellbeing/
│   │   ├── physical_health/
│   │   ├── companion/       # Companion tools
│   │   ├── memory/          # Memory tools
│   │   └── misc/            # Utility tools
│   └── auth/
│       └── authorization.py # Auth middleware
│
├── interfaces/               # Layer 3: External communication
│   ├── http/
│   │   └── routers/         # FastAPI routers
│   │       ├── sessions.py
│   │       ├── prompt.py
│   │       ├── files.py
│   │       ├── journal.py
│   │       ├── trackers.py
│   │       ├── goals.py
│   │       ├── plans.py
│   │       ├── surveys.py
│   │       ├── meals.py
│   │       ├── langgraph_chat.py
│   │       └── memory_debug.py
│   └── dependencies/
│       └── dependencies.py  # FastAPI DI
│
├── shared/                   # Shared: Cross-cutting concerns
│   ├── models/              # Shared models
│   └── types/               # Shared types
│
├── utils/                    # Shared: Utilities
│   ├── logging/             # Logging utilities
│   ├── persistence/         # Persistence utilities
│   ├── serialization.py    # Serialization
│   ├── time_context.py     # Time utilities
│   └── uuid_utils.py        # UUID utilities
│
├── files/                    # Shared: File management
│   ├── models.py
│   ├── persistence.py
│   └── services.py
│
├── usage/                    # Shared: Usage tracking
│   ├── models.py
│   ├── persistence.py
│   ├── pricing_service.py
│   └── ...
│
├── demo/                     # Shared: Demo data & scripts
│   ├── models.py
│   ├── persistence.py
│   ├── services.py
│   └── scripts/
│
├── web/                      # Web application entry point
│   ├── main.py              # FastAPI app (main entry)
│   └── deployment/          # Deployment configs
│
├── worker/                   # Background worker
│   ├── main.py              # Worker entry point
│   ├── scheduler/           # Job scheduler
│   └── jobs/                # Background jobs
│
├── kafka_worker/             # Kafka message consumer
│   ├── main.py
│   ├── consumer.py
│   └── handlers/
│
└── settings.json             # Application settings
```

---

## Component Interactions

### How Components Communicate

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP Request Flow                        │
└─────────────────────────────────────────────────────────────┘

Client
  │
  │ HTTP POST /Sessions/Current/Prompt
  ▼
┌─────────────────┐
│ web/main.py     │  FastAPI app
│ - Middleware    │  - CORS
│ - Error handler │  - Logging
└────────┬────────┘
         │
         │ app.include_router()
         ▼
┌─────────────────┐
│ interfaces/http │  HTTP Router
│ /routers/       │  - Validates request
│ prompt.py       │  - Extracts auth context
└────────┬────────┘
         │
         │ Calls domain service
         ▼
┌─────────────────┐
│ domain/chat/    │  Chat Service
│ services.py     │  - Orchestrates chat flow
└────────┬────────┘
         │
         │ Creates LangGraph service
         ▼
┌─────────────────┐
│ domain/agents/  │  LangGraph System
│ langgraph/      │  - Router node
│                 │  - Agent node
│                 │  - Aggregator node
└────────┬────────┘
         │
         │ Agent calls tools
         ▼
┌─────────────────┐
│ infrastructure/ │  Function Host
│ functions/      │  - Tool registry
│ core.py         │  - Tool execution
└────────┬────────┘
         │
         │ Tool calls repository
         ▼
┌─────────────────┐
│ domain/         │  Repository Interface
│ trackers/       │  (Abstract)
│ persistence.py  │
└────────┬────────┘
         │
         │ Implemented by
         ▼
┌─────────────────┐
│ infrastructure/ │  MongoDB Repository
│ (via app.state) │  - MongoOxeAITrackerRepository
│                 │  - Executes MongoDB queries
└────────┬────────┘
         │
         │ Queries database
         ▼
┌─────────────────┐
│   MongoDB       │  Database
│   (Atlas)       │  - Stores data
└─────────────────┘
```

### Dependency Injection Flow

```
┌─────────────────────────────────────────────────────────────┐
│              Application Startup (lifespan.py)               │
└─────────────────────────────────────────────────────────────┘

1. Create database connection
   oxeai_db = OxeAIDb(connection_string)

2. Create repositories
   tracker_repo = MongoOxeAITrackerRepository(oxeai_db)
   session_repo = MongoOxeAISessionRepository(oxeai_db)
   ...

3. Create services
   pricing_service = PricingService(pricing_repo)
   memory_facade = LongTermMemoryFacade(...)
   ...

4. Create function host factory
   function_host_factory = DefaultOxeAIFunctionHostFactory(...)

5. Store in app.state
   app.state.tracker_repository = tracker_repo
   app.state.session_repository = session_repo
   app.state.function_host_factory = function_host_factory
   ...

┌─────────────────────────────────────────────────────────────┐
│              Request Handling (Dependency Injection)         │
└─────────────────────────────────────────────────────────────┘

Router Handler:
  async def prompt_endpoint(
    app_state: AppState = Depends(get_app_state)
  ):
    # Access dependencies via app.state
    tracker_repo = app_state.tracker_repository
    function_host = app_state.function_host_factory.create()
    ...
```

---

## Data Flow

### Chat Request with Memory Retrieval

```
User Message: "How's my workout progress?"
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Router receives request                                  │
│    - Extracts user_id, session_id                          │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. LangGraph Router Node                                    │
│    - Analyzes query                                          │
│    - Selects WorkoutCoachAgent                              │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Memory Retrieval (Before Agent Execution)                │
│    - LongTermMemoryFacade.search()                         │
│    - Retrieves top 20 relevant memories                     │
│    - Filters by time context                                │
│    - Scores and ranks results                               │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Memory Injection                                         │
│    - Memories injected into agent's system prompt            │
│    - Provides context about user's workout history           │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. Agent Execution                                          │
│    - WorkoutCoachAgent processes query                      │
│    - Uses memory context to personalize response            │
│    - Calls tools: get_workout_session()                    │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. Tool Execution                                            │
│    - get_workout_session tool                               │
│    - Queries MongoDB for recent workouts                    │
│    - Returns workout data                                   │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. Agent Generates Response                                 │
│    - Uses workout data + memory context                     │
│    - Generates personalized response                        │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. Memory Hit Tracking                                      │
│    - Records which memories were used                       │
│    - Extends TTL for used memories                          │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ 9. Response Aggregation                                     │
│    - Formats final response                                 │
│    - Adds metadata (agent_type, topic)                      │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ 10. Save Conversation                                       │
│     - Saves user message to conversations collection        │
│     - Saves agent response to conversations collection      │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
         Response sent to client
```

---

## Key Design Patterns

### 1. Repository Pattern
**Purpose**: Abstracts data access logic

```
Domain Layer (Interface):
  class OxeAITrackerRepository(Protocol):
      async def get_by_id(...) -> Tracker

Infrastructure Layer (Implementation):
  class MongoOxeAITrackerRepository:
      async def get_by_id(...) -> Tracker:
          # MongoDB-specific code
```

### 2. Dependency Injection
**Purpose**: Loose coupling, testability

```
# Dependencies stored in app.state
app.state.tracker_repository = MongoOxeAITrackerRepository(db)

# Accessed via FastAPI Depends
async def endpoint(
    repo: OxeAITrackerRepository = Depends(get_tracker_repo)
):
    ...
```

### 3. Factory Pattern
**Purpose**: Creates complex objects

```
# Function host factory
function_host_factory = DefaultOxeAIFunctionHostFactory(...)
function_host = function_host_factory.create()
```

### 4. Facade Pattern
**Purpose**: Simplified interface to complex subsystem

```
# LongTermMemoryFacade provides simple API
# to complex memory system (search, extract, hit tracking, etc.)
memory_facade = LongTermMemoryFacade(...)
memories = await memory_facade.search(user_id, query)
```

---

## Summary

### Architecture Principles

1. **Separation of Concerns**: Each layer has a specific responsibility
2. **Dependency Direction**: 
   - Domain → Infrastructure (Domain defines interfaces, Infrastructure implements)
   - Interfaces → Domain (Interfaces call domain services)
   - Application → All (Application orchestrates everything)

3. **Testability**: 
   - Domain layer can be tested without infrastructure
   - Infrastructure can be mocked in tests

4. **Flexibility**: 
   - Can swap MongoDB for another database (change Infrastructure only)
   - Can add new HTTP endpoints (add to Interfaces only)
   - Can change business logic (modify Domain only)

### Key Takeaways

- **Domain Layer** = "What" the business does (pure logic)
- **Infrastructure Layer** = "How" it's implemented (technical details)
- **Interfaces Layer** = "How" external systems communicate (HTTP, etc.)
- **Application Layer** = "Orchestration" (startup, DI, coordination)

### File Naming Conventions

- `models.py` = Domain models or data structures
- `persistence.py` = Repository interfaces/implementations
- `services.py` = Business logic services
- `core.py` = Core functionality of a module
- `registration.py` = Registration/factory code

---

**Document Version**: 1.0  
**Last Updated**: January 2025  
**For**: New developers learning the codebase
