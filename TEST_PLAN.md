# Plan de Pruebas — Travel Hotel: Motor de Reservas

**Proyecto:** Travel Hotel — Motor de Reservas  
**Versión:** 1.0.0-MVP  
**Fecha:** 2026-03-24  
**Autor:** QA  
**HUs en Alcance:** HU0–HU3, HU5–HU8, HU11  
**Story Points Totales:** 34  

---

## 1. Contexto

Este plan cubre la estrategia de calidad integral para el sistema **Travel Hotel**, un motor transaccional cuyo núcleo opera sobre tres invariantes de negocio críticas:

1. **Consistencia de inventario** — ninguna habitación puede quedar bloqueada o reservada dos veces para el mismo rango de fechas.
2. **Atomicidad del bloqueo (Hold)** — el proceso de checkout garantiza exclusividad por 10 minutos mediante `SELECT FOR UPDATE`.
3. **Idempotencia de pagos** — reintentos de red no generan cargos duplicados bajo ninguna circunstancia.


| Clasificación | Significado | Acción |
|---|---|---|
| **A — Alto** | Fallo tiene impacto directo en negocio o integridad de datos | Obligatorio — bloquea el release |
| **S — Medio** | Fallo degrada experiencia o genera deuda técnica | Recomendado |
| **D — Bajo** | Fallo no impacta flujo principal | Opcional |

## 2. Alcance

### 2.1 Historias Post-MVP excluidas de este plan

| HU | Nombre | Motivo de exclusión |
|---|---|---|
| HU4 | Persistencia del Timer de Reserva | Mejora estética de UI — el backend controla el tiempo vía HU8 |
| HU9 | Resolución de Carrera Pago-Expiración | Ajuste de milisegundos — se incorpora en fase de optimización post-lanzamiento |
| HU10 | Protección contra Bloqueos Masivos (Rate Limiting) | Seguridad avanzada — no bloquea la funcionalidad core del hotel |

---

### 2.2 Historias incluidas en este plan

| HU | Nombre | SP | Tipo | Riesgo ASD |
|---|---|---|---|---|
| HU0 | Configuración del Ecosistema de Datos | 5 | Infraestructura | **A — Alto** |
| HU1 | Seeder de Inventario Inicial | 2 | Infraestructura | D — Bajo |
| HU2 | Consulta de Disponibilidad Consistente | 3 | Funcional–API | **A — Alto** |
| HU3 | Bloqueo Atómico de Checkout (Hold) | 8 | Funcional–Concurrencia | **A — Alto** |
| HU5 | Procesamiento de Pago Idempotente | 5 | Funcional–Pago | **A — Alto** |
| HU6 | Confirmación Definitiva de Reserva | 3 | Funcional–Estado | **A — Alto** |
| HU7 | Liberación Proactiva por Fallo de Pago | 2 | Funcional–Estado | **A — Alto** |
| HU8 | Expiración Automática de Bloqueos (Worker) | 3 | Worker–Async | **A — Alto** |
| HU11 | Validación de Integridad de Fechas | 3 | Validación | S — Medio |

---

## 3. Estrategia de Pruebas

### 3.1 Niveles de Prueba

| Nivel | Descripción | HUs Principales |
|---|---|---|
| **Unitario** | Lógica de negocio aislada (servicios, validadores) | HU11, HU6, HU7 |
| **Integración** | DB + Servicios + Endpoints de API | HU0, HU2, HU5, HU8 |
| **E2E** | Flujos completos: viajero → confirmación | HU3, HU6 |
| **Concurrencia** | Race conditions y bloqueos atómicos | HU3 |
| **Seguridad** | Inyección de entradas | HU2, HU3 |
| **Regresión** | Suite crítica tras cada release | HU2, HU3, HU5, HU6, HU7 |

### 3.2 Tipos de Prueba

| Tipo | Herramienta Propuesta | Condición de Ejecución |
|---|---|---|
| Funcional / API | JEST | Siempre — obligatorio |
| Concurrencia | pytest-asyncio + threading | HU3 — obligatorio |
| Seguridad (OWASP) | OWASP ZAP + pruebas manuales | HU3 — obligatorio |
| Performance / Load | k6 | Solo si hay SLAs definidos en spec |

---

## 4. Criterios de Entrada y Salida (DoR / DoD)

### 4.1 Definition of Ready (DoR)

