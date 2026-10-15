# Actividad 5. Gobernanza de arquitectura

- **Estudiante:** Santiago Andres Fonseca Rojas
- **Documento:** 1023890893
**Caso:** Plataforma de Inscripción Académica (PIA)  
**Rama de trabajo:** `dev_sfonseca93`

## 1. Propósito y alcance

La gobernanza busca que la arquitectura evolucione sin perder los invariantes de matrícula ni crear un proceso de aprobación imposible para el equipo institucional de cuatro desarrolladores y un administrador de infraestructura. Cada regla de esta propuesta tiene un responsable, una evidencia y un mecanismo de control.

La propuesta actual no incluye código de la PIA, lenguaje, framework ni plataforma de CI definidos. Por tanto, los controles automáticos aquí descritos son **requisitos para configurar el repositorio cuando se incorpore el código**, no controles que ya estén instalados o ejecutándose.

## 2. Políticas arquitectónicas verificables

| ID | Política | Responsable | Mecanismo de verificación | Criterio de cumplimiento |
|---|---|---|---|---|
| PA-01 | **La admisión de cupos es propiedad exclusiva del módulo de inscripciones.** Ningún controlador, interfaz, integración o módulo diferente puede modificar directamente la capacidad, crear/eliminar inscripciones o cambiar el estado de una inscripción activa. La decisión de cupo e inscripción debe ser atómica en PostgreSQL, protegida por restricciones persistentes e idempotente. | Responsable técnico de arquitectura; responsable del módulo de inscripciones implementa. | Control automático de dependencias del módulo y pruebas de integración concurrentes. Revisión de cambios en esquema y migraciones. | CI falla si se detecta una dependencia prohibida o si una prueba de concurrencia produce `inscripciones activas > capacidad`, duplicados ante cinco reintentos iguales o efectos parciales ante rollback. |
| PA-02 | **Ninguna operación que dependa de prerrequisitos puede ser confirmada sin una respuesta válida del sistema legado.** Los estados locales y del legado se deben distinguir. Una llamada remota no puede mantener abierta la transacción que reserva cupo. La sincronización posterior debe ser rastreable y recuperable. | Responsable de integración; responsable técnico de arquitectura; responsable académico valida semántica de confirmación. | Pruebas de contrato del adaptador, pruebas con timeout/fallo simulado y revisión de transacciones. | CI falla si una prueba demuestra confirmación apoyada en una respuesta ausente, una transacción de cupo abierta durante la llamada remota o una operación sin estado/identificador de correlación. |
| PA-03 | **Todo cambio de arquitectura o regla académica que afecte a inscripción debe tener ADR y pruebas acordes con el riesgo.** El cambio no puede depender de una decisión implícita en el código o en un diagrama desactualizado. | Autor del cambio prepara el ADR; responsable técnico de arquitectura decide; responsable académico aprueba cambios de comportamiento académico. | Plantilla y revisión de ADR; matriz de requisitos/pruebas en el PR; inspección de cambios a documentación y diagramas. | Un cambio significativo no se integra sin ADR aprobado y actualizado. Cambios de comportamiento de cupos, prerrequisitos, cancelación o confirmación requieren aprobación explícita del responsable académico y pruebas de regresión. |

### Alcance de “cambio significativo”

Se considera significativo cualquier cambio que altere: límites o dependencias entre módulos; persistencia o transacciones de inscripción; control de concurrencia o idempotencia; protocolo, datos o semántica del sistema legado; mecanismo de autenticación/autorización; objetivos de disponibilidad/recuperación; infraestructura o componentes operativos; o una decisión arquitectónica previamente aceptada.

Un cambio visual o una refactorización interna que preserve interfaces, datos, invariantes y comportamiento puede tramitarse sin ADR nuevo, siempre que las pruebas sigan pasando y el responsable técnico confirme en el PR que no modifica una decisión arquitectónica.

## 3. Responsables de decisión y aprobación

| Rol | Responsabilidades |
|---|---|
| **Autor del PR** (uno de los desarrolladores) | Explicar el cambio, enlazar requisitos y drivers, actualizar pruebas, documentación, diagramas y ADR afectados; declarar excepciones o deuda técnica. |
| **Responsable técnico de arquitectura** (un desarrollador designado por el equipo) | Mantener el mapa de módulos y ADR, clasificar si el cambio es significativo, verificar límites arquitectónicos, convocar revisión técnica y registrar la decisión. No puede aprobar en solitario un cambio que él mismo propuso. |
| **Segundo desarrollador revisor** | Revisar corrección, pruebas, mantenibilidad y efectos en otros módulos. Para cambios significativos, verifica el ADR y la evidencia técnica. |
| **Administrador de infraestructura** | Revisar cambios de despliegue, Linux, PostgreSQL, respaldos, recuperación, seguridad operacional, monitorización y uso de recursos. Aprueba desde la perspectiva operativa cuando el PR afecte su área; no sustituye la aprobación de negocio académico. |
| **Responsable académico designado por la universidad** | Aprobar cambios que alteren reglas académicas, validez de prerrequisitos, capacidad autorizada, fechas de cancelación o el significado de “inscripción confirmada”. |

El responsable técnico coordina la arquitectura; la autoridad sobre reglas académicas y datos oficiales permanece en la universidad. Un PR que no tenga un representante académico disponible puede avanzar en trabajo técnico que no cambie esas reglas, pero no puede activar ni desplegar una semántica académica nueva sin esa aprobación.

## 4. Procedimiento para proponer, revisar, aprobar y actualizar un ADR

1. **Identificar el disparador.** El autor o revisor detecta una decisión nueva, un cambio de contexto, un conflicto entre drivers o evidencia de que una decisión vigente no satisface sus criterios.
2. **Redactar una propuesta.** Crear un ADR numerado en `docs/actividad3/` usando la plantilla del repositorio. Incluir contexto, drivers afectados, alternativas técnicamente viables, criterios, decisión propuesta, consecuencias positivas/negativas, riesgos, mitigaciones y condiciones de revisión. El estado inicial es `Propuesto`.
3. **Enlazar el ADR al PR.** El PR identifica el ADR y explica qué archivos, módulos, datos, reglas, diagramas y pruebas cambian. Si no se requiere ADR nuevo, explica por qué la decisión existente no cambia.
4. **Revisar antes de aprobar.** El responsable técnico comprueba coherencia con drivers y políticas; otro desarrollador verifica las alternativas y consecuencias. El administrador revisa impactos operativos cuando corresponda; el responsable académico revisa cambios de reglas o significado de estados oficiales.
5. **Resolver explícitamente.** La propuesta queda `Aceptada` solo cuando los aprobadores requeridos han dejado su decisión en la revisión del PR. Si se rechaza, el ADR conserva la razón y pasa a `Rechazada`; si se difiere, pasa a `Propuesta aplazada` con el trabajo pendiente. La aprobación del ADR no reemplaza los controles del PR.
6. **Implementar y actualizar.** Al integrar el cambio, actualizar el ADR con fecha, estado final, enlace al PR y cualquier desviación respecto a lo decidido. Si el ADR existente deja de ser vigente, crear uno nuevo que lo reemplace y cambiar el estado del anterior a `Reemplazada por ADR-NNN`; no reescribir el historial como si la decisión anterior nunca hubiese existido.
7. **Revisar decisiones vigentes.** En una revisión arquitectónica trimestral y antes de cada período de matrícula, revisar ADR aceptados cuyas condiciones de revisión puedan haberse cumplido, así como métricas, excepciones y deuda relacionada. Registrar decisión y acciones como cambios trazables.

## 5. Controles automáticos del repositorio

Los controles se ejecutarán para cada PR dirigido a la rama de integración acordada y antes de fusionar. La rama de trabajo permanente del estudiante es `dev_sfonseca93`; el equipo debe definir la rama de integración protegida antes de aplicar la política a un repositorio compartido.

### CA-01 — Verificación automática de límites arquitectónicos

- **Objetivo:** impedir que módulos externos a inscripciones escriban directamente sus datos o dependan de implementaciones internas prohibidas.
- **Implementación:** al definir el lenguaje y la estructura del código, fijar una lista explícita de dependencias permitidas entre módulos y ejecutar en CI un analizador de dependencias o una prueba estática equivalente. Complementarlo con pruebas de esquema/migración que verifiquen restricciones únicas e invariantes de cupo.
- **Ejemplos de fallas bloqueantes:** controlador que actualiza directamente el contador; módulo de presentación que importa repositorios PostgreSQL; integración del legado que modifica una inscripción sin pasar por el caso de uso; ausencia de restricción requerida.
- **Resultado exigido:** estado verde en CI, sin dependencias no permitidas ni violaciones. Las excepciones temporales deben estar registradas y autorizadas según la sección 7.

### CA-02 — Pruebas automatizadas en CI

- **Objetivo:** ejecutar la suite de pruebas pertinente ante cada cambio, no depender de que el autor afirme que el código funciona.
- **Implementación:** el pipeline ejecuta, como mínimo, compilación o comprobación equivalente, pruebas unitarias y pruebas de integración. Los cambios en inscripción, concurrencia o idempotencia deben ejecutar pruebas que compitan por cupos, repitan la solicitud cinco veces y comprueben rollback; los cambios de integración ejecutan pruebas con timeout, recuperación y evento duplicado.
- **Resultado exigido:** todas las comprobaciones obligatorias finalizan correctamente. Las pruebas omitidas por limitación justificada se registran como excepción con responsable y vencimiento; no se reportan como aprobadas.
- **Configuración pendiente:** cuando el equipo seleccione lenguaje y CI, documentar los comandos exactos usados y mantenerlos en scripts/configuración versionados. Esta actividad no presupone ni afirma que una plataforma o pipeline esté configurado hoy.

### CA-03 — Controles complementarios recomendados

1. Validar en CI que los Markdown no tengan enlaces relativos rotos y que cada diagrama referido exista.
2. Ejecutar pruebas de migraciones para detectar pérdida o alteración accidental de restricciones de capacidad, idempotencia y unicidad.
3. Solicitar revisión adicional si un PR cambia `docs/actividad3/`, el esquema de datos, el adaptador del legado o los mecanismos de autenticación.

CA-03 complementa a CA-01 y CA-02; no los reemplaza.

## 6. Procedimiento para actualizar documentación y diagramas

1. Todo PR identifica las secciones de documentación afectadas, incluso si la respuesta es “ninguna”, con una justificación breve.
2. Si cambia un actor, sistema externo, responsabilidad, contenedor, componente, dependencia o comunicación, actualizar en el mismo PR los diagramas C4 de `docs/diagramas/` y su explicación en `docs/actividad2.md`.
3. Si cambia una decisión, crear o actualizar el ADR en `docs/actividad3/`; no usar el diagrama como sustituto de la justificación.
4. Si cambia un driver, criterio de aceptación o táctica, revisar `docs/actividad1.md` y `docs/actividad4.md`, respectivamente, y explicar cualquier vínculo que deje de aplicar.
5. Los archivos Markdown y diagramas deben estar en el mismo PR que el cambio de código correspondiente. No se acepta “actualizar la documentación después” como estado final de una funcionalidad integrada.
6. El revisor compara los documentos con el código y las pruebas. CA-03 valida referencias rotas; la revisión humana valida que los diagramas representen responsabilidades y relaciones reales.

## 7. Excepciones, incumplimientos y deuda técnica

Una excepción no convierte una política en opcional ni permite ocultar un fallo:

1. El autor crea un registro de excepción en el PR o en el ADR con política afectada, motivo, alcance, riesgo, mitigación temporal, responsable y fecha de vencimiento.
2. El responsable técnico revisa toda excepción. El administrador de infraestructura también aprueba las que afecten operación/seguridad y el responsable académico las que puedan afectar validez de inscripciones o reglas oficiales.
3. Las excepciones de una política P0 (capacidad, idempotencia o validación académica oficial) no pueden autorizar una sobreinscripción, duplicación conocida ni confirmación basada en una validación ausente. El cambio se bloquea hasta corregir el defecto o definir una conducta segura que mantenga esos invariantes.
4. Para una excepción temporal de otra política, se documenta una fecha de caducidad concreta y un issue/tarea de remediación. Como límite propuesto, vence en 30 días o antes del siguiente período de matrícula, lo que ocurra primero. Renovarla exige nueva revisión y evidencia; no hay renovaciones tácitas.
5. Un control automático fallido bloquea el PR salvo excepción registrada y aprobada. Una falla que pueda violar capacidad, idempotencia, autorización o integridad académica bloquea la fusión y el despliegue; no se admite como deuda técnica aceptable.
6. El responsable técnico mantiene un registro visible de excepciones y deuda con impacto, riesgo, propietario, acción correctiva y vencimiento. En la revisión trimestral verifica vencimientos; si una excepción caduca sin solución, se bloquean cambios relacionados y se escala al responsable del equipo.

## 8. Criterios de aceptación o rechazo de un PR

### Aceptar para fusión únicamente cuando

- El PR está dirigido a la rama acordada, describe el objetivo y enlaza el requisito, driver o defecto que atiende.
- Un desarrollador distinto del autor revisó y aprobó el cambio; los cambios de arquitectura tienen ADR aprobado por el responsable técnico y las aprobaciones académica/operativa aplicables.
- CA-01 y CA-02 están en estado exitoso, y las pruebas proporcionan evidencia para los invariantes que el PR pueda afectar.
- No se excede la capacidad de ningún grupo; los reintentos no crean duplicados; no se confirma una validación de prerrequisitos no recibida.
- Los controles de autorización y la auditoría no se debilitan; no se incorporan secretos ni datos personales innecesarios.
- La documentación, ADR, diagramas y pruebas afectados se actualizaron en el mismo PR; los enlaces internos verificables no están rotos.
- Toda excepción está registrada, aprobada por los roles necesarios, mitigada, asignada a un responsable y no ha vencido.
- El administrador de infraestructura aprobó los cambios que afecten despliegue, base de datos, copias de seguridad, recuperación o monitorización.

### Rechazar o solicitar cambios cuando

- Falla un control obligatorio o falta evidencia de prueba; el resultado no puede reemplazarse por una captura o afirmación manual.
- El cambio rompe una política P0, acepta cupos mediante una comprobación solo en cliente o muestra como oficial una operación sin validación/acuse exigido.
- Falta el ADR requerido, las alternativas no comparan costos/criterios, o la decisión contradice un ADR aceptado sin reemplazarlo.
- Código, diagramas y documentación describen arquitecturas distintas, o un diagrama deja de reflejar un límite alterado.
- Se introduce una dependencia comercial o infraestructura no autorizada sin decisión y aprobación explícitas.
- Existe una excepción vencida, sin mitigación, sin propietario o con riesgo no aceptado por el responsable correspondiente.

Una solicitud de cambios no equivale a rechazo definitivo: el autor puede corregir el PR y solicitar una nueva revisión. Solo la fusión del PR aceptado integra la modificación; esta actividad no define ni automatiza el mecanismo de despliegue a producción.

## 9. Revisión y evolución de esta gobernanza

El responsable técnico revisa estas reglas cada trimestre y antes del período de matrícula con el equipo de desarrollo y el administrador de infraestructura. El responsable académico participa cuando se revise el significado de estados, prerrequisitos, cupos o cancelaciones. Toda modificación de una política requiere un PR, justificación vinculada a drivers y aprobación de los roles afectados.

La gobernanza se considera efectiva cuando los controles están configurados, bloquean incumplimientos reales y su evidencia se conserva con cada PR. La sola existencia de este documento no demuestra que el repositorio tenga protección de ramas, CI ni automatizaciones instaladas.
