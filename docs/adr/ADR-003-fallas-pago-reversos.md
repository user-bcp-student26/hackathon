# ADR-003: Manejo de fallas de pago, expiraciones y reversos

- **Estado:** Propuesta
- **Fecha:** 2026-10-16
- **Historia de usuario:** [H-05: Manejo de fallas de pago y reversos](https://github.com/user-bcp-student26/hackathon/issues/5) (equivalente a `packages/user-stories/H05-payment-failure-reversal-handling.sp.md`)
- **Épica:** [MVP de wallet en Perú](../../packages/epic/peru-wallet-mvp.sp.md)
- **API de referencia:** [W5 · Fallas y reversos](../../packages/api-md/W5%20-%20FallasReversos.md)
- **Diagrama:** `docs/diagrams/ADR-003-fallas-reversos.drawio`

## Contexto

La historia H-05 exige que los pagos fallidos, expirados y disputados sigan rutas de atención explícitas:

1. Clasificar y almacenar los resultados fallido y expirado con un código de motivo legible por máquina.
2. Mostrar a soporte si un pago es elegible para reintento, investigación o flujo de disputa.
3. Diferenciar, en la ruta de ayuda al usuario, una falla técnica de una transferencia ya ejecutada.
4. Registrar las acciones de soporte con referencia de transacción y estado de resolución.

ADR-002 fija la máquina de estados (`created` → `processing` → terminal `succeeded` / `failed` / `expired`), con una sola transición terminal y registro en `payment_events`. ADR-002 §6 fue corregido en este mismo cambio para no permitir reintento en `expired` con resultado incierto (ver D2).

La API W5 define `POST /payments/{id}/resolve`, `POST /payments/{id}/reversals`, `GET /reversals/{id}` y `GET /support/cases`. Indica que un error de comunicación (`504`) no demuestra que el pago haya fallado, y que `PROVIDER_ERROR` (`503`) ocurre antes de procesar y no crea operación ni reserva de saldo.

La épica exige que el MVP no asuma un único riel, que las integraciones sean contractuales y que el producto opere 24x7x365. Identifica como riesgo que la cancelación unilateral de P2P no está garantizada de forma uniforme. No existe una regla pública uniforme para cancelar transferencias P2P ejecutadas, y las obligaciones regulatorias sobre fraude, disputas y devoluciones/reversos aumentan desde 2027. Este ADR no define el tratamiento legal de esas obligaciones.

## Decisión

### 1. Clasificación y código de motivo

Cada pago en estado `failed`, `expired` o con resultado incierto almacena:

- `execution_status`: `NOT_EXECUTED` (el proveedor confirma que no hubo cobro), `UNKNOWN` (sin confirmación) o `EXECUTED` (el proveedor confirma la transferencia). Determina la ruta; no se infiere del estado del pago.
- `failure_code`: enumeración cerrada, versionada en el contrato de API:

| `failure_code` | Estado | `execution_status` | Origen en W5 / significado |
|---|---|---|---|
| `PROVIDER_ERROR` | `failed` | `NOT_EXECUTED` | Respuesta `503` antes de procesar; no crea operación ni reserva de saldo. |
| `RESULT_UNKNOWN` | `processing` | `UNKNOWN` | Respuesta `504` con `paymentId` (`TIMEOUT_AFTER_PROCESSING`). El proveedor pudo procesar el pago. |
| `PROVIDER_REJECTED` | `failed` | *Pendiente* | El proveedor rechaza (`REJECTED` vía control). Confirmar si implica `NOT_EXECUTED`. |
| `PENDING_TIMEOUT` | `expired` | `UNKNOWN` | Expira el plazo con resultado `PENDING` del proveedor (se abre caso de soporte, W5). |
| `EXPIRED_NO_RESULT` | `expired` | `UNKNOWN` | Expira el plazo sin resultado del proveedor. |
| `LATE_CONFIRMATION` | `expired` | `EXECUTED` | El proveedor confirma éxito después de `expired`. Se registra en `payment_events`; el estado terminal no cambia (ADR-002 §3). |

- `failure_code` y `execution_status` son códigos de catálogo. No contienen texto libre, datos personales, números de cuenta, teléfonos, nombres ni respuestas crudas del proveedor.
- El código de error original del proveedor se guarda como `provider_error_code` (opaco, sin mensaje) y se mapea al catálogo interno por proveedor.
- El campo `scenario` de W5 es un artefacto de simulación. No puede aparecer en respuestas de producción ni en la vista de soporte (D6).

### 2. Elegibilidad y vista de soporte

La elegibilidad se calcula al consultar, a partir de `payments.status`, `execution_status`, casos y reversos asociados. No se persiste.

| Estado | `execution_status` | Reverso / caso | Elegibilidad a soporte |
|---|---|---|---|
| `failed` | `NOT_EXECUTED` | ninguno | **Reintento** (nuevo intento, nueva `Idempotency-Key`, tras confirmación del usuario) |
| `failed` | `UNKNOWN` | ninguno | **Investigación** (sin reintento) |
| `processing` | `UNKNOWN` | ninguno | **Investigación**; acción `POST /payments/{id}/resolve` |
| `expired` | `UNKNOWN` | ninguno | **Investigación** (sin reintento hasta resolver) |
| `expired` | `EXECUTED` | ninguno | **Investigación** (conciliación; política pendiente) |
| `succeeded` | `EXECUTED` | ninguno | **Disputa** (solo con reclamo; categorías pendientes) |
| `succeeded` | `EXECUTED` | reverso `PENDING` | **Investigación** (reverso en curso) |
| `succeeded` | `EXECUTED` | reverso `SUCCEEDED` | Cerrado (saldo devuelto) |

- **Regla de reintento:** solo `execution_status = NOT_EXECUTED` habilita reintento (corrige ADR-002 §6 para `expired`; ver D2).
- La vista de soporte (`GET /support/cases`, con campos ampliados) muestra: `payment_id`, `transaction_reference` (si existe), `status`, `failure_code`, `execution_status`, `eligibility` (`retry` / `investigate` / `dispute`), `wallet_id` enmascarado y `resolution_status`.
- Los casos OPEN y RESOLVED se listan en el mismo endpoint, que también atiende casos KYC (W5).

### 3. Ruta de ayuda al usuario

El backend devuelve `user_message_key` (enumeración, no texto libre) y `next_actions`. La app web muestra el texto según la clave:

- **Ruta A: falla técnica sin cobro** (`NOT_EXECUTED`). Mensaje: el pago no se completó y no hubo cobro. `next_actions: ["retry", "back"]`. El reintento requiere confirmación explícita (ADR-002 §6).
- **Ruta B: resultado en verificación** (`UNKNOWN`, `processing` o `expired`). Mensaje: estamos verificando el estado; no vuelvas a enviar el pago. `next_actions: ["check_status", "contact_support"]`. Sin reintento.
- **Ruta C: transferencia ejecutada** (`EXECUTED`). Mensaje: la transferencia se realizó; se muestra `transaction_reference`. `next_actions: ["view_receipt", "open_dispute"]`. Esta ruta nunca usa la palabra "fallido".

Los mensajes son genéricos y no revelan motivos técnicos del proveedor. Los textos concretos quedan en Pendientes.

### 4. Acciones de soporte, referencia y estado de resolución

- **Referencia:** `payment_id` (UUID v4) es la referencia de soporte desde la creación del pago. `transaction_reference` (ADR-002 §5) existe solo en `succeeded`. Ambas se devuelven en la vista de soporte.
- **`support_cases`:** `id`, `payment_id` (FK, nullable para casos KYC), `reason_code` (catálogo), `eligibility`, `resolution_status` (`OPEN`, `IN_REVIEW`, `RESOLVED`), `opened_at`, `resolved_at`, `resolved_by_agent_id`. Restricción única: un caso `OPEN` por `payment_id`.
- **`support_actions`** (solo inserciones, sin `UPDATE` ni `DELETE`): `id`, `case_id` (FK), `payment_id` (FK), `transaction_reference`, `action_type` (`RESOLVE_QUERY`, `REQUEST_REVERSAL`, `OPEN_CASE`, `CHANGE_RESOLUTION_STATUS`), `agent_id`, `resolution_status_before`, `resolution_status_after`, `reason_code`, `correlation_id`, `occurred_at`.
- Cambiar el estado de resolución genera una nueva fila; no se edita la anterior.
- **`reversals`:** `id` (prefijo `rev`), `payment_id` (FK, restricción única: un reverso completo por pago, según W5), `status` (`PENDING`, `SUCCEEDED`), `requested_by_agent_id`, `provider_reference`, `created_at`, `completed_at`.
- Un reverso `PENDING` no cambia el estado terminal de `payments`. W5 indica que el proveedor pasa a `REVERSED`; ese estado se guarda como `provider_status` y no sustituye el estado del pago.
- Evento `payment.reversed` publicado mediante `outbox_events` al completarse el reverso (`SUCCEEDED`).
- **Aprobación tardía (W5):** una aprobación del proveedor recibida después de resolver el pago y completar su reverso se acepta sin nuevos cargos ni cambio de estado. Se registra en `payment_events`.

### 5. Acceso de soporte y auditoría (propuesta, no definida en W5)

- **Rol explícito:** `/support/*`, `POST /payments/{id}/resolve` y `POST /payments/{id}/reversals` requieren el rol `support`, validado en servidor en cada solicitud. Un usuario final no accede a estos endpoints aunque tenga credenciales válidas. Extiende ADR-001, que no tiene roles (D5).
- **Mínimo privilegio:** el agente ve solo casos asignados o según su nivel de escalamiento (pendiente).
- **Auditoría:** cada lectura de detalle de pago y cada acción se registra en `support_actions` con `agent_id`, `correlation_id` y `occurred_at`.
- **Enmascarado:** `wallet_id` e identificadores de usuario se muestran enmascarados.
- **Errores:** `404` uniforme cuando el recurso está fuera del alcance del agente. `409` con código (`PAYMENT_NOT_REVERSIBLE`, según W5), no texto del proveedor.

### 6. Idempotencia y concurrencia

- `POST /payments/{id}/resolve` no crea cobros (W5). Repetirlo conserva el resultado. No abre un segundo caso `OPEN` para el mismo pago. También puede completar un reverso pendiente (W5).
- `POST /payments/{id}/reversals` no requiere `Idempotency-Key` (W5). Su idempotencia se garantiza con la restricción única sobre `reversals.payment_id` y la respuesta `200` con el mismo reverso.
- Los límites de frecuencia (`429`) aplican también a los endpoints de soporte.

## Alternativas consideradas

| Tema | Alternativa | Motivo de descarte |
|---|---|---|
| Cancelar P2P ejecutada | Cancelación unilateral por soporte | No existe regla pública uniforme. El reverso solo procede sobre pagos `SUCCEEDED` (W5) y sujeto a política pendiente. |
| Reintento | Reintento automático en `failed` y `expired` | Riesgo de cobro duplicado (ADR-002 ya lo descarta). |
| Reintento | Reintento en `expired` sin verificar | Puede existir cobro (`LATE_CONFIRMATION`). Contradice H-05. |
| Reverso | Estado `REVERSED` en `payments` | Rompe la transición terminal única de ADR-002. Se usa `reversals`. |
| Elegibilidad | Persistir `eligibility` en `payments` | Riesgo de desfase con el estado real. Se calcula al consultar. |
| Código de motivo | Texto libre o respuesta cruda del proveedor | Riesgo de filtrar datos y dificulta la lógica automática. Se usa enum. |
| Auditoría | Tabla de acciones editable | Pierde trazabilidad. Se usa tabla de solo inserciones. |
| Referencia de soporte | `transaction_reference` solo en `succeeded` | No cumple H-05 criterio 4 para fallidos y expirados. Se usa `payment_id`. |

## Consecuencias

**Positivas**
- Cada pago fallido o expirado tiene motivo legible por máquina y una ruta de atención.
- Se evita reintentar donde podría existir un cobro.
- Soporte audita todas sus acciones y cambios de resolución.
- Los reversos no alteran el invariante de estados de ADR-002.

**Negativas y riesgos**
- **Seguridad:** el rol `support` requiere modelo de roles nuevo. Con Basic Auth (ADR-001) las credenciales viajan en cada solicitud; se recomienda autenticación reforzada para soporte (pendiente).
- **Seguridad:** `GET /payments/{id}` para soporte rompe el aislamiento por dueño de ADR-002; por eso el acceso es por rol y auditado.
- **Seguridad:** ampliar el catálogo de códigos sin revisión puede filtrar datos en logs.
- **Operación:** SQLite tiene un solo escritor; soporte, resoluciones y outbox compiten con los pagos (heredado de ADR-001 y ADR-002).
- **Operación:** sin SLA ni niveles de escalamiento, las colas de soporte no se pueden medir.
- **Dependencia:** el proveedor debe confirmar `NOT_EXECUTED` de forma explícita; sin eso, todo fallo queda en investigación.
- **Dependencia:** el proveedor debe ofrecer consulta de estado y reverso, 24x7x365 según la épica.
- **Épica:** métricas de fallas, disputas y tiempo de resolución no están definidas.

## Desviaciones

| # | Criterio o decisión previa | Desviación | Riesgo | Aprobación requerida |
|---|---|---|---|---|
| D1 | H-05 criterios 2 y 4; W5 `GET /support/cases` | W5 devuelve solo `id`, `resource`, `reason`, `status`. No muestra elegibilidad, referencia ni estado de resolución. | Soporte no cumple su flujo con el contrato actual. | Producto y equipo de API (cambio de contrato). |
| D2 | ADR-002 §6 (`retry` en `expired`) | Corregido: en `expired` con `execution_status = UNKNOWN` no se permite reintento. | Posible cobro duplicado (`LATE_CONFIRMATION`). Contradice H-05 criterio 3. | Aprobado para ADR-002 en este cambio. |
| D3 | ADR-002 §3 (estado terminal inmutable) vs W5 `POST /payments/{id}/resolve` ("si ya terminó, aplica el resultado") | W5 permite aplicar resultado sobre un pago terminal; ADR-002 lo prohíbe. | Contradicción entre contratos. Propuesta: `resolve` cambia estado solo desde no terminal; sobre terminal registra evento y marca conciliación. | Producto y arquitectura (modifica ADR-002 o W5). |
| D4 | ADR-002 §5 (`transaction_reference` solo en `succeeded`) vs H-05 criterio 4 | Sin `transaction_reference` en fallidos y expirados. | Acciones de soporte sin referencia de transacción. Se usa `payment_id` como referencia de soporte. | Producto. |
| D5 | ADR-001 (sin roles) vs H-05 criterio 2 | No existe modelo de roles. | Sin roles no hay control de acceso de soporte. | Seguridad (extiende ADR-001). |
| D6 | W5 (campo `scenario` en la respuesta de pago) | Campo de simulación visible en la respuesta. | Expone artefactos de prueba. | Equipo de API: retirar en producción. |

## Pendientes antes de implementar

**Negocio y producto**
- Categorías de disputa para MVP y post-MVP (pregunta abierta de H-05). Hasta confirmarlas, la ruta de disputa existe sin categorías.
- Niveles de escalamiento y SLA por nivel (supuesto de H-05).
- Política de reverso de P2P ejecutadas: completo o parcial, y quién lo autoriza.
- Si se exige doble aprobación para reversos.
- Tratamiento de `LATE_CONFIRMATION` (reabrir, conciliar o reembolsar). Bloqueante para la ruta de soporte.
- Impacto de obligaciones de fraude, disputas y reversos desde 2027. Requiere revisión de cumplimiento; no se incluye en el MVP sin confirmación.
- Textos de `user_message_key` para las rutas A, B y C.
- Métricas de fallas, disputas y tiempo de resolución (no están en la épica).

**Proveedores**
- Catálogo de códigos de error del proveedor y su mapeo a `failure_code`, por riel.
- Confirmar si `REJECTED` implica `NOT_EXECUTED` (`PROVIDER_REJECTED`).
- Confirmar que el proveedor distingue `NOT_EXECUTED` de `UNKNOWN`.
- Plazo de reverso del proveedor y su política de notificación de reversos.
- Confirmar 24x7x365 para consulta de resolución y reverso.

**Seguridad y operación**
- Modelo de roles (`support` y niveles) y mecanismo de autenticación para soporte frente a Basic Auth (ADR-001).
- Retención de `support_actions`, `payment_events` y datos de soporte.
- Política de auditoría de lecturas de datos de pago por soporte.
- Límites de frecuencia para endpoints de soporte.
- Cambios de contrato W5 (D1, D3, D6).
- Relación con la conciliación de H-06 (casos de soporte como fuente de investigaciones).
