# ShiftBridge: Application Overview & Scheduling Guide

**ShiftBridge** is a full-stack workforce scheduling application that combines an interactive weekly roster with an AI-powered scheduling assistant and a dual-approval workflow. It enables workers to request shift changes conversationally while giving managers full visibility and final sign-off over team schedules.

---

## 1. High-Level Architecture

ShiftBridge is built as a decoupled client-server application:

- **Frontend:** A **React + TypeScript** single-page application styled with **Tailwind CSS** (`App.tsx`, `CalendarGrid.tsx`, `SidebarChat.tsx`, `ProposalCard.tsx`, `AuthCard.tsx`, `Header.tsx`), communicating with the backend via `services/api.ts`.
- **Backend:** A **FastAPI** REST API (`main.py`) backed by **SQLite** via **SQLAlchemy 2.0** (`models.py`, `database.py`), secured with **JWT Bearer authentication** (`auth.py`, `security.py`).
- **AI Engine:** A local LLM integration (`ai_service.py`) using an OpenAI-compatible client with function-calling tools (`tools.py`) to inspect schedules and generate reschedule proposals.

---

## 2. Core Data Entities

The scheduling system revolves around four primary database models:

1. **Worker (`workers`):** Represents staff members and managers. Stores profile details (`name`, `role`, `department`), weekly hour limits (`max_hours`, `current_hours`), and privilege level (`is_manager`).
2. **Project (`projects`):** Groups workers under a supervising manager (`manager_id`). Workers and projects have a many-to-many relationship (`project_workers`).
3. **Shift (`shifts`):** Represents a scheduled work block for a specific `worker_id` and `project_id` on a given `day` (`Monday`–`Sunday`), with `start_time`, `length` (hours), `type` (`regular`, `off`, `updated`, `conflict`), and optional `note` or `updated_by_ai` flags.
4. **Proposal (`proposals`):** Represents a pending shift modification. Stores `worker_id`, a JSON payload (`new_shift_data`) comparing current vs. proposed shift details, and two boolean flags: `isWorkerApproved` and `isManagerApproved`.

---

## 3. How Scheduling Works

### A. Viewing the Weekly Schedule (`CalendarGrid.tsx`)

When a user logs in, the frontend fetches workers, shifts, and proposals in parallel and renders a 7-day (`Monday`–`Sunday`) schedule grid:

- **Worker View:** Displays only the logged-in worker's personal weekly shifts.
- **Manager View:** Displays all staff members (excluding the manager) along with filters for **Worker**, **Search Query** (name, role, or department), and **Role** (`Senior RN`, `Charge Nurse`, `RN`, `LPN`, `Support Specialist`).
- **Weekly Hours Tracking:** Each row calculates the worker's total scheduled hours against their `maxHours` limit, color-coding the badge (indigo for under limit, slate for at limit, rose for overtime).
- **Shift States:**
  - **Standard Roster (`regular`):** Normal scheduled shift.
  - **Day Off (`off`):** Unscheduled day.
  - **Updated via AI (`updated` / `updatedByAi`):** Highlighted in emerald green with an `Approved` badge.
- Clicking any cell opens a detail bar at the bottom of the grid showing the full shift time, duration, and notes.

---

### B. AI-Assisted Rescheduling (`SidebarChat.tsx` → `/chat`)

Workers can request schedule changes through the **Chat Sidebar**:

1. **User Prompt:** The worker types a request (e.g., *"Move my Tuesday shift to Thursday at 09:00 for 8 hours"*). The frontend sends the message and conversation history to `POST /chat`.
2. **AI Tool Calling (`ai_service.py` & `tools.py`):**
   The `LocalLLMShiftAssistant` receives the authenticated worker's context and has access to three database tools:
   - `get_my_shifts`: Inspects the worker's current shifts.
   - `get_other_workers_shifts`: Inspects coworkers' shifts in the same project(s) to check availability and avoid conflicts.
   - `create_reschedule_proposal`: Creates a new `Proposal` record in SQLite with `isWorkerApproved=False` and `isManagerApproved=False`.
3. **Inline Proposal Rendering:** If the AI creates a proposal, it is returned in `created_proposal` and rendered directly inside the chat thread as a `ProposalCard`, showing **Current** vs. **Proposed** day, start time, length, and any notes.

#### The AI Pipeline (Deep Dive)

The AI engine is not a single model call. Because it runs against a **local, lower-capability LLM**, the pipeline is explicitly structured into **three deterministic-orchestrated steps** so that weak model output can never corrupt schedule data. The guiding principle is:

