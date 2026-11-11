# 2. Manual técnico — Órbita

> Este manual no incluye contraseñas, claves ni tokens. Los valores sensibles se indican como `<valor>` y se obtienen del responsable del proyecto.

## 2.1 Arquitectura del sistema

```
┌─────────────────────┐        ┌──────────────────────────────┐
│  Web (React + Vite) │        │  Android "Órbita Citas"      │
│  Vercel (Hobby)     │        │  Kotlin + Jetpack Compose    │
│  orbita-web         │        │  orbita-movil                │
└─────────┬───────────┘        └──────────────┬───────────────┘
          │ HTTPS (supabase-js)               │ HTTPS (supabase-kt)
          ▼                                   ▼
┌───────────────────────────────────────────────────────────────┐
│ Supabase (plan gratuito)                                      │
│  • Auth: correo y contraseña, Google (OAuth / ID token)       │
│  • PostgREST /rest/v1: tablas con RLS y funciones RPC         │
│  • Storage: avatars, chat-files, client-covers (+ evidencia*) │
│  • Realtime: notificaciones, chat, asistencia, citas          │
│  • Edge Functions: send-user-invitation, enviar-push          │
│  • PostgreSQL 17 + pg_cron + pg_net                           │
└───────┬───────────────────────────────┬───────────────────────┘
        │ SMTP 465                      │ FCM HTTP v1
        ▼                               ▼
   Gmail (correos de bienvenida)   Firebase Cloud Messaging → Android
* El bucket de evidencia está definido en sql/ pero no aplicado en producción.
```

**Decisión de diseño (confirmada en el código):** no hay un servidor de API propio. La lógica de negocio vive en la base, en funciones `SECURITY DEFINER` y políticas RLS, y ambos clientes llaman directamente a Supabase. El repositorio `orbita-api` solo contiene la documentación de esa API (`README.md` y `openapi.yaml`).

Además existe una función serverless en Vercel, `api/check-spelling.ts`, para el corrector ortográfico. Requiere `GEMINI_API_KEY`, que según las notas del proyecto **no está configurada**.

## 2.2 Tecnologías
| Capa | Tecnología | Versión (según manifiestos) |
|---|---|---|
| Web | React | ^19.2.6 |
| Web | TypeScript | (devDependency) |
| Web | Vite | (devDependency) |
| Web | Tailwind CSS + CSS propio (`src/styles/DashboardV2.css`) | — |
| Web | `@supabase/supabase-js` | ^2.105.4 |
| Web | `recharts` (gráficas), `jspdf` y `html2canvas` (PDF), `three` (fondo animado), `lucide-react` (iconos), `@imgly/background-removal` | ver `package.json` |
| Web | `react-router-dom` | ^7.15.0 (dependencia; la navegación principal es por estado de vista) |
| Android | Kotlin | 2.2.10 |
| Android | Android Gradle Plugin | 9.3.3 |
| Android | Jetpack Compose (BOM) + Material 3 | 2026.02.01 |
| Android | supabase-kt (postgrest, auth, storage) + Ktor OkHttp | 3.4.1 / 3.4.0 |
| Android | Credential Manager + Google ID | 1.5.0 / 1.1.1 |
| Android | WorkManager | 2.10.5 |
| Android | Firebase BOM (Messaging) + google-services | 34.4.0 / 4.4.4 |
| Backend | Supabase (PostgreSQL 17, PostgREST, Auth, Storage, Realtime, Edge Functions con Deno) | Administrado |
| Backend | Edge Functions: `npm:nodemailer` 6.9.16 (correo); Web Crypto (firma JWT para FCM) | — |
| Infraestructura | Vercel (web), GitHub Releases (APK), Firebase (push) | Planes gratuitos |

## 2.3 Repositorios y estructura
| Repositorio | Contenido | Rama principal |
|---|---|---|
| `orbita-sinertegia/orbita-web` | Web, scripts SQL, Edge Functions, función de Vercel y pruebas | `main` (cada push publica en Vercel) |
| `orbita-sinertegia/orbita-movil` | App Android | `main` |
| `orbita-sinertegia/orbita-api` | Documentación de la API (`README.md`, `openapi.yaml`) | `main` |

