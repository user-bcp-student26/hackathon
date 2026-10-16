# ADR-002: Confirmación de pagos de bajo monto

- **Estado:** Propuesta
- **Fecha:** 2026-10-16
- **Historia de usuario:** [H-02: Confirmación de pagos de bajo monto](https://github.com/user-bcp-student26/hackathon/issues/2)
- **Épica:** [MVP de wallet en Perú](../../packages/epic/peru-wallet-mvp.sp.md)

## Contexto

La historia H-02 exige que el pagador:

1. Vea identidad del destino y monto antes de confirmar.
2. Reciba un único resultado terminal por intento: exitoso, fallido o expirado.
3. En éxito, vea una referencia de transacción persistida en el historial.
4. En fallo o expiración, tenga una siguiente acción clara sin cobros duplicados.
5. Vea reducida la caída en checkout de 30% a 8% (métrica de negocio).
6. Emita el evento de activación usado en el cálculo de MAU.

La épica exige no asumir un único riel de procesamiento y contempla transiciones asíncronas de estado. Los límites por proveedor pueden ser menores al máximo regulatorio.

## Decisión

### 1. Vista previa previa a la confirmación
- El backend devuelve un **resumen de pago** (nombre del destino, alias o `wallet_id` enmascarado, monto, moneda) antes de la confirmación final.
- El monto se valida en backend contra la regla de monto bajo del proveedor/canal antes de crear el intento. Si excede el límite, se rechaza con mensaje explícito.
- Antes de crear el intento, el backend verifica que el pagador tenga pagos habilitados (`payments_enabled`, ADR-001) y que el destino exista.

### 2. Intento de pago e idempotencia
- Cada intento se crea con una **`Idempotency-Key`** (UUID v4) generada por el cliente al abrir la pantalla de confirmación.
- La combinación (`user_id`, `Idempotency-Key`) tiene restricción **única** en base de datos. Reenvíos devuelven el mismo intento, nunca un nuevo cobro.
- El backend valida formato de `Idempotency-Key` (UUID v4), monto y moneda, y responde con error explícito si son inválidos. Los endpoints de pagos requieren autenticación según ADR-001 y HTTPS obligatorio.
- `GET /payments/{id}` devuelve `404` si el intento no pertenece al usuario autenticado, sin revelar su existencia.

### 3. Máquina de estados
- Estados: `created` → `processing` → **terminal** (`succeeded`, `failed`, `expired`).
- Solo se permite una transición a estado terminal; las transiciones posteriores se ignoran y se registran.
- Un intento sin resultado del proveedor dentro del plazo pasa a `expired`.
- Una confirmación del proveedor recibida después de `expired` no cambia el estado terminal; se registra en `payment_events` y queda marcada para conciliación (ver Riesgos y Pendientes).

### 4. Confirmación asíncrona
- El resultado del proveedor llega por **webhook** y el cliente consulta el estado mediante `GET /payments/{id}` hasta alcanzar un estado terminal.
- El webhook solo se procesa si la firma del proveedor es válida; las firmas inválidas se rechazan sin procesar y se registran. Los webhooks duplicados son seguros por la regla de transición única. El esquema de firma se confirma con el proveedor (ver Pendientes).
- Un job de conciliación consulta intentos en `processing` vencidos y los resuelve.

### 5. Referencia y historial
- Al llegar a `succeeded`, se genera `transaction_reference` (UUID v4) y se persiste junto al intento. Se muestra en pantalla y en el historial.

### 6. Siguiente acción en fallo o expiración
- `failed` devuelve `next_actions: ["retry", "back"]` solo si el proveedor confirma que no hubo cobro (`execution_status = NOT_EXECUTED`, ADR-003).
- `expired` con resultado incierto devuelve `next_actions: ["check_status", "contact_support"]`, sin reintento, para evitar cobros duplicados (ADR-003, D2).
- Reintentar crea un **nuevo** intento con nueva `Idempotency-Key`, solo tras confirmación explícita del usuario.
- Los mensajes al usuario son genéricos y accionables; los detalles técnicos del proveedor solo se registran en logs internos, sin datos sensibles.

### 7. Evento de activación (MAU)
- Al pasar a `succeeded`, el backend emite el evento `payment.succeeded` con `user_id` y `occurred_at`, que alimenta el cálculo de activación para MAU. Se publica tras persistir el estado (patrón outbox).

### 8. Modelo de datos (mínimo)
- `payments`: `id`, `payer_user_id` (FK), `destination_wallet_id` (FK), `provider` (riel/proveedor usado), `amount`, `currency`, `status`, `idempotency_key`, `provider_reference`, `transaction_reference` (nullable), `created_at`, `updated_at`. Restricción única: (`payer_user_id`, `idempotency_key`).
- `payment_events`: `id`, `payment_id` (FK), `from_status`, `to_status`, `occurred_at` (auditoría y conciliación).
- `outbox_events`: `id`, `payment_id` (FK), `event_type` (`payment.succeeded`), `payload`, `created_at`, `published_at` (nullable). Se escribe en la misma transacción que el cambio a `succeeded`.

Diagrama de referencia: `docs/diagrams/ADR-002-flujo-pagos.drawio`.

## Alternativas consideradas

| Tema | Alternativa | Motivo de descarte |
|---|---|---|
| Idempotencia | Deduplicar solo por monto y destino en ventana de tiempo | Falsos positivos y falsos negativos; no garantiza unicidad de intento. |
| Resultado asíncrono | Solo polling sin webhook | Mayor latencia y carga; se mantiene polling como respaldo. |
| Resultado asíncrono | Solo webhook sin consulta | Si el webhook se pierde, el usuario queda sin resultado. |
| Reintento | Reintento automático | Riesgo de cobro duplicado y viola confirmación explícita del pagador. |
| Evento MAU | Calcularlo desde tabla de pagos | Acopla reporte a la base transaccional; el outbox desacopla. |

## Consecuencias

**Positivas**
- Un solo resultado terminal por intento, trazable en `payment_events`.
- Reintentos seguros gracias a la idempotencia.
- Historial y MAU alimentados desde una fuente única de verdad.

**Negativas y riesgos**
- **Riesgo de seguridad:** la `Idempotency-Key` no es secreto; debe validarse contra `user_id` para evitar que otro usuario consulte o reutilice intentos.
- **Riesgo de seguridad:** los webhooks deben validar firma del proveedor; sin ello, cualquiera puede forzar `succeeded`.
- **Riesgo operativo:** SQLite (ADR-001) tiene un solo escritor; es insuficiente para concurrencia de pagos en producción.
- **Riesgo operativo:** el outbox requiere un proceso de publicación y monitoreo; si falla, el MAU queda subcontado.
- **Dependencia:** el proveedor debe ofrecer webhooks y un identificador de referencia estable; sin ello, la conciliación no es posible. Además, debe soportar operación 24x7x365, requisito de la épica.
- **Riesgo de conciliación:** un cobro confirmado por el proveedor después de `expired` deja al pagador con un resultado contrario al cobro real, hasta que se defina y aplique la política de conciliación.
- **Riesgo de abuso:** la creación de intentos y las consultas de estado no tienen límite de frecuencia definido; se requiere rate limiting (ver Pendientes).
- **Desviación:** el criterio 5 (reducción de caída de 30% a 8%) no es verificable por esta decisión técnica; se cubre con instrumentación del funnel, pero requiere aceptación de producto (aprueba: Product Owner).

## Pendientes antes de implementar

- Definir la regla inicial de monto bajo por proveedor/canal (pregunta abierta de la historia).
- Confirmar etiquetas de los estados terminales para el usuario (supuesto de producto).
- Definir el plazo de expiración de un intento en `processing`.
- Confirmar si el proveedor ofrece webhook firmado y su política de reintentos.
- Definir qué evento exacto cuenta como "activación" para MAU.
- Definir la política de reverso/disputa para pagos `succeeded` (riesgo de la épica).
- Definir la política para confirmaciones del proveedor posteriores a `expired` (reabrir, conciliar o reembolsar).
- Confirmar el esquema de firma del webhook y la cabecera correspondiente con el contrato del proveedor.
- Confirmar que el proveedor cumple operación 24x7x365 y su SLA.
- Definir la política de rate limiting para creación de intentos y consultas (decisión de seguridad y producto).
- Confirmar el alcance multirriel del lanzamiento: qué proveedores o canales se habilitan en el MVP.
- Confirmar que el reporte de MAU (1,000-2,000) y CAC (PEN 10) se mide fuera de este ADR y su responsable.
