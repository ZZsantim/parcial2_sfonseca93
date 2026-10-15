# ADR-002: Cada cupo es una plaza numerada reclamable

| Campo | Valor |
|---|---|
| **Identificador** | ADR-002 |
| **Título** | Representar la capacidad como plazas reclamables con idempotencia persistida |
| **Fecha** | 2026-10-14 |
| **Estado** | Aceptada para la propuesta |

## Contexto y problema

En matrícula, varios estudiantes pueden competir por los últimos cupos y un cliente puede enviar hasta cinco veces la misma solicitud. Una validación previa de disponibilidad seguida de una escritura posterior es vulnerable a carreras: múltiples solicitudes podrían observar el mismo último cupo y exceder la capacidad. Si cada reintento se procesa como una operación nueva, puede duplicarse la inscripción o descontarse varias veces la capacidad.

El sistema debe garantizar el invariante `inscripciones activas ≤ capacidad autorizada` aun con solicitudes concurrentes. También debe reconocer reintentos de una misma intención y devolver un resultado coherente. La solución se apoya en PostgreSQL institucional y no presupone un broker ni un servicio externo.

## Driver arquitectónico relacionado

- **D-01 (P0):** consistencia de cupos bajo concurrencia.
- **D-02 (P0):** idempotencia de solicitudes repetidas.
- **Escenario B:** 1.800 estudiantes concurrentes y hasta cinco envíos de la misma solicitud.
- **R-03:** ningún grupo puede superar su capacidad autorizada.

## Alternativas consideradas

1. **Plazas numeradas como filas reclamables (seleccionada):** crear una fila por cada cupo autorizado; una inscripción reclama una plaza libre, con idempotencia y efectos locales en una transacción.
2. **Contador agregado con incremento condicional:** mantener un único contador por grupo y aumentar el valor solo si todavía es menor que la capacidad.
3. **Lectura de disponibilidad en la aplicación y actualización posterior sin condición atómica:** comprobar cupos antes de insertar; el cliente reintenta si recibe un error o timeout.
4. **Servicio distribuido de inventario o broker:** delegar la capacidad y coordinar la inscripción mediante mensajes entre componentes independientes.

## Criterios de comparación

- Garantía del límite de capacidad ante solicitudes simultáneas.
- Prevención de duplicados si los reintentos llegan antes, durante o después del primer resultado.
- Complejidad de operación bajo las restricciones de Linux/PostgreSQL y el equipo institucional.
- Capacidad de recuperar un resultado cuando el cliente no recibió respuesta, aunque la primera operación se haya confirmado.
- Alcance de las garantías frente a fallos del sistema legado.
- Facilidad para explicar y conciliar cada unidad de capacidad, sin depender de que un contador agregado coincida con las inscripciones.

## Decisión

Modelar cada cupo autorizado de un grupo como una **plaza identificable** persistida en PostgreSQL, con estado `LIBRE` u `OCUPADA` y relación con la inscripción que la ocupa. No se genera un asiento por estudiante en tiempo de solicitud: las filas de plazas se crean al configurar la capacidad autorizada del grupo.

1. Cada comando recibe una clave de idempotencia estable, asociada al estudiante y a la intención de inscripción. Se conserva el resultado de la operación.
2. Si llega la misma clave con el mismo contenido, se devuelve el resultado ya registrado sin repetir sus efectos. Si se reutiliza con un contenido distinto, se rechaza como conflicto.
3. Dentro de una transacción, la PIA valida las reglas locales y selecciona una plaza libre del grupo con bloqueo de fila (`SELECT ... FOR UPDATE`), sin `SKIP LOCKED`, y la cambia a `OCUPADA`. Si otra solicitud está reclamando una plaza, la operación espera el resultado de ese bloqueo y luego vuelve a evaluar las plazas libres; así no confunde temporalmente una plaza bloqueada con un grupo lleno. Si no queda ninguna plaza libre, no hay cupo y no se confirma la operación.
4. En la misma transacción se persisten la inscripción asociada a la plaza, el resultado idempotente, la auditoría y el evento outbox. Si falla cualquier paso, se revierte todo y la plaza vuelve a estar libre.
5. Restricciones únicas sobre la clave idempotente y sobre una inscripción activa por estudiante/grupo/período protegen contra carreras. La cancelación cambia el estado de la inscripción y devuelve su plaza a `LIBRE` en una sola transacción; una repetición de la cancelación no libera la plaza dos veces.
6. El número de plazas del grupo representa su capacidad autorizada. Un aumento de capacidad agrega plazas autorizadas; una reducción solo puede retirar plazas libres y nunca puede dejar la capacidad por debajo de las plazas ocupadas.

