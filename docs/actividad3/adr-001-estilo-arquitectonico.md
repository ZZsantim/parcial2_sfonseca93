# ADR-001: Una PIA de dos carriles con núcleo modular

| Campo | Valor |
|---|---|
| **Identificador** | ADR-001 |
| **Título** | Separar consultas y comandos en dos carriles dentro de un monolito modular |
| **Fecha** | 2026-10-14 |
| **Estado** | Aceptada para la propuesta |

## Contexto y problema

La Plataforma de Inscripción Académica (PIA) debe atender consultas e inscripciones durante períodos de alta concurrencia, preservar la capacidad autorizada de los grupos, evitar duplicados por reintentos, integrar el sistema académico legado y permitir cambios futuros en reglas académicas. La universidad dispone de Linux y PostgreSQL, no autoriza inicialmente contratar servicios comerciales adicionales y asignará cuatro desarrolladores y un administrador de infraestructura.

La arquitectura debe proteger las transacciones críticas sin introducir una carga operativa desproporcionada. También debe aislar las reglas académicas de la interfaz, la persistencia y el protocolo particular del sistema legado. El escenario individual agrega 50% a la concurrencia base; la solución debe evitar que la lentitud de una consulta remota del legado retenga recursos de las transacciones de cupos.

## Driver arquitectónico relacionado

- **D-07 (P2):** modificabilidad y mantenibilidad para incorporar reglas académicas con bajo riesgo.
- **D-01 y D-02 (P0):** consistencia de cupos e idempotencia, que favorecen una transacción local en un almacén ACID.
- **R-01 y R-05:** infraestructura institucional Linux/PostgreSQL y equipo de mantenimiento reducido.

## Alternativas consideradas

1. **Monolito modular de dos carriles (seleccionado):** una aplicación desplegable con módulos hexagonales; dentro de ella, las consultas de lectura y los comandos que cambian inscripciones usan rutas de ejecución diferenciadas.
2. **Microservicios:** varios servicios desplegables y escalables independientemente, comunicados por red.
3. **Monolito en capas sin límites modulares explícitos:** una aplicación desplegable, con responsabilidades organizadas principalmente por capas técnicas y sin límites claros de dominio.

## Criterios de comparación

- Preservación de la atomicidad en inscripción, cupo, idempotencia y registro de eventos.
- Complejidad de despliegue, monitoreo y recuperación para el equipo institucional.
- Aislamiento de las reglas de negocio respecto a infraestructura y sistemas externos.
- Facilidad para evolucionar reglas académicas sin regresiones.
- Compatibilidad con Linux/PostgreSQL y ausencia de nuevas dependencias comerciales obligatorias.
- Capacidad de aislar el trabajo de lectura del camino crítico de escritura sin crear una infraestructura distribuida.

## Decisión

Adoptar un **monolito modular de dos carriles con arquitectura hexagonal**. La PIA se desplegará como una aplicación institucional, con módulos de oferta, reglas académicas, inscripciones, auditoría e integración. Los casos de uso dependerán de puertos; los adaptadores encapsularán PostgreSQL, la interfaz HTTP y el protocolo que soporte el sistema legado.

Los carriles son una separación lógica de prioridades, no microservicios ni despliegues distintos:

- **Carril de lectura:** consultas de oferta y estado, de solo lectura, evaluadas con el objetivo p95 ≤ 2 segundos bajo la carga de 1.800 estudiantes. No puede reservar cupos ni declarar una inscripción confirmada.
- **Carril de comando:** inscribir y cancelar. Prioriza la corrección transaccional sobre la velocidad de una lectura; aplica idempotencia y escribe todos los efectos locales juntos. Una llamada de red al legado ocurre fuera de la transacción de cupo.

Ambos carriles usan el mismo dominio modular y PostgreSQL institucional. La priorización se implementa con límites de trabajo y tiempos de espera en la aplicación, sin asumir que existen colas, balanceadores o servicios comerciales concretos.

El módulo de inscripciones será el dueño de las invariantes de cupo y del ciclo de vida de las inscripciones. PostgreSQL será la autoridad de las inscripciones del período administradas por la PIA; el sistema legado conservará la autoridad del historial académico y los prerrequisitos.

La aplicación podrá ejecutarse en varias instancias si la infraestructura institucional lo permite, sin depender de estado de negocio en memoria local. Esto es una capacidad del diseño, no una afirmación de que las metas de disponibilidad o carga ya estén demostradas.

## Consecuencias

### Positivas

- La reserva de cupo, la inscripción, el resultado idempotente, la auditoría y el evento de sincronización pueden escribirse dentro de una transacción local de PostgreSQL.
- Los límites entre dominio, casos de uso y adaptadores facilitan probar reglas sin depender del sistema legado ni de una interfaz concreta.
- Las consultas y comandos tienen objetivos y métricas diferenciados: una pantalla rápida nunca se confunde con una reserva, y las consultas no deben consumir indefinidamente los recursos del camino crítico de matrícula.
- La operación y el despliegue iniciales son más simples que los de una arquitectura distribuida, adecuados a la capacidad del equipo.
- Un protocolo diferente del legado puede encapsularse en su adaptador sin propagar detalles al dominio.

### Negativas

- Las partes de la PIA comparten ciclo de despliegue; una actualización requiere desplegar la aplicación completa.
- No es posible escalar un módulo de forma independiente; el escalado horizontal se aplica a la aplicación completa y depende de que la infraestructura y PostgreSQL lo soporten.
- Los límites modulares y de carril requieren revisiones y pruebas para prevenir dependencias indebidas y evitar que una consulta invoque accidentalmente una escritura.

### Riesgos y mitigaciones

- **Riesgo:** el monolito se degrada a una aplicación acoplada. **Mitigación:** definir dependencias permitidas entre módulos y verificarlas mediante revisiones y controles automáticos del repositorio.
- **Riesgo:** PostgreSQL se convierte en cuello de botella ante la carga de matrícula. **Mitigación:** ejecutar pruebas con 1.800 estudiantes concurrentes, observar saturación y latencias, y dimensionar la infraestructura antes de la puesta en producción.
- **Riesgo:** la separación lógica de carriles existe en el diagrama, pero no en la ejecución. **Mitigación:** exponer métricas separadas de latencia/errores para lectura y comandos y comprobarlas bajo carga mixta.
- **Riesgo:** el equipo interpreta el despliegue modular como garantía de alta disponibilidad. **Mitigación:** validar por separado topología, respaldos, recuperación y objetivos RTO/RPO.

## Condiciones para revisar la decisión

- Pruebas representativas demuestran que la aplicación no puede satisfacer las metas de rendimiento aun después de optimizar consultas y escalar horizontalmente en la infraestructura disponible.
- Un módulo tiene requisitos comprobados de escalado, despliegue o aislamiento de fallas que no pueden cubrirse con límites internos y escalado de la aplicación.
- Cambia sustancialmente la capacidad operativa del equipo o se autoriza infraestructura que reduzca el costo de operar servicios distribuidos.
- Los límites modulares no consiguen aislar reglas y cambios después de varias iteraciones, según métricas de defectos, acoplamiento y esfuerzo de mantenimiento.
- La medición demuestra que carriles lógicos no aíslan suficientemente la carga y la infraestructura institucional permite y necesita un mecanismo adicional de aislamiento.
