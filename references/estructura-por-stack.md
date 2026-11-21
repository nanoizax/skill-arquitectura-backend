# Estructura por stack

Mismas reglas de dependencia (`principios-y-decisiones.md`, sección 3), distinta sintaxis. **No existe una estructura oficial de Clean Architecture para estos frameworks**: los árboles combinan guías de comunidad y criterio de esta guía. Las herramientas de CI son las estándar de cada ecosistema, recomendadas por conocimiento general; toda configuración marcada **[verificar]** debe probarse en el proyecto antes de darla por norma (la prueba mínima: introducir a propósito un import prohibido y confirmar que el CI falla).

Reglas comunes a los cuatro stacks:
- El nombre del módulo es un bounded context (`billing`, `catalog`, `identity`), no un tipo técnico.
- Solo `public/` (fachada) es importable desde otro módulo.
- El cableado de puertos → implementaciones vive en un solo archivo por módulo.
- Validación y formato de error compartidos viven en `shared/`/`core/`, no se copian por módulo.

---

## 1. NestJS (TypeScript) + Prisma

Nest ya impone módulos: úsalos como fronteras del monolito. Fuentes de la comunidad recomiendan estructura por feature y pruebas por bounded context. **[fuente]**

```
src/
  main.ts  app.module.ts
  shared/                        # kernel pequeño: config, logger, problem-details, tenant-context, errores base
  modules/
    billing/
      billing.module.ts          # exporta SOLO el provider de la fachada
      public/billing.api.ts      # interfaz + token de inyección (lo único importable desde fuera)
      domain/                    # entidades, VO, eventos: TS puro, sin @nestjs/* ni Prisma
      application/
        commands/   create-invoice.handler.ts
        queries/    list-invoices.handler.ts
        ports/      invoice.repository.ts       # interfaz + token (Symbol o clase abstracta)
      infrastructure/
        persistence/ invoice.prisma-repository.ts  # único lugar que importa PrismaService
        http/        invoice.controller.ts, dto/*.ts
      billing.e2e-spec.ts
prisma/  schema.prisma  migrations/   # migraciones versionadas
```

Dónde va cada cosa:
- **DTO de entrada/salida y decoradores de validación** (`class-validator` o Zod): `infrastructure/http/dto`. Nunca en `domain`.
- **Tipos generados por Prisma** (`Prisma.*`, `@prisma/client`): solo en `infrastructure/persistence`. El repositorio mapea a entidades o DTOs de `application`.
- **Filtro de excepciones global** que emite `application/problem+json`: `shared/problem-details`.
- **Tenant**: middleware o guard que resuelve el tenant y lo guarda con `AsyncLocalStorage` (p. ej. `nestjs-cls`); ver `multi-tenancy-y-saas.md` para la transacción con `set_config`. Evita providers `REQUEST`-scoped (degradan rendimiento). **[criterio]**
- **CQRS**: handlers simples; `@nestjs/cqrs` es opcional y no se exige.
- Endpoints que pueden devolver `null`: ver trampa en `pruebas-y-calidad.md` (Nest envía 0 bytes).

Regla de dependencias, cumplimiento en CI:
- Opción A `dependency-cruiser`: reglas `forbidden` con `from`/`to` por ruta. Ejemplo **[verificar]**:
  ```js
  // .dependency-cruiser.cjs (fragmento)
  forbidden: [
    { name: 'domain-puro', severity: 'error',
      from: { path: '^src/modules/[^/]+/domain' },
      to:   { path: ['^src/modules/[^/]+/(application|infrastructure)', 'node_modules/(@nestjs|@prisma|class-validator)'] } },
    { name: 'application-sin-infra', severity: 'error',
      from: { path: '^src/modules/[^/]+/application' },
      to:   { path: '^src/modules/[^/]+/infrastructure' } },
    { name: 'modulos-solo-por-public', severity: 'error',
      from: { path: '^src/modules/([^/]+)/' },
      to:   { path: '^src/modules/([^/]+)/(domain|application|infrastructure)', pathNot: '^src/modules/$1/' } },
  ]
  ```
  Comando en CI: `npx depcruise src --config .dependency-cruiser.cjs` (falla con código distinto de 0 ante `error`).
