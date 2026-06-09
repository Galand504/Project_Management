# Team Management Specification

## Purpose

Manages teams: creation, member management, and association with projects. Teams are groups of users who collaborate on one or more projects.

## Requirements

### Requirement: Team Creation

The system SHALL allow authenticated users to create teams with a name and optional description.

#### Scenario: Successful creation

- GIVEN an authenticated user on the new team page
- WHEN they submit a name (1-80 chars) and description
- THEN the system creates the team and assigns the creator as `owner` role

#### Scenario: Duplicate team name allowed

- GIVEN a team named "Frontend" already exists
- WHEN the user submits a new team with the same name
- THEN the system allows it (team names are NOT unique globally)

### Requirement: Team Member Management

The system SHALL allow the team owner to add and remove members.

#### Scenario: Owner adds member

- GIVEN the team owner on the members page
- WHEN they add user "alice" to the team
- THEN "alice" becomes a `member` and gains access to the team's projects

#### Scenario: Owner removes member

- GIVEN the team owner on the members page
- WHEN they remove user "alice"
- THEN "alice" loses access to the team's projects

#### Scenario: Non-owner denied

- GIVEN a team member who is not the owner
- WHEN they attempt to add a member
- THEN the system returns HTTP 403

#### Scenario: Owner cannot self-remove

- GIVEN the team owner attempting to remove themselves
- WHEN they submit the removal
- THEN the system rejects with "Owner cannot leave. Transfer ownership first"

### Requirement: Team-to-Project Association

The system SHALL allow linking teams to projects, granting all team members access to the project.

#### Scenario: Associate team with project

- GIVEN a project owner on the project settings page
- WHEN they associate team "Frontend" with project "Alpha"
- THEN all "Frontend" members gain access to "Alpha"

#### Scenario: Dissociate team from project

- GIVEN a project owner on the project settings page
- WHEN they remove team "Frontend" from project "Alpha"
- THEN "Frontend" members lose access (unless they have individual access)

#### Scenario: Duplicate association ignored

- GIVEN team "Frontend" is already associated with project "Alpha"
- WHEN the owner attempts to associate them again
- THEN the system ignores the duplicate without error

### Requirement: Team Listing

The system SHALL display all teams the user belongs to.

#### Scenario: User with teams

- GIVEN a user who is a member of 3 teams
- WHEN they view the team list
- THEN the system shows all 3 teams with name, member count, and their role

#### Scenario: User with no teams

- GIVEN a user who belongs to no teams
- WHEN they view the team list
- THEN the system shows "No teams found" and a "Create Team" button

### Requirement: Team Detail View

The system SHALL display team details including members, associated projects, and the user's role.

#### Scenario: Owner views detail

- GIVEN the team owner on the team detail page
- WHEN the page loads
- THEN the system shows members, projects, and management controls

#### Scenario: Member views detail

- GIVEN a team member on the team detail page
- WHEN the page loads
- THEN the system shows members and projects without management controls

### Requirement: Team Deletion

The system SHALL allow the team owner to delete a team.

#### Scenario: Owner deletes team

- GIVEN the team owner on the delete confirmation page
- WHEN they confirm deletion
- THEN the system removes all associations and soft-deletes the team

#### Scenario: Active project warning

- GIVEN a team linked to an active project
- WHEN the owner attempts to delete the team
- THEN the system warns "This team is linked to active projects" and requires confirmation
