# Casos de Prueba — Travel Hotel: Motor de Reservas

**Proyecto:** Travel Hotel — Motor de Reservas  
**Versión:** 1.0.0-MVP  
**Fecha:** 2026-03-24  
**Autor:** QA  
**HUs en Alcance:** HU0, HU1, HU2, HU3, HU5, HU6, HU7, HU8, HU11  
**Total de Casos:** 30  

---

## Convenciones

| Campo | Descripción |
|---|---|
| **TC-ID** | Identificador único del caso: `TC-HUX-NN` |
| **Tipo** | Happy path / Error path / Edge case / Concurrencia / Seguridad / Negativo |
| **ASD** | Clasificación de riesgo: A (Alto) / S (Medio) / D (Bajo) |
| **Estado** | Pendiente / Ejecutado / Bloqueado / Pasó / Falló |

---

## HU0 — Configuración del Ecosistema de Datos

> **Riesgo ASD:** A — Alto | **SP:** 5 | **Tipo:** Infraestructura

---

### TC-HU0-01 — Bloqueo de fila con SELECT FOR UPDATE impide modificación concurrente

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU0-01 |
| **HU** | HU0 |
| **Tipo** | Happy path — Infraestructura |
| **ASD** | A — Alto |
| **Tags** | `@smoke @alto @infraestructura` |
| **Estado** | Pendiente |

**Precondiciones**
- PostgreSQL inicializado y accesible.
- La habitación con ID `101` existe en la tabla de habitaciones.
- Dos sesiones de base de datos disponibles (Transacción A y Transacción B).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Iniciar Transacción A en la base de datos | — |
| 2 | Ejecutar `SELECT FOR UPDATE` sobre la fila de habitación `101` desde Transacción A | `habitacion_id = '101'` |
| 3 | Desde Transacción B, intentar ejecutar `UPDATE` sobre la misma fila de habitación `101` | `habitacion_id = '101'` |
| 4 | Verificar el estado de Transacción B | — |
| 5 | Ejecutar `COMMIT` en Transacción A | — |
| 6 | Verificar que Transacción B se desbloquea y aplica su modificación | — |

**Resultado Esperado**
- En el paso 4, Transacción B queda en estado de espera (bloqueada).
- Tras el `COMMIT` de Transacción A, Transacción B se desbloquea y aplica su `UPDATE` exitosamente.
- En ningún momento ambas transacciones modifican el registro de forma simultánea.

---

### TC-HU0-02 — ROLLBACK restaura el estado ante error en transacción

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU0-02 |
| **HU** | HU0 |
| **Tipo** | Error path — Infraestructura |
| **ASD** | A — Alto |
| **Tags** | `@alto @infraestructura` |
| **Estado** | Pendiente |

**Precondiciones**
- PostgreSQL inicializado.
- Habitación `102` existe con estado `disponible`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Iniciar una transacción de reserva sobre la habitación `102` | `habitacion_id = '102'` |
| 2 | Ejecutar parcialmente la operación de reserva (actualizar estado a `bloqueada`) | `estado = 'bloqueada'` |
| 3 | Simular un error durante la operación (timeout o lanzamiento de excepción) | Error forzado |
| 4 | Verificar que se ejecuta `ROLLBACK` automático | — |
| 5 | Consultar el registro de la habitación `102` en la base de datos | `habitacion_id = '102'` |

**Resultado Esperado**
- El `ROLLBACK` se ejecuta sin errores adicionales.
- La habitación `102` mantiene su estado original `disponible`.
- No existe ningún registro parcial de la transacción fallida en la base de datos.

---

### TC-HU0-03 — Pipeline CI/CD ejecuta migraciones sin afectar datos de prueba

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU0-03 |
| **HU** | HU0 |
| **Tipo** | Infraestructura |
| **ASD** | A — Alto |
| **Tags** | `@alto @infraestructura` |
| **Estado** | Pendiente |

**Precondiciones**
- Pipeline CI/CD configurado apuntando a base de datos de prueba separada (no producción).
- Existe al menos una migración pendiente en el repositorio.
- La base de datos de prueba contiene datos pre-existentes cargados por seeder.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Registrar el conteo de registros en tablas clave antes del despliegue | — |
| 2 | Disparar el pipeline CI/CD con la versión que contiene migraciones pendientes | — |
| 3 | Verificar el log de migraciones del pipeline | — |
| 4 | Registrar el conteo de registros en tablas clave después del despliegue | — |
| 5 | Comparar conteos antes y después | — |

**Resultado Esperado**
- Las migraciones se aplican sin errores en el log.
- El conteo de registros pre-existentes permanece igual antes y después del despliegue.
- El entorno de prueba no pierde datos ni queda en estado inconsistente.

---

## HU1 — Seeder de Inventario Inicial

> **Riesgo ASD:** D — Bajo | **SP:** 2 | **Tipo:** Infraestructura

---

