---
name: arquitectura-backend
description: Arquitectura estándar de todos los backends, independiente de la tecnología, con apéndice por stack (NestJS + Prisma, Laravel, CodeIgniter 4, FastAPI). Úsala SIEMPRE que se hable de arquitectura, backend, API, capas, clean architecture, hexagonal, módulo, bounded context, estructura de carpetas, servicio, caso de uso, controlador, repositorio, dependencias entre módulos, monolito o microservicios, endpoint nuevo, errores de API (Problem Details), paginación, versionado, idempotencia, autenticación, autorización por objeto, rate limiting, logs, configuración por entorno, CORS, multi-tenant o SaaS, o cuando un plan o tarea cree o cambie código de servidor — aunque el pedido no diga "arquitectura" explícitamente. Define qué importa qué, qué está prohibido y cómo se comprueba cada regla.
---

# Arquitectura de backend — estándar recomendado

Una sola forma de construir backends, sea cual sea el framework. Cambia la sintaxis, no las reglas. Esta skill manda sobre **estructura de código y contrato HTTP**. Lo de **esquema, migraciones, roles y RLS** se trata aparte (una guía de base de datos del proyecto): enlázala, no la dupliques.

## Veredicto (resumen; justificación en `references/principios-y-decisiones.md`)

1. **Monolito modular**: un despliegue, módulos = bounded contexts, datos propios por módulo.
2. **Por dentro de cada módulo, Clean/Hexagonal pragmática organizada por feature.** Nada demostrable es mejor que la familia Clean/Hexagonal/Onion; lo que importa es la regla de dependencias. Es el estándar que propone esta skill.
3. **DDD estratégico siempre** (contextos, lenguaje). **DDD táctico** (agregados, value objects) **solo** donde haya reglas de negocio complejas (cobros, reservas, facturación).
4. **CQRS ligero**: comandos y consultas separados como casos de uso distintos; sin event sourcing ni buses obligatorios.
5. **Microservicios solo con una razón medible** (ver abajo). Por defecto no.

## Regla de dependencias (única e innegociable)

```
api ──► application ──► domain
infrastructure ──► application ──► domain
```
- `domain`: no importa nada del framework, ORM, HTTP ni de otras capas. Comprobar: `grep` de imports prohibidos + herramienta de CI del stack.
- `application`: casos de uso y **puertos** (interfaces). Importa `domain`. No conoce HTTP ni ORM.
- `api` (HTTP) e `infrastructure` (BD, colas, correo, clientes externos) dependen hacia adentro, nunca al revés.
- **Entre módulos**: solo por su API pública (fachada o evento). Prohibido importar internos de otro módulo; prohibido JOIN o FK a tablas de otro módulo (se referencia por ID). Comprobar: la herramienta de CI falla el build.
- **Una regla sin comprobación automática no cuenta como cumplida.** Cada stack tiene su herramienta en `references/estructura-por-stack.md`.

## Estructura común

```
modulo/                      # un bounded context (billing, catalog, identity…)
  api/ o http/               # controladores/rutas, DTOs de entrada y salida, mapeo de errores
  application/               # casos de uso (commands/queries), puertos, transacción
  domain/                    # entidades, value objects, reglas, eventos (puro)
  infrastructure/            # repositorios, ORM, clientes externos, colas
  public/ (fachada)          # lo único que otros módulos pueden importar
shared/ (kernel pequeño)     # config validada, logging, Problem Details, contexto de tenant
```
Organiza por feature, no por tipo de archivo global (`controllers/`, `services/`, `models/` en la raíz está prohibido).

## Qué hace cada capa

| Capa | Hace | No hace |
|---|---|---|
| api | Valida forma (tipos, tamaños, lista blanca), autentica, llama a **un** caso de uso, serializa, traduce errores | Lógica de negocio, consultas, transacciones |
| application | Orquesta un caso de uso = **una transacción**; autoriza por objeto; dispara efectos externos **después** del commit | Conocer HTTP, SQL o el ORM concreto |
| domain | Invariantes y reglas de negocio | Importar framework, ORM, red o reloj global |
| infrastructure | Implementa puertos; aísla al proveedor externo | Decidir reglas de negocio |

