# Workout Plan Codebase Roadmap

This document provides a comprehensive guide to understanding the workout plan functionality in the oxeai-api codebase.

---

## 1. Domain Models (Data Layer)

**Location:** `src/domain/workouts/plans/models.py`

This defines the hierarchical data structure:

```
WorkoutPlan (root)
  └── WeekData (1-12 weeks)
      └── DayData (1-7 days)
          └── SessionData (workout sessions)
              └── ExerciseData (individual exercises)
                  └── KeyMetrics (sets, reps, weight, etc.)
```

### Key Models

| Model | Description |
|-------|-------------|
| `WorkoutPlan` | Top-level container with program name, goal, total weeks, user_id |
| `WeekData` | Weekly container with week_number, start_date, focus, week_summary |
| `DayData` | Daily container with day_number, planned_date, summary, tags |
| `SessionData` | Workout session with duration, exercises (uses `NestedDocumentModel`) |
| `ExerciseData` | Individual exercise with name, type, body_parts, equipment |
| `KeyMetrics` | Structured performance data (sets, reps, weight_kg, duration_min, distance_km) |

### Key Concepts

- `WorkoutPlan` uses `DocumentModel` (root level for MongoDB)
- `SessionData` and `ExerciseData` use `NestedDocumentModel` for proper MongoDB ID mapping
- `KeyMetrics` contains optional fields to support different exercise types (strength vs cardio)

---

## 2. Persistence Layer (Database)

**Location:** `src/domain/workouts/plans/persistence.py`

**MongoDB Collection:** `oxeAIWorkoutPlans`

### Repository Methods

| Method | Purpose |
|--------|---------|
| `get_workout_plan(user_id: UUID)` | Retrieve user's active workout plan |
| `upsert_workout_plan(user_id: UUID, plan: WorkoutPlan)` | Create or replace entire plan atomically |
| `delete_workout_plan(user_id: UUID)` | Remove user's plan, returns bool |

### Database Design

- One document per user (userId as filter key)
- Embedded structure (all data in single document)
- Uses `oxefit_pymongo` for ODM operations
- Handles UUID/camelCase conversion for MongoDB

---

## 3. LLM Tool Functions (Infrastructure Layer)

**Location:** `src/infrastructure/functions/workout_plan/`

### Function Overview

| File | Function | Purpose |
|------|----------|---------|
| `save_training_plan.py` | `SaveTrainingPlanFunction` | **Macro tool** - Creates program skeleton (no session details) |
| `save_daily_workout_plan.py` | `SaveDailyWorkoutPlanFunction` | **Micro tool** - Fills in daily session details (JIT) |
| `get_workout_plan.py` | `GetWorkoutPlanFunction` | Retrieves plan with flexible filtering |
| `update_workout_plan.py` | `UpdateWorkoutPlanFunction` | Unified write operation at any scope level |

### Key Design Pattern: Lazy Generation (Macro-Micro Split)

1. **Macro (SaveTrainingPlanFunction):** Creates skeleton structure early (low token cost)
   - Program name, goal, total weeks
   - Week summaries and day summaries
   - Sessions always empty (`sessions=[]`)

2. **Micro (SaveDailyWorkoutPlanFunction):** Fills details Just-In-Time as user approaches them
   - Replaces existing sessions for a specific day
   - Adds full exercise details with metrics

### GetWorkoutPlanFunction - Filtering Options

**Levels:** `program` | `week` | `day` | `session` | `exercise`

**Modes:** `summary` | `detailed`

**Filters:**
- `week_numbers`: List of weeks to include
- `day_numbers`: List of days (requires week_numbers)
- `session_ids`: Specific sessions
- `exercise_ids`: Specific exercises
- `start_date` / `end_date`: Date range (mutually exclusive with week/day numbers)

### UpdateWorkoutPlanFunction - Scope Determination

| Scope | Parameters Required |
|-------|---------------------|
| Entire plan | `plan_data` only |
| Week | `target_week` + `week_data` |
| Day | `target_week` + `target_day` + `day_data` |
| Session | `target_week` + `target_day` + `target_session_id` + `session_data` |

---

## 4. AI Agents

**Location:** `src/domain/agents/langgraph/agents/`

### Agent Overview

| Agent | File | Responsibility |
|-------|------|----------------|
| **WorkoutPlanAgent** | `workout_plan.py` | Multi-week program design, periodization, progressive overload |
| **WorkoutCoachAgent** | `workout_coach.py` | Daily session coaching, exercise guidance, progress tracking |
| **WorkoutAnalystAgent** | `workout_analyst.py` | Post-workout analysis and feedback |

### Agent Prompts

**Location:** `src/domain/agents/langgraph/prompts/agents/`

- `workout_plan.md` - Program design instructions
- `workout_coach.md` - Daily coaching instructions
- `workout_analyst.md` - Analysis instructions

### Agent Specializations

**WorkoutPlanAgent:**
- Creates multi-week training programs
- Applies progressive overload principles
- Uses periodization (linear, undulating, block)
- Balances volume, intensity, frequency
- Provides week 1-2 detailed, weeks 3+ summaries

**WorkoutCoachAgent:**
- Tracks daily workouts
- Provides exercise guidance
- Monitors progress
- Celebrates achievements
- Temperature: 0.3 (low for consistency)

---

## 5. Related Domains

### WorkoutProfile (User Preferences)

**Location:** `src/domain/profiles/training/models.py`

Provides context for AI-safe plan generation:

