# ADR-003: Integrar el legado con recibos de validación y doble reloj

| Campo | Valor |
|---|---|
| **Identificador** | ADR-003 |
| **Título** | Mantener evidencia de validación y separar el reloj local del reloj oficial |
| **Fecha** | 2026-10-14 |
| **Estado** | Aceptada para la propuesta; contrato de integración pendiente de validar |

## Contexto y problema

El sistema académico legado conserva la autoridad sobre el historial académico y los prerrequisitos, mientras que la PIA administrará las inscripciones del período y debe sincronizarlas con ese sistema. El legado puede no responder durante hasta 40 minutos. La PIA no debe presentar como verificada una condición académica cuya comprobación no pudo realizarse ni perder silenciosamente una inscripción confirmada si la sincronización falla después del commit local.

No se han especificado el protocolo del legado, su semántica de acuse, su capacidad de deduplicación ni una API. La arquitectura debe aislar esos detalles y mantener las operaciones pendientes identificables y recuperables. Además, el estudiante no debe tener que adivinar si una operación fue aceptada por la PIA, validada contra la fuente académica o recibida por el legado.

## Driver arquitectónico relacionado

- **D-05 (P0):** interoperabilidad y corrección académica frente a la fuente oficial.
- **D-04 (P1):** disponibilidad y recuperabilidad; los objetivos institucionales son disponibilidad mensual ≥ 99,9%, RTO ≤ 30 minutos y RPO ≤ 5 minutos.
- **D-02 (P0):** idempotencia para reintentos sin duplicar efectos.
- **C05:** interrupciones del legado de hasta 40 minutos.

Esta decisión es también la **decisión adicional** requerida por la actividad: persistir durablemente los cambios de integración para permitir reintentos y recuperación, en relación directa con D-04 y D-02.

## Alternativas consideradas

1. **Llamada síncrona directa al legado dentro de la operación de inscripción:** el caso de uso consulta o actualiza el legado y confirma en función de la respuesta.
2. **Recibo de validación y doble reloj (seleccionada):** combinar puerto/adaptador, evidencia mínima de la consulta oficial, estados explícitos de cada operación y outbox transaccional.
3. **Copia local de prerrequisitos más sincronización periódica por lotes:** consultar una réplica local aunque el legado no responda y reconciliar después.

## Criterios de comparación

- Respeto por la autoridad del legado para historial y prerrequisitos.
- Comportamiento explícito y seguro cuando el legado está fuera de servicio.
- Durabilidad de las operaciones confirmadas localmente ante fallas de red o reinicio de la aplicación.
- Prevención y detección de eventos duplicados tras reintentos.
- Complejidad y dependencias operativas adicionales.
- Posibilidad de conciliar discrepancias y explicar el estado de una operación al estudiante y al personal.
- Capacidad de demostrar qué validación y qué respuesta originaron el estado mostrado, sin reemplazar la autoridad del legado.

## Decisión

Adoptar un **puerto de integración implementado por un adaptador del legado**, un **recibo mínimo de validación** y una **outbox transaccional**. Cada operación relevante tiene dos progresos relacionados pero distintos —el **reloj local PIA** y el **reloj oficial del legado**— que se muestran y auditan sin confundirlos.

1. La consulta de prerrequisitos se efectúa mediante el puerto antes de confirmar localmente una inscripción que depende de esa respuesta. La PIA guarda un recibo de evidencia acotado: identificador de la operación, sistema y tipo de consulta, instante, resultado, correlación y versión/identificador de respuesta si el legado lo proporciona. No se almacena historial académico completo como copia autoritativa.
2. Un recibo solo prueba qué respuesta procesó la PIA y cuándo; no convierte una respuesta antigua en verdad permanente ni reemplaza al legado. Si el legado no responde o no puede validar, no se crea un recibo exitoso ni se presenta la inscripción como confirmada. La solicitud queda con un estado explícito pendiente/no disponible o rechazada, según la regla institucional que se acuerde.
3. Al confirmar localmente una inscripción o cancelación, la PIA persiste en la misma transacción el cambio, la auditoría y un evento outbox con identificador estable. Se muestran dos ejes de estado: **PIA** (`recibida`, `validada`, `aceptada localmente`, `cancelada`) y **legado** (`pendiente de envío`, `acuse recibido`, `requiere conciliación`).
4. Un trabajador reclama eventos pendientes, los envía mediante el adaptador y actualiza el reloj del legado con el acuse o el error reintentable. Los eventos no se descartan silenciosamente; los fallos persistentes se hacen visibles para operación y conciliación.
5. Se reintentan entregas con espera progresiva y límites operativos configurables. El trabajador puede continuar después de un reinicio. La entrega puede repetirse: se usa un identificador estable y, si el legado lo soporta, una clave idempotente en destino. Si no lo soporta, se debe acordar una consulta de conciliación o un procedimiento operacional antes de afirmar ausencia de duplicados.
6. La pantalla muestra una línea de tiempo legible para el estudiante y el personal: solicitud recibida, validación oficial (si ocurrió), aceptación local y acuse del legado. El lenguaje de “confirmada” queda sujeto a la definición institucional de qué sistema y evento constituyen confirmación oficial.