### TC-HU1-01 — Carga exitosa de datos maestros

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU1-01 |
| **HU** | HU1 |
| **Tipo** | Happy path |
| **ASD** | D — Bajo |
| **Tags** | `@smoke @bajo` |
| **Estado** | Pendiente |

**Precondiciones**
- Base de datos de prueba vacía (sin datos en Hoteles ni Habitaciones).
- Script de seeder disponible y ejecutable.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Verificar que la tabla Hoteles está vacía | — |
| 2 | Verificar que la tabla Habitaciones está vacía | — |
| 3 | Ejecutar el script de seeder | — |
| 4 | Consultar el conteo de registros en tabla Hoteles | — |
| 5 | Consultar el conteo de registros en tabla Habitaciones | — |
| 6 | Verificar que los registros de Habitaciones tienen campos `tipo` y `capacidad` válidos | — |

**Resultado Esperado**
- Tabla Hoteles contiene al menos 1 registro válido.
- Tabla Habitaciones contiene al menos 3 registros con `tipo` y `capacidad` poblados.
- No se producen errores durante la ejecución del seeder.

---

### TC-HU1-02 — Seeder es idempotente al ejecutarse dos veces

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU1-02 |
| **HU** | HU1 |
| **Tipo** | Edge case |
| **ASD** | D — Bajo |
| **Tags** | `@bajo @edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Seeder ya ejecutado una vez exitosamente.
- Se conoce el conteo exacto de registros tras la primera ejecución.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Registrar el conteo de registros en Hoteles y Habitaciones | — |
| 2 | Ejecutar el script de seeder por segunda vez | — |
| 3 | Consultar el conteo de registros en Hoteles | — |
| 4 | Consultar el conteo de registros en Habitaciones | — |
| 5 | Comparar conteos antes y después de la segunda ejecución | — |

**Resultado Esperado**
- El conteo de registros en Hoteles y Habitaciones es idéntico al de la primera ejecución.
- No se crean registros duplicados.

---

### TC-HU1-03 — Seeder falla con mensaje claro si la base de datos está offline

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU1-03 |
| **HU** | HU1 |
| **Tipo** | Error path |
| **ASD** | D — Bajo |
| **Tags** | `@bajo @error-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Base de datos desconectada o inaccesible (servicio detenido o credenciales incorrectas).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Asegurarse de que la base de datos está offline | — |
| 2 | Ejecutar el script de seeder | — |
| 3 | Capturar la salida del proceso (stdout/stderr) | — |
| 4 | Verificar el código de salida del proceso | — |

**Resultado Esperado**
- El proceso termina con código de error distinto de 0.
- La salida contiene un mensaje descriptivo del error de conexión.
- No quedan tablas o registros en estado inconsistente.

---

## HU2 — Consulta de Disponibilidad Consistente

> **Riesgo ASD:** A — Alto | **SP:** 3 | **Tipo:** Funcional–API

---

### TC-HU2-01 — Habitación sin bloqueo aparece en resultados de disponibilidad

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU2-01 |
| **HU** | HU2 |
| **Tipo** | Happy path |
| **ASD** | A — Alto |
| **Tags** | `@smoke @alto @happy-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Seeder ejecutado. Habitaciones `101` (doble, cap. 2), `102` (individual, cap. 1), `103` (suite, cap. 4) registradas.
- Habitación `102` sin ningún Hold ni reserva activa.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `GET /disponibilidad` con rango de fechas | `fecha_inicio: 2026-04-10`, `fecha_fin: 2026-04-12` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar que la habitación `102` aparece en el listado de resultados | — |

**Resultado Esperado**
- HTTP 200 OK.
- La respuesta contiene la habitación `102` con sus atributos (`tipo: individual`, `capacidad: 1`).

---

### TC-HU2-02 — Hold en fechas distintas no afecta disponibilidad para otras fechas

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU2-02 |
| **HU** | HU2 |
| **Tipo** | Happy path |
| **ASD** | A — Alto |
| **Tags** | `@alto @happy-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación `103` tiene Hold activo del `2026-04-20` al `2026-04-22`.
- No tiene Hold ni reserva en el rango `2026-04-10` al `2026-04-12`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `GET /disponibilidad` buscando del `2026-04-10` al `2026-04-12` | `fecha_inicio: 2026-04-10`, `fecha_fin: 2026-04-12` |
| 2 | Verificar que la habitación `103` aparece en el resultado | — |

**Resultado Esperado**
- HTTP 200 OK.
- La habitación `103` aparece en el listado ya que su Hold no se superpone con el rango consultado.

---

