# Actividad 4. Selección y validación de tácticas arquitectónicas

**Estudiante:** Santiago Andres Fonseca Rojas  
**Documento:** 1023890893  
**Caso:** Plataforma de Inscripción Académica (PIA)  
**Escenario individual:** Escenario B — concurrencia

## 1. Propósito y criterio de diseño

La propuesta no trata las tácticas como una lista de patrones independientes. Las organiza alrededor de una propiedad del sistema: **cada decisión de matrícula debe poder explicarse con evidencia y sin confundir disponibilidad, aceptación local y confirmación oficial**.

Una inscripción pasa por tres límites distintos:

1. La PIA recibe una intención identificable.
2. La PIA valida reglas y acepta o rechaza el cambio local de manera atómica.
3. El sistema académico legado recibe y reconoce la sincronización, cuando corresponda.

Esta separación atiende los drivers D-01 a D-06 de la Actividad 1 y concreta el monolito modular, la transacción local y la integración mediante puerto/adaptador de la Actividad 2. Las tácticas descritas son decisiones propuestas; no se afirma que su efectividad ya haya sido probada. Los mecanismos de esta actividad definen cómo demostrarla.

## 2. Táctica T-01 — Admisión de cupo en un único punto de decisión

| Aspecto | Descripción |
|---|---|
| **Atributos de calidad** | Consistencia, integridad y auditabilidad. |
| **Drivers relacionados** | D-01, D-02; escenario B; C04. |
| **Problema específico** | Concurrencia por el último cupo y repetición de la misma solicitud hasta cinco veces. Leer “hay cupo” y escribir después no protege el límite: varias solicitudes podrían observar el mismo espacio. |
| **Componentes** | Caso de uso de inscripción, servicio de cupos, repositorio de idempotencia e inscripciones, auditoría y PostgreSQL. |
| **Funcionamiento técnico** | Cada comando incluye una clave de idempotencia estable y una huella del contenido solicitado. PostgreSQL es el punto único de admisión: en una transacción, el sistema reclama el cupo con una actualización condicional equivalente a `UPDATE grupo SET enrolled_count = enrolled_count + 1 WHERE id = :grupo AND enrolled_count < capacity RETURNING enrolled_count`. Si no se actualiza ninguna fila, no queda cupo. En la misma transacción se registra el resultado de idempotencia, la inscripción, su auditoría y el evento de sincronización. Restricciones únicas impiden repetir la misma clave o crear dos inscripciones activas para el mismo estudiante, grupo y período. Si hay error, todo se revierte. Una repetición con la misma clave y contenido recibe el resultado original; la misma clave con contenido diferente se rechaza como conflicto. |
| **Beneficios** | El invariante de capacidad se decide donde se serializan las escrituras; no depende de la pantalla ni de una lectura previa. Los reintentos no producen una nueva inscripción ni vuelven a consumir capacidad. El resultado confirmado puede recuperarse después de un timeout del cliente. |
| **Costos y límites** | Las transacciones sobre el mismo grupo compiten por la fila y pueden aumentar la latencia. La tabla de idempotencia necesita una política de retención. La garantía es local a PostgreSQL: no vuelve atómica la comunicación con el legado. |
| **Verificación** | Ejecutar pruebas con múltiples estudiantes compitiendo por uno o varios cupos y repetir cada solicitud hasta cinco veces usando la misma clave. Criterios: `inscripciones activas ≤ capacidad` para cada grupo; cero duplicados por clave o por estudiante/grupo/período; como máximo un incremento de cupo por comando idempotente; conflictos y errores no dejan actualizaciones parciales. Reportar latencia de escritura y contención, sin sacrificar el invariante para mejorar la métrica. |

### Secuencia de decisión

```text
solicitud + clave idempotente
        |
        v
¿ya existe el resultado? -- sí --> devolver el resultado original
        |
        no
        v
transacción PostgreSQL:
  admitir solo si enrolled_count < capacity
  guardar inscripción + auditoría + outbox + resultado idempotente
        |
        +-- falla cualquier paso --> rollback completo
        |
        +-- commit --> devolver resultado local persistido
```

La consulta de oferta puede mostrar disponibilidad estimada, pero no realiza la admisión ni promete que el cupo seguirá libre al enviar el comando.

## 3. Táctica T-02 — Pasaporte de decisión con evidencia mínima

| Aspecto | Descripción |
|---|---|
| **Atributos de calidad** | Auditabilidad, seguridad, idempotencia y capacidad de diagnóstico. |
| **Drivers relacionados** | D-02 y D-06; requisito funcional 5 y C09. |
| **Problema específico** | Cuando un usuario reintenta, el legado falla o surge una discrepancia, un estado final aislado no explica qué solicitud se procesó, qué regla se evaluó ni por qué la PIA aceptó o rechazó la operación. Un log de texto o un registro que replique datos académicos introduce riesgos sin garantizar la trazabilidad del negocio. |
| **Componentes** | API de inscripción, caso de uso, validador de reglas, repositorios de idempotencia e inscripciones, auditoría, puerto del legado y outbox. |
| **Funcionamiento técnico** | La PIA asigna un identificador de operación y genera un **pasaporte de decisión** estructurado que enlaza: identidad autorizada del actor, huella del comando, clave idempotente protegida, versión de las reglas locales, referencia y hora de la respuesta oficial si hubo validación, decisión de cupo, resultado de la operación, estado del evento de sincronización e identificador de correlación. El pasaporte es un registro auditable de la decisión y sus referencias; no contiene una copia del expediente académico ni declara que una respuesta antigua siga siendo válida. Los datos mínimos del pasaporte y la decisión local se persisten en la misma transacción del cambio de negocio. Los intentos fallidos antes de esa transacción registran un resultado de solicitud separado sin aparentar un commit. |
| **Beneficios** | Permite explicar una decisión concreta sin reconstruirla a partir de logs distribuidos. Facilita responder consultas de estudiantes y auditorías, relaciona los reintentos con su operación original y ayuda a detectar si la falla ocurrió antes o después de la aceptación local. |
| **Costos y límites** | Añade un esquema de auditoría que debe versionarse, protegerse y retenerse. La evidencia indica qué datos y reglas usó la PIA, pero no constituye una firma del legado ni una transacción conjunta. Guardar contenido personal innecesario puede infringir el principio de minimización. |
| **Verificación** | Para una muestra automatizada de inscripciones, cancelaciones, reintentos, conflictos de cupo y fallos de legado, comprobar que el 100% de las decisiones locales tiene identificador de operación, actor/servicio, marca temporal, versión de regla, resultado y correlación. Verificar que los cinco reintentos apuntan a un solo resultado; que un auditor autorizado puede reconstruir la cronología; que pruebas de autorización impiden a un estudiante consultar pasaportes ajenos; y que el registro no incluye historial académico completo, contraseñas ni secretos. |

### Forma conceptual del pasaporte

```text
operacion: <identificador>
actor: <identidad autorizada>
comando: <huella del contenido, no copia innecesaria>
reglas: <versión evaluada>
validacion_oficial: <referencia + instante + resultado, si ocurrió>
decision_local: <aceptada | rechazada + motivo>
cupos: <grupo + resultado de admisión>
legado: <pendiente | acuse | requiere conciliación>
correlacion: <identificador para seguir la operación>
```

Este esquema es ilustrativo; los campos finales, la retención y los permisos deben aprobarse con la universidad.

## 4. Táctica T-03 — Dos relojes y compuerta de validación del legado

| Aspecto | Descripción |
|---|---|
| **Atributos de calidad** | Corrección académica, interoperabilidad, disponibilidad degradada y transparencia del estado. |
| **Drivers relacionados** | D-04, D-05 y D-06; C05; condición de autoridad del legado. |
| **Problema específico** | Durante una interrupción de hasta 40 minutos, ni la disponibilidad mostrada por la PIA ni una solicitud enviada al legado prueban que un prerrequisito fue validado o que la inscripción fue recibida oficialmente. Confirmar por anticipado puede crear una matrícula inválida; bloquear una transacción mientras se espera al legado consume recursos de base de datos. |
| **Componentes** | Validador de reglas, puerto/adaptador del legado, repositorio del pasaporte, caso de uso de inscripción, outbox, trabajador y presentación del estado. |
| **Funcionamiento técnico** | Antes de aceptar localmente una inscripción que requiere prerrequisitos, la PIA consulta al legado fuera de la transacción que reclama el cupo. Si obtiene una respuesta, conserva un recibo mínimo con referencia, instante y resultado. Si el legado no responde, no se registra una validación exitosa, no se reserva cupo y la solicitud se muestra como `pendiente de validación` o `no procesable`, según la política institucional. Tras la recuperación se vuelve a consultar al legado y se reevalúa la capacidad actual; no se promete mantener un cupo durante la interrupción. Después del commit local, el evento outbox avanza por el reloj de sincronización del legado independientemente del estado local de la PIA. La interfaz distingue los estados de ambos relojes y el contrato universitario define cuál equivale a confirmación oficial. |
| **Beneficios** | Evita falsear la validación oficial, no mantiene bloqueos SQL durante llamadas de red y hace visible cuándo una operación está pendiente de cada sistema. Una indisponibilidad no convierte una espera en una confirmación implícita. |
| **Costos y límites** | Un estudiante puede perder disponibilidad de un cupo mientras espera y tendrá que volver a intentar la solicitud después de la recuperación. La operación depende de la latencia y contrato del legado. La evidencia registrada no evita que el historial cambie después de la consulta. El estado de dos sistemas requiere textos claros y conciliación operativa. |
| **Verificación** | Simular una caída del legado de 40 minutos y luego su recuperación. Criterios: cero inscripciones confirmadas apoyadas en una validación ausente; cero transacciones de cupo abiertas mientras se espera una llamada remota; cada solicitud afectada expone su estado e identificador; al recuperarse, la PIA revalida antes de reclamar cupo y procesa el backlog sin pérdida de eventos locales; cada operación muestra por separado estado PIA y acuse del legado. Medir antigüedad del evento pendiente hasta que vuelva la conexión. |

### Máquina de estados propuesta

| Estado PIA | Significado | ¿Hay plaza/cupo reservado? | Estado del legado |
|---|---|---:|---|
| `RECIBIDA` | La PIA conoce la intención y su identificador. | No | No consultado |
| `PENDIENTE_VALIDACION` | El legado no respondió o la validación no concluyó. | No | No disponible / sin respuesta |
| `RECHAZADA_REGLA` | Una regla oficial o local rechazó la solicitud. | No | Validación negativa o regla local negativa |
| `ACEPTADA_LOCAL` | La validación requerida tuvo éxito y la transacción local confirmó la inscripción. | Sí | Pendiente de sincronización o acuse recibido |
| `CANCELADA_LOCAL` | La cancelación local fue confirmada y liberó el cupo una sola vez. | No | Pendiente de sincronización o acuse recibido |
| `REQUIERE_CONCILIACION` | El resultado remoto es ambiguo y requiere consulta o acción operativa. | Según el estado local registrado | Acuse incierto / discrepancia |

Los nombres expuestos al estudiante deben ser acordados institucionalmente. `ACEPTADA_LOCAL` no debe mostrarse como inscripción oficial si la universidad define que la confirmación depende del acuse del legado.

## 5. Táctica T-04 — Presupuestos de trabajo para proteger el camino crítico

| Aspecto | Descripción |
|---|---|
| **Atributos de calidad** | Rendimiento, disponibilidad y degradación controlada. |
| **Drivers relacionados** | D-03 y D-04; escenario B; C01, C02 y C03. |
| **Problema específico** | El aumento de concurrencia y una dependencia externa lenta pueden consumir hilos o conexiones compartidos. La saturación de consultas o de reintentos al legado no debe impedir que comandos válidos se procesen con el control de consistencia requerido. |
| **Componentes** | API de consultas, API/casos de uso de comandos, cliente del legado, trabajador outbox, configuración de concurrencia de la aplicación y PostgreSQL. |
| **Funcionamiento técnico** | Separar lógicamente presupuestos de ejecución para lecturas, comandos y llamadas remotas: las consultas tienen un límite de concurrencia y timeouts; los comandos preservan capacidad de ejecución y usan transacciones cortas; las llamadas al legado tienen timeout y límite de concurrencia independiente; el trabajador reclama lotes acotados del outbox. Las capacidades se configuran a partir de pruebas en la infraestructura institucional y del límite de conexiones de PostgreSQL, no con cifras inventadas. Cuando una clase alcanza su límite, el sistema falla rápido con un estado explícito y, para comandos repetibles, instruye reintentar con la misma clave idempotente. No se usan consultas cacheadas para decidir prerrequisitos ni confirmar cupos. |
| **Beneficios** | Evita que una dependencia lenta retenga recursos ilimitadamente; hace visible qué clase de trabajo se satura; conserva el invariante de cupos aun si el sistema debe rechazar o aplazar carga. |
| **Costos y límites** | El ajuste requiere mediciones y puede causar respuestas de reintento en picos. Los presupuestos no agregan capacidad física ni prueban por sí solos el objetivo de disponibilidad. Una configuración incorrecta puede dejar recursos ociosos o saturar PostgreSQL. |
| **Verificación** | Ejecutar carga mixta representativa con 1.800 estudiantes concurrentes y medir por separado p95 de consultas, latencia de comandos, rechazos, timeouts, conexiones PostgreSQL, errores y edad del backlog. Aceptación: p95 de consultas de asignaturas ≤ 2 segundos; ninguna sobreinscripción ni duplicado; al inyectar demora/falla del legado, los recursos de espera permanecen acotados y comandos/consultas devuelven estados explícitos. Registrar el perfil de carga, duración y recursos para que la prueba sea reproducible. |

## 6. Relación entre tácticas y criterios de aceptación

| Táctica | Evidencia que produce la prueba | Driver satisfecho |
|---|---|---|
| T-01 Admisión de cupo en un único punto | Sin sobreinscripciones ni efectos dobles tras reintentos, con datos consultables por operación. | D-01, D-02 |
| T-02 Pasaporte de decisión | Cronología reconstruible, accesible solo a roles autorizados y con minimización de datos. | D-02, D-06 |
| T-03 Dos relojes y compuerta | Ninguna confirmación basada en validación ausente; solicitudes pendientes reconciliables después de los 40 minutos. | D-04, D-05 |
| T-04 Presupuestos de trabajo | p95 de consultas dentro del objetivo sin sacrificar los invariantes de comandos. | D-01, D-03, D-04 |

## 7. Plan de validación y límites de la propuesta

1. **Prueba de concurrencia y reintentos:** competir por los últimos cupos con solicitudes únicas y repetir cada comando hasta cinco veces. Comprobar capacidad, unicidad y resultados persistidos.
2. **Prueba de legado desconectado:** simular hasta 40 minutos sin respuesta, luego restaurarla. Comprobar que no se confirme una validación ausente, que la solicitud pendiente tenga identificador y que se vuelva a validar antes de reservar.
3. **Prueba de carga mixta:** simular 1.800 estudiantes concurrentes con consultas y comandos; medir p95 de consulta, latencia de comando, errores, conexiones y backlog.
4. **Prueba de autorización y auditoría:** intentar acceder a pasaportes de otro estudiante y verificar registros completos de las operaciones autorizadas sin replicar historiales académicos completos.
5. **Prueba de recuperación:** respaldar y restaurar PostgreSQL para comprobar RTO ≤ 30 minutos y RPO ≤ 5 minutos. Ninguna táctica descrita sustituye el respaldo ni demuestra estos objetivos sin ejecutar el ejercicio.

La universidad debe acordar con el responsable del legado el protocolo, el significado de sus acuses, la semántica de deduplicación/conciliación y qué estado cuenta como confirmación oficial. Hasta validar ese contrato y ejecutar las pruebas, las tácticas son una propuesta arquitectónica, no resultados medidos ni garantías de producción.
