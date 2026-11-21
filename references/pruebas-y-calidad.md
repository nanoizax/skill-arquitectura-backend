# Pruebas y calidad

Una prueba en verde solo prueba lo que ejecutó. Esta guía define la pirámide recomendada y, sobre todo, **los casos reales en que el verde ocultó un fallo**. Marcas: **[criterio]** recomendación de esta skill; **[caso real]** ocurrió en un proyecto; **[verificar]** sin comprobar.

## 1. Pirámide

| Nivel | Cantidad | Qué prueba | Notas |
|---|---|---|---|
| Unitarias de dominio y validación | Muchas, en ms | Reglas e invariantes, sin framework ni BD | `domain` se prueba sin arrancar nada |
| **Integración de caso de uso / slice** | **Las más valiosas** | Caso de uso contra **PostgreSQL/MariaDB real** (contenedor), con migraciones, RLS y autorización | No mockees el repositorio para probar consultas |
| Contrato / API | Por endpoint público | Respuesta contra el esquema OpenAPI; formato de error `problem+json` | Herramientas posibles: Schemathesis, Dredd u otras **[verificar]** |
| Arquitectura | En cada CI | Reglas de dependencia (`estructura-por-stack.md`) | Falla el build |
| Aislamiento de tenant | En cada CI | Sección 7 de `multi-tenancy-y-saas.md` | Obligatoria |
| E2E | Pocas | Flujos críticos: registro, pago, permisos entre tenants | Por el camino del cliente real |

Razón: las pruebas llenas de mocks prueban la implementación, no el comportamiento (crítica de Bogard **[fuente]**). Base de datos de pruebas con el **mismo motor y versión** que producción (MariaDB ≤ 10.9 y sus `TIMESTAMP`/`ENUM` se comportan distinto).

## 2. Regla madre

**Prueba por el mismo camino que usa el cliente real.** Si el cliente es una app Flutter, un panel o un proveedor de pagos, el test debe producir la misma petición: mismas cabeceras, mismo cuerpo, mismo proxy. Si no puedes en el test, haz una verificación con `curl` replicando la petición exacta y deja el comando en el PR.

## 3. Lo que un test en verde oculta (casos reales)

| # | Síntoma | Por qué el verde mentía | Cómo evitarlo |
|---|---|---|---|
| 1 | Laravel: una ruta que debe aceptar POST cross-site (retorno de pasarela, webhook) rechazaba con 419 en producción | PHPUnit desactiva la verificación CSRF automáticamente; el test sin token siempre pasa | Verifica exclusiones de CSRF con `curl` sin cookie ni token contra el entorno real; nunca con PHPUnit **[caso real]** |
| 2 | Laravel: tokens vencidos daban 500 en vez de 401 y el login fallaba en el dispositivo real | `getJson()`/`postJson()` añaden `Accept: application/json`; el cliente real (Dio) no lo enviaba. Además el test solo verificaba lo que el backend ya mandaba, no lo que el DTO del cliente necesitaba (faltaban campos de expiración) | Replica las cabeceras del cliente real en al menos un test; revisa el cliente y sus DTOs **[caso real]** |
| 3 | Laravel: tras subir de versión mayor, 262 tests en verde y el sitio público cayó al limpiar vistas | La caché de Blade compilada con la versión anterior seguía válida; una directiva nueva (`@context`) rompió plantillas con JSON-LD literal | `php artisan view:clear` antes de correr la suite tras actualizar el framework; escapa `@@` los `@` literales **[caso real]** |
| 4 | NestJS: `nest build` y `tsc` en verde, el proceso no arrancaba (502 y bucle de reinicio) | Un guard usado sin importar el módulo que lo provee; la inyección se resuelve en tiempo de ejecución | Arranque en seco del `dist` en otro puerto y espera de la línea de arranque antes de `pm2 restart`; prueba de arranque (`Test.createTestingModule` del módulo raíz) en CI **[caso real]** |
| 5 | NestJS: handler con `return null` devolvía cuerpo de 0 bytes y el cliente reventaba al parsear | Los tests de servicio comparan el valor devuelto, no el cuerpo HTTP; Nest/Express tratan `null` como "sin cuerpo" | En endpoints que legítimamente devuelven `null`, `@Res()` con `res.json(null)` solo en ese handler; prueba mirando el cuerpo real de la respuesta **[caso real]** |
| 6 | NestJS Throttler: 28 límites por endpoint no existían y uno global de 10/min daba 429 al sondeo del chat a los 50 s | Un `@Throttle` con nombre no registrado es invisible; los limitadores se acumulan | Test que lee la metadata de decoradores y la compara con los nombres registrados, más canario; tráfico real (N peticiones: 429 presente donde debe, ausente donde no) **[caso real]** |
| 7 | CI4: el comando de diagnóstico de IP daba siempre `0.0.0.0` | Escribir en `$_SERVER` no llega a la copia que cachea el servicio `superglobals` | `service('superglobals')->setServer(...)` **[caso real]** |
| 8 | Limitador de login eludible | Se probó en vivo sin cabecera hostil; nadie envió `X-Forwarded-For: 127.0.0.1` | Incluye una petición con `X-Forwarded-For` falsificada que no debe creerse. Dos auditores independientes por lectura de código lo hallaron **[caso real]** |
| 9 | Límite de tasa mudo: 6 de 6 peticiones pasaban contra un límite de 5 | Se "arregló" el cuelgue de Redis apagando la cola offline, lo que impidió cargar el script del limitador | Probar juntas (A) límite activo con Redis arriba y (B) respuesta rápida con Redis caído (`docker stop`), midiendo código HTTP y tiempo **[caso real]** |
| 10 | Cabeceras de seguridad "presentes" pero ausentes en el HTML | `curl` a `/api/health` mostraba helmet; el HTML lo sirve Nginx sin cabeceras | `curl -sI` a `/` y a `/api/…` por separado **[caso real]** |
| 11 | Pagos/retornos rotos sin que se notara | Se probó el endpoint, no el flujo que dispara el botón | Verifica por el camino del botón real, completo **[caso real, nota de pagos]** |
| 12 | App móvil sin agente ni subida de fotos desde semanas antes | Las pruebas usaban el flavor de desarrollo, cuyo vhost no tenía esos bloques; un `map` de Nginx ignoraba la cabecera del invitado | Prueba en el flavor de producción y como invitado además de logueado **[caso real]** |
| 13 | Enum en español del modelo de lenguaje violaba un `CHECK`, el análisis se descartaba durante meses | El `catch` tragaba el error; el ping al modelo daba "PONG" | Normaliza en código antes de escribir; al cambiar de modelo revalida el formato de salida de cada consumidor; un `catch` que traga un resultado importante deja estado visible **[caso real]** |
| 14 | Regla de tenant | Un endpoint probado solo con GET parecía seguro; la escritura estaba rota | La matriz de autorización cubre escritura, no solo lectura **[caso real]** |
| 15 | Pruebas de borrado | `deleteMany` + `create` destruía nietos por `ON DELETE CASCADE` sin error | Compara los `id`, no solo el contenido; revisa las FK y sus `ON DELETE` **[caso real]** |
| 16 | Dos dominios ("test" y producción) apuntaban al mismo backend | No había entorno de pruebas separado; un despliegue "a test" era el único backend | Confirma a qué proceso y base llegas antes de desplegar; usa arranque en seco **[caso real]** |
| 17 | Textos con voseo en un producto para Chile | Nadie leía los textos | Test que falla si reaparecen formas de voseo en los textos de la app (lista de verbos ampliable) **[caso real]** |