### TC-HU2-03 — Habitación con Hold activo es excluida de resultados

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU2-03 |
| **HU** | HU2 |
| **Tipo** | Error path |
| **ASD** | A — Alto |
| **Tags** | `@smoke @alto @error-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación `101` tiene Hold activo del `2026-04-10` al `2026-04-12` con estado `PENDING`.
- Al menos una habitación alternativa disponible en el mismo rango.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `GET /disponibilidad` | `fecha_inicio: 2026-04-10`, `fecha_fin: 2026-04-12` |
| 2 | Verificar que la habitación `101` NO aparece en los resultados | — |
| 3 | Verificar que al menos otra habitación alternativa sí aparece | — |

**Resultado Esperado**
- HTTP 200 OK.
- La habitación `101` está ausente de los resultados.
- El listado contiene al menos una habitación alternativa disponible.

---

### TC-HU2-04 — Habitación con reserva confirmada activa es excluida de resultados

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU2-04 |
| **HU** | HU2 |
| **Tipo** | Error path |
| **ASD** | A — Alto |
| **Tags** | `@alto @error-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación `103` tiene reserva con estado `CONFIRMED` del `2026-04-10` al `2026-04-12`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `GET /disponibilidad` | `fecha_inicio: 2026-04-10`, `fecha_fin: 2026-04-12` |
| 2 | Verificar que la habitación `103` NO aparece en los resultados | — |

**Resultado Esperado**
- HTTP 200 OK.
- La habitación `103` está ausente del listado de resultados.

---

### TC-HU2-05 — Solapamiento parcial de fechas excluye la habitación

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU2-05 |
| **HU** | HU2 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación `102` tiene Hold activo del `2026-04-11` al `2026-04-14` (solapamiento parcial con `04-10`–`04-12`).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `GET /disponibilidad` | `fecha_inicio: 2026-04-10`, `fecha_fin: 2026-04-12` |
| 2 | Verificar que la habitación `102` NO aparece en los resultados | — |

**Resultado Esperado**
- HTTP 200 OK.
- La habitación `102` está ausente del listado por solapamiento parcial de fechas.

---

### TC-HU2-06 — Hold expirado no afecta disponibilidad actual

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU2-06 |
| **HU** | HU2 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación `101` tiene un Hold con estado `EXPIRED` caducado el `2026-03-01`.
- No tiene bloqueos ni reservas activas en la actualidad.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `GET /disponibilidad` | `fecha_inicio: 2026-04-10`, `fecha_fin: 2026-04-12` |
| 2 | Verificar que la habitación `101` SÍ aparece en los resultados | — |

**Resultado Esperado**
- HTTP 200 OK.
- La habitación `101` aparece en el listado porque el Hold está expirado.

---

### TC-HU2-07 — Sin habitaciones disponibles muestra mensaje informativo

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU2-07 |
| **HU** | HU2 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Todas las habitaciones tienen bloqueos o reservas activas del `2026-04-10` al `2026-04-12`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `GET /disponibilidad` | `fecha_inicio: 2026-04-10`, `fecha_fin: 2026-04-12` |
| 2 | Verificar el cuerpo de la respuesta | — |
| 3 | Verificar que no hay fichas de habitaciones en el resultado | — |

**Resultado Esperado**
- HTTP 200 OK (o 404 según contrato de API).
- La respuesta contiene un mensaje indicando que no hay habitaciones disponibles.
- El array de resultados está vacío (`[]`).

---

## HU3 — Bloqueo Atómico de Checkout (Hold)

> **Riesgo ASD:** A — Alto | **SP:** 8 | **Tipo:** Funcional–Concurrencia

---

### TC-HU3-01 — Bloqueo exitoso genera Hold PENDING con 10 minutos de duración

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU3-01 |
| **HU** | HU3 |
| **Tipo** | Happy path |
| **ASD** | A — Alto |
| **Tags** | `@smoke @alto @happy-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación `202` disponible (sin Hold ni reserva activa).
- Usuario autenticado con token válido.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Registrar la hora actual `T0` | — |
| 2 | Ejecutar `POST /hold` con el token del usuario | `habitacion_id: '202'`, `fecha_inicio: '2026-04-10'`, `fecha_fin: '2026-04-12'` |
| 3 | Verificar el código de respuesta HTTP | — |
| 4 | Verificar el estado del Hold en la respuesta | — |
| 5 | Verificar el campo `expira_en` del Hold | — |

**Resultado Esperado**
- HTTP 201 Created.
- El Hold retornado tiene `estado: PENDING`.
- El campo `expira_en` es igual a `T0 + 10 minutos` (tolerancia ± 5 segundos).

---

### TC-HU3-02 — Prevención de Race Condition: solo el primer usuario obtiene el Hold

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU3-02 |
| **HU** | HU3 |
| **Tipo** | Concurrencia |
| **ASD** | A — Alto |
| **Tags** | `@smoke @alto @concurrencia` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación `202` disponible.
- Usuarios A y B autenticados con tokens distintos.
- Capacidad de ejecutar dos hilos concurrentes (pytest-asyncio o threading).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Lanzar simultáneamente (dos hilos) `POST /hold` para la habitación `202` con usuario A y usuario B | `habitacion_id: '202'`, mismas fechas, tokens distintos |
| 2 | Recopilar ambas respuestas HTTP | — |
| 3 | Verificar cuántos Holds existen en la base de datos para la habitación `202` en esas fechas | — |
| 4 | Identificar cuál usuario recibió éxito y cuál recibió error | — |

**Resultado Esperado**
- Exactamente UNO de los dos recibe HTTP 201 con Hold en estado `PENDING`.
- El otro recibe HTTP 409 (o equivalente) con mensaje `"Habitación no disponible"`.
- La base de datos contiene exactamente UN Hold activo para la habitación `202` en esas fechas.

---

### TC-HU3-03 — Intento de Hold sobre habitación ya bloqueada es rechazado

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU3-03 |
| **HU** | HU3 |
| **Tipo** | Error path |
| **ASD** | A — Alto |
| **Tags** | `@alto @error-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación `202` tiene Hold activo del usuario A en estado `PENDING` del `2026-04-10` al `2026-04-12`.
- Usuario B autenticado con token válido.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `POST /hold` como usuario B | `habitacion_id: '202'`, `fecha_inicio: '2026-04-10'`, `fecha_fin: '2026-04-12'` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar el mensaje de error en la respuesta | — |
| 4 | Consultar el Hold del usuario A en la base de datos | — |

