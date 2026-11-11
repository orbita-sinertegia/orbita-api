# 5. Documento de base de datos — Órbita

| Dato | Valor |
|---|---|
| Motor | PostgreSQL 17 administrado por Supabase (plan gratuito) |
| Esquema documentado | `public`, más los objetos de `auth`, `storage` y `cron` que usa la aplicación |
| Fuente | **Extraído en modo solo lectura de la base en producción el 11 de noviembre de 2026** (catálogos `information_schema` y `pg_catalog`) |
| Tablas en `public` | 47 |
| Funciones en `public` | 103 |
| Repositorio de scripts | `orbita-web/sql/` (scripts incrementales; ver sección 5.8) |

**Convenciones de este documento**

- ✅ **Confirmado:** existe en la base de producción al momento de la extracción.
- 🟡 **Inferencia técnica (I):** el propósito se dedujo de los nombres de columnas y del código; no hay documentación de origen.
- ⚠️ **Pendiente de verificar o no aplicado:** está en el código o en `sql/` pero no en la base.

No se incluyen datos personales, contraseñas, tokens ni claves.

## 5.1 Hallazgos importantes

| # | Hallazgo | Estado | Impacto |
|---|---|---|---|
| 1 | La tabla `appointment_files`, la función `register_appointment_file` y el bucket `appointment-evidence` (script `sql/2026-11-10_evidencia_citas.sql`) **no existen en la base**. | ⚠️ No aplicado | La evidencia de citas (fotos o PDF) de la app Android fallará hasta correr el script. |
| 2 | Los buckets `avatars`, `chat-files` y `client-covers` son **públicos**: cualquiera con el enlace abre el archivo. | ✅ Confirmado | Riesgo de privacidad para archivos de chat. Así lo dice la política de privacidad. |
| 3 | La carpeta `sql/` por sí sola no recrea la base: la tabla `users` y el trigger de alta de Auth vienen de la copia de estructura original. | ✅ Confirmado (notas del proyecto) | Para un entorno nuevo hace falta un volcado de esquema (`pg_dump --schema-only`). |
| 4 | `workflow_chains` no tiene datos. | ✅ Confirmado | Las tareas encadenadas no se activan. |
| 5 | Al momento de la extracción: 3 usuarios, 0 proyectos, 0 tareas y 0 citas. | ✅ Confirmado | No hay evidencia de pruebas de punta a punta con datos reales en producción. |

## 5.2 Modelo general y relaciones principales

```
auth.users 1───1 users ──< user_permissions          role_permissions (por rol)
                  │
clients >──1 users (rol cliente)     clients ──< client_team_members >── users (equipo)
   │
   └──< projects ──< tasks ──< task_comments / task_links / task_subtasks / task_ratings / task_time_entries
            │          └──< task_audit_log, project_delay_events, task_leader_delays
            │
            └──< appointments ──< appointment_participants >── users
                       ├──< appointment_agreements      appointment_kinds (catálogo)
                       └──< appointment_events          appointment_places (catálogo)

users ──< work_sessions ──< work_session_corrections        users ──< notifications, device_tokens
project_templates ──< template_tasks                          chat_rooms ──< chat_messages
direct_conversations ──< direct_messages                      client_invitations / team_invitations
```

Las relaciones exactas (llaves foráneas y su comportamiento al borrar) se listan por tabla en la sección 5.4.

## 5.3 Seguridad del acceso a datos

- **RLS:** las tablas tienen *Row Level Security* activado (ver columna RLS en el índice). Sin sesión, las tablas devuelven 0 filas. Lo probó una consulta anónima durante la configuración (ver `sql/2026-11-05_cerrar_privacidad.sql`).
- **Funciones de ayuda:** `is_active_admin()`, `app_is_staff()`, `app_client_owns_project(uuid)`, `app_client_owns_task(uuid)` y `app_is_appointment_member(uuid)` deciden qué filas ve cada rol.
- **Escrituras de citas, asistencia y evidencia:** solo por funciones `SECURITY DEFINER`, que validan rol, estado y datos. Las tablas de citas solo tienen `SELECT` para `authenticated`.
- **Registro:** el trigger `on_auth_user_created` (antes de insertar en `auth.users`) rechaza correos sin invitación pendiente.

## 5.4 Índice de tablas

| # | Tabla | Módulo | RLS | Políticas | Realtime | Descripción |
|---|---|---|---|---|---|---|
| 1 | `client_team_members` | Núcleo | Sí | 4 | — | Integrantes del equipo asignados a cada cliente. |
| 2 | `clients` | Núcleo | Sí | 3 | — | Clientes de la agencia. `user_id` liga al usuario con rol cliente que accede al portal. |
| 3 | `users` | Núcleo | Sí | 4 | — | Perfil de cada cuenta (admin, equipo o cliente): rol, departamento, horario, estado activo y superadmin. El `id` coincide con `auth.users`. |
| 4 | `app_license` | Acceso | Sí | 2 | — | Licencia de uso de la plataforma (activa o suspendida) y sus datos administrativos. |
| 5 | `client_invitations` | Acceso | Sí | 1 | — | Invitaciones de cliente: correo, cliente y permisos iniciales. Se marcan como usadas al aceptarlas. |
| 6 | `role_permissions` | Acceso | Sí | 2 | — | Permisos por rol y sección de la interfaz. |
| 7 | `team_invitations` | Acceso | Sí | 1 | — | Invitaciones de equipo o administración: correo, nombre, rol, departamento y permisos. |
| 8 | `user_permissions` | Acceso | Sí | 2 | — | Permisos individuales por persona y sección; tienen prioridad sobre los del rol. |
| 9 | `project_delay_events` | Proyectos y tareas | Sí | 2 | — | Historial de retrasos de un proyecto, con su responsable (cliente o agencia) y los días. |
| 10 | `project_plan` | Proyectos y tareas | Sí | 1 | — | (I) Prioridad y fecha objetivo por proyecto, para planificación. |
| 11 | `project_templates` | Proyectos y tareas | Sí | 1 | — | Embudos (plantillas de proyecto) con nombre, icono y bandera de archivado. |
| 12 | `projects` | Proyectos y tareas | Sí | 2 | — | Proyectos de cada cliente: estado, modalidad de grabación, fecha de lanzamiento y embudo de origen. |
| 13 | `task_audit_log` | Proyectos y tareas | Sí | 1 | — | Bitácora de cambios hechos por administradores a las tareas (antes y después). |
| 14 | `task_comments` | Proyectos y tareas | Sí | 3 | — | Comentarios de equipo y clientes en una tarea. |
| 15 | `task_leader_delays` | Proyectos y tareas | Sí | 1 | — | (I) Retrasos atribuidos al líder por tarea, con justificación opcional. |
| 16 | `task_links` | Proyectos y tareas | Sí | 8 | — | Entregables: enlaces o archivos adjuntos a una tarea. |
| 17 | `task_plans` | Proyectos y tareas | Sí | 1 | — | (I) Planeación de inicio y fin de una tarea. |
| 18 | `task_ratings` | Proyectos y tareas | Sí | 3 | — | Calificación (estrellas y comentario) que el cliente da a una tarea. |
| 19 | `task_requests` | Proyectos y tareas | Sí | 3 | Sí | Solicitudes de tareas enviadas por clientes, con estado pendiente, aprobada o rechazada. |
| 20 | `task_subtasks` | Proyectos y tareas | Sí | 4 | — | Subtareas (lista de verificación) de una tarea. |
| 21 | `task_time_entries` | Proyectos y tareas | Sí | 4 | — | Registros de tiempo trabajado en una tarea. |
| 22 | `tasks` | Proyectos y tareas | Sí | 6 | — | Tareas: asignado, estado, prioridad, fechas, revisión, aprobación del cliente, fase y origen del embudo. |
| 23 | `template_tasks` | Proyectos y tareas | Sí | 1 | — | Tareas predefinidas de cada embudo (título, departamento y prioridad). |
| 24 | `workflow_chain_steps` | Proyectos y tareas | Sí | 1 | — | (I) Pasos de cada cadena de tareas. |
| 25 | `workflow_chains` | Proyectos y tareas | Sí | 1 | — | (I) Cadenas de tareas encadenadas por plantilla. **Sin datos en esta base**: la función de tareas encadenadas queda inactiva. |
| 26 | `workflow_owners` | Proyectos y tareas | Sí | 1 | — | (I) Responsables por clave de dueño en las cadenas. |
| 27 | `workflow_settings` | Proyectos y tareas | Sí | 1 | — | (I) Interruptor global de las cadenas de tareas. |
| 28 | `daily_closures` | Equipo | Sí | 3 | — | (I) Cierre diario de cada persona: tareas terminadas, pendientes y resumen. |
| 29 | `team_departments` | Equipo | Sí | 2 | — | Catálogo editable de departamentos (roles operativos). |
| 30 | `team_goals` | Equipo | Sí | 2 | — | (I) Metas semanales por persona o departamento y tipo de trabajo; las usaba el Tablero semanal, ya retirado de la web. |
| 31 | `work_type_weights` | Equipo | Sí | 2 | — | (I) Puntos por tipo de trabajo; los usaba el Tablero semanal, ya retirado. |
| 32 | `attendance_alerts` | Asistencia | Sí | 0 | — | Control para no repetir las alertas de "sin registro" y el resumen diario. |
| 33 | `user_daily_activity` | Asistencia | Sí | 1 | — | Primera y última actividad del día por persona. |
| 34 | `work_session_corrections` | Asistencia | Sí | 1 | — | Correcciones de sesiones hechas por un administrador, con el motivo. |
| 35 | `work_sessions` | Asistencia | Sí | 1 | Sí | Sesiones de trabajo (entrada y salida), con modalidad, retardo y minutos regulares y extra. |
| 36 | `chat_messages` | Comunicación | Sí | 2 | — | Mensajes de chat con archivos, respuestas, reacciones, fijados y leídos. |
| 37 | `chat_rooms` | Comunicación | Sí | 2 | — | Salas de chat: la del equipo y una por cliente. |
| 38 | `device_tokens` | Comunicación | Sí | 0 | — | Tokens de Firebase de los teléfonos con la app, para notificaciones push. Solo se usa por funciones. |
| 39 | `direct_conversations` | Comunicación | Sí | 2 | — | Conversaciones directas entre dos personas. |
| 40 | `direct_messages` | Comunicación | Sí | 4 | Sí | Mensajes de las conversaciones directas. |
| 41 | `notifications` | Comunicación | Sí | 4 | — | Avisos de la campana web y de la app móvil (tipo, mensaje, tarea o cita relacionada, leído y enviado por push). |
| 42 | `appointment_agreements` | Citas | Sí | 1 | — | Acuerdos de una cita, cada uno marcado como visible o no para el cliente. |
| 43 | `appointment_events` | Citas | Sí | 1 | — | Historial de cada cita (solicitada, programada, respuestas, recordatorios, cierre). |
| 44 | `appointment_kinds` | Citas | Sí | 1 | — | Catálogo de tipos de cita con su modalidad sugerida. Los administradores pueden agregar tipos. |
| 45 | `appointment_participants` | Citas | Sí | 1 | Sí | Invitados de cada cita: obligatorio u opcional, respuesta, asistencia y franja ocupada (`slot`) para evitar empalmes. |
| 46 | `appointment_places` | Citas | Sí | 1 | — | Estudios o lugares de la agencia, con dirección y coordenadas. |
| 47 | `appointments` | Citas | Sí | 1 | Sí | Citas: tipo, estado, modalidad, lugar o enlace, horario, horarios propuestos, solicitante y coordinador. |

## 5.5 Detalle de tablas

### 5.5.1 `client_team_members`

Integrantes del equipo asignados a cada cliente.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `client_id` | uuid | No |  |
| `user_id` | uuid | No |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `client_team_members_pkey` | `PRIMARY KEY (client_id, user_id)` |
| Llave foránea | `client_team_members_client_id_fkey` | `FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE` |
| Llave foránea | `client_team_members_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |

### 5.5.2 `clients`

Clientes de la agencia. `user_id` liga al usuario con rol cliente que accede al portal.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `name` | text | No |  |
| `company` | text | Sí |  |
| `user_id` | uuid | Sí |  |
| `created_at` | timestamptz | Sí | `now()` |
| `notes` | text | Sí |  |
| `cover_image` | text | Sí |  |
| `active` | boolean | No | `true` |
| `directory_hidden` | boolean | No | `false` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `clients_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `clients_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id)` |

### 5.5.3 `users`

Perfil de cada cuenta (admin, equipo o cliente): rol, departamento, horario, estado activo y superadmin. El `id` coincide con `auth.users`.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No |  |
| `email` | text | No |  |
| `full_name` | text | No |  |
| `role` | text | No |  |
| `department` | text | Sí |  |
| `created_at` | timestamptz | Sí | `now()` |
| `must_change_password` | boolean | Sí | `true` |
| `avatar_url` | text | Sí |  |
| `work_start` | integer | Sí | `9` |
| `work_end` | integer | Sí | `17` |
| `active` | boolean | No | `true` |
| `is_super_admin` | boolean | No | `false` |
| `directory_hidden` | boolean | No | `false` |
| `work_schedule` | jsonb | Sí | `jsonb_build_object('0', jsonb_build_object('enabled', fal…` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `users_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `users_id_fkey` | `FOREIGN KEY (id) REFERENCES auth.users(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `users_role_check` | `CHECK ((role = ANY (ARRAY['admin'::text, 'team'::text, 'client'::text])))` |

### 5.5.4 `app_license`

Licencia de uso de la plataforma (activa o suspendida) y sus datos administrativos.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | integer | No | `1` |
| `status` | text | No | `'active'::text` |
| `client_name` | text | Sí |  |
| `expires_at` | date | Sí |  |
| `notes` | text | Sí |  |
| `updated_at` | timestamptz | No | `now()` |
| `start_date` | date | Sí |  |
| `monthly_amount` | numeric | No | `0` |
| `payment_link` | text | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `app_license_pkey` | `PRIMARY KEY (id)` |
| Verificación (CHECK) | `app_license_status_check` | `CHECK ((status = ANY (ARRAY['active'::text, 'suspended'::text])))` |
| Verificación (CHECK) | `single_row` | `CHECK ((id = 1))` |

### 5.5.5 `client_invitations`

Invitaciones de cliente: correo, cliente y permisos iniciales. Se marcan como usadas al aceptarlas.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `client_id` | uuid | No |  |
| `created_at` | timestamptz | Sí | `now()` |
| `used` | boolean | Sí | `false` |
| `email` | text | Sí |  |
| `permissions` | jsonb | No | `'{}'::jsonb` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `client_invitations_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `client_invitations_client_id_fkey` | `FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `client_invitations_email_required` | `CHECK ((NULLIF(TRIM(BOTH FROM email), ''::text) IS NOT NULL)) NOT VALID` |

### 5.5.6 `role_permissions`

Permisos por rol y sección de la interfaz.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `role` | text | No |  |
| `section_key` | text | No |  |
| `allowed` | boolean | No | `true` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `role_permissions_pkey` | `PRIMARY KEY (id)` |
| Única | `role_permissions_role_section_key_key` | `UNIQUE (role, section_key)` |
| Verificación (CHECK) | `role_permissions_role_check` | `CHECK ((role = ANY (ARRAY['team'::text, 'client'::text])))` |

### 5.5.7 `team_invitations`

Invitaciones de equipo o administración: correo, nombre, rol, departamento y permisos.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `email` | text | No |  |
| `full_name` | text | Sí |  |
| `role` | text | No |  |
| `department` | text | Sí |  |
| `used` | boolean | No | `false` |
| `created_at` | timestamptz | No | `now()` |
| `permissions` | jsonb | No | `'{}'::jsonb` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `team_invitations_pkey` | `PRIMARY KEY (id)` |
| Verificación (CHECK) | `team_invitations_role_check` | `CHECK ((role = ANY (ARRAY['team'::text, 'admin'::text])))` |

### 5.5.8 `user_permissions`

Permisos individuales por persona y sección; tienen prioridad sobre los del rol.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `user_id` | uuid | No |  |
| `section_key` | text | No |  |
| `allowed` | boolean | No | `true` |
| `created_at` | timestamptz | No | `now()` |
| `updated_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `user_permissions_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `user_permissions_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |
| Única | `user_permissions_user_id_section_key_key` | `UNIQUE (user_id, section_key)` |

### 5.5.9 `project_delay_events`

Historial de retrasos de un proyecto, con su responsable (cliente o agencia) y los días.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `project_id` | uuid | No |  |
| `task_id` | uuid | Sí |  |
| `responsible` | text | No |  |
| `days` | integer | Sí |  |
| `reason` | text | Sí |  |
| `created_by` | uuid | Sí |  |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `project_delay_events_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `project_delay_events_created_by_fkey` | `FOREIGN KEY (created_by) REFERENCES users(id)` |
| Llave foránea | `project_delay_events_project_id_fkey` | `FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE` |
| Llave foránea | `project_delay_events_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE SET NULL` |
| Verificación (CHECK) | `project_delay_events_responsible_check` | `CHECK ((responsible = ANY (ARRAY['cliente'::text, 'agencia'::text])))` |

### 5.5.10 `project_plan`

(I) Prioridad y fecha objetivo por proyecto, para planificación.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `project_id` | uuid | No |  |
| `priority` | integer | Sí |  |
| `target_date` | date | Sí |  |
| `updated_at` | timestamptz | No | `now()` |
| `updated_by` | uuid | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `project_plan_pkey` | `PRIMARY KEY (project_id)` |
| Llave foránea | `project_plan_project_id_fkey` | `FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE` |
| Llave foránea | `project_plan_updated_by_fkey` | `FOREIGN KEY (updated_by) REFERENCES users(id) ON DELETE SET NULL` |
| Verificación (CHECK) | `project_plan_priority_check` | `CHECK (((priority IS NULL) OR (priority >= 1)))` |

### 5.5.11 `project_templates`

Embudos (plantillas de proyecto) con nombre, icono y bandera de archivado.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | text | No |  |
| `name` | text | No |  |
| `icon` | text | No | `'🗂️'::text` |
| `description` | text | Sí |  |
| `created_at` | timestamptz | No | `now()` |
| `archived` | boolean | No | `false` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `project_templates_pkey` | `PRIMARY KEY (id)` |

### 5.5.12 `projects`

Proyectos de cada cliente: estado, modalidad de grabación, fecha de lanzamiento y embudo de origen.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `client_id` | uuid | Sí |  |
| `name` | text | No |  |
| `description` | text | Sí |  |
| `status` | text | Sí | `'activo'::text` |
| `created_at` | timestamptz | Sí | `now()` |
| `recording_mode` | text | No | `'oficina'::text` |
| `launch_date` | date | Sí |  |
| `launch_date_original` | date | Sí |  |
| `stage` | text | Sí |  |
| `funnel_type` | text | Sí |  |
| `source_template_id` | text | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `projects_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `projects_client_id_fkey` | `FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `projects_funnel_type_check` | `CHECK (((funnel_type IS NULL) OR (funnel_type = ANY (ARRAY['low2funnel'::text, 'vsl_high_ticket'::text, 'high_ticket'::text, 'restaurante'::text, 'webinar'::text, 'otro'::text]))))` |
| Verificación (CHECK) | `projects_recording_mode_check` | `CHECK ((recording_mode = ANY (ARRAY['oficina'::text, 'remota'::text])))` |
| Verificación (CHECK) | `projects_stage_check` | `CHECK (((stage IS NULL) OR (stage = ANY (ARRAY['en_proceso'::text, 'corriendo'::text]))))` |
| Verificación (CHECK) | `projects_status_check` | `CHECK ((status = ANY (ARRAY['activo'::text, 'pausado'::text, 'completado'::text])))` |

### 5.5.13 `task_audit_log`

Bitácora de cambios hechos por administradores a las tareas (antes y después).

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `task_id` | uuid | No |  |
| `actor_id` | uuid | Sí |  |
| `at` | timestamptz | No | `now()` |
| `action` | text | No |  |
| `before` | jsonb | Sí |  |
| `after` | jsonb | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `task_audit_log_pkey` | `PRIMARY KEY (id)` |
| Verificación (CHECK) | `task_audit_log_action_check` | `CHECK ((action = ANY (ARRAY['update'::text, 'archive'::text, 'restore'::text, 'delete'::text])))` |

### 5.5.14 `task_comments`

Comentarios de equipo y clientes en una tarea.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `task_id` | uuid | Sí |  |
| `user_id` | uuid | Sí |  |
| `content` | text | No |  |
| `created_at` | timestamptz | Sí | `now()` |
| `image_url` | text | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `task_comments_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `task_comments_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE` |
| Llave foránea | `task_comments_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id)` |

### 5.5.15 `task_leader_delays`

(I) Retrasos atribuidos al líder por tarea, con justificación opcional.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `task_id` | uuid | Sí |  |
| `project_id` | uuid | Sí |  |
| `user_id` | uuid | Sí |  |
| `task_title` | text | Sí |  |
| `kind` | text | No |  |
| `planned_date` | date | No |  |
| `actual_date` | date | No |  |
| `business_days` | integer | No |  |
| `created_by` | uuid | Sí |  |
| `created_at` | timestamptz | No | `now()` |
| `excused_at` | timestamptz | Sí |  |
| `excused_by` | uuid | Sí |  |
| `excuse_reason` | text | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `task_leader_delays_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `task_leader_delays_created_by_fkey` | `FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL` |
| Llave foránea | `task_leader_delays_excused_by_fkey` | `FOREIGN KEY (excused_by) REFERENCES users(id) ON DELETE SET NULL` |
| Llave foránea | `task_leader_delays_project_id_fkey` | `FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE SET NULL` |
| Llave foránea | `task_leader_delays_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE SET NULL` |
| Llave foránea | `task_leader_delays_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL` |
| Verificación (CHECK) | `task_leader_delays_business_days_check` | `CHECK ((business_days >= 0))` |
| Verificación (CHECK) | `task_leader_delays_kind_check` | `CHECK ((kind = ANY (ARRAY['inicio'::text, 'entrega'::text])))` |

### 5.5.16 `task_links`

Entregables: enlaces o archivos adjuntos a una tarea.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `task_id` | uuid | Sí |  |
| `label` | text | No |  |
| `url` | text | No |  |
| `created_at` | timestamptz | Sí | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `task_links_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `task_links_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE` |

### 5.5.17 `task_plans`

(I) Planeación de inicio y fin de una tarea.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `task_id` | uuid | No |  |
| `planned_start` | date | No |  |
| `planned_end` | date | No |  |
| `planned_at` | timestamptz | No | `now()` |
| `planned_by` | uuid | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `task_plans_pkey` | `PRIMARY KEY (task_id)` |
| Llave foránea | `task_plans_planned_by_fkey` | `FOREIGN KEY (planned_by) REFERENCES users(id) ON DELETE SET NULL` |
| Llave foránea | `task_plans_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `task_plans_range` | `CHECK ((planned_end >= planned_start))` |

### 5.5.18 `task_ratings`

Calificación (estrellas y comentario) que el cliente da a una tarea.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `task_id` | uuid | Sí |  |
| `client_id` | uuid | Sí |  |
| `stars` | integer | No |  |
| `comment` | text | Sí |  |
| `created_at` | timestamptz | Sí | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `task_ratings_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `task_ratings_client_id_fkey` | `FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE` |
| Llave foránea | `task_ratings_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE` |
| Única | `task_ratings_task_id_client_id_key` | `UNIQUE (task_id, client_id)` |
| Verificación (CHECK) | `task_ratings_stars_check` | `CHECK (((stars >= 1) AND (stars <= 5)))` |

### 5.5.19 `task_requests`

Solicitudes de tareas enviadas por clientes, con estado pendiente, aprobada o rechazada.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `client_id` | uuid | No |  |
| `requester_id` | uuid | No |  |
| `project_id` | uuid | Sí |  |
| `department` | text | No |  |
| `assigned_to` | uuid | Sí |  |
| `title` | text | No |  |
| `description` | text | Sí |  |
| `status` | text | No | `'pendiente'::text` |
| `created_at` | timestamptz | No | `now()` |
| `updated_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `task_requests_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `task_requests_assigned_to_fkey` | `FOREIGN KEY (assigned_to) REFERENCES users(id) ON DELETE SET NULL` |
| Llave foránea | `task_requests_client_id_fkey` | `FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE` |
| Llave foránea | `task_requests_project_id_fkey` | `FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE SET NULL` |
| Llave foránea | `task_requests_requester_id_fkey` | `FOREIGN KEY (requester_id) REFERENCES users(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `task_requests_status_check` | `CHECK ((status = ANY (ARRAY['pendiente'::text, 'aprobada'::text, 'rechazada'::text])))` |

### 5.5.20 `task_subtasks`

Subtareas (lista de verificación) de una tarea.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `task_id` | uuid | No |  |
| `title` | text | No |  |
| `completed` | boolean | No | `false` |
| `position` | integer | No | `0` |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `task_subtasks_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `task_subtasks_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE` |

### 5.5.21 `task_time_entries`

Registros de tiempo trabajado en una tarea.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `task_id` | uuid | No |  |
| `user_id` | uuid | No |  |
| `started_at` | timestamptz | No | `now()` |
| `ended_at` | timestamptz | Sí |  |
| `minutes` | integer | Sí |  |
| `note` | text | Sí |  |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `task_time_entries_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `task_time_entries_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE` |
| Llave foránea | `task_time_entries_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |

### 5.5.22 `tasks`

Tareas: asignado, estado, prioridad, fechas, revisión, aprobación del cliente, fase y origen del embudo.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `project_id` | uuid | Sí |  |
| `title` | text | No |  |
| `description` | text | Sí |  |
| `assigned_to` | uuid | Sí |  |
| `department` | text | Sí |  |
| `status` | text | Sí | `'pendiente'::text` |
| `priority` | text | Sí | `'media'::text` |
| `start_date` | date | Sí |  |
| `commitment_date` | date | Sí |  |
| `completion_date` | date | Sí |  |
| `created_by` | uuid | Sí |  |
| `created_at` | timestamptz | Sí | `now()` |
| `updated_at` | timestamptz | Sí | `now()` |
| `needed_date` | date | Sí |  |
| `reviewer_id` | uuid | Sí |  |
| `review_status` | text | Sí |  |
| `client_approval` | text | Sí |  |
| `change_count` | integer | Sí | `0` |
| `depends_on_task_id` | uuid | Sí |  |
| `approver_id` | uuid | Sí |  |
| `approver_ids` | _uuid | No | `'{}'::uuid[]` |
| `delay_responsible` | text | Sí |  |
| `delay_days` | integer | Sí |  |
| `delay_reason` | text | Sí |  |
| `delay_marked_at` | timestamptz | Sí |  |
| `delay_marked_by` | uuid | Sí |  |
| `phase` | text | Sí |  |
| `archived_at` | timestamptz | Sí |  |
| `archived_by` | uuid | Sí |  |
| `archive_reason` | text | Sí |  |
| `work_type` | text | Sí |  |
| `work_units` | numeric | Sí |  |
| `archived_via` | text | Sí |  |
| `chain_key` | text | Sí |  |
| `chain_step` | integer | Sí |  |
| `chain_round` | integer | No | `1` |
| `chain_target_task_id` | uuid | Sí |  |
| `chain_outcome` | text | Sí |  |
| `chain_note` | text | Sí |  |
| `chain_date` | date | Sí |  |
| `progress_pct` | smallint | Sí |  |
| `sort_order` | integer | Sí |  |
| `planned_start_date` | date | Sí |  |
| `source_template_id` | text | Sí |  |
| `source_template_key` | text | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `tasks_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `tasks_approver_id_fkey` | `FOREIGN KEY (approver_id) REFERENCES users(id) ON DELETE SET NULL` |
| Llave foránea | `tasks_archived_by_fkey` | `FOREIGN KEY (archived_by) REFERENCES users(id) ON DELETE SET NULL` |
| Llave foránea | `tasks_assigned_to_fkey` | `FOREIGN KEY (assigned_to) REFERENCES users(id)` |
| Llave foránea | `tasks_chain_target_task_id_fkey` | `FOREIGN KEY (chain_target_task_id) REFERENCES tasks(id) ON DELETE SET NULL` |
| Llave foránea | `tasks_created_by_fkey` | `FOREIGN KEY (created_by) REFERENCES users(id)` |
| Llave foránea | `tasks_delay_marked_by_fkey` | `FOREIGN KEY (delay_marked_by) REFERENCES users(id)` |
| Llave foránea | `tasks_depends_on_task_id_fkey` | `FOREIGN KEY (depends_on_task_id) REFERENCES tasks(id) ON DELETE SET NULL` |
| Llave foránea | `tasks_project_id_fkey` | `FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE` |
| Llave foránea | `tasks_reviewer_id_fkey` | `FOREIGN KEY (reviewer_id) REFERENCES users(id)` |
| Verificación (CHECK) | `tasks_chain_outcome_check` | `CHECK (((chain_outcome IS NULL) OR (chain_outcome = ANY (ARRAY['aprobada'::text, 'regresada'::text]))))` |
| Verificación (CHECK) | `tasks_delay_responsible_check` | `CHECK ((delay_responsible = ANY (ARRAY['cliente'::text, 'agencia'::text])))` |
| Verificación (CHECK) | `tasks_phase_check` | `CHECK (((phase IS NULL) OR (phase = ANY (ARRAY['entrada'::text, 'estrategia'::text, 'guiones'::text, 'produccion_video'::text, 'produccion_paginas'::text, 'revision'::text, 'lanzamiento'::text, 'optimizacion'::text]))))` |
| Verificación (CHECK) | `tasks_priority_check` | `CHECK ((priority = ANY (ARRAY['critica'::text, 'alta'::text, 'media'::text, 'baja'::text])))` |
| Verificación (CHECK) | `tasks_progress_pct_check` | `CHECK (((progress_pct IS NULL) OR ((progress_pct >= 0) AND (progress_pct <= 100))))` |
| Verificación (CHECK) | `tasks_review_status_check` | `CHECK ((review_status = ANY (ARRAY['pendiente_revision'::text, 'pendiente_trafficker'::text, 'en_revision'::text, 'aprobada'::text, 'cambios_solicitados'::text])))` |
| Verificación (CHECK) | `tasks_status_check` | `CHECK ((status = ANY (ARRAY['pendiente'::text, 'en_progreso'::text, 'completada'::text, 'bloqueada'::text])))` |
| Verificación (CHECK) | `tasks_work_units_check` | `CHECK (((work_units IS NULL) OR (work_units > (0)::numeric)))` |

### 5.5.23 `template_tasks`

Tareas predefinidas de cada embudo (título, departamento y prioridad).

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `title` | text | No |  |
| `department` | text | No |  |
| `priority` | text | No | `'alta'::text` |
| `template_id` | text | No |  |
| `created_at` | timestamptz | Sí | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `template_tasks_pkey` | `PRIMARY KEY (id)` |

### 5.5.24 `workflow_chain_steps`

(I) Pasos de cada cadena de tareas.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `chain_key` | text | No |  |
| `step` | integer | No |  |
| `step_key` | text | No |  |
| `title` | text | No |  |
| `department` | text | No |  |
| `kind` | text | No |  |
| `owner_key` | text | No |  |
| `target_step` | integer | Sí |  |
| `phase` | text | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `workflow_chain_steps_pkey` | `PRIMARY KEY (chain_key, step)` |
| Llave foránea | `workflow_chain_steps_chain_key_fkey` | `FOREIGN KEY (chain_key) REFERENCES workflow_chains(chain_key) ON DELETE CASCADE` |
| Verificación (CHECK) | `workflow_chain_steps_kind_check` | `CHECK ((kind = ANY (ARRAY['produccion'::text, 'revision'::text, 'revision_final'::text, 'compartir'::text, 'cliente'::text, 'fecha'::text])))` |

### 5.5.25 `workflow_chains`

(I) Cadenas de tareas encadenadas por plantilla. **Sin datos en esta base**: la función de tareas encadenadas queda inactiva.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `chain_key` | text | No |  |
| `label` | text | No |  |
| `templates` | _text | No | `'{}'::text[]` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `workflow_chains_pkey` | `PRIMARY KEY (chain_key)` |

### 5.5.26 `workflow_owners`

(I) Responsables por clave de dueño en las cadenas.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `owner_key` | text | No |  |
| `user_id` | uuid | No |  |
| `position` | integer | No | `1` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `workflow_owners_pkey` | `PRIMARY KEY (owner_key, user_id)` |
| Llave foránea | `workflow_owners_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |

### 5.5.27 `workflow_settings`

(I) Interruptor global de las cadenas de tareas.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | integer | No | `1` |
| `enabled` | boolean | No | `true` |
| `updated_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `workflow_settings_pkey` | `PRIMARY KEY (id)` |
| Verificación (CHECK) | `workflow_settings_id_check` | `CHECK ((id = 1))` |

### 5.5.28 `daily_closures`

(I) Cierre diario de cada persona: tareas terminadas, pendientes y resumen.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `user_id` | uuid | No | `auth.uid()` |
| `work_date` | date | No |  |
| `submitted_at` | timestamptz | No | `now()` |
| `finished_count` | integer | No | `0` |
| `pending_count` | integer | No | `0` |
| `summary` | jsonb | No | `'[]'::jsonb` |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `daily_closures_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `daily_closures_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |
| Única | `daily_closures_user_date_key` | `UNIQUE (user_id, work_date)` |
| Verificación (CHECK) | `daily_closures_finished_count_check` | `CHECK ((finished_count >= 0))` |
| Verificación (CHECK) | `daily_closures_pending_count_check` | `CHECK ((pending_count >= 0))` |

### 5.5.29 `team_departments`

Catálogo editable de departamentos (roles operativos).

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | text | No |  |
| `name` | text | No |  |
| `active` | boolean | No | `true` |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `team_departments_pkey` | `PRIMARY KEY (id)` |
| Única | `team_departments_name_key` | `UNIQUE (name)` |
| Verificación (CHECK) | `team_departments_id_check` | `CHECK ((id ~ '^[a-z0-9_]{2,40}$'::text))` |
| Verificación (CHECK) | `team_departments_name_check` | `CHECK (((length(TRIM(BOTH FROM name)) >= 2) AND (length(TRIM(BOTH FROM name)) <= 60)))` |

### 5.5.30 `team_goals`

(I) Metas semanales por persona o departamento y tipo de trabajo; las usaba el Tablero semanal, ya retirado de la web.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `user_id` | uuid | Sí |  |
| `department` | text | Sí |  |
| `work_type` | text | No |  |
| `weekly_target` | numeric | No |  |
| `updated_by` | uuid | Sí |  |
| `updated_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `team_goals_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `team_goals_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `team_goals_scope` | `CHECK (((user_id IS NOT NULL) <> (department IS NOT NULL)))` |
| Verificación (CHECK) | `team_goals_weekly_target_check` | `CHECK ((weekly_target >= (0)::numeric))` |

### 5.5.31 `work_type_weights`

(I) Puntos por tipo de trabajo; los usaba el Tablero semanal, ya retirado.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `work_type` | text | No |  |
| `label` | text | No |  |
| `points` | numeric | No |  |
| `updated_by` | uuid | Sí |  |
| `updated_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `work_type_weights_pkey` | `PRIMARY KEY (work_type)` |
| Verificación (CHECK) | `work_type_weights_points_check` | `CHECK ((points >= (0)::numeric))` |

### 5.5.32 `attendance_alerts`

Control para no repetir las alertas de "sin registro" y el resumen diario.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `user_id` | uuid | No |  |
| `date` | date | No |  |
| `kind` | text | No |  |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `attendance_alerts_pkey` | `PRIMARY KEY (user_id, date, kind)` |
| Llave foránea | `attendance_alerts_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |

### 5.5.33 `user_daily_activity`

Primera y última actividad del día por persona.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `user_id` | uuid | No |  |
| `date` | date | No |  |
| `first_activity_at` | timestamptz | No |  |
| `first_activity_kind` | text | Sí |  |
| `last_activity_at` | timestamptz | No |  |
| `events` | integer | No | `1` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `user_daily_activity_pkey` | `PRIMARY KEY (user_id, date)` |
| Llave foránea | `user_daily_activity_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |

### 5.5.34 `work_session_corrections`

Correcciones de sesiones hechas por un administrador, con el motivo.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `session_id` | uuid | No |  |
| `admin_id` | uuid | No |  |
| `old_started_at` | timestamptz | Sí |  |
| `old_ended_at` | timestamptz | Sí |  |
| `new_started_at` | timestamptz | Sí |  |
| `new_ended_at` | timestamptz | Sí |  |
| `reason` | text | No |  |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `work_session_corrections_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `work_session_corrections_admin_id_fkey` | `FOREIGN KEY (admin_id) REFERENCES users(id)` |
| Llave foránea | `work_session_corrections_session_id_fkey` | `FOREIGN KEY (session_id) REFERENCES work_sessions(id) ON DELETE CASCADE` |

### 5.5.35 `work_sessions`

Sesiones de trabajo (entrada y salida), con modalidad, retardo y minutos regulares y extra.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `user_id` | uuid | No |  |
| `location` | text | No |  |
| `started_at` | timestamptz | No | `now()` |
| `ended_at` | timestamptz | Sí |  |
| `regular_minutes` | integer | Sí | `0` |
| `overtime_minutes` | integer | Sí | `0` |
| `date` | date | No | `CURRENT_DATE` |
| `created_at` | timestamptz | No | `now()` |
| `scheduled_start_min` | integer | Sí |  |
| `late_minutes` | integer | No | `0` |
| `is_first_of_day` | boolean | No | `true` |
| `auto_closed` | boolean | No | `false` |
| `close_source` | text | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `work_sessions_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `work_sessions_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `work_sessions_location_check` | `CHECK ((location = ANY (ARRAY['oficina'::text, 'home_office'::text])))` |

### 5.5.36 `chat_messages`

Mensajes de chat con archivos, respuestas, reacciones, fijados y leídos.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `room_id` | uuid | No |  |
| `sender_id` | uuid | Sí |  |
| `content` | text | No |  |
| `created_at` | timestamptz | Sí | `now()` |
| `converted_to_task` | boolean | Sí | `false` |
| `task_id` | uuid | Sí |  |
| `file_url` | text | Sí |  |
| `file_type` | text | Sí |  |
| `file_name` | text | Sí |  |
| `deleted_for_everyone` | boolean | No | `false` |
| `deleted_for` | _uuid | No | `'{}'::uuid[]` |
| `reply_to_id` | uuid | Sí |  |
| `is_pinned` | boolean | No | `false` |
| `reactions` | jsonb | No | `'{}'::jsonb` |
| `read_by` | _uuid | No | `'{}'::uuid[]` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `chat_messages_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `chat_messages_reply_to_id_fkey` | `FOREIGN KEY (reply_to_id) REFERENCES chat_messages(id) ON DELETE SET NULL` |
| Llave foránea | `chat_messages_room_id_fkey` | `FOREIGN KEY (room_id) REFERENCES chat_rooms(id) ON DELETE CASCADE` |
| Llave foránea | `chat_messages_sender_id_fkey` | `FOREIGN KEY (sender_id) REFERENCES users(id) ON DELETE SET NULL` |
| Llave foránea | `chat_messages_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE SET NULL` |

### 5.5.37 `chat_rooms`

Salas de chat: la del equipo y una por cliente.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `type` | text | No |  |
| `client_id` | uuid | Sí |  |
| `name` | text | No |  |
| `created_at` | timestamptz | Sí | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `chat_rooms_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `chat_rooms_client_id_fkey` | `FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `chat_rooms_type_check` | `CHECK ((type = ANY (ARRAY['team'::text, 'client'::text])))` |

### 5.5.38 `device_tokens`

Tokens de Firebase de los teléfonos con la app, para notificaciones push. Solo se usa por funciones.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `token` | text | No |  |
| `user_id` | uuid | No |  |
| `platform` | text | No | `'android'::text` |
| `updated_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `device_tokens_pkey` | `PRIMARY KEY (token)` |
| Llave foránea | `device_tokens_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `device_tokens_platform_check` | `CHECK ((platform = 'android'::text))` |
| Verificación (CHECK) | `device_tokens_token_check` | `CHECK (((length(token) >= 20) AND (length(token) <= 4096)))` |

### 5.5.39 `direct_conversations`

Conversaciones directas entre dos personas.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `user_a` | uuid | No |  |
| `user_b` | uuid | No |  |
| `pair_key` | text | No |  |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `direct_conversations_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `direct_conversations_user_a_fkey` | `FOREIGN KEY (user_a) REFERENCES users(id) ON DELETE CASCADE` |
| Llave foránea | `direct_conversations_user_b_fkey` | `FOREIGN KEY (user_b) REFERENCES users(id) ON DELETE CASCADE` |
| Única | `direct_conversations_pair_key_key` | `UNIQUE (pair_key)` |
| Verificación (CHECK) | `direct_conversations_different_users` | `CHECK ((user_a <> user_b))` |

### 5.5.40 `direct_messages`

Mensajes de las conversaciones directas.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `conversation_id` | uuid | No |  |
| `sender_id` | uuid | No |  |
| `content` | text | No | `''::text` |
| `file_url` | text | Sí |  |
| `file_type` | text | Sí |  |
| `file_name` | text | Sí |  |
| `created_at` | timestamptz | No | `now()` |
| `deleted_for` | _uuid | No | `'{}'::uuid[]` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `direct_messages_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `direct_messages_conversation_id_fkey` | `FOREIGN KEY (conversation_id) REFERENCES direct_conversations(id) ON DELETE CASCADE` |
| Llave foránea | `direct_messages_sender_id_fkey` | `FOREIGN KEY (sender_id) REFERENCES users(id) ON DELETE CASCADE` |

### 5.5.41 `notifications`

Avisos de la campana web y de la app móvil (tipo, mensaje, tarea o cita relacionada, leído y enviado por push).

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `user_id` | uuid | Sí |  |
| `title` | text | No |  |
| `message` | text | No |  |
| `type` | text | Sí | `'task'::text` |
| `task_id` | uuid | Sí |  |
| `read` | boolean | Sí | `false` |
| `created_at` | timestamptz | Sí | `now()` |
| `appointment_id` | uuid | Sí |  |
| `pushed_at` | timestamptz | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `notifications_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `notifications_appointment_id_fkey` | `FOREIGN KEY (appointment_id) REFERENCES appointments(id) ON DELETE CASCADE` |
| Llave foránea | `notifications_task_id_fkey` | `FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE` |
| Llave foránea | `notifications_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |
| Verificación (CHECK) | `notifications_type_check` | `CHECK ((type = ANY ('{approval,attendance,change_request,client_delay,comment,deadline,review,task,task_request,appointment}'::text[])))` |

### 5.5.42 `appointment_agreements`

Acuerdos de una cita, cada uno marcado como visible o no para el cliente.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `appointment_id` | uuid | No |  |
| `body` | text | No |  |
| `visible_to_client` | boolean | No | `false` |
| `created_by` | uuid | Sí |  |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `appointment_agreements_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `appointment_agreements_appointment_id_fkey` | `FOREIGN KEY (appointment_id) REFERENCES appointments(id) ON DELETE CASCADE` |
| Llave foránea | `appointment_agreements_created_by_fkey` | `FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL` |
| Verificación (CHECK) | `appointment_agreements_body_check` | `CHECK (((length(TRIM(BOTH FROM body)) >= 2) AND (length(TRIM(BOTH FROM body)) <= 1000)))` |

### 5.5.43 `appointment_events`

Historial de cada cita (solicitada, programada, respuestas, recordatorios, cierre).

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | bigint | No |  |
| `appointment_id` | uuid | No |  |
| `actor_id` | uuid | Sí |  |
| `action` | text | No |  |
| `detail` | jsonb | No | `'{}'::jsonb` |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `appointment_events_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `appointment_events_actor_id_fkey` | `FOREIGN KEY (actor_id) REFERENCES users(id) ON DELETE SET NULL` |
| Llave foránea | `appointment_events_appointment_id_fkey` | `FOREIGN KEY (appointment_id) REFERENCES appointments(id) ON DELETE CASCADE` |

### 5.5.44 `appointment_kinds`

Catálogo de tipos de cita con su modalidad sugerida. Los administradores pueden agregar tipos.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `slug` | text | No |  |
| `label` | text | No |  |
| `default_modality` | text | No |  |
| `active` | boolean | No | `true` |
| `sort_order` | integer | No | `100` |
| `created_by` | uuid | Sí |  |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `appointment_kinds_pkey` | `PRIMARY KEY (slug)` |
| Llave foránea | `appointment_kinds_created_by_fkey` | `FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL` |
| Verificación (CHECK) | `appointment_kinds_default_modality_check` | `CHECK ((default_modality = ANY (ARRAY['presencial'::text, 'virtual'::text])))` |
| Verificación (CHECK) | `appointment_kinds_label_check` | `CHECK (((length(TRIM(BOTH FROM label)) >= 2) AND (length(TRIM(BOTH FROM label)) <= 40)))` |
| Verificación (CHECK) | `appointment_kinds_slug_check` | `CHECK ((slug ~ '^[a-z0-9_]{2,40}$'::text))` |

### 5.5.45 `appointment_participants`

Invitados de cada cita: obligatorio u opcional, respuesta, asistencia y franja ocupada (`slot`) para evitar empalmes.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `appointment_id` | uuid | No |  |
| `user_id` | uuid | No |  |
| `role_label` | text | Sí |  |
| `required` | boolean | No | `true` |
| `response` | text | No | `'pendiente'::text` |
| `response_note` | text | Sí |  |
| `proposed_slots` | jsonb | No | `'[]'::jsonb` |
| `responded_at` | timestamptz | Sí |  |
| `attendance` | text | Sí |  |
| `slot` | tstzrange | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `appointment_participants_pkey` | `PRIMARY KEY (appointment_id, user_id)` |
| Llave foránea | `appointment_participants_appointment_id_fkey` | `FOREIGN KEY (appointment_id) REFERENCES appointments(id) ON DELETE CASCADE` |
| Llave foránea | `appointment_participants_user_id_fkey` | `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE` |
| Exclusión | `appointment_participants_sin_empalmes` | `EXCLUDE USING gist (user_id WITH =, slot WITH &&) WHERE ((required AND (slot IS NOT NULL)))` |
| Verificación (CHECK) | `appointment_participants_attendance_check` | `CHECK ((attendance = ANY (ARRAY['presente'::text, 'ausente'::text])))` |
| Verificación (CHECK) | `appointment_participants_response_check` | `CHECK ((response = ANY (ARRAY['pendiente'::text, 'aceptada'::text, 'rechazada'::text, 'cambio_solicitado'::text])))` |
| Verificación (CHECK) | `appointment_participants_response_note_check` | `CHECK (((response_note IS NULL) OR (length(response_note) <= 1000)))` |

### 5.5.46 `appointment_places`

Estudios o lugares de la agencia, con dirección y coordenadas.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `name` | text | No |  |
| `address` | text | Sí |  |
| `latitude` | double | Sí |  |
| `longitude` | double | Sí |  |
| `active` | boolean | No | `true` |
| `created_by` | uuid | Sí |  |
| `created_at` | timestamptz | No | `now()` |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `appointment_places_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `appointment_places_created_by_fkey` | `FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL` |
| Verificación (CHECK) | `appointment_places_address_check` | `CHECK (((address IS NULL) OR (length(address) <= 300)))` |
| Verificación (CHECK) | `appointment_places_latitude_check` | `CHECK (((latitude IS NULL) OR ((latitude >= ('-90'::integer)::double precision) AND (latitude <= (90)::double precision))))` |
| Verificación (CHECK) | `appointment_places_longitude_check` | `CHECK (((longitude IS NULL) OR ((longitude >= ('-180'::integer)::double precision) AND (longitude <= (180)::double precision))))` |
| Verificación (CHECK) | `appointment_places_name_check` | `CHECK (((length(TRIM(BOTH FROM name)) >= 2) AND (length(TRIM(BOTH FROM name)) <= 80)))` |

### 5.5.47 `appointments`

Citas: tipo, estado, modalidad, lugar o enlace, horario, horarios propuestos, solicitante y coordinador.

| Campo | Tipo | Nulo | Valor por omisión |
|---|---|---|---|
| `id` | uuid | No | `gen_random_uuid()` |
| `project_id` | uuid | Sí |  |
| `kind` | text | No |  |
| `title` | text | No |  |
| `reason` | text | Sí |  |
| `status` | text | No | `'solicitada'::text` |
| `modality` | text | Sí |  |
| `location` | text | Sí |  |
| `meeting_url` | text | Sí |  |
| `starts_at` | timestamptz | Sí |  |
| `ends_at` | timestamptz | Sí |  |
| `timezone` | text | No | `'America/Mexico_City'::text` |
| `proposed_slots` | jsonb | No | `'[]'::jsonb` |
| `related_task_id` | uuid | Sí |  |
| `related_task_title` | text | Sí |  |
| `requested_by` | uuid | Sí |  |
| `coordinator_id` | uuid | Sí |  |
| `cancel_reason` | text | Sí |  |
| `close_note` | text | Sí |  |
| `started_at` | timestamptz | Sí |  |
| `closed_at` | timestamptz | Sí |  |
| `created_at` | timestamptz | No | `now()` |
| `updated_at` | timestamptz | No | `now()` |
| `location_kind` | text | Sí |  |
| `place_id` | uuid | Sí |  |
| `address` | text | Sí |  |
| `latitude` | double | Sí |  |
| `longitude` | double | Sí |  |

**Restricciones**

| Tipo | Nombre | Definición |
|---|---|---|
| Llave primaria | `appointments_pkey` | `PRIMARY KEY (id)` |
| Llave foránea | `appointments_coordinator_id_fkey` | `FOREIGN KEY (coordinator_id) REFERENCES users(id) ON DELETE SET NULL` |
| Llave foránea | `appointments_kind_fkey` | `FOREIGN KEY (kind) REFERENCES appointment_kinds(slug) ON UPDATE CASCADE` |
| Llave foránea | `appointments_place_id_fkey` | `FOREIGN KEY (place_id) REFERENCES appointment_places(id) ON DELETE SET NULL` |
| Llave foránea | `appointments_project_id_fkey` | `FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE SET NULL` |
| Llave foránea | `appointments_related_task_id_fkey` | `FOREIGN KEY (related_task_id) REFERENCES tasks(id) ON DELETE SET NULL` |
| Llave foránea | `appointments_requested_by_fkey` | `FOREIGN KEY (requested_by) REFERENCES users(id) ON DELETE SET NULL` |
| Verificación (CHECK) | `appointments_address_check` | `CHECK (((address IS NULL) OR (length(address) <= 300)))` |
| Verificación (CHECK) | `appointments_horario` | `CHECK (((ends_at IS NULL) OR (starts_at IS NULL) OR (ends_at > starts_at)))` |
| Verificación (CHECK) | `appointments_latitude_check` | `CHECK (((latitude IS NULL) OR ((latitude >= ('-90'::integer)::double precision) AND (latitude <= (90)::double precision))))` |
| Verificación (CHECK) | `appointments_location_check` | `CHECK (((location IS NULL) OR (length(location) <= 200)))` |
| Verificación (CHECK) | `appointments_location_kind_check` | `CHECK ((location_kind = ANY (ARRAY['estudio'::text, 'cliente'::text, 'otro'::text])))` |
| Verificación (CHECK) | `appointments_longitude_check` | `CHECK (((longitude IS NULL) OR ((longitude >= ('-180'::integer)::double precision) AND (longitude <= (180)::double precision))))` |
| Verificación (CHECK) | `appointments_meeting_url_check` | `CHECK (((meeting_url IS NULL) OR (meeting_url ~ '^https://'::text)))` |
| Verificación (CHECK) | `appointments_modality_check` | `CHECK ((modality = ANY (ARRAY['presencial'::text, 'virtual'::text])))` |
| Verificación (CHECK) | `appointments_programada_completa` | `CHECK (((status = ANY (ARRAY['solicitada'::text, 'cancelada'::text])) OR ((starts_at IS NOT NULL) AND (ends_at IS NOT NULL) AND (modality IS NOT NULL))))` |
| Verificación (CHECK) | `appointments_reason_check` | `CHECK (((reason IS NULL) OR (length(reason) <= 2000)))` |
| Verificación (CHECK) | `appointments_status_check` | `CHECK ((status = ANY (ARRAY['solicitada'::text, 'programada'::text, 'en_curso'::text, 'realizada'::text, 'incompleta'::text, 'cancelada'::text])))` |
| Verificación (CHECK) | `appointments_title_check` | `CHECK (((length(TRIM(BOTH FROM title)) >= 3) AND (length(TRIM(BOTH FROM title)) <= 140)))` |

## 5.6 Funciones de la base de datos (API)

`SECURITY DEFINER` = se ejecuta con permisos del dueño y valida internamente quién la llama. Columnas: **Auth** = puede ejecutarla un usuario con sesión; **Anon** = puede ejecutarla alguien sin sesión. Las funciones de trigger y las internas no se exponen.

| Función | Parámetros | Devuelve | Definer | Auth | Anon |
|---|---|---|---|---|---|
| `accept_client_invitation` | p_invitation_id uuid, p_full_name text | uuid | Sí | Sí | No |
| `accept_team_invitation` | p_invitation_id uuid, p_full_name text | uuid | Sí | Sí | No |
| `add_appointment_agreement` | p_appointment_id uuid, p_body text, p_visible_to_client boolean | uuid | Sí | Sí | No |
| `add_appointment_kind` | p_label text, p_default_modality text | text | Sí | Sí | No |
| `add_appointment_place` | p_name text, p_address text, p_lat double precision, p_lng double precision | uuid | Sí | Sí | No |
| `admin_archive_task` | p_task_id uuid, p_reason text | tasks | Sí | Sí | No |
| `admin_complete_team_task` | p_task_id uuid | void | Sí | Sí | No |
| `admin_corregir_sesion` | p_session_id uuid, p_started_at timestamp with time zone, p_ended_at timestamp with time zone, p_reason text | void | Sí | Sí | No |
| `admin_delete_project` | p_project_id uuid | uuid | Sí | Sí | No |
| `admin_delete_team_task` | p_task_id uuid | void | Sí | Sí | No |
| `admin_excuse_leader_delay` | p_delay_id uuid, p_reason text | void | Sí | Sí | No |
| `admin_remove_template_tasks` | p_template_id text, p_task_keys text[] | integer | Sí | Sí | No |
| `admin_reorder_member_tasks` | p_member_id uuid, p_task_ids uuid[] | integer | Sí | Sí | No |
| `admin_restore_task` | p_task_id uuid | tasks | Sí | Sí | No |
| `admin_save_task_plan` | p_items jsonb, p_remove uuid[] | integer | Sí | Sí | No |
| `admin_set_leader_plan` | p_task_id uuid, p_planned_start date, p_needed date | tasks | Sí | Sí | No |
| `admin_set_project_plan` | p_items jsonb | integer | Sí | Sí | No |
| `admin_update_task` | p_task_id uuid, p_changes jsonb | tasks | Sí | Sí | No |
| `app_active_admin_ids` | — | uuid[] | Sí | No | No |
| `app_appointment_kind_label` | p_kind text | text | Sí | Sí | No |
| `app_appointment_log` | p_appointment_id uuid, p_action text, p_detail jsonb | void | Sí | No | No |
| `app_appointment_notify` | p_appointment_id uuid, p_user_ids uuid[], p_title text, p_message text | void | Sí | No | No |
| `app_appointment_reminders` | — | void | Sí | No | No |
| `app_appointment_suggested_modality` | p_kind text | text | Sí | Sí | No |
| `app_appointment_when` | p_starts timestamp with time zone, p_tz text | text | No | Sí | Sí |
| `app_client_owns_project` | p_project_id uuid | boolean | Sí | Sí | No |
| `app_client_owns_task` | p_task_id uuid | boolean | Sí | Sí | No |
| `app_is_appointment_member` | p_appointment_id uuid | boolean | Sí | Sí | No |
| `app_is_staff` | — | boolean | Sí | Sí | No |
| `app_resolve_place` | p_kind text, p_place_id uuid, p_location text, p_address text, p_lat double precision, p_lng double precision, p_estricto boolean | TABLE(lugar text, direccion text, lat double precision, lng double precision, tipo text, place_id uuid) | Sí | No | No |
| `app_sync_appointment_slots` | p_appointment_id uuid | void | Sí | No | No |
| `app_validate_slots` | p_slots jsonb, p_min integer | jsonb | No | No | No |
| `apply_invitation_permissions` | invitation_id uuid, invitation_kind text | void | Sí | Sí | No |
| `asistencia_rango` | p_from date, p_to date, p_tolerancia integer | TABLE(user_id uuid, full_name text, department text, date date, laborable boolean, inicio_programado_min integer, fin_programado_min integer, entrada timestamp with time zone, ubicacion text, retardo_min integer, primera_actividad timestamp with time zone, primera_actividad_tipo text, ultima_actividad timestamp with time zone, salida timestamp with time zone, salida_automatica boolean, minutos_regulares integer, minutos_extra integer, estado text) | Sí | Sí | No |
| `asistencia_rango_interno` | p_date date, p_tolerancia integer | TABLE(user_id uuid, full_name text, department text, date date, laborable boolean, inicio_programado_min integer, fin_programado_min integer, entrada timestamp with time zone, ubicacion text, retardo_min integer, primera_actividad timestamp with time zone, primera_actividad_tipo text, ultima_actividad timestamp with time zone, salida timestamp with time zone, salida_automatica boolean, minutos_regulares integer, minutos_extra integer, estado text) | Sí | No | No |
| `can_edit_task_items` | p_task_id uuid | boolean | Sí | Sí | No |
| `cancel_appointment` | p_appointment_id uuid, p_reason text | void | Sí | Sí | No |
| `cerrar_sesiones_olvidadas` | — | void | Sí | No | No |
| `chain_create_step` | p_project_id uuid, p_chain_key text, p_step integer, p_round integer, p_prev_id uuid, p_actor uuid | uuid | Sí | No | No |
| `chain_enabled` | — | boolean | Sí | Sí | No |
| `chain_is_reviewer_of` | p_task_id uuid | boolean | Sí | Sí | No |
| `chain_reopen` | p_task_id uuid, p_actor uuid, p_reason text | void | Sí | No | No |
| `chain_resolve_owner` | p_owner_key text, p_project_id uuid, OUT user_id uuid, OUT fallback boolean | record | Sí | No | No |
| `chain_review_decision` | p_task_id uuid, p_decision text, p_comment text | tasks | Sí | Sí | No |
| `chain_set_recording_date` | p_task_id uuid, p_date date | tasks | Sí | Sí | No |
| `chain_start_project` | p_project_id uuid, p_chains text[] | integer | Sí | Sí | No |
| `check_appointment_conflicts` | p_starts timestamp with time zone, p_ends timestamp with time zone, p_user_ids uuid[], p_exclude_appointment uuid | TABLE(user_id uuid, full_name text, appointment_id uuid, appointment_title text, required_there boolean) | Sí | Sí | No |
| `client_complete_task` | p_task_id uuid | uuid | Sí | Sí | No |
| `close_appointment` | p_appointment_id uuid, p_status text, p_note text | void | Sí | Sí | No |
| `delete_appointment_agreement` | p_agreement_id uuid | void | Sí | Sí | No |
| `delete_task_permanently_internal` | p_task_id uuid, p_reason text | void | Sí | No | No |
| `finalize_project` | p_project_id uuid, p_mode text | integer | Sí | Sí | No |
| `get_client_invitation` | p_invitation_id uuid | TABLE(id uuid, email text, used boolean, client_name text, client_company text) | Sí | Sí | Sí |
| `get_team_invitation` | p_invitation_id uuid | TABLE(id uuid, email text, full_name text, role text, department text, used boolean, permissions jsonb) | Sí | Sí | Sí |
| `has_full_task_access` | — | boolean | Sí | Sí | No |
| `is_active_admin` | — | boolean | Sí | Sí | No |
| `is_active_super_admin` | — | boolean | Sí | Sí | No |
| `is_task_delivered` | p_status text, p_review_status text | boolean | No | Sí | Sí |
| `leader_business_days` | p_user uuid, p_from date, p_to date | integer | Sí | No | No |
| `mark_chat_room_read` | p_room_id uuid | void | Sí | Sí | No |
| `mark_client_delays` | — | void | Sí | No | No |
| `mi_sesion_hoy` | — | SETOF work_sessions | Sí | Sí | No |
| `notify_overdue_client_tasks` | — | void | Sí | No | No |
| `notify_upcoming_deadlines` | — | void | Sí | No | No |
| `purge_finalized_project_tasks` | p_project_id uuid | integer | Sí | No | No |
| `record_user_activity` | p_kind text | void | Sí | No | No |
| `register_device_token` | p_token text | void | Sí | Sí | No |
| `registrar_entrada` | p_location text | work_sessions | Sí | Sí | No |
| `registrar_salida` | — | integer | Sí | Sí | No |
| `request_appointment` | p_project_id uuid, p_kind text, p_reason text, p_slots jsonb, p_location_kind text, p_place_id uuid, p_address text, p_lat double precisi… | uuid | Sí | Sí | No |
| `resolve_change_request` | p_appointment_id uuid, p_user_id uuid, p_accept boolean, p_slot_index integer, p_note text | void | Sí | Sí | No |
| `respond_appointment` | p_appointment_id uuid, p_response text, p_note text, p_slots jsonb | void | Sí | Sí | No |
| `resumen_asistencia_diario` | — | void | Sí | No | No |
| `revisar_asistencia` | — | void | Sí | No | No |
| `schedule_appointment` | p_appointment_id uuid, p_project_id uuid, p_kind text, p_title text, p_starts timestamp with time zone, p_ends timestamp with time zone, … | uuid | Sí | Sí | No |
| `set_appointment_attendance` | p_appointment_id uuid, p_user_id uuid, p_attendance text | void | Sí | Sí | No |
| `set_appointment_kind_active` | p_slug text, p_active boolean | void | Sí | Sí | No |
| `set_appointment_place_active` | p_place_id uuid, p_active boolean | void | Sí | Sí | No |
| `set_user_access_role` | p_user_id uuid, p_role text | uuid | Sí | Sí | No |
| `start_appointment` | p_appointment_id uuid | void | Sí | Sí | No |
| `super_admin_delete_task` | p_task_id uuid | uuid | Sí | Sí | No |
| `super_admin_set_task_status` | p_task_id uuid, p_status text, p_completion_date date | tasks | Sí | Sí | No |
| `unregister_device_token` | p_token text | void | Sí | Sí | No |
| `update_task_review_status` | p_task_id uuid, p_review_status text | void | Sí | Sí | No |
| `work_day_schedule` | p_user uuid, p_date date | jsonb | Sí | No | No |
| `work_session_minutes` | p_user uuid, p_start timestamp with time zone, p_end timestamp with time zone, OUT regular_minutes integer, OUT overtime_minutes integer | record | Sí | No | No |

**Funciones de trigger**

`enforce_client_task_assignment`, `handle_new_user`, `protect_user_access_fields`, `trg_activity_generic`, `trg_activity_task_update`, `trg_aviso_entrada`, `trg_enviar_push`, `trg_projects_delete_super_admin_only`, `trg_projects_finalize_delete`, `trg_projects_reactivate_restore`, `trg_projects_status_super_admin_only`, `trg_tasks_chain`, `trg_tasks_clear_archived_via`, `trg_tasks_clear_sort_order_on_reassign`, `trg_tasks_completion_date`, `trg_tasks_fill_real_start`, `trg_tasks_leader_delays`

## 5.7 Triggers, tareas programadas, almacenamiento y tiempo real

**Triggers**

| Tabla | Trigger | Momento | Eventos | Función |
|---|---|---|---|---|
| `notifications` | `enviar_push` | AFTER | INSERT | `trg_enviar_push` |
| `projects` | `projects_delete_super_admin_only` | BEFORE | DELETE | `trg_projects_delete_super_admin_only` |
| `projects` | `projects_finalize_delete` | AFTER | UPDATE | `trg_projects_finalize_delete` |
| `projects` | `projects_reactivate_restore` | AFTER | UPDATE | `trg_projects_reactivate_restore` |
| `projects` | `projects_status_super_admin_only` | BEFORE | INSERT,UPDATE | `trg_projects_status_super_admin_only` |
| `task_comments` | `activity_task_comment` | AFTER | INSERT | `trg_activity_generic` |
| `task_subtasks` | `activity_subtask` | AFTER | INSERT,UPDATE | `trg_activity_generic` |
| `task_time_entries` | `activity_time_entry` | AFTER | INSERT | `trg_activity_generic` |
| `tasks` | `activity_task_update` | AFTER | UPDATE | `trg_activity_task_update` |
| `tasks` | `enforce_client_task_assignment` | BEFORE | INSERT,UPDATE | `enforce_client_task_assignment` |
| `tasks` | `tasks_chain_advance` | AFTER | UPDATE | `trg_tasks_chain` |
| `tasks` | `tasks_clear_archived_via` | BEFORE | UPDATE | `trg_tasks_clear_archived_via` |
| `tasks` | `tasks_clear_sort_order_on_reassign` | BEFORE | UPDATE | `trg_tasks_clear_sort_order_on_reassign` |
| `tasks` | `tasks_completion_date` | BEFORE | INSERT,UPDATE | `trg_tasks_completion_date` |
| `tasks` | `tasks_fill_real_start` | BEFORE | UPDATE | `trg_tasks_fill_real_start` |
| `tasks` | `tasks_leader_delays` | AFTER | UPDATE | `trg_tasks_leader_delays` |
| `users` | `on_auth_user_created` | BEFORE | INSERT | `handle_new_user` |
| `users` | `protect_user_access_fields` | BEFORE | INSERT,UPDATE | `protect_user_access_fields` |
| `work_sessions` | `aviso_entrada` | AFTER | INSERT | `trg_aviso_entrada` |

**Tareas programadas (pg_cron, hora UTC; CDMX = UTC−6)**

| Nombre | Horario (cron) | Comando |
|---|---|---|
| `alertas-vencimiento-diarias` | `0 15 * * *` | `INSERT INTO public.notifications (user_id, title, message, type)   SELECT     t.assigned_to,     '⚠️ Tarea vence mañana'` |
| `checador-cerrar-olvidadas` | `15 6 * * *` | `select public.cerrar_sesiones_olvidadas()` |
| `checador-resumen-diario` | `30 15 * * 1-6` | `select public.resumen_asistencia_diario()` |
| `checador-revisar-asistencia` | `*/15 13-20 * * 1-6` | `select public.revisar_asistencia()` |
| `citas-recordatorios` | `*/10 * * * *` | `select public.app_appointment_reminders()` |
| `mark-client-delays-daily` | `10 14 * * *` | `select public.mark_client_delays()` |
| `notify-overdue-client-tasks-daily` | `15 14 * * *` | `select public.notify_overdue_client_tasks()` |
| `notify-upcoming-deadlines-daily` | `0 14 * * *` | `select notify_upcoming_deadlines()` |

**Buckets de Storage**

| Bucket | Público | Límite | Tipos permitidos |
|---|---|---|---|
| `avatars` | Sí | Sin límite | Cualquiera |
| `chat-files` | Sí | Sin límite | Cualquiera |
| `client-covers` | Sí | Sin límite | Cualquiera |
| `appointment-evidence` | No (definido en `sql/2026-11-10_evidencia_citas.sql`) | 10 MB | JPEG, PNG, WEBP, HEIC, PDF | 

> ⚠️ `appointment-evidence` **no existe aún en producción** (hallazgo 1).

**Tablas publicadas en Realtime:** `appointment_participants`, `appointments`, `direct_messages`, `task_requests`, `work_sessions`. `notifications` se escucha desde la web por suscripción.

## 5.8 Scripts SQL del repositorio

Los scripts de `orbita-web/sql/` son **incrementales** y se corren a mano en el SQL Editor de Supabase. Los recientes (desde `2026-11-05`) incluyen pruebas internas: si alguna falla, se cancela toda la transacción. Los archivos `*_00_revision.sql` son consultas de diagnóstico previas.

| Script | Contenido principal | Estado en producción |
|---|---|---|
| `2026-11-05_cerrar_privacidad.sql` | RLS por rol, registro solo por invitación, funciones de ayuda | ✅ Aplicado (políticas y trigger presentes) |
| `2026-11-06_admin_borra_tareas.sql` | Borrado definitivo de tareas por administradores | ✅ Aplicado (verificado el 11 nov) |
| `2026-11-09_citas.sql` | Citas, participantes, acuerdos, historial y funciones | ✅ Aplicado |
| `2026-11-10_citas_v2_y_asistencia.sql` | Tipos de cita, cambio de horario, recordatorios y aviso de entrada | ✅ Aplicado |
| `2026-11-10_evidencia_citas.sql` | Evidencia en bucket privado | ⚠️ **No aplicado** |
| `2026-11-11_ubicacion_citas.sql` | Lugares de la agencia y ubicación de citas presenciales | ✅ Aplicado |
| `2026-11-11_push.sql` | Tokens de teléfonos y trigger de push | ✅ Aplicado |
| `2026-11-11_departamentos.sql` | Catálogo de departamentos | ✅ Aplicado |
| `2026-11-11_embudos.sql` | Borrado de embudos y sus tareas | ✅ Aplicado |

Los scripts anteriores a noviembre de 2026 provienen del proyecto de origen. Su estado individual no se verificó; la estructura se copió de la base original.

