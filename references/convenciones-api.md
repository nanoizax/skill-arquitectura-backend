# Convenciones de API (contrato HTTP común a todos los stacks)

Mismo contrato en NestJS, Laravel, CodeIgniter 4 y FastAPI. Marcas: **[fuente]** citado de documentación o estándar; **[criterio]** recomendación de esta skill; **[borrador]** especificación IETF aún no final; **[verificar]** sin comprobar.

## 1. Errores — RFC 9457 (Problem Details)

- RFC 9457 (jul-2023) obsoleta la RFC 7807. Media type `application/problem+json`. Miembros estándar: `type` (URI, por defecto `about:blank`), `status`, `title`, `detail`, `instance`; se permiten extensiones. **[fuente]**
- No filtres trazas ni detalles internos en `detail`. **[fuente]**
- Forma adoptada **[criterio]**:
```json
{
  "type": "https://api.ejemplo.com/problems/validation-error",
  "title": "La solicitud no es válida",
  "status": 422,
  "detail": "Hay 2 campos con errores.",
  "instance": "/v1/invoices",
  "code": "validation_error",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [{ "pointer": "/email", "code": "invalid_format", "detail": "Formato de correo inválido" }]
}
```
- `code` estable y legible por máquina: los clientes (Flutter, paneles) **no** parsean `title`. `trace_id` correlaciona con los logs y se devuelve también en `X-Request-Id`.
- Mapa de estados: 400 sintaxis · 401 sin autenticar · 403 sin permiso · **404 también para "existe pero es de otro tenant"** (no filtra existencia) · 409 conflicto · 422 validación semántica · 429 límite. 500 nunca con detalle interno.
- `title` y `detail` se traducen según `Accept-Language` y van en la variedad del mercado: **tuteo neutro** en español (nunca voseo: "ingresa", "tienes", "revisa"; con pronombre lleva tilde: "actívalo"), portugués de Brasil para Brasil. `code` no cambia. Pruébalo con un test que falle si reaparece voseo en los textos.
- Si un mensaje de error puede venir de un modelo de lenguaje o de un tercero, normalízalo a tu `code` antes de responder.
- **Comprobar**: prueba que provoca cada clase de error (400, 401, 403, 404, 409, 422, 429, 500) y valida el esquema `problem+json`; prueba que una excepción no controlada no devuelve traza.

## 2. Paginación

- Offset cuesta más cuanto más profundo; el cursor/keyset mantiene costo estable, resiste inserciones y borrados, pero exige orden estable y no permite "ir a la página 7". **[fuente]**
- **Por defecto cursor opaco** **[criterio]**: `?limit=50&cursor=…` → `{ "data": [...], "next_cursor": "…", "has_more": true }`. Orden determinista `(created_at, id)`; `limit` máximo 100 (rechaza o recorta, documentado).
- Offset solo en tablas de administración pequeñas donde la interfaz necesita números de página y total.
- Filtros y orden con **lista blanca** de campos. Índices compuestos que empiecen por `tenant_id`.
- Evita `COUNT(*)` exacto en tablas grandes.
- **Comprobar**: prueba que inserta filas entre dos páginas y verifica que no hay duplicados ni saltos; prueba con `limit` por encima del máximo.

## 3. Versionado

- Prefijo de ruta `/v1/`: simple, cacheable, visible en logs. **[criterio]** Cambios aditivos (campos opcionales nuevos) no crean versión. Stripe usa versiones por fecha fijadas a la cuenta; solo compensa con muchos integradores externos. **[fuente]**
- Las apps móviles ya instaladas obligan a mantener versiones viejas: define una ventana (p. ej. 12 meses, **[criterio]**) y **mídela** (versión de la app en logs o cabecera) antes de retirar.
- Deprecación con cabeceras `Deprecation` y `Sunset` **[verificar]** el estado de esos RFC al implementarlas.
- **Comprobar**: prueba de contrato (sección pruebas) sobre la versión publicada; el CI falla si un cambio rompe el esquema de `/v1`.

## 4. Idempotencia — **borrador**

- Borrador IETF `draft-ietf-httpapi-idempotency-key-header` (versión -07, oct-2025, **aún borrador**; caducaba abr-2026): cabecera `Idempotency-Key` (UUID recomendado) para tolerar reintentos en POST/PATCH. Comportamiento descrito: primera petición se procesa; repetición tras completar devuelve el resultado previo; repetición concurrente → 409; misma clave con payload distinto → 422; clave exigida y ausente → 400. **[fuente, borrador]** Revisa su estado al implementar.
- Implementación sugerida **[criterio]**: tabla `idempotency_keys(tenant_id, key, request_hash, status, response_code, response_body, created_at, expires_at)` con `UNIQUE (tenant_id, key)`; se inserta **dentro de la misma transacción** del caso de uso; TTL 24 h. Obligatoria en pagos, cobros, envío de mensajes y creación de facturas. La tabla se crea con una migración.
- Webhooks entrantes: deduplica por ID de evento del proveedor.
- **Comprobar**: prueba que envía dos veces la misma petición (secuencial y concurrente) y verifica una sola fila creada; misma clave con otro cuerpo → 422.