**Resultado Esperado**
- HTTP 409 Conflict (o 400 Bad Request).
- Mensaje de error: `"Habitación no disponible"`.
- El Hold original del usuario A permanece sin modificación (`estado: PENDING`, `expira_en` sin cambios).

---

### TC-HU3-04 — Usuario no autenticado no puede crear un Hold

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU3-04 |
| **HU** | HU3 |
| **Tipo** | Seguridad |
| **ASD** | A — Alto |
| **Tags** | `@alto @seguridad` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación `202` disponible.
- Solicitud enviada sin cabecera `Authorization` (o con token expirado/inválido).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `POST /hold` sin cabecera de autenticación | `habitacion_id: '202'`, `fecha_inicio: '2026-04-10'`, `fecha_fin: '2026-04-12'` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar que no se creó ningún Hold en la base de datos | — |

**Resultado Esperado**
- HTTP 401 Unauthorized.
- No existe ningún registro de Hold para la habitación `202` creado por esta solicitud.

---

## HU5 — Procesamiento de Pago Idempotente

> **Riesgo ASD:** A — Alto | **SP:** 5 | **Tipo:** Funcional–Pago

---

### TC-HU5-01 — Reintento de pago con la misma clave retorna éxito original sin nuevo cobro

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU5-01 |
| **HU** | HU5 |
| **Tipo** | Happy path |
| **ASD** | A — Alto |
| **Tags** | `@smoke @alto @happy-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold `ID-99` existe en estado `PENDING`.
- Pago previo procesado exitosamente para `ID-99` con clave `KEY-ABC`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Registrar el ID de transacción del pago original | `pago_id: PAY-001` |
| 2 | Enviar segunda solicitud de pago con la misma clave | `hold_id: 'ID-99'`, `clave_idempotencia: 'KEY-ABC'`, `monto: 250.00` |
| 3 | Verificar el código de respuesta HTTP | — |
| 4 | Verificar el `pago_id` retornado | — |
| 5 | Verificar que no existe un segundo registro de pago en la base de datos | — |

**Resultado Esperado**
- HTTP 200 OK.
- La respuesta retorna los datos del pago original (`pago_id: PAY-001`, `estado: SUCCESS`).
- Solo existe un registro de pago para `ID-99` con clave `KEY-ABC` en la base de datos.

---

### TC-HU5-02 — Reintento de pago rechazado retorna el rechazo original

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU5-02 |
| **HU** | HU5 |
| **Tipo** | Error path |
| **ASD** | A — Alto |
| **Tags** | `@alto @error-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold `ID-100` existe.
- Pago previo rechazado para `ID-100` con clave `KEY-DEF`, estado `DECLINED`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Enviar segunda solicitud de pago con la misma clave | `hold_id: 'ID-100'`, `clave_idempotencia: 'KEY-DEF'` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar que el estado retornado es `DECLINED` | — |
| 4 | Verificar que no se realizó un nuevo intento de cobro | — |

**Resultado Esperado**
- HTTP 200 OK (respuesta idempotente).
- La respuesta indica `estado: DECLINED` (resultado original).
- No existe un nuevo registro de pago en la pasarela para ese Hold y esa clave.

---

### TC-HU5-03 — Nueva clave de idempotencia permite un nuevo intento de pago

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU5-03 |
| **HU** | HU5 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@alto @edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold `ID-101` existe en estado `PENDING`.
- Pago previo rechazado con clave `KEY-GHI`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Enviar solicitud de pago con clave diferente | `hold_id: 'ID-101'`, `clave_idempotencia: 'KEY-XYZ'`, `monto: 250.00` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar que el sistema procesó la solicitud como nueva transacción | — |

