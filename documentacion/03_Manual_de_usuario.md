# 3. Manual de usuario — Órbita

Este manual describe las funciones disponibles en el código actual. Los nombres de botones y secciones son los que aparecen en pantalla. Donde una función depende de una configuración pendiente, se indica con ⚠️.

| Plataforma | Dirección |
|---|---|
| Web | `https://orbita-web-ten.vercel.app` |
| App Android "Órbita Citas" | Descarga en `https://orbita-web-ten.vercel.app/app.html` (Android 11 o superior) |

---

## 3.1 Acceso (web)

### 3.1.1 Crear una cuenta por invitación
1. Abre el enlace que recibiste por correo o por mensaje. Termina en `?invite=…` (cliente) o `?team_invite=…` (equipo).
2. Verás el correo de la invitación, fijo. Escribe tu **nombre completo**, una **contraseña** de al menos 8 caracteres y su confirmación.
3. Pulsa **Crear mi cuenta**.
4. Si tu correo **ya tenía cuenta**, escribe la contraseña que ya usas: la invitación se acepta con esa cuenta.
5. Al terminar verás "¡Cuenta creada!". Pulsa el botón para ir al inicio de sesión o al portal.

> No es posible registrarse sin invitación, ni con correo ni con Google.

### 3.1.2 Iniciar sesión
1. Entra a la web, escribe **correo** y **contraseña** y pulsa **Iniciar sesión**. También puedes usar **Continuar con Google** con el mismo correo de tu invitación.
2. **¿Olvidaste tu contraseña?** Escribe tu correo y pide el enlace. ⚠️ El correo depende de la configuración SMTP del sistema; si no llega, pide al administrador que cambie la contraseña.

### 3.1.3 Registro de asistencia (solo equipo y administradores)
1. Al entrar en un día laboral aparece **Registrar entrada**. Elige **Oficina** o **Home office**.
2. El contador superior muestra el tiempo trabajado. Al terminar, registra tu **salida** desde ese mismo control.
3. Si olvidas registrar la salida, el sistema cierra la sesión automáticamente en la madrugada.

---

## 3.2 Web — Administrador

El menú lateral se agrupa en **Operación**, **Trabajo**, **Comunicación** y **Administración**. Las secciones visibles dependen de tus permisos; el superadministrador ve todas.

### 3.2.1 Panel de Control (Proyectos)
1. Revisa los indicadores: proyectos activos, tareas abiertas, terminadas en 7 días y personas activas. Cada tarjeta lleva a su sección.
2. **Resumen ejecutivo:** salud de la operación, tareas vencidas, entrega a tiempo y retrabajo.
3. **Requiere atención:** tareas por autorizar, fuera de fecha, con cambios del cliente o con retrasos del cliente. Pulsa una para abrirla.
4. **Proyectos:** filtra con Todos, Activos, Pausados o Finalizados, y abre una tarjeta para ver el detalle. **Ver reporte** genera el PDF.

### 3.2.2 Crear un proyecto
1. Pulsa **Nuevo proyecto**.
2. Elige un **Cliente existente** o crea uno nuevo. Si el cliente aún no tiene acceso, el sistema pide invitarlo primero.
3. Escribe el nombre y la descripción, y elige un **embudo** (o **Sin plantilla**).
4. En cada embudo, el menú **⋯** permite **Ver tareas**, **Editar** o **Eliminar** (con confirmación).
5. Revisa las tareas propuestas, asigna responsables y fechas, y confirma.

### 3.2.3 Tareas
1. **Nueva tarea:** título, descripción, responsable, prioridad, fecha compromiso y aprobadores.
2. **Tareas del equipo:** elige "Todo el equipo" o una persona, busca y filtra por estado.
   - Marca varias tareas para **reasignar**, **completar**, **fijar fecha** o **eliminar** en lote.
   - El lápiz edita la tarea, con registro en la bitácora.
3. **Mis tareas:** tus propias tareas, con indicadores y gráficas.
4. **Solicitudes:** peticiones de clientes. Pulsa **Aprobar** (crea la tarea) o **Rechazar**.

### 3.2.4 Hoja de ruta
1. Revisa las 4 etapas (Descubrimiento, Producción, Revisión y Entrega) con su avance.
2. Usa **Proyecto: Todos** para abrir la hoja de ruta de un proyecto, o busca y filtra por fase, semáforo, responsable o retraso.
3. En el detalle del proyecto, la sección **Próximas reuniones** muestra sus citas, con **Unirse** o **Ver en Google Maps**.

