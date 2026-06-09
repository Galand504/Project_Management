# Tasks: Rebuild GestionProyectos desde cero

## Review Workload Forecast

| Field | Value |
|-------|-------|
| Estimated changed lines | ~2,200 |
| 400-line budget risk | High |
| Chained PRs recommended | Yes |
| Suggested split | 7 PRs (one per phase) |
| Delivery strategy | ask-on-risk |
| Chain strategy | feature-branch-chain |

Decision needed before apply: Yes
Chained PRs recommended: Yes
Chain strategy: feature-branch-chain
400-line budget risk: High

### Suggested Work Units

| Unit | Goal | Likely PR | Notes |
|------|------|-----------|-------|
| 1 | Foundation: project structure, config, routing, DB, migrations | PR 1 | base: feature/rebuild; includes phpunit setup |
| 2 | Auth: registration, login, logout, sessions, CSRF | PR 2 | base: PR 1 branch |
| 3 | RBAC: role middleware, admin protection, ownership checks | PR 3 | base: PR 2 branch |
| 4 | Projects: CRUD, owner scoping, pagination | PR 4 | base: PR 3 branch |
| 5 | Teams: CRUD, member management, project association | PR 5 | base: PR 4 branch |
| 6 | Tasks: CRUD, state machine, assignment, filtering | PR 6 | base: PR 5 branch |
| 7 | Hardening: validation, error handling, logging, rate limiting | PR 7 | base: PR 6 branch |

---

## Phase 1: Foundation

**Objetivo**: Estructura del proyecto, configuración, sistema de routing, conexión a DB, y migraciones.

### 1.1 Scaffold project directory structure
**Archivos**: `src/`, `public/`, `config/`, `database/migrations/`, `tests/`
**Depende de**: —
**Descripción**: Create the full directory tree per design.md. Add `.gitignore`, `composer.json` (PHPUnit only), and `config/.env.example`.
**Criterios de aceptación**:
- [ ] All directories exist: `src/{Controllers,Models,Views,Middleware,Helpers,Router,Database,Config}`
- [ ] `public/assets/`, `database/migrations/`, `tests/{Unit,Integration}/` exist
- [ ] `composer.json` requires `phpunit/phpunit ^11` only
- [ ] `.gitignore` excludes `vendor/`, `.env`, `*.log`
**Tests asociados**: None (structure only)