**Resultado Esperado**
- HTTP 200 OK o equivalente según resultado del procesador.
- El sistema procesa la solicitud como transacción nueva (no retorna el resultado de `KEY-GHI`).
- Se genera un nuevo registro de intento de pago en la base de datos.

---

### TC-HU5-04 — Clave de idempotencia no puede reutilizarse para un Hold diferente

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU5-04 |
| **HU** | HU5 |
| **Tipo** | Seguridad |
| **ASD** | A — Alto |
| **Tags** | `@alto @seguridad` |
| **Estado** | Pendiente |

**Precondiciones**
- Clave `KEY-ABC` usada exitosamente para Hold `ID-99`.
- Hold `ID-200` existe en estado `PENDING` (Hold diferente).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Enviar solicitud de pago usando la clave ya utilizada pero para un Hold diferente | `hold_id: 'ID-200'`, `clave_idempotencia: 'KEY-ABC'` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar el mensaje de error | — |

**Resultado Esperado**
- HTTP 400 Bad Request (o 409 Conflict).
- Mensaje de error: `"Clave de idempotencia inválida para este Hold"`.
- No se procesa ningún cobro para `ID-200`.

---

## HU6 — Confirmación Definitiva de Reserva

> **Riesgo ASD:** A — Alto | **SP:** 3 | **Tipo:** Funcional–Estado

---

### TC-HU6-01 — Transición PENDING → CONFIRMED tras pago exitoso

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU6-01 |
| **HU** | HU6 |
| **Tipo** | Happy path |
| **ASD** | A — Alto |
| **Tags** | `@smoke @alto @happy-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold en estado `PENDING` para habitación `305` del `2026-04-10` al `2026-04-12`.
- Procesador de pagos configurado para retornar `SUCCESS`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Enviar confirmación de pago exitosa para el Hold | `hold_id`, `estado_pasarela: 'SUCCESS'` |
| 2 | Consultar el estado del Hold en la base de datos | — |
| 3 | Consultar disponibilidad de habitación `305` para esas fechas | `fecha_inicio: '2026-04-10'`, `fecha_fin: '2026-04-12'` |

**Resultado Esperado**
- El Hold cambia al estado `CONFIRMED`.
- La habitación `305` no aparece en los resultados de disponibilidad para esas fechas.

---

### TC-HU6-02 — Pago fallido no modifica el estado PENDING del Hold

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU6-02 |
| **HU** | HU6 |
| **Tipo** | Error path |
| **ASD** | A — Alto |
| **Tags** | `@alto @error-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold en estado `PENDING` para habitación `305`.
- Procesador de pagos configurado para retornar `FAILED`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Enviar señal de pago fallido al sistema | `hold_id`, `estado_pasarela: 'FAILED'` |
| 2 | Consultar el estado del Hold en la base de datos | — |
| 3 | Consultar disponibilidad de habitación `305` | — |

**Resultado Esperado**
- El Hold permanece en estado `PENDING` (no cambia a `CONFIRMED`).
- La habitación `305` no queda bloqueada permanentemente.

---

### TC-HU6-03 — Intento de confirmar Hold expirado es rechazado

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU6-03 |
| **HU** | HU6 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@alto @edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold fue marcado como `EXPIRED` por el Worker antes de recibir confirmación de pago.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Intentar enviar confirmación de pago `SUCCESS` para el Hold expirado | `hold_id` con `estado: EXPIRED` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar el mensaje de error | — |
| 4 | Consultar el estado del Hold en la base de datos | — |

**Resultado Esperado**
- HTTP 409 Conflict (o 400 Bad Request).
- Mensaje de error: `"Hold expirado — no se puede confirmar"`.
- El estado del Hold permanece en `EXPIRED`.

---

### TC-HU6-04 — Inventario descontado permanentemente impide nueva disponibilidad

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU6-04 |
| **HU** | HU6 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@alto @edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold `ID-150` fue confirmado exitosamente (estado `CONFIRMED`).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar `GET /disponibilidad` para la misma habitación y fechas del Hold `ID-150` | Misma `habitacion_id`, `fecha_inicio`, `fecha_fin` que `ID-150` |
| 2 | Verificar que la habitación no aparece en los resultados | — |

**Resultado Esperado**
- HTTP 200 OK.
- La habitación confirmada NO aparece en el listado de disponibilidad.

---

## HU7 — Liberación Proactiva por Fallo de Pago

> **Riesgo ASD:** A — Alto | **SP:** 2 | **Tipo:** Funcional–Estado

---