### 3.2.5 Asistencia
1. **Hoy en vivo:** contadores por estado y una tarjeta por persona con su semáforo. Se actualiza solo.
2. **Semana o rango:** elige fechas para ver totales por persona. **Exportar CSV** descarga el reporte.
3. Si alguien no registró su salida, usa **Corregir**: escribe el motivo y guarda.
4. Recibirás un aviso 🟢 o 🟠 cada vez que alguien registre su primera entrada del día.

### 3.2.6 Rendimiento
Muestra el **Scorecard del equipo**: métricas por integrante.

### 3.2.7 Entregables
1. Elige un cliente.
2. Busca y filtra por proyecto y estado (Pendientes, En revisión o Aprobados).
3. Usa el ojo para ver un archivo y el botón de copiar para su enlace. A la derecha, **Seguimiento del cliente** lista sus comentarios.

### 3.2.8 Usuarios y permisos
1. Elige la pestaña **Clientes**, **Equipo** o **Administradores**.
2. **Agregar persona:**
   1. Escribe nombre y correo. Para el equipo, elige el **departamento**; puedes crear uno nuevo.
   2. Ajusta los accesos iniciales y pulsa crear.
   3. Se envía el correo de bienvenida. Copia el enlace y compártelo también por mensaje.
3. **Invitación pendiente:** aparece en la lista hasta que la persona crea su cuenta, con **Copiar enlace**, **Reenviar correo** y **Eliminar invitación**.
4. **Persona con cuenta:** usa los interruptores para dar o quitar acceso a cada sección.
   - En el equipo puedes cambiar el **rol operativo** y su **horario semanal**.
   - **Asignar administrador** o **Regresar a Equipo** cambia su rol.
   - **Desactivar acceso** la bloquea sin borrarla.
   - **Eliminar persona** la quita del directorio; puedes reasignar antes sus tareas activas.

### 3.2.9 Otras secciones
- **Clientes:** listado de clientes, con su equipo asignado.
- **Embudos:** **Crear embudo**, **Editar** y **Eliminar** plantillas de tareas.
- **Calendario:** tareas del equipo por fecha.
- **Carga del equipo:** tareas activas por persona.
- **Chat Equipo, Chat Clientes y mensajes directos:** mensajes con archivos, menciones (@) y reacciones.
- **Licencia** (superadministrador): estado de la licencia de uso.

---

## 3.3 Web — Integrante del equipo
| Sección | Qué hacer |
|---|---|
| Mi día | Ver prioridades: tareas por revisar, vencidas, con cambios o sin fecha |
| Mis tareas | Cambiar estado, subir entregables (enlace o archivo), comentar y mandar a revisión |
| Mi Calendario | Ver tus tareas por fecha |
| Hoja de ruta | Ver las fases de tus proyectos |
| Chat del Equipo y Chat con Clientes | Comunicarte con el equipo o con tus clientes |
| Entregables | Consultar entregables de tus clientes |

> El equipo **no puede usar la app Android**, salvo que sea administrador. Si entra, verá "La app móvil es solo para clientes y administradores".

---

## 3.4 Web — Cliente (portal)
| Pestaña | Qué hacer |
|---|---|
| Mis pendientes | Tareas que requieren tu acción (por ejemplo, enviar material). Márcalas como completadas. |
| Todas las tareas / Por área | Ver el avance del proyecto |
| Calendario | Fechas de entrega |
| Hoja de ruta | Fases del proyecto, fecha de lanzamiento y próximas reuniones |
| Chat con el equipo | Mensajes con la agencia |
| Entregables | Revisar, aprobar o pedir cambios; calificar con estrellas |
| Solicitar tarea | Botón para pedir un trabajo nuevo; un administrador lo aprueba |

---

## 3.5 App Android "Órbita Citas"

### 3.5.1 Instalar y actualizar
1. En el teléfono, abre `https://orbita-web-ten.vercel.app/app.html` y pulsa **Descargar para Android**.
2. Abre `orbita-citas.apk`. Si Android lo pide, permite **instalar apps de este origen** y pulsa **Instalar**.
3. **Actualizaciones:** al abrir la app, si hay una versión nueva aparece **"Hay una versión nueva"**. Pulsa **Actualizar** e **Instalar**; conservas la sesión.
4. Al entrar por primera vez, **permite las notificaciones**.

