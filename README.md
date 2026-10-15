- **Estudiante:** Santiago Andres Fonseca Rojas
- **Documento:** 1023890893
**Caso:** Plataforma de Inscripción Académica (PIA)  
**Rama de trabajo:** `dev_sfonseca93`

# Plataforma de Inscripción Académica (PIA)

## Resumen ejecutivo

Esta propuesta define la arquitectura de una plataforma para consultar la oferta y gestionar la inscripción de asignaturas de la Universidad de La Salle. Busca mantener los cupos consistentes bajo concurrencia, impedir duplicados por reintentos, respetar al sistema académico legado como fuente oficial de prerrequisitos e historial, y hacer trazable el estado de cada operación.

Se propone un monolito modular con límites hexagonales, PostgreSQL como autoridad de las inscripciones del período y una integración recuperable con el sistema legado. El diseño del escenario de concurrencia contempla 1.800 estudiantes concurrentes y hasta cinco envíos de la misma solicitud. La propuesta es documentación arquitectónica; **este repositorio aún no contiene una implementación ejecutable de la PIA**.

## Datos del estudiante

- **Nombre:** Santiago Andres Fonseca Rojas
- **Documento:** 1023890893
- **Rama de trabajo:** `dev_sfonseca93`

## Navegación de la documentación

| Entregable | Contenido |
|---|---|
| [Actividad 1 — Drivers arquitectónicos](docs/actividad1.md) | Objetivos de negocio, drivers priorizados, restricciones, escenarios verificables y conflictos entre atributos de calidad. |
| [Actividad 2 — Diseño arquitectónico](docs/actividad2.md) | Estilo seleccionado, límites, responsabilidades, transacciones, comunicaciones y correspondencia con los drivers. |
| [C4 nivel 1 — Contexto](docs/diagramas/actividad2-contexto.svg) | Usuarios, PIA y sistema académico legado. |
| [C4 nivel 2 — Contenedores](docs/diagramas/actividad2-contenedores.svg) | Aplicación, almacenamiento, proceso de sincronización y sistema externo. |
| [C4 nivel 3 — Componentes](docs/diagramas/actividad2-componentes.svg) | Componentes del flujo de inscripción y del control de cupos. |
| [Actividad 3 — ADR 001](docs/actividad3/adr-001-estilo-arquitectonico.md) | Decisión sobre estilo arquitectónico y separación de consultas y comandos. |
| [Actividad 3 — ADR 002](docs/actividad3/adr-002-consistencia-cupos-idempotencia.md) | Decisión sobre plazas identificables, concurrencia e idempotencia. |
| [Actividad 3 — ADR 003](docs/actividad3/adr-003-integracion-legado-outbox.md) | Decisión sobre integración, evidencia de validación y sincronización recuperable. |
| [Actividad 4 — Tácticas](docs/actividad4.md) | Tácticas propuestas y métodos cuantificables para verificar su efectividad. |
| [Actividad 5 — Gobernanza](docs/actividad5.md) | Políticas, responsabilidades, revisión de ADR, controles de PR y gestión de excepciones. |

## Decisiones y tecnologías propuestas

- **Infraestructura objetivo:** Linux y PostgreSQL institucionales, de acuerdo con la restricción del caso.
- **Estilo:** monolito modular con límites hexagonales; no se presupone un lenguaje o framework.
- **Consistencia de cupos:** representar la capacidad como plazas identificables reclamadas dentro de una transacción PostgreSQL.
- **Integración:** puerto y adaptador para el legado; outbox durable para reintentar sincronizaciones sin afirmar entrega exactamente una vez si el sistema externo no la soporta.
- **Diagramas:** formato SVG vectorial dentro de `docs/diagramas/`.
- **Servicios comerciales:** no se consideran dependencias necesarias de la propuesta inicial.

La selección final de lenguaje, framework, protocolo del legado, proveedor de identidad, topología de despliegue y plataforma de CI queda pendiente de validación institucional; no se inventan decisiones para componentes que el enunciado no especifica.

## Uso y validación

Los documentos Markdown y los diagramas SVG se pueden consultar directamente desde el repositorio. No hay actualmente dependencias de aplicación, comandos de compilación ni pruebas ejecutables porque todavía no se ha construido el software.

Los criterios cuantificables de aceptación están descritos en las actividades 1 y 4, entre ellos: p95 ≤ 2 segundos para consultas de asignaturas, cero inscripciones por encima de la capacidad autorizada, idempotencia ante cinco envíos y ejercicios de recuperación para RTO/RPO. Deben ejecutarse contra una implementación y una infraestructura representativas antes de considerar demostrados esos objetivos.

## Entrega y contribuciones

- Mantener los entregables bajo `docs/` y actualizar los diagramas cuando cambie la estructura arquitectónica.
- Cada actividad debe tener al menos un commit independiente, de acuerdo con las instrucciones de entrega.
- Los cambios a decisiones arquitectónicas deben documentarse como ADR y revisarse con el procedimiento de [Actividad 5](docs/actividad5.md).
- Los cambios de comportamiento académico requieren aprobación de la autoridad académica definida por la universidad.