### TC-HU7-01 — Pago DECLINED cambia Hold a RELEASED y libera la habitación

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU7-01 |
| **HU** | HU7 |
| **Tipo** | Happy path |
| **ASD** | A — Alto |
| **Tags** | `@smoke @alto @happy-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold activo en estado `PENDING` para habitación `305`.
- Pasarela de pagos configurada para retornar `DECLINED`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Enviar señal de pago con estado `DECLINED` | `hold_id`, `estado_pasarela: 'DECLINED'` |
| 2 | Consultar el estado del Hold inmediatamente en la base de datos | — |
| 3 | Ejecutar `GET /disponibilidad` para las fechas afectadas | Fechas del Hold |

**Resultado Esperado**
- El Hold cambia a estado `RELEASED` de forma inmediata.
- La habitación `305` vuelve a aparecer en los resultados de disponibilidad.

---

### TC-HU7-02 — Señal de rechazo tardía no modifica Hold ya CONFIRMED

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU7-02 |
| **HU** | HU7 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@alto @edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold `ID-200` en estado `CONFIRMED` (pago ya aprobado previamente).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Enviar señal de pago tardía con estado `DECLINED` para el Hold `ID-200` | `hold_id: 'ID-200'`, `estado_pasarela: 'DECLINED'` |
| 2 | Consultar el estado del Hold `ID-200` en la base de datos | — |

**Resultado Esperado**
- El Hold `ID-200` permanece en estado `CONFIRMED`.
- El sistema ignora la señal tardía o retorna error controlado sin modificar datos.

---

### TC-HU7-03 — Múltiples señales de rechazo no generan estados inconsistentes

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU7-03 |
| **HU** | HU7 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@alto @edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold `ID-201` ya fue liberado (estado `RELEASED`).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Enviar segunda señal `DECLINED` para el Hold `ID-201` | `hold_id: 'ID-201'`, `estado_pasarela: 'DECLINED'` |
| 2 | Consultar el estado del Hold `ID-201` | — |
| 3 | Verificar el conteo de registros de liberación para ese Hold | — |

**Resultado Esperado**
- El estado del Hold permanece en `RELEASED`.
- No se crean registros duplicados de liberación en la base de datos.

---

## HU8 — Expiración Automática de Bloqueos (Worker)

> **Riesgo ASD:** A — Alto | **SP:** 3 | **Tipo:** Worker–Async

---

### TC-HU8-01 — Worker expira Hold con más de 10 minutos sin pago

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU8-01 |
| **HU** | HU8 |
| **Tipo** | Happy path |
| **ASD** | A — Alto |
| **Tags** | `@smoke @alto @worker` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold en estado `PENDING` con `creado_en = ahora - 11 minutos` (mock de tiempo).
- Worker de expiración disponible para ejecución manual.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Crear Hold con timestamp de creación simulado a 11 minutos atrás | `creado_en: T0 - 11min`, `estado: PENDING` |
| 2 | Ejecutar el proceso Worker de limpieza | — |
| 3 | Consultar el estado del Hold en la base de datos | — |
| 4 | Consultar disponibilidad de la habitación afectada | — |

**Resultado Esperado**
- El Hold cambia a estado `EXPIRED`.
- La habitación asociada aparece como disponible en el buscador.

---

### TC-HU8-02 — Worker NO expira Hold con menos de 10 minutos

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU8-02 |
| **HU** | HU8 |
| **Tipo** | Negativo |
| **ASD** | A — Alto |
| **Tags** | `@alto @negativo` |
| **Estado** | Pendiente |

**Precondiciones**
- Hold en estado `PENDING` con `creado_en = ahora - 8 minutos` (dentro del tiempo válido).

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Crear Hold con timestamp simulado a 8 minutos atrás | `creado_en: T0 - 8min`, `estado: PENDING` |
| 2 | Ejecutar el Worker de limpieza | — |
| 3 | Consultar el estado del Hold en la base de datos | — |

**Resultado Esperado**
- El Hold permanece en estado `PENDING`.
- La habitación asociada NO queda disponible (sigue bloqueada).

---

### TC-HU8-03 — Worker procesa múltiples Holds expirados en una sola ejecución

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU8-03 |
| **HU** | HU8 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@alto @edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- 5 Holds en estado `PENDING` con antigüedad mayor a 10 minutos cada uno, sobre habitaciones distintas.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Crear 5 Holds con timestamps simulados a más de 10 minutos atrás | `creado_en: T0 - 11min` para cada uno |
| 2 | Ejecutar el Worker de limpieza | — |
| 3 | Consultar los estados de los 5 Holds en la base de datos | — |
| 4 | Consultar disponibilidad de las 5 habitaciones afectadas | — |

**Resultado Esperado**
- Los 5 Holds cambian a estado `EXPIRED` en una sola ejecución del Worker.
- Las 5 habitaciones aparecen como disponibles para nuevas búsquedas.

---

### TC-HU8-04 — Worker no modifica Holds en estado CONFIRMED o RELEASED

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU8-04 |
| **HU** | HU8 |
| **Tipo** | Edge case |
| **ASD** | A — Alto |
| **Tags** | `@alto @edge-case` |
| **Estado** | Pendiente |

**Precondiciones**
- Existen Holds con estado `CONFIRMED` y `RELEASED` con más de 10 minutos de antigüedad.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Registrar los estados actuales de los Holds `CONFIRMED` y `RELEASED` | — |
| 2 | Ejecutar el Worker de limpieza | — |
| 3 | Consultar los estados de esos Holds en la base de datos | — |

**Resultado Esperado**
- Los Holds en estado `CONFIRMED` permanecen en `CONFIRMED`.
- Los Holds en estado `RELEASED` permanecen en `RELEASED`.
- Ningún dato de reserva confirmada resulta modificado.

---

## HU11 — Validación de Integridad de Fechas

> **Riesgo ASD:** S — Medio | **SP:** 3 | **Tipo:** Validación

---

### TC-HU11-01 — Fecha de salida anterior a la de entrada es rechazada

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU11-01 |
| **HU** | HU11 |
| **Tipo** | Error path |
| **ASD** | S — Medio |
| **Tags** | `@smoke @medio @error-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Ninguna. Validación aplicable en cualquier estado del sistema.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar solicitud de reserva con fechas incoherentes | `check_in: '2026-10-20'`, `check_out: '2026-10-18'` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar el mensaje de error en la respuesta | — |
| 4 | Verificar que no se creó ningún Hold en la base de datos | — |