**Pragmatismo**: en un CRUD sin reglas, un slice puede ser controlador → caso de uso → ORM, sin dominio puro ni repositorio. Las entidades ricas y los puertos se reservan para reglas reales y fronteras reales (pagos, correo, WhatsApp, almacenamiento). Si añadir un campo toca más de 4-5 archivos en capas distintas, sobra ceremonia.

## Monolito modular vs microservicios

**Por defecto monolito modular.** Extrae un módulo a servicio solo si cumples **al menos una**, con evidencia: (a) perfil de escalado radicalmente distinto, (b) aislamiento de fallos o cumplimiento exigido, (c) equipo o ritmo de despliegue realmente independiente, (d) tecnología distinta necesaria. Registra la decisión como ADR. Cómo dejarlo barato: fronteras de módulo reales (sin JOINs cruzados), comunicación por interfaz o evento.

## Prohibido

- Lógica de negocio o SQL en controladores; `domain` importando framework u ORM.
- Un módulo importando internos de otro, o uniendo sus tablas.
- Tomar `tenant_id`, `user_id` o `rol` del cuerpo o de la URL sin contrastarlo con el contexto autenticado.
- Conectar la aplicación con el rol dueño o con `BYPASSRLS`.
- Errores con formato propio por endpoint; devolver trazas o detalles internos al cliente.
- Dar por bueno un cambio porque "compila" o "los tests pasan" sin arrancar el servicio y probar por el camino del cliente real (ver `references/pruebas-y-calidad.md`).
- Crear instancias, contenedores o procesos por cliente (ver `references/multi-tenancy-y-saas.md`).

## Contrato HTTP mínimo (detalle en `references/convenciones-api.md`)

Errores RFC 9457 (`application/problem+json`) con `code` estable y `trace_id` · paginación por cursor con orden determinista · `/v1/` en la ruta · autorización por objeto en cada acceso · validación en el borde y en el dominio · límites de tasa con `429` y `Retry-After` · logs JSON con `trace_id` · configuración validada al arrancar · textos de cara al usuario en la variedad del mercado del cliente (tuteo neutro; pt-BR en Brasil).

## Cómo empezar una tarea de backend

1. Identifica el módulo (bounded context) al que pertenece. Si no existe, propón el contexto antes de crear carpetas.
2. Si toca datos: el cambio de esquema va por una migración versionada y revisada; no ejecutes DDL a mano.
3. Escribe el caso de uso y su prueba de integración con BD real antes del controlador.
4. Corre la herramienta de dependencias del stack y el arranque en seco del servicio.
5. Revisa con `references/checklist-de-revision.md`; registra las decisiones nuevas como ADR.

## Honestidad sobre la evidencia

Lo marcado **[criterio]** es recomendación de esta skill, no hecho citado. Lo marcado **[verificar]** no se comprobó y debe probarse antes de volverlo norma. `Idempotency-Key` y `RateLimit-*` son **borradores** de IETF, no RFC finales.

## Archivos de apoyo

- `references/principios-y-decisiones.md` — veredicto con justificación, cuándo romper la regla, plantilla de ADR.
- `references/estructura-por-stack.md` — árbol, regla de dependencias y comprobación en CI para NestJS, Laravel, CodeIgniter 4 y FastAPI.
- `references/convenciones-api.md` — errores, paginación, versionado, idempotencia, validación, autenticación, rate limiting, logs, configuración, CORS y cabeceras.
- `references/multi-tenancy-y-saas.md` — una instancia por producto, identidad central, tenant por ruta o `Host`, `tenant_id` + RLS, integraciones con llaves del cliente.
- `references/pruebas-y-calidad.md` — pirámide, contrato, y lo que un test en verde oculta (casos reales).
- `references/checklist-de-revision.md` — lista para que el líder o QA verifique que un backend cumple.
