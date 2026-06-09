# SDD Progress — Rebuild GestionProyectos

> Guardado: 2026-06-08
> Proyecto: PHP vanilla MVC + PostgreSQL + PHPUnit
> Contexto: Rebuild desde cero de un proyecto de clase PHP 2023

---

## Estado General

| Fase | Estado |
|------|--------|
| ✅ Session Preflight | Interactivo / OpenSpec / Ask-on-risk / 400 lines |
| ✅ sdd-init | Completado |
| ✅ Exploración | Completada — análisis de ~35 archivos PHP |
| ✅ Propuesta | Completada |
| ✅ Specs (7) | Completadas — 43 reqs, 95 escenarios |
| ✅ Diseño Técnico | Completado — MVC, PostgreSQL, 25 rutas |
| ✅ Tareas (35) | Completadas — 7 fases, ~2.200 líneas |
| 🔄 **Implementación** | **En progreso — vos codeás, yo guío** |

---

## Decisiones tomadas

### Alcance v1 (Core)
- Auth (login, registro, logout, sesiones, CSRF)
- RBAC (roles admin/usuario)
- Projects CRUD
- Teams CRUD con miembros
- Tasks CRUD con estados y asignación
- PostgreSQL + migraciones
- Routing + Front Controller
- PHPUnit tests desde el día 1

### Postergado para v2
- Foro, mensajes privados, documentos, PDF, gráficos, Kanban, i18n

### Stack
- PHP vanilla MVC (sin frameworks)
- PostgreSQL 15+
- PDO con prepared statements
- PHPUnit 11+
- Front Controller con Router propio
- Sesiones seguras (httponly, samesite)
- CSRF en todos los formularios
- Config via .env

### Dinámica de trabajo
- **Opción B**: Vos codeás, yo guío y reviso
- Cada tarea la explicamos, la implementás, la revisamos

---

## Archivos generados hasta ahora

```
openspec/
├── config.yaml
├── specs/
│   ├── user-auth/spec.md
│   ├── rbac/spec.md
│   ├── project-management/spec.md
│   ├── task-management/spec.md
│   ├── team-management/spec.md
│   ├── database-layer/spec.md
│   └── routing/spec.md
└── changes/
    └── rebuild-desde-cero/
        ├── exploration.md
        ├── proposal.md
        ├── design.md
        └── tasks.md
```

---

## Tarea actual

### 1.1 — Scaffold directory structure

**A crear:**
```
GestionProyectos/
├── public/assets/{css,js,fonts}/
├── src/{Controllers,Models,Views/{auth,projects,teams,tasks,components},Middleware,Helpers,Router,Database,Config}/
├── config/
├── database/migrations/
├── tests/{Unit,Integration}/
├── storage/logs/
└── cli/
```

**Archivos:**
- `composer.json` — solo `phpunit/phpunit ^11`
- `.gitignore` — `vendor/`, `.env`, `*.log`

**Criterios:**
- [ ] Todos los directorios existen
- [ ] `composer.json` completo
- [ ] `.gitignore` configurado

---

## Próximas tareas (orden)

| # | Tarea | Depende de |
|---|-------|-----------|
| 1.2 | Config loader (.env) | 1.1 |
| 1.3 | Database connection (PDO singleton) | 1.2 |
| 1.4 | Migration runner + schema SQL | 1.3 |
| 1.5 | Router class | 1.1 |
| 1.6 | Middleware pipeline | 1.5 |
| 1.7 | Front Controller | 1.2, 1.5, 1.6 |
| 1.8 | PHPUnit config | 1.1 |
| 2.1-2.5 | Auth (User model, CSRF, AuthMiddleware, AuthController, routes) | Fase 1 |
| 3.1-3.4 | RBAC (RoleMiddleware, Authorization, audit log, admin routes) | Fase 2 |
| 4.1-4.4 | Projects CRUD + paginación | Fase 3 |
| 5.1-5.4 | Teams CRUD + miembros | Fase 4 |
| 6.1-6.4 | Tasks CRUD + state machine + asignación | Fase 5 |
| 7.1-7.6 | Hardening (Validator, Logger, Rate Limit, CSP, Seed) | Fase 6 |