**Resultado Esperado**
- HTTP 400 Bad Request.
- Mensaje de error: `"Fechas inválidas: la salida debe ser posterior a la entrada"`.
- No existe ningún registro de Hold para estas fechas.

---

### TC-HU11-02 — Fecha de salida igual a la de entrada es rechazada (mínimo 1 noche)

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU11-02 |
| **HU** | HU11 |
| **Tipo** | Error path |
| **ASD** | S — Medio |
| **Tags** | `@medio @error-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Ninguna.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar solicitud de reserva con misma fecha de entrada y salida | `check_in: '2026-10-20'`, `check_out: '2026-10-20'` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar el mensaje de error en la respuesta | — |
| 4 | Verificar que no se creó ningún Hold | — |

**Resultado Esperado**
- HTTP 400 Bad Request.
- Mensaje de error: `"La estancia mínima es de 1 noche"`.
- No existe ningún registro de Hold para estas fechas.

---

### TC-HU11-03 — Fecha de entrada en el pasado es rechazada

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU11-03 |
| **HU** | HU11 |
| **Tipo** | Error path |
| **ASD** | S — Medio |
| **Tags** | `@medio @error-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Fecha actual del sistema: `2026-03-24`.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar solicitud de reserva con fecha de entrada en el pasado | `check_in: '2025-01-01'`, `check_out: '2025-01-03'` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar el mensaje de error en la respuesta | — |
| 4 | Verificar que no se creó ningún Hold | — |

**Resultado Esperado**
- HTTP 400 Bad Request.
- Mensaje de error: `"La fecha de entrada no puede ser en el pasado"`.
- No existe ningún registro de Hold para estas fechas.

---

### TC-HU11-04 — Fechas coherentes y futuras son aceptadas

| Campo | Detalle |
|---|---|
| **TC-ID** | TC-HU11-04 |
| **HU** | HU11 |
| **Tipo** | Happy path |
| **ASD** | S — Medio |
| **Tags** | `@medio @happy-path` |
| **Estado** | Pendiente |

**Precondiciones**
- Habitación disponible para el rango solicitado.
- Usuario autenticado con token válido.

**Pasos**

| # | Acción | Dato de entrada |
|---|---|---|
| 1 | Ejecutar solicitud de reserva con fechas válidas y futuras | `check_in: '2026-10-20'`, `check_out: '2026-10-25'` |
| 2 | Verificar el código de respuesta HTTP | — |
| 3 | Verificar que el flujo continúa al siguiente paso del checkout | — |

**Resultado Esperado**
- HTTP 200 OK o 201 Created.
- No se retorna error de validación de fechas.
- El flujo avanza correctamente al siguiente paso (creación de Hold o similar).

---

## Resumen de Cobertura

| HU | Nombre | Casos | Tipo de Prueba | ASD |
|---|---|---|---|---|
| HU0 | Configuración del Ecosistema de Datos | TC-HU0-01, TC-HU0-02, TC-HU0-03 | Infraestructura / DB | A |
| HU1 | Seeder de Inventario Inicial | TC-HU1-01, TC-HU1-02, TC-HU1-03 | Funcional / Infraestructura | D |
| HU2 | Consulta de Disponibilidad Consistente | TC-HU2-01 al TC-HU2-07 | API / Edge case | A |
| HU3 | Bloqueo Atómico de Checkout (Hold) | TC-HU3-01 al TC-HU3-04 | Funcional / Concurrencia / Seguridad | A |
| HU5 | Procesamiento de Pago Idempotente | TC-HU5-01 al TC-HU5-04 | Funcional / Seguridad | A |
| HU6 | Confirmación Definitiva de Reserva | TC-HU6-01 al TC-HU6-04 | Funcional / Estado | A |
| HU7 | Liberación Proactiva por Fallo de Pago | TC-HU7-01 al TC-HU7-03 | Funcional / Estado | A |
| HU8 | Expiración Automática de Bloqueos (Worker) | TC-HU8-01 al TC-HU8-04 | Worker / Async | A |
| HU11 | Validación de Integridad de Fechas | TC-HU11-01 al TC-HU11-04 | Validación | S |
| **Total** | | **30 casos** | | |

