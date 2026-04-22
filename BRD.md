# Business Requirements Document for Orbit

## Executive Summary
Orbit is an intelligent task manager designed to help users efficiently manage their tasks, projects, and deadlines. By providing a smart interface that incorporates advanced algorithms for task prioritization and scheduling, Orbit empowers users to focus on what matters most, ensuring that no task is overlooked.

## Business Objectives
- Increase user productivity by providing intelligent task suggestions.
- Streamline task management through automation and smart features.
- Enhance user collaboration for shared projects.
- Maximize user satisfaction with an intuitive and engaging interface.

## Core Features
| Feature                           | Description                                                                                          |
|-----------------------------------|------------------------------------------------------------------------------------------------------|
| Smart To-Do List                  | Intelligent task suggestions based on user behavior and deadlines.                                   |
| Task Structure                     | Create tasks with subtasks for complex projects.                                                    |
| Recurring Tasks                   | Automate the creation of repetitive tasks.                                                          |
| Time Estimates                     | Users can set estimated completion times for tasks, aiding in scheduling.                          |
| Attachments                        | Support for attaching files to tasks, facilitating easy access to relevant documents.               |
| Priority Customization             | Users can assign different priority levels to tasks, altering their urgency.                        |
| Task States Customization          | Customizable states for tasks to reflect their progress.                                           |
| Tags and Color Coding              | Use tags with specific colors for easy visual identification of task categories.                    |
| Filtering and Sorting Capabilities | Advanced options to filter and sort tasks based on various criteria.                               |
| Bulk Operations                    | Perform bulk actions on multiple tasks at once for efficiency.                                     |

## Technical Architecture
- **Frontend**: Developed using React for a responsive user experience.
- **Backend**: Node.js with Express for handling API requests.
- **Database**: PostgreSQL for data storage and retrieval.

Perfect! I'll generate the **complete, detailed F01 specification** as one comprehensive markdown document that you can copy directly into your BRD.md.

Here it is:

```markdown
---

# F01: Smart To-Do List - Complete Detailed Specification

## 1. Overview

The Smart To-Do List is the core feature of Orbit, providing users with a powerful yet intuitive task management system. It allows users to create, organize, prioritize, and track tasks efficiently while supporting advanced features like subtasks, recurring tasks, time estimates, attachments, and extensive customization options.

**Key Value Proposition:**
- Frictionless task entry (seconds to create)
- Comprehensive organization (priorities, tags, states)
- Flexible customization (user-defined priorities, states, tags)
- Advanced scheduling (recurring tasks, time estimates)
- Bulk operations for efficiency
- Complete audit trail (soft delete with recovery)

---

## 2. Task Structure & Metadata

### 2.1 Core Task Entity

Each task in the system consists of the following attributes:

| Attribute | Type | Required | Length | Description |
|-----------|------|----------|--------|-------------|
| **id** | UUID | Yes | - | Unique identifier for the task (generated on creation) |
| **user_id** | UUID | Yes | - | Owner of the task (Foreign Key to users table) |
| **title** | VARCHAR | Yes | 255 | Task name/title (concise description) |
| **description** | TEXT | No | Unlimited | Detailed explanation, notes, or context for the task |
| **priority_id** | UUID | No | - | Link to priority level (Foreign Key to priorities table, nullable) |
| **status_id** | UUID | Yes | - | Link to current task state (Foreign Key to task_states table) |
| **due_date** | TIMESTAMP | No | - | Deadline for task completion (timezone-aware) |
| **time_estimate_minutes** | INTEGER | No | - | Estimated time to complete in minutes (e.g., 120 = 2 hours) |
| **parent_task_id** | UUID | No | - | Reference to parent task if this is a subtask (Self-join FK, nullable) |
| **recurrence_rule** | VARCHAR | No | 255 | RFC 5545 RRULE string for recurring tasks |
| **recurrence_series_id** | UUID | No | - | Links all recurring instances to their series |
| **is_recurring_instance** | BOOLEAN | No | - | Flag indicating if task is instance of recurring series (default: FALSE) |
| **gcal_event_id** | VARCHAR | No | 255 | Linked Google Calendar event ID (for F03 integration) |
| **completed_at** | TIMESTAMP | No | - | Timestamp when task was marked complete (null until completion) |
| **display_order** | INTEGER | No | - | Custom sort order for manual prioritization (default: 0) |
| **created_at** | TIMESTAMP | Yes | - | When task was created (auto-set to NOW()) |
| **updated_at** | TIMESTAMP | Yes | - | When task was last updated (auto-updated on any change) |
| **deleted_at** | TIMESTAMP | No | - | Soft delete timestamp (null if not deleted, triggers 30-day retention) |

### 2.2 Task State Transitions

Tasks progress through states as follows:

```
User creates task
        ↓
[Pending] ← default state
   ↓ ↓ ↓ (user can transition to any state)
   ├→ [In Progress]
   ├→ [Blocked]
   ├→ [Completed] ← sets completed_at timestamp
   ├→ [Custom State 1]
   └→ [Custom State N]

Reopening completed task → clears completed_at timestamp
Soft deleting task → sets deleted_at timestamp
Hard delete after 30 days → permanently removes from DB
```

### 2.3 Task Creation Flow

**When user creates a new task:**
1. User provides: title (required), description (optional), due_date (optional), other fields
2. System automatically sets:
   - id = NEW UUID
   - user_id = current logged-in user
   - status_id = first default state (Pending)
   - priority_id = NULL (user can set later)
   - created_at = NOW()
   - updated_at = NOW()
   - deleted_at = NULL
   - display_order = (count of user's tasks + 1)
3. Task is immediately searchable and visible in task list

---

## 3. Subtasks

### 3.1 Subtask Definition & Structure

A subtask is a child task linked to a parent task via the `parent_task_id` field. Subtasks allow users to break down complex tasks into manageable components.

**Subtask Characteristics:**
- Inherits from parent but maintains independence
- Can have different priority, status, due date, tags than parent
- Can be marked complete independently
- Appears indented under parent in UI
- Max nesting: **1 level only** (no sub-sub-tasks for MVP)

**Subtask Attributes:**
- Same structure as regular tasks (see 2.1)
- Plus: `parent_task_id` field references parent task UUID
- **Constraint**: `parent_task_id != id` (cannot be parent of itself)

### 3.2 Subtask Rules & Constraints

```sql
-- Database constraints for subtasks
ALTER TABLE tasks ADD CONSTRAINT no_self_parent 
    CHECK (parent_task_id != id);

-- Subtask depth constraint (max 1 level)
-- Enforced in application: when creating subtask, check parent's parent_task_id IS NULL
```

**Rules:**
1. Subtask can only be created under a root task (parent must have parent_task_id = NULL)
2. Cannot create subtask under another subtask (max 1 nesting level)
3. Deleting parent task soft-deletes all subtasks
4. Subtask can be promoted to root (set parent_task_id = NULL)
5. Subtask can be reassigned to different parent

### 3.3 Subtask Completion Logic

**Progress Tracking:**
- Parent task displays: "X of Y subtasks complete" indicator
- Example: "3 of 5 subtasks complete" → 60% progress bar

**Completion Scenarios:**

**Scenario 1: User completes a single subtask**
```
Subtask marked complete
  ↓
Set subtask.completed_at = NOW()
Set subtask.status_id = "Completed" state
  ↓
Recalculate parent progress
  ↓
If all subtasks complete:
  - Show notification: "All subtasks done. Complete parent task?"
  - User can choose YES or NO
  - If YES: mark parent as Completed
  - If NO: parent remains in current state
```

**Scenario 2: User marks parent task as complete**
```
Parent marked complete
  ↓
System checks for incomplete subtasks
  ↓
If incomplete subtasks exist:
  - Show confirmation: "Complete all subtasks too? (X incomplete)"
  - User chooses: YES / NO / CANCEL
  - If YES: mark all incomplete subtasks as Completed
  - If NO: parent moves to Completed, subtasks stay independent
  - If CANCEL: abort completion
```

**Scenario 3: User reopens completed parent task**
```
Parent reopened (moved away from Completed state)
  ↓
completed_at timestamp cleared
  ↓
Subtasks stay in their current states (no automatic change)
```

### 3.4 Subtask Lifecycle Examples

**Example 1: Simple breakdown**
```
Parent: "Finish Project Report"
  ├── Subtask 1: "Research requirements" [Completed]
  ├── Subtask 2: "Write introduction" [In Progress]
  ├── Subtask 3: "Create visuals" [Pending]
  └── Subtask 4: "Final review" [Pending]
  
Progress: 1/4 complete (25%)
```

**Example 2: Priority inheritance**
```
Parent: "Urgent client meeting" [Priority: High] [Due: Today 5 PM]
  ├── Subtask 1: "Prepare slides" [Priority: Low] [Due: Today 2 PM]
  │   → Independent priority, can override parent
  ├── Subtask 2: "Compile data" [Priority: High] [Due: Today 3 PM]
  └── Subtask 3: "Send agenda" [Priority: Medium] [Due: Today 1 PM]
```

---

## 4. Recurring Tasks (RFC 5545 RRULE)

### 4.1 Recurrence Pattern Storage

Recurring tasks use the RFC 5545 RRULE format (standard for calendar applications):

**Format:** `FREQ=frequency[;option1=value1;option2=value2...]`

### 4.2 Common RRULE Examples

| Pattern | RRULE | Description |
|---------|-------|-------------|
| Daily | `FREQ=DAILY;INTERVAL=1` | Every day |
| Every 2 days | `FREQ=DAILY;INTERVAL=2` | Every other day |
| Weekly (M, W, F) | `FREQ=WEEKLY;BYDAY=MO,WE,FR` | Monday, Wednesday, Friday |
| Monthly (15th) | `FREQ=MONTHLY;BYMONTHDAY=15` | 15th of every month |
| Quarterly | `FREQ=MONTHLY;INTERVAL=3` | Every 3 months |
| Yearly (Dec 25) | `FREQ=YEARLY;BYMONTHDAY=25;BYMONTH=12` | Every December 25th |
| Last weekday | `FREQ=MONTHLY;BYDAY=-1MO` | Last Monday of month |
| Business days | `FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR` | Monday through Friday |

### 4.3 Recurring Task Creation & Generation

**When user creates recurring task:**
```
User sets:
- Title: "Weekly team meeting"
- Due Date: Monday 10 AM
- Recurrence: FREQ=WEEKLY;BYDAY=MO
- End Date (optional): None