### 2.3.1 `orbita-web`
```
api/check-spelling.ts            Función serverless de Vercel (corrector con Gemini)
public/                          app.html (descarga Android), privacidad.html, condiciones.html, favicon
sql/                             52 scripts SQL incrementales (ver Documento de BD §5.8)
supabase/functions/
  send-user-invitation/index.ts  Correo de bienvenida (Gmail SMTP)
  enviar-push/index.ts           Envío de push por FCM
src/
  App.tsx, main.tsx              Arranque; elige vista por rol y rutas de invitación (?invite, ?team_invite)
  contexts/                      AuthContext (sesión, perfil, licencia), WorkSessionContext (entrada y salida)
  components/admin/              Panel del administrador (proyectos, tareas, asistencia, citas, usuarios…)
  components/team/               Panel del equipo (TeamDashboard, TaskDetailModal)
  components/client/             Portal del cliente
  components/auth/               Inicio de sesión, Google, registro por invitación, cuenta creada
  components/chat/               Chat de equipo, de clientes y mensajes directos
  components/roadmap/            Hoja de ruta y próximas reuniones
  components/shared/             Campana, perfil, modales comunes
  lib/                           Lógica: fechas, fases, estados, embudos, invitaciones, permisos de edición…
  data/projectTemplates.ts       Embudos de fábrica e inferencia de fase
  types/index.ts                 Tipos de dominio (User, Project, Task…)
tests/*.test.cjs                 Pruebas (node --test)
tailwind.config.js               Paleta de marca (blue-* = violeta, purple-* = lila)
```

### 2.3.2 `orbita-movil`
```
app/src/main/java/com/orbitasinertegia/orbita/
  MainActivity.kt                Arranque; abre una cita desde una notificación
  data/Repo.kt                   Cliente Supabase: lecturas, RPC, Storage, sesión
  data/Models.kt                 Modelos serializables (Appointment, Participant…)
  data/Fechas.kt                 Zona America/Mexico_City y formatos
  data/Actualizacion.kt          Revisa el último Release de GitHub
  ui/App.kt                      Sesión, validación de rol y licencia, navegación
  ui/screens/                    Login, Agenda, Solicitudes/Avisos/Perfil, Solicitar, Programar,
                                 Detalle, CambioScreen, EnCurso
  ui/Evidencia.kt, ui/Ubicacion.kt   Evidencia (cámara o archivo) y selector de lugar con GPS
  avisos/                        Notificaciones, WorkManager, receptor de botones y servicio de Firebase
  ui/theme/                      Paleta y tema
app/src/main/res/                Iconos, ic_notificacion, xml/fotos.xml (FileProvider)
```

## 2.4 Configuración del entorno

### 2.4.1 Requisitos
| Herramienta | Uso |
|---|---|
| Node.js 22 y npm | Compilar y probar la web |
| Android Studio (incluye su JDK, JBR) y Android SDK 37 | Compilar la app |
| Supabase CLI (`npx supabase`) | Publicar Edge Functions y secretos; consultar la BD |
| GitHub CLI (`gh`), opcional | Pull Requests y Releases |

### 2.4.2 Variables de configuración (sin valores)
| Dónde | Variable | Descripción |
|---|---|---|
| Web `.env` (no se versiona; ver `.env.example`) y Vercel | `VITE_SUPABASE_URL` | URL del proyecto Supabase |
| Web `.env` y Vercel | `VITE_SUPABASE_ANON_KEY` | Clave pública (anon) |
| Vercel (opcional) | `GEMINI_API_KEY` | Corrector ortográfico. ⚠️ No configurada |
| Android `local.properties` (no se versiona) | `supabase.url`, `supabase.anonKey` | Mismos valores que la web |
| Android `local.properties` | `firma.archivo` | Ruta al archivo de propiedades de la llave de publicación, fuera del repositorio |
| Android `app/google-services.json` (no se versiona) | — | Configuración de Firebase del paquete `com.orbitasinertegia.orbita` |
| Secretos de Edge Functions (Supabase) | `GMAIL_USER`, `GMAIL_APP_PASSWORD` | Envío de correos |
| Secretos de Edge Functions | `FCM_SERVICE_ACCOUNT` | JSON de la cuenta de servicio de Firebase |
| Automáticos de Supabase | `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` | Disponibles dentro de las Edge Functions |

## 2.5 Instalación y ejecución local

### 2.5.1 Web
```bash
git clone https://github.com/orbita-sinertegia/orbita-web.git
cd orbita-web
npm install
cp .env.example .env        # completar con la URL y la clave pública
npm run dev                 # http://localhost:5173
npm test                    # pruebas
npm run build               # compilación de producción (tsc -b + vite build)
npm run lint
```