- Opción B `eslint-plugin-boundaries` o `no-restricted-paths` de `eslint-plugin-import` **[verificar]** (útil si ya hay ESLint en el pipeline).
- **Prueba de arranque**: un guard o provider sin su módulo importado compila y **no arranca** (`UnknownDependenciesException`). El CI debe levantar el `dist` y esperar el mensaje de arranque exitoso; no basta con `tsc`/`nest build`. Ver `pruebas-y-calidad.md`.
- En el servidor con PM2: si el script es un *shim* de `node_modules/.bin`, declarar `interpreter: 'none'`; y recordar que `pm2 restart` no relee el `.env` (hay que `delete` + `start`). Ver `convenciones-api.md` sección de configuración.

---

## 2. Laravel (PHP)

Laravel no separa dominio y framework (Eloquent es Active Record). Modularización disponible: `nwidart/laravel-modules`, `internachi/modular`, o carpetas propias con providers; patrón de clases *Action* de propósito único; DTO con `spatie/laravel-data`. **[fuente]**

```
app-modules/ (o src/Modules/)
  Billing/
    src/
      Api/             # Controllers, FormRequests, Resources, routes.php
      Application/     # Actions (CreateInvoice), Queries, DTOs, Contracts (puertos)
      Domain/          # Entidades/VO/eventos en PHP puro (solo si hay reglas)
      Infrastructure/  # Models Eloquent, Repositories, Jobs, Listeners, Providers
      Public/          # fachada que otros módulos pueden usar
    database/migrations/   # migraciones versionadas
    tests/
    BillingServiceProvider.php
```

Dónde va cada cosa:
- Controlador delgado → `Action` → modelo/repositorio. `FormRequest` valida; `API Resource` serializa; `Policy`/`Gate` autorizan por objeto.
- **Eloquent queda en `Infrastructure`** del módulo; ningún otro módulo toca sus modelos (solo DTOs o la fachada). `Domain` y `Application` sin `Illuminate\*` salvo excepción registrada en ADR.
- `withoutGlobalScopes()` prohibido salvo en código de administración auditado (rompe el aislamiento de tenant).
- Errores: manejarlos en `bootstrap/app.php` (Laravel 11+) o en el `Handler` para emitir Problem Details.
- Límites de tasa: **siempre limitadores con nombre** (`RateLimiter::for('otp-send', …)` y `throttle:otp-send`); dos `throttle:N,M` anónimos en la misma ruta comparten clave y cuentan doble. Ver `convenciones-api.md`.
- Tenant: paquetes `stancl/tenancy` o `spatie/laravel-multitenancy` (ambos soportan BD única) **[fuente]**, o global scope propio + RLS. Serializa el tenant en cada job de cola.
- Exclusión de CSRF para webhooks y retornos de pasarelas: verificarla con `curl`, no con PHPUnit (ver `pruebas-y-calidad.md`).

Regla de dependencias, cumplimiento en CI:
- `deptrac` (capas y colectores por namespace) o `phparkitect`; o pruebas de arquitectura de Pest **[verificar]**:
  ```php
  arch('domain no usa framework')->expect('Modules\Billing\Domain')
      ->not->toUse(['Illuminate', 'Modules\Billing\Infrastructure', 'Modules\Billing\Api']);
  arch('modulos solo via Public')->expect('Modules\Billing\Infrastructure')
      ->not->toUse(['Modules\Catalog\Infrastructure', 'Modules\Catalog\Domain']);
  ```
- Tras subir de versión mayor de Laravel: `php artisan view:clear` **antes** de correr la suite (la caché de Blade compilada con la versión anterior puede tapar errores). Escapa `@@` los `@` literales en plantillas (JSON-LD).

---

## 3. CodeIgniter 4 (PHP)

CI4 soporta módulos por namespaces PSR-4 en `app/Config/Autoload.php` (`$psr4`); un módulo puede tener Controllers, Models, Config, Database/Migrations. **[fuente]** No tiene inyección obligatoria ni autowiring rico: la disciplina debe venir del equipo.

```
app/Config/Autoload.php        # $psr4['Modules\\Billing'] = ROOTPATH.'modules/Billing'
modules/
  Billing/
    Api/             # Controllers (BaseController/ResourceController), Filters, Routes.php
    Application/     # UseCases (__invoke/execute), DTOs, Ports (interfaces)
    Domain/          # Entidades/VO PHP puro
    Infrastructure/  # Models (CodeIgniter\Model), repositorios, clientes externos
    Public/          # fachada
    Config/Services.php     # cableado puertos → implementaciones
    Database/Migrations/    # migraciones versionadas
    Tests/
```