System:
1. Validate RRULE syntax (must be valid RFC 5545)
2. Create recurrence_series_id = NEW UUID
3. Generate next 12 months of instances:
   - Instance 1: Monday Week 1, 10 AM → id=uuid1, recurrence_series_id=series_uuid, is_recurring_instance=true
   - Instance 2: Monday Week 2, 10 AM → id=uuid2, recurrence_series_id=series_uuid, is_recurring_instance=true
   - Instance 3: Monday Week 3, 10 AM → id=uuid3, recurrence_series_id=series_uuid, is_recurring_instance=true
   - ... (up to 12 months forward)
4. Mark original task as series template (store RRULE in recurrence_rule field)
5. Set recurrence_series_id on all instances
```

### 4.4 Instance Independence

**Critical Rule:** Completing or deleting one instance does NOT affect other instances.

**Example:**
```
Series: "Daily standup" (FREQ=DAILY)
- Instance 1 (Monday): [Completed] → completed_at set
- Instance 2 (Tuesday): [Pending] → unaffected
- Instance 3 (Wednesday): [Pending] → unaffected

If user deletes Instance 2:
- Instance 2 marked deleted (deleted_at set)
- Instance 1 & 3 continue existing
- Series continues generating new instances
```

### 4.5 Recurring Task Editing

**When user edits recurrence pattern:**

**Case 1: "Edit just this one"**
```
User edits Instance 2 (Tuesday standup)
- Original RRULE: FREQ=DAILY
- User changes to: "Skip next week"

System:
1. Create exception for Instance 2
2. Unlink Instance 2 from series (set recurrence_series_id = NULL)
3. Instance 2 becomes independent task
4. Series continues generating new instances
5. Other instances unaffected
```

**Case 2: "Edit this and future"**
```
User edits Instance 2 and chooses "This and future"
- Original RRULE: FREQ=DAILY (starts Monday)
- User changes to: FREQ=WEEKLY starting Instance 2

System:
1. Keep Instance 1 as-is (linked to old series)
2. Create new series (new recurrence_series_id)
3. Instance 2+ linked to new series with new RRULE
4. Old series stops generating new instances after Instance 1
```

### 4.6 Recurring Task Lifecycle

```
Creation Date: Jan 1, 2026
Recurrence: FREQ=MONTHLY;BYMONTHDAY=1

Jan 1: Instance 1 created [Pending]
Feb 1: Instance 2 created [Pending]
Mar 1: Instance 3 created [Pending]
...
Dec 1: Instance 12 created [Pending]

User completes Jan 1 instance:
  Jan 1: [Completed] ✓
  Feb 1-Dec 1: [Pending] (unaffected)

System reaches Dec 1 (last generated instance):
  - Check if series has UNTIL date
  - If no END, generate next 12 months:
    Jan 1, 2027: Instance 13 created
    Feb 1, 2027: Instance 14 created
    ... (continues indefinitely or until UNTIL date)

User removes recurrence:
  - Delete RRULE from template
  - Existing instances remain but no new ones generate
```

### 4.7 Timezone Considerations for Recurring Tasks

**Important:** Recurring tasks respect user's timezone.

**Example:**
```
User (UTC-5 / EST): Sets recurring task at "9 AM"
- System stores: due_date with UTC-5 offset
- Recurring instances generated at 9 AM EST daily
- If user travels to UTC+8: instances still generate at 9 AM EST (original tz)
- User can update timezone in settings; recurrence pattern respects new tz
```

---

## 5. Time Estimates

### 5.1 Time Estimate Storage & Display

**Storage:**
- Field: `time_estimate_minutes` (INTEGER)
- Unit: Minutes (flexible precision)
- Examples:
  - 15 minutes = Quick task
  - 60 minutes = 1 hour
  - 120 minutes = 2 hours
  - 150 minutes = 2.5 hours

**Display Format in UI:**
- 15 min → "15 mins"
- 60 min → "1 hour"
- 90 min → "1h 30m"
- 120 min → "2 hours"
- 240 min → "4 hours"

**Formula for conversion:**
```
hours = time_estimate_minutes / 60
minutes_remainder = time_estimate_minutes % 60

Display:
- If hours = 0: show minutes only (e.g., "30 mins")
- If minutes = 0: show hours only (e.g., "3 hours")
- If both > 0: show both (e.g., "2h 30m")
```

### 5.2 Use Cases & Applications

**Use Case 1: Daily Capacity Planning**
```
Dashboard shows:
"You have 8 hours of tasks scheduled for today"

Calculation:
- Task 1 (30 mins) = 0.5 hours
- Task 2 (2 hours) = 2 hours
- Task 3 (1.5 hours) = 1.5 hours
- Task 4 (4 hours) = 4 hours
Total = 8 hours
```

**Use Case 2: Sorting by Quick Wins**
```
Sort by Time Estimate (Ascending):
1. "Reply to email" (15 mins)
2. "Update status" (30 mins)
3. "Fix bug" (90 mins)
4. "Refactor code" (240 mins)

User gets momentum by completing quick tasks first
```

**Use Case 3: Capacity Warning**
```
Today's tasks total > 8 hours (user's workday):
⚠️ Warning: "You have 12 hours of tasks today. Consider moving some to tomorrow."
```

**Use Case 4: Analytics & Insights**
```
Weekly Summary:
- Estimated: 20 hours
- Actually took: 18 hours
- Accuracy: 90%
```

### 5.3 Time Estimate Validation & Rules

```sql
-- Database constraint
ALTER TABLE tasks ADD CONSTRAINT valid_time_estimate 
    CHECK (time_estimate_minutes IS NULL OR time_estimate_minutes > 0);
```

**Rules:**
1. Time estimate must be positive (if provided)
2. Time estimate must be ≤ 43,200 minutes (30 days)
3. NULL value allowed (optional, unestimated task)
4. Cannot be negative
5. Typically in 15-minute increments (enforce via UI)

**Validation Examples:**
```
Valid:     15, 30, 45, 60, 90, 120, 300, 600
Invalid:   0, -30, -1, 50000
Edge:      1 (1 minute), 43200 (30 days)
```

---

## 6. Attachments

### 6.1 Attachment Constraints & Specifications

**Per-Task Limits:**
- Maximum: **5 files per task**
- Failure handling: Reject 6th file with error message

**Per-File Limits:**
- Maximum size: **10 MB** (10,485,760 bytes)
- Failure handling: Reject oversized file with error message

**Supported File Types (MIME Types):**

| Category | Types | MIME Types |
|----------|-------|-----------|
| Images | JPG, PNG, GIF | image/jpeg, image/png, image/gif |
| Documents | PDF | application/pdf |
| Office | DOCX, XLSX, PPTX | application/vnd.openxmlformats-officedocument.* |
| Text | TXT | text/plain |

**Rejected Types:** Executables (.exe, .sh), Archives (.zip, .rar), Scripts (.js, .py)

### 6.2 Attachment Storage Strategy

**Storage Architecture:**
```
Local Storage (development):
  /uploads/tasks/{task_id}/{file_id}_{filename}

Cloud Storage (production):
  AWS S3: s3://orbit-attachments/{user_id}/{task_id}/{file_id}_{filename}
  CDN: https://cdn.orbit.app/attachments/{file_id}
```

**Metadata Stored in Database:**
```sql
attachment {
  id: UUID                    -- Unique file identifier
  task_id: UUID               -- Link to parent task
  file_url: VARCHAR(500)      -- S3/CDN URL
  file_name: VARCHAR(255)     -- Original filename (user-facing)
  file_size_bytes: INTEGER    -- Actual file size in bytes
  mime_type: VARCHAR(100)     -- MIME type (image/jpeg, etc.)
  uploaded_at: TIMESTAMP      -- When file was uploaded
  uploaded_by: UUID           -- User who uploaded (FK to users)
}
```

### 6.3 Attachment Lifecycle

**Upload Flow:**
```
1. User selects file(s) to attach
2. Client validates:
   - File size < 10 MB
   - File type in whitelist
   - Count < 5 per task
3. Client uploads to S3 (pre-signed URL)
4. S3 confirms upload success
5. Client sends metadata to Orbit API:
   POST /api/v1/tasks/{task_id}/attachments
   {
     "file_url": "s3://orbit-attachments/...",
     "file_name": "requirements.pdf",
     "file_size_bytes": 2048576,
     "mime_type": "application/pdf"
   }
6. Backend stores attachment record in DB
7. Attachment appears in task details
```

**View/Download Flow:**
```
1. User clicks attachment in task UI
2. Browser loads file from CDN URL (cached)
3. File opens in new tab or downloads (based on type)
4. Analytics: Track download count
```

**Delete Flow:**
```
1. User clicks delete on attachment
2. Backend soft-deletes from attachment table (set deleted_at)
3. Alternative: Hard-delete from S3 immediately (choice based on policy)
4. Attachment removed from task UI
```

**Cascade Delete:**
```
When task is soft-deleted:
  ↓
All associated attachments also soft-deleted (deleted_at set)
After 30 days:
  ↓
Hard-delete task + all attachments
  ↓
Call S3 API to delete files from bucket
```

### 6.4 Attachment Validation (Database Level)

```sql
-- Check max 5 attachments per task
CREATE FUNCTION check_attachment_limit()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT COUNT(*) FROM attachments 
        WHERE task_id = NEW.task_id 
        AND deleted_at IS NULL) > 5 THEN
        RAISE EXCEPTION 'Maximum 5 attachments per task';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_check_attachment_limit
BEFORE INSERT ON attachments
FOR EACH ROW EXECUTE FUNCTION check_attachment_limit();

-- Check file size constraint
ALTER TABLE attachments ADD CONSTRAINT valid_file_size 
    CHECK (file_size_bytes > 0 AND file_size_bytes <= 10485760);

-- Check MIME type whitelist
ALTER TABLE attachments ADD CONSTRAINT valid_mime_type
    CHECK (mime_type IN (
        'image/jpeg', 'image/png', 'image/gif',
        'application/pdf',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        'application/vnd.openxmlformats-officedocument.presentationml.presentation',
        'text/plain'
    ));
