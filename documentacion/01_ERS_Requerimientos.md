# 1. Especificación de Requerimientos de Software (ERS) — Órbita

| Dato | Valor |
|---|---|
| Sistema | Órbita: plataforma de gestión de proyectos, tareas, citas, asistencia y evidencias |
| Componentes | Aplicación web (`orbita-web`), aplicación Android "Órbita Citas" (`orbita-movil`), API sobre Supabase (documentada en `orbita-api`), base de datos PostgreSQL |
| Versión analizada | `orbita-web` en el commit `75e2deb`, `orbita-movil` en `2edf5f1` (app 1.4) y `orbita-api` en `bd15ba0` (11 de noviembre de 2026) |
| Base | Revisión del código fuente y extracción del esquema de la base en modo solo lectura |

**Leyenda de estado**
- ✅ **Implementado (confirmado):** el código existe en el repositorio y, cuando aplica, el objeto existe en la base.
- 🟡 **Inferencia técnica:** se deduce del código, pero no hay evidencia de prueba.
- ⚠️ **Pendiente de verificar o incompleto.**

> Ningún requerimiento de este documento tiene un resultado de prueba formal registrado. La única evidencia automatizada es la suite web (`npm test`: 239 de 241 pruebas pasan; ver el Manual técnico). Las pruebas manuales que reportó la responsable del proyecto se indican como tales.

---

## 1.1 Objetivo
Centralizar la operación de una agencia de marketing digital en una sola plataforma:
- **Equipo interno:** gestiona proyectos, tareas, entregables, asistencia y comunicación.
- **Clientes:** consultan el avance, aprueban entregables y coordinan citas (grabaciones, entrevistas, reuniones y accesos a cuentas) desde la web y desde una app Android.

## 1.2 Alcance
**Incluido (implementado en el código):**
1. Gestión de clientes, proyectos, tareas, subtareas, comentarios y entregables.
2. Embudos: plantillas de tareas reutilizables por tipo de proyecto.
3. Hoja de ruta por fases, con semáforo de salud y retrasos atribuibles (cliente o agencia).
4. Registro de asistencia del equipo (entrada y salida), panel "Hoy en vivo" y alertas.
5. Citas: solicitud, programación, respuesta, cambio de horario, cita en curso, acuerdos, evidencia y recordatorios.
6. Notificaciones en la web (campana) y en Android (periódicas y push).
7. Control de acceso por rol, permisos individuales e invitaciones.
8. Chat de equipo, chat con clientes y mensajes directos.
9. Licencia de uso (activa o suspendida).

**Fuera de alcance o no implementado:**
- Pagos y facturación (solo existen campos informativos en `app_license`).
- App iOS.
- Publicación en Google Play: la app se distribuye como APK por GitHub Releases.
- Servidor de API independiente (Node/Express): la API es Supabase; ver la Documentación de la API.

## 1.3 Actores
| Actor | Descripción | Acceso |
|---|---|---|
| Superadministrador | Administrador con control total (permisos, licencia) | Web y Android |
| Administrador | Coordina proyectos, equipo y citas | Web y Android |
| Integrante del equipo (`team`) | Ejecuta tareas, registra asistencia | **Solo web**; la app Android le niega el acceso desde la versión 1.4 |
| Cliente (`client`) | Da seguimiento a sus proyectos, aprueba entregables y agenda citas | Web (portal) y Android |
| Sistema (tareas programadas) | `pg_cron` y triggers: recordatorios, alertas y avisos | Base de datos |

## 1.4 Requerimientos funcionales

### 1.4.1 Acceso y cuentas
| ID | Requerimiento | Estado | Evidencia en código |
|---|---|---|---|
| RF-01 | El sistema permitirá iniciar sesión con correo y contraseña. | ✅ | `src/components/auth/LoginPage.tsx`; Android `ui/screens/LoginScreen.kt` |
| RF-02 | El sistema permitirá iniciar sesión con Google en la web y en Android. | ✅ (reportado funcionando en Android por la responsable) | `GoogleSignInButton.tsx`; Android `ui/App.kt` (`entrarConGoogle`) |
| RF-03 | Solo podrá crear cuenta quien tenga una invitación pendiente. | ✅ | Trigger `on_auth_user_created` → `handle_new_user()`; `sql/2026-11-05_cerrar_privacidad.sql` |
| RF-04 | Un administrador podrá invitar clientes, integrantes del equipo y administradores; se enviará un correo de bienvenida y se podrá copiar el enlace. | ✅ (correo recibido, según la responsable) | `RolePermissionsManager.tsx`, `supabase/functions/send-user-invitation` |
| RF-05 | La página de invitación mostrará el correo fijo. Si el correo ya tiene cuenta, la invitación se aceptará con la contraseña existente. | ✅ | `ClientRegistrationPage.tsx`, `TeamRegistrationPage.tsx`, `src/lib/inviteAccount.ts` |
| RF-06 | Las invitaciones de equipo pendientes se podrán copiar, reenviar o eliminar. | ✅ | `RolePermissionsManager.tsx` |
| RF-07 | Los permisos se podrán ajustar por sección, por rol y por persona. | ✅ | Tablas `role_permissions`, `user_permissions` |
| RF-08 | Una cuenta desactivada o con licencia suspendida no podrá usar el sistema. | ✅ | `AuthContext.tsx`; Android `ui/App.kt` (`validarCuenta`) |
| RF-09 | La app Android solo admitirá clientes y administradores. | ✅ (desde 1.4) | Android `ui/App.kt` |
| RF-10 | Se podrá recuperar la contraseña por correo. | ⚠️ | Existe el flujo. La entrega depende de configurar el SMTP de Supabase Auth; no se verificó. |

### 1.4.2 Proyectos, tareas y entregables
| ID | Requerimiento | Estado | Evidencia |
|---|---|---|---|
| RF-11 | Se podrán crear, editar, pausar y finalizar proyectos por cliente, con modalidad (oficina o remota) y fecha de lanzamiento. | ✅ | `CreateProjectModal.tsx`, `ProjectDetail.tsx` |
| RF-12 | Al crear un proyecto se podrán generar sus tareas desde un embudo. | ✅ | `CreateProjectModal.tsx`, `src/data/projectTemplates.ts` |
| RF-13 | Los embudos se podrán crear, editar, consultar y eliminar, también desde "Nuevo proyecto". | ✅ | `FunnelTemplatesManager.tsx`, `src/lib/funnels.ts`, `sql/2026-11-11_embudos.sql` |
| RF-14 | Las tareas tendrán asignado, prioridad, estado, fecha compromiso, subtareas, comentarios y entregables. | ✅ | `CreateTaskModal.tsx`, `TaskDetailModal.tsx` |
| RF-15 | Las tareas pasarán por revisión interna (aprobadores) y aprobación del cliente. | ✅ | `TaskDetailModal.tsx`; campos `review_status`, `client_approval` |
| RF-16 | Un administrador podrá editar tareas con bitácora, cambiarlas en lote y borrarlas definitivamente. | ✅ | `TeamTasksManager.tsx`, `BulkTaskActionModal.tsx`, `task_audit_log` |
| RF-17 | Los clientes podrán solicitar tareas y un administrador las aprobará o rechazará. | ✅ | `RequestTaskModal.tsx`, `TaskRequestsPanel.tsx` |
| RF-18 | Los entregables se mostrarán por cliente con vista previa, estado y comentarios del cliente. | ✅ | `ClientDeliverables.tsx` |
| RF-19 | La hoja de ruta mostrará fases, avance y semáforo; los retrasos se atribuirán al cliente o a la agencia. | ✅ | `ClientRoadmapsManager.tsx`, `ClientRoadmapView.tsx`, `src/lib/roadmapPhases.ts` |
| RF-20 | Se podrá generar un reporte PDF del proyecto. | ✅ | `ClientReportModal.tsx`, `src/lib/clientReportPdf.ts` |

### 1.4.3 Asistencia
| ID | Requerimiento | Estado | Evidencia |
|---|---|---|---|
| RF-21 | El equipo registrará su entrada (oficina o home office) y su salida, con la hora del servidor. | ✅ | `WorkSessionContext.tsx`, funciones `registrar_entrada` y `registrar_salida` |
| RF-22 | Se calcularán el retardo y las horas regulares y extra según el horario semanal de cada persona. | ✅ | `asistencia_rango`, `work_session_minutes` |
| RF-23 | Los administradores recibirán un aviso en cada primera entrada del día: verde a tiempo, naranja con retardo. | ✅ (falta prueba con datos) | Trigger `aviso_entrada` |
| RF-24 | El panel "Hoy en vivo" mostrará el estado de cada persona y se actualizará al instante. | ✅ | `AttendancePanel.tsx` (Realtime en `work_sessions`) |
| RF-25 | Habrá alertas de "sin registro", resumen diario y cierre de sesiones olvidadas. | ✅ | `pg_cron`: `checador-*` |

### 1.4.4 Citas (web y Android)
| ID | Requerimiento | Estado | Evidencia |
|---|---|---|---|
| RF-26 | Un cliente o integrante solicitará una cita con tipo, proyecto, motivo y de 1 a 3 horarios. Si es presencial, indicará "estudio" o "domicilio" (dirección o ubicación actual). | ✅ | `request_appointment`; Android `SolicitarScreen.kt` |
| RF-27 | Un administrador programará o reprogramará: horario, modalidad, lugar o enlace https de Meet o Zoom, invitados obligatorios y opcionales, y tarea de referencia. | ✅ | `schedule_appointment`; `ProgramarScreen.kt` |
| RF-28 | El sistema impedirá que un invitado obligatorio tenga dos citas empalmadas; con opcionales solo avisará. | ✅ | Restricción de exclusión `appointment_participants_sin_empalmes`; `check_appointment_conflicts` |
| RF-29 | Los invitados aceptarán, rechazarán con motivo o propondrán otro horario; el administrador resolverá el cambio. | ✅ | `respond_appointment`, `resolve_change_request`; `DetalleScreen.kt`, `CambioScreen.kt` |
| RF-30 | Durante la cita se registrarán asistencia y acuerdos (visibles para el cliente o internos), y se cerrará como realizada o incompleta. | ✅ | `EnCursoScreen.kt`; `set_appointment_attendance`, `add_appointment_agreement`, `close_appointment` |
| RF-31 | Se podrá subir evidencia (foto o PDF) a almacenamiento privado. | ⚠️ **Código listo, base sin aplicar** | `ui/Evidencia.kt`; `sql/2026-11-10_evidencia_citas.sql` no está aplicado en producción |
| RF-32 | Se enviarán recordatorios 24 h y 1 h antes de cada cita. | ✅ (falta prueba con datos) | `pg_cron` `citas-recordatorios` |
| RF-33 | Los administradores podrán agregar tipos de cita y lugares de la agencia. | ✅ | `add_appointment_kind`, `add_appointment_place` |
| RF-34 | La web mostrará "Próximas reuniones" del proyecto, con enlace a Maps o a la videollamada. | ✅ | `src/components/roadmap/UpcomingMeetings.tsx` |

### 1.4.5 Notificaciones y comunicación
| ID | Requerimiento | Estado | Evidencia |
|---|---|---|---|
| RF-35 | La web mostrará avisos en una campana, en tiempo real. | ✅ | `NotificationBell.tsx` |
| RF-36 | Android mostrará notificaciones de 3 tipos: básica, de toque (abre la cita) y con botones (Ver/Aceptar, Unirse o Cómo llegar). | ✅ | `avisos/Notificaciones.kt`, `AccionReceiver.kt` |
| RF-37 | Android recibirá push al instante (Firebase) y tendrá una revisión de respaldo cada 15 minutos. | 🟡 Implementado; la entrega push no se confirmó de punta a punta | `PushService.kt`, `AvisosWorker.kt`, `supabase/functions/enviar-push` |
| RF-38 | Habrá chat de equipo, chat por cliente y mensajes directos con archivos, menciones y reacciones. | ✅ | `src/components/chat/*` |
| RF-39 | La app Android avisará cuando haya una versión nueva publicada. | ✅ | `data/Actualizacion.kt` |

## 1.5 Requerimientos no funcionales
| ID | Categoría | Requerimiento | Estado |
|---|---|---|---|
| RNF-01 | Seguridad | Todo acceso a datos exige sesión; las filas se filtran con RLS por rol y pertenencia. | ✅ |
| RNF-02 | Seguridad | Las escrituras sensibles (citas, asistencia, evidencia, tokens) solo pasan por funciones que validan rol y estado. | ✅ |
| RNF-03 | Seguridad | No se guardan secretos en los repositorios: `.env`, `local.properties`, `google-services.json` y la llave de firma están excluidos. | ✅ Confirmado en `.gitignore` |
| RNF-04 | Privacidad | Política de privacidad y condiciones publicadas (`public/privacidad.html`, `public/condiciones.html`). | ✅ (sin revisión legal) |
| RNF-05 | Usabilidad | Interfaz en español con zona horaria `America/Mexico_City`, tema claro y oscuro en la web, y paleta de marca. | ✅ |
| RNF-06 | Compatibilidad | Web en navegadores modernos; Android 11 o superior (minSdk 30, targetSdk 37). | ✅ |
| RNF-07 | Disponibilidad | Planes gratuitos: Vercel Hobby (uso no comercial) y Supabase Free (se pausa tras unos 7 días sin actividad). | ⚠️ Limitación |
| RNF-08 | Integridad | Los empalmes de citas se garantizan en la base, aunque dos administradores publiquen al mismo tiempo. | ✅ |
| RNF-09 | Mantenibilidad | Pruebas automatizadas de la web con el ejecutor nativo de Node (`tests/*.test.cjs`). | ✅ 239 de 241 pasan |
| RNF-10 | Mantenibilidad | Pruebas automatizadas de Android. | ⚠️ Solo existen las pruebas de ejemplo del proyecto base |

