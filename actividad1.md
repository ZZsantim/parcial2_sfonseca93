# Actividad 1. Identificación y priorización de drivers arquitectónicos

**Estudiante:** Santiago Andres Fonseca Rojas  
**Documento:** 1023890893
**Caso:** Plataforma de Inscripción Académica (PIA)  
**Escenario individual:** Escenario B — concurrencia

> El número de documento no fue suministrado. Completar ese dato antes de entregar el documento.

## 1. Objetivos de negocio

| ID | Objetivo | Resultado esperado |
|---|---|---|
| ON-01 | Permitir que los estudiantes consulten la oferta e inscriban o cancelen asignaturas durante las fechas establecidas. | Completar el proceso de matrícula de forma confiable, con el estado de cada solicitud visible para el estudiante. |
| ON-02 | Mantener la inscripción del período consistente con la capacidad autorizada y con la información académica oficial. | No confirmar cupos excedidos ni inscripciones que incumplan prerrequisitos; sincronizar las inscripciones confirmadas con el sistema académico legado. |
| ON-03 | Proteger y hacer auditables las operaciones académicas. | Poder determinar quién realizó cada cambio, cuándo ocurrió y cuál fue el resultado. |

## 2. Drivers priorizados

Las prioridades **P0**, **P1** y **P2** indican, respectivamente, condiciones críticas para la validez de la matrícula, objetivos operativos de alto impacto y objetivos necesarios para la evolución sostenible. Los criterios de aceptación se deberán verificar mediante pruebas y métricas observables, no solo por inspección del diseño.

| ID | Tipo | Descripción | Origen | Prioridad | Medida de aceptación | Justificación |
|---|---|---|---|---|---|---|
| D-01 | Atributo de calidad: consistencia | Evitar que las inscripciones confirmadas superen la capacidad de un grupo, incluso cuando muchas solicitudes compitan por el último cupo. | C04; escenario B | P0 | En pruebas concurrentes de inscripción, para todo grupo se cumple `inscripciones confirmadas ≤ capacidad autorizada`; no se confirma ninguna operación que exceda el cupo. | Una sobreinscripción invalida la oferta académica y afecta a los estudiantes. |
| D-02 | Atributo de calidad: idempotencia | Procesar los reintentos de una misma solicitud sin crear inscripciones duplicadas ni consumir cupos repetidamente. | Problema de duplicados; escenario B | P0 | Al enviar hasta cinco veces la misma solicitud con la misma clave de idempotencia, se registra como máximo una inscripción y un único efecto sobre el cupo; los reintentos reciben el resultado asociado a la operación original. | El escenario exige gestionar reintentos automáticos; el mecanismo debe distinguir una repetición de una nueva intención de inscripción. |
| D-03 | Atributo de calidad: rendimiento | Responder consultas de oferta académica durante la carga de matrícula. | C01, C02; escenario B | P1 | En una prueba representativa con 1.800 estudiantes concurrentes (1.200 más 50%), el percentil 95 de las consultas de asignaturas responde en ≤ 2 segundos. Informar también tasa de errores y duración de la prueba. | Una respuesta lenta puede provocar más reintentos y agravar la contención durante matrícula. |
| D-04 | Atributo de calidad: disponibilidad y recuperabilidad | Mantener el servicio disponible durante matrícula y recuperar el servicio ante fallas sin perder operaciones confirmadas más allá de los objetivos institucionales. | C03, C06, C07 | P1 | Disponibilidad mensual ≥ 99,9% durante los períodos de matrícula; ejercicios de recuperación demuestran RTO ≤ 30 minutos y RPO ≤ 5 minutos. | La indisponibilidad en una ventana acotada afecta directamente la posibilidad de matricularse. |
| D-05 | Atributo de calidad: interoperabilidad y corrección académica | Consultar la fuente oficial para validar prerrequisitos y sincronizar las inscripciones confirmadas con el sistema legado, sin asumir que una interrupción de este confirma una inscripción. | Requisito funcional 6; condición adicional; C05 | P0 | Cuando el legado no responde, ninguna solicitud que requiera una validación no disponible se muestra como confirmada; cada operación queda identificada para reintento/reconciliación y no genera duplicados tras recuperar la conexión. | El legado sigue siendo fuente oficial de historial y prerrequisitos, y puede fallar durante hasta 40 minutos. |
| D-06 | Atributo de calidad: seguridad y auditabilidad | Restringir el acceso a datos académicos y conservar trazabilidad de las operaciones de inscripción y cancelación. | Requisito funcional 5; C09 | P1 | El 100% de las operaciones de inscripción, cancelación y sincronización registra actor o identidad de servicio, fecha/hora, identificador de operación, acción y resultado; las pruebas de autorización impiden acceso no permitido a datos de otro estudiante. | La plataforma procesa datos personales y decisiones académicas que deben poder auditarse. |
| D-07 | Atributo de calidad: modificabilidad y mantenibilidad | Permitir cambios en reglas académicas con bajo riesgo y dentro de la capacidad del equipo institucional. | Problema de alto esfuerzo para nuevas reglas; C10 | P2 | Cada regla priorizada tiene una responsabilidad y pruebas automatizadas identificables; una modificación de regla no requiere cambiar componentes ajenos a esa responsabilidad y pasa la suite de pruebas en CI. | El equipo de mantenimiento es pequeño: cuatro desarrolladores y un administrador de infraestructura. |

## 3. Restricciones arquitectónicas

| ID | Restricción | Origen | Implicación para la arquitectura |
|---|---|---|---|
| R-01 | La universidad dispone de infraestructura Linux y PostgreSQL, y no autoriza inicialmente contratar servicios comerciales adicionales. | C08 | La propuesta debe operar en infraestructura institucional y no depender de servicios comerciales externos para cumplir sus objetivos esenciales. |
| R-02 | El sistema académico legado seguirá siendo la fuente oficial del historial académico y los prerrequisitos. | Condición adicional | La PIA no puede sustituir ni tratar como autoritativa una copia local de esos datos; debe definir explícitamente validación, indisponibilidad y sincronización. |
| R-03 | Ningún grupo puede superar su capacidad autorizada. | C04 | La aceptación de inscripciones debe preservar este invariante ante concurrencia y fallas; no basta con validar disponibilidad únicamente en la interfaz. |
| R-04 | Las metas operativas incluyen hasta 1.200 estudiantes simultáneos, consultas con p95 ≤ 2 s, disponibilidad mensual ≥ 99,9%, RTO ≤ 30 min y RPO ≤ 5 min. | C01–C03, C06–C07 | Estas metas deben orientar el diseño y verificarse en pruebas operativas, de carga y recuperación. El escenario individual eleva la prueba de concurrencia a 1.800 estudiantes. |
| R-05 | El equipo institucional de operación y mantenimiento consta de cuatro desarrolladores y un administrador de infraestructura. | C10 | La solución y su operación deben ser comprensibles y sostenibles para un equipo reducido; la complejidad operativa requiere justificación. |

## 4. Escenarios de calidad verificables

| ID | Driver | Estímulo | Contexto | Respuesta esperada | Medida cuantificable |
|---|---|---|---|---|---|
| EQ-01 | Consistencia de cupos (D-01) | Varios estudiantes solicitan simultáneamente el último cupo de un grupo. | Período de matrícula, con solicitudes concurrentes sobre el mismo grupo. | El sistema confirma como máximo una solicitud por cada cupo restante; las demás reciben un estado no confirmado y no alteran el total confirmado. | En pruebas repetibles, `confirmadas ≤ capacidad`; cero sobreinscripciones en todos los grupos probados. |
| EQ-02 | Idempotencia (D-02) | Un cliente reenvía hasta cinco veces la misma solicitud por timeout o reintento automático. | Carga de matrícula; mismo estudiante, grupo e identificador de idempotencia. | El sistema devuelve el resultado de la operación original sin duplicar la inscripción ni descontar cupo adicional. | Cinco envíos idénticos producen como máximo un registro de inscripción y un único efecto sobre la capacidad. |
| EQ-03 | Rendimiento (D-03) | Estudiantes consultan asignaturas y grupos disponibles. | Prueba de carga con 1.800 estudiantes concurrentes, representando el aumento del 50% sobre C01. | El sistema sirve las consultas sin degradación que impida completar la navegación normal; registra latencia y errores. | p95 de consulta ≤ 2 segundos; resultados de latencia y tasa de errores reportados para la duración y perfil definidos de la prueba. |
| EQ-04 | Fallo del legado (D-05) | El sistema académico legado deja de responder durante 40 minutos en período de matrícula. | PIA operativa; el legado continúa siendo la fuente oficial de prerrequisitos e historial. | La PIA informa que la validación/sincronización está pendiente o no disponible y no presenta como confirmada una inscripción que depende de una validación oficial no realizada. Al volver el legado, procesa y concilia operaciones pendientes de forma controlada, sin duplicarlas. | Simulación de 40 minutos: cero confirmaciones basadas en una validación no disponible; cada operación pendiente tiene estado e identificador consultables; reconciliación sin duplicados tras la recuperación. |
| EQ-05 | Recuperabilidad (D-04) | Se interrumpe el servicio o se pierde el nodo activo. | Operación de matrícula con respaldos y procedimientos de recuperación disponibles. | El servicio se restaura con datos consistentes y pérdida dentro del objetivo acordado. | Ejercicio cronometrado: RTO ≤ 30 minutos y RPO ≤ 5 minutos. |
| EQ-06 | Seguridad y auditoría (D-06) | Un usuario intenta consultar o modificar información de otro estudiante; o se ejecuta una operación académica válida. | Uso normal y pruebas de autorización. | La operación no autorizada se deniega; la operación válida queda trazada con actor, acción, momento, resultado e identificador de correlación. | Cero accesos cruzados exitosos en pruebas de autorización; 100% de las operaciones objetivo con registro de auditoría completo. |

## 5. Conflictos entre drivers y resolución propuesta

### 5.1 Disponibilidad de matrícula frente a validación oficial del legado

**Conflicto:** mantener la matrícula disponible durante una interrupción de hasta 40 minutos (C05) puede llevar a aceptar inscripciones sin poder comprobar prerrequisitos ni registrar su estado en el sistema oficial. La disponibilidad no debe producir una confirmación académica falsa.

**Resolución propuesta:** separar explícitamente los estados de solicitud, validación, confirmación y sincronización. Si no se puede realizar una validación oficial requerida, comunicar que la solicitud está pendiente o no disponible y no presentarla como inscripción confirmada. Al recuperarse el legado, reintentar y reconciliar con identificadores idempotentes. La política final de qué solicitudes pueden quedar en espera debe detallarse en las siguientes actividades; ninguna espera equivale a confirmación.

### 5.2 Rendimiento bajo concurrencia frente a consistencia de cupos

**Conflicto:** reducir la latencia y permitir 1.800 estudiantes concurrentes puede incentivar validaciones rápidas basadas en una lectura de disponibilidad que quede desactualizada antes de confirmar la inscripción. Eso podría exceder el cupo.

**Resolución propuesta:** mantener la consulta de oferta optimizada para lectura, pero hacer que la operación que confirma una inscripción aplique una validación atómica de capacidad en el límite que persiste la inscripción. El objetivo de p95 ≤ 2 segundos se verifica para consultas y se mide por separado para las escrituras; nunca se sacrifica el invariante de capacidad para cumplir una meta de latencia.

### 5.3 Reintentos transparentes frente a evitar duplicados

**Conflicto:** los reintentos hasta cinco veces ayudan a recuperarse de demoras y fallas transitorias, pero pueden crear múltiples inscripciones si cada envío se interpreta como una operación nueva.

**Resolución propuesta:** asociar cada intento repetido de la misma operación a una clave de idempotencia estable y persistir el resultado asociado. Una nueva intención explícita del estudiante debe tener un identificador de operación distinto y someterse a las mismas reglas académicas y de cupo.

## 6. Síntesis de priorización

1. **P0 — Corrección académica:** D-01, D-02 y D-05. Sin cupos consistentes, protección frente a duplicados y respeto por la fuente oficial, la matrícula no es confiable.
2. **P1 — Continuidad y experiencia operativa:** D-03, D-04 y D-06. Rendimiento, disponibilidad, recuperación, protección y trazabilidad son necesarios para operar durante la ventana de matrícula y proteger los datos.
3. **P2 — Evolución sostenible:** D-07. La mantenibilidad reduce el costo y el riesgo de incorporar nuevas reglas, dadas las restricciones del equipo.

El Escenario B modifica la validación de capacidad: las pruebas de rendimiento y consistencia deben contemplar **1.800 estudiantes concurrentes** y hasta **cinco envíos de la misma solicitud**, no solo la carga base de C01. Estos valores son criterios del escenario de evaluación y no una estimación de capacidad máxima de producción.