### 1.2 Create config loader (.env)
**Archivos**: `config/Config.php`, `config/.env.example`
**Depende de**: 1.1
**Descripción**: Implement `Config::load()` that reads `.env` with `getenv()`, validates required keys (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS`, `APP_ENV`), and throws on missing. Support `APP_ENV=production` for session secure flags.
**Criterios de aceptación**:
- [ ] `Config::get('DB_HOST')` returns value from `.env`
- [ ] Missing required key throws `RuntimeException`
- [ ] `.env.example` documents all keys with placeholder values
**Tests asociados**: `tests/Unit/ConfigTest.php` — test load, test missing key throws

### 1.3 Implement Database connection singleton
**Archivos**: `src/Database/Database.php`
**Depende de**: 1.2
**Descripción**: Create `Database::getInstance()` returning a PDO connection to PostgreSQL using `Config` values. Use `PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION`, `PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC`, persistent connections (`PDO::ATTR_PERSISTENT => true`). No string concatenation in queries — ever.
**Criterios de aceptación**:
- [ ] `Database::getInstance()` returns PDO instance
- [ ] Connection uses prepared statement mode by default
- [ ] Invalid credentials throw `PDOException` without exposing password
- [ ] Singleton pattern (second call returns same instance)
**Tests asociados**: `tests/Integration/DatabaseTest.php` — test connection with valid/invalid creds

### 1.4 Create migration runner and initial migration
**Archivos**: `database/migrations/001_create_users_table.sql`, `database/Migrator.php`, `cli/migrate`
**Depende de**: 1.3
**Descripción**: Implement `Migrator` class that: (1) creates a `migrations` tracking table, (2) reads numbered `.sql` files from `database/migrations/`, (3) applies unapplied migrations in order, (4) records applied migrations. Write CLI script `cli/migrate` that runs the migrator. First migration creates all tables per design.md schema (users, people, projects, teams, team_members, tasks) with foreign keys, indexes, CHECK constraints, `deleted_at` columns, `created_at`/`updated_at` timestamps.
**Criterios de aceptación**:
- [ ] `php cli/migrate` applies all migrations on fresh DB
- [ ] Running again skips already-applied migrations
- [ ] All tables created with correct columns, FKs, indexes, constraints
- [ ] `deleted_at` column present on users, projects, tasks, teams
**Tests asociados**: `tests/Integration/MigratorTest.php` — test fresh migrate, test idempotency

### 1.5 Implement Router class
**Archivos**: `src/Router/Router.php`, `src/Router/Route.php`
**Depende de**: 1.1
**Descripción**: Implement `Router` with: `get()`, `post()`, `put()`, `delete()` methods. Support `{param}` extraction (e.g., `/projects/{id}`). Support route groups with prefix and middleware array. Support named routes for URL generation. Match by method+path, return 404 on no match, 405 on method mismatch.
**Criterios de aceptación**:
- [ ] `GET /projects/5` extracts `['id' => 5]`
- [ ] `POST /projects` routes to different handler than GET
- [ ] Unknown route returns 404
- [ ] Known path but wrong method returns 405
- [ ] Route groups apply prefix and middleware to all routes
- [ ] Named routes generate correct URLs
**Tests asociados**: `tests/Unit/RouterTest.php` — test param extraction, groups, 404/405, named routes

### 1.6 Implement Middleware pipeline
**Archivos**: `src/Middleware/MiddlewareInterface.php`, `src/Middleware/Pipeline.php`
**Depende de**: 1.5
**Descripción**: Define `MiddlewareInterface` with `handle($request, $next): Response`. Implement `Pipeline` that chains middleware in order, each calling `$next()` to continue or returning early (redirect/abort). Pipeline receives request + controller closure, returns Response.
**Criterios de aceptación**:
- [ ] Middleware executes in defined order
- [ ] Middleware can short-circuit (return without calling `$next`)
- [ ] Controller only runs if all middleware pass
- [ ] Pipeline handles exceptions gracefully
**Tests asociados**: `tests/Unit/PipelineTest.php` — test order, short-circuit, exception handling

### 1.7 Create Front Controller
**Archivos**: `public/index.php`
**Depende de**: 1.2, 1.5, 1.6
**Descripción**: Single entry point that: (1) loads Config, (2) initializes Router with all routes (placeholder controllers), (3) matches current request, (4) runs middleware pipeline, (5) dispatches to controller, (6) sends response. Include bootstrap for autoloading (`spl_autoload_register` or Composer autoloader).
**Criterios de aceptación**:
- [ ] `public/index.php` handles all requests (no direct script access)
- [ ] Autoloader works for `src/` namespace
- [ ] 404 page renders for unknown routes
- [ ] Request lifecycle: route → middleware → controller → response
**Tests asociados**: `tests/Integration/FrontControllerTest.php` — test routing to controller, 404

### 1.8 Setup PHPUnit configuration
**Archivos**: `phpunit.xml`
**Depende de**: 1.1
**Descripción**: Configure PHPUnit with separate test suites for Unit and Integration. Integration suite requires DB connection. Set coverage target >80% on business logic.
**Criterios de aceptación**:
- [ ] `vendor/bin/phpunit` runs Unit suite without DB
- [ ] `vendor/bin/phpunit --testsuite Integration` runs with DB
- [ ] Coverage report generates correctly
**Tests asociados**: None (config only)

---

## Phase 2: Auth

**Objetivo**: Login, registro, logout, sesiones seguras, CSRF, y hash de contraseñas.

### 2.1 Implement User model
**Archivos**: `src/Models/User.php`
**Depende de**: 1.3
**Descripción**: Create `User` model with methods: `findByEmail($email)`, `create($data)`, `findById($id)`. Use prepared statements only. Password hashing via `password_hash(PASSWORD_BCRYPT, ['cost' => 12])`. Store `username`, `email`, `password_hash`, `role` (default 'user').
**Criterios de aceptación**:
- [ ] `User::create()` hashes password before storing
- [ ] `User::findByEmail()` returns user array or null
- [ ] `User::findById()` returns user array or null
- [ ] Zero string concatenation in SQL
**Tests asociados**: `tests/Unit/UserTest.php` — test create/find, test password is hashed

### 2.2 Implement CsrfMiddleware
**Archivos**: `src/Middleware/CsrfMiddleware.php`, `src/Helpers/Csrf.php`
**Depende de**: 1.6
**Descripción**: `Csrf::generate()` creates token, stores in `$_SESSION`, returns token. `Csrf::validate($token)` compares. `CsrfMiddleware` skips GET requests, validates token on POST/PUT/DELETE, returns 403 on mismatch. Helper `csrf_field()` generates hidden input.
**Criterios de aceptación**:
- [ ] GET request passes without token check
- [ ] POST with valid token passes
- [ ] POST without token returns 403
- [ ] POST with wrong token returns 403
- [ ] `csrf_field()` outputs `<input type="hidden" name="_token" value="...">`
**Tests asociados**: `tests/Unit/CsrfTest.php` — test generate/validate, test middleware blocks invalid

### 2.3 Implement AuthMiddleware
**Archivos**: `src/Middleware/AuthMiddleware.php`
**Depende de**: 1.6, 2.1
**Descripción**: Check `$_SESSION['user_id']` exists and user exists in DB. If not, redirect to `/auth/login`. If valid, attach user data to request.
**Criterios de aceptación**:
- [ ] Request with valid session passes through
- [ ] Request without session redirects to login
- [ ] Request with invalid user_id (deleted user) redirects to login
**Tests asociados**: `tests/Unit/AuthMiddlewareTest.php` — test pass, test redirect on no session

### 2.4 Implement AuthController
**Archivos**: `src/Controllers/AuthController.php`, `src/Views/auth/login.php`, `src/Views/auth/register.php`
**Depende de**: 2.1, 2.2, 2.3
**Descripción**: Implement `login()` (GET: show form, POST: validate credentials, set session, redirect), `register()` (GET: show form, POST: validate input, create user, redirect to login), `logout()` (destroy session, redirect to login). Session config: `httponly`, `samesite=Lax`, `secure` in production, 30-min timeout.
**Criterios de aceptación**:
- [ ] Registration creates user with hashed password, defaults to 'user' role
- [ ] Duplicate email shows "Email already in use"
- [ ] Password mismatch shows "Passwords do not match"
- [ ] Login with correct credentials sets session and redirects to dashboard
- [ ] Login with wrong credentials shows "Invalid email or password"
- [ ] Logout destroys session and redirects to login
- [ ] Session regenerated on login (new session ID)
- [ ] Session expires after 30 min inactivity
**Tests asociados**: `tests/Integration/AuthTest.php` — test full registration flow, login flow, logout, session expiry

### 2.5 Register Auth routes
**Archivos**: `public/index.php` (update), `config/routes.php`
**Depende de**: 1.5, 2.4
**Descripción**: Add route definitions: `GET/POST /auth/login`, `GET/POST /auth/register`, `GET /auth/logout` (auth required). Public group for login/register, auth-protected for logout.
**Criterios de aceptación**:
- [ ] `/auth/login` renders login form (GET) and processes login (POST)
- [ ] `/auth/register` renders register form (GET) and processes registration (POST)
- [ ] `/auth/logout` requires session, destroys it
- [ ] CSRF token required on all POST requests
**Tests asociados**: Covered in `tests/Integration/AuthTest.php`

---

## Phase 3: RBAC

**Objetivo**: Role enforcement middleware, admin-only route protection, ownership checks.

### 3.1 Implement RoleMiddleware
**Archivos**: `src/Middleware/RoleMiddleware.php`
**Depende de**: 2.3, 1.6
**Descripción**: Check user's `role` from session/DB against required role. Return 403 "Access denied" if insufficient. Support `role:admin` parameter in route middleware config.
**Criterios de aceptación**:
- [ ] Admin user passes `role:admin` check
- [ ] Regular user blocked by `role:admin` with 403
- [ ] Middleware reads role from session, not re-querying DB every request
**Tests asociados**: `tests/Unit/RoleMiddlewareTest.php` — test pass, test block, test 403 message

### 3.2 Implement ownership check helper
**Archivos**: `src/Helpers/Authorization.php`
**Depende de**: 1.3
**Descripción**: `Authorization::isOwner($userId, $resourceType, $resourceId)` checks ownership in DB. `Authorization::canModify($userId, $role, $resourceType, $resourceId)` returns true if owner OR admin.
**Criterios de aceptación**:
- [ ] Owner of project can modify
- [ ] Admin can modify any project
- [ ] Non-owner non-admin gets denied
**Tests asociados**: `tests/Unit/AuthorizationTest.php` — test owner, admin, non-owner

### 3.3 Add role audit logging
**Archivos**: `database/migrations/002_create_role_audit_log.sql`, `src/Models/RoleAuditLog.php`
**Depende de**: 1.4, 2.1
**Descripción**: Migration creates `role_audit_log` table (id, admin_id, target_user_id, old_role, new_role, created_at). Model logs role changes. Prevent self-role-change.
**Criterios de aceptación**:
- [ ] Role change creates audit log entry with all fields
- [ ] Admin cannot change own role (rejected with "Cannot change your own role")
**Tests asociados**: `tests/Unit/RoleAuditLogTest.php` — test log creation, test self-change prevention

### 3.4 Register RBAC routes and admin pages
**Archivos**: `public/index.php` (update), `src/Controllers/AdminController.php`, `src/Views/admin/users.php`
**Depende de**: 3.1, 3.2
**Descripción**: Add admin routes under `/admin/*` group with `auth` + `role:admin` middleware. Create user listing page for admin role management.
**Criterios de aceptación**:
- [ ] `/admin/users` requires admin role
- [ ] Regular user sees 403 on `/admin/users`
- [ ] Admin can view user list
**Tests asociados**: `tests/Integration/RbacTest.php` — test admin access, test user blocked

---

## Phase 4: Projects

**Objetivo**: CRUD de proyectos con owner scoping, paginación, y validación.

### 4.1 Implement Project model
**Archivos**: `src/Models/Project.php`
**Depende de**: 1.3
**Descripción**: Methods: `create($data)`, `findById($id)`, `findByOwner($userId, $page, $perPage)`, `update($id, $data)`, `softDelete($id)`, `countByOwner($userId)`. Enforce: `name` required (1-100 chars), `status` CHECK constraint, `deleted_at` filtering on all queries.
**Criterios de aceptación**:
- [ ] `create()` sets `owner_id` from authenticated user
- [ ] `findByOwner()` excludes soft-deleted projects
- [ ] `softDelete()` sets `deleted_at` timestamp
- [ ] Status defaults to 'pending'
- [ ] Prepared statements only
**Tests asociados**: `tests/Unit/ProjectTest.php` — test CRUD, test soft delete, test status default

### 4.2 Implement Pagination helper
**Archivos**: `src/Helpers/Pagination.php`, `src/Views/components/pagination.php`
**Depende de**: 1.1
**Descripción**: `Pagination::make($currentPage, $totalItems, $perPage)` returns array with `items`, `currentPage`, `totalPages`, `hasNext`, `hasPrev`. View component renders page links.
**Criterios de aceptación**:
- [ ] 25 items with perPage=10 shows page 1 of 3
- [ ] Page 0 or negative defaults to 1
- [ ] Page beyond total defaults to last page
- [ ] Pagination controls render correctly
**Tests asociados**: `tests/Unit/PaginationTest.php` — test boundaries, default values

### 4.3 Implement ProjectController
**Archivos**: `src/Controllers/ProjectController.php`, `src/Views/projects/index.php`, `src/Views/projects/create.php`, `src/Views/projects/show.php`, `src/Views/projects/edit.php`
**Depende de**: 4.1, 4.2, 2.2
**Descripción**: Actions: `index()` (paginated list with status filter), `create()` (show form), `store()` (validate + create), `show()` (detail, check ownership for edit button), `edit()` (owner/admin only), `update()` (validate + update), `destroy()` (soft delete, cascade).
**Criterios de aceptación**:
- [ ] Index shows paginated projects with filter by status
- [ ] Create validates name required, deadline in future
- [ ] Show displays owner, status, edit button only for owner/admin
- [ ] Update restricted to owner or admin
- [ ] Delete soft-deletes and cascades
- [ ] All views use `htmlspecialchars()` on output
**Tests asociados**: `tests/Integration/ProjectTest.php` — test CRUD flow, ownership enforcement, pagination

### 4.4 Register Project routes
**Archivos**: `public/index.php` (update), `config/routes.php` (update)
**Depende de**: 4.3, 2.5
**Descripción**: Add routes: `GET/POST /projects`, `GET/POST /projects/create`, `GET/PUT/DELETE /projects/{id}`. All under auth middleware.
**Criterios de aceptación**:
- [ ] All project routes require authentication
- [ ] PUT/DELETE work via `_method` hidden field
- [ ] Correct controller methods dispatched
**Tests asociados**: Covered in `tests/Integration/ProjectTest.php`

---

## Phase 5: Teams

**Objetivo**: CRUD de equipos, gestión de miembros, asociación a proyectos.

### 5.1 Implement Team model
**Archivos**: `src/Models/Team.php`, `src/Models/TeamMember.php`
**Depende de**: 1.3
**Descripción**: `Team` methods: `create($data)`, `findById($id)`, `findByUser($userId)`, `update($id, $data)`, `softDelete($id)`. `TeamMember` methods: `add($teamId, $userId)`, `remove($teamId, $userId)`, `getMembers($teamId)`, `isMember($teamId, $userId)`. Team names NOT unique globally.
**Criterios de aceptación**:
- [ ] `Team::create()` sets creator as owner
- [ ] `TeamMember::add()` enforces unique(team_id, user_id)
- [ ] `Team::findByUser()` returns all teams user belongs to
- [ ] Soft delete removes team and cascades memberships
**Tests asociados**: `tests/Unit/TeamTest.php` — test CRUD, test member add/remove, test uniqueness

### 5.2 Implement TeamController
**Archivos**: `src/Controllers/TeamController.php`, `src/Views/teams/index.php`, `src/Views/teams/create.php`, `src/Views/teams/show.php`, `src/Views/teams/edit.php`, `src/Views/teams/members.php`
**Depende de**: 5.1, 2.2
**Descripción**: Actions: `index()` (user's teams), `create()`, `store()`, `show()` (members + projects), `edit()` (owner only), `update()`, `destroy()` (owner only, warn if active projects), `addMember()` (owner only), `removeMember()` (owner only, cannot self-remove).
**Criterios de aceptación**:
- [ ] Index shows user's teams with member count and role
- [ ] Show displays members and associated projects
- [ ] Owner can add/remove members
- [ ] Non-owner gets 403 on member management
- [ ] Owner cannot remove self (error: "Owner cannot leave. Transfer ownership first")
- [ ] Delete warns if linked to active projects
**Tests asociados**: `tests/Integration/TeamTest.php` — test CRUD, member management, ownership enforcement

### 5.3 Implement team-project association
**Archivos**: `database/migrations/003_create_project_teams.sql` (if needed), `src/Models/ProjectTeam.php`
**Depende de**: 5.1, 4.1
**Descripción**: Create `project_teams` junction table (project_id, team_id, UNIQUE). Methods: `associate($projectId, $teamId)`, `dissociate($projectId, $teamId)`, `getTeamsForProject($projectId)`, `getProjectsForTeam($teamId)`. Duplicate association ignored.
**Criterios de aceptación**:
- [ ] Associate grants team members access to project
- [ ] Dissociate removes access
- [ ] Duplicate association silently ignored
- [ ] Cascade delete when project or team deleted
**Tests asociados**: `tests/Unit/ProjectTeamTest.php` — test associate/dissociate, test duplicate handling

### 5.4 Register Team routes
**Archivos**: `public/index.php` (update), `config/routes.php` (update)
**Depende de**: 5.2, 5.3
**Descripción**: Add routes: `GET/POST /projects/{project_id}/teams`, `GET/PUT/DELETE /teams/{id}`, `POST/DELETE /teams/{id}/members`. All under auth middleware.
**Criterios de aceptación**:
- [ ] Team routes nested under project where appropriate
- [ ] Member routes require team ownership
- [ ] All routes require authentication
**Tests asociados**: Covered in `tests/Integration/TeamTest.php`

---

## Phase 6: Tasks

**Objetivo**: CRUD de tareas, state machine, asignación a miembros, filtrado.

### 6.1 Implement Task model
**Archivos**: `src/Models/Task.php`
**Depende de**: 1.3
**Descripción**: Methods: `create($data)`, `findById($id)`, `findByProject($projectId, $filters, $page, $perPage)`, `update($id, $data)`, `softDelete($id)`. State machine: `pending` → `in-progress` → `completed` only. Validate transitions, reject invalid ones.
**Criterios de aceptación**:
- [ ] Default status is 'pending'
- [ ] `pending` → `in-progress` transition valid
- [ ] `in-progress` → `completed` transition valid
- [ ] `completed` → `in-progress` rejected ("Cannot reopen a completed task")
- [ ] `pending` → `completed` rejected ("Task must be in-progress first")
- [ ] Filtering by status, priority, assignee works
**Tests asociados**: `tests/Unit/TaskTest.php` — test all state transitions, test filtering

### 6.2 Implement TaskController
**Archivos**: `src/Controllers/TaskController.php`, `src/Views/tasks/index.php`, `src/Views/tasks/create.php`, `src/Views/tasks/show.php`, `src/Views/tasks/edit.php`
**Depende de**: 6.1, 4.1, 2.2
**Descripción**: Actions: `index()` (project tasks with filters), `create()` (project member only), `store()`, `show()`, `edit()` (project member), `update()`, `destroy()` (project owner/admin only). Show overdue/due-soon flags.
**Criterios de aceptación**:
- [ ] Index shows tasks with status/priority/assignee filters
- [ ] Create restricted to project members
- [ ] Delete restricted to project owner or admin
- [ ] Overdue tasks (past due, not completed) flagged visually
- [ ] Due-soon tasks (within 24h, pending) flagged visually
- [ ] All views escape output with `htmlspecialchars()`
**Tests asociados**: `tests/Integration/TaskTest.php` — test CRUD, test member-only create, test owner-only delete

### 6.3 Implement task assignment
**Archivos**: `src/Models/TaskAssignment.php`
**Depende de**: 6.1, 5.1
**Descripción**: `assign($taskId, $userId)` — verify user is project member, then assign. `unassign($taskId, $userId)`. `getAssigned($taskId)`.
**Criterios de aceptación**:
- [ ] Assign to project member succeeds
- [ ] Assign to non-member rejected ("User is not a project member")
- [ ] Assignment recorded correctly
**Tests asociados**: `tests/Unit/TaskAssignmentTest.php` — test assign, test non-member rejection

### 6.4 Register Task routes
**Archivos**: `public/index.php` (update), `config/routes.php` (update)
**Depende de**: 6.2, 6.3
**Descripción**: Add routes: `GET/POST /teams/{team_id}/tasks`, `GET/PUT/DELETE /tasks/{id}`. All under auth middleware.
**Criterios de aceptación**:
- [ ] Task routes nested under team
- [ ] All routes require authentication
- [ ] PUT/DELETE via `_method` field
**Tests asociados**: Covered in `tests/Integration/TaskTest.php`

---

## Phase 7: Hardening

**Objetivo**: Input validation, error handling, logging, rate limiting, polish.

### 7.1 Implement Validator helper
**Archivos**: `src/Helpers/Validator.php`
**Depende de**: 1.1
**Descripción**: `Validator::make($data, $rules)` — validate required, email, min/max length, regex patterns. Return array of errors. Use for all form submissions.
**Criterios de aceptación**:
- [ ] Required field check returns error if missing
- [ ] Email validation with filter_var
- [ ] Min/max length enforcement
- [ ] Returns structured error array
**Tests asociados**: `tests/Unit/ValidatorTest.php` — test all rule types

### 7.2 Implement error handling and logging
**Archivos**: `src/Helpers/ErrorHandler.php`, `src/Helpers/Logger.php`
**Depende de**: 1.2
**Descripción**: Global error handler: catch exceptions, log to file, show generic 500 page (no stack trace in production). `Logger` writes to `storage/logs/app.log` with levels (ERROR, WARNING, INFO).
**Criterios de aceptación**:
- [ ] Uncaught exception returns 500 page, not stack trace
- [ ] Errors logged with timestamp, level, message
- [ ] Log file rotates or appends safely
**Tests asociados**: `tests/Unit/ErrorHandlerTest.php` — test exception handling, test log format

### 7.3 Add input sanitization layer
**Archivos**: `src/Helpers/Sanitizer.php`
**Depende de**: 7.1
**Descripción**: `Sanitizer::string($input)` — trim, strip tags, limit length. `Sanitizer::int($input)` — cast to int. Apply before validation on all inputs.
**Criterios de aceptación**:
- [ ] String input trimmed and stripped
- [ ] Integer input cast correctly
- [ ] NULL/empty handled gracefully
**Tests asociados**: `tests/Unit/SanitizerTest.php` — test string/int sanitization

### 7.4 Add rate limiting (basic)
**Archivos**: `src/Middleware/RateLimitMiddleware.php`
**Depende de**: 1.6, 1.3
**Descripción**: Track requests per IP in DB or session. Limit login attempts to 5 per 15 minutes per IP. Return 429 "Too many requests" when exceeded.
**Criterios de aceptación**:
- [ ] 5th failed login from same IP triggers lockout
- [ ] Lockout lasts 15 minutes
- [ ] Successful login resets counter
- [ ] Returns 429 with user-friendly message
**Tests asociados**: `tests/Integration/RateLimitTest.php` — test lockout after 5 attempts, test reset

### 7.5 Add view escaping and CSP headers
**Archivos**: `src/Helpers/Escape.php`, `public/index.php` (update)
**Depende de**: 1.7
**Descripción**: `Escape::html($string)` wrapper around `htmlspecialchars` with `ENT_QUOTES`. Add `Content-Security-Policy` header to restrict scripts to self only. Add `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`.
**Criterios de aceptación**:
- [ ] All output uses `Escape::html()` or `htmlspecialchars()`
- [ ] CSP header present on all responses
- [ ] X-Frame-Options prevents iframe embedding
**Tests asociados**: `tests/Unit/EscapeTest.php` — test HTML entity encoding

### 7.6 Create database seed script for dev
**Archivos**: `cli/seed`, `database/seeds/dev.sql`
**Depende de**: 1.4
**Descripción**: Seed script creates: 1 admin user, 3 regular users, 2 projects, 4 teams, 8 tasks with various states. Useful for development and demo.
**Criterios de aceptación**:
- [ ] `php cli/seed` populates DB with test data
- [ ] Admin user has admin role
- [ ] Tasks in all three states (pending, in-progress, completed)
- [ ] Teams linked to projects with members
**Tests asociados**: None (dev tool)

---

## Summary

| Phase | Tasks | Focus |
|-------|-------|-------|
| Phase 1 | 8 | Foundation: structure, config, routing, DB, migrations |
| Phase 2 | 5 | Auth: registration, login, logout, sessions, CSRF |
| Phase 3 | 4 | RBAC: roles, middleware, ownership, audit |
| Phase 4 | 4 | Projects: CRUD, pagination, owner scoping |
| Phase 5 | 4 | Teams: CRUD, members, project association |
| Phase 6 | 4 | Tasks: CRUD, state machine, assignment, filtering |
| Phase 7 | 6 | Hardening: validation, errors, logging, rate limiting |
| **Total** | **35** | |

## Implementation Order

1. **Foundation first** — everything depends on it
2. **Auth second** — all protected routes need it
3. **RBAC third** — admin routes need role checks
4. **Projects fourth** — teams and tasks depend on projects
5. **Teams fifth** — tasks depend on teams
6. **Tasks sixth** — depends on teams and projects
7. **Hardening last** — wraps everything with security and polish

## Review Workload Forecast

- Estimated changed lines: ~2,200
- 400-line budget risk: High
- Chained PRs recommended: Yes
- Delivery strategy: ask-on-risk
- Decision needed before apply: Yes
- Suggested work-unit PR split: 7 PRs (one per phase)

## Next Step

Ask the user which chain strategy to use before starting sdd-apply:
- **feature-branch-chain**: PR 1→2→3→4→5→6→7, final merge to main
- **stacked-to-main**: Each PR merges to main directly
- **size:exception**: Single PR with maintainer approval
