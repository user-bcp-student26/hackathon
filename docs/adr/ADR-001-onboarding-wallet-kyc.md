# ADR-001: Registro, activación de wallet, contactos, autenticación y KYC del MVP

- **Estado:** Propuesta
- **Fecha:** 2026-10-16
- **Historia de usuario:** [H-01: Registro y activación del MVP de wallet](https://github.com/user-bcp-student26/hackathon/issues/1)
- **Épica:** [MVP de wallet en Perú](../../packages/epic/peru-wallet-mvp.sp.md)

## Contexto

La historia H-01 exige que un usuario de la app web pueda:

1. Registrarse con nombre, correo, celular y contraseña obligatorios (validación bloqueante y contraseña solo como hash).
2. Generar una wallet propia (relación 1 a 1, `wallet_id` único generado por el sistema, estado activo).
3. Registrar contactos con nombre, celular y `wallet_id` obligatorios, validando que la wallet exista.
4. Autenticarse con correo y contraseña, con error genérico ante credenciales incorrectas.
5. Completar una verificación KYC solo con sesión activa, con resultado aprobado, rechazado o pendiente.
6. Habilitar la wallet para pagos de bajo monto solo si KYC está aprobado.

La épica añade restricciones: el MVP no debe asumir un único riel de procesamiento, las integraciones de proveedores son contractuales, y el onboarding debe ser rápido (objetivo: de 8 a 3 minutos). La generación de wallet no requiere KYC aprobado; los pagos sí.

## Decisión

### 1. Contraseñas
- Se almacenan únicamente como hash **bcrypt** (con costo configurable). Nunca en texto plano ni en logs.

### 2. Autenticación y sesión
- Mecanismo elegido: **HTTP Basic Auth** sobre HTTPS. Las credenciales (correo y contraseña) se envían en cada solicitud protegida.
- El endpoint de autenticación valida correo y contraseña y responde con código de éxito o con error genérico (`401`) sin indicar qué campo falló.
- **Desviación del criterio 4:** la historia pide "una sesión o token válido". Basic Auth no emite token ni sesión. Esta desviación debe aceptarse explícitamente por producto/seguridad (ver Riesgos).

### 3. Wallet
- Relación 1 usuario a 1 wallet, con restricción única sobre el usuario en base de datos.
- `wallet_id` es un **UUID v4** generado por el sistema, almacenado como `string`.
- Estado inicial: `active`. La habilitación para pagos se controla por separado (ver KYC).

### 4. Contactos
- Campos obligatorios: nombre, celular y `wallet_id`.
- Al registrar un contacto, el backend valida que el `wallet_id` exista en la tabla de wallets y persiste el `wallet_id` junto al contacto.

### 5. KYC
- La verificación la ejecuta un **proveedor externo** vía API. El resultado (`approved`, `rejected`, `pending`) se registra en base de datos.
- El endpoint de KYC requiere autenticación y recibe los datos de identidad requeridos.
- Si el resultado llega de forma asíncrona (`pending`), el sistema actualiza el estado cuando el proveedor responde.
- Solo `approved` habilita pagos de bajo monto. `rejected` y `pending` mantienen la wallet deshabilitada para pagos y devuelven un mensaje accionable.

### 6. Modelo de datos (mínimo)
- `users`: `id`, `name`, `email` (único), `phone`, `password_hash`, `created_at`.
- `wallets`: `wallet_id` (UUID, único), `user_id` (único, FK a `users`), `status` (`active`), `payments_enabled` (boolean), `created_at`.
- `contacts`: `id`, `owner_user_id` (FK), `name`, `phone`, `wallet_id` (FK a `wallets`), `created_at`.
- `kyc_verifications`: `id`, `user_id` (FK), `provider_reference`, `status` (`approved`/`rejected`/`pending`), `updated_at`.

### 7. Validaciones
- Frontend y backend validan obligatorios y formato. El backend es la fuente de verdad.

## Alternativas consideradas

| Tema | Alternativa | Motivo de descarte |
|---|---|---|
| Hash de contraseña | Argon2id | Válida y más moderna, pero bcrypt fue la elección del equipo. |
| Sesión | JWT con refresh | Requiere emisión y rotación de tokens; no fue la elección. Se puede reconsiderar. |
| Sesión | Sesión server-side con cookie httpOnly | Permite revocar sesiones; no fue la elección. Se puede reconsiderar. |
| KYC | Revisión interna/manual | Mayor costo operativo y tiempo de onboarding; no fue la elección. |

## Consecuencias

**Positivas**
- Implementación simple y sin estado de sesión en el servidor.
- Un único modelo de identidad para registro, contactos y KYC.
- Separar "wallet creada" de "pagos habilitados" permite cumplir el criterio 6 sin bloquear el onboarding.

**Negativas y riesgos**
- **Basic Auth envía la contraseña en cada request.** Exige HTTPS obligatorio y HSTS. Un token filtrado en logs o en el navegador expone la credencial real.
- **No hay logout ni revocación de sesión.** Cambiar la contraseña es la única forma de cortar acceso.
- **No cumple literalmente el criterio 4** ("sesión o token válido"). Requiere aceptación formal de producto y seguridad.
- Sin límite de intentos, Basic Auth es vulnerable a fuerza bruta. Se requiere rate limiting por cuenta e IP.
- Un proveedor KYC externo añade dependencia contractual y de disponibilidad (la épica ya lo identifica como dependencia).

## Pendientes antes de implementar

- Aprobar la desviación de Basic Auth frente al criterio 4, o reemplazar el mecanismo de sesión.
- Definir con cumplimiento y producto los documentos y datos exigidos para KYC (supuesto de la historia).
- Confirmar el proveedor KYC y su contrato de webhook o consulta de resultado.
- Definir el costo de bcrypt y la política de rate limiting.