- [ ] Historia de Usuario con criterios de aceptación Gherkin aprobados
- [ ] Ambiente PostgreSQL de prueba disponible y aislado (no producción)
- [ ] Datos maestros cargados via seeder (HU1 ejecutada como precondición)
- [ ] Contrato de API documentado para los endpoints relacionados
- [ ] Casos de prueba revisados y firmados por el equipo

### 4.2 Definition of Done (DoD)

- [ ] 100% de escenarios clasificados **Alto** ejecutados sin fallo bloqueante
- [ ] Escenario de concurrencia (HU3) verificado con mínimo 2 hilos simultáneos
- [ ] Matriz de trazabilidad AC → Escenario → Resultado actualizada
- [ ] Defectos críticos cerrados o con workaround documentado
- [ ] Resultados y evidencia almacenados en `docs/output/qa/`

---

## 3. Matriz de Riesgo

| HU | Riesgo Técnico | Riesgo de Negocio | Clasificación | Acción Obligatoria |
|---|---|---|---|---|
| HU0 | Fallo de transacciones ACID | Inconsistencia de datos en todo el sistema | **A** | Test de DB isolation + CI/CD gate |
| HU2 | Dirty read en consulta | Viajero ve habitaciones no disponibles | **A** | Test de aislamiento de lectura (READ COMMITTED) |
| HU3 | Race condition en Hold | Doble reserva — pérdida de confianza del cliente | **A** | Test de concurrencia (2+ hilos simultáneos) |
| HU5 | Pago duplicado | Cargo doble — pérdida financiera / fraude | **A** | Test de idempotencia + simulación de reintento de red |
| HU6 | Transición de estado incorrecta | Reserva sin confirmación o pago sin reserva | **A** | Test de máquina de estados (happy + sad path) |
| HU7 | Habitación bloqueada tras pago fallido | Inventario perdido permanentemente | **A** | Test de liberación inmediata tras DECLINED |
| HU8 | Worker no libera a tiempo / libera de más | Carritos abandonados retienen inventario | **A** | Test de Worker con mock de tiempo |
| HU11 | Fechas incoherentes aceptadas | Error lógico en inventario y precios | **S** | Test de validación de rango de fechas |
| HU1 | Seeder falla | Bloquea ejecución de pruebas funcionales | **D** | Verificación como precondición del smoke test |

---

## 6. Escenarios Gherkin por Historia de Usuario

---

### HU0 — Configuración del Ecosistema de Datos

**Riesgo:** A — Alto | **Story Points:** 5

| # | Tipo | Flujo | Tag |
|---|------|-------|-----|
| 1 | Happy path | SELECT FOR UPDATE bloquea fila ante transacción concurrente | `@smoke @alto` |
| 2 | Error path | ROLLBACK mantiene integridad ante error en transacción | `@alto` |
| 3 | Infra | Pipeline CI/CD aplica migraciones sin afectar datos de prueba | `@alto @infraestructura` |

```gherkin
#language: es
Característica: Configuración del Ecosistema de Datos
  Como equipo de ingeniería
  Quiero un entorno de persistencia con transacciones ACID
  Para garantizar que el motor de reservas opere con consistencia y seguridad

  @smoke @alto @infraestructura
  Escenario: Bloqueo de fila impide modificación concurrente (SELECT FOR UPDATE)
    Dado que la base de datos PostgreSQL está inicializada
    Y la habitación "101" existe en la tabla de habitaciones
    Cuando la transacción A ejecuta SELECT FOR UPDATE sobre la habitación "101"
    Y la transacción B intenta actualizar la habitación "101" antes de que A haga COMMIT
    Entonces la transacción B debe quedar en espera
    Y la modificación de B se aplica únicamente DESPUÉS de que A finalice con COMMIT

  @alto @infraestructura
  Escenario: ROLLBACK mantiene consistencia ante error en transacción
    Dado que una transacción inicia una operación de reserva sobre la habitación "102"
    Cuando ocurre un error durante la operación (timeout / excepción)
    Entonces la transacción hace ROLLBACK automáticamente
    Y la habitación "102" no presenta ningún cambio de estado en la base de datos

  @alto @infraestructura
  Escenario: Pipeline CI/CD ejecuta migraciones en entorno de pruebas sin afectar datos
    Dado que el pipeline CI/CD está configurado con una base de datos de prueba separada
    Cuando se despliega una versión con migraciones pendientes
    Entonces las migraciones se aplican sin error
    Y los datos pre-existentes del entorno de pruebas no se ven afectados
```

