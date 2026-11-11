# 7. Guía de integración — Reutilizar componentes de Órbita en otro proyecto

Esta guía identifica qué partes de Órbita se pueden llevar al proyecto cuatrimestral del equipo, qué archivos intervienen, qué necesitan y qué riesgos revisar. La valoración de esfuerzo es una **inferencia técnica**, no una medición.

## 7.1 Criterios para elegir
1. **Acoplamiento a Supabase:** casi toda la lógica vive en PostgreSQL (RLS y funciones). Reutilizarla es directo si el otro proyecto usa Supabase o PostgreSQL. Con otra base habría que reescribirla.
2. **Acoplamiento al dominio:** las tablas `users`, `projects`, `tasks` y `clients` están referenciadas en muchas funciones. Los módulos que dependen de ellas exigen esos modelos o adaptarlos.
3. **Licencia:** los repositorios no declaran licencia. Antes de copiar código, el equipo debe acordar su uso con la dueña del proyecto.

## 7.2 Componentes candidatos

| # | Componente | Valor para otro proyecto | Archivos principales | Dependencias | Esfuerzo (inferido) |
|---|---|---|---|---|---|
| 1 | **Módulo de citas con anti-empalmes** | Agenda con solicitud, programación, respuestas y bloqueo de choques garantizado en la base | `orbita-web/sql/2026-11-09_citas.sql`, `2026-11-10_citas_v2_y_asistencia.sql`, `2026-11-11_ubicacion_citas.sql`; `orbita-api/README.md` | PostgreSQL con `btree_gist` y `pg_cron`; tabla de usuarios con rol | Medio: adaptar `users`, `projects` y `tasks` y las funciones `is_active_admin` y `app_is_staff` |
| 2 | **App Android de citas** | Cliente Compose completo: agenda, solicitar, programar, detalle, en curso | `orbita-movil/app/src/main/java/.../ui/screens/*`, `data/Repo.kt`, `data/Models.kt` | supabase-kt 3.4.1, Compose BOM 2026.02.01, Kotlin 2.2 | Medio-alto: depende de las mismas funciones RPC |
| 3 | **Notificaciones Android (3 tipos + push)** | Patrón listo: básica, de toque, con botones, WorkManager de respaldo y FCM | `and/avisos/*`; `orbita-web/supabase/functions/enviar-push/index.ts`; `orbita-web/sql/2026-11-11_push.sql` | Firebase (Spark), pg_net, cuenta de servicio | Bajo-medio: es el componente más independiente |
| 4 | **Registro solo por invitación** | Altas controladas, sin registro libre | `orbita-web/sql/2026-11-05_cerrar_privacidad.sql` (trigger `handle_new_user`); `src/components/auth/*RegistrationPage.tsx`, `src/lib/inviteAccount.ts` | Supabase Auth | Bajo |
| 5 | **Correo transaccional sin dominio (Gmail SMTP)** | Enviar correos desde una Edge Function | `orbita-web/supabase/functions/send-user-invitation/index.ts` | `npm:nodemailer`, contraseña de aplicación de Gmail | Bajo. Riesgo de spam. |
| 6 | **Asistencia (checador)** | Entrada y salida con hora del servidor, retardo, horas extra y panel en vivo | `orbita-web/sql/2026-10-23_checador_servidor.sql`, `2026-11-10_citas_v2_y_asistencia.sql` (aviso de entrada); `src/contexts/WorkSessionContext.tsx`, `src/components/admin/AttendancePanel.tsx` | Horario semanal en `users.work_schedule`; pg_cron | Medio |
| 7 | **Permisos por rol y por persona** | Interruptores por sección | `RolePermissionsManager.tsx`; tablas `role_permissions` y `user_permissions` | — | Bajo-medio |
| 8 | **Evidencia en bucket privado** | Subir fotos o PDF con enlaces firmados y visibilidad por rol | `orbita-web/sql/2026-11-10_evidencia_citas.sql`; `and/ui/Evidencia.kt` | Supabase Storage, FileProvider | Bajo. ⚠️ No probado: el script no está aplicado en Órbita. |
| 9 | **Selector de ubicación con GPS** | Dirección o coordenadas con Geocoder y enlace a Maps | `and/ui/Ubicacion.kt`, `data/Models.kt` (`mapsUrl`) | Permisos de ubicación | Bajo |
| 10 | **Aviso de versión nueva (fuera de Play Store)** | Distribución de APK con aviso de actualización | `and/data/Actualizacion.kt`; `orbita-web/public/app.html` | GitHub Releases | Bajo |
| 11 | **Hoja de ruta por fases y salud** | Fases, semáforo y retrasos atribuibles | `src/lib/roadmapPhases.ts`, `src/components/roadmap/*` | Modelo de tareas con fase | Medio |
| 12 | **Reporte PDF desde la web** | Exportar un resumen a PDF | `src/lib/clientReportPdf.ts`, `ClientReportModal.tsx` | jsPDF, html2canvas | Bajo |
| 13 | **Documentación de API (OpenAPI)** | Base para documentar una API RPC | `orbita-api/openapi.yaml`, `orbita-api/README.md` | — | Bajo |

