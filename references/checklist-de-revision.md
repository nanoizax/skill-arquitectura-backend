# Checklist de revisión de arquitectura

Para el líder o QA. Cada punto es verificable: el medio de comprobación va entre paréntesis. Marca **OK / NO / N/A** y deja evidencia (comando, salida, enlace). Un solo NO en las secciones A, D o E bloquea la entrega. Lo que no se pudo comprobar se marca **[verificar]**, nunca OK.

## A. Estructura y dependencias

- [ ] Existe una carpeta por módulo (bounded context) con `api`, `application`, `domain`, `infrastructure` y fachada pública (árbol del proyecto vs `estructura-por-stack.md`).
- [ ] No hay carpetas globales por tipo técnico (`controllers/`, `services/`, `models/` en la raíz) (`ls`).
- [ ] `domain` no importa framework, ORM ni HTTP (herramienta de CI del stack, y un import prohibido introducido a propósito hace fallar el CI).
- [ ] `application` no importa `infrastructure` ni `api` (ídem).
- [ ] Ningún módulo importa internos de otro; solo la fachada (ídem, regla de módulos independientes).
- [ ] Ninguna consulta ni migración hace JOIN o FK hacia tablas de otro módulo (revisión de migraciones y del catálogo de la BD).
- [ ] Un caso de uso = una transacción; correo, WhatsApp y otros efectos externos ocurren después del commit (lectura del caso de uso + prueba con proveedor fallido).
- [ ] Los controladores no contienen lógica ni consultas (lectura; umbral orientativo: si supera ~30 líneas por acción, revisar) **[criterio]**.
- [ ] Las excepciones a la regla están en un ADR con fecha de revisión (`docs/adr/`).
- [ ] No hay ceremonia sobrante: añadir un campo no toca más de 4-5 archivos en capas distintas.

## B. Decisión macro

- [ ] Es un monolito modular de un solo despliegue, o existe un ADR que justifica cada servicio extraído con al menos uno de los 4 criterios medibles.
- [ ] No se crean procesos, contenedores ni puertos por cliente (alta de prueba + `pm2 list`/`docker ps` antes y después).
- [ ] CQRS es ligero (casos de uso separados); no hay event sourcing ni buses sin ADR.

## C. Contrato de API

- [ ] Todos los errores salen como `application/problem+json` con `code` estable y `trace_id` (prueba por cada clase de error).
- [ ] Ninguna respuesta de error incluye traza, SQL ni rutas internas (prueba con excepción no controlada).
- [ ] Listados que crecen usan cursor opaco, orden determinista y `limit` máximo; filtros y orden con lista blanca (prueba con inserciones entre páginas).
- [ ] Rutas bajo `/v1/`; existe política de depreciación y medición de versiones de app en uso.
- [ ] POST críticos (pago, cobro, envío de mensaje, factura) son idempotentes; la tabla de claves existe y se creó con una migración (doble envío concurrente → una fila). `Idempotency-Key` se trata como **borrador**, no como estándar cerrado.
- [ ] Validación en el borde y en el dominio; campos desconocidos rechazados o ignorados (prueba).
- [ ] Endpoints que pueden devolver `null` entregan un cuerpo JSON real (inspección del cuerpo HTTP).
- [ ] Textos de cara al usuario en la variedad del mercado (tuteo neutro; pt-BR en Brasil) y prueba que detecta voseo (`grep` de formas de voseo + test).
- [ ] Salidas de servicios de lenguaje o de terceros normalizadas en código antes de escribir en BD (revisión + caso con valor inesperado).
- [ ] CORS con lista blanca por entorno, sin `*` con credenciales (`curl` con `Origin` ajeno).

## D. Seguridad

- [ ] Autorización por objeto en **cada** acceso; matriz recurso × actor × operación, incluidas escrituras, da 403/404 donde corresponde (prueba automatizada).
- [ ] `tenant_id`, `user_id` y rol nunca se toman del cuerpo ni de la URL sin contraste con el contexto autenticado (prueba).
- [ ] Autenticación: tokens de acceso de vida corta, refresh rotativo, algoritmo de JWT fijado en el servidor (configuración) **[criterio]**.
- [ ] Todas las cabeceras de credencial de los clientes reales (Bearer, token de invitado…) están contempladas en el proxy (`curl` con cada una contra el dominio público).
- [ ] Límite de tasa efectivo: los nombres de limitadores registrados coinciden con los usados; 429 con `Retry-After` en login, reseteo y OTP (tráfico real y prueba de metadata).
- [ ] La IP del cliente sale de la cabecera que el proxy sobrescribe, y una `X-Forwarded-For` falsificada no se cree (petición de prueba).
- [ ] Comportamiento definido y probado con Redis caído: responde rápido y el límite sigue cortando con Redis arriba (`docker stop` del contenedor).
- [ ] Cabeceras de seguridad presentes en el HTML **y** en la API (`curl -sI` a `/` y a `/api/…`).
- [ ] Sin secretos en el árbol ni en el historial (escáner de secretos; indica cuál se usó). `.env` de producción con permisos 600. Copia de llaves y credenciales fuera del servidor.
- [ ] Llaves de integración del cliente: cifradas, últimos 4 caracteres, auditadas, sin IDOR (prueba de integración).
- [ ] Dependencias auditadas (`npm audit`, `composer audit`, `pip-audit` u equivalente) con resultado adjunto.

