# Principios y decisiones de arquitectura

Síntesis de la investigación del 30-sep-2026. Convención: **[fuente]** viene de las páginas consultadas (lista al final); **[criterio]** es recomendación de esta skill o inferencia, no un hecho citado; **[verificar]** no se comprobó.

## 1. Veredicto

1. **No hay evidencia de una arquitectura demostrablemente mejor que Clean/Hexagonal/Onion.** Robert C. Martin las presenta como variantes de una misma idea (con Screaming, DCI y BCE) y formula la regla común: las dependencias del código fuente solo apuntan hacia adentro. **[fuente]** No existen estudios controlados que muestren que una sea superior; hay opiniones y experiencias. **[criterio]**
2. Lo que sí cambia el resultado es operar en **dos niveles** (Jovanović): *macro* = monolito modular (cómo se parte el sistema en módulos con datos propios); *micro* = cómo se organiza el código dentro del módulo. **[fuente]**
3. **Estándar recomendado:** monolito modular + Clean/Hexagonal pragmática por feature + DDD estratégico siempre + DDD táctico solo con reglas complejas + CQRS ligero. Un solo despliegue. **[criterio]**
4. Microservicios, CQRS completo y event sourcing: solo con razón medible.

## 2. Por qué esa combinación

| Enfoque | Qué responde | Decisión recomendada |
|---|---|---|
| Monolito modular | Cómo dividir el sistema | **Siempre** como punto de partida |
| Clean/Hexagonal/Onion | Regla de dependencias dentro del módulo | **Estándar**, en versión pragmática |
| Vertical Slice (Bogard) | Organizar por caso de uso; cada slice elige su estilo | Se adopta la **organización por feature**; el estilo interno lo decide la complejidad |
| DDD estratégico | Contextos, lenguaje ubicuo | **Siempre** |
| DDD táctico | Agregados, value objects, eventos | Solo donde hay invariantes reales |
| CQRS | Separar escritura y lectura | Ligero (casos de uso distintos); completo solo con cargas muy distintas |
| Microservicios | Despliegue y escalado independientes | Solo con criterios de la sección 4 |

Posturas de autores (resúmenes de sus artículos; no se leyeron libros completos) **[fuente]**:
- **Bogard**: critica capas rígidas, abstracciones innecesarias y pruebas cargadas de mocks; advierte que Vertical Slice exige equipos que detecten olores de código y refactoricen.
- **Jovanović**: un monolito modular no dice cómo organizar el interior del módulo; los slices llenan ese hueco, y cada módulo puede usar su propio estilo porque la frontera lo hace seguro.
- **Fowler (MonolithFirst)**: casi todas las historias exitosas de microservicios empezaron como monolito; la evidencia que ofrece es limitada y reconoce excepciones con equipos experimentados.
- **Simon Brown**: sus charlas sobre monolitos modulares no pudieron verificarse; no se le atribuye postura. El modelo C4 sirve para dibujar módulos.

Consecuencia práctica: **conserva la regla de dependencias (lo valioso) y reduce la ceremonia** (lo que critican Bogard y Jovanović).

## 3. Regla de dependencias: formulación verificable

| # | Regla | Cómo se comprueba |
|---|---|---|
| R1 | `domain` no importa `application`, `api`, `infrastructure` ni el framework/ORM | Herramienta de CI del stack (ver `estructura-por-stack.md`) |
| R2 | `application` importa solo `domain` y sus propios puertos | Ídem |
| R3 | `api` e `infrastructure` no se importan entre sí | Ídem |
| R4 | Un módulo solo importa `public/` de otro | Ídem, regla de "módulos independientes" |
| R5 | Sin JOIN ni FK entre tablas de módulos distintos | Revisión de migraciones + consulta al catálogo (`information_schema`) en CI **[criterio]** |
| R6 | Un caso de uso = una transacción; efectos externos después del commit | Prueba de integración: forzar fallo del proveedor externo y verificar que la BD no quedó a medias |
| R7 | `tenant_id`, `user_id`, `rol` salen del contexto autenticado | Prueba de aislamiento (ver `multi-tenancy-y-saas.md`) |

## 4. Monolito o microservicios

**No extraer** si: equipo menor de ~15-20 personas, una sola BD transaccional, producto buscando encaje, sin plataforma madura (CI/CD, observabilidad). **[criterio]**

**Extraer un módulo** si se cumple al menos una, con evidencia: (a) escalado radicalmente distinto (procesamiento de video, envío masivo de mensajes); (b) aislamiento de fallos o de cumplimiento; (c) equipo o ritmo de despliegue independiente; (d) tecnología distinta (p. ej. un worker Python). **[criterio]**

Casos citados **[fuente]**: Segment volvió de microservicios a monolito (2017) por costo operativo; Prime Video (un equipo, carga muy particular) pasó a un proceso único y reportó 90 % menos costo de infraestructura — no es condena general; Shopify escala un monolito Rails de ~2,8 M de líneas con componentes.

Hacer barata la extracción futura: API pública por módulo, sin JOINs cruzados, sin estado global, comunicación por interfaz o evento.

