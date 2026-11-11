# Documentación del proyecto Órbita

Documentación técnica y funcional generada a partir de la revisión del código y del esquema de la base el **11 de noviembre de 2026**.

## Repositorios analizados
| Repositorio | Componente | Commit analizado | Acceso |
|---|---|---|---|
| `orbita-sinertegia/orbita-web` | Aplicación web (React + Vite), scripts SQL, Edge Functions, función de Vercel | `75e2deb` | ✅ Lectura completa |
| `orbita-sinertegia/orbita-movil` | Aplicación Android "Órbita Citas" (Kotlin + Compose), versión 1.4 | `2edf5f1` | ✅ Lectura completa |
| `orbita-sinertegia/orbita-api` | Documentación de la API (`README.md`, `openapi.yaml`); no contiene código de servidor | `bd15ba0` | ✅ Lectura completa |
| Base de datos Supabase (producción) | PostgreSQL 17: esquema, funciones, triggers, cron y Storage | Extracción de solo lectura | ✅ Solo lectura |

No se analizó el proyecto de origen del que se derivó Órbita.

## Documentos
| # | Documento | Archivo |
|---|---|---|
| 1 | Especificación de requerimientos (ERS) | [01_ERS_Requerimientos.md](01_ERS_Requerimientos.md) |
| 2 | Manual técnico | [02_Manual_tecnico.md](02_Manual_tecnico.md) |
| 3 | Manual de usuario (web y Android) | [03_Manual_de_usuario.md](03_Manual_de_usuario.md) |
| 4 | Documentación de la API | [04_API.md](04_API.md) |
| 5 | Documento de base de datos | [05_Base_de_datos.md](05_Base_de_datos.md) |
| 6 | Inventario de funcionalidades | [06_Inventario_funcionalidades.md](06_Inventario_funcionalidades.md) |
| 7 | Guía de integración | [07_Guia_de_integracion.md](07_Guia_de_integracion.md) |

Versiones en Word en `docx/` y en PDF en `pdf/`. Se generaron a partir de estos Markdown; si cambian, hay que volver a generarlas.

## Leyenda usada en todos los documentos
| Marca | Significado |
|---|---|
| ✅ | **Confirmado:** existe en el código y, si aplica, en la base de producción |
| 🟡 | **Inferencia técnica, o implementado sin evidencia de prueba de punta a punta** |
| ⚠️ | **Pendiente de verificar, incompleto o no aplicado** |
| ⛔ | Retirado a propósito |

## Hallazgos principales
1. ⚠️ **La evidencia de citas no funciona en producción:** el script `sql/2026-11-10_evidencia_citas.sql` no está aplicado (no existen `appointment_files` ni el bucket `appointment-evidence`), aunque la app Android ya lo usa.
2. ⚠️ **No hay datos de prueba en producción:** 3 usuarios, 0 proyectos, 0 tareas y 0 citas. Las funciones de citas, recordatorios y push no tienen pruebas de punta a punta registradas.
3. ✅ **Seguridad:** acceso a datos solo con sesión (RLS), registro solo por invitación y escrituras sensibles por funciones validadas. ⚠️ Tres buckets de archivos son públicos.
4. ✅ **Pruebas web:** 239 de 241 pasan; 2 fallas registradas desde antes en `TeamTasksManager`. Android no tiene pruebas automatizadas propias.
5. ⚠️ **Archivo sin versionar:** `src/components/auth/CuentaCreada.tsx` y su uso en las páginas de registro están en la copia local y no están en GitHub.
6. ⚠️ **Limitaciones de infraestructura:** planes gratuitos (Vercel Hobby no comercial; Supabase se pausa por inactividad) y correos por Gmail sin dominio propio.

No se incluyen contraseñas, tokens, claves de API ni valores de variables de entorno.