## 5. Validación

- **En el borde** (DTO/Request/Schema): tipos, formato, rangos, tamaño máximo, lista blanca de campos (rechaza o ignora desconocidos). **En el dominio**: invariantes de negocio, sin depender de que el borde las haya verificado.
- Un único formato de error (sección 1).
- `tenant_id`, `user_id` y `rol` **nunca** se aceptan del cuerpo: se derivan del contexto autenticado.
- Límites de tamaño de cuerpo y de subida en el proxy **y** en la app.
- Salida de un modelo de lenguaje o de un tercero: **nunca** confíes en enums ni longitudes; normaliza en código (mapa de sinónimos, lista permitida, recorte a la longitud de la columna, valor por defecto registrado). El `CHECK` de la base es la última defensa, no la primera. Al cambiar de modelo detrás de un alias, revalida el **formato** de cada consumidor, no solo que responda 200.
- **Comprobar**: pruebas de propiedades mínimas (campo de más, tipo erróneo, cadena gigante, `tenant_id` en el cuerpo ignorado).

## 6. Autenticación y autorización

- **Autenticación [criterio, práctica estándar; no verificada en fuente específica]**: web propia con cookie `HttpOnly; Secure; SameSite` + CSRF; móvil y terceros con token de acceso de vida corta (10-15 min) + refresh rotativo con detección de reutilización. JWT solo con algoritmo fijado en el servidor, `exp`, `aud`, `iss`; nunca datos sensibles en el payload.
- **Cabeceras de credencial de todos los clientes**: si la app usa `Authorization: Bearer`, `X-Guest-Token` u otra, el proxy y cualquier `map` de Nginx deben contemplarlas todas (un proxy que solo mira `Authorization` dejó sin servicio a los invitados de la app móvil). Prueba cada cliente real, también como invitado.
- **Autorización**: RBAC con roles por tenant (`membership(user, tenant, role)`) más ownership/ABAC cuando el rol no basta. El control se aplica **en cada acceso a un objeto**, no solo en el endpoint. BOLA (autorización a nivel de objeto rota) es el puesto 1 del OWASP API Security Top 10 2023. **[fuente]** Prevención: chequeo por objeto en cada función que accede a datos, IDs no secuenciales (UUID/ULID), consultas siempre filtradas por `tenant_id`, RLS como red de seguridad, pruebas automáticas de autorización.
- Política centralizada: Guards (Nest), Policies/Gates (Laravel), Filters (CI4), dependencias (FastAPI).
- Si el tenant también llega en la ruta, valida que coincide con el del token (evita BOLA horizontal).
- Claves de integración del cliente (pagos, SMTP, modelos de lenguaje): cifradas en reposo, solo se muestran los últimos 4 caracteres, con auditoría (ver `multi-tenancy-y-saas.md`).
- **Comprobar**: matriz de pruebas "usuario A intenta leer/modificar/borrar objeto de usuario B y de tenant B" para **cada recurso**; debe dar 404 o 403, nunca 200. Incluye pruebas de escritura: una matriz que solo hace GET oculta escrituras rotas.

## 7. Rate limiting

- Responde `429` con `Retry-After`. Las cabeceras `RateLimit-Limit/Remaining/Reset/Policy` son un **borrador** IETF (no RFC): publícalas solo como informativas. **[fuente, borrador]**
- Dos niveles **[criterio]**: (a) por IP en el proxy (Nginx) para abuso anónimo; (b) por tenant y usuario en la app (token bucket o ventana deslizante) con cuotas por plan. Más estricto en login, reseteo de contraseña y OTP.
- **IP real detrás de proxy**: confía en la cabecera que el proxy **sobrescribe** (`X-Real-IP $remote_addr`), no en `X-Forwarded-For` con `$proxy_add_x_forwarded_for`, que **anexa** y deja al cliente fijar el primer elemento. Nunca exentes por valor de IP que venga de una cabecera; exime solo si la IP de la conexión (`REMOTE_ADDR`) es loopback. Mantén siempre el cubo por usuario.
- **Trampas por stack** (casos reales):
  - NestJS Throttler: un `@Throttle({nombre})` con nombre no registrado en `forRoot` **no hace nada**, sin aviso; los limitadores registrados se **acumulan** y manda el más estricto, por lo que uno "de auth" aplicado a todas las rutas capó el sondeo de un chat con 429; el `APP_GUARD` no llega a los handlers de mensajes de WebSocket.
  - Laravel: dos `throttle:N,M` anónimos en la misma ruta comparten clave; usa `RateLimiter::for('nombre', …)`.
  - CI4: `Throttler` con caché de respaldo `dummy` o no escribible falla abierto en silencio.
