# Plan de Pruebas — Travel Hotel: Motor de Reservas

**Proyecto:** Travel Hotel — Motor de Reservas  
**Versión:** 1.0.0-MVP  
**Fecha:** 2026-03-24  
**Autor:** Joel Tates (QA)  
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
| Funcional / API | Karate | Siempre — obligatorio |
| Concurrencia | k6 + threading | HU3 — obligatorio |
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


## 5. Entorno de Pruebas

### 5.1 Ambientes

| Ambiente                    | Propósito                                                  | Configuración                                                           |
| --------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------- |
| **Local (DEV)**             | Desarrollo y validación rápida                             | Docker Compose (API + PostgreSQL), datos mock                           |
| **QA / Testing**            | Ejecución formal de pruebas funcionales, integración y E2E | Entorno aislado con PostgreSQL real, datos controlados vía Seeder (HU1) |
| **Staging**                 | Validación pre-release                                     | Configuración similar a producción, incluye worker activo (HU8)         |
| **Producción (solo smoke)** | Verificación post-deploy                                   | Datos reales, pruebas limitadas no destructivas                         |

---

### 5.2 Configuración Técnica

* **Backend:** FastAPI
* **Base de Datos:** PostgreSQL 16+
* **Control de Concurrencia:** `SELECT FOR UPDATE`
* **Worker Async:** proceso separado para expiración de holds (HU8)
* **Datos de prueba:** cargados mediante Seeder (HU1)
* **Aislamiento DB:** `READ COMMITTED` (mínimo requerido)

---

### 5.3 Reglas del Entorno

* ❌ Prohibido ejecutar pruebas destructivas en producción
* ✅ Cada ejecución de pruebas debe iniciar con datos consistentes (reset DB o rollback)
* ✅ Logs habilitados para auditoría de concurrencia y pagos
* ✅ Tiempo del sistema controlado/mocable para pruebas de expiración (HU8)

---

## 6. Herramientas

| Herramienta                 | Tipo               | Propósito                                            |
| --------------------------- | ------------------ | ---------------------------------------------------- |
| **Karate DSL**              | Testing API        | Automatización de pruebas funcionales y E2E          |
| **k6**                      | Performance / Load | Pruebas de carga, stress y concurrencia              |
| **OWASP ZAP**               | Seguridad          | Detección de vulnerabilidades OWASP Top 10           |
| **JEST**                  | Unit Testing       | Validación de lógica de negocio en servicios         |
| **Docker / Docker Compose** | Infraestructura    | Levantar entornos reproducibles                      |
| **Postman (opcional)**      | Exploratorio       | Pruebas manuales rápidas                             |
| **CI/CD (GitHub Actions)**  | Automatización     | Ejecución de suites (smoke, regresión, concurrencia) |
| **DBeaver / psql**          | DB Client          | Validación directa de datos y locks                  |

---

## 7. Roles y Responsabilidades

### 7.1 QA (Quality Assurance)

* Diseñar estrategia de pruebas y plan de QA
* Definir y mantener casos de prueba (Gherkin + automatización)
* Ejecutar pruebas:

  * Funcionales (Karate)
  * Concurrencia (k6)
  * Seguridad (ZAP)
* Validar invariantes críticas:

  * No doble reserva
  * Idempotencia de pagos
  * Expiración correcta de holds
* Reportar defectos con evidencia clara
* Mantener matriz de trazabilidad actualizada
* Validar criterios DoD antes del release

---

### 7.2 DEV (Desarrollo)

* Implementar lógica de negocio y endpoints
* Garantizar cumplimiento de contratos API
* Escribir pruebas unitarias (Pytest)
* Soportar debugging de issues detectados por QA
* Implementar fixes y validar regresión
* Configurar logs y métricas necesarias
* Asegurar control transaccional (ACID)

---

### 7.3 Responsabilidad Compartida

* Definición de criterios de aceptación
* Revisión de historias (DoR)
* Análisis de defectos críticos
* Validación de performance y estabilidad

---

## 8. Cronograma y Estimación QA (Ajustado a Micro-Sprints)

### 8.1 Enfoque de Ejecución

Dado que el tiempo disponible es **4 días (2 micro-sprints de 2 días)**, se adopta una estrategia de:

* **Risk-Based Testing (RBT)** — foco exclusivo en HUs críticas (**A — Alto**)
* **Testing mínimo viable (MVT)** — validar invariantes del negocio
* **Automatización selectiva** — solo flujos críticos
* **Cobertura reducida pero suficiente para liberar MVP**

---

### 8.2 Alcance Realista QA

| Tipo          | Incluido                | Excluido   |
| ------------- | ----------------------- | ---------- |
| **Alto (A)**  | ✅ 80% obligatorio      | —          |
| **Medio (S)** | ⚠️ Parcial (happy path) | Edge cases |
| **Bajo (D)**  | ❌ Excluido              | Todo       |