---

## Matriz de Trazabilidad TC → Escenario Gherkin

| TC-ID | Escenario Gherkin (Test Plan §6) | Estado |
|---|---|---|
| TC-HU0-01 | Bloqueo de fila impide modificación concurrente (SELECT FOR UPDATE) | Pendiente |
| TC-HU0-02 | ROLLBACK mantiene consistencia ante error en transacción | Pendiente |
| TC-HU0-03 | Pipeline CI/CD ejecuta migraciones en entorno de pruebas sin afectar datos | Pendiente |
| TC-HU1-01 | Carga exitosa de datos maestros | Pendiente |
| TC-HU1-02 | Seeder es idempotente al ejecutarse dos veces | Pendiente |
| TC-HU1-03 | Seeder falla con error claro si la base de datos está offline | Pendiente |
| TC-HU2-01 | Habitación sin bloqueo ni reserva aparece en resultados de disponibilidad | Pendiente |
| TC-HU2-02 | Hold en fechas distintas no afecta disponibilidad para otras fechas | Pendiente |
| TC-HU2-03 | Exclusión de habitación con bloqueo temporal activo (Hold) | Pendiente |
| TC-HU2-04 | Habitación con reserva confirmada activa es excluida de resultados | Pendiente |
| TC-HU2-05 | Solapamiento parcial de fechas excluye la habitación | Pendiente |
| TC-HU2-06 | Hold expirado no afecta disponibilidad actual | Pendiente |
| TC-HU2-07 | Sin habitaciones disponibles para las fechas solicitadas muestra mensaje informativo | Pendiente |
| TC-HU3-01 | Bloqueo exitoso genera un Hold en estado PENDING con 10 minutos de duración | Pendiente |
| TC-HU3-02 | Prevención de colisión de reserva (Race Condition) | Pendiente |
| TC-HU3-03 | Intento de bloqueo sobre habitación ya bloqueada es rechazado | Pendiente |
| TC-HU3-04 | Usuario no autenticado no puede crear un Hold | Pendiente |
| TC-HU5-01 | Reintento de pago con la misma clave de idempotencia retorna el éxito anterior | Pendiente |
| TC-HU5-02 | Reintento de pago rechazado con la misma clave retorna el rechazo original | Pendiente |
| TC-HU5-03 | Nueva clave de idempotencia permite un nuevo intento de pago | Pendiente |
| TC-HU5-04 | Clave de idempotencia no puede ser reutilizada para un Hold diferente | Pendiente |
| TC-HU6-01 | Transición de estado PENDING a CONFIRMED tras pago exitoso | Pendiente |
| TC-HU6-02 | Pago fallido no modifica el estado PENDING del Hold | Pendiente |
| TC-HU6-03 | Intento de confirmar un Hold ya expirado es rechazado | Pendiente |
| TC-HU6-04 | Inventario descontado permanentemente impide nueva disponibilidad | Pendiente |
| TC-HU7-01 | Pago declinado por el banco cambia el Hold a RELEASED y libera la habitación | Pendiente |
| TC-HU7-02 | Señal de rechazo tardía no modifica Hold ya CONFIRMED | Pendiente |
| TC-HU7-03 | Múltiples señales de rechazo no generan registros inconsistentes | Pendiente |
| TC-HU8-01 | Worker expira un Hold con más de 10 minutos sin pago | Pendiente |
| TC-HU8-02 | Worker NO expira un Hold con menos de 10 minutos | Pendiente |
| TC-HU8-03 | Worker procesa múltiples Holds expirados en una sola ejecución | Pendiente |
| TC-HU8-04 | Worker no modifica Holds en estado CONFIRMED o RELEASED | Pendiente |
| TC-HU11-01 | Fecha de salida anterior a la de entrada es rechazada | Pendiente |
| TC-HU11-02 | Fecha de salida igual a la de entrada es rechazada (mínimo 1 noche) | Pendiente |
| TC-HU11-03 | Fecha de entrada en el pasado es rechazada | Pendiente |
| TC-HU11-04 | Fechas coherentes y futuras son procesadas exitosamente | Pendiente |

---

*Generado por QA Agent (ASDD) — 2026-03-24*  
*Referencia: `travel-hotel-test-plan.md` v1.0*  
*Actualizar columna "Estado" con los resultados reales tras cada ciclo de ejecución.*