```

---

## 7. Priority Levels

### 7.1 Default Priority System

Every user gets three immutable default priorities:

| Priority | Weight | Color | Use Case |
|----------|--------|-------|----------|
| **Low** | 1 | #90EE90 (light green) | Non-urgent, can be deferred, nice-to-have |
| **Medium** | 2 | #FFD700 (gold) | Standard priority, normal workflow |
| **High** | 3 | #FF6B6B (red) | Urgent, should be prioritized, blocking items |

**Properties:**
- `is_default = TRUE` (cannot be deleted or modified)
- Weight used for sorting (higher weight = higher priority)
- Weight unique per user (no duplicates)

### 7.2 Custom Priority Levels

Users can create custom priorities beyond the default three.

**Custom Priority Creation:**
```
User creates: "Critical"
System generates:
{
  id: NEW UUID
  user_id: current_user_id
  name: "Critical"
  weight: 4  (automatically assigned higher than High)
  is_default: FALSE
  created_at: NOW()
}
```

**Custom Priority Rules:**
1. Max 20 custom priorities per user (to prevent bloat)
2. Weight must be > 0 and ≤ 100
3. Name must be unique per user (case-insensitive)
4. Cannot edit default priorities (Low, Medium, High)
5. Can delete custom priorities if no tasks use them

### 7.3 Priority Usage & Application

**Sorting by Priority:**
```sql
-- Tasks sorted by priority weight descending (High first)
SELECT t.* FROM tasks t
JOIN priorities p ON t.priority_id = p.id
WHERE t.user_id = $1 AND t.deleted_at IS NULL
ORDER BY p.weight DESC, t.due_date ASC;

Result order:
1. Critical (weight 4)
2. High (weight 3)
3. Medium (weight 2)
4. Low (weight 1)
5. Unpriorized (NULL - last)
```

**Priority in UI:**
- Task list: Show priority as colored label/badge
- Example: `[🔴 High] Task Title`
- Filters: Multi-select priorities to show/hide
- Dashboard: Priority distribution pie chart

**Priority Escalation (Future Enhancement):**
```
If task due_date < TODAY:
  - Flag as "Overdue"
  - Visual highlight (red background)
  - System optionally auto-escalate priority
```

### 7.4 Priority Defaults for New Tasks

```
When user creates task without specifying priority:
- priority_id = NULL (unpriorized)
- User can add priority later
- Unprioritized tasks appear last in sorted lists
```

---

## 8. Task States & Customization

### 8.1 Default Task States

Every user gets four immutable default states:

| State | Description | Semantics | Color |
|-------|-------------|-----------|-------|
| **Pending** | Task not yet started | No work begun | #E0E0E0 (gray) |
| **In Progress** | Currently actively working | Active work | #2196F3 (blue) |
| **Blocked** | Waiting on something/someone | Blocked, needs resolution | #FF9800 (orange) |
| **Completed** | Task finished successfully | Done, sets completed_at | #4CAF50 (green) |

**Properties:**
- `is_default = TRUE` (cannot be deleted or modified)
- Color provided for UI display
- Default state for new tasks: "Pending"

### 8.2 Custom Task States

Users can create custom states beyond the default four.

**Custom State Creation:**
```
User creates: "In Review"
System generates:
{
  id: NEW UUID
  user_id: current_user_id
  name: "In Review"
  is_default: FALSE
  color: #9C27B0 (purple) -- user selects
  created_at: NOW()
}
```

**Custom State Rules:**
1. Max 20 custom states per user
2. Name must be unique per user (case-insensitive)
3. Cannot edit default states (Pending, In Progress, Blocked, Completed)
4. Can delete custom states ONLY if no tasks reference them
5. Each state can have optional color (HEX format)

### 8.3 State Transitions & Flow Control

**State Transition Rules:**
- No restrictions: Task can transition from ANY state to ANY other state
- No required flow: User has full control of workflow
- State changes tracked: `updated_at` timestamp updated on each change

**Example Workflow (Flexible):**
```
Scenario 1: Linear flow
Pending → In Progress → Completed

Scenario 2: Non-linear (with blocking)
Pending → In Progress → Blocked → In Progress → Completed

Scenario 3: Custom flow
Pending → In Review → Pending → In Progress → Completed

Scenario 4: Dead task
Pending → Blocked (abandoned, never completed)
```

### 8.4 Completed State & Timestamp Logic

**When task moves to "Completed" state:**
```sql
UPDATE tasks 
SET 
  status_id = (SELECT id FROM task_states WHERE name = 'Completed' AND user_id = $1),
  completed_at = NOW(),
  updated_at = NOW()
WHERE id = $2;
```

**When task moves AWAY from "Completed" state (reopen):**
```sql
UPDATE tasks 
SET 
  status_id = $1,  -- new state ID
  completed_at = NULL,  -- clear timestamp
  updated_at = NOW()
WHERE id = $2;
```

**Rules:**
1. Only "Completed" state can set `completed_at`
2. Moving to any other state from "Completed" clears `completed_at`
3. `completed_at` used for analytics (completion time, on-time rate)
4. `completed_at` never automatically set; always user-initiated

### 8.5 State Deletion Safety

**When user attempts to delete a custom state:**

```sql
-- Check if state is in use
SELECT COUNT(*) FROM tasks 
WHERE status_id = $1 AND deleted_at IS NULL;

If count > 0:
  ↓ Return error:
  "Cannot delete state 'In Review' - X tasks still use it.
   Reassign tasks first or choose a different state."
  
If count = 0:
  ↓ Allow deletion
  DELETE FROM task_states WHERE id = $1;
```

**Reassignment Flow:**
```
User deletes "In Review" state (currently unused)
  ↓
System confirms deletion
  ↓
State removed from list
  ↓
Tasks with this state can be reassigned if any exist
```

---

## 9. Tags & Color Coding

### 9.1 Tag Structure

**Tag Attributes:**
```
Tag {
  id: UUID                    -- Unique identifier
  user_id: UUID               -- Owner (Foreign Key to users)
  name: VARCHAR(50)           -- Tag name (e.g., "Work", "Urgent", "Learning")
  color: VARCHAR(7)           -- HEX color (e.g., "#FF5733")
  created_at: TIMESTAMP       -- When created
  updated_at: TIMESTAMP       -- When last modified
}
```

**Tag Naming Rules:**
1. Max 50 characters
2. Unique per user (cannot have two tags with same name)
3. Case-insensitive matching (e.g., "work" = "Work" = "WORK")
4. Whitespace trimmed (e.g., " work " → "work")
5. No special characters except hyphen and underscore

**Valid Examples:**
- "Work", "Personal", "Urgent", "Learning", "Bug-Fix", "In_Review"

**Invalid Examples:**
- "Work@123" (special char @)
- "Very Long Tag Name For Everything" (>50 chars)
- "" (empty)

### 9.2 Color Coding System

**Color Assignment:**
- User manually selects HEX color when creating/editing tag
- UI provides color picker (standard hex input)
- Color persists with tag

**Color Format:**
- Standard 6-digit HEX: #RRGGBB (e.g., #FF5733)
- Validation: Regex /^#[0-9A-F]{6}$/i

**Color Examples:**
```
#FF5733 (red-orange)     - Urgent
#2196F3 (blue)           - Work
#4CAF50 (green)          - Done/Complete
#FFC107 (amber)          - In Progress
#9C27B0 (purple)         - Personal
#00BCD4 (cyan)           - Learning
#FF1744 (pink)           - Blocked
#795548 (brown)          - Research
```

### 9.3 Many-to-Many Relationship

**Junction Table: task_tags**
```sql
CREATE TABLE task_tags (
  id UUID PRIMARY KEY,
  task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  created_at TIMESTAMP,
  UNIQUE(task_id, tag_id)  -- Prevent duplicate tag assignments
);
```

**Relationship Rules:**
1. Task can have 0 to N tags
2. Tag can be applied to 0 to M tasks
3. Cannot assign same tag twice to same task (UNIQUE constraint)
4. Deleting tag cascades to delete all task_tags entries
5. Deleting task cascades to delete all task_tags entries

### 9.4 Tag Management Operations

**Create Tag:**
```
POST /api/v1/tags
{
  "name": "Work",
  "color": "#2196F3"
}
```

**Assign Tag to Task:**
```
PATCH /api/v1/tasks/{task_id}
{
  "tags": ["tag-id-1", "tag-id-2"]
}
```

**Modify Tag (Name or Color):**
```
PATCH /api/v1/tags/{tag_id}
{
  "name": "Work-Backend",
  "color": "#1976D2"
}

Note: Changes apply to ALL tasks using this tag (cascading update)
```

**Remove Tag from Task:**
```
DELETE /api/v1/tasks/{task_id}/tags/{tag_id}
```

**Delete Tag (if unused):**
```
DELETE /api/v1/tags/{tag_id}

Check first: SELECT COUNT(*) FROM task_tags WHERE tag_id = $1
If count > 0: Return error "Tag in use by X tasks"
If count = 0: Allow deletion
```

### 9.5 Tag UI Presentation

**Tag Display in Task List:**
```
Task Title
┌─────────────────────────────────────┐
│ [Work] [Urgent] [Backend]           │
│  #2196F3  #FF5733  #1976D2          │
└─────────────────────────────────────┘

Visual: Colored pills/badges with white text
Interaction: Click tag to filter by that tag
```

**Tag Autocomplete:**
```
When user creates/edits task, typing in tag field:
  "work" → Suggests: [Work], [Workout], [Working-on]
  
User selects existing tag → Tag added with its color
User types new tag → System creates new tag on save
```

### 9.6 Tag Analytics

**Dashboard Widget: Tag Usage**
```
Tag Usage Distribution:
- Work: 45 tasks (55%)
- Personal: 25 tasks (30%)
- Urgent: 12 tasks (15%)

Pie chart showing tag distribution
```

**Filter by Tag:**
```
User clicks tag in UI
  ↓
Filter applied: Show all tasks with this tag
  ↓
Can combine with other filters (AND logic)
  Example: Show tasks tagged "Work" AND priority "High"