No se incluye un broker como requisito inicial: PostgreSQL y el trabajador son suficientes para persistir y reclamar eventos, y evitan añadir un servicio que el equipo deba operar. El recibo guarda metadatos mínimos y no replica la autoridad académica. El protocolo, la semántica real de sincronización y la definición institucional de confirmación quedan sujetos a acuerdo con los responsables del legado.

## Consecuencias

### Positivas

- Las reglas de la PIA no dependen directamente del protocolo externo; el adaptador concentra traducción, timeouts y errores.
- Un evento que corresponde a una inscripción confirmada no se pierde por una caída del proceso entre el commit local y el envío.
- La PIA puede reanudar la sincronización después de la recuperación del legado y consultar operaciones pendientes.
- Un fallo de sincronización no mantiene abierta la transacción de matrícula ni bloquea cupos mientras se espera una respuesta remota.
- Los recibos de validación permiten auditar qué evidencia condujo a una decisión sin convertir una réplica local en fuente oficial.
- Los dos estados hacen visible la diferencia entre “aceptada por la PIA” y “recibida por el legado”, reduciendo confirmaciones ambiguas.
- La solución no requiere servicios comerciales adicionales ni un broker para la primera versión.

### Negativas

- La inscripción local y la sincronización con el legado no son una transacción atómica: puede existir un intervalo en que el evento esté pendiente.
- El trabajador, la outbox, las métricas, las alertas y los procedimientos de conciliación requieren implementación y operación.
- Si el legado carece de idempotencia y de consulta para conciliar, un timeout posterior a la recepción puede dejar incierto si un evento fue aplicado.
- La validación de prerrequisitos sigue dependiendo de una respuesta del legado y restringe las solicitudes que requieren dicha validación durante su interrupción.
- Guardar evidencia de validación implica definir retención, acceso y minimización de datos; se deben registrar metadatos y no una copia innecesaria del historial.
- Dos relojes de estado agregan conceptos a la interfaz y requieren textos institucionales consistentes para no confundir al estudiante.

### Riesgos y mitigaciones

- **Riesgo:** duplicación en el legado tras un timeout ambiguo. **Mitigación:** exigir una clave idempotente del lado destino o una operación de consulta/reconciliación; acordar el contrato antes de producción.
- **Riesgo:** crecimiento de eventos pendientes durante una interrupción de 40 minutos. **Mitigación:** dimensionar almacenamiento, alertar por antigüedad y cantidad de pendientes, y probar el drenaje de cola tras la recuperación.
- **Riesgo:** presentar un estado confuso al estudiante. **Mitigación:** definir estados y textos aprobados por la universidad que distingan solicitud, validación académica, confirmación local y sincronización oficial.
- **Riesgo:** interpretar el recibo como garantía de que la regla no cambió después de la consulta. **Mitigación:** registrar su instante y versión si el legado la ofrece; documentar que acredita una consulta, no una exclusión mutua ni una transacción con el sistema oficial.
- **Riesgo:** conservar datos personales innecesarios en recibos. **Mitigación:** almacenar resultado mínimo y referencias/correlación, con control de acceso y retención institucional; no replicar expediente académico completo.
- **Riesgo:** perder datos por una estrategia de respaldo insuficiente. **Mitigación:** respaldar PostgreSQL y verificar mediante ejercicios que se cumplan RTO ≤ 30 minutos y RPO ≤ 5 minutos; la outbox no sustituye los respaldos.

## Condiciones para revisar la decisión

- El sistema legado ofrece una API transaccional con semántica idempotente que permita sustituir el flujo outbox por otro contrato más simple sin perder recuperabilidad.
- El volumen o retraso de eventos excede, en mediciones de matrícula, la capacidad del trabajador y de PostgreSQL institucional.
- La universidad autoriza y puede operar un broker institucional, y se demuestra que aporta una ventaja necesaria sobre la outbox.
- Se modifica la regla institucional que determina cuándo una inscripción se considera confirmada o cómo operar durante interrupciones del legado.
- Los ejercicios de recuperación no cumplen RTO/RPO y un cambio de infraestructura o estrategia resulta necesario.
