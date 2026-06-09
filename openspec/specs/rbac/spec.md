# Role-Based Access Control Specification

## Purpose

Enforces authorization by assigning roles to users and restricting access to resources based on those roles. Works in conjunction with authentication to ensure users can only perform actions their role permits.

## Requirements

### Requirement: Role Assignment

The system SHALL assign each user exactly one role: `admin` or `user`.

#### Scenario: Default role on registration

- GIVEN a new user completes registration
- WHEN the account is created
- THEN the system assigns the `user` role by default

#### Scenario: Admin promotes a user

- GIVEN an admin on the user management page
- WHEN they change a user's role to `admin`
- THEN the system updates the role and the user gains admin privileges immediately

### Requirement: Role-Based Route Protection

The system SHALL enforce role access via middleware on protected routes.

#### Scenario: Admin accesses admin-only route

- GIVEN a user with the `admin` role
- WHEN they request `/admin/users`
- THEN the system grants access

#### Scenario: Regular user denied admin route

- GIVEN a user with the `user` role
- WHEN they request `/admin/users`
- THEN the system returns HTTP 403 and shows "Access denied"

#### Scenario: Unauthenticated user denied protected route

- GIVEN a visitor without a session
- WHEN they request any protected route
- THEN the system redirects to login

### Requirement: Resource Ownership Enforcement

The system SHALL allow users to modify only resources they own or are assigned to, unless they have the `admin` role.

#### Scenario: Owner modifies own project

- GIVEN a user who owns a project
- WHEN they update the project details
- THEN the system allows the change

#### Scenario: Non-owner denied modification

- GIVEN a user who does not own a project
- WHEN they attempt to update it
- THEN the system returns HTTP 403

#### Scenario: Admin overrides ownership

- GIVEN an admin who does not own a project
- WHEN they update the project details
- THEN the system allows the change

### Requirement: Role Middleware Pipeline

The system SHALL apply role checks as middleware in the routing pipeline, before controller execution.

#### Scenario: Middleware blocks unauthorized request

- GIVEN a route with `role:admin` middleware
- WHEN a `user`-role user requests that route
- THEN the middleware returns HTTP 403 before the controller runs

#### Scenario: Middleware passes authorized request

- GIVEN a route with `role:admin` middleware
- WHEN an `admin`-role user requests that route
- THEN the middleware passes the request to the controller

### Requirement: Role Change Audit

The system SHALL log all role changes with the actor, target user, old role, new role, and timestamp.

#### Scenario: Role change logged

- GIVEN an admin promotes user "alice" from `user` to `admin`
- WHEN the promotion is saved
- THEN the audit log records the change with all fields populated

#### Scenario: Self-role change prevented

- GIVEN an admin attempting to demote themselves
- WHEN they submit the role change
- THEN the system rejects the change and shows "Cannot change your own role"