---

### HU1 — Seeder de Inventario Inicial

**Riesgo:** D — Bajo | **Story Points:** 2

| # | Tipo | Flujo | Tag |
|---|------|-------|-----|
| 1 | Happy path | Seeder carga tablas con registros válidos | `@smoke @bajo` |
| 2 | Edge case | Seeder es idempotente (sin duplicados en segunda ejecución) | `@bajo` |
| 3 | Error path | Seeder falla con mensaje claro si DB está offline | `@bajo` |

```gherkin
#language: es
Característica: Seeder de Inventario Inicial
  Como equipo de desarrollo
  Quiero una carga automática de hoteles y habitaciones
  Para realizar pruebas funcionales sin ingresos manuales

  @smoke @bajo
  Escenario: Carga exitosa de datos maestros
    Dado que el entorno de base de datos está vacío
    Cuando se ejecuta el script de seeder
    Entonces la tabla de Hoteles debe contener al menos 1 registro válido
    Y la tabla de Habitaciones debe contener al menos 3 registros válidos con tipo y capacidad

  @bajo @edge-case
  Escenario: Seeder es idempotente al ejecutarse dos veces
    Dado que el seeder ya fue ejecutado una vez con datos válidos
    Cuando se ejecuta el seeder por segunda vez
    Entonces no se crean registros duplicados en Hoteles ni en Habitaciones
    Y el conteo de registros permanece igual al de la primera ejecución

  @bajo @error-path
  Escenario: Seeder falla con error claro si la base de datos está offline
    Dado que la base de datos está offline o inaccesible
    Cuando se intenta ejecutar el script de seeder
    Entonces el proceso termina con un mensaje de error descriptivo
    Y no deja la base de datos en estado inconsistente
```

---

### HU2 — Consulta de Disponibilidad Consistente

**Riesgo:** A — Alto | **Story Points:** 3

| # | Tipo | Flujo | Tag |
|---|------|-------|-----|
| 1 | Happy path | Habitación sin bloqueo aparece en resultados | `@smoke @alto` |
| 2 | Happy path | Hold en fechas distintas no afecta búsqueda actual | `@alto` |
| 3 | Error path | Habitación con Hold activo excluida de resultados | `@smoke @alto` |
| 4 | Error path | Habitación con reserva confirmada excluida | `@alto` |
| 5 | Edge case | Solapamiento parcial de fechas excluye habitación | `@edge-case` |
| 6 | Edge case | Hold expirado no afecta disponibilidad | `@edge-case` |
| 7 | Edge case | Sin habitaciones disponibles muestra mensaje informativo | `@edge-case` |

```gherkin
#language: es
Característica: Consulta de Disponibilidad Consistente
  Como viajero
  Quiero ver solo las habitaciones que no tienen reservas ni bloqueos activos
  Para tomar una decisión basada en la disponibilidad real del hotel

  Antecedentes:
    Dado que el sistema tiene registradas las siguientes habitaciones:
      | habitacion | tipo       | capacidad |
      | 101        | doble      | 2         |
      | 102        | individual | 1         |
      | 103        | suite      | 4         |

  @smoke @alto @happy-path
  Escenario: Habitación sin bloqueo ni reserva aparece en resultados de disponibilidad
    Dado que la habitación "102" no tiene ningún bloqueo ni reserva activa
    Cuando el viajero busca disponibilidad del "2026-04-10" al "2026-04-12"
    Entonces el sistema muestra la habitación "102" en los resultados

  @alto @happy-path
  Escenario: Hold en fechas distintas no afecta disponibilidad para otras fechas
    Dado que la habitación "103" tiene un Hold activo del "2026-04-20" al "2026-04-22"
    Cuando el viajero busca disponibilidad del "2026-04-10" al "2026-04-12"
    Entonces el sistema muestra la habitación "103" en los resultados de disponibilidad

  @smoke @alto @error-path
  Escenario: Exclusión de habitación con bloqueo temporal activo (Hold)
    Dado que la habitación "101" tiene un Hold activo del "2026-04-10" al "2026-04-12"
    Cuando el viajero busca disponibilidad del "2026-04-10" al "2026-04-12"
    Entonces el sistema NO muestra la habitación "101" en los resultados
    Y el sistema muestra al menos una habitación alternativa disponible

  @alto @error-path
  Escenario: Habitación con reserva confirmada activa es excluida de resultados
    Dado que la habitación "103" tiene una reserva confirmada del "2026-04-10" al "2026-04-12"
    Cuando el viajero busca disponibilidad del "2026-04-10" al "2026-04-12"
    Entonces el sistema NO muestra la habitación "103" en los resultados

  @edge-case
  Escenario: Solapamiento parcial de fechas excluye la habitación
    Dado que la habitación "102" tiene un Hold activo del "2026-04-11" al "2026-04-14"
    Cuando el viajero busca disponibilidad del "2026-04-10" al "2026-04-12"
    Entonces el sistema NO muestra la habitación "102" en los resultados

  @edge-case
  Escenario: Hold expirado no afecta disponibilidad actual
    Dado que la habitación "101" tuvo un Hold que expiró el "2026-03-01"
    Y no tiene bloqueos ni reservas activas en la actualidad
    Cuando el viajero busca disponibilidad del "2026-04-10" al "2026-04-12"
    Entonces el sistema muestra la habitación "101" en los resultados

  @edge-case
  Escenario: Sin habitaciones disponibles para las fechas solicitadas muestra mensaje informativo
    Dado que todas las habitaciones tienen bloqueos o reservas activas del "2026-04-10" al "2026-04-12"
    Cuando el viajero busca disponibilidad del "2026-04-10" al "2026-04-12"
    Entonces el sistema muestra un mensaje indicando que no hay habitaciones disponibles
    Y no se muestran fichas de habitaciones en los resultados
```