Dónde va cada cosa:
- Controller sin lógica ni consultas: valida con el servicio `validation`, llama a un caso de uso, responde.
- Autenticación, tenant y rate limiting en `Filters`.
- Cableado con `Config\Services` (o factories), centralizado por módulo.
- Errores: `failValidationErrors()` **no** es Problem Details; crea un formateador propio compartido.
- Tenant: Filter que resuelve el tenant y lo fija en un servicio compartido; `SET LOCAL app.tenant_id` al inicio de cada transacción (cuidado con conexiones persistentes) **[verificar]**.
- IP real detrás de proxy: configura `proxyIPs` y confía en la cabecera que el proxy **sobrescribe** (`X-Real-IP`), no en `X-Forwarded-For` anexada (ver `convenciones-api.md`).
- Pruebas o comandos que simulen `$_SERVER`: usar `service('superglobals')->setServer(...)`; escribir en `$_SERVER` directamente no surte efecto en CI4 4.7. **[fuente: nota interna]**
- `Throttler` con caché no escribible falla abierto (`dummy` como respaldo): registra `critical` o falla cerrado.

Regla de dependencias, cumplimiento en CI: `deptrac` o `phparkitect` sobre `modules/` **[verificar]**, con las mismas capas que Laravel.

---

## 4. FastAPI (Python)

Recomendación difundida: estructura **por dominio** (inspirada en Netflix Dispatch), no por tipo de archivo. `async def` solo con I/O no bloqueante; `def` para código bloqueante (threadpool); CPU pesada fuera del event loop. **[fuente]**

```
src/
  main.py
  core/  config.py  logging.py  problem_details.py  tenant_context.py  db.py
  billing/
    api/            router.py  schemas.py (Pydantic)  dependencies.py
    application/    commands.py  queries.py  ports.py
    domain/         entities.py  value_objects.py  events.py     # Python puro
    infrastructure/ orm_models.py (SQLAlchemy)  repositories.py  clients.py
    public.py       # fachada
  tests/
migrations/ (Alembic)   # migraciones versionadas
```

Dónde va cada cosa:
- Pydantic, SQLAlchemy y FastAPI solo en `api` e `infrastructure`; `domain` usa `dataclasses` o clases simples.
- `Depends` solo en el borde; los casos de uso reciben puertos por constructor.
- Errores: `exception_handler` global para Problem Details, incluido `RequestValidationError` → 422 con `errors[]`.
- Tenant: `Depends(get_tenant)` + `contextvars`; sesión por petición que ejecuta `set_config('app.tenant_id', …, true)` dentro de la transacción.
- Configuración con `pydantic-settings` (falla al arrancar si falta algo).

Regla de dependencias, cumplimiento en CI: `import-linter` **[verificar]**:
```ini
# .importlinter
[importlinter]
root_packages = src
[importlinter:contract:capas]
name = Capas por modulo
type = layers
layers =
    src.billing.api | src.billing.infrastructure
    src.billing.application
    src.billing.domain
[importlinter:contract:dominio-puro]
name = Dominio sin framework
type = forbidden
source_modules = src.billing.domain
forbidden_modules = fastapi, pydantic, sqlalchemy
```
Comando: `lint-imports`. Para independencia entre módulos, contrato `independence` con `src.billing` y `src.catalog` **[verificar]**.

---

## 5. Tabla de equivalencias

| Concepto | NestJS | Laravel | CI4 | FastAPI |
|---|---|---|---|---|
| Borde HTTP | Controller + DTO | Controller + FormRequest + Resource | Controller + Filter | Router + Pydantic |
| Caso de uso | Handler | Action | UseCase | función/clase en `commands.py` |
| Puerto | interfaz + token | Contract | interfaz | `Protocol`/ABC |
| Implementación | repositorio Prisma | repositorio Eloquent | repositorio sobre Model | repositorio SQLAlchemy |
| Errores globales | ExceptionFilter | `bootstrap/app.php` | handler propio | `exception_handler` |
| Contexto de tenant | `AsyncLocalStorage` | scope + middleware | Filter + servicio | `contextvars` |
| Comprobación de capas | dependency-cruiser | deptrac / arch tests | deptrac | import-linter |
| Validar config | `ConfigModule` + Zod/Joi | validación propia en provider | chequeo en `Config` | `pydantic-settings` |

Las migraciones y el esquema se gestionan siempre con migraciones versionadas: Prisma Migrate, migraciones de Laravel/CI4 o Alembic según el proyecto.