### 3.5.2 Iniciar sesión
- Usa tu correo y contraseña, o **Continuar con Google**.
- Solo entran **clientes y administradores**.
- Si la licencia está suspendida o tu cuenta desactivada, verás el aviso y no podrás continuar.

### 3.5.3 Pestañas
| Pestaña | Contenido |
|---|---|
| Agenda | Semana de lunes a viernes con flechas, filtros **Tipo** y **Estado**, y tus citas por día. La campana abre los avisos. |
| Solicitudes | **Cliente:** lo que pediste y los cambios que propusiste. **Admin:** bandeja de solicitudes por programar y cambios pedidos. |
| Avisos | Notificaciones; pulsa una para abrir su cita. **Marcar todas como leídas**. |
| Perfil | Tus datos, **Abrir Órbita web** y **Cerrar sesión** |

### 3.5.4 Cliente: solicitar una cita
1. En **Agenda** o **Solicitudes**, pulsa **+**.
2. Elige el **tipo** (Reunión, Grabación, Entrevista, Accesos a cuentas u otro que haya creado la agencia) y el **proyecto**.
3. Si la cita es **presencial**, en **¿Dónde será?** elige:
   - **En el estudio:** la agencia elige cuál, o eliges uno de la lista.
   - **En mi domicilio:** escribe la dirección o pulsa **Usar mi ubicación actual**.
4. Escribe el **motivo** y agrega de **1 a 3 horarios** con **+ Agregar horario** (día, hora y duración).
5. Pulsa **Enviar solicitud**. La solicitud no reserva el horario: el coordinador lo confirma.

### 3.5.5 Responder una cita
1. Abre la cita desde Agenda, Avisos o una notificación.
2. Revisa la fecha, el lugar (con **Cómo llegar (Google Maps)**) o el enlace (**Unirse por Meet/Zoom**), y los participantes.
3. En **¿Asistirás?** pulsa:
   - **Aceptar.**
   - **Rechazar:** escribe el motivo.
   - **Proponer otro horario:** de 1 a 3 opciones y una nota.
4. Desde la notificación también puedes pulsar **Aceptar** sin abrir la app.

### 3.5.6 Administrador: programar una cita
1. Pulsa **+** (o **Programar** en una solicitud). Si viene de una solicitud, toca uno de los horarios propuestos para usarlo.
2. Elige el **tipo**; con **+ Otro tipo** creas uno nuevo. Luego escribe el **título** y elige el **proyecto**.
3. Define **Inicio** y **Fin** y la **modalidad**:
   - **Presencial:** elige **Estudio** (con **+ Agregar estudio o lugar de la agencia** das de alta el Bunker una sola vez), **Domicilio del cliente** u **Otro lugar**.
   - **Virtual:** pega el enlace `https://` de Meet o Zoom.
4. En **Invitados → + Agregar**, elige a cada persona y márcala como **Obligatorio** u **Opcional**.
   - Cada invitado muestra **Libre** u **Ocupado**.
   - Si un obligatorio está ocupado, no se puede publicar.
5. Opcional: elige una **tarea de referencia**. Pulsa **Publicar cita**.

### 3.5.7 Administrador: cambios de horario
1. En **Solicitudes → Piden cambios**, abre la cita.
2. Pulsa **Aceptar el nuevo horario** (si hay varios, elige uno), **Reprogramar** o **Rechazar** con un mensaje.

### 3.5.8 Administrador: cita en curso
1. En el detalle de una cita programada, pulsa **Iniciar** (desde una hora antes).
2. Registra **Presente** o **Ausente** por invitado.
3. Agrega **Acuerdos** y marca si son **Visibles para el cliente**.
4. **Evidencia:** pulsa **Cámara** o **Foto o PDF** y elige **Interna** o **Visible para el cliente**.
   ⚠️ Esta función no operará hasta que se aplique en la base el script de evidencia; ver el Documento de base de datos.
5. Pulsa **Cerrar realizada** o **Incompleta**; la segunda pide el motivo.

### 3.5.9 Notificaciones del teléfono
| Tipo | Ejemplo | Acción |
|---|---|---|
| Básica | Aviso de asistencia, aviso general | Tocar abre la app |
| De toque | Cualquier aviso de cita | Tocar abre el detalle de la cita |
| Con botones | Nueva cita o cita reprogramada | **Ver** o **Aceptar** |
| Con botones | Recordatorio | **Ver** y **Unirse** (virtual) o **Cómo llegar** (presencial) |

Llegan al instante por push o, como respaldo, en máximo 15 minutos.
