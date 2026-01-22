# Workout Plan Data Structure Diagram

## Hierarchy Overview

```
WorkoutPlan (DocumentModel - MongoDB Document)
├── id: GeminiSafeUUID
├── user_id: GeminiSafeUUID
├── program_name: str
├── goal: str
├── program_start_date: str (ISO 8601)
├── total_weeks: int (1-12)
├── overall_summary: str
└── weeks: List[WeekData] (max 12 weeks)
    │
    └── WeekData
        ├── week_number: int (1-12)
        ├── start_date: str (ISO 8601, Monday)
        ├── focus: str
        ├── week_summary: str
        └── days: List[DayData] (max 7 days)
            │
            └── DayData
                ├── day_number: int (1-7, Monday=1)
                ├── planned_date: str (ISO 8601)
                ├── summary: str
                ├── tags: List[str]
                └── sessions: List[SessionData]
                    │
                    └── SessionData (NestedDocumentModel)
                        ├── id: UUID
                        ├── start_time: str (ISO 8601)
                        ├── duration_min: float (>0)
                        ├── summary: str
                        ├── detailed_description: str
                        ├── tags: List[str]
                        └── exercises: List[ExerciseData] (min 1)
                            │
                            └── ExerciseData (NestedDocumentModel)
                                ├── name: str (snake_case, lowercase)
                                ├── type: str
                                ├── body_parts: List[str]
                                ├── equipment: str
                                ├── exercise_detailed_description: str
                                └── key_metrics: KeyMetrics
                                    │
                                    └── KeyMetrics
                                        ├── sets: Optional[int]
                                        ├── reps: Optional[List[int]]
                                        ├── weight_kg: Optional[List[float]]
                                        ├── distance_km: Optional[float]
                                        ├── speed_kmh: Optional[float]
                                        └── duration_min: Optional[List[float]]
```

## Detailed Class Diagrams

### WorkoutPlan (Top-Level Document)
```
┌─────────────────────────────────────────┐
│         WorkoutPlan                     │
│         (DocumentModel)                  │
├─────────────────────────────────────────┤
│ + id: GeminiSafeUUID                    │
│ + user_id: GeminiSafeUUID               │
│ + program_name: str                     │
│ + goal: str                             │
│ + program_start_date: str (ISO 8601)    │
│ + total_weeks: int (1-12)               │
│ + overall_summary: str                  │
│ + weeks: List[WeekData] (max 12)        │
└─────────────────────────────────────────┘
              │
              │ contains
              ▼
```

### WeekData
```
┌─────────────────────────────────────────┐
│            WeekData                      │
│            (BaseModel)                   │
├─────────────────────────────────────────┤
│ + week_number: int (1-12)               │
│ + start_date: str (ISO 8601, Monday)    │
│ + focus: str                            │
│ + week_summary: str                     │
│ + days: List[DayData] (max 7)          │
└─────────────────────────────────────────┘
              │
              │ contains
              ▼
```

### DayData
```
┌─────────────────────────────────────────┐
│            DayData                      │
│            (BaseModel)                   │
├─────────────────────────────────────────┤
│ + day_number: int (1-7, Monday=1)      │
│ + planned_date: str (ISO 8601)         │
│ + summary: str                          │
│ + tags: List[str]                       │
│ + sessions: List[SessionData]           │
│                                         │
│ Semantic:                               │
│ - sessions non-empty → detailed day     │
│ - sessions empty → summary day          │
└─────────────────────────────────────────┘
              │
              │ contains
              ▼
```

### SessionData
```
┌─────────────────────────────────────────┐
│          SessionData                     │
│      (NestedDocumentModel)                │
├─────────────────────────────────────────┤
│ + id: UUID                              │
│ + start_time: str (ISO 8601)            │
│ + duration_min: float (>0)              │
│ + summary: str                          │
│ + detailed_description: str             │
│ + tags: List[str]                        │
│ + exercises: List[ExerciseData] (min 1) │
└─────────────────────────────────────────┘
              │
              │ contains
              ▼
```

### ExerciseData
```
┌─────────────────────────────────────────┐
│         ExerciseData                    │
│      (NestedDocumentModel)                │
├─────────────────────────────────────────┤
│ + name: str (snake_case, lowercase)    │
│ + type: str                             │
│ + body_parts: List[str]                 │
│ + equipment: str                       │
│ + exercise_detailed_description: str    │
│ + key_metrics: KeyMetrics               │
└─────────────────────────────────────────┘
              │
              │ contains
              ▼
```

### KeyMetrics
```
┌─────────────────────────────────────────┐
│          KeyMetrics                     │
│          (BaseModel)                    │
├─────────────────────────────────────────┤
│ + sets: Optional[int]                  │
│ + reps: Optional[List[int]]             │
│ + weight_kg: Optional[List[float]]     │
│ + distance_km: Optional[float]        │
│ + speed_kmh: Optional[float]           │
│ + duration_min: Optional[List[float]]  │
│                                         │
│ Note: All fields always present         │
│ (null if not applicable)                │
└─────────────────────────────────────────┘
```

## Relationships Summary

```
WorkoutPlan (1) ──→ (0..12) WeekData
WeekData (1) ──→ (0..7) DayData
DayData (1) ──→ (0..*) SessionData
SessionData (1) ──→ (1..*) ExerciseData
ExerciseData (1) ──→ (1) KeyMetrics
```

## Key Constraints

### WorkoutPlan
- `total_weeks`: 1-12
- `weeks`: Max 12 items
- Week numbers must be unique

### WeekData
- `week_number`: 1-12
- `days`: Max 7 items

### DayData
- `day_number`: 1-7 (Monday=1)
- Semantic inference:
  - `sessions` non-empty → detailed day
  - `sessions` empty → summary day

### SessionData
- `duration_min`: Must be > 0
- `exercises`: Must have at least 1 exercise

### ExerciseData
- `name`: Must be lowercase snake_case
- `name`: Cannot be empty

### KeyMetrics
- All 6 fields always present (null if not applicable)
- Arrays match length:
  - `reps` length = `weight_kg` length
  - `duration_min` length = `sets` count

## Design Principles

1. **Weekly-based hierarchy**: Max 12 weeks
2. **Mixed time model**: Absolute `program_start_date` + relative week/day indices
3. **Adaptive detail**: Detailed plans for current period, summaries for future
4. **AI-safe minimalism**: Meaning inferred from content presence (e.g., empty sessions = summary day)
5. **Hybrid semantics**: Structured metrics (`KeyMetrics`) + natural language descriptions

## Example Structure

```
WorkoutPlan
├── program_name: "8-Week Strength Foundation"
├── goal: "muscle_gain"
├── program_start_date: "2025-10-25"
├── total_weeks: 8
├── overall_summary: "Progressive strength program..."
└── weeks: [
    WeekData(
        week_number: 1,
        start_date: "2025-10-25",
        focus: "Build strength foundation",
        days: [
            DayData(
                day_number: 1,
                planned_date: "2025-10-25",
                summary: "Upper body push day",
                sessions: [
                    SessionData(
                        start_time: "2025-10-25T18:00:00Z",
                        duration_min: 60.0,
                        summary: "Chest and Triceps Workout",
                        exercises: [
                            ExerciseData(
                                name: "bench_press",
                                type: "strength",
                                body_parts: ["chest", "triceps"],
                                equipment: "barbell",
                                key_metrics: KeyMetrics(
                                    sets: 4,
                                    reps: [8, 8, 6, 6],
                                    weight_kg: [80.0, 80.0, 85.0, 85.0]
                                )
                            )
                        ]
                    )
                ]
            )
        ]
    )
]
```
