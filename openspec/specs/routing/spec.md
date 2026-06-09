# Routing Specification

## Purpose

Provides a front-controller routing system that maps URLs to controller actions, supports route groups with shared middleware, and executes a middleware pipeline before each request.

## Requirements

### Requirement: Front Controller

The system SHALL use a single entry point (`public/index.php`) that dispatches all requests to the appropriate controller.

#### Scenario: Valid route dispatched

- GIVEN a request to `GET /projects`
- WHEN the front controller processes it
- THEN it routes to `ProjectController@index`

#### Scenario: Unknown route

- GIVEN a request to `GET /nonexistent`
- WHEN the front controller processes it
- THEN it returns HTTP 404 with a "Page not found" response

### Requirement: HTTP Method Routing

The system SHALL match routes by both path and HTTP method.

#### Scenario: GET route

- GIVEN a `GET /projects/5` request
- WHEN matched against route definitions
- THEN it dispatches to `ProjectController@show` with `id=5`

#### Scenario: POST route

- GIVEN a `POST /projects` request
- WHEN matched against route definitions
- THEN it dispatches to `ProjectController@store`

#### Scenario: Method not allowed

- GIVEN a `DELETE /projects` request (if only GET/POST defined)
- WHEN no matching route is found
- THEN the system returns HTTP 405 "Method not allowed"

### Requirement: Route Groups

The system SHALL support grouping routes under a common prefix with shared middleware.

#### Scenario: Auth-protected group

- GIVEN a route group `/admin/*` with `auth` and `role:admin` middleware
- WHEN a request matches `/admin/users`
- THEN both middleware run before the controller

#### Scenario: Public group

- GIVEN a route group `/auth/*` with no middleware
- WHEN a request matches `/auth/login`
- THEN no middleware runs and the controller executes directly

### Requirement: Middleware Pipeline

The system SHALL execute middleware in order before the controller, where each middleware can continue, redirect, or abort.

#### Scenario: Pipeline passes

- GIVEN middleware [auth, csrf, role:admin]
- WHEN all three pass
- THEN the controller executes

#### Scenario: Auth middleware blocks

- GIVEN middleware [auth, csrf]
- WHEN auth detects no session
- THEN auth redirects to login and csrf/Controller never run

#### Scenario: CSRF middleware blocks

- GIVEN middleware [auth, csrf]
- WHEN auth passes but csrf detects an invalid token
- THEN csrf returns HTTP 403 and the controller never runs

### Requirement: Route Parameters

The system SHALL extract named parameters from route patterns and pass them to the controller.

#### Scenario: Single parameter

- GIVEN route pattern `GET /projects/{id}`
- WHEN a request to `GET /projects/42` arrives
- THEN the controller receives `['id' => 42]`

#### Scenario: Multiple parameters

- GIVEN route pattern `GET /projects/{project_id}/tasks/{task_id}`
- WHEN a request to `GET /projects/3/tasks/7` arrives
- THEN the controller receives `['project_id' => 3, 'task_id' => 7]`

### Requirement: Named Routes

The system SHALL support naming routes for URL generation.

#### Scenario: Generate URL from name

- GIVEN a named route `projects.show` with pattern `GET /projects/{id}`
- WHEN the application generates a URL for `projects.show` with `id=5`
- THEN it produces `/projects/5`

### Requirement: Request Lifecycle

The system SHALL handle each request in a consistent lifecycle: routing → middleware → controller → response.

#### Scenario: Full lifecycle

- GIVEN a request to `POST /projects`
- WHEN the lifecycle executes
- THEN the response is sent only after routing, middleware, and controller all complete

#### Scenario: Controller exception

- GIVEN a controller that throws an exception
- WHEN the exception propagates
- THEN the system catches it, logs it, and returns HTTP 500 without exposing the stack trace