## 1.6 Reglas de negocio
| ID | Regla |
|---|---|
| RN-01 | Sin invitación pendiente no hay cuenta. El perfil nace al aceptar la invitación. |
| RN-02 | Solo los administradores activos coordinan citas: programar, iniciar, registrar asistencia, acuerdos y evidencia, y cerrar. |
| RN-03 | Grabación es presencial por omisión; entrevista, reunión y accesos a cuentas son virtuales. Los tipos nuevos definen su modalidad. |
| RN-04 | Una cita virtual exige un enlace `https://`. Una presencial exige un lugar: estudio de la agencia, domicilio del cliente con dirección o coordenadas, u otro lugar. |
| RN-05 | Un cliente solo ve sus propios proyectos y citas, y solo los acuerdos y la evidencia marcados como "visibles para el cliente". |
| RN-06 | Un retraso atribuido al cliente no penaliza las métricas de "a tiempo" de la agencia. |
| RN-07 | La tolerancia de retardo en asistencia es de 5 minutos. |
| RN-08 | Una cita se puede iniciar desde una hora antes de su horario. |
| RN-09 | Cada aviso se envía por push una sola vez y solo si tiene menos de 10 minutos de creado. |
| RN-10 | Borrar una tarea es definitivo y solo lo hacen administradores activos. |

## 1.7 Criterios de aceptación (propuestos para pruebas del equipo)
Ninguno de estos criterios tiene un resultado registrado. Se proponen para validarlos.

| ID | Criterio | Relacionado |
|---|---|---|
| CA-01 | Dado un correo sin invitación, al intentar registrarse con correo o con Google, entonces el sistema rechaza el alta con un mensaje de "sin invitación". | RF-03 |
| CA-02 | Dado un invitado que abre su enlace, al crear su contraseña, entonces entra a la vista de su rol y la invitación queda marcada como usada. | RF-04, RF-05 |
| CA-03 | Dada una cuenta de equipo no administradora, al iniciar sesión en Android, entonces ve el mensaje de acceso restringido y se cierra la sesión. | RF-09 |
| CA-04 | Dados dos administradores que programan citas empalmadas para el mismo invitado obligatorio, al publicar la segunda, entonces el sistema responde "CONFLICTO: … ya tiene otra cita en ese horario". | RF-28 |
| CA-05 | Dada una cita programada virtual, al pulsar "Unirse", entonces se abre el enlace de Meet o Zoom. | RF-27 |
| CA-06 | Dado un invitado que propone otro horario, cuando el administrador lo acepta, entonces la cita cambia a ese horario y los demás invitados vuelven a "pendiente". | RF-29 |
| CA-07 | Dada una cita a menos de 24 h, en máximo 10 minutos los participantes reciben "⏰ Recordatorio". | RF-32 |
| CA-08 | Dado un integrante que registra su primera entrada del día, los administradores reciben un aviso verde o naranja. | RF-23 |
| CA-09 | Dada la app cerrada y un aviso nuevo para el usuario, el teléfono muestra la notificación en segundos (push) o en máximo 15 minutos (respaldo). | RF-37 |
| CA-10 | Dado un cliente, al consultar acuerdos de una cita, solo ve los marcados como visibles para el cliente. | RN-05 |

## 1.8 Supuestos y restricciones
- La API es la capa REST de Supabase. No existe un servidor de aplicación propio, salvo dos Edge Functions y una función de Vercel.
- Los correos de bienvenida salen por SMTP de Gmail, sin dominio propio, y pueden caer en spam.
- El repositorio `orbita-web` es público (requisito de Vercel Hobby para repositorios de organización).
