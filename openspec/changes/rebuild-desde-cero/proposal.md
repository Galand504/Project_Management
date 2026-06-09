# Proposal: Rebuild GestionProyectos desde cero

## Intent

Rebuild GestionProyectos from scratch using a clean PHP vanilla MVC architecture with PostgreSQL, replacing the current procedural spaghetti code (~35 app files, zero separation of concerns, multiple critical security vulnerabilities). The current codebase is unsalvageable: SQL injection in `Registrar.php`, XSS in `user_list.php`, zero CSRF protection across all forms, hardcoded DB credentials in every file, and no tests. A greenfield rebuild is faster and safer than refactoring.

## Scope

### In Scope

- **Authentication**: Login, registration, logout with secure sessions (httponly, samesite), password hashing (bcrypt), CSRF tokens on all forms
- **Authorization**: Role-based access control (admin/usuario) with middleware enforcement, role assignment restricted to admin only
- **Project CRUD**: Create, read, update, delete projects; owner-based access control; paginated listings with filters
- **Task CRUD**: Create, read, update, delete tasks with state transitions (pending/in-progress/completed); assigned to teams; paginated listings
- **Team CRUD**: Create, read, update, delete teams; member management (add/remove); teams scoped to projects
- **Database**: PostgreSQL with migration system, foreign keys, proper indexes, prepared statements everywhere
- **Infrastructure**: Front controller with routing, centralized config (.env), PHPUnit testing from day 1, no CDN dependencies

### Out of Scope

- Forum, private messages, documents, reports/graphs, Kanban board, i18n, PDF generation
- Data migration (fresh start — no legacy data)
- Real-time features (websockets, notifications)
- File upload/download

## Capabilities

### New Capabilities

- `user-auth`: Secure authentication with login, registration, logout, session management, and CSRF protection
- `rbac`: Role-based access control middleware enforcing admin/usuario permissions across all routes
- `project-management`: Full CRUD for projects with owner-based access, status tracking, and paginated listings
- `task-management`: Full CRUD for tasks with state machine transitions, team assignment, and filtered views
- `team-management`: Full CRUD for teams with member management, scoped to projects
- `database-layer`: PostgreSQL schema with migrations, foreign keys, prepared statements, and connection pooling
- `routing`: Front controller pattern with URL routing, route groups, and middleware pipeline

### Modified Capabilities

None — this is a greenfield rebuild.

## Approach

1. **Phase 1 — Foundation**: Project structure, routing, config, database connection, migration system
2. **Phase 2 — Auth**: User registration/login/logout, password hashing, session management, CSRF middleware
3. **Phase 3 — RBAC**: Role enforcement middleware, admin-only route protection
4. **Phase 4 — Projects**: CRUD with owner scoping, pagination, validation
5. **Phase 5 — Teams**: CRUD with member management, project scoping
6. **Phase 6 — Tasks**: CRUD with state transitions, team assignment, filtering
7. **Phase 7 — Hardening**: Input validation, error handling, logging, rate limiting

Each phase includes PHPUnit tests written alongside implementation.

## Affected Areas

- Entire codebase — complete rewrite. No existing files are preserved.
- New directory structure: `src/`, `public/`, `config/`, `database/migrations/`, `tests/`
- PostgreSQL database (new instance, no existing data)

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Zero test coverage initially | High | Write PHPUnit tests from Phase 1 onward |
| Scope creep on v1 CORE | Medium | Strict feature gate — forum, messages, docs, graphs excluded until v2 |
| PostgreSQL learning curve | Medium | Consistent prepared statements (PDO), document dialect differences |

## Rollback Plan

No rollback needed — the existing codebase remains untouched in a separate branch. The rebuild lives in its own directory tree. If abandoned, the old code is still functional.

## Dependencies

- PHP 8.2+ with PDO_pgsql extension
- PostgreSQL 15+
- Composer (for PHPUnit only — no framework dependencies)
- PHPUnit 11+

## Success Criteria

- [ ] User can register, login, logout with secure session
- [ ] Admin role cannot be self-assigned during registration
- [ ] All form submissions include valid CSRF tokens
- [ ] All DB queries use prepared statements (zero string concatenation)
- [ ] CRUD operations work for projects, tasks, and teams
- [ ] Role middleware blocks unauthorized access to admin routes
- [ ] All list views include pagination
- [ ] PHPUnit test suite passes with >80% line coverage on business logic
- [ ] Zero hardcoded credentials — all config via .env
- [ ] No external CDN dependencies — all assets served locally