### 2.5.2 Android
1. Clonar `orbita-movil` y abrirlo en Android Studio.
2. Crear `local.properties` con `sdk.dir`, `supabase.url` y `supabase.anonKey`.
3. Colocar `app/google-services.json`. Sin él, el plugin de Google Services falla al compilar.
4. Ejecutar con ▶ en un dispositivo con depuración USB, o en terminal:
```bash
./gradlew :app:assembleDebug
```
5. Versión firmada: `./gradlew :app:assembleRelease`. Requiere `firma.archivo` en `local.properties`; sin él, el APK sale sin firmar.

## 2.6 Conexión a la base de datos
- **Desde las apps:** solo por HTTPS a Supabase, con la clave pública y el JWT del usuario. No hay conexión directa a PostgreSQL desde los clientes.
- **Administración:** desde el SQL Editor del panel de Supabase o con `npx supabase db query --remote=<ref>`, que requiere `supabase login`.
- **Cambios de esquema:** se corren los scripts de `sql/` en orden de fecha. Los recientes son transaccionales y traen pruebas internas.
- **Entorno nuevo:** `sql/` no basta. Hay que restaurar primero un volcado de esquema de la base actual (`pg_dump --schema-only`). Al hacerlo:
  - Quitar `CREATE SCHEMA "public"` y `ALTER DEFAULT PRIVILEGES FOR ROLE "supabase_admin"`.
  - Revisar los permisos sobrantes que el proyecto nuevo otorga por omisión.

  Ambos ajustes vienen de las notas internas del proyecto.

## 2.7 Despliegue
| Componente | Cómo se despliega | Estado |
|---|---|---|
| Web | Push o merge a `main` de `orbita-web` → Vercel compila y publica automáticamente | ✅ Confirmado durante el proyecto |
| Edge Functions | `npx supabase functions deploy <nombre> --project-ref <ref> --use-api` (`enviar-push` lleva `--no-verify-jwt`) | ✅ Ambas publicadas el 11 nov 2026 |
| SQL | Correr el script en el SQL Editor | Ver Documento de BD §5.8 |
| Android | Subir `versionCode` y `versionName` → `assembleRelease` → crear Release en GitHub con el archivo **`orbita-citas.apk`** | ✅ Release `v1.4.0` publicado |
| Descarga | `https://orbita-web-ten.vercel.app/app.html` enlaza a `releases/latest/download/orbita-citas.apk` | ✅ |

**Firma de la app:** la llave de publicación vive fuera de los repositorios. Si se pierde, no se puede actualizar la app instalada. Debe respaldarse en un lugar privado.

## 2.8 Pruebas
| Componente | Tipo | Resultado conocido |
|---|---|---|
| Web | `npm test` (Node test runner, 13 archivos en `tests/`) | **239 de 241 pasan.** Fallan 2 sobre `TeamTasksManager` ("casillas, seleccionar todas…" y "persona y prioridad editables…"), registradas como fallas anteriores a los cambios recientes. |
| Web | `npm run build` | Compila sin errores en el último análisis |
| SQL | Pruebas dentro de cada script reciente (`PRUEBA FALLIDA` cancela la transacción) | Se ejecutan al aplicar el script |
| Android | `assembleDebug` y `assembleRelease` | Compilan. No hay pruebas automatizadas propias. |
| Manual | Inicio con Google en Android y registro de equipo con correo de bienvenida | Reportado como funcionando por la responsable. Sin acta formal. |

## 2.9 Limitaciones y errores conocidos
1. ⚠️ **Evidencia de citas sin base:** falta aplicar `sql/2026-11-10_evidencia_citas.sql`.
2. ⚠️ **Buckets públicos:** `avatars`, `chat-files` y `client-covers` son públicos.
3. ⚠️ **Planes gratuitos:** Vercel Hobby no admite uso comercial y Supabase Free se pausa tras unos 7 días sin actividad.
4. ⚠️ **Correo:** sale por Gmail sin dominio propio y puede llegar a spam. La recuperación de contraseña depende del SMTP de Supabase Auth, que no se ha verificado.
5. ⚠️ **Push:** implementado y publicado, pero la entrega de punta a punta no está confirmada.
6. ⚠️ **Android sin Play Store:** cada actualización requiere que la persona confirme la instalación.
7. ⚠️ **Repositorios públicos:** el código queda visible.
8. 2 pruebas web fallan (ver §2.8).
9. Funciones heredadas sin uso o con datos vacíos: tareas encadenadas (`workflow_*`), metas y puntos del antiguo Tablero semanal.
10. El historial de Git de `orbita-web` conserva archivos del proyecto de origen.