```

---

## 10. Filtering & Sorting

### 10.1 Filter Types & Options

**1. Status Filter (Multi-select)**
- Options: All user's custom + default states
- Logic: OR (show tasks in ANY selected state)
- UI: Checkbox list or dropdown
- Example:
  ```
  Selected: [In Progress] [Blocked]
  Shows: All tasks with status "In Progress" OR "Blocked"
  ```

**2. Priority Filter (Multi-select)**
- Options: All user's custom + default priorities
- Logic: OR (show tasks with ANY selected priority)
- UI: Checkbox list with color badges
- Example:
  ```
  Selected: [High] [Critical]
  Shows: All tasks with priority "High" OR "Critical"
  ```

**3. Tags Filter (Multi-select + Logic)**
- Options: All user's created tags
- Logic: Configurable
  - "Contains any" (OR): Show tasks with ANY selected tag
  - "Contains all" (AND): Show tasks with ALL selected tags
- UI: Tag selector with toggle for logic
- Examples:
  ```
  Selected: [Work] [Backend]
  Logic: Contains any
  Shows: Tasks tagged "Work" OR "Backend"
  
  Selected: [Work] [Urgent]
  Logic: Contains all
  Shows: Tasks tagged BOTH "Work" AND "Urgent"
  ```

**4. Date Range Filter**
- Options: Custom date picker (start + end)
- Logic: `start_date ≤ due_date ≤ end_date`
- Presets:
  - "Today" (today's date only)
  - "This week" (Mon-Sun)
  - "This month" (1st-last day)
  - "Overdue" (due_date < TODAY)
  - "Custom" (user picks dates)
- Example:
  ```
  Selected: Apr 22 - Apr 29
  Shows: Tasks with due_date between Apr 22 and Apr 29
  ```

**5. Completion Status Filter**
- Options: 
  - "Pending" (not completed)
  - "Completed" (marked complete)
  - "All" (both)
- Logic: Boolean
- Example:
  ```
  Selected: "Pending"
  Shows: All tasks where status != "Completed" AND deleted_at IS NULL
  ```

**6. Time Estimate Filter (Range)**
- Options: Predefined ranges or custom min/max
- Presets:
  - "Quick wins" (0-30 mins)
  - "Short" (30-60 mins)
  - "Medium" (1-2 hours)
  - "Long" (2+ hours)
  - "Custom" (user sets range)
- Logic: `min_estimate ≤ time_estimate_minutes ≤ max_estimate`
- Example:
  ```
  Selected: "Quick wins" (0-30 mins)
  Shows: Tasks that take ≤ 30 minutes
  ```

### 10.2 Sorting Options

**1. Due Date (Ascending - Default)**
- Order: Earliest due date first
- Use: See urgent deadlines first
- Nulls: Tasks without due dates appear last
- Example:
  ```
  1. Apr 22, 10 AM
  2. Apr 23, 5 PM
  3. Apr 25, 2 PM
  4. (no due date)
  ```

**2. Priority (Descending)**
- Order: Highest priority weight first
- Use: Tackle most important tasks first
- Nulls: Unpriorized tasks appear last
- Example:
  ```
  1. [Critical] (weight 4)
  2. [High] (weight 3)
  3. [Medium] (weight 2)
  4. [Low] (weight 1)
  5. (no priority)
  ```

**3. Created Date (Descending)**
- Order: Newest tasks first
- Use: See recently added tasks
- Example:
  ```
  1. Created today
  2. Created yesterday
  3. Created last week
  ```

**4. Time Estimate (Ascending)**
- Order: Quickest tasks first
- Use: Identify quick wins for momentum
- Nulls: Unestimated tasks appear last
- Example:
  ```
  1. 15 mins
  2. 30 mins
  3. 90 mins
  4. 240 mins
  5. (no estimate)
  ```

**5. Custom Order (User drag-and-drop)**
- Order: User-defined via `display_order` field
- Use: Manual prioritization override
- Persistence: Saved in DB
- Example:
  ```
  User drags "Task C" to top
  ↓
  display_order updated: C=1, A=2, B=3
  ↓
  Tasks sorted by display_order
  ```

### 10.3 Filter Combination Logic

**Multiple filters work with AND logic:**

```
Example filters applied:
- Status: "In Progress"
- Priority: "High"
- Tags: "Work" (contains any)
- Date: Apr 22-29
- Estimate: 0-60 mins

Query:
WHERE status = "In Progress"
  AND priority = "High"
  AND (tag = "Work")
  AND (due_date BETWEEN Apr 22 AND Apr 29)
  AND (time_estimate_minutes BETWEEN 0 AND 60)
  AND deleted_at IS NULL

Result: Only tasks matching ALL criteria
```

### 10.4 Filter Persistence & Session Storage

**Session-Level Persistence:**
- Filters persist while user is on task list page
- Stored in browser session (localStorage or URL params)
- When user navigates away and returns: filters cleared
- Page refresh: filters retained

**URL Encoding (for sharing):**
```
https://orbit.app/tasks?
  status=in_progress,blocked
  &priority=high
  &tags=work&tag_logic=or
  &date_from=2026-04-22
  &date_to=2026-04-29
  &estimate_min=0
  &estimate_max=60
  &sort_by=due_date
  &sort_order=asc
```

**Clear Filters:**
- One-click "Clear All" button resets all filters
- Confirmation: "Clear all filters?" (optional)
- Returns to default view (all tasks, sorted by due date)

### 10.5 API Query Format

**Endpoint:** `GET /api/v1/tasks`

**Full Query Example:**
```
GET /api/v1/tasks?
  status=pending,in_progress
  &priority=high,medium
  &tags=work,backend&tag_logic=contains_all
  &date_from=2026-04-22
  &date_to=2026-04-29
  &time_estimate_min=0
  &time_estimate_max=120
  &completion=pending
  &sort_by=due_date
  &sort_order=asc
  &page=1
  &limit=20

Response:
{
  "data": [
    {
      "id": "uuid1",
      "title": "Fix login bug",
      "priority": "High",
      "status": "In Progress",
      "due_date": "2026-04-24T17:00:00Z",
      "time_estimate_minutes": 90,
      "tags": ["Work", "Backend"],
      ...
    },
    ...
  ],
  "pagination": {
    "total": 45,
    "page": 1,
    "limit": 20,
    "total_pages": 3
  }
}
```

**Query Parameter Reference:**
| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| status | string (comma-separated) | State IDs or names | pending,in_progress |
| priority | string (comma-separated) | Priority IDs or names | high,medium |
| tags | string (comma-separated) | Tag IDs or names | work,backend |
| tag_logic | string | AND or OR logic | contains_all, contains_any |
| date_from | ISO 8601 | Start date filter | 2026-04-22 |
| date_to | ISO 8601 | End date filter | 2026-04-29 |
| time_estimate_min | integer | Min minutes | 0 |
| time_estimate_max | integer | Max minutes | 120 |
| completion | string | pending/completed/all | pending |
| sort_by | string | Sort field | due_date, priority, created_at, time_estimate, custom_order |
| sort_order | string | asc or desc | asc |
| page | integer | Page number (1-indexed) | 1 |
| limit | integer | Results per page | 20 |

---

## 11. Bulk Operations

### 11.1 Bulk Edit

**Capability:** Select multiple tasks and edit common properties simultaneously.

**Editable Properties:**
- ✅ priority
- ✅ status
- ✅ tags
- ✅ due_date
- ✅ time_estimate_minutes

**Non-Editable (too risky):**
- ❌ title (too specific to each task)
- ❌ description (too specific)
- ❌ attachments (different files per task)

**UI Flow:**
```
1. User selects multiple tasks via checkboxes
2. System shows "5 tasks selected" counter
3. User clicks "Edit" button
4. Dialog opens: "Edit 5 tasks"
5. User selects which properties to change:
   - Priority: High → [dropdown]
   - Status: Completed → [dropdown]
   - Tags: + Add tag → [tag selector]
   - Due date: 2026-05-01 → [date picker]
   - Time estimate: 60 mins → [input]
6. Preview shows: "Update 5 tasks with these changes?"
7. User confirms: "Update"
8. Backend updates all selected tasks
9. Toast notification: "Updated 5 tasks"
```

### 11.2 Bulk Change Status

**Capability:** Move multiple tasks to same status.

**Example:**
```
User selects: [Task 1], [Task 2], [Task 3] (all Pending)
Action: "Move to In Progress"
Result: All 3 tasks moved to "In Progress" status
```

**Implementation:**
```
POST /api/v1/tasks/bulk-actions
{
  "task_ids": ["uuid1", "uuid2", "uuid3"],
  "action": "change_status",
  "payload": {
    "status_id": "uuid-in-progress-state"
  }
}

Backend:
1. Validate all task_ids exist and belong to current user
2. Validate status_id is valid for user
3. Update: SET status_id = $1, updated_at = NOW() WHERE id IN (...)
4. Set completed_at = NOW() if moving to Completed state
5. Return: { success: true, updated_count: 3 }
```

### 11.3 Bulk Delete

**Capability:** Soft delete multiple tasks with confirmation.

**Example:**
```
User selects: [Task 1], [Task 2], [Task 3]
Action: "Delete"
Confirmation: "Delete 3 tasks? They will be recoverable for 30 days."
Result: All 3 tasks marked deleted (deleted_at set)
```

**Implementation:**
```
POST /api/v1/tasks/bulk-actions
{
  "task_ids": ["uuid1", "uuid2", "uuid3"],
  "action": "delete",
  "payload": {}
}

Backend:
1. Validate all task_ids exist and belong to current user
2. Soft-delete: SET deleted_at = NOW(), updated_at = NOW() WHERE id IN (...)
3. Also soft-delete all subtasks of deleted tasks
4. Log deletion for audit trail
5. Return: { success: true, deleted_count: 3, recoverable_until: "2026-05-22" }
```

### 11.4 Bulk Add Tags

**Capability:** Add same tags to multiple tasks.

**Example:**
```
User selects: [Task 1], [Task 2], [Task 3]
Action: "Add tags"
Tags to add: [Work], [Urgent]
Result: Both tags added to all 3 tasks
```

**Implementation:**
```
POST /api/v1/tasks/bulk-actions
{
  "task_ids": ["uuid1", "uuid2", "uuid3"],
  "action": "add_tags",
  "payload": {
    "tag_ids": ["tag-work-uuid", "tag-urgent-uuid"]
  }
}

Backend:
1. Validate all task_ids and tag_ids
2. For each task_id:
   - Insert into task_tags: (task_id, tag_id1), (task_id, tag_id2)
   - Ignore if already exists (ON CONFLICT DO NOTHING)
3. Return: { success: true, updated_count: 3 }
```

### 11.5 Bulk Operations Endpoint

**Main Endpoint:**
```
POST /api/v1/tasks/bulk-actions