> **The model routes and phrases; Python verifies and executes.** No shift ID, date, or proposal status ever originates from the model — every fact comes from a tool result captured verbatim by the backend.

```
┌─────────────────────────────────────────────────────────────────┐
│                        chat(message, db, user)                  │
└─────────────────────────────────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
┌───────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ STEP 1        │    │ STEP 2           │    │ STEP 3           │
│ Hint          │    │ Bounded Tool     │    │ Grounded Message │
│ Extraction    │───▶│ Loop             │───▶│ Formatting       │
│ (Python)      │    │ (LLM + tools)    │    │ (LLM + facts)    │
└───────────────┘    └──────────────────┘    └──────────────────┘
        │                      │                      │
        ▼                      ▼                      ▼
  regex entities       tools_executed[]        final reply text
  (shift_id, length,   + created_proposal      grounded in
   date_str)                                      verified results
```

**Step 1 — Hint Extraction (deterministic, no LLM).**
`extract_entities_fallback(text)` runs regexes over the raw user message to pull out anything explicitly stated: `shift_id` (e.g. "shift #42"), `project_id`, `length` (e.g. "for 6 hours", "8-hour shift"), and ISO-style `date_str`. These hints are appended to the message sent to the model as a bracketed suffix (`[Extracted hints from text: {...}]`), giving a weak model a second chance to notice explicit IDs it might otherwise drop. Hints are also used later by `_normalize_tool_args` to backfill missing tool arguments. The caller's `chat_history` is never mutated.

**Step 2 — Bounded Tool Loop.**
The model is called with `tools=OPENAI_TOOLS` and `tool_choice="auto"`, up to `max_tool_steps` times (5). Each iteration: call the model; save any `content` as a `draft_reply` (a phrasing hint only, not trusted as fact); if there are no `tool_calls`, stop; otherwise parse each tool call's arguments (`json.loads`, falling back to `{}` on error), normalize them (backfill `shift_id`/`length` from Step-1 hints, coerce `length` to float), execute via `tools.execute_tool_by_name(...)` with the DB session and current user ID, append `{tool, arguments, result}` to `tools_executed`, capture `created_proposal_data` on a successful proposal, and feed the result back as a `role: "tool"` message so the model can reason on the next turn. The loop terminates on: no tool calls, step limit reached, or exception.

The **tool router prompt** is what makes this work with a small model. It states the capability gap ("You do NOT know any schedules… You MUST call tools"), maps intent to each tool, and — critically — prescribes the reschedule order: **(A)** resolve the shift (use a provided `shift_id`, else `get_my_shifts`), **(B)** optionally gather candidates via `get_other_workers_shifts` if coverage is asked, **(C)** call `create_reschedule_proposal` with the resolved `shift_id` and `length`, **(D)** stop and summarize. It also enforces hard guards: never invent IDs, never call the proposal tool without a `shift_id`, never repeat an identical call, and no blind retries.

**Step 3 — Grounded Message Formatting.**
`_format_final_message` first builds a **deterministic summary** of every tool result (`_build_verified_facts_summary`), then asks the model to rewrite that summary into a short user-facing reply under strict rules: only facts present in the summary, explicitly say **"not completed"** if the proposal failed, no tool names, no JSON, no invented alternatives, 1–3 sentences. The formatter prompt is domain-aware and distinguishes four outcomes — proposal **succeeded** (confirm, cite shift ID and details), proposal **failed** (say it was not completed and give the reason), tools only **gathered data** (summarize and ask the user to confirm the target), and **no shift identified** (ask for the ID or date). If the formatter call fails or returns empty, the pipeline **falls back to the deterministic summary itself**, so the user always receives an accurate — if less polished — answer. If no tools ran at all, the cleaned `draft_reply` is returned directly.

**End-to-end reschedule example.**
User: *"Reschedule my 8-hour shift tomorrow to something else."*
Step 1 extracts `length = 8.0` (no `shift_id` stated). Step 2 iteration 1 calls `get_my_shifts` → returns shift ID `42`; iteration 2 optionally calls `get_other_workers_shifts` for candidates; iteration 3 calls `create_reschedule_proposal(shift_id=42, length=8)` → `{"success": true, "proposal": {...}}`; iteration 4 returns content with no tool calls and the loop breaks. Step 3 sees `SUCCEEDED` and replies: *"Your 8-hour shift on 2025-03-14 (ID 42) has been submitted for reschedule. Proposal #… created."*

