# Project Management Specification

## Purpose

Manages the lifecycle of projects: creation, reading, updating, and deletion. Enforces ownership-based access control and supports paginated listing for large datasets.

## Requirements

### Requirement: Project Creation

The system SHALL allow authenticated users to create projects with a name, description, and optional deadline.

#### Scenario: Successful creation

- GIVEN an authenticated user on the new project page
- WHEN they submit a valid name (1-100 chars), description, and optional deadline
- THEN the system creates the project, assigns the user as owner, and redirects to the project list

#### Scenario: Missing required field

- GIVEN the user submits a project without a name
- WHEN the form is processed
- THEN the system rejects the submission and shows "Name is required"

#### Scenario: Deadline in the past

- GIVEN the user submits a deadline earlier than today
- WHEN the form is processed
- THEN the system rejects the submission and shows "Deadline must be in the future"

### Requirement: Project Listing with Pagination

The system SHALL display projects the user owns or is a team member of, with paginated results.

#### Scenario: Paginated list

- GIVEN a user with 25 projects
- WHEN they view the project list with default page size (10)
- THEN the system shows 10 projects and pagination controls

#### Scenario: Filtered by status

- GIVEN a user on the project list page
- WHEN they filter by status "active"
- THEN the system shows only active projects matching the filter

#### Scenario: Empty result

- GIVEN a user with no projects
- WHEN they view the project list
- THEN the system shows "No projects found"

### Requirement: Project Detail View

The system SHALL display full project details including name, description, deadline, status, owner, and team members.

#### Scenario: Owner views project

- GIVEN the project owner on the project detail page
- WHEN the page loads
- THEN the system shows all project fields and an edit button

#### Scenario: Team member views project

- GIVEN a user who is a team member on the project
- WHEN the page loads
- THEN the system shows project details without the edit button

### Requirement: Project Update

The system SHALL allow the project owner or an admin to update project details.

#### Scenario: Owner updates project

- GIVEN the project owner on the edit page
- WHEN they change the description and save
- THEN the system updates the project and shows "Project updated"

#### Scenario: Non-owner denied update

- GIVEN a user who is not the project owner or admin
- WHEN they attempt to access the edit URL
- THEN the system returns HTTP 403

#### Scenario: Optimistic locking conflict

- GIVEN two users editing the same project simultaneously
- WHEN one saves while the other is still editing
- THEN the second save triggers a conflict error and prompts retry

### Requirement: Project Deletion

The system SHALL allow the project owner or admin to soft-delete a project.

#### Scenario: Owner deletes project

- GIVEN the project owner on the delete confirmation page
- WHEN they confirm deletion
- THEN the system marks the project as deleted and redirects to the list

#### Scenario: Cascading soft delete

- GIVEN a project with associated tasks and team memberships
- WHEN the project is soft-deleted
- THEN all associated tasks and memberships are also soft-deleted

### Requirement: Project Status Management

The system SHALL track project status as `active`, `completed`, or `archived`.

#### Scenario: Status transition

- GIVEN an active project
- WHEN the owner marks it as completed
- THEN the system updates the status and records the completion timestamp

#### Scenario: Invalid transition

- GIVEN an archived project
- WHEN a user attempts to set it back to active
- THEN the system rejects the change with "Cannot reactivate archived project"