Request:
{
  "task_ids": ["uuid1", "uuid2", "uuid3"],
  "action": "update|delete|change_status|add_tags|remove_tags|change_priority",
  "payload": {
    // Depends on action type
  }
}

Response:
{
  "success": true,
  "action": "update",
  "affected_count": 5,
  "details": {
    "updated": 5,
    "skipped": 0,
    "errors": 0
  },
  "message": "Successfully updated 5 tasks"
}
```

### 11.6 Bulk Operations Validation

**Validation Rules:**
1. task_ids array must have 1-500 items
2. All task_ids must belong to current user
3. Cannot bulk-operate on soft-deleted tasks
4. Cannot exceed rate limits (max 100 bulk ops per minute per user)
5. Payload validation depends on action type

**Error Handling:**
```
If validation fails:
{
  "success": false,
  "error": "bulk_validation_failed",
  "message": "Task uuid1 not found or not owned by user",
  "affected_count": 0
}

If partial success:
{
  "success": true,
  "affected_count": 3,
  "warnings": ["Task uuid4 already has tag 'Work'"]
}
```

### 11.7 Bulk Operations UI Patterns

**Selection:**
```
☑ [Select All] checkbox at top of list
  ☑ Task 1
  ☐ Task 2
  ☑ Task 3
  ☐ Task 4
  ☑ Task 5

Display: "3 tasks selected"
Options: [Edit] [Delete] [Move] [Add Tags]
```

**Confirmation Dialog:**
```
┌─────────────────────────────┐
│ Update 3 tasks?             │
├─────────────────────────────┤
│ Changes:                    │
│ • Status: In Progress       │
│ • Priority: High            │
│ • Tags: + Work, + Urgent    │
│                             │
│ [Cancel] [Update]           │
└─────────────────────────────┘
```

**Success Message:**
```
✅ Updated 3 tasks
   Undo (expires in 5 mins) / Dismiss
```

---

## 12. Complete PostgreSQL Database Schema

### 12.1 Users Table

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(100) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    timezone VARCHAR(50) DEFAULT 'UTC',
    telegram_id VARCHAR(100),
    whatsapp_number VARCHAR(20),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_email CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'),
    CONSTRAINT valid_timezone CHECK (timezone IN (
        'UTC', 'EST', 'CST', 'MST', 'PST', 'GMT', 'CET', 'IST', 'JST', 'AEST'
        -- Add all supported timezones
    ))
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_deleted_at ON users(deleted_at);
```

### 12.2 Priorities Table

```sql
CREATE TABLE priorities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL,
    weight INTEGER NOT NULL CHECK (weight > 0 AND weight <= 100),
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(user_id, name)
);

CREATE INDEX idx_priorities_user_id ON priorities(user_id);

-- Trigger: Auto-create default priorities on new user
CREATE FUNCTION create_default_priorities()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO priorities (user_id, name, weight, is_default)
    VALUES 
        (NEW.id, 'Low', 1, TRUE),
        (NEW.id, 'Medium', 2, TRUE),
        (NEW.id, 'High', 3, TRUE);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_create_default_priorities
AFTER INSERT ON users
FOR EACH ROW EXECUTE FUNCTION create_default_priorities();
```

### 12.3 Task States Table

```sql
CREATE TABLE task_states (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL,
    is_default BOOLEAN DEFAULT FALSE,
    color VARCHAR(7),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(user_id, name),
    CONSTRAINT valid_color CHECK (color ~* '^#[0-9A-F]{6}$')
);

CREATE INDEX idx_task_states_user_id ON task_states(user_id);

-- Trigger: Auto-create default states on new user
CREATE FUNCTION create_default_task_states()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO task_states (user_id, name, is_default, color)
    VALUES 
        (NEW.id, 'Pending', TRUE, '#E0E0E0'),
        (NEW.id, 'In Progress', TRUE, '#2196F3'),
        (NEW.id, 'Blocked', TRUE, '#FF9800'),
        (NEW.id, 'Completed', TRUE, '#4CAF50');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_create_default_task_states
AFTER INSERT ON users
FOR EACH ROW EXECUTE FUNCTION create_default_task_states();
```

### 12.4 Tags Table

```sql
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL,
    color VARCHAR(7) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(user_id, LOWER(name)),
    CONSTRAINT valid_color CHECK (color ~* '^#[0-9A-F]{6}$')
);

CREATE INDEX idx_tags_user_id ON tags(user_id);
CREATE INDEX idx_tags_user_name ON tags(user_id, LOWER(name));
```

### 12.5 Tasks Table (Main)

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Core content
    title VARCHAR(255) NOT NULL,
    description TEXT,
    
    -- Relationships
    priority_id UUID REFERENCES priorities(id) ON DELETE SET NULL,
    status_id UUID NOT NULL REFERENCES task_states(id) ON DELETE RESTRICT,
    parent_task_id UUID REFERENCES tasks(id) ON DELETE CASCADE,
    
    -- Timing
    due_date TIMESTAMP WITH TIME ZONE,
    time_estimate_minutes INTEGER CHECK (time_estimate_minutes IS NULL OR time_estimate_minutes > 0),
    
    -- Recurrence
    recurrence_rule VARCHAR(255),
    recurrence_series_id UUID,
    is_recurring_instance BOOLEAN DEFAULT FALSE,
    
    -- External Integration
    gcal_event_id VARCHAR(255),
    
    -- Status Tracking
    completed_at TIMESTAMP WITH TIME ZONE,
    display_order INTEGER DEFAULT 0,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    -- Constraints
    CONSTRAINT no_self_parent CHECK (parent_task_id != id),
    CONSTRAINT valid_recurrence CHECK (
        (recurrence_rule IS NULL AND recurrence_series_id IS NULL) OR
        (recurrence_rule IS NOT NULL)
    ),
    CONSTRAINT no_past_due_date CHECK (due_date IS NULL OR due_date >= CURRENT_TIMESTAMP)
);

CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_tasks_status_id ON tasks(status_id);
CREATE INDEX idx_tasks_priority_id ON tasks(priority_id);
CREATE INDEX idx_tasks_parent_task_id ON tasks(parent_task_id);
CREATE INDEX idx_tasks_due_date ON tasks(due_date);
CREATE INDEX idx_tasks_deleted_at ON tasks(deleted_at);
CREATE INDEX idx_tasks_recurrence_series_id ON tasks(recurrence_series_id);
CREATE INDEX idx_tasks_user_created ON tasks(user_id, created_at DESC);
CREATE INDEX idx_tasks_user_due ON tasks(user_id, due_date);
CREATE INDEX idx_tasks_user_status ON tasks(user_id, status_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_tasks_user_priority ON tasks(user_id, priority_id) WHERE deleted_at IS NULL;
```

### 12.6 Task Tags Junction Table

```sql
CREATE TABLE task_tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(task_id, tag_id)
);

CREATE INDEX idx_task_tags_task_id ON task_tags(task_id);
CREATE INDEX idx_task_tags_tag_id ON task_tags(tag_id);
```

### 12.7 Attachments Table

```sql
CREATE TABLE attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    file_url VARCHAR(500) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    file_size_bytes INTEGER CHECK (file_size_bytes > 0 AND file_size_bytes <= 10485760),
    mime_type VARCHAR(100),
    uploaded_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    uploaded_by UUID REFERENCES users(id) ON DELETE SET NULL,
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_mime_type CHECK (mime_type IN (
        'image/jpeg', 'image/png', 'image/gif',
        'application/pdf',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        'application/vnd.openxmlformats-officedocument.presentationml.presentation',
        'text/plain'
    ))
);

CREATE INDEX idx_attachments_task_id ON attachments(task_id);
CREATE INDEX idx_attachments_uploaded_by ON attachments(uploaded_by);

-- Trigger: Enforce max 5 attachments per task
CREATE FUNCTION check_attachment_limit()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT COUNT(*) FROM attachments 
        WHERE task_id = NEW.task_id 
        AND deleted_at IS NULL) > 5 THEN
        RAISE EXCEPTION 'Maximum 5 attachments per task';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_check_attachment_limit
BEFORE INSERT ON attachments
FOR EACH ROW EXECUTE FUNCTION check_attachment_limit();
```

### 12.8 Summary: All Indices

```sql
-- Users
idx_users_email
idx_users_username
idx_users_deleted_at

-- Priorities
idx_priorities_user_id

-- Task States
idx_task_states_user_id

-- Tags
idx_tags_user_id
idx_tags_user_name

-- Tasks (most critical)
idx_tasks_user_id
idx_tasks_status_id
idx_tasks_priority_id
idx_tasks_parent_task_id
idx_tasks_due_date
idx_tasks_deleted_at
idx_tasks_recurrence_series_id
idx_tasks_user_created
idx_tasks_user_due
idx_tasks_user_status
idx_tasks_user_priority

-- Task Tags
idx_task_tags_task_id
idx_task_tags_tag_id

-- Attachments
idx_attachments_task_id
idx_attachments_uploaded_by
```

---

## 13. All API Endpoints

### 13.1 Task CRUD Operations

```
POST   /api/v1/tasks
Create new task

Request:
{
  "title": "Fix login bug",
  "description": "Users unable to login with Google OAuth",
  "priority_id": "uuid-high",
  "status_id": "uuid-pending",
  "due_date": "2026-04-25T17:00:00Z",
  "time_estimate_minutes": 120,
  "tag_ids": ["uuid-tag1", "uuid-tag2"]
}

Response:
{
  "id": "new-task-uuid",
  "title": "Fix login bug",
  "created_at": "2026-04-22T10:00:00Z",
  ...
}
```

```
GET    /api/v1/tasks
List all tasks with filtering, sorting, pagination

Query Parameters:
- status: comma-separated state IDs
- priority: comma-separated priority IDs
- tags: comma-separated tag IDs
- tag_logic: "contains_any" or "contains_all"
- date_from, date_to: ISO dates
- time_estimate_min, time_estimate_max: minutes
- completion: "pending", "completed", "all"
- sort_by: "due_date", "priority", "created_at", "time_estimate", "custom_order"
- sort_order: "asc" or "desc"
- page, limit: pagination

Response:
{
  "data": [
    { "id": "uuid1", "title": "...", ... },
    { "id": "uuid2", "title": "...", ... }
  ],
  "pagination": {
    "total": 45,
    "page": 1,
    "limit": 20,
    "total_pages": 3
  }
}
```

```
GET    /api/v1/tasks/:id
Get single task details (including subtasks, attachments, tags)

