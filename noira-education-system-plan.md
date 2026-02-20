# Noira — Personalized Education System for Swedish Grundskolan

## Context

The `noira-kunskapsbas/` folder contains 4 structured JSON knowledge blocks (out of 58 planned) covering Historia and Religionskunskap for grades 4-6, mapped to LGR22. Each block has a three-level progression system (nivå 1/2/3) with key concepts, core facts, reasoning structures, and a 3x3 question matrix. Currently there is **no application code** — just static JSON + Markdown content.

**Goal:** Build a web application that serves this content as a personalized learning platform for students (ages 10-12) and teachers, with adaptive level selection and learning path navigation through the curriculum's prerequisite graph.

---

## Technology Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Backend** | FastAPI (Python 3.11+) | Pydantic models match the JSON schema perfectly; async support; auto-generated API docs; `.gitignore` already targets Python |
| **Frontend** | Jinja2 + HTMX + vanilla JS | No build step; fast loading on school Chromebooks; server-rendered for accessibility; HTMX gives dynamic behavior without SPA complexity |
| **Database** | SQLite via SQLAlchemy | Zero-config; sufficient for school-scale usage; stores users, progress, quiz attempts |
| **CSS** | Custom CSS with CSS variables | Nivå color-coding (green/blue/purple); child-friendly typography; no framework dependencies |
| **Auth** | Session-based with bcrypt | Schools often block OAuth; simple cookie sessions; teachers create student accounts |

---

## Project Structure

```
noira/
├── noira-kunskapsbas/           # Existing content (UNCHANGED)
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app, lifespan (block loading)
│   ├── config.py                # Settings: paths, DB URL, secret key
│   ├── auth.py                  # Session middleware, login/logout, role checks
│   ├── models/
│   │   ├── block.py             # Pydantic models for JSON knowledge blocks
│   │   ├── user.py              # SQLAlchemy: User, StudentProfile, ClassGroup
│   │   └── progress.py          # SQLAlchemy: BlockProgress, QuizAttempt, LearningPath, Assignment
│   ├── data/
│   │   ├── block_loader.py      # Scan noira-kunskapsbas/so/*.json, parse with Pydantic
│   │   ├── block_registry.py    # In-memory indexed store with lookup by ID/subject/theme
│   │   └── db.py                # SQLite connection, session factory, table creation
│   ├── services/
│   │   ├── content_service.py   # Assemble block content at correct nivå (cumulative)
│   │   ├── level_engine.py      # Determine nivå for student+block (grade default, quiz history, teacher override)
│   │   ├── path_engine.py       # DAG traversal, topological sort, next-block recommendation
│   │   ├── quiz_engine.py       # Select questions from 3x3 matrix, save answers
│   │   └── progress_service.py  # Track block completion, query progress
│   ├── routers/
│   │   ├── auth_routes.py       # Login/logout pages
│   │   ├── student.py           # Student-facing HTML pages
│   │   ├── teacher.py           # Teacher dashboard HTML pages
│   │   └── api.py               # JSON API endpoints (HTMX partial responses)
│   ├── templates/
│   │   ├── base.html            # Swedish-language base layout
│   │   ├── auth/login.html
│   │   ├── student/             # home, block, quiz, path, profile, subjects
│   │   └── teacher/             # dashboard, class_view, student_detail, assign, quiz_review
│   └── static/
│       ├── css/noira.css
│       └── js/htmx.min.js, noira.js
├── tests/
│   ├── test_block_loader.py
│   ├── test_level_engine.py
│   ├── test_path_engine.py
│   ├── test_content_service.py
│   └── test_api.py
└── pyproject.toml               # Dependencies
```

---

## Data Layer

### Block Loading (startup)
- Recursively scan `noira-kunskapsbas/so/**/*.json`
- Parse each file with Pydantic `KunskapsBlock` model (common base + optional subject-specific fields)
- Register in `BlockRegistry` with indexes: by ID, by subject, by theme, prerequisite graph