## 5. Cuándo romper la regla (y cómo dejarlo escrito)

Se permite romper una regla solo si: (1) hay una razón concreta (rendimiento medido, CRUD trivial, limitación del framework), (2) la excepción queda **acotada a un módulo** y (3) se registra un ADR con fecha de revisión.

| Excepción habitual | Condición |
|---|---|
| CRUD sin reglas: controlador → caso de uso → ORM, sin dominio puro ni repositorio | Sin invariantes de negocio; el día que aparezca una, se sube a dominio |
| Modelo del ORM como entidad (Eloquent, `CodeIgniter\Model`, Prisma) | El modelo nunca sale del módulo; otros módulos reciben DTOs o usan la fachada |
| Consulta de lectura que une tablas del mismo módulo con SQL directo | Permitido en `queries/`; nunca hacia tablas de otro módulo |
| Lectura agregada entre módulos (reportes) | Vista o tabla de lectura alimentada por eventos, no JOIN directo; o servicio de reportes con rol de solo lectura **[criterio]** |

**Nunca** se negocia: aislamiento de tenant, autorización por objeto, rol de la aplicación sin privilegios.

## 6. Señales de que el enfoque falla [criterio]

- Añadir un campo toca más de 4-5 archivos en capas distintas → ceremonia: pasa ese módulo a slices simples.
- Módulos que se importan internos o hacen JOINs cruzados → fronteras irreales: refuerza la herramienta de CI antes de crecer.
- Un tenant "ve" datos de otro en pruebas de aislamiento → detén el trabajo; revisa RLS, rol de conexión y fijación del tenant por transacción.
- Los tests necesitan más mocks que código → estás probando la implementación, no el comportamiento.

## 7. Registro de decisiones (ADR)

Un ADR por decisión que afecte fronteras, dependencias, seguridad o despliegue. Va en `docs/adr/NNNN-titulo.md` del repositorio del proyecto. Numeración correlativa, nunca se reescribe: se sustituye con otro ADR que lo referencia.

```markdown
# ADR-0007: <decisión en una frase>
- Fecha: AAAA-MM-DD      - Estado: propuesta | aceptada | sustituida por ADR-00NN
- Contexto: qué problema, qué restricciones, qué evidencia (enlaces, mediciones)
- Decisión: qué haremos, en imperativo
- Alternativas descartadas: y por qué
- Consecuencias: lo que ganamos, lo que pagamos, cómo se comprueba
- Revisión: fecha o condición que obliga a reabrirla
```

ADR obligatorio cuando: se extrae un módulo a servicio, se rompe una regla de la sección 3, se cambia de ORM o de modelo de tenancy, se añade un broker de mensajes, se elige una librería de autenticación.

## 8. Plan de adopción en un proyecto existente

1. Conserva la regla de dependencias y **reduce ceremonia**: quita repositorios/interfaces/mapeos que no aportan en módulos CRUD; mantén puertos en fronteras reales.
2. Reorganiza por módulo/feature en lugar de capas globales.
3. Define 4-8 bounded contexts (identidad/tenants, facturación, catálogo, notificaciones…), cada uno con API pública y tablas propias.
4. Automatiza en CI la regla de dependencias y las pruebas de aislamiento de tenant.
5. Estandariza en una plantilla interna: Problem Details, paginación por cursor, idempotencia, contexto de tenant, logging con `trace_id`, configuración validada. Es lo que más reutilización da entre stacks: mismo contrato HTTP, distinto framework.
6. No migres a microservicios, CQRS completo ni event sourcing sin razón medible.

## 9. Límites de la evidencia

- Varias afirmaciones de autores vienen de resúmenes de sus artículos, no de libros completos.
- La estructura de carpetas por stack combina guías de comunidad y criterio propio; no existe una "estructura oficial" de Clean Architecture para estos frameworks.
- Las recomendaciones de sesiones/JWT, expand/contract, pirámide de pruebas y observabilidad son práctica estándar aplicada por criterio, no verificadas en una fuente específica.
- Herramientas de CI por stack (dependency-cruiser, deptrac, import-linter…): recomendadas por conocimiento general del ecosistema, no por las búsquedas. **[criterio]**

## 10. Fuentes

- Martin, The Clean Architecture: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- Jovanović, vertical slices dentro del monolito modular: https://milanjovanovic.tech/blog/where-vertical-slices-fit-inside-the-modular-monolith-architecture
- Bogard, Vertical Slice Architecture: https://www.jimmybogard.com/vertical-slice-architecture/
- Fowler, MonolithFirst: https://martinfowler.com/bliki/MonolithFirst.html
- Grzybek, monolito y architectural drivers (InfoQ): https://www.infoq.com/news/2020/01/monolith-architectural-drivers/
- Shopify y su monolito: https://shopify.engineering/shopify-monolith
- Segment de vuelta al monolito (InfoQ): https://www.infoq.com/news/2020/04/microservices-back-again/
- Prime Video de vuelta al monolito: https://nordicapis.com/back-to-the-monolith-why-did-amazon-dump-microservices/
- C4: https://c4model.com/faq