Response:
{
  "id": "task-uuid",
  "title": "Fix login bug",
  "priority": { "id": "...", "name": "High" },
  "status": { "id": "...", "name": "In Progress" },
  "tags": [{ "id": "...", "name": "Work", "color": "#2196F3" }],
  "subtasks": [
    { "id": "...", "title": "...", "status": "..." },
    ...
  ],
  "attachments": [
    { "id": "...", "file_name": "...", "file_url": "...", ... },
    ...
  ],
  "due_date": "2026-04-25T17:00:00Z",
  "time_estimate_minutes": 120,
  "completed_at": null,
  "created_at": "2026-04-22T10:00:00Z",
  "updated_at": "2026-04-22T10:00:00Z"
}
```

```
PATCH  /api/v1/tasks/:id
Update task (any field)

Request:
{
  "title": "Fix critical login bug",
  "priority_id": "uuid-critical",
  "status_id": "uuid-in-progress",
  "due_date": "2026-04-24T17:00:00Z"
}

Response:
{
  "id": "task-uuid",
  "title": "Fix critical login bug",
  ...
  "updated_at": "2026-04-22T10:30:00Z"
}
```

```
DELETE /api/v1/tasks/:id
Soft delete task (marks deleted_at, cascades to subtasks)

Response:
{
  "id": "task-uuid",
  "deleted_at": "2026-04-22T10:45:00Z",
  "recoverable_until": "2026-05-22T10:45:00Z"
}
```

```
POST   /api/v1/tasks/:id/restore
Restore soft-deleted task

Response:
{
  "id": "task-uuid",
  "deleted_at": null,
  "message": "Task restored successfully"
}
```

### 13.2 Task Status & Completion

```
POST   /api/v1/tasks/:id/complete
Mark task as completed (moves to Completed state, sets completed_at)

Request:
{
  "complete_subtasks": true  // optional: complete all subtasks too
}

Response:
{
  "id": "task-uuid",
  "status": { "name": "Completed" },
  "completed_at": "2026-04-22T10:50:00Z"
}
```

```
POST   /api/v1/tasks/:id/reopen
Reopen completed task (move away from Completed state)

Request:
{
  "status_id": "uuid-pending"
}

Response:
{
  "id": "task-uuid",
  "status": { "name": "Pending" },
  "completed_at": null
}
```

```
POST   /api/v1/tasks/:id/change-status
Change task status

Request:
{
  "status_id": "uuid-in-progress"
}

Response:
{
  "id": "task-uuid",
  "status": { "name": "In Progress" },
  "updated_at": "2026-04-22T10:55:00Z"
}
```

### 13.3 Subtask Operations

```
POST   /api/v1/tasks/:id/subtasks
Create subtask under parent

Request:
{
  "title": "Research requirements",
  "priority_id": "uuid-high",
  "due_date": "2026-04-24T10:00:00Z"
}

Response:
{
  "id": "new-subtask-uuid",
  "title": "Research requirements",
  "parent_task_id": "parent-uuid",
  ...
}
```

```
GET    /api/v1/tasks/:id/subtasks
List all subtasks of parent

Response:
{
  "data": [
    { "id": "...", "title": "Research requirements", "status": "Pending" },
    { "id": "...", "title": "Design visuals", "status": "In Progress" },
    { "id": "...", "title": "Final review", "status": "Pending" }
  ],
  "progress": "1 of 3 complete (33%)"
}
```

```
PATCH  /api/v1/tasks/:subtask_id/promote
Promote subtask to root (remove parent)

Response:
{
  "id": "subtask-uuid",
  "parent_task_id": null,
  "message": "Subtask promoted to root task"
}
```

### 13.4 Recurring Tasks

```
POST   /api/v1/tasks/:id/recurrence
Set recurrence pattern

Request:
{
  "recurrence_rule": "FREQ=WEEKLY;BYDAY=MO,WE,FR",
  "start_date": "2026-04-22",
  "end_date": "2027-04-22"  // optional
}

Response:
{
  "id": "task-uuid",
  "recurrence_rule": "FREQ=WEEKLY;BYDAY=MO,WE,FR",
  "recurrence_series_id": "series-uuid",
  "instances_created": 52,
  "next_instances": [
    { "id": "...", "due_date": "2026-04-27" },
    { "id": "...", "due_date": "2026-04-29" },
    ...
  ]
}
```

```
DELETE /api/v1/tasks/:id/recurrence
Remove recurrence pattern (stop generating new instances)

Response:
{
  "id": "task-uuid",
  "recurrence_rule": null,
  "message": "Recurrence removed. Existing instances remain."
}
```

```
GET    /api/v1/recurring-series/:series_id
Get all instances in a recurring series

Response:
{
  "series_id": "series-uuid",
  "recurrence_rule": "FREQ=DAILY",
  "instances": [
    { "id": "...", "due_date": "2026-04-22", "status": "Pending" },
    { "id": "...", "due_date": "2026-04-23", "status": "Pending" },
    { "id": "...", "due_date": "2026-04-24", "status": "Completed" },
    ...
  ]
}
```

### 13.5 Bulk Operations

```
POST   /api/v1/tasks/bulk-actions
Bulk edit, delete, or change status

Request:
{
  "task_ids": ["uuid1", "uuid2", "uuid3"],
  "action": "update",
  "payload": {
    "priority_id": "uuid-high",
    "status_id": "uuid-in-progress",
    "tag_ids": ["uuid-tag1", "uuid-tag2"],
    "due_date": "2026-05-01T17:00:00Z"
  }
}

Response:
{
  "success": true,
  "action": "update",
  "affected_count": 3,
  "message": "Updated 3 tasks",
  "updated_at": "2026-04-22T11:00:00Z"
}
```

### 13.6 Tags Management

```
POST   /api/v1/tags
Create new tag

Request:
{
  "name": "Work",
  "color": "#2196F3"
}

Response:
{
  "id": "tag-uuid",
  "name": "Work",
  "color": "#2196F3",
  "created_at": "2026-04-22T11:05:00Z"
}
```

```
GET    /api/v1/tags
List all user's tags

Response:
{
  "data": [
    { "id": "...", "name": "Work", "color": "#2196F3", "task_count": 45 },
    { "id": "...", "name": "Personal", "color": "#4CAF50", "task_count": 23 },
    { "id": "...", "name": "Urgent", "color": "#FF5733", "task_count": 8 }
  ]
}
```

```
PATCH  /api/v1/tags/:id
Update tag name/color

Request:
{
  "name": "Work-Backend",
  "color": "#1976D2"
}

Response:
{
  "id": "tag-uuid",
  "name": "Work-Backend",
  "color": "#1976D2",
  "updated_at": "2026-04-22T11:10:00Z"
}
```

```
DELETE /api/v1/tags/:id
Delete tag (only if no tasks use it)

Response:
{
  "id": "tag-uuid",
  "message": "Tag deleted successfully"
}
```

### 13.7 Priorities Management

```
POST   /api/v1/priorities
Create custom priority

Request:
{
  "name": "Critical",
  "weight": 4
}

Response:
{
  "id": "priority-uuid",
  "name": "Critical",
  "weight": 4,
  "is_default": false,
  "created_at": "2026-04-22T11:15:00Z"
}
```

```
GET    /api/v1/priorities
List all priorities (default + custom)

Response:
{
  "data": [
    { "id": "...", "name": "Low", "weight": 1, "is_default": true },
    { "id": "...", "name": "Medium", "weight": 2, "is_default": true },
    { "id": "...", "name": "High", "weight": 3, "is_default": true },
    { "id": "...", "name": "Critical", "weight": 4, "is_default": false }
  ]
}
```

```
PATCH  /api/v1/priorities/:id
Update custom priority (weight only, cannot edit defaults)

Request:
{
  "weight": 5
}

Response:
{
  "id": "priority-uuid",
  "weight": 5,
  "updated_at": "2026-04-22T11:20:00Z"
}
```

```
DELETE /api/v1/priorities/:id
Delete custom priority (cannot delete defaults)

Response:
{
  "id": "priority-uuid",
  "message": "Priority deleted successfully"
}
```

### 13.8 Task States Management

```
POST   /api/v1/task-states
Create custom state

Request:
{
  "name": "In Review",
  "color": "#9C27B0"
}

Response:
{
  "id": "state-uuid",
  "name": "In Review",
  "color": "#9C27B0",
  "is_default": false,
  "created_at": "2026-04-22T11:25:00Z"
}
```

```
GET    /api/v1/task-states
List all states (default + custom)

Response:
{
  "data": [
    { "id": "...", "name": "Pending", "is_default": true, "color": "#E0E0E0" },
    { "id": "...", "name": "In Progress", "is_default": true, "color": "#2196F3" },
    { "id": "...", "name": "Blocked", "is_default": true, "color": "#FF9800" },
    { "id": "...", "name": "Completed", "is_default": true, "color": "#4CAF50" },
    { "id": "...", "name": "In Review", "is_default": false, "color": "#9C27B0" }
  ]
}
```

```
PATCH  /api/v1/task-states/:id
Update custom state (name/color only, cannot edit defaults)

Request:
{
  "name": "Needs Review",
  "color": "#673AB7"
}

Response:
{
  "id": "state-uuid",
  "name": "Needs Review",
  "color": "#673AB7",
  "updated_at": "2026-04-22T11:30:00Z"
}
```

```
DELETE /api/v1/task-states/:id
Delete custom state (cannot delete defaults, must have 0 tasks)

Response:
{
  "id": "state-uuid",
  "message": "State deleted successfully"
}
```

### 13.9 Attachments

```
POST   /api/v1/tasks/:id/attachments
Upload attachment (usually via S3 pre-signed URL, then create metadata)

Request:
{
  "file_url": "s3://orbit-attachments/...",
  "file_name": "requirements.pdf",
  "file_size_bytes": 2048576,
  "mime_type": "application/pdf"
}

Response:
{
  "id": "attachment-uuid",
  "task_id": "task-uuid",
  "file_url": "s3://orbit-attachments/...",
  "file_name": "requirements.pdf",
  "uploaded_at": "2026-04-22T11:35:00Z"
}
```

```
GET    /api/v1/tasks/:id/attachments
List all attachments of task

