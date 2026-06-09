# Design: Rebuild GestionProyectos desde cero

## Technical Approach

PHP vanilla MVC, Front Controller, PostgreSQL, PHPUnit. Zero framework dependencies.

## Architecture Decisions

### Front Controller
**Choice**: Single `public/index.php` with Router class.
**Rationale**: Maximum learning control, zero dependencies.

### MVC separation
**Choice**: Models (DB), Controllers (HTTP), Views (PHP templates).
**Rationale**: Testable layers, clean responsibilities.

### PDO over mysqli
**Choice**: PDO with prepared statements.
**Rationale**: PostgreSQL support, mandatory prepared statements, flexible fetch modes.

### PHP as template engine
**Choice**: Plain PHP templates, no Twig/Blade.
**Rationale**: Vanilla means zero dependencies.

### Migration-based schema
**Choice**: Numbered SQL files + CLI runner.
**Rationale**: Version-controlled, reproducible, no external tools.

## Directory Structure

```
public/index.php + assets/
src/{Controllers,Models,Views,Middleware,Helpers,Router,Database,Config}
config/.env
database/migrations/
tests/{Unit,Integration}/
composer.json
```

## Data Flow

```
Request → index.php → Router → Middleware → Controller → Model → View → Response
```

Pipeline order: CSRF → Auth → Role → Controller.

## Database Schema (PostgreSQL)

### `users`
| Column | Type | Constraints |
|--------|------|-------------|
| id | SERIAL | PK |
| username | VARCHAR(50) | UNIQUE NOT NULL |
| email | VARCHAR(100) | UNIQUE NOT NULL |
| password_hash | VARCHAR(255) | NOT NULL |
| role | VARCHAR(20) | DEFAULT 'user' CHECK (role IN ('admin','user')) |
| created_at | TIMESTAMP | DEFAULT NOW() |

### `people`
| Column | Type | Constraints |
|--------|------|-------------|
| id | SERIAL | PK |
| user_id | INTEGER | FK → users(id) CASCADE, UNIQUE |
| first_name | VARCHAR(50) | NOT NULL |
| last_name | VARCHAR(50) | NOT NULL |
| dni | VARCHAR(20) | |
| phone | VARCHAR(30) | |

### `projects`
| Column | Type | Constraints |
|--------|------|-------------|
| id | SERIAL | PK |
| owner_id | INTEGER | FK → users(id) |
| name | VARCHAR(100) | NOT NULL |
| description | TEXT | |
| start_date | DATE | |
| end_date | DATE | |
| status | VARCHAR(20) | DEFAULT 'pending' CHECK (IN 'pending','in-progress','completed') |
| created_at | TIMESTAMP | DEFAULT NOW() |

Indexes: `idx_projects_owner_id`, `idx_projects_status`.

### `teams`
| Column | Type | Constraints |
|--------|------|-------------|
| id | SERIAL | PK |
| project_id | INTEGER | FK → projects(id) CASCADE |
| name | VARCHAR(100) | NOT NULL |
| description | TEXT | |
| created_at | TIMESTAMP | DEFAULT NOW() |

### `team_members`
| Column | Type | Constraints |
|--------|------|-------------|
| id | SERIAL | PK |
| team_id | INTEGER | FK → teams(id) CASCADE |
| user_id | INTEGER | FK → users(id) CASCADE |
| | | UNIQUE(team_id, user_id) |

### `tasks`
| Column | Type | Constraints |
|--------|------|-------------|
| id | SERIAL | PK |
| team_id | INTEGER | FK → teams(id) CASCADE |
| creator_id | INTEGER | FK → users(id) |
| name | VARCHAR(100) | NOT NULL |
| description | TEXT | |
| start_date | DATE | |
| end_date | DATE | |
| status | VARCHAR(20) | DEFAULT 'pending' CHECK (IN 'pending','in-progress','completed') |
| created_at | TIMESTAMP | DEFAULT NOW() |

Indexes: `idx_tasks_team_id`, `idx_tasks_status`, `idx_tasks_creator_id`.

### Relationships
```
users 1←→1 people (CASCADE)
users 1←→N projects (owner_id)
projects 1←→N teams (CASCADE)
teams 1←→N team_members (CASCADE)
users N←→N teams (via team_members)
teams 1←→N tasks (CASCADE)
users 1←→N tasks (creator_id)
```

## Routes

| Method | Path | Controller | Auth |
|--------|------|-----------|------|
| GET/POST | /auth/login | Auth@login | No |
| GET/POST | /auth/register | Auth@register | No |
| GET | /auth/logout | Auth@logout | Yes |
| GET/POST | /projects | Project@index/store | Yes |
| GET/POST | /projects/create | Project@create | Yes |
| GET/PUT/DELETE | /projects/{id} | Project@show/update/destroy | Yes |
| GET/POST | /projects/{id}/teams | Team@index/store | Yes |
| GET/PUT/DELETE | /teams/{id} | Team@show/update/destroy | Yes |
| POST/DELETE | /teams/{id}/members | Team@add/removeMember | Yes |
| GET/POST | /teams/{id}/tasks | Task@index/store | Yes |
| GET/PUT/DELETE | /tasks/{id} | Task@show/update/destroy | Yes |

PUT/DELETE via `_method` hidden field.

## Controllers

- **AuthController**: login, register, logout
- **ProjectController**: index, create, store, show, edit, update, destroy
- **TeamController**: index, create, store, show, edit, update, destroy, addMember, removeMember
- **TaskController**: index, create, store, show, edit, update, destroy

## Middleware

- **CsrfMiddleware**: token on GET, validate on POST/PUT/DELETE
- **AuthMiddleware**: check session, redirect if unauthenticated
- **RoleMiddleware**: check role against route requirement

## Testing Strategy

- **Unit**: Helpers (Validator, Csrf, Pagination), Models (ownership, transitions)
- **Integration**: Auth flows, CRUD with DB, middleware enforcement
- **PHPUnit**: >80% coverage on business logic

## Security

- `password_hash(PASSWORD_BCRYPT, cost: 12)`
- Session: httponly, samesite=Lax, secure (prod)
- CSRF tokens on all state-changing requests
- Prepared statements only — zero SQL concatenation
- `htmlspecialchars()` on all output
- Registration defaults to `user` role
