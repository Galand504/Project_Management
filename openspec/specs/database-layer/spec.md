# Database Layer Specification

## Purpose

Provides the data persistence layer using PostgreSQL. Enforces referential integrity, supports schema migrations, and ensures all queries use prepared statements to prevent SQL injection.

## Requirements

### Requirement: PostgreSQL Connection

The system SHALL connect to PostgreSQL using persistent connections with configurable host, port, database, user, and password.

#### Scenario: Successful connection

- GIVEN valid database credentials in configuration
- WHEN the application starts
- THEN the system establishes a connection and is ready for queries

#### Scenario: Connection failure

- GIVEN invalid credentials or unreachable host
- WHEN the application attempts to connect
- THEN the system throws a clear error and does not expose credentials

### Requirement: Prepared Statements

The system SHALL use parameterized queries (prepared statements) for ALL database interactions.

#### Scenario: Parameterized query executed

- GIVEN a query with user-supplied input
- WHEN the query is executed
- THEN the system uses bound parameters, never string concatenation

#### Scenario: SQL injection attempt blocked

- GIVEN input containing `'; DROP TABLE users; --`
- WHEN the input is passed to a parameterized query
- THEN the system treats it as a literal string, not executable SQL

### Requirement: Referential Integrity

The system SHALL enforce foreign key constraints between all related tables.

#### Scenario: Cascading delete

- GIVEN a project with associated tasks
- WHEN the project is deleted (hard delete in cleanup)
- THEN the database cascades the delete to all related tasks

#### Scenario: FK violation rejected

- GIVEN a task referencing a non-existent project ID
- WHEN the insert is attempted
- THEN the database rejects it and the system returns a user-friendly error

### Requirement: Schema Migrations

The system SHALL manage schema changes through numbered migration files applied in order.

#### Scenario: Fresh database setup

- GIVEN an empty PostgreSQL database
- WHEN migrations are applied
- THEN all tables, indexes, and constraints are created in correct order

#### Scenario: Incremental migration

- GIVEN a database at migration v3
- WHEN migration v4 is applied
- THEN only the changes in v4 are executed

#### Scenario: Migration already applied

- GIVEN a database where migration v4 is already applied
- WHEN the system attempts to apply v4 again
- THEN the system skips it without error

#### Scenario: Migration failure

- GIVEN a migration with a syntax error
- WHEN the system attempts to apply it
- THEN the system rolls back the partial change and reports the error

### Requirement: Soft Deletes

The system SHALL use soft deletes for users, projects, tasks, and teams via a `deleted_at` timestamp column.

#### Scenario: Soft delete applied

- GIVEN a record in the `projects` table
- WHEN the user deletes the project
- THEN `deleted_at` is set to the current timestamp; the row remains

#### Scenario: Soft-deleted records excluded from queries

- GIVEN a project with `deleted_at` set
- WHEN a user lists their projects
- THEN the soft-deleted project is excluded from results

### Requirement: Timestamps

The system SHALL maintain `created_at` and `updated_at` columns on all tables, auto-populated on insert and update.

#### Scenario: Timestamps set on create

- GIVEN a new record being inserted
- WHEN the insert completes
- THEN `created_at` and `updated_at` are set to the current timestamp

#### Scenario: Timestamps updated on modify

- GIVEN an existing record
- WHEN any field is updated
- THEN `updated_at` is set to the current timestamp

### Requirement: Indexes

The system SHALL create indexes on foreign keys and frequently queried columns.

#### Scenario: Query performance

- GIVEN a `tasks` table with 10,000 rows
- WHEN a query filters by `project_id`
- THEN the index is used and the query completes in < 50ms