Response:
{
  "data": [
    { "id": "...", "file_name": "requirements.pdf", "file_url": "...", "file_size_bytes": 2048576 },
    { "id": "...", "file_name": "screenshot.png", "file_url": "...", "file_size_bytes": 512000 }
  ]
}
```

```
DELETE /api/v1/attachments/:id
Delete attachment (soft or hard delete)

Response:
{
  "id": "attachment-uuid",
  "message": "Attachment deleted successfully"
}
```

---

## 14. Business Logic & Edge Cases

### 14.1 Subtask Completion Logic

**Scenario 1: User marks subtask complete**
```
Input: PATCH /api/v1/tasks/{subtask_id} with status_id = "Completed"

System:
1. Update subtask: status_id = Completed, completed_at = NOW()
2. Query parent task: SELECT * FROM tasks WHERE id = parent_task_id
3. Calculate: completed_subtasks = count(WHERE status_id = Completed)
4. Calculate: total_subtasks = count(ALL)
5. If completed_subtasks == total_subtasks:
   → Show notification: "All subtasks done! Complete parent task?"
   → User can click YES or DISMISS
   → If YES: Mark parent as Completed too
   → If DISMISS: Parent stays in current state
6. Update parent: completed_count, total_count for progress display
7. Emit websocket event: "subtask_completed" for real-time UI update
```

**Scenario 2: User marks parent complete with incomplete subtasks**
```
Input: POST /api/v1/tasks/{parent_id}/complete

System:
1. Check: SELECT COUNT(*) FROM tasks WHERE parent_task_id = parent_id AND status_id != Completed
2. If incomplete_count > 0:
   → Show confirmation: "Complete all subtasks too? (X incomplete tasks)"
   → User chooses: YES / NO / CANCEL
   → If YES:
      - UPDATE tasks SET status_id = Completed, completed_at = NOW() WHERE parent_task_id = parent_id
      - UPDATE parent: status_id = Completed, completed_at = NOW()
   → If NO:
      - UPDATE parent: status_id = Completed, completed_at = NOW()
      - Subtasks stay unchanged
   → If CANCEL: Abort, no changes
3. If incomplete_count == 0:
   - Directly complete parent (no confirmation)
4. Emit events for all updated tasks
```

### 14.2 Recurring Task Generation Algorithm

**Initial Creation:**
```
POST /api/v1/tasks with recurrence_rule = "FREQ=WEEKLY;BYDAY=MO,WE,FR"

System:
1. Validate RRULE syntax using RFC 5545 parser
2. Generate recurrence_series_id = NEW UUID
3. Parse RRULE:
   - Frequency: WEEKLY
   - Days: Monday, Wednesday, Friday
   - Start: today
   - End: today + 12 months (or UNTIL if specified)
4. Generate instances:
   today = Monday:
     Instance 1: due_date = Mon, status = Pending, parent_series_id = series_uuid
   Wed:
     Instance 2: due_date = Wed, status = Pending, parent_series_id = series_uuid
   Fri:
     Instance 3: due_date = Fri, status = Pending, parent_series_id = series_uuid
   (next Monday, Wed, Fri, etc... up to 12 months)
5. Insert all instances into tasks table
6. Return: task_uuid, series_uuid, instances_count
```

**Lazy Instance Extension:**
```
Background job runs daily at 2 AM UTC:
1. Query all tasks WHERE is_recurring_instance = true AND due_date < TODAY + 1 day
2. For each series:
   - Get recurrence_rule and series_id
   - Check: last_instance_due_date vs. UNTIL date (if exists)
   - If last_instance < TODAY + 12 months:
     - Generate next N instances (next 30 days)
     - Insert into tasks table
     - Update last_generated_date
3. This keeps series "topped up" with always 12+ months of instances
```

### 14.3 Soft Delete & 30-Day Recovery

**Delete Flow:**
```
DELETE /api/v1/tasks/{id}

System:
1. Validate task_id exists and belongs to current_user
2. Soft delete:
   UPDATE tasks SET deleted_at = NOW() WHERE id = $1
3. Cascade soft delete subtasks:
   UPDATE tasks SET deleted_at = NOW() WHERE parent_task_id = $1 AND deleted_at IS NULL
4. Soft delete attachments:
   UPDATE attachments SET deleted_at = NOW() WHERE task_id = $1 AND deleted_at IS NULL
5. Log audit entry: user_id, action="delete", task_id, timestamp
6. Return: task_uuid, deleted_at, recoverable_until = deleted_at + 30 days
7. UI shows: "Deleted. Restore? (Expires in 30 days)"
```

**Recovery Flow:**
```
POST /api/v1/tasks/{id}/restore

System:
1. Validate task_id exists, belongs to current_user, deleted_at IS NOT NULL
2. Check: (NOW() - deleted_at) < 30 days
   → If FALSE: Return error "Task permanently deleted"
   → If TRUE: Proceed
3. Restore:
   UPDATE tasks SET deleted_at = NULL WHERE id = $1
4. Restore subtasks:
   UPDATE tasks SET deleted_at = NULL WHERE parent_task_id = $1
5. Restore attachments:
   UPDATE attachments SET deleted_at = NULL WHERE task_id = $1