La disponibilidad mostrada en una consulta es informativa, no una reserva. La inscripción solo se considera aceptada por la PIA después del commit exitoso. Esta garantía cubre el estado local de la PIA; no convierte la transacción con PostgreSQL y la llamada al sistema legado en una transacción distribuida.

## Consecuencias

### Positivas

- Cada plaza tiene una identidad persistida y puede estar asociada a como máximo una inscripción activa; dos transacciones concurrentes no pueden reclamar la misma fila bloqueada, y una espera por el último cupo termina en una decisión consistente.
- Una clave idempotente impide aplicar varias veces el mismo comando y permite devolver el resultado original tras un timeout del cliente.
- Cupo, inscripción, auditoría y evento de salida mantienen atomicidad local; no queda una inscripción confirmada sin su evento durable de sincronización.
- La conciliación puede comparar plazas ocupadas con inscripciones activas y localizar la diferencia por grupo, en vez de investigar únicamente una discrepancia de contador.
- Las restricciones de base de datos protegen los invariantes incluso si una ruta de aplicación no ejecuta una validación esperada.

### Negativas

- Se requieren tantas filas de plazas como cupos autorizados haya; los grupos de gran capacidad consumen más filas que el modelo de contador.
- Las solicitudes del mismo grupo compiten por plazas libres; una fila bloqueada puede hacer que otras solicitudes esperen y elevar la latencia del último cupo.
- La tabla de idempotencia requiere una política de retención, protección de datos y consulta eficiente.
- La implementación debe manejar conflictos de unicidad y rollback sin transformarlos en respuestas de éxito.
- La consulta remota de prerrequisitos ocurre antes de la transacción local para no retener bloqueos durante una llamada de red; el estado del legado podría cambiar entre esa consulta y el commit.

### Riesgos y mitigaciones

- **Riesgo:** plazas duplicadas, faltantes o configuración de capacidad errónea. **Mitigación:** crear/retirar plazas mediante un caso de uso administrativo auditado, y comprobar periódicamente `plazas totales = capacidad autorizada` y `plazas ocupadas = inscripciones activas`.
- **Riesgo:** contención o recorrido costoso de plazas bajo carga. **Mitigación:** índice por grupo y estado, reclamación de lotes solo si una medición lo justifica y prueba de latencia de escrituras bajo 1.800 usuarios concurrentes; no omitir filas bloqueadas devolviendo una falsa respuesta de “sin cupo”.
- **Riesgo:** clave idempotente reutilizada incorrectamente o retención insuficiente. **Mitigación:** validar su alcance, almacenar una huella del comando, rechazar contenido distinto con la misma clave y definir retención acorde a la ventana de reintentos y conciliación.
- **Riesgo:** cambios de prerrequisitos entre validación y commit. **Mitigación:** acordar con el responsable del legado la vigencia/versionado de la respuesta y un procedimiento de conciliación; la PIA no afirma atomicidad distribuida que el legado no proporcione.

## Condiciones para revisar la decisión

- Pruebas de carga muestran que la contención de PostgreSQL incumple objetivos medidos incluso después de ajustar índices, consultas y capacidad institucional.
- Se demuestra una necesidad de distribución que no pueda resolverse con transacciones y escalado de la base existente.
- El sistema legado ofrece una operación transaccional/idempotente que cambie de forma material el límite de consistencia entre sistemas.
- La universidad cambia las reglas de negocio sobre reservas temporales, reingreso tras cancelación o alcance temporal de las claves idempotentes.
- Pruebas con la distribución real de capacidades muestran que el costo de mantener una fila por plaza supera la facilidad de auditoría y que un contador condicional ofrece el mismo nivel de evidencia y consistencia con menor costo.