- **Redis caído**: decide y documenta fail-open o fail-closed. Con BullMQ/ioredis la cola offline deja las peticiones **colgadas para siempre** (no rechaza). Acota cada encolado con timeout corto (p. ej. 2 s) y usa `commandTimeout` en el limitador; **no** apagues la cola offline del limitador (rompe la carga de su script y deja el límite mudo). Prueba las dos propiedades juntas: (A) con Redis arriba el límite corta, (B) con Redis caído la petición responde rápido.
- **Comprobar**: prueba de tráfico real: N peticiones seguidas (con el límite arriba: 429 presente; en endpoints sin límite: 0 códigos 429); una petición con `X-Forwarded-For` falsificada no se cree; test que lee la metadata de los decoradores y la compara con los nombres registrados, con un canario que falle si el formato cambia.

## 8. Observabilidad y logs

- Logs JSON estructurados con `trace_id` y `span_id` tomados del contexto de OpenTelemetry. **[fuente]**
- Campos mínimos **[criterio]**: `timestamp`, `level`, `service`, `env`, `trace_id`, `span_id`, `tenant_id`, `user_id` (id, no correo), `request_id`, `route`, `status`, `duration_ms`. Nunca contraseñas, tokens, tarjetas ni cuerpos completos.
- Métricas RED por ruta y tenant (rate, errors, duration). Salud separada: `/healthz` (proceso vivo) y `/readyz` (BD y dependencias críticas).
- Alertas: tasa de 5xx, latencia p95, saturación del pool de conexiones, disco del servidor.
- "Online" en PM2 **no** prueba que el servicio sirva: verifica con el health endpoint y los logs recientes.
- **Comprobar**: una petición de prueba genera una línea con los campos mínimos y el mismo `trace_id` que devuelve en la respuesta.

## 9. Configuración por entorno

- Configuración en variables de entorno; sin secretos en el repositorio ni en su historial; **validación del esquema al arrancar** y fallo inmediato si falta algo (`ConfigModule`+Zod/Joi, validación propia en Laravel, chequeo en `Config` de CI4, `pydantic-settings`).
- Entornos por variable, no por ramas de código. `.env` de producción con permisos restrictivos (600). Secretos en gestor con rotación. Una `DATABASE_URL` de aplicación distinta de la de migraciones.
- Un cliente (Flutter) nunca debe depender de que el binario de release lleve variables de compilación sin verificar: un release sin ellas arranca y muere.
- **PM2** **[fuente: notas internas]**: cambiar el `.env` y hacer `pm2 restart` no aplica el cambio (PM2 conserva el entorno del primer `start` y `dotenv` no pisa variables existentes); usa `pm2 delete` + `pm2 start ecosystem… ` + `pm2 save`. Un script apuntando a un shim de `node_modules/.bin` requiere `interpreter: 'none'`. Verifica el efecto **funcionalmente** (pegando al endpoint), no leyendo `/proc/<pid>/environ`.
- **Antes de reiniciar el proceso vivo**: respalda `dist`, arranca en seco en otro puerto y espera la línea de arranque exitoso; solo entonces reinicia. No existe entorno "test" separado si dos dominios apuntan al mismo proceso: confirma a qué backend llegas.
- Copia de las llaves y credenciales fuera del servidor de ejecución (copia en un disco o servicio distinto del servidor).
- **Comprobar**: arrancar con una variable obligatoria ausente debe fallar con mensaje claro.

## 10. CORS y cabeceras de seguridad

- CORS con lista blanca explícita de orígenes por entorno (nunca `*` con credenciales). Métodos y cabeceras mínimos. `Vary: Origin`. **[criterio]**
- Cabeceras de seguridad (CSP, HSTS, `X-Content-Type-Options`, `X-Frame-Options`/`frame-ancestors`, `Referrer-Policy`, `Permissions-Policy`): si **Nginx sirve el HTML del SPA**, `helmet` de Express **no** las pone en el documento, solo en la API. Verifica las dos rutas por separado: `curl -sI https://sitio/` y `curl -sI https://sitio/api/…`. Ponlas en el `location /` (no a nivel `server`, o el navegador aplica la intersección de dos CSP). `add_header` declarado en un bloque anidado descarta los heredados del padre: repite los necesarios con `always`.
- Ajusta la CSP a lo que la app hace (estilos en línea, `blob:`, service worker). `Permissions-Policy` puede romper micrófono o captura de pantalla en silencio si no los permites.
- Subida de archivos: tipo y tamaño validados, almacenamiento con prefijo por tenant, nunca servir desde la ruta de ejecución.
- **Comprobar**: `curl -sI` de la raíz y de la API tras cada `nginx reload`, y consola del navegador sin violaciones de CSP.

## 11. Rutas y proxy (Nginx)

- Un `location` específico para rutas `/api/...` que pertenecen a otro proceso (p. ej. un front Next.js) debe ir **antes** del `/api/` genérico, y el `proxy_pass` no puede apuntar fijo a un solo upstream si el cliente móvil usa las mismas URLs con `Authorization: Bearer`; usa un `map` con la clave de **todas** las cabeceras de credencial.
- El vhost vivo es el de `sites-enabled`; no recortes configuración de producción con `sed`/`awk`.
- **Comprobar**: tras cada reload, `curl` por las dos vías (con y sin credencial) y por el dominio público.