## 4. Pruebas por tipo de riesgo (plantillas)

- **Arranque**: levantar el proceso real con variables de producción de ejemplo y esperar el health endpoint. Falla si el módulo no resuelve dependencias.
- **Contrato con el cliente**: por cada DTO que consume el cliente móvil, una prueba que verifique que los campos **no nulos** del cliente vienen siempre en la respuesta.
- **Errores**: un caso por código de estado del mapa de `convenciones-api.md`, con `problem+json` válido.
- **Autorización por objeto**: matriz recurso × actor (dueño, otro usuario del mismo tenant, usuario de otro tenant, anónimo) × operación (leer, crear, actualizar, borrar).
- **Migraciones**: aplicar desde cero y sobre una copia del esquema real; el código anterior debe seguir funcionando tras la fase *expand* de una migración *expand/contract*.
- **Idempotencia**: doble envío secuencial y concurrente (ver `convenciones-api.md`, sección 4).
- **Degradación**: dependencia caída (Redis, proveedor de correo, servicio de lenguaje) → respuesta acotada en tiempo, no cuelgue.
- **Datos de ensayo**: bórralos por el mismo camino que los creó; limpiar a mano una sola tabla deja huérfanos que rompen pantallas.

## 5. Puertas de calidad en CI [criterio]

Orden sugerido (cada puerta bloquea la siguiente):
1. Formato y análisis estático.
2. **Reglas de dependencia** (herramienta del stack).
3. Unitarias.
4. Integración con BD real + aislamiento de tenant.
5. Contrato OpenAPI.
6. **Arranque del artefacto** (no del código fuente): `dist`, imagen o paquete exacto que irá a producción.
7. Escaneo de secretos y de dependencias (si no hay gitleaks/semgrep/trivy, di con qué se hizo).
8. Solo entonces despliegue, con arranque en seco y `curl` post-despliegue por las vías reales (dominio público, con y sin credencial).

Regla: si un fallo llegó a producción, **primero** se escribe la prueba que lo reproduce (y se ve fallar), después se corrige.

## 6. Qué no hacer

- Dar por bueno un cambio porque compila, pasa lint o el estado de PM2 dice "online".
- Mockear la base de datos para probar consultas, RLS o restricciones.
- Desactivar middleware o protecciones en pruebas sin una verificación real posterior.
- Probar solo con el usuario administrador, o solo con GET.
- Ejecutar pruebas destructivas contra el único backend existente sin respaldo ni arranque en seco.