6. Log audit entry
7. Return: task_uuid, deleted_at = null, message = "Task restored"
```

**Permanent Deletion (Background Job):**
```
Cleanup job runs daily at 3 AM UTC:
1. Query: SELECT * FROM tasks WHERE deleted_at < NOW() - INTERVAL '30 days'
2. For each task:
   - Get all attachment file_urls
   - For each attachment:
     - Call S3 API: DeleteObject(s3://bucket/path)
     - Wait for confirmation
   - Hard delete from DB:
     DELETE FROM task_tags WHERE task_id = $1
     DELETE FROM attachments WHERE task_id = $1
     DELETE FROM tasks WHERE id = $1 (cascades to subtasks)
   - Log: "Permanently deleted task_id, freed X MB, deleted N attachments"
3. Confirm all deletions in audit log
```

### 14.4 Default Creation on User Signup

**User Registration:**
```
POST /api/v1/auth/register
{
  "email": "user@example.com",
  "password": "...",
  "username": "john_doe"
}

System:
1. Validate input
2. Hash password
3. INSERT INTO users: (id, email, password_hash, username, timezone, created_at)
4. Triggers execute:
   - create_default_priorities(): Inserts Low, Medium, High
   - create_default_task_states(): Inserts Pending, In Progress, Blocked, Completed
5. User account ready with default priorities and states
6. Return: user_id, token, message = "Account created"
```

### 14.5 Validation Rules (50+)

**Task Validation:**
```sql
1. title: NOT NULL, LENGTH 1-255, TRIM whitespace
2. description: Optional, LENGTH 0-unlimited
3. due_date: Optional, CHECK due_date >= NOW() (not in past)
4. time_estimate_minutes: Optional, CHECK > 0 and <= 43200
5. priority_id: Optional, CHECK exists in priorities table for user
6. status_id: NOT NULL, CHECK exists in task_states table for user
7. parent_task_id: Optional, CHECK != id, CHECK parent is root (has no parent)
8. recurrence_rule: Optional, CHECK matches RFC 5545 RRULE pattern
9. gcal_event_id: Optional, LENGTH max 255
10. tags: Array, each tag_id must exist in tags table for user
```

**Subtask Validation:**
```sql
11. parent_task_id: NOT NULL for subtask
12. parent_task_id != id: CHECK constraint prevents self-reference
13. parent's parent_task_id: CHECK IS NULL (max 1 level nesting)
14. Cannot set parent_task_id to a subtask
15. Cannot create circular references (A → B → A)
```

**Recurring Validation:**
```sql
16. recurrence_rule: Must be valid RFC 5545 (parse and validate)
17. FREQ must be one of: DAILY, WEEKLY, MONTHLY, YEARLY
18. INTERVAL must be positive integer
19. BYDAY format: MO,TU,WE,TH,FR,SA,SU (or -1MO for last Monday)
20. BYMONTHDAY: 1-31 or -1 to -31
21. BYMONTH: 1-12
22. UNTIL: If specified, must be after START date
23. COUNT: If specified, must be positive integer
```

**Attachment Validation:**
```sql
24. file_size_bytes: CHECK 0 < size <= 10485760
25. mime_type: CHECK in whitelist (image/jpeg, application/pdf, etc.)
26. file_url: CHECK is valid URL (starts with http:// or s3://)
27. file_name: NOT NULL, LENGTH 1-255
28. attachment_count per task: CHECK <= 5
29. total_attachment_size per task: CHECK <= 50 MB (5 × 10 MB)
```

**Priority/State Validation:**
```sql
30. priority name: NOT NULL, LENGTH 1-50, UNIQUE per user
31. priority weight: CHECK 1 <= weight <= 100
32. state name: NOT NULL, LENGTH 1-50, UNIQUE per user
33. state color: CHECK matches #RRGGBB pattern
34. Cannot modify is_default priorities/states
35. Cannot delete priorities/states if tasks reference them
```

**Tag Validation:**
```sql
36. tag name: NOT NULL, LENGTH 1-50, UNIQUE per user (case-insensitive)
37. tag color: CHECK matches #RRGGBB pattern
38. Cannot assign same tag twice to task (UNIQUE constraint in junction)
39. tag_count per task: No hard limit (but recommend < 10)
40. tag_count per user: Recommend < 100 for performance
```

**Filter/Sort Validation:**
```sql
41. sort_by: Must be one of allowed fields (due_date, priority, created_at, etc.)
42. sort_order: Must be "asc" or "desc"
43. limit: CHECK 1 <= limit <= 100 (prevent excessive data transfer)
44. page: CHECK > 0
45. date_from, date_to: date_from <= date_to
46. time_estimate_min, max: min <= max
47. status filter: All status_ids must belong to user
48. priority filter: All priority_ids must belong to user
49. tag filter: All tag_ids must belong to user
50. tag_logic: Must be "contains_any" or "contains_all"
```

---

## 15. Business Logic - Data Integrity

### 15.1 Constraints & Triggers Summary

| Constraint | Type | Purpose |
|-----------|------|---------|
| no_self_parent | CHECK | Prevent task from being parent of itself |
| valid_email | CHECK | Validate email format |
| valid_timezone | CHECK | Ensure timezone is supported |
| valid_time_estimate | CHECK | Time estimate must be positive |
| valid_color | CHECK | Color must be #RRGGBB hex format |
| valid_mime_type | CHECK | Attachment MIME type in whitelist |
| valid_file_size | CHECK | Attachment size 1 byte - 10 MB |
| no_past_due_date | CHECK | Due date cannot be in past |
| valid_recurrence | CHECK | RRULE present if recurring |
| UNIQUE(user_id, name) | UNIQUE | Prevent duplicate priorities/states/tags per user |
| UNIQUE(task_id, tag_id) | UNIQUE | Prevent duplicate tag assignments |
| create_default_priorities | TRIGGER | Auto-create priorities on user signup |
| create_default_task_states | TRIGGER | Auto-create states on user signup |
| check_attachment_limit | TRIGGER | Enforce max 5 attachments per task |

### 15.2 Cascading Operations

```sql
When parent task is deleted:
  UPDATE tasks SET deleted_at = NOW() WHERE parent_task_id = $1

When tag is deleted:
  DELETE FROM task_tags WHERE tag_id = $1 (cascades via FK)

When priority is deleted:
  UPDATE tasks SET priority_id = NULL WHERE priority_id = $1

When state is deleted:
  -- Prevented by constraint if tasks exist
  RESTRICT DELETE (user must reassign first)

When user is deleted:
  DELETE FROM tasks WHERE user_id = $1 (cascades all related data)
```

---

## 16. Acceptance Criteria (50+)

### Feature: Task CRUD
- [ ] User can create task with title, description, priority, tags, due date, time estimate
- [ ] User can view task with all details including subtasks and attachments
- [ ] User can update any task property (title, description, priority, tags, due date, time estimate)
- [ ] User can soft-delete task (moved to trash, recoverable for 30 days)
- [ ] User can permanently delete task from trash
- [ ] User can restore deleted task before 30-day expiration

### Feature: Subtasks
- [ ] User can create subtask under any task
- [ ] Subtask appears indented in task list UI
- [ ] Parent task shows "X/Y subtasks complete" progress indicator
- [ ] User can complete subtask independently of parent
- [ ] User can promote subtask to root task (remove parent)
- [ ] Completing parent task prompts to complete all children
- [ ] Deleting parent task soft-deletes all subtasks

### Feature: Recurring Tasks
- [ ] User can set recurrence pattern (daily, weekly, monthly, yearly)
- [ ] System generates instances for next 12 months upon creation
- [ ] Completing one instance doesn't affect other instances
- [ ] Editing recurrence asks "Just this one?" or "This and future?"
- [ ] Next instance auto-generates when current reaches due_date

### Feature: Time Estimates
- [ ] User can set time estimate in minutes (15, 30, 60, 120, etc.)
- [ ] Estimate displayed as "Xh Ym" format (e.g., "2h 30m") in UI
- [ ] Total estimates shown in dashboard ("You have 8h of work today")
- [ ] Can sort by time estimate (quickest first)
- [ ] Can filter by time estimate range

### Feature: Attachments
- [ ] User can upload up to 5 files per task
- [ ] Max file size enforced at 10 MB
- [ ] Supported types validated (images, PDFs, docs)
- [ ] Attachment displays file name, size, upload date
- [ ] User can delete individual attachments
- [ ] Attachments deleted from cloud storage when task deleted
- [ ] Error handling if file upload fails

### Feature: Priorities
- [ ] Default priorities (Low, Medium, High) present for all users
- [ ] User can create custom priority level
- [ ] User can edit custom priority (name, weight)
- [ ] User can delete custom priority (reassign affected tasks first)
- [ ] Cannot delete default priorities
- [ ] Task defaults to no priority (optional)
- [ ] Sorting by priority works correctly (High → Medium → Low)

### Feature: Task States
- [ ] Default states (Pending, In Progress, Blocked, Completed) present for all users
- [ ] User can create custom state
- [ ] User can edit custom state (name, color)
- [ ] User can delete custom state (reassign affected tasks first)
- [ ] Cannot delete default states
- [ ] Task defaults to "Pending" state
- [ ] Completing task sets completed_at timestamp
- [ ] Reopening task clears completed_at

### Feature: Tags
- [ ] User can create tag with custom name and color
- [ ] Tag appears as colored pill in task UI
- [ ] User can assign multiple tags to same task
- [ ] User can remove tag from task
- [ ] User can edit tag name/color (updates all tasks using it)
- [ ] User can delete unused tags
- [ ] Tag name is unique per user

### Feature: Filtering & Sorting
- [ ] User can filter by: status (single or multi-select)
- [ ] User can filter by: priority (single or multi-select)
- [ ] User can filter by: tags (with AND/OR logic option)
- [ ] User can filter by: date range (custom date picker)
- [ ] User can filter by: completion status (pending/complete/all)
- [ ] User can filter by: time estimate range
- [ ] User can sort by: due date (ascending)
- [ ] User can sort by: priority (descending)
- [ ] User can sort by: created date (descending)
- [ ] User can sort by: time estimate (ascending)
- [ ] User can sort by: custom order (drag-and-drop)
- [ ] Filters persist during session
- [ ] User can clear all filters with one click
- [ ] Multiple active filters work together (AND logic)

### Feature: Bulk Operations
- [ ] User can select multiple tasks via checkboxes
- [ ] Bulk select-all checkbox selects all visible tasks
- [ ] Bulk deselect-all clears all selections
- [ ] User can bulk-edit: priority, status, tags, due date, time estimate
- [ ] User can bulk-delete with confirmation dialog
- [ ] User can bulk-change status
- [ ] Bulk actions apply to all selected tasks correctly
- [ ] Bulk operations show count of affected tasks ("Update 5 tasks?")
- [ ] Cannot perform bulk operations on 0 selected tasks (button disabled)

### Feature: Data Integrity
- [ ] Soft-deleted tasks excluded from all queries by default
- [ ] Trash view shows only soft-deleted tasks
- [ ] 30-day deletion timer accurate
- [ ] Cannot set due date in past (client + server validation)
- [ ] Cannot create self-referencing subtask (parent = child)
- [ ] Subtask cannot be nested more than 1 level
- [ ] Recurrence rule validates against RFC 5545
- [ ] Attachment file size validated (≤ 10 MB)
- [ ] Tag name max 50 characters enforced
- [ ] Task title required and max 255 characters

---

## 17. Development Roadmap (5 Sprints)

### Sprint 1: User Authentication & Basic Task CRUD
**Duration:** 2 weeks

**Tasks:**
- [ ] Set up project structure (backend API, database)
- [ ] Implement user registration & login (JWT)
- [ ] Create users table and auth middleware
- [ ] Implement POST /api/v1/tasks (create)
- [ ] Implement GET /api/v1/tasks (list, no filtering)
- [ ] Implement GET /api/v1/tasks/:id (get single)
- [ ] Implement PATCH /api/v1/tasks/:id (update)
- [ ] Implement DELETE /api/v1/tasks/:id (soft delete)
- [ ] Implement basic task validation
- [ ] Write unit tests for auth and CRUD endpoints
- [ ] Deploy to staging

### Sprint 2: Advanced Task Features & Customization
**Duration:** 2 weeks

**Tasks:**
- [ ] Create priorities table, triggers for defaults
- [ ] Create task_states table, triggers for defaults
- [ ] Create tags table and task_tags junction table
- [ ] Implement subtasks: create, list, promote
- [ ] Implement recurring tasks: create, generate instances, delete recurrence
- [ ] Implement time estimates field and validation
- [ ] Add priority/state/tag API endpoints (CRUD)
- [ ] Update tasks endpoints to handle new fields
- [ ] Write integration tests for subtasks and recurring
- [ ] Deploy to staging

### Sprint 3: Filtering, Sorting, & Bulk Operations
**Duration:** 2 weeks

**Tasks:**
- [ ] Implement advanced filtering (status, priority, tags, date, time estimate)
- [ ] Implement sorting (due_date, priority, created_at, time_estimate, custom)
- [ ] Build query builder for complex filter combinations
- [ ] Implement bulk operations endpoint
- [ ] Implement bulk edit, delete, change status
- [ ] Write tests for all filter combinations
- [ ] Performance optimization (indices, query optimization)
- [ ] Deploy to staging

### Sprint 4: Attachments & Error Handling
**Duration:** 1.5 weeks

**Tasks:**
- [ ] Create attachments table with validation triggers
- [ ] Implement S3 integration for file upload
- [ ] Implement POST /api/v1/tasks/:id/attachments (create)
- [ ] Implement GET /api/v1/tasks/:id/attachments (list)
- [ ] Implement DELETE /api/v1/attachments/:id (delete)
- [ ] Implement file validation (size, type, count)
- [ ] Implement cascade deletion (delete task → delete attachments)
- [ ] Implement error handling and validation messaging
- [ ] Write tests for attachment operations
- [ ] Deploy to staging

### Sprint 5: Testing, Optimization, & Preparation for F02-F05
**Duration:** 1.5 weeks

**Tasks:**
- [ ] Write comprehensive unit tests (target 80% coverage)
- [ ] Write integration tests (full workflows)
- [ ] Write E2E tests for critical paths
- [ ] Performance testing (load, stress tests)
- [ ] Security review and fixes
- [ ] Database schema optimization & final review
- [ ] Documentation (API docs, developer guide)
- [ ] Prepare schema for F02-F05 (reminders, calendar, notifications tables)
- [ ] Deploy to production
- [ ] Post-launch monitoring and bug fixes

---

## 18. Summary

### F01 Provides:
✅ Flexible task creation with rich metadata
✅ Subtask decomposition (1-level nesting)
✅ Recurring task automation (RFC 5545)
✅ Time estimates for capacity planning
✅ File attachments (5 max, 10 MB each)
✅ Customizable priorities & states
✅ User-defined tags with color coding
✅ Advanced filtering & sorting
✅ Bulk operations for efficiency
✅ Soft delete with 30-day recovery
✅ Complete audit trail & data integrity

### Database Contains:
✅ 8 tables with proper relationships
✅ 20+ indices for performance
✅ 5+ triggers for business logic automation
✅ 50+ validation constraints
✅ Cascading deletes for data consistency

### API Provides:
✅ 30+ endpoints covering all features
✅ Comprehensive query parameters
✅ Proper error handling & validation
✅ Pagination support
✅ Real-time websocket-ready structure

### Ready For:
✅ F02: Reminder System integration (uses tasks & users)
✅ F03: Google Calendar sync (uses gcal_event_id field)
✅ F04: 3rd party notifications (users have telegram_id, whatsapp_number)
✅ F05: Analytics dashboard (completed_at, time_estimates for metrics)

---