---

### HU3 — Bloqueo Atómico de Checkout (Hold)

**Riesgo:** A — Alto | **Story Points:** 8

| # | Tipo | Flujo | Tag |
|---|------|-------|-----|
| 1 | Happy path | Bloqueo exitoso genera Hold con 10 min de duración | `@smoke @alto` |
| 2 | Concurrencia | Solo el primer usuario obtiene el bloqueo | `@smoke @alto @concurrencia` |
| 3 | Error path | Intento de Hold sobre habitación ya bloqueada | `@alto` |
| 4 | Seguridad | Usuario no autenticado no puede crear Hold | `@alto @seguridad` |

```gherkin
#language: es
Característica: Bloqueo Atómico de Checkout (Hold)
  Como viajero
  Quiero que la habitación se aparte exclusivamente para mí por 10 minutos al seleccionarla
  Para completar mis datos de pago sin riesgo de que alguien más la reserve

  @smoke @alto @happy-path
  Escenario: Bloqueo exitoso genera un Hold en estado PENDING con 10 minutos de duración
    Dado que la habitación "202" está disponible
    Cuando el viajero autenticado solicita bloquear la habitación "202" del "2026-04-10" al "2026-04-12"
    Entonces el sistema crea un Hold con estado "PENDING"
    Y el Hold tiene una fecha de expiración de 10 minutos desde el momento actual

  @smoke @alto @concurrencia
  Escenario: Prevención de colisión de reserva (Race Condition)
    Dado que la habitación "202" está disponible
    Cuando el usuario A y el usuario B intentan bloquear la habitación "202" simultáneamente
    Entonces el sistema confirma el bloqueo al primer usuario que completa la transacción
    Y rechaza la solicitud del segundo usuario con el mensaje "Habitación no disponible"
    Y solo existe UN Hold activo en la base de datos para la habitación "202"

  @alto @error-path
  Escenario: Intento de bloqueo sobre habitación ya bloqueada es rechazado
    Dado que la habitación "202" ya tiene un Hold activo del usuario A
    Cuando el usuario B intenta crear un Hold para la habitación "202" en las mismas fechas
    Entonces el sistema responde con error "Habitación no disponible"
    Y el Hold del usuario A permanece sin modificación

  @alto @seguridad
  Escenario: Usuario no autenticado no puede crear un Hold
    Dado que el usuario no está autenticado (sin token válido)
    Cuando intenta crear un bloqueo para la habitación "202"
    Entonces el sistema responde con error de autenticación 401
    Y no se crea ningún registro de Hold en la base de datos
```

---

### HU5 — Procesamiento de Pago Idempotente

**Riesgo:** A — Alto | **Story Points:** 5

| # | Tipo | Flujo | Tag |
|---|------|-------|-----|
| 1 | Happy path | Reintento con misma clave retorna éxito original sin nuevo cobro | `@smoke @alto` |
| 2 | Error path | Reintento de pago rechazado retorna rechazo original | `@alto` |
| 3 | Edge case | Nueva clave permite nuevo intento de pago | `@alto` |
| 4 | Seguridad | Clave no puede reutilizarse en Hold diferente | `@alto @seguridad` |

```gherkin
#language: es
Característica: Procesamiento de Pago Idempotente
  Como viajero
  Quiero que mi pago se procese una sola vez ante reintentos de red
  Para evitar cargos duplicados en mi cuenta

  @smoke @alto @happy-path
  Escenario: Reintento de pago con la misma clave de idempotencia retorna el éxito anterior
    Dado que un pago fue procesado exitosamente para el Hold "ID-99"
    Y la clave de idempotencia usada fue "KEY-ABC"
    Cuando el sistema recibe una segunda solicitud de pago con la misma clave "KEY-ABC"
    Entonces el sistema retorna el resultado de éxito de la transacción original
    Y no se realiza ningún nuevo cobro al usuario

  @alto @error-path
  Escenario: Reintento de pago rechazado con la misma clave retorna el rechazo original
    Dado que un pago fue rechazado para el Hold "ID-100" con la clave "KEY-DEF"
    Cuando el sistema recibe una segunda solicitud con la misma clave "KEY-DEF"
    Entonces el sistema retorna el resultado de rechazo original
    Y no se intenta procesar el pago nuevamente

  @alto @edge-case
  Escenario: Nueva clave de idempotencia permite un nuevo intento de pago
    Dado que un pago fue rechazado para el Hold "ID-101" con la clave "KEY-GHI"
    Cuando el sistema recibe una solicitud con la nueva clave "KEY-XYZ"
    Entonces el sistema procesa la solicitud de pago como una nueva transacción
    Y el resultado puede ser éxito o rechazo según el estado actual

  @alto @seguridad
  Escenario: Clave de idempotencia no puede ser reutilizada para un Hold diferente
    Dado que la clave "KEY-ABC" fue usada exitosamente para el Hold "ID-99"
    Cuando se intenta usar la misma clave "KEY-ABC" para el Hold "ID-200"
    Entonces el sistema rechaza la operación
    Y retorna error "Clave de idempotencia inválida para este Hold"
```

---

### HU6 — Confirmación Definitiva de Reserva

**Riesgo:** A — Alto | **Story Points:** 3

| # | Tipo | Flujo | Tag |
|---|------|-------|-----|
| 1 | Happy path | Hold pasa de PENDING a CONFIRMED tras pago exitoso | `@smoke @alto` |
| 2 | Error path | Pago fallido no modifica el estado del Hold | `@alto` |
| 3 | Edge case | Intento de confirmar Hold expirado es rechazado | `@alto` |
| 4 | Edge case | Inventario descontado permanentemente tras confirmación | `@alto` |

```gherkin
#language: es
Característica: Confirmación Definitiva de Reserva
  Como viajero
  Quiero que mi reserva pase de Bloqueada a Confirmada tras el pago
  Para recibir mi garantía de estancia

  @smoke @alto @happy-path
  Escenario: Transición de estado PENDING a CONFIRMED tras pago exitoso
    Dado que existe un Hold en estado "PENDING" para la habitación "305" del "2026-04-10" al "2026-04-12"
    Cuando el procesador de pagos retorna el estado "SUCCESS"
    Entonces el Hold cambia al estado "CONFIRMED"
    Y la habitación "305" queda marcada como no disponible permanentemente para esas fechas

  @alto @error-path
  Escenario: Pago fallido no modifica el estado PENDING del Hold
    Dado que existe un Hold en estado "PENDING" para la habitación "305"
    Cuando el procesador de pagos retorna el estado "FAILED"
    Entonces el Hold NO cambia a estado "CONFIRMED"
    Y la habitación "305" no queda bloqueada permanentemente

  @alto @edge-case
  Escenario: Intento de confirmar un Hold ya expirado es rechazado
    Dado que un Hold fue marcado como "EXPIRED" por el Worker antes de recibir confirmación de pago
    Cuando el sistema intenta cambiar el estado del Hold a "CONFIRMED"
    Entonces el sistema rechaza la transición con error "Hold expirado — no se puede confirmar"
    Y el estado permanece en "EXPIRED"

  @alto @edge-case
  Escenario: Inventario descontado permanentemente impide nueva disponibilidad
    Dado que el Hold "ID-150" fue confirmado exitosamente
    Cuando otro viajero busca disponibilidad para la misma habitación y las mismas fechas
    Entonces el sistema NO muestra la habitación como disponible
```