**Return contract.** `chat()` returns `{"reply": str, "tools_used": [{tool, arguments, result}, ...], "created_proposal": dict | None}`. The `tools_used` list is a full audit trail in execution order; `created_proposal` is populated only when a proposal succeeded. On an unhandled exception the pipeline returns `{"reply": "Error", "tools_used": [], "created_proposal": None}` and logs the traceback.

**Configuration & reliability notes.** Base URL, API key, and model name come from `OPENAI_BASE_URL` / `OPENAI_API_KEY` / `LOCAL_LLM_MODEL` env vars (falling back to `settings`). The client and per-call timeouts are both `httpx.Timeout(215.0, connect=3.0)` — **a short client-level timeout (the original 1.5s) preempts the per-call value and causes near-constant silent failures on a local model**. `max_tool_steps` is 5 (3 is too tight to complete resolve → gather → propose → summarize), `max_history_turns` is 8, and `max_retries` is 0. `<think>...</think>` reasoning blocks are stripped from all model text by `_clean_model_text` before it reaches the user. The module exposes a singleton, `ai_engine = LocalLLMShiftAssistant()`.

---

### C. Dual-Approval Workflow (`ProposalCard.tsx` → `POST /data/approve`)

A shift change is **not** applied to the live schedule until **both** the worker and their supervising manager approve it:

1. **Viewing Proposals:**
   - Workers see their own proposals in the chat thread or the **Proposals** tab.
   - Managers see all pending proposals for workers assigned to projects they manage.
2. **Recording an Approval:**
   - Clicking **Approve** sends `proposal_id` to `POST /data/approve`.
   - The backend verifies authorization:
     - If the caller is the proposal's `worker_id`, `isWorkerApproved` is set to `True`.
     - If the caller is a manager of a project the worker belongs to, `isManagerApproved` is set to `True`.
     - Otherwise, a `403 Forbidden` error is returned.
3. **Partial Approval:**
   - If only one party has approved, the proposal remains in the database (`deleted: false`) and updates the UI to show **Approved** for that party while waiting on the other.
4. **Full Approval (Both Parties):**
   - Once both `isWorkerApproved` and `isManagerApproved` are `True`:
     - The backend extracts `new_shift_data` and updates the existing `Shift` (or creates a new one if no `shift_id` exists).
     - The `Proposal` record is **deleted** from the database (`deleted: true`).
     - The frontend removes the proposal from the **Proposals** list and chat thread, and immediately updates the `CalendarGrid` state with `result.new_shift`.

---

## 4. Authentication & Role Detection

- Users sign in via `AuthCard.tsx` (`POST /auth/login`), which returns a JWT stored in `localStorage` (`jwt`).
- All protected API calls attach `Authorization: Bearer <token>`.
- On load, `App.tsx` calls `GET /auth/me` to restore the session and determines whether to render the **Worker** or **Manager** interfac

# ShiftBridge: Application Overview & Scheduling Guide

**ShiftBridge** is a full-stack workforce scheduling application that combines an interactive weekly roster with an AI-powered scheduling assistant and a dual-approval workflow. It enables workers to request shift changes conversationally while giving managers full visibility and final sign-off over team schedules.

---

## 1. High-Level Architecture

ShiftBridge is built as a decoupled client-server application:

- **Frontend:** A **React + TypeScript** single-page application styled with **Tailwind CSS** (`App.tsx`, `CalendarGrid.tsx`, `SidebarChat.tsx`, `ProposalCard.tsx`, `AuthCard.tsx`, `Header.tsx`), communicating with the backend via `services/api.ts`.
- **Backend:** A **FastAPI** REST API (`main.py`) backed by **SQLite** via **SQLAlchemy 2.0** (`models.py`, `database.py`), secured with **JWT Bearer authentication** (`auth.py`, `security.py`).
- **AI Engine:** A local LLM integration (`ai_service.py`) using an OpenAI-compatible client with function-calling tools (`tools.py`) to inspect schedules and generate reschedule proposals.

---

## 2. Core Data Entities

The scheduling system revolves around four primary database models:

1. **Worker (`workers`):** Represents staff members and managers. Stores profile details (`name`, `role`, `department`), weekly hour limits (`max_hours`, `current_hours`), and privilege level (`is_manager`).
2. **Project (`projects`):** Groups workers under a supervising manager (`manager_id`). Workers and projects have a many-to-many relationship (`project_workers`).
3. **Shift (`shifts`):** Represents a scheduled work block for a specific `worker_id` and `project_id` on a given `day` (`Monday`–`Sunday`), with `start_time`, `length` (hours), `type` (`regular`, `off`, `updated`, `conflict`), and optional `note` or `updated_by_ai` flags.
4. **Proposal (`proposals`):** Represents a pending shift modification. Stores `worker_id`, a JSON payload (`new_shift_data`) comparing current vs. proposed shift details, and two boolean flags: `isWorkerApproved` and `isManagerApproved`.

---

## 3. How Scheduling Works

### A. Viewing the Weekly Schedule (`CalendarGrid.tsx`)

When a user logs in, the frontend fetches workers, shifts, and proposals in parallel and renders a 7-day (`Monday`–`Sunday`) schedule grid:

- **Worker View:** Displays only the logged-in worker's personal weekly shifts.
- **Manager View:** Displays all staff members (excluding the manager) along with filters for **Worker**, **Search Query** (name, role, or department), and **Role** (`Senior RN`, `Charge Nurse`, `RN`, `LPN`, `Support Specialist`).
- **Weekly Hours Tracking:** Each row calculates the worker's total scheduled hours against their `maxHours` limit, color-coding the badge (indigo for under limit, slate for at limit, rose for overtime).
- **Shift States:**
  - **Standard Roster (`regular`):** Normal scheduled shift.
  - **Day Off (`off`):** Unscheduled day.
  - **Updated via AI (`updated` / `updatedByAi`):** Highlighted in emerald green with an `Approved` badge.
- Clicking any cell opens a detail bar at the bottom of the grid showing the full shift time, duration, and notes.

---

### B. AI-Assisted Rescheduling (`SidebarChat.tsx` → `/chat`)

Workers can request schedule changes through the **Chat Sidebar**:

1. **User Prompt:** The worker types a request (e.g., *"Move my Tuesday shift to Thursday at 09:00 for 8 hours"*). The frontend sends the message and conversation history to `POST /chat`.
2. **AI Tool Calling (`ai_service.py` & `tools.py`):**
   The `LocalLLMShiftAssistant` receives the authenticated worker's context and has access to three database tools:
   - `get_my_shifts`: Inspects the worker's current shifts.
   - `get_other_workers_shifts`: Inspects coworkers' shifts in the same project(s) to check availability and avoid conflicts.
   - `create_reschedule_proposal`: Creates a new `Proposal` record in SQLite with `isWorkerApproved=False` and `isManagerApproved=False`.
3. **Inline Proposal Rendering:** If the AI creates a proposal, it is returned in `created_proposal` and rendered directly inside the chat thread as a `ProposalCard`, showing **Current** vs. **Proposed** day, start time, length, and any notes.

---

### C. Dual-Approval Workflow (`ProposalCard.tsx` → `POST /data/approve`)

A shift change is **not** applied to the live schedule until **both** the worker and their supervising manager approve it:

1. **Viewing Proposals:**
   - Workers see their own proposals in the chat thread or the **Proposals** tab.
   - Managers see all pending proposals for workers assigned to projects they manage.
2. **Recording an Approval:**
   - Clicking **Approve** sends `proposal_id` to `POST /data/approve`.
   - The backend verifies authorization:
     - If the caller is the proposal's `worker_id`, `isWorkerApproved` is set to `True`.
     - If the caller is a manager of a project the worker belongs to, `isManagerApproved` is set to `True`.
     - Otherwise, a `403 Forbidden` error is returned.
3. **Partial Approval:**
   - If only one party has approved, the proposal remains in the database (`deleted: false`) and updates the UI to show **Approved** for that party while waiting on the other.
4. **Full Approval (Both Parties):**
   - Once both `isWorkerApproved` and `isManagerApproved` are `True`:
     - The backend extracts `new_shift_data` and updates the existing `Shift` (or creates a new one if no `shift_id` exists).
     - The `Proposal` record is **deleted** from the database (`deleted: true`).
     - The frontend removes the proposal from the **Proposals** list and chat thread, and immediately updates the `CalendarGrid` state with `result.new_shift`.

---

## 4. Authentication & Role Detection

- Users sign in via `AuthCard.tsx` (`POST /auth/login`), which returns a JWT stored in `localStorage` (`jwt`).
- All protected API calls attach `Authorization: Bearer <token>`.
- On load, `App.tsx` calls `GET /auth/me` to restore the session and determines whether to render the **Worker** or **Manager** interface.