## E. Multi-tenant

- [ ] Toda tabla de datos de cliente tiene `tenant_id NOT NULL`, primera columna de índices, y RLS con política (consulta al catálogo de la base).
- [ ] La aplicación se conecta con un rol sin privilegios: no dueño, sin `BYPASSRLS`, y distinto del rol de migraciones (`\du` y `DATABASE_URL` de runtime).
- [ ] El tenant se fija con `set_config(..., true)` dentro de cada transacción (lectura del código + prueba con conexiones reutilizadas).
- [ ] Sin tenant fijado, tablas y vistas devuelven 0 filas; las vistas usan `security_invoker` (prueba).
- [ ] Tenant A no lee ni escribe datos de B en ninguna tabla (prueba de aislamiento en CI con dos tenants de semilla).
- [ ] Caché, archivos, colas, jobs y limitadores son por tenant (revisión de claves y prefijos).
- [ ] Un único punto resuelve el tenant (ruta o `Host`); slugs reservados rechazados; ruta y token coinciden (pruebas).
- [ ] Marca, dominio e integraciones son datos del tenant, no variables de entorno.

## F. Observabilidad y operación

- [ ] Logs JSON con `trace_id`, `tenant_id`, `route`, `status`, `duration_ms`; el `trace_id` coincide con el de la respuesta (petición de prueba).
- [ ] Sin contraseñas, tokens ni cuerpos completos en logs (búsqueda en logs de prueba).
- [ ] `/healthz` y `/readyz` separados; `readyz` verifica la BD.
- [ ] Configuración validada al arrancar: falta una variable obligatoria → el proceso falla con mensaje claro (prueba).
- [ ] PM2: `interpreter: 'none'` si el script es un shim; los cambios de `.env` se aplican con `delete` + `start`, no `restart`; el efecto se verifica pegando al endpoint.
- [ ] Antes del reinicio del proceso vivo: respaldo de `dist` y arranque en seco en otro puerto con la línea de arranque exitoso.
- [ ] Tras cada `nginx reload`: `curl` por el dominio público, con y sin credencial; el vhost editado es el de `sites-enabled`.
- [ ] Hay alertas de 5xx, latencia p95, pool de conexiones y espacio en disco.

## G. Pruebas

- [ ] Integración contra BD real (mismo motor y versión que producción), sin mocks de repositorio para consultas.
- [ ] Contrato OpenAPI verificado en CI.
- [ ] Pruebas de arquitectura y de aislamiento de tenant en CI como puertas bloqueantes.
- [ ] Hay una prueba de arranque del artefacto exacto que irá a producción.
- [ ] Al menos una prueba replica la petición del cliente real (cabeceras incluidas), y las exclusiones de CSRF se verificaron con `curl`.
- [ ] Todo fallo previo de producción tiene su prueba de regresión.

## H. Datos

- [ ] Los cambios de esquema pasan por migraciones versionadas y revisadas, no por cambios manuales en la base.
- [ ] Las migraciones son *expand → migrate → contract*, compatibles con el código anterior y el posterior.
- [ ] Sin JOINs ni FKs entre módulos; referencias por ID.
- [ ] El diccionario de datos del proyecto está actualizado.

## I. Cierre

- [ ] Decisiones nuevas registradas como ADR en el repositorio.
- [ ] Cada afirmación de estado en el informe se verificó en esta sesión (no se arrastró de notas anteriores).
- [ ] Lo no comprobado aparece como **[verificar]** con responsable.

### Veredicto

| Resultado | Condición |
|---|---|
| **Cumple** | Todo OK o N/A justificado |
| **Cumple con observaciones** | NO solo en A/B/C/F/G/H/I con plan y fecha; ningún NO en D ni E |
| **No cumple** | Cualquier NO en D o E, o fallo de regla de dependencia sin ADR |