## 7.3 Pasos de integración recomendados (módulo de citas como ejemplo)
1. **Base:**
   1. Crear o adaptar `users` con `id` = `auth.users.id`, `role`, `active` e `is_super_admin`.
   2. Crear `is_active_admin()` y `app_is_staff()`.
   3. Correr en orden: `2026-11-09_citas.sql`, luego `2026-11-10_citas_v2_y_asistencia.sql` (solo las secciones de citas, si no se usa asistencia), luego `2026-11-11_ubicacion_citas.sql` y, por último, `2026-11-10_evidencia_citas.sql`.
   4. Revisar las referencias a `projects`, `tasks` y `notifications`, y quitarlas o adaptarlas.
2. **Avisos:** si no hay tabla `notifications` con columna `appointment_id`, adaptar `app_appointment_notify`.
3. **Android:**
   1. Copiar `data/` y `ui/`, y cambiar el `applicationId`.
   2. Configurar `local.properties`.
   3. Sin Firebase, quitar el plugin `google-services` y `avisos/PushService.kt`. La revisión cada 15 minutos sigue funcionando.
4. **Pruebas:** los scripts SQL traen pruebas internas. Agregar pruebas propias de la app, que hoy no existen.

## 7.4 Riesgos técnicos a revisar
| # | Riesgo | Detalle | Mitigación |
|---|---|---|---|
| R1 | Lógica en la base | Las reglas están en PL/pgSQL; cambiar de motor implica reescribirlas | Mantener PostgreSQL o Supabase |
| R2 | Esquema no reproducible solo con `sql/` | Falta la tabla `users` y el trigger de Auth originales | Partir de `pg_dump --schema-only` o crear un script base |
| R3 | Funciones `SECURITY DEFINER` | Un error de validación expone datos | Revisar cada `if not public.is_active_admin()`; mantener `revoke` para `anon` |
| R4 | Buckets públicos | `chat-files`, `avatars` y `client-covers` son públicos | Usar buckets privados con enlaces firmados, como en la evidencia |
| R5 | Planes gratuitos | Pausa de Supabase y uso no comercial de Vercel | Plan de pago o servidor propio para producción |
| R6 | Push sin verificar | La cadena trigger → pg_net → Edge Function → FCM no tiene prueba registrada | Probar con un aviso real antes de depender de ella |
| R7 | Versiones recientes | Compose BOM 2026.02, AGP 9.3 y Kotlin 2.2; algunas librerías nuevas exigen otro Kotlin (por eso `googleid` quedó en 1.1.1) | Alinear las versiones de Kotlin y AGP del otro proyecto |
| R8 | Sin pruebas Android | No hay pruebas unitarias ni instrumentadas propias | Agregar pruebas de `Repo` y de las pantallas clave |
| R9 | Dependencia de Google y Firebase | Clientes OAuth atados a la huella SHA-1 de la llave de firma | Registrar la SHA-1 de cada llave usada |
| R10 | Datos de prueba | La base de producción no tiene proyectos, tareas ni citas | Crear un entorno de pruebas con datos ficticios |
| R11 | Sin licencia declarada | El código es público pero sin licencia | Acordar el uso con la responsable antes de copiar |

## 7.5 Componentes que **no** se recomienda reutilizar tal cual
- **Tareas encadenadas (`workflow_*`):** sin datos y sin uso activo.
- **Tablas del Tablero semanal** (`team_goals`, `work_type_weights`): la interfaz se retiró.
- **Corrector ortográfico:** depende de una clave externa no configurada.
- **Paleta y estilos:** son específicos de la marca Órbita (`tailwind.config.js`, `DashboardV2.css`).
