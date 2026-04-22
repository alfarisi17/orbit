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

## F01 Smart To-Do List Specification
- **Task Structure**: Tasks can include subtasks and dependencies.
- **Subtasks**: Defined under a main task with their attributes.
- **Recurring Tasks**: Users can define frequency (daily, weekly, monthly) for task repetition.
- **Time Estimates**: Users input expected start and end times.
- **Attachments**: Upload files related to each task.
- **Priority Customization**: Select low, medium, high priorities through a dropdown.
- **Task States Customization**: Users can define states like 'Not Started', 'In Progress', 'Completed'.
- **Tags**: Users can add tags that can be color-coded.
- **Filtering and Sorting**: Users can filter tasks based on their states and search tags.
- **Bulk Operations**: Select multiple tasks for operations like delete or mark as complete.

## PostgreSQL Database Schema
- **Tasks Table**: `CREATE TABLE tasks (id SERIAL PRIMARY KEY, title TEXT, description TEXT, priority INT, state TEXT, tags TEXT[], created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ);`
- **Subtasks Table**: `CREATE TABLE subtasks (id SERIAL PRIMARY KEY, task_id INT REFERENCES tasks(id), title TEXT, state TEXT, created_at TIMESTAMPTZ, completed_at TIMESTAMPTZ);`
- **Recurring Tasks Table**: `CREATE TABLE recurring_tasks (id SERIAL PRIMARY KEY, task_id INT REFERENCES tasks(id), frequency TEXT, start_date TIMESTAMPTZ, end_date TIMESTAMPTZ);`

## API Endpoints
- **CRUD for Tasks**: `POST /api/tasks`, `GET /api/tasks/{id}`, `PUT /api/tasks/{id}`, `DELETE /api/tasks/{id}`
- **Subtasks**: `POST /api/tasks/{id}/subtasks`, `GET /api/subtasks/{id}`, `PUT /api/subtasks/{id}`, `DELETE /api/subtasks/{id}`
- **Recurring Operations**: `POST /api/tasks/{id}/recurrence`, `GET /api/recurring/{id}`, etc.
- **Tags Management**: `POST /api/tags`, `GET /api/tags` 
- **Priority and States**: `GET /api/priorities`, `GET /api/states`

## Business Logic
- **Subtasks Completion**: Automatically update parent task status upon completion of all subtasks.
- **Recurring Task Generation**: Create new instances based on defined frequencies.
- **Soft Delete and Recovery**: Tasks can be soft deleted with recovery options.
- **Priority and State Defaults**: Default settings based on user preferences.

## Acceptance Criteria
1. User can successfully create a task with all attributes filled out.
2. User can add and complete subtasks.
3. User can set tasks to recurring.
4. User can filter tasks by priority and states.
5. User can perform bulk operations efficiently.
6. UI displays tasks sorted by user preference.
7. Tests cover at least 30 different scenarios.

## Development Roadmap (5 Sprints)
1. **Sprint 1**: User Authentication & Task Creation.
2. **Sprint 2**: Subtasks and Recurrence Features.
3. **Sprint 3**: Database Integration & API Development.
4. **Sprint 4**: UI/UX Design Implementation.
5. **Sprint 5**: Testing and Bug Fixes.