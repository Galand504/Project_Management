# Exploration: Rebuild GestionProyectos desde cero

## Executive Summary

El proyecto **GestionProyectos** es un sistema de gestión de proyectos web desarrollado con PHP 8+ vanilla, MySQL/Mysqli, y programación procedural/lineal. Generado con GPT-4.0 en 2023, presenta una arquitectura típica de prototipo rápido: cada archivo PHP es autónomo, con lógica de negocio, acceso a datos y presentación mezclados en el mismo archivo. No existe separación de capas, no hay sistema de rutas, y la configuración de BD está duplicada en ~40+ archivos. El proyecto requiere una reconstrucción completa para alcanzar una arquitectura escalable, mantenible y segura.

## Current Architecture

### Tipo de Arquitectura
- **Patrón**: Scripting procedural / "file-per-feature"
- **Base de datos**: MySQL con extensiones `mysqli`
- **Sesiones**: PHP nativo (`session_start()`)
- **Frontend**: HTML inline, CSS inline/embebido, JavaScript vanilla
- **Librerías externas**: FPDF (PDF generation), CanvasJS (gráficos vía CDN), Font Awesome (iconos vía CDN), SB Admin 2 (template admin)

### Patrones de Código
- Cada archivo PHP contiene: conexión a BD + lógica + HTML + CSS + JS
- No existe capa de modelo, vista o controlador
- No hay sistema de routing
- No hay sistema de autoloading ni Composer
- No hay tests
- No hay migraciones de BD

### Roles de Usuario
- `administrador`: Acceso completo al menú admin (CRUD completo)
- `usuario`: Acceso limitado a menú de usuario (documentos, tareas, equipos, foro)

## Feature Catalog

### 1. Authentication
- **Archivos**: `Login/Login.php`, `Login/Registrar.php`, `Login/Logout.php`
- **Funcionalidad**: Login con email/usuario, registro con rol, logout con destrucción de sesión
- **Cómo funciona**:
  - Login: Prepared statements, `password_verify()`, redirección por rol
  - Registro: Generación aleatoria de username (`nombre3+apellido3+rand`), `password_hash()` con BCRYPT, inserción en tablas `usuario` y `persona`
  - Logout: Destrucción completa de sesión y cookies
- **Problemas**:
  - `Registrar.php` usa **SQL concatenado** (VULNERABILIDAD SQL INJECTION en líneas 34-35)
  - Sin validación de CSRF en ningún formulario
  - Sin rate limiting en login
  - Sin validación de fortaleza de contraseña
  - Sin verificación de email
  - El rol se elige en el formulario de registro (cualquiera puede registrarse como admin)
  - Credenciales de BD hardcodeadas en cada archivo

### 2. Users / Profile
- **Archivos**: `Profile/profile.php`, `Profile/profileUser.php`, `Profile/update_profile.php`, `Profile/update_profile_user.php`, `Login/CRUD.php`, `MenuPrincipal/user_list.php`
- **Funcionalidad**: Ver perfil, editar datos personales, cambiar contraseña, CRUD de usuarios (admin)
- **Cómo funciona**:
  - Perfil: JOIN entre `persona` y `usuario`, formularios de edición
  - CRUD admin: Agregar/Actualizar/Eliminar usuarios completos
  - User list: Listado de todos los usuarios con enlace al CRUD
- **Problemas**:
  - Duplicación masiva: `profile.php` y `profileUser.php` son prácticamente idénticos
  - `update_profile.php` y `update_profile_user.php` son prácticamente idénticos
  - `user_list.php` usa `echo` directo sin htmlspecialchars (VULNERABILIDAD XSS)
  - CRUD.php tiene lógica de negocio compleja mezclada con presentación
  - Sin paginación en listado de usuarios
  - Sin validación de permisos (cualquier usuario podría editar otros)

### 3. Projects
- **Archivos**: `MenuPrincipal/crear_proyecto.php`, `MenuPrincipal/listar_proyectos.php`, `MenuPrincipal/proyecto.php`, `MenuPrincipal/get_proyecto_data.php`, `MenuPrincipal/update_proyecto.php`
- **Funcionalidad**: Crear proyectos, listar proyectos, actualizar datos de proyecto
- **Cómo funciona**:
  - Crear: Formulario con nombre, descripción, fechas, estado. Prepared statements
  - Listar: SELECT * sin filtros (muestra TODOS los proyectos)
  - Update: Endpoint JSON para actualización AJAX
  - Get data: Endpoint JSON para obtener datos de proyecto
- **Problemas**:
  - No hay CRUD completo de proyectos (falta DELETE y UPDATE visual)
  - Listar muestra todos los proyectos sin filtro por usuario
  - Sin paginación
  - Sin validación de fechas (fecha fin < fecha inicio)
  - Sin control de acceso por proyecto (cualquiera edita cualquiera)
  - `listar_proyectos.php` no tiene verificación de sesión

### 4. Tasks
- **Archivos**: `MenuPrincipal/tarea.php`, `MenuPrincipal/listar_tarea.php`, `MenuPrincipal/listar_tarea_Usuario.php`, `MenuPrincipal/update_task.php`
- **Funcionalidad**: Crear tareas, listar tareas, actualizar estado vía Kanban buttons
- **Cómo funciona**:
  - Crear: Formulario con equipo, nombre, descripción, fechas, estado. Prepared statements
  - Listar: Tabla con todas las tareas, botones para cambiar estado (AJAX)
  - Update: Endpoint que actualiza solo el estado
- **Problemas**:
  - No hay DELETE de tareas
  - No hay edición completa de tareas
  - Kanban es solo botones de estado, no un tablero real
  - `listar_tarea.php` y `listar_tarea_Usuario.php` son casi idénticos
  - Sin filtrado por equipo o usuario
  - Sin paginación
  - `listar_tarea.php` usa `$_SESSION['language']` sin `session_start()`

### 5. Teams
- **Archivos**: `MenuPrincipal/equipos.php`, `MenuPrincipal/listar_equipos.php`, `MenuPrincipal/listar_equipos_usuario.php`, `MenuPrincipal/unirse_equipo.php`
- **Funcionalidad**: Crear equipos (asociados a proyectos), listar equipos, unirse a equipos
- **Cómo funciona**:
  - Crear: Formulario con proyecto, nombre, descripción. Prepared statements
  - Listar: JOIN con proyecto para mostrar nombre
  - Unirse: Inserta en tabla `equipo_usuario` o `usuario_equipo` (inconsistencia)
  - Listar usuario: Muestra equipos propios y disponibles para unirse
- **Problemas**:
  - Inconsistencia en nombres de tabla: `equipo_usuario` vs `usuario_equipo`
  - No hay gestión de miembros (agregar/quitar usuarios de equipos)
  - No hay roles dentro del equipo (líder, miembro)
  - Sin eliminación de equipos
  - Sin edición de equipos
  - `listar_equipos_usuario.php` usa `unirse_equipo.php` que usa tabla diferente

### 6. Forum
- **Archivos**: `Chat/foro.php`, `Chat/foro_user.php`, `Chat/crear_forum.php`, `Chat/crear_forum_user.php`, `Chat/cargar_mensajes_foro.php`
- **Funcionalidad**: Publicar mensajes en foro, ver mensajes
- **Cómo funciona**:
  - Crear: Formulario con título y mensaje, inserción en tabla `foro`
  - Listar: Consulta con JOIN a usuario, ordenados por fecha
  - Panel integrado en menú principal (toggle)
- **Problemas**:
  - Foro muy básico: sin hilos, sin respuestas, sin categorías
  - `foro.php` y `foro_user.php` son prácticamente idénticos pero no cargan mensajes
  - `cargar_mensajes_foro.php` existe pero no se incluye en foro.php/foro_user.php
  - Sin paginación
  - Sin moderación
  - `foro_user.php` referencia variable `$mensajes` que nunca se define

### 7. Documents
- **Archivos**: `MenuPrincipal/crear_documento.php`, `MenuPrincipal/listar_documentos.php`, `MenuPrincipal/listar_documentos_usuario.php`, `MenuPrincipal/documentos.php`, `MenuPrincipal/generar_informe.php`, `MenuPrincipal/get_documentos.php`
- **Funcionalidad**: Crear documentos, listar documentos, generar informes PDF
- **Cómo funciona**:
  - Crear: Formulario con nombre, descripción, proyecto
  - Listar: Tabla con documentos y botón para generar informe
  - Generar informe: FPDF genera PDF con datos del documento
  - Get documentos: Endpoint JSON por proyecto
- **Problemas**:
  - `documentos.php` y `crear_documento.php` hacen lo mismo (duplicados)
  - No hay subida de archivos reales (es texto plano)
  - No hay descarga de documentos
  - `generar_informe.php` hardcodea el nombre del generador
  - `generar_informe.php` usa `$_POST` sin verificar sesión
  - Sin control de acceso por documento
  - Sin versionado de documentos

### 8. Reports / Graphics
- **Archivos**: `MenuPrincipal/reportes.php`, `MenuPrincipal/generar_informe.php`, `Graficos/Login.php`, `Graficos/Documento.php`
- **Funcionalidad**: Crear reportes, generar PDFs, gráficos de acceso y documentos
- **Cómo funciona**:
  - Reportes: CRUD básico de reportes en tabla `reportes`
  - Gráficos: CanvasJS mostrando accesos y documentos por día de semana
  - PDF: FPDF genera informes detallados de documentos
- **Problemas**:
  - `reportes.php` no tiene verificación de sesión
  - Gráficos usan CanvasJS vía CDN (dependencia externa)
  - Gráficos muy básicos (solo línea por día de semana)
  - Sin autenticación en páginas de gráficos
  - `reportes.php` no muestra reportes existentes, solo formulario de creación

### 9. Internationalization (i18n)
- **Archivos**: `Idioma/lang.php`, `Idioma/lang_user.php`
- **Funcionalidad**: Sistema de traducciones ES/EN
- **Cómo funciona**:
  - Archivos PHP con arrays asociativos de traducciones
  - Cambio de idioma vía parámetro GET `?lang=es|en`
  - Idioma guardado en sesión
- **Problemas**:
  - Dos archivos de traducción separados (lang.php y lang_user.php) con claves duplicadas
  - Muchas claves duplicadas dentro del mismo archivo
  - Sin fallback completo si falta una clave
  - No se traducen todos los textos (muchos hardcodeados en español)
  - Cambio de idioma vía GET sin protección CSRF
  - No hay detección automática de idioma del navegador

### 10. Kanban Board
- **Archivos**: `Kanban/kanban.html`
- **Funcionalidad**: Tablero Kanban con drag & drop
- **Cómo funciona**:
  - HTML con 3 columnas: To Do, In Progress, Done
  - JavaScript vanilla para drag & drop
  - Agregar tareas via `prompt()`
- **Problemas**:
  - **Totalmente desconectado** del resto de la aplicación
  - Sin persistencia (todo se pierde al recargar)
  - No está integrado con la BD
  - No usa las tareas existentes
  - Es un prototipo/demo sin valor funcional

### 11. Private Messages
- **Archivos**: `Chat/mensajes_privados.php`, `Chat/mensajes_privados_user.php`, `Chat/enviar_mensaje-privado.php`, `Chat/enviar_mensaje-privado_user.php`, `Chat/cargar_mensajes_privados.php`, `Chat/cargar_mensajes_privados_user.php`
- **Funcionalidad**: Enviar y recibir mensajes privados entre usuarios
- **Cómo funciona**:
  - Enviar: Busca usuario por nombre de usuario, inserta en `MensajePrivado`
  - Recibir: Lista mensajes donde el usuario es destino
  - Formulario con accordion para ver mensajes
- **Problemas**:
  - Duplicación masiva: versión admin y versión usuario del mismo código
  - Sin conversaciones agrupadas
  - Sin marcado de leído/no leído
  - Sin eliminación de mensajes
  - Sin paginación
  - `cargar_mensajes_privados.php` y `cargar_mensajes_privados_user.php` son idénticos

## Database Schema

> **Nota**: No existe archivo SQL de esquema. El siguiente esquema fue inferido de las consultas PHP en el código fuente.

### Tables (Inferred)

#### `usuario`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdUsuario | INT AUTO_INCREMENT PK | |
| NombreUsuario | VARCHAR | Generado: substr(nombre,0,3)+substr(apellido,0,3)+rand(1000,9999) |
| CorreoElectronico | VARCHAR | |
| Contraseña | VARCHAR | bcrypt hash |
| Rol | ENUM('administrador','usuario') | |
| Estado | ENUM('activo','inactivo') | |

#### `persona`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdPersona | INT AUTO_INCREMENT PK | |
| DNI | VARCHAR | |
| Nombre | VARCHAR | |
| Apellido | VARCHAR | |
| Telefono | VARCHAR | |
| IdUsuario | INT FK → usuario | |
| Rol | VARCHAR | Duplicado de usuario.Rol |
| FechaRegistro | DATETIME | |

#### `proyecto`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdProyecto | INT AUTO_INCREMENT PK | |
| NombreProyecto | VARCHAR | |
| Descripcion | TEXT | |
| FechaInicio | DATE | |
| FechaFin | DATE | |
| Estado | VARCHAR | 'En Progreso', 'Finalizado', 'Pendiente' |
| IdUsuario | INT FK → usuario | Creador del proyecto |

#### `equipos`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdEquipo | INT AUTO_INCREMENT PK | |
| IdProyecto | INT FK → proyecto | |
| Nombre | VARCHAR | |
| Descripcion | TEXT | |
| FechaCreacion | DATETIME | |
| Creador | INT FK → usuario | |

#### `tarea`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdTarea | INT AUTO_INCREMENT PK | |
| NombreTarea | VARCHAR | |
| Descripcion | TEXT | |
| FechaInicio | DATE | |
| FechaFin | DATE | |
| Estado | VARCHAR | 'Pendiente', 'En Progreso', 'Completada' |
| IdEquipo | INT FK → equipos | |
| IdUsuario | INT FK → usuario | |

#### `documento`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdDocumento | INT AUTO_INCREMENT PK | |
| NombreDocumento | VARCHAR | |
| Descripcion | TEXT | |
| FechaCreacion | DATE/DATETIME | |
| MiembroCreador | VARCHAR/INT | Inconsistente |
| Contenido | TEXT | |
| Autor | VARCHAR/INT FK → usuario | Tipo inconsistente |
| IdProyecto | INT FK → proyecto | |
| IdUsuario | INT FK → usuario | Duplicado con Autor |

#### `foro`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdForo (assumed) | INT AUTO_INCREMENT PK | |
| Titulo | VARCHAR | |
| Mensaje | TEXT | |
| FechaCreacion | DATETIME | |
| IdUsuario | INT FK → usuario | |

#### `MensajePrivado`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdMensajePrivado | INT AUTO_INCREMENT PK | |
| IdUsuarioOrigen | INT FK → usuario | |
| IdUsuarioDestino | INT FK → usuario | |
| Mensaje | TEXT | |
| FechaEnvio | DATETIME | |

#### `equipo_usuario` / `usuario_equipo`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdUsuario | INT FK → usuario | |
| IdEquipo | INT FK → equipos | |

> **Nota**: El código usa `equipo_usuario` (unirse_equipo.php) y `usuario_equipo` (listar_equipos_usuario.php) inconsistently.

#### `reportes`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdReporte (assumed) | INT AUTO_INCREMENT PK | |
| TituloReporte | VARCHAR | |
| Descripcion | TEXT | |
| FechaCreacion | DATE | |
| IdUsuario | INT FK → usuario | |

#### `informe_detallado`
| Column | Type (Inferred) | Notes |
|--------|----------------|-------|
| IdInforme (assumed) | INT AUTO_INCREMENT PK | |
| IdDocumento | INT FK → documento | |
| DatosEspecificos | TEXT | |
| FechaGeneracion | DATETIME | |
| GeneradorInforme | VARCHAR | Hardcoded |

### Relationships
```
usuario 1←→1 persona (via IdUsuario)
usuario 1←→N proyecto (via IdUsuario as creator)
usuario 1←→N foro (via IdUsuario)
usuario 1←→N MensajePrivado (via IdUsuarioOrigen/IdUsuarioDestino)
usuario 1←→N equipo_usuario (via IdUsuario)
proyecto 1←→N equipos (via IdProyecto)
proyecto 1←→N tarea (via间接 via equipos)
proyecto 1←→N documento (via IdProyecto)
equipos 1←→N tarea (via IdEquipo)
equipos 1←→N equipo_usuario (via IdEquipo)
documento 1←→N informe_detallado (via IdDocumento)
```

### Indices
- No se detectan índices explícitos más allá de las PRIMARY KEY
- No hay FOREIGN KEY constraints definidos en el código
- No hay UNIQUE constraints en CorreoElectronico

## File Structure

```
GestionProyectos/
├── Chat/                          # Foro y mensajes privados
│   ├── cargar_mensajes_foro.php   # AJAX loader for forum messages
│   ├── cargar_mensajes_privados.php
│   ├── cargar_mensajes_privados_user.php  # DUPLICATE
│   ├── chat.js
│   ├── crear_forum.php
│   ├── crear_forum_user.php       # DUPLICATE
│   ├── enviar_mensaje-privado.php
│   ├── enviar_mensaje-privado_user.php   # DUPLICATE
│   ├── foro.php
│   ├── foro_user.php              # DUPLICATE
│   ├── mensajes_privados.php
│   ├── mensajes_privados_user.php # DUPLICATE
│   └── stylechat.css
├── Graficos/                      # Charts/Graphs
│   ├── Documento.php
│   └── Login.php
├── Idioma/                        # i18n translations
│   ├── lang.php
│   └── lang_user.php
├── Image/                         # Static images
│   ├── Login.jpg
│   ├── Logosistema.jpg
│   ├── Menu.jpg
│   └── Registrar.jpg
├── Kanban/                        # Standalone Kanban prototype
│   └── kanban.html
├── Librerias/                     # Third-party libraries
│   ├── fpdf.php
│   └── font/                      # FPDF fonts
├── Login/                         # Authentication
│   ├── CRUD.php                   # User CRUD (admin)
│   ├── Login.php
│   ├── Logout.php
│   ├── Registrar.php
│   ├── get_user_data.php
│   ├── login.css
│   ├── registrar.css
│   ├── script.js
│   ├── stylecrud.css
│   └── Validar_login.js
├── MenuPrincipal/                 # Main menu & core features
│   ├── css/                       # CSS files
│   │   ├── equipos.css
│   │   ├── listar.css
│   │   ├── listar_equipos.css
│   │   ├── sb-admin-2.css
│   │   ├── sb-admin-2.min.css
│   │   ├── styles2.css
│   │   └── tarea.css
│   ├── js/                        # JS files
│   │   ├── demo/                  # SB Admin 2 demo scripts
│   │   ├── menu.js
│   │   ├── sb-admin-2.js
│   │   └── sb-admin-2.min.js
│   ├── crear_documento.php
│   ├── crear_proyecto.php
│   ├── documentos.php             # DUPLICATE of crear_documento
│   ├── equipos.php
│   ├── generar_informe.php
│   ├── get_documentos.php
│   ├── get_proyecto_data.php
│   ├── listar_documentos.php
│   ├── listar_documentos_usuario.php
│   ├── listar_equipos.php
│   ├── listar_equipos_usuario.php
│   ├── listar_proyectos.php
│   ├── listar_tarea.php
│   ├── listar_tarea_Usuario.php
│   ├── menu.php                   # Admin menu
│   ├── menuUsuario.php            # User menu
│   ├── proyecto.php
│   ├── reportes.php
│   ├── tarea.php
│   ├── unirse_equipo.php
│   ├── update_proyecto.php
│   ├── update_task.php
│   └── user_list.php
├── Profile/                       # User profile
│   ├── profile.css
│   ├── profile.php
│   ├── profileUser.php            # DUPLICATE
│   ├── update_profile.php
│   └── update_profile_user.php    # DUPLICATE
└── openspec/                      # New: SDD artifacts
    └── changes/
        └── rebuild-desde-cero/
            └── exploration.md
```

### Key Observations
- **56 archivos PHP** en total (including library fonts)
- **~35 archivos PHP de aplicación** (excluding FPDF library)
- **Mínimo 10 archivos duplicados** (admin vs user versions)
- **CSS embebido** en la mayoría de archivos PHP (no en archivos separados)
- **Sin Composer**, sin autoloading, sin dependencies management
- **SB Admin 2** incluido pero no utilizado plenamente

## Security Analysis

### SQL Injection
| Archivo | Vulnerable | Tipo |
|---------|-----------|------|
| Login/Login.php | No | Prepared statements |
| Login/Registrar.php | **SÍ** | SQL concatenado (líneas 34-35) |
| Login/CRUD.php | No | Prepared statements |
| MenuPrincipal/* | Mixto | La mayoría usa prepared statements |
| MenuPrincipal/user_list.php | Potencialmente | Sin htmlspecialchars en output |

### XSS (Cross-Site Scripting)
| Archivo | Vulnerable | Tipo |
|---------|-----------|------|
| MenuPrincipal/user_list.php | **SÍ** | Output directo sin htmlspecialchars |
| Chat/cargar_mensajes_foro.php | No | Usa htmlspecialchars |
| La mayoría de archivos | No |usan htmlspecialchars() en output |

### CSRF (Cross-Site Request Forgery)
- **TODOS los formularios** son vulnerables a CSRF
- No existe ningún token CSRF en ningún formulario
- El cambio de idioma vía GET también es vulnerable

### Session Management
- Sesiones verificadas en la mayoría de archivos (excepto algunos como `listar_tarea.php`, `documentos.php`, `reportes.php`)
- Logout correcto con destrucción de sesión
- Sin configuración de cookies seguras (httponly, secure, samesite)

### Input Validation
- Validación mínima: solo `required` en HTML
- Sin validación server-side de tipos de dato
- Sin sanitización de inputs antes de procesar
- Sin validación de longitud de campos

### Credentials
- Credenciales de BD hardcodeadas en CADA archivo: `localhost`, `root`, `""`, `sistemagestionproyectos`
- Sin variables de entorno ni archivo de configuración

### Summary
| Category | Status | Severity |
|----------|--------|----------|
| SQL Injection | 1 archivo crítico (Registrar.php) | **CRITICAL** |
| XSS | 1 archivo (user_list.php) | **HIGH** |
| CSRF | Todos los formularios | **HIGH** |
| Hardcoded Credentials | Todos los archivos | **HIGH** |
| Missing Auth | ~5 archivos | **MEDIUM** |
| Session Config | Insegura | **MEDIUM** |
| Input Validation | Mínima | **MEDIUM** |

## Recommendations

### Keep (Core del negocio — re-implementar con arquitectura correcta)
1. **Autenticación con roles** (admin/usuario) — funcionalidad core válida
2. **CRUD de Proyectos** — funcionalidad core
3. **CRUD de Tareas** con estados — funcionalidad core
4. **CRUD de Equipos** con asociación a proyectos — funcionalidad core
5. **Foro** — funcionalidad de comunicación
6. **Mensajes Privados** — funcionalidad de comunicación
7. **Documentos** con generación de PDF — funcionalidad core
8. **Perfiles de usuario** — funcionalidad core
9. **Internacionalización (ES/EN)** — funcionalidad útil

### Improve (Útil pero mal implementado)
1. **Tablero Kanban** — la idea es buena, pero está completamente desconectado. Re-implementar como tablero real con drag & drop sobre tareas de la BD
2. **Reportes/Gráficos** — mejorar con más tipos de gráficos y datos relevantes (no solo por día de semana)
3. **Gestión de Equipos** — agregar roles de equipo (líder/miembro), gestión de miembros
4. **Documentos** — implementar subida real de archivos, versionado, control de acceso
5. **Perfiles** — agregar avatar, campos adicionales, historial de actividad

### Remove (No se usa, roto, irrelevante, o duplicado)
1. **archivos duplicados admin/user** — consolidar con lógica de roles en el controlador
2. **documentos.php** — duplicado de crear_documento.php
3. **Kanban/kanban.html** — prototipo sin integrar, reemplazar por implementación real
4. **Graficos/Login.php** — gráfico mal implementado (cuenta registros de persona, no logins)
5. **SB Admin 2** — incluido pero no utilizado, eliminar dependencia
6. **CanvasJS vía CDN** — reemplazar con Chart.js local o similar
7. **cargar_mensajes_foro.php** — nunca se incluye efectivamente
8. **Tablas duplicadas** (`equipo_usuario` vs `usuario_equipo`) — consolidar

### Add (Funcionalidades que faltan)
1. **Middleware de autenticación** — centralizar verificación de sesión
2. **Sistema de permisos** — control de acceso basado en roles y propiedad
3. **CSRF tokens** — en todos los formularios
4. **Validación server-side completa** — en todos los inputs
5. **Gestión de archivos real** — subida, descarga, almacenamiento seguro
6. **Paginación** — en todos los listados
7. **Búsqueda y filtros** — en proyectos, tareas, documentos
8. **Notificaciones** — sistema de notificaciones para asignaciones, mensajes
9. **Actividad/Log** — historial de cambios en proyectos y tareas
10. **Dashboard** — resumen con métricas relevantes
11. **Migraciones de BD** — control de versiones del esquema
12. **Tests** — unitarios y de integración
13. **Configuración centralizada** — archivo `.env` o `config.php`
14. **Gestión de errores** — logging, display control
15. **Rate limiting** — protección contra abuso

## Risks

1. **Pérdida de datos**: No hay esquema SQL oficial. Se debe inferir y validar la estructura actual antes de migrar.
2. **Integridad referencial**: Las FK probablemente no están definidas en MySQL. Al migrar a PostgreSQL se deben crear explícitamente.
3. **Duplicación de tablas**: `equipo_usuario` vs `usuario_equipo` indica inconsistencias que deben resolverse.
4. **Campos inconsistentes**: `documento.Autor` puede ser VARCHAR o INT dependiendo del archivo.
5. **Sin tests**: No hay garantía de que la funcionalidad actual se preserve al reescribir.
6. **Dependencia CanvasJS**: Servidor CDN puede caer. Considerar librería local.
7. **Roles auto-asignados**: Cualquier usuario puede registrarse como administrador. Riesgo de seguridad crítica.
8. **Migración MySQL → PostgreSQL**: Diferencias en sintaxis SQL, tipos de datos, y funciones (DAYOFWEEK, etc.).

## Ready for Proposal
Yes
