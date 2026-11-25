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