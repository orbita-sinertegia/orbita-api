# 6. Inventario de funcionalidades — Órbita

**Estados**
- ✅ **Implementada:** el código existe y, si aplica, la base tiene los objetos.
- 🟡 **Implementada sin verificar:** el código y la base existen, pero no hay evidencia de prueba de punta a punta.
- ⚠️ **Incompleta o no operativa:** falta configuración, script o datos.
- ⛔ **Retirada:** se eliminó a propósito.

Rutas: `web/` = `orbita-web`; `and/` = `orbita-movil/app/src/main/java/com/orbitasinertegia/orbita`.

## 6.1 Acceso y administración
| # | Funcionalidad | Descripción | Ubicación en el código | Estado | Dependencias |
|---|---|---|---|---|---|
| 1 | Inicio de sesión con correo | Correo y contraseña; validación de cuenta activa y de licencia | `web/src/components/auth/LoginPage.tsx`, `web/src/contexts/AuthContext.tsx`; `and/ui/screens/LoginScreen.kt`, `and/ui/App.kt` | ✅ | Supabase Auth |
| 2 | Inicio de sesión con Google (web) | Botón oficial de Google (ID token con nonce); redirección como respaldo | `web/src/components/auth/GoogleSignInButton.tsx` | ✅ | Proveedor Google en Supabase; cliente OAuth web |
| 3 | Inicio de sesión con Google (Android) | Credential Manager con selector completo de cuentas | `and/ui/App.kt` (`entrarConGoogle`) | ✅ (reportado funcionando) | Clientes OAuth Android con SHA-1 |
| 4 | Registro solo por invitación | Bloquea correos sin invitación pendiente | Trigger `on_auth_user_created`; `web/sql/2026-11-05_cerrar_privacidad.sql` | ✅ | — |
| 5 | Invitaciones de cliente, equipo y admin | Creación, correo de bienvenida, copiar enlace y pendientes visibles | `web/src/components/admin/RolePermissionsManager.tsx`, `web/src/lib/inviteEmail.ts` | ✅ | Edge Function `send-user-invitation`, secretos de Gmail |
| 6 | Aceptar invitación con cuenta existente | Entra con la contraseña actual y acepta | `web/src/lib/inviteAccount.ts`, `*RegistrationPage.tsx` | ✅ | — |
| 7 | Pantalla "¡Cuenta creada!" | Pantalla final del registro con fondo animado | `web/src/components/auth/CuentaCreada.tsx` | ⚠️ **Sin versionar**: archivo local no subido a Git | `FloatingLines.tsx` (three.js) |
| 8 | Permisos por rol y por persona | Interruptores por sección | `RolePermissionsManager.tsx`; tablas `role_permissions`, `user_permissions` | ✅ | — |
| 9 | Departamentos editables | Crear roles operativos | `RolePermissionsManager.tsx`; `web/sql/2026-11-11_departamentos.sql` | ✅ | — |
| 10 | Licencia | Activa o suspendida; bloquea el acceso | `web/src/components/admin/LicenseManager.tsx`; `app_license` | ✅ | — |
| 11 | Restricción de la app móvil | Solo clientes y administradores | `and/ui/App.kt` (`validarCuenta`) | ✅ (1.4) | — |
| 12 | Recuperar contraseña | Envío del enlace por correo | `LoginPage.tsx`, `ResetPasswordPage.tsx`; `and/data/Repo.kt` | ⚠️ | SMTP de Supabase Auth sin verificar |

## 6.2 Proyectos y tareas
| # | Funcionalidad | Descripción | Ubicación | Estado | Dependencias |
|---|---|---|---|---|---|
| 13 | Panel de Control | Indicadores, resumen ejecutivo, atención y proyectos | `web/src/components/admin/AdminDashboard.tsx`, `web/src/styles/DashboardV2.css` | ✅ | — |
| 14 | Proyectos | Crear, editar, pausar y finalizar; modalidad y fecha de lanzamiento | `CreateProjectModal.tsx`, `ProjectDetail.tsx`, `web/src/lib/projectStatus.ts` | ✅ | — |
| 15 | Embudos (plantillas) | Crear, editar y eliminar; menú ⋯ en Nuevo proyecto | `FunnelTemplatesManager.tsx`, `CreateFunnelTemplateModal.tsx`, `web/src/lib/funnels.ts`, `web/src/data/projectTemplates.ts` | ✅ | `admin_remove_template_tasks` |
| 16 | Tareas | Crear, asignar, prioridad, fechas, subtareas, comentarios y entregables | `CreateTaskModal.tsx`, `EditTaskModal.tsx`, `web/src/components/team/TaskDetailModal.tsx` | ✅ | Bucket `chat-files` (archivos) |
| 17 | Revisión y aprobación | Aprobadores internos y aprobación del cliente | `TaskDetailModal.tsx` | ✅ | — |
| 18 | Tareas del equipo | Por persona, filtros, edición en línea y acciones en lote | `TeamTasksManager.tsx`, `BulkTaskActionModal.tsx`, `web/src/lib/taskSelection.ts`, `inlineTaskEdit.ts` | 🟡 (2 pruebas automatizadas fallan) | — |
| 19 | Borrado definitivo de tareas | Solo administradores | `DeleteTaskModal.tsx`; `super_admin_delete_task` | ✅ | — |
| 20 | Solicitudes de clientes | El cliente pide; el admin aprueba o rechaza | `web/src/components/client/RequestTaskModal.tsx`, `TaskRequestsPanel.tsx` | ✅ | — |
| 21 | Hoja de ruta | Fases, semáforo, retrasos cliente o agencia, 4 etapas | `ClientRoadmapsManager.tsx`, `web/src/components/roadmap/*`, `web/src/lib/roadmapPhases.ts` | ✅ | — |
| 22 | Entregables por cliente | Vista previa, estado y comentarios del cliente | `ClientDeliverables.tsx` | ✅ | — |
| 23 | Reporte PDF | Reporte del proyecto | `ClientReportModal.tsx`, `web/src/lib/clientReportPdf.ts` | ✅ | jsPDF, html2canvas |
| 24 | Calendario y carga del equipo | Tareas por fecha y por persona | `TeamCalendar.tsx`, `TeamCapacity.tsx` | ✅ | — |
| 25 | Scorecard | Métricas por integrante | `TeamPerformance.tsx` | ✅ | — |
| 26 | Tablero semanal | Metas y puntos por tipo de trabajo | — | ⛔ Retirado (tablas `team_goals` y `work_type_weights` conservadas) | — |
| 27 | Tareas encadenadas | Crear la siguiente tarea al completar | Tablas `workflow_*` | ⚠️ Sin datos: inactiva | — |
| 28 | Corrector ortográfico | Revisa el texto de una URL con Gemini | `SpellingChecker.tsx`, `web/api/check-spelling.ts` | ⚠️ Falta `GEMINI_API_KEY` | Vercel, Gemini |
| 29 | Portal del cliente | Pendientes, avance, calendario, hoja de ruta, chat, entregables y calificaciones | `web/src/components/client/ClientPortal.tsx` | ✅ | — |

## 6.3 Asistencia
| # | Funcionalidad | Descripción | Ubicación | Estado | Dependencias |
|---|---|---|---|---|---|
| 30 | Entrada y salida | Oficina o home office; hora del servidor | `web/src/contexts/WorkSessionContext.tsx`, `CheckInModal.tsx`, `OvertimeModal.tsx` | ✅ | `registrar_entrada`, `registrar_salida` |
| 31 | Panel Hoy en vivo y rango | Semáforo por persona, tabla, CSV y correcciones | `AttendancePanel.tsx` | ✅ | Realtime `work_sessions`, `asistencia_rango` |
| 32 | Aviso de entrada | 🟢 a tiempo / 🟠 retardo para admins | Trigger `aviso_entrada` | 🟡 | — |
| 33 | Alertas automáticas | Sin registro, resumen diario, cierre de olvidadas | `pg_cron` `checador-*` | 🟡 | pg_cron |

## 6.4 Citas
| # | Funcionalidad | Descripción | Ubicación | Estado | Dependencias |
|---|---|---|---|---|---|
| 34 | Solicitar cita | Tipo, proyecto, motivo, 1 a 3 horarios y lugar si es presencial | `and/ui/screens/SolicitarScreen.kt`, `and/ui/Ubicacion.kt`; `request_appointment` | 🟡 | GPS y Geocoder (opcional) |
| 35 | Programar o reprogramar | Invitados obligatorios u opcionales, Libre/Ocupado, Meet o Zoom, estudio o domicilio | `and/ui/screens/ProgramarScreen.kt`; `schedule_appointment`, `check_appointment_conflicts` | 🟡 | — |
| 36 | Bloqueo de empalmes | Restricción de exclusión en la base | `appointment_participants_sin_empalmes` | ✅ (probado dentro del script SQL) | `btree_gist` |
| 37 | Responder y proponer horario | Aceptar, rechazar con motivo, cambio de horario | `DetalleScreen.kt`; `respond_appointment` | 🟡 | — |
| 38 | Resolver cambio de horario | Aceptar el horario propuesto, reprogramar o rechazar | `CambioScreen.kt`; `resolve_change_request` | 🟡 (prueba interna del script) | — |
| 39 | Cita en curso | Asistencia, acuerdos y cierre | `EnCursoScreen.kt`; RPC de cita | 🟡 | — |
| 40 | Evidencia | Cámara o archivo; interna o visible para el cliente; bucket privado | `and/ui/Evidencia.kt`; `web/sql/2026-11-10_evidencia_citas.sql` | ⚠️ **Script no aplicado en producción** | Storage, FileProvider |
| 41 | Tipos y lugares | Catálogos editables por admins | `SolicitarScreen.kt` (`SelectorTipo`), `Ubicacion.kt`; `add_appointment_kind`, `add_appointment_place` | 🟡 (0 lugares registrados) | — |
| 42 | Recordatorios | 24 h y 1 h antes | `pg_cron` `citas-recordatorios`; `app_appointment_reminders` | 🟡 | — |
| 43 | Próximas reuniones (web) | Tarjeta en la hoja de ruta del proyecto | `web/src/components/roadmap/UpcomingMeetings.tsx` | ✅ | — |

## 6.5 Notificaciones, comunicación y distribución
| # | Funcionalidad | Descripción | Ubicación | Estado | Dependencias |
|---|---|---|---|---|---|
| 44 | Campana web | Avisos en tiempo real que abren la tarea o la sección | `web/src/components/shared/NotificationBell.tsx` | ✅ | Realtime |
| 45 | Notificaciones Android (3 tipos) | Básica, de toque y con botones | `and/avisos/Notificaciones.kt`, `AccionReceiver.kt` | ✅ | Permiso POST_NOTIFICATIONS |
| 46 | Revisión periódica | Cada 15 minutos (respaldo) | `and/avisos/AvisosWorker.kt` | ✅ | WorkManager |
| 47 | Push (Firebase) | Envío inmediato al crear un aviso | `and/avisos/PushService.kt`; `web/supabase/functions/enviar-push`; `web/sql/2026-11-11_push.sql` | 🟡 No confirmado de punta a punta | Firebase, pg_net, `FCM_SERVICE_ACCOUNT` |
| 48 | Chat | Equipo, por cliente y directos; archivos, menciones y reacciones | `web/src/components/chat/*` | ✅ | Bucket `chat-files` (público) |
| 49 | Descarga de la app | Página con enlace al último Release | `web/public/app.html` | ✅ | GitHub Releases |
| 50 | Aviso de versión nueva | La app compara su versión con el último Release | `and/data/Actualizacion.kt` | ✅ | API pública de GitHub |
| 51 | Correo de bienvenida | Gmail SMTP | `web/supabase/functions/send-user-invitation/index.ts` | ✅ (reportado recibido; puede ir a spam) | Secretos de Gmail |
| 52 | Política de privacidad y condiciones | Páginas estáticas | `web/public/privacidad.html`, `condiciones.html` | ✅ (sin revisión legal) | — |

## 6.6 Resumen
| Estado | Cantidad |
|---|---|
| ✅ Implementada | 35 |
| 🟡 Implementada sin verificar | 11 |
| ⚠️ Incompleta o no operativa | 5 |
| ⛔ Retirada | 1 |
