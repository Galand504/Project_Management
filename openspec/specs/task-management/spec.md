# Task Management Specification

## Purpose

Manages tasks within projects: creation, assignment, state transitions, and deletion. Tasks are discrete work items tied to a project and optionally assigned to team members.

## Requirements

### Requirement: Task Creation

The system SHALL allow project owners and team members to create tasks within a project.

#### Scenario: Successful creation

- GIVEN a user on the new task page for project "Alpha"
- WHEN they submit a title (1-150 chars), description, priority (low/medium/high), and due date
- THEN the system creates the task with status `pending` in project "Alpha"

#### Scenario: Missing title

- GIVEN the user submits a task without a title
- WHEN the form is processed
- THEN the system rejects the submission and shows "Title is required"

#### Scenario: Non-member denied

- GIVEN a user who is not a member of project "Alpha"
- WHEN they attempt to create a task in that project
- THEN the system returns HTTP 403

### Requirement: Task State Machine

The system SHALL enforce valid state transitions: `pending` → `in-progress` → `completed`.

#### Scenario: Forward transition

- GIVEN a task in `pending` state
- WHEN a team member sets it to `in-progress`
- THEN the system updates the state and records the transition timestamp

#### Scenario: Completion

- GIVEN a task in `in-progress` state
- WHEN a team member marks it as `completed`
- THEN the system records the completion timestamp

#### Scenario: Invalid transition

- GIVEN a task in `completed` state
- WHEN a user attempts to set it back to `in-progress`
- THEN the system rejects with "Cannot reopen a completed task"

#### Scenario: Skip state rejected

- GIVEN a task in `pending` state
- WHEN a user attempts to set it directly to `completed`
- THEN the system rejects with "Task must be in-progress first"

### Requirement: Task Assignment

The system SHALL allow assigning tasks to team members of the associated project.

#### Scenario: Assign to team member

- GIVEN a task in project "Alpha"
- WHEN the owner assigns user "bob" (a team member)
- THEN the system records the assignment and notifies "bob"

#### Scenario: Assign to non-member rejected

- GIVEN a task in project "Alpha"
- WHEN the owner attempts to assign user "carol" (not a team member)
- THEN the system rejects with "User is not a project member"

### Requirement: Task Listing and Filtering

The system SHALL display tasks for a project with filtering by status, priority, and assignee.

#### Scenario: Filter by status

- GIVEN a project with 15 tasks across all states
- WHEN the user filters by status `pending`
- THEN the system shows only pending tasks

#### Scenario: Empty filter result

- GIVEN a project with no high-priority tasks
- WHEN the user filters by priority `high`
- THEN the system shows "No tasks match your filters"

### Requirement: Task Deletion

The system SHALL allow the project owner or admin to delete tasks.

#### Scenario: Owner deletes task

- GIVEN the project owner on the task detail page
- WHEN they confirm deletion
- THEN the system soft-deletes the task and returns to the task list

#### Scenario: Non-owner cannot delete

- GIVEN a team member who is not the project owner
- WHEN they attempt to delete a task
- THEN the system returns HTTP 403

### Requirement: Task Due Date Alerts

The system SHALL flag tasks approaching their due date.

#### Scenario: Overdue task flagged

- GIVEN a task with a due date in the past and status not `completed`
- WHEN the project list loads
- THEN the system marks the task as overdue visually

#### Scenario: Due-soon task flagged

- GIVEN a task due within 24 hours and status `pending`
- WHEN the project list loads
- THEN the system marks the task as due-soon visually