### Key Pydantic Models (`app/models/block.py`)
- `BlockMetadata`: block_id, amne, titel, prerequisites[], leder_till[], rekommenderad_niva, etc.
- `Nyckelbegrepp`: begrepp, niva_1, niva_2, niva_3, kontext
- `KarnfaktaSektion`: rubrik, niva_1, niva_2, niva_3
- `KunskapsBlock`: All common fields + optional `kronologi` (Historia), `historiska_kallor` (Historia), `religiosa_traditioner` (Religion), `etik_och_livsfragor` (Religion), etc.

### SQLite Models (`app/models/user.py`, `progress.py`)
- **User**: id, username, password_hash, role ("student"/"teacher"), display_name
- **StudentProfile**: user_id, arskurs (4/5/6), current_niva (1/2/3), teacher_id
- **ClassGroup**: name, teacher_id
- **ClassMembership**: student_id, class_id
- **BlockProgress**: student_id, block_id, status (not_started/in_progress/completed), niva_used
- **QuizAttempt**: student_id, block_id, niva, betyg_level, question_text, student_answer, score
- **Assignment**: teacher_id, class/student_id, block_id, niva_override, due_date
- **LearningPath**: student_id, name, block_sequence (JSON array)

---

## Core Services

### Content Service (`content_service.py`)
Assembles block content cumulatively up to a target nivå:
- **Nivå 1**: Only niva_1 text, events/items with niva=1
- **Nivå 2**: niva_1 + niva_2 text, events with niva≤2
- **Nivå 3**: All three layers
- Each nivå layer gets a CSS class for visual distinction (colored left border)

### Level Engine (`level_engine.py`)
Determines which nivå to serve:
1. Default from student's `arskurs` (grade 4→1, grade 5→2, grade 6→3)
2. Teacher override (via Assignment.niva_override) takes precedence
3. Adaptive adjustment: if last 3 quiz scores in a subject are consistently strong → suggest level up; consistently weak → suggest level down
4. Block's `rekommenderad_niva` used as guidance

### Path Engine (`path_engine.py`)
- Builds prerequisite DAG from all blocks' `prerequisites[]` and `leder_till[]`
- Topological sort per subject for recommended sequence
- `get_available_blocks(completed)`: blocks whose prerequisites are all met
- `get_next_block(student)`: single recommended next block
- Handles missing blocks gracefully (54 of 58 don't exist yet — show as "Kommer snart")

### Quiz Engine (`quiz_engine.py`)
- Selects questions from `fragor_for_resonemang` 3x3 matrix (nivå × betyg)
- Presents free-text questions; student writes answers
- Simple heuristic scoring (word count, key concept mentions)
- Teacher reviews and assigns E/C/A quality rating
- No AI/LLM grading — formative assessment only

### Progress Service (`progress_service.py`)
- Track per-block status and nivå used
- Calculate completion percentage per subject
- Detect students who are stalled or struggling
- Feed data to level engine for adaptive adjustments

---

## Pages

### Student Views
| Page | Route | Purpose |
|------|-------|---------|
| Home | `/student/` | Welcome, "Fortsätt där du var", subject grid, progress bar |
| Subjects | `/student/subjects` | Blocks per subject as visual path; completed/current/locked status |
| Block | `/student/block/{id}` | Content view: nyckelbegrepp sidebar, kärnfakta sections, samband cards, timeline |
| Quiz | `/student/quiz/{id}` | One question at a time, free-text answer, adaptive difficulty |
| Path | `/student/path` | Visual DAG of blocks with color-coded status |
| Profile | `/student/profile` | Stats, level, recent activity |

### Teacher Views
| Page | Route | Purpose |
|------|-------|---------|
| Dashboard | `/teacher/` | Class list, average progress, attention flags |
| Class | `/teacher/class/{id}` | Student table with progress columns |
| Student detail | `/teacher/student/{id}` | Full progress timeline, quiz history, level controls |
| Assign | `/teacher/assign` | Pick blocks → assign to class/student, optional nivå override |
| Quiz review | `/teacher/quiz-review` | Queue of unreviewed answers, mark E/C/A |

---

## Implementation Phases

### Phase 1: Foundation — Data + Content Rendering
- `pyproject.toml` with dependencies (fastapi, uvicorn, jinja2, sqlalchemy, aiosqlite, bcrypt)
- Pydantic models for block schema, validated against all 4 existing blocks
- Block loader + registry with indexed lookups
- FastAPI app with startup block loading
- Content service: assemble block at specified nivå
- `base.html` template (Swedish), `noira.css` (nivå color scheme)
- Student block view: render full block content
- **Verify:** Browse to `/student/block/HI-46-B1`, see content at each nivå

### Phase 2: Auth + User Management
- SQLite setup with SQLAlchemy
- User/StudentProfile/ClassGroup models
- Session-based auth with bcrypt
- Login page, role-based routing
- Teacher can create student accounts and classes
- Seed script for test data
- **Verify:** Log in as teacher, create student, log in as student

### Phase 3: Student Experience
- Progress tracking (BlockProgress, QuizAttempt models)
- Student home page with subject grid and progress
- Subject browse with block status indicators
- Quiz system using fragor_for_resonemang matrix
- Progress updates on block read and quiz completion
- **Verify:** Student reads block, takes quiz, sees progress update

### Phase 4: Learning Paths
- Path engine: topological sort, available blocks, next-block recommendation
- Path visualization (HTML/CSS graph or simple list)
- "Nästa block" recommendation on student home
- Graceful handling of missing blocks ("Kommer snart")
- Cross-subject links from `tvaramnes_kopplingar`
- **Verify:** Complete a block, see next recommendations update

### Phase 5: Teacher Dashboard
- Class overview with progress summary
- Student detail with per-block breakdown
- Assignment system (assign blocks, set nivå override)
- Quiz review queue with E/C/A marking
- **Verify:** Teacher assigns block, student sees it, teacher reviews quiz answer

### Phase 6: Adaptive Progression
- Level adjustment logic based on quiz history
- Teacher notifications for students ready to level up/down
- Student-facing suggestions ("Du verkar redo för nivå 2!")
- **Verify:** Simulate good quiz scores → verify level-up suggestion

### Phase 7: Polish
- Swedish language throughout all UI
- Responsive design (Chromebooks + tablets)
- Error pages in Swedish
- Test suite (unit + integration)

---

## Critical Files to Reference

| File | Why |
|------|-----|
| `noira-kunskapsbas/so/historia/HI-46-B1.json` | Primary reference for Pydantic models (all Historia-specific fields) |
| `noira-kunskapsbas/so/religion/RE-46-G5.json` | Validates subject-specific field variations (Religion fields) |
| `noira-kunskapsbas/plan/noira-kunskapsbas-plan-SO-ak46_1.md` | Full DAG of 58 blocks, prerequisite chains, theme structure |
| `noira-kunskapsbas/kvalitet/granskning-HI-46-B1.md` | 18-point quality checklist (informs block validation) |
| `.gitignore` | Already configured for Python (venv, __pycache__, db.sqlite3) |

## MVP Handling of Missing Blocks (54 of 58)

- Prerequisites referencing unloaded blocks: show as "Inte tillgänglig ännu" but allow access anyway
- `leder_till` referencing missing blocks: show as "Kommer snart"
- Cross-subject links to missing blocks: gray out with tooltip
- Path engine operates only over loaded blocks; UI shows full plan as reference

---

## Verification

### End-to-End Test Flow
1. Start app → all 4 blocks load without errors
2. Teacher creates class "Klass 5A", student "Elev1" (grade 5)
3. Student logs in → home shows 4 blocks across 2 subjects
4. Opens RE-46-G5 → content at nivå 2 (grade 5 default); niva_3 content hidden
5. Takes quiz → answers nivå 2/betyg E question → answer saved
6. Teacher reviews answer → marks as "C" quality
7. Student views path → sees block dependencies, completed blocks checkmarked
8. Teacher assigns HI-46-B1 with nivå 3 override → student sees all 3 layers
9. After strong quiz performance → system suggests "Redo för nivå 3"