| Field | Purpose |
|-------|---------|
| `training_goals` | e.g., 'build muscle', 'lose weight' |
| `training_level` | beginner, intermediate, advanced |
| `weekly_frequency` | 1-7 days per week |
| `regular_exercises` | Activities user does often |
| `equipment` | Available equipment |
| `injuries` | Current/recent injuries for safety |
| `special_instructions` | User preferences/constraints |

**Functions:** `GetWorkoutProfileFunction`, `UpdateWorkoutProfileFunction`

### WorkoutSession (Coach Domain)

**Location:** `src/domain/workouts/coach/models.py`

Tracks actual workout performance (vs planned):

- `ExerciseMetrics` - Actual performance metrics
- `ExerciseRecord` - Individual exercise record

**Functions:** `GetWorkoutSessionFunction`, `UpdateWorkoutSessionFunction`

### OxeAIPlan (Activity Planning)

**Location:** `src/domain/plans/models.py`

A different, higher-level plan model for activities/segments (not workout-specific).

---

## 6. Function Registration

**Location:** `src/infrastructure/functions/registration.py`

Functions are registered to specific agent types:

```python
# Macro tool - available to WorkoutPlanAgent
SaveTrainingPlanFunction(workout_plan_db) → [AgentType.WORKOUT_PLAN]

# Micro tool - available to WorkoutPlanAgent
SaveDailyWorkoutPlanFunction(workout_plan_db) → [AgentType.WORKOUT_PLAN]

# Read tool - available to multiple agents
GetWorkoutPlanFunction(workout_plan_db) → [
    AgentType.WORKOUT_COACH,
    AgentType.WORKOUT_PLAN,
    AgentType.NUTRITION,
    AgentType.SLEEP,
    ...
]
```

Each function has:
- `access_type` (READ, DOMAIN_WRITE)
- Registered agent types
- Dependency injection (e.g., `DbWorkoutPlan` repository)

---

## 7. Data Flow Diagram

```
User Request
    ↓
WorkoutPlanAgent
    ↓
SaveTrainingPlanFunction → Creates skeleton (weeks/days, no sessions)
    ↓
SaveDailyWorkoutPlanFunction → Fills session details JIT
    ↓
MongoDB (oxeAIWorkoutPlans collection)
    ↓
WorkoutCoachAgent → Retrieves today's plan
    ↓
GetWorkoutPlanFunction → Returns filtered plan data
    ↓
User executes workout
    ↓
UpdateWorkoutSessionFunction → Log actual performance
    ↓
WorkoutAnalystAgent → Post-workout analysis
```

---

## 8. File Structure Summary

```
src/
├── domain/
│   ├── workouts/
│   │   ├── plans/
│   │   │   ├── models.py          # WorkoutPlan, WeekData, DayData, SessionData, ExerciseData
│   │   │   ├── persistence.py     # DbWorkoutPlan repository
│   │   │   └── __init__.py
│   │   ├── coach/
│   │   │   ├── models.py          # ExerciseMetrics, ExerciseRecord
│   │   │   ├── persistence.py     # DbWorkoutSession
│   │   │   └── __init__.py
│   │   └── review/
│   │       └── models.py          # OxeAIWorkoutReview
│   ├── profiles/training/
│   │   └── models.py              # WorkoutProfile
│   ├── plans/
│   │   └── models.py              # OxeAIPlan (different plan model)
│   └── agents/langgraph/
│       ├── agents/
│       │   ├── workout_plan.py
│       │   ├── workout_coach.py
│       │   └── workout_analyst.py
│       └── prompts/agents/
│           ├── workout_plan.md
│           ├── workout_coach.md
│           └── workout_analyst.md
├── infrastructure/
│   └── functions/
│       ├── workout_plan/
│       │   ├── save_training_plan.py
│       │   ├── save_daily_workout_plan.py
│       │   ├── get_workout_plan.py
│       │   ├── update_workout_plan.py
│       │   └── __init__.py
│       ├── profile/
│       │   ├── get_workout_profile.py
│       │   └── update_workout_profile.py
│       ├── workout_coach/
│       │   ├── get_workout_session.py
│       │   └── update_workout_session.py
│       └── registration.py
└── interfaces/
    └── http/routers/
        └── plans.py               # HTTP API for OxeAIPlan

tests/
├── functions/workout_plan/
│   ├── test_get_workout_plan.py
│   └── test_update_workout_plan.py
└── fixtures/
    └── plans.py

specs/
└── workouts/
    └── spec.md                    # Design specification

constants/
└── persistence.py                 # CollectionNames.OxeAIWorkoutPlans
```

---

## 9. Recommended Learning Path

1. **Read models first:** `src/domain/workouts/plans/models.py` - Understand the data structure
2. **Check persistence:** `src/domain/workouts/plans/persistence.py` - See how data is stored
3. **Study the functions:** Start with `get_workout_plan.py` (simplest), then `save_training_plan.py`
4. **Review agents:** `workout_plan.py` and its prompt file to see how AI uses these tools
5. **Look at tests:** `tests/functions/workout_plan/` for usage examples
6. **Read the spec:** `specs/workouts/spec.md` for design decisions

---

## 10. Key Design Patterns

| Pattern | Description |
|---------|-------------|
| **Lazy Generation** | Macro skeleton creation + JIT micro detail filling |
| **Hierarchical Filtering** | 5-level filtering (program → exercise) with summary/detailed modes |
| **Semantic Inference** | Empty sessions = future day, non-empty = detailed day |
| **Scope-Based Updates** | Single update function with scope determined by parameters |
| **AI-Safe Minimalism** | NestedDocumentModel for correct ID mapping, system fields hidden from LLM |
| **Multi-Agent Coordination** | Plan (design) + Coach (execution) + Analyst (review) |
