# skill-arquitectura-backend

**Una sola arquitectura para todos tus backends: cambia el framework, no las reglas. Y cada regla trae su comprobación.**

Una skill con un estándar de arquitectura de backend independiente de la tecnología, con apéndice por stack (NestJS + Prisma, Laravel, CodeIgniter 4 y FastAPI) y un criterio claro: **una regla sin comprobación automática no cuenta como cumplida.**

---

## ¿Qué es esto?

Una *skill* es un archivo de texto con tu criterio escrito. Tu herramienta de desarrollo lo lee y construye o revisa código de servidor con esas reglas.

Esta skill define **qué importa qué, qué está prohibido y cómo se comprueba cada regla** en un backend.

## El problema que resuelve

Cuando programas con IA, la herramienta escribe código que funciona hoy y se vuelve inmanejable en tres meses: lógica de negocio en los controladores, SQL por todos lados, módulos que se importan entre sí, errores de API distintos en cada endpoint.

Y cuando coexisten varios stacks (un backend en Node, otro en PHP, otro en Python), cada uno acaba con su propia filosofía.

La skill fija **una estructura común** y una **regla de dependencias** que se puede verificar con una herramienta de CI, no con buena voluntad.

## Para qué sirve

- Arrancar un backend nuevo con una estructura clara: **monolito modular**, módulos como bounded contexts y Clean/Hexagonal pragmática por feature.
- Decidir **cuándo sí y cuándo no** extraer un microservicio (solo con una razón medible).
- Diseñar el **contrato HTTP**: errores (Problem Details), paginación, versionado, idempotencia, autenticación, autorización por objeto, rate limiting, logs, CORS.
- Construir un **backend multi-tenant / SaaS** sin fugas entre clientes (fila + RLS por defecto).
- **Revisar** un backend existente con una checklist.

## Cuándo usarla

- Vas a crear un endpoint, un módulo o un servicio nuevo.
- Quieres que tu herramienta deje de mezclar capas.
- Estás revisando un PR de backend o heredando un proyecto.

## Cuándo **no** usarla

- Es un script de una tarde o un prototipo desechable: la estructura es demasiada ceremonia.
- Tu proyecto es solo frontend.
- Necesitas decisiones de esquema SQL muy específicas: la skill las deja a una guía de base de datos aparte.

## Cómo instalarla

1. Descarga este repositorio.
2. Copia su contenido (`SKILL.md` y la carpeta `references/`) dentro de una carpeta llamada `arquitectura-backend` en el directorio de skills de tu herramienta de desarrollo.
3. Pídele: *"Con la skill arquitectura-backend, diseña el módulo de facturación"* o *"revisa este PR con su checklist"*.

Cada herramienta guarda sus skills en un sitio distinto; consulta la documentación de la tuya.

## Ejemplo

**Tú:** "Agrega el endpoint para cancelar una reserva."

**Con la skill, la herramienta:** crea un caso de uso (una transacción), pone la validación de forma en la capa HTTP, la autorización por objeto en el caso de uso y los efectos externos (correo, cobros) **después** del commit; devuelve los errores en formato Problem Details; no mete SQL en el controlador; y propone la comprobación de CI que impide romper la regla de dependencias.

## Qué incluye la skill

- El **veredicto** en cinco puntos (monolito modular, Clean/Hexagonal por feature, DDD estratégico siempre y táctico solo donde hay reglas complejas, CQRS ligero, microservicios solo con razón medible).
- La **regla de dependencias** (única e innegociable) y la estructura común.
- Lo **prohibido**, el **contrato HTTP mínimo** y cómo empezar una tarea de backend.
- Archivos de apoyo en `references/`: principios y decisiones (con plantilla de ADR), estructura por stack, convenciones de API, multi-tenancy y SaaS, pruebas y calidad (qué oculta un test en verde) y una checklist de revisión.
- Cada afirmación va marcada: **[fuente]** (documentación citada), **[criterio]** (recomendación de la skill) o **[verificar]** (sin comprobar).

## Límites honestos

- Es una **recomendación fundamentada, no una ley**: no existe una estructura oficial de Clean Architecture para estos frameworks.
- Lo marcado **[verificar]** hay que probarlo en tu proyecto antes de volverlo norma.
- `Idempotency-Key` y `RateLimit-*` son borradores de IETF, no RFC finales.

## Skills relacionadas

[`skill-documentacion`](https://github.com/nanoizax/skill-documentacion) y [`skill-qa`](https://github.com/nanoizax/skill-qa).

## Licencia y autoría

[MIT](LICENSE) · Hecha por Leandro Manuel Perez Montañana.
Más skills y herramientas gratis en **[soytegno.sonholab.com](https://soytegno.sonholab.com)** · TikTok [@soytegno](https://www.tiktok.com/@soytegno)
