# Multi-tenancy y SaaS

Cómo construir un backend que atiende a varios clientes (tenants) sin que uno vea los datos de otro. Marcas: **[fuente]** documentación oficial o citada; **[criterio]** recomendación de esta skill; **[verificar]** sin comprobar. Lo de esquema, roles y RLS en SQL conviene tenerlo en una guía de base de datos aparte: enlázala, no la copies.

## 1. Decisiones de producto que condicionan el código (tómalas antes de escribir)

1. **Una sola instancia por producto** **[criterio]**. Todos los clientes de un SaaS usan el mismo proceso. Crecer = crecer en la base de datos, no levantar procesos o contenedores por cliente. Provisionar un cliente = crear filas (y, como máximo, una base o esquema), no infraestructura.
2. **Identidad central única** **[criterio]**. Una cuenta global que pertenece a uno o varios tenants mediante **membresías**; evita registros y usuarios duplicados por producto.
3. **Cómo se resuelve el tenant** **[criterio]**. Por ruta (`/<slug>/…`) o por `Host` (tabla `dominio → empresa`); nunca por parámetros del cuerpo. Reserva slugs para rutas fijas del producto (`registro`, `demo`, `api`, `admin`…).
4. **Qué es por tenant** **[criterio]**. Marca, logo, dominio, integraciones, llaves y cuotas son datos del tenant, no configuración de despliegue.
5. **Integraciones con llaves del cliente (BYOK)** **[criterio]**. Si el cliente trae sus propias llaves (pagos, mapas, correo, servicios de lenguaje), la configuración es **por tenant** (ver sección 5).

## 2. Consecuencias de arquitectura (reglas verificables)

| # | Regla | Cómo se comprueba |
|---|---|---|
| T1 | Ningún alta crea procesos, contenedores ni puertos por cliente | Revisar el código de aprovisionamiento: solo escribe en BD; prueba: el alta de un tenant no cambia la lista de procesos ni de contenedores |
| T2 | La instancia resuelve el tenant **antes** de la lógica de negocio, en **un solo lugar** (middleware/filtro/dependencia) | Un único archivo resuelve el tenant; `grep` de lecturas de `slug`/`Host` fuera de él no da resultados |
| T3 | Resolución por ruta (`/<slug>/…`) o por `Host`; nunca por parámetros del cuerpo | Prueba: petición con `tenant_id` en el cuerpo distinto al del contexto → se ignora o se rechaza |
| T4 | Slugs reservados: rutas fijas del producto y cualquier nombre que choque con rutas | Prueba de alta con cada nombre reservado → rechazada |
| T5 | Si el tenant llega en ruta **y** en el token, deben coincidir | Prueba: token del tenant A en ruta del tenant B → 403/404 |
| T6 | Marca, logo, dominio e integraciones son datos del tenant | No hay variables de entorno por cliente |
| T7 | Si hay entorno demo: se reinicia de forma programada y las operaciones destructivas están bloqueadas | Prueba: el reinicio restaura el estado; en la demo las operaciones sensibles devuelven error |
| T8 | La cuenta central es identidad global con **membresías** a tenants (`membership(user, tenant, role)`) | Esquema: no hay usuarios duplicados por producto |

## 3. Modelos de aislamiento de datos

| Modelo | Aislamiento | Escala | Cuándo |
|---|---|---|---|
| **Fila + RLS** (`tenant_id`) | Lógico, fuerte con RLS | Muy alta | **Por defecto** |
| Esquema por tenant | Medio | De ~100 a pocos miles; luego el catálogo crece y las migraciones se multiplican | Pocos tenants grandes |
| Base por tenant | Máximo | Limitada por operación | Contratos enterprise o residencia de datos |

Fuente: modelos *pool/bridge/silo* descritos por AWS y guías de comunidad. **[fuente]**

**Veredicto [criterio]**: fila + RLS, `tenant_id NOT NULL` en **toda** tabla de datos de cliente, primera columna de PK/índices/UNIQUE, y RLS activada. Mantén abierta la puerta a mover un tenant grande a su propia base: sin JOINs entre módulos y sin estado global.

Una variante válida es **una base por tenant dentro de una única instancia de proceso** (resolver el tenant antes del framework y mantener la configuración solo en variables de entorno de lectura). Es el modelo *silo*; úsalo solo con razón (contratos o residencia de datos).

## 4. Reglas de código y base de datos

- **Rol de la aplicación sin privilegios**: no dueño de las tablas, sin `BYPASSRLS` ni superusuario. Las migraciones usan otro rol. Razón oficial: dueños y `BYPASSRLS` se saltan la RLS (el dueño puede someterse con `FORCE ROW LEVEL SECURITY`); `TRUNCATE` y las comprobaciones de integridad referencial también la evitan. **[fuente: documentación de PostgreSQL]**
- **Fijar el tenant por transacción**: `set_config('app.tenant_id', $1, true)` (alcance de **transacción**) al inicio de cada transacción. Un valor a nivel de sesión puede filtrarse a la siguiente petición que reutilice la conexión; con PgBouncer en modo transaction funciona, en modo session no. **[fuente]**
- **Política**: `USING` y `WITH CHECK` sobre `tenant_id = current_setting('app.tenant_id', true)`. Con tenant sin fijar debe devolver **0 filas**, no error ni todas.
- **Vistas**: consultan con los permisos y la RLS de su **dueño**; usa `security_invoker = true` (PG 15+) y cambia el dueño de tablas, secuencias y vistas. Una vista con dueño `BYPASSRLS` devuelve datos de todas las empresas.
- **Roles de nombre fijo en migraciones**: parametriza el rol en los `GRANT` para que las migraciones no dependan de un nombre de rol concreto.
- **Implementación por stack**: NestJS + Prisma → extensión de cliente `$extends` que envuelve cada consulta en una transacción con `set_config` (mide el costo de la transacción extra) y `nestjs-cls`; Laravel → global scope + RLS, serializar el tenant en jobs; CI4 → Filter + `SET LOCAL` **[verificar]** con conexiones persistentes; FastAPI → `contextvars` + sesión por petición.
- **Todo lo demás también es por tenant**: prefijo de clave en caché, prefijo en almacenamiento de archivos, colas y jobs con el tenant serializado, búsquedas, limitadores de tasa y cuotas por plan. Es la fuente más común de fugas fuera de la base de datos. **[criterio]**
- **Seeds con al menos dos tenants** para detectar fugas.
- Borrado y exportación por tenant con procedimiento probado (LGPD/GDPR).
- Autorización por objeto en cada acceso, además de RLS: RLS es defensa en profundidad, no el único control (ver `convenciones-api.md`, sección 6).

## 5. Integraciones y llaves del cliente (BYOK)

Reglas que conviene cumplir antes de dar por completo un módulo de integraciones **[criterio]**:
- Cada llave con **modo prueba y modo producción** intercambiables y un botón "probar conexión".
- **Guías dentro del sistema** para conseguir cada llave (por ejemplo, el paso a paso para generar el JSON de una cuenta de servicio).
- **Cifrado en reposo**; en pantalla solo los **últimos 4 caracteres**; registro de auditoría de altas, cambios y usos.
- **Prueba de IDOR**: el tenant A no puede leer, sobrescribir ni usar la llave del tenant B.
- Si hay un agente con modelo de lenguaje, se configura por empresa, con la llave del cliente.
- Nunca se registra el valor de una llave en logs; nunca viaja al cliente (navegador o app).
- Para canales de mensajería, usa la vía oficial del proveedor.
- Antes de aceptar el módulo, una revisión de seguridad; y si maneja datos personales, revisa que los textos legales lo cubran.

**Comprobar**: prueba de integración que guarda una llave, verifica que en BD no está en claro, que la API devuelve solo 4 caracteres, que otro tenant obtiene 404 y que el log no la contiene.

## 6. Ciclo de alta de un tenant (plantilla)

1. El formulario central crea empresa + administrador + membresías (idempotente: `Idempotency-Key` o clave natural).
2. El servicio central llama al endpoint interno de aprovisionamiento de cada producto elegido. Síncrono si es solo filas; asíncrono con aviso por correo si crea una base.
3. El producto crea filas del tenant con `tenant_id` y datos semilla, **sin reiniciar** nada.
4. Se crea la demo solo una vez por producto.
5. **Comprobar**: prueba de aislamiento automatizada y verificación de que el alta no cambió procesos ni contenedores **[verificar]**.

## 7. Pruebas de aislamiento (obligatorias en CI)

- Como rol de aplicación y tenant A: leer, actualizar y borrar filas del tenant B → 0 filas/errores, en **cada tabla con `tenant_id`**.
- Sin tenant fijado: toda tabla y toda vista devuelven 0 filas.
- Token de A con ruta de B: rechazado.
- Jobs en cola de A nunca procesados con el contexto de B.
- Archivos y caché: claves de A inaccesibles desde B.
- Estas pruebas corren contra **PostgreSQL real** (contenedor), con el rol sin privilegios, no contra mocks ni SQLite.
