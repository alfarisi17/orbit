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

# F01 Specification Document

## 1. Executive Summary
Provide a brief overview of the Smart To-Do List application, focusing on its goals and intended impact.

## 2. Business Objectives  
- To improve task management efficiency for users.  
- To offer a customizable and user-friendly interface.  
- To integrate reminders and recurring tasks for better planning.  

## 3. Core Features Overview  
- Smart Task Creation  
- Subtasks Management  
- Recurring Task Options  
- Attachments Support  
- API Integration for Third-party Applications  

## 4. Technical Architecture  
Include diagram and description of the system architecture, including front-end, back-end, and database components.

## 5. F01: Smart To-Do List  
### - Overview  
The application serves as a digital assistant that helps users organize their tasks effectively.

### - Task Structure & Metadata  
| Field          | Description               |
|----------------|---------------------------|
| id             | Unique identifier         |
| title          | Task name                 |
| description    | Task details              |
| created_at     | Creation timestamp         |
| updated_at     | Last updated timestamp     |
| due_date       | Task deadline             |
| priority       | Priority level            |
| status         | Current state of task     |
| tags           | Associated tags           |

### - Subtasks  
- Subtasks must be independent but linked to their parent task.  
- Completion of parent task is dependent on all subtasks being completed.

### - Recurring Tasks  
- Tasks can follow RFC 5545 RRULE format.  
- Lazy generation of task instances based on user-defined rules.

### - Time Estimates  
- Store estimated time in hours.  
- Display format will be HH:MM.  
- Use cases include task planning and deadline setting.

### - Attachments  
- Maximum of 5 attachments per task, each up to 10MB.  
- Attachments should be stored in cloud storage.

### - Priority Levels  
- Default levels: Low, Medium, High.  
- Users can customize priority levels as needed.

### - Task States  
- Default states: Pending, In Progress, Blocked, Completed.
- Users may define custom states as required.

### - Tags & Color Coding  
- Users can create custom tags for organization.  
- Many-to-many relationships with tasks for efficient categorization.

### - Filtering & Sorting  
- Essential features include filtering by due date, priority, and tags.

### - Bulk Operations  
- Support for bulk edit, delete, and change status through API.  

## 6. Complete PostgreSQL Database Schema  
- **Users Table**: Stores user information including timezone, telegram_id, whatsapp_number.
- **Priorities Table**: Contains priority levels and a weight system with triggers.
- **Task States Table**: Includes task states linked with color codes and triggers.
- **Tags Table**: Supports user-defined tags, ensuring scoping by user.
- **Tasks Table**: Main table holding all fields, constraints, indices.
- **Task Tags Junction Table**: To support many-to-many relationships.
- **Attachments Table**: Stores attachment information with validation rules.

## 7. All 30+ API Endpoints Organized by Category
- **Task Endpoints**: Create, read, update, delete tasks.
- **User Endpoints**: Manage user accounts, preferences.
- **Attachment Endpoints**: Upload, delete, retrieve attachments.

## 8. Business Logic & Edge Cases  
- **Subtask Completion Logic**: Address how completion of subtasks influences parent tasks.
- **Recurring Task Generation**: Details of the algorithm for generating recurrence based on rules.
- **Soft Delete & 30-Day Recovery**: Logic for reactivating deleted tasks.
- **Default Creation on User Signup**: Tasks created automatically during sign up.
- **Validation Rules**: Overview of 50+ rules ensuring data integrity.

## 9. Acceptance Criteria Section  
- Provide a detailed list of 50+ criteria organized by feature to ensure quality assurance.

## 10. 5-Sprint Development Roadmap  
- **Sprint 1**: Initial setup, user authentication, and basic task CRUD functionality.
- **Sprint 2**: Implement subtasks, basic API, and filtering.
- **Sprint 3**: Finalize recurring tasks and attachments.
- **Sprint 4**: Complete the Database Schema and prepare for deployment.
- **Sprint 5**: Testing, bug fixes, and launch preparation.