---

### HU7 — Liberación Proactiva por Fallo de Pago

**Riesgo:** A — Alto | **Story Points:** 2

| # | Tipo | Flujo | Tag |
|---|------|-------|-----|
| 1 | Happy path | Pago DECLINED cambia Hold a RELEASED inmediatamente | `@smoke @alto` |
| 2 | Edge case | Señal de rechazo tardía no afecta Hold ya CONFIRMED | `@alto` |
| 3 | Edge case | Múltiples señales de rechazo no generan estado inconsistente | `@alto` |

```gherkin
#language: es
Característica: Liberación Proactiva por Fallo de Pago
  Como sistema
  Quiero liberar la habitación de inmediato si el pago es rechazado
  Para que el hotel no pierda oportunidades de venta con otros clientes

  @smoke @alto @happy-path
  Escenario: Pago declinado por el banco cambia el Hold a RELEASED y libera la habitación
    Dado que existe un Hold activo en estado "PENDING" para la habitación "305"
    Cuando la pasarela de pagos retorna el estado "DECLINED"
    Entonces el sistema cambia el estado del Hold a "RELEASED" inmediatamente
    Y la habitación "305" vuelve a aparecer como disponible en el buscador

  @alto @edge-case
  Escenario: Señal de rechazo tardía no modifica Hold ya CONFIRMED
    Dado que el Hold "ID-200" está en estado "CONFIRMED" (pago ya aprobado)
    Cuando se recibe una segunda señal de pago con estado "DECLINED" (señal duplicada o tardía)
    Entonces el sistema NO cambia el estado del Hold
    Y la reserva permanece en estado "CONFIRMED"

  @alto @edge-case
  Escenario: Múltiples señales de rechazo no generan registros inconsistentes
    Dado que el Hold "ID-201" ya fue liberado (estado "RELEASED")
    Cuando el sistema recibe una segunda señal de pago "DECLINED" para el mismo Hold
    Entonces el estado permanece en "RELEASED"
    Y no se crean registros duplicados de liberación en la base de datos
```

---

### HU8 — Expiración Automática de Bloqueos (Worker)

**Riesgo:** A — Alto | **Story Points:** 3

| # | Tipo | Flujo | Tag |
|---|------|-------|-----|
| 1 | Happy path | Worker expira Hold con más de 10 min sin pago | `@smoke @alto @worker` |
| 2 | Negativo | Worker NO expira Hold con menos de 10 min | `@alto` |
| 3 | Edge case | Worker procesa múltiples Holds expirados en una ejecución | `@alto` |
| 4 | Edge case | Worker no modifica Holds ya CONFIRMED o RELEASED | `@alto` |

```gherkin
#language: es
Característica: Expiración Automática de Bloqueos (Worker)
  Como administrador
  Quiero que el sistema libere automáticamente los bloqueos que superen los 10 minutos
  Para evitar que el inventario quede retenido por carritos abandonados

  @smoke @alto @worker
  Escenario: Worker expira un Hold con más de 10 minutos sin pago
    Dado que existe un Hold en estado "PENDING" creado hace 11 minutos
    Cuando el proceso de limpieza (Worker) se ejecuta
    Entonces el estado del Hold cambia a "EXPIRED"
    Y la habitación asociada queda libre para nuevas búsquedas

  @alto @negativo
  Escenario: Worker NO expira un Hold con menos de 10 minutos
    Dado que existe un Hold en estado "PENDING" creado hace 8 minutos
    Cuando el proceso de limpieza (Worker) se ejecuta
    Entonces el Hold permanece en estado "PENDING"
    Y la habitación asociada NO queda disponible

  @alto @edge-case
  Escenario: Worker procesa múltiples Holds expirados en una sola ejecución
    Dado que existen 5 Holds en estado "PENDING" con más de 10 minutos de antigüedad
    Cuando el Worker se ejecuta
    Entonces los 5 Holds cambian a estado "EXPIRED"
    Y las 5 habitaciones asociadas quedan disponibles para nuevas búsquedas

  @alto @edge-case
  Escenario: Worker no modifica Holds en estado CONFIRMED o RELEASED
    Dado que existen Holds con estado "CONFIRMED" y "RELEASED" con más de 10 minutos de antigüedad
    Cuando el Worker se ejecuta
    Entonces esos Holds NO cambian de estado
    Y los datos de reserva permanecen intactos
```