---

### 8.3 Priorización de Historias

| Prioridad  | HU       | Motivo                       |
| ---------- | -------- | ---------------------------- |
| 🔴 Crítico | HU3      | Concurrencia (doble reserva) |
| 🔴 Crítico | HU5      | Pago idempotente             |
| 🔴 Crítico | HU2      | Disponibilidad consistente   |
| 🔴 Crítico | HU6      | Confirmación correcta        |
| 🔴 Crítico | HU7      | Liberación de inventario     |
| 🔴 Crítico | HU8      | Expiración automática        |
| 🟡 Medio   | HU11     | Validación básica            |
| ⚪ Bajo     | HU0, HU1 | Solo smoke                   |

---

### 8.4 Estimación QA Ajustada

| HU      | Tipo         | Esfuerzo Original | Esfuerzo Ajustado | Estrategia                    |
| ------- | ------------ | ----------------- | ----------------- | ----------------------------- |
| HU3     | Concurrencia | 24h               | 8h                | 1 escenario crítico (2 hilos) |
| HU5     | Pago         | 16h               | 6h                | Idempotencia básica           |
| HU2     | API          | 12h               | 4h                | Happy path + 1 negativo       |
| HU6     | Estado       | 10h               | 4h                | Flujo principal               |
| HU7     | Estado       | 8h                | 3h                | Caso principal                |
| HU8     | Worker       | 12h               | 5h                | Expiración básica             |
| HU11    | Validación   | 8h                | 2h                | Validación mínima             |
| HU0/HU1 | Infra        | 20h               | 2h                | Smoke                         |

**Total ajustado QA:** ~34 horas → comprimido a ejecución intensiva en 4 días

---

### 8.5 Plan por Micro-Sprint

#### 🚀 Micro-Sprint 1 (Día 1–2)

**Objetivo:** Validar núcleo transaccional

| Día     | Actividad                          |
| ------- | ---------------------------------- |
| Día 1   | Setup entorno + smoke (HU0, HU1)   |
| Día 1   | Pruebas HU2 (disponibilidad)       |
| Día 1–2 | Pruebas HU3 (concurrencia crítica) |
| Día 2   | Pruebas HU6 (confirmación)         |

**Salida esperada:**

* Validación de **no doble reserva**
* Flujo básico de reserva funcional

---

#### 🚀 Micro-Sprint 2 (Día 3–4)

**Objetivo:** Validar resiliencia del sistema

| Día     | Actividad                   |
| ------- | --------------------------- |
| Día 3   | HU5 (idempotencia pagos)    |
| Día 3   | HU7 (liberación por fallo)  |
| Día 3–4 | HU8 (worker expiración)     |
| Día 4   | HU11 (validaciones mínimas) |
| Día 4   | Regresión crítica + smoke   |



---

## 9. Entregables de Prueba

### 9.1 Artefactos Generados

| Entregable                    | Descripción                         |
| ----------------------------- | ----------------------------------- |
| **Scripts Automatizados**     | Serenity + Cucumber, Karate, k6, Pytest                  |
| **Reportes de Ejecución**     | Resultados por suite (JSON / HTML)  |
| **Repositorio de pruebas**    | Cada uno independiente     |
| **Ejecución manual**    | Pantallazo o video corto         |
| **Reporte de bugs**          | Tambien se reportan incidencias |

---

## 10. Matriz de Riesgo

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

## Anexos

## Datos de Prueba Consolidados

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

## Matriz de Trazabilidad: AC → Escenario → Cobertura

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

## Consideraciones de Performance

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

## Consideraciones de Seguridad (OWASP Top 10)

| Riesgo OWASP | HU Relacionada | Escenario de Prueba |
|---|---|---|
| A01 — Broken Access Control | HU3 | Hold solo creado por usuario autenticado (token válido requerido) |
| A03 — Injection (SQL / Input) | HU2, HU11 | Test con fechas con caracteres especiales (`'; DROP TABLE`), IDs malformados |
| A07 — Auth Failures | HU3, HU5 | Test de Hold y pago sin token válido / con token expirado |

---

## Resumen de Tags para Ejecución por Suite

| Suite | Tags | HUs Incluidas | Condición |
|---|---|---|---|
| Smoke | `@smoke` | HU0, HU1, HU2, HU3, HU5, HU6, HU7, HU8, HU11 | Siempre — cada deploy |
| Regresión Crítica | `@alto` | HU0, HU2, HU3, HU5, HU6, HU7, HU8 | Siempre — pre-release |
| Concurrencia | `@concurrencia` | HU3 | Siempre — obligatorio |
| Seguridad | `@seguridad` | HU3 | Siempre — obligatorio |
| Worker | `@worker` | HU8 | Siempre |
| Completa | Todos | HU0–HU3, HU5–HU8, HU11 | Release final |

---