---

### HU11 — Validación de Integridad de Fechas

**Riesgo:** S — Medio | **Story Points:** 3

| # | Tipo | Flujo | Tag |
|---|------|-------|-----|
| 1 | Error path | Check-out anterior a Check-in es rechazado | `@smoke @medio` |
| 2 | Error path | Check-out igual a Check-in es rechazado (mínimo 1 noche) | `@medio` |
| 3 | Error path | Check-in en el pasado es rechazado | `@medio` |
| 4 | Happy path | Fechas coherentes y futuras son aceptadas | `@medio` |

```gherkin
#language: es
Característica: Validación de Integridad de Fechas
  Como sistema
  Quiero validar que las fechas de reserva sean lógicamente coherentes
  Para evitar errores lógicos en el inventario y en los cálculos de precio

  @smoke @medio @error-path
  Escenario: Fecha de salida anterior a la de entrada es rechazada
    Cuando el usuario intenta reservar con Check-in "2026-10-20" y Check-out "2026-10-18"
    Entonces el sistema rechaza la operación con el error "Fechas inválidas: la salida debe ser posterior a la entrada"
    Y no se crea ningún Hold en la base de datos

  @medio @error-path
  Escenario: Fecha de salida igual a la de entrada es rechazada (mínimo 1 noche)
    Cuando el usuario intenta reservar con Check-in "2026-10-20" y Check-out "2026-10-20"
    Entonces el sistema rechaza la operación con el error "La estancia mínima es de 1 noche"
    Y no se crea ningún Hold en la base de datos

  @medio @error-path
  Escenario: Fecha de entrada en el pasado es rechazada
    Cuando el usuario intenta reservar con Check-in "2025-01-01" y Check-out "2025-01-03"
    Entonces el sistema rechaza la operación con el error "La fecha de entrada no puede ser en el pasado"
    Y no se crea ningún Hold en la base de datos

  @medio @happy-path
  Escenario: Fechas coherentes y futuras son procesadas exitosamente
    Cuando el usuario intenta reservar con Check-in "2026-10-20" y Check-out "2026-10-25"
    Entonces el sistema procesa la solicitud exitosamente
    Y permite continuar al siguiente paso del flujo de checkout
```

---

## 7. Datos de Prueba Consolidados

| HU | Campo | Valor Válido | Valor Inválido | Valor Borde |
|---|---|---|---|---|
| HU2 | Fecha entrada | `2026-04-10` | `texto-libre` | Hoy (`2026-03-24`) |
| HU2 | Fecha salida | `2026-04-12` | `2026-04-08` (anterior) | Mañana (`2026-03-25`) |
| HU3 | Habitación Hold | `202` (disponible) | N/A | Habitación con Hold a punto de expirar |
| HU3 | Duración Hold | `10 min` | `< 1 min` | Exactamente `10 min` |
| HU5 | Clave idempotencia | `KEY-ABC` (única por Hold) | Misma clave en Hold diferente | Clave generada en el límite de tiempo |
| HU6 | Transición estado | `PENDING → CONFIRMED` | `EXPIRED → CONFIRMED` | `CONFIRMED` (no retrocede) |
| HU7 | Señal de pago | `DECLINED` | N/A | Señal tardía tras Hold CONFIRMED |
| HU8 | Antigüedad Hold | `11 min` (expira) | `8 min` (no expira) | Exactamente `10 min` |
| HU11 | Check-in / Check-out | `2026-10-20 / 2026-10-25` | `2026-10-20 / 2026-10-18` | Hoy / Mañana |

### Fixture de Hold de Referencia

```json
{
  "hold_id": "ID-99",
  "habitacion_id": "202",
  "estado": "PENDING",
  "clave_idempotencia": "KEY-ABC",
  "fecha_inicio": "2026-04-10",
  "fecha_fin": "2026-04-12",
  "expira_en": "2026-04-10T10:10:00Z",
  "ip_origen": "192.168.1.100",
  "creado_en": "2026-04-10T10:00:00Z"
}
```

### Fixture de Pago de Referencia

```json
{
  "pago_id": "PAY-001",
  "hold_id": "ID-99",
  "clave_idempotencia": "KEY-ABC",
  "monto": 250.00,
  "moneda": "USD",
  "estado_pasarela": "SUCCESS",
  "procesado_en": "2026-04-10T10:05:30Z"
}
```

---

## 8. Matriz de Trazabilidad: AC → Escenario → Cobertura

| HU | Criterio de Aceptación (Backlog) | Escenario Gherkin | Estado |
|---|---|---|---|
| HU0 | SELECT FOR UPDATE impide modificación concurrente | "Bloqueo de fila impide modificación concurrente" | ✅ Cubierto |
| HU1 | Seeder carga tablas con registros válidos | "Carga exitosa de datos maestros" | ✅ Cubierto |
| HU2 | Habitación con Hold activo NO aparece en resultados | "Exclusión de habitación con bloqueo temporal activo" | ✅ Cubierto |
| HU3 | Solo el primer usuario obtiene el bloqueo simultáneo | "Prevención de colisión de reserva (Race Condition)" | ✅ Cubierto |
| HU5 | Reintento con misma clave no genera nuevo cobro | "Reintento de pago con la misma clave de idempotencia" | ✅ Cubierto |
| HU6 | Hold pasa a CONFIRMED y descuenta inventario | "Transición de estado PENDING a CONFIRMED" | ✅ Cubierto |
| HU7 | Hold pasa a RELEASED inmediatamente tras DECLINED | "Pago declinado cambia el Hold a RELEASED y libera la habitación" | ✅ Cubierto |
| HU8 | Worker cambia Hold >10 min a EXPIRED | "Worker expira un Hold con más de 10 minutos sin pago" | ✅ Cubierto |
| HU11 | Check-out anterior a Check-in retorna error | "Fecha de salida anterior a la de entrada es rechazada" | ✅ Cubierto |

**Cobertura total:** 9/9 criterios de aceptación cubiertos con escenarios Gherkin.

---

## 9. Consideraciones de Performance

> Condición de ejecución: activar `/performance-analyzer` cuando se definan SLAs formales en la spec del proyecto.

### SLAs Propuestos para Referencia

| Endpoint | Métrica | Umbral P95 | TPS Mínimo |
|---|---|---|---|
| `GET /disponibilidad` | Latencia | < 500 ms | 100 TPS |
| `POST /hold` | Latencia | < 300 ms | 50 TPS (incluye `SELECT FOR UPDATE`) |
| `POST /pago` | Latencia | < 1000 ms | 30 TPS |
| Worker de expiración | Ciclo completo | < 5 segundos por lote | — |

### Pruebas de Carga Recomendadas

| Tipo | Descripción | Duración | HU Relacionada |
|---|---|---|---|
| Load Test | Carga nominal — 50 usuarios concurrentes en búsqueda y Hold | 15 min | HU2, HU3 |
| Spike Test | Pico de 200 usuarios en 30 segundos | 5 min | HU3 |
| Race Condition Test | 2+ hilos simultáneos sobre el mismo Hold | Instantáneo | HU3 |
| Soak Test | Carga sostenida durante 120 minutos (verificar Worker) | 120 min | HU8 |

---

## 10. Consideraciones de Seguridad (OWASP Top 10)

| Riesgo OWASP | HU Relacionada | Escenario de Prueba |
|---|---|---|
| A01 — Broken Access Control | HU3 | Hold solo creado por usuario autenticado (token válido requerido) |
| A03 — Injection (SQL / Input) | HU2, HU11 | Test con fechas con caracteres especiales (`'; DROP TABLE`), IDs malformados |
| A07 — Auth Failures | HU3, HU5 | Test de Hold y pago sin token válido / con token expirado |

---

## 11. Resumen de Tags para Ejecución por Suite

| Suite | Tags | HUs Incluidas | Condición |
|---|---|---|---|
| Smoke | `@smoke` | HU0, HU1, HU2, HU3, HU5, HU6, HU7, HU8, HU11 | Siempre — cada deploy |
| Regresión Crítica | `@alto` | HU0, HU2, HU3, HU5, HU6, HU7, HU8 | Siempre — pre-release |
| Concurrencia | `@concurrencia` | HU3 | Siempre — obligatorio |
| Seguridad | `@seguridad` | HU3 | Siempre — obligatorio |
| Worker | `@worker` | HU8 | Siempre |
| Completa | Todos | HU0–HU3, HU5–HU8, HU11 | Release final |

---

*Generado por QA Agent (ASDD) — 2026-03-24*  
*Skills aplicados: `/gherkin-case-generator` + `/risk-identifier`*  
*Actualizar la columna "Estado" de la Matriz de Trazabilidad con los resultados reales de ejecución tras cada sprint.*
