# QA Test Report — Travel Hotel: Motor de Reservas

---

## 1. Informacion General

| Campo | Valor |
|---|---|
| Proyecto | Travel Hotel — Motor de Reservas |
| Version probada | MVP v1.0.0 |
| Ambiente | QA (Docker Compose local + CI) |
| Fecha de ejecucion | 2026-04-06 |
| QA responsable | Joel Tates |
| Commit objetivo | `82cc2d1acbd29786c7973ae4eb33d99ae16fc3ec` |
| Branch objetivo | `dev` |
| Repositorio backend | [EGgames/HOTEL-MVP](https://github.com/EGgames/HOTEL-MVP) |

### Repositorios de prueba

| Suite | Repositorio |
|---|---|
| Pruebas funcionales (Karate) | [HOTEL_QA_TEST_KARATE](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE) |
| Pruebas de rendimiento (k6) | [HOTEL_QA_TEST_K6](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_K6) |
| Pruebas de base de datos (pytest) | [HOTEL_QA_TEST_BD](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_BD) |

---

## 2. Alcance de las Pruebas

### Dentro del alcance

| Tipo | Herramienta | HUs cubiertas |
|---|---|---|
| Pruebas funcionales de API | Karate + JUnit 5 + Gradle | HU2, HU3, HU5, HU6, HU11 |
| Pruebas de rendimiento (smoke, load, stress) | k6 (Grafana) | HU2, HU3, HU5, HU6, HU7, HU11 |
| Pruebas de base de datos e infraestructura | pytest + psycopg + Docker Compose | HU0, HU1 |
| Escaneo de seguridad DAST automatizado | OWASP ZAP | Todos los endpoints expuestos |

### Fuera del alcance

- Pruebas de interfaz grafica (UI)
- Pruebas en dispositivos moviles
- Pruebas de accesibilidad
- HU8 (worker de expiracion de holds): la API no expone hook de disparo controlable desde suites externas
- TC-HU3-02 (concurrencia exacta): no estable con el contrato observable actual
- TC-HU5-04 (idempotency key cross-hold): el backend no rechaza esta condicion hoy

---

## 3. Estrategia de Pruebas

### Pruebas funcionales — Karate

Automatizacion externa orientada a contrato, ejecutada con Gradle sobre un checkout del backend levantado en Docker Compose. Los runners de dominio se ejecutan en paralelo con 5 hilos (availability, holds, payments, reservations, validation).

### Pruebas de base de datos — pytest

Suite externa de integracion que levanta PostgreSQL en contenedor efimero, aplica limpieza de schema entre casos y valida concurrencia, rollback transaccional y bootstrap de API directamente contra la base. Cobertura medida con `pytest-cov`.

### Pruebas de rendimiento — k6

Tres niveles de prueba ejecutados de forma incremental:

| Nivel | VUs | Duracion | Objetivo |
|---|---|---|---|
| Smoke | 1 VU | ~1m45s | Verificar que el script y la API responden correctamente |
| Load | 16 VUs (8+5+3) | 3 min | Validar SLOs bajo carga sostenida normal |
| Stress | 80 VUs (40+50) | 12 min | Buscar punto de quiebre y validar degradacion controlada |

### Escaneo de seguridad — OWASP ZAP

Escaneo DAST automatizado sobre el spec OpenAPI del backend. Ejecutado en paralelo con Karate en el mismo pipeline de CI.

---

## 4. Resultados — Pruebas Funcionales (Karate)

**Estado general: PASS**

| Metrica | Valor |
|---|---|
| Scenarios ejecutados | 6 |
| Scenarios pasados | 6 |
| Scenarios fallidos | 0 |
| Features ejecutados | 1 |
| Tasa de exito | 100% |
| Efficiency index | 0.42 |

### Detalle por dominio activo

| Suite / HU | TC activos | Pasados | Fallidos | Ignorados |
|---|---|---|---|---|
| HU2 — Disponibilidad | 5 | 5 | 0 | 2 |
| HU3 — Hold | 2 | 2 | 0 | 0 |
| HU5 — Idempotencia | 1 | 1 | 0 | 1 |
| HU6 — Confirmacion | 2 | 2 | 0 | 0 |
| HU11 — Validacion fechas | 4 | 4 | 0 | 0 |
| **Total activos** | **14** | **14** | **0** | **3** |

> Los 3 casos con `@ignore` (TC-HU2-04, TC-HU2-06, TC-HU5-02) quedan excluidos del runner y documentados para reactivacion futura.

### Casos fuera de alcance (no implementados)

| Caso | Motivo |
|---|---|
| TC-HU3-02 | Concurrencia end-to-end no estable con contrato observable actual |
| TC-HU5-04 | El backend reutiliza resultado cacheado; no rechaza llave cross-hold |
| TC-HU7-02 | Requiere control de eventos tardios no expuesto por la API |
| TC-HU8-01/02/03/04 | Worker de expiracion no activable desde suite externa |

---

## 5. Resultados — Pruebas de Base de Datos (pytest)

**Estado general: PASS**

| Metrica | Valor |
|---|---|
| Tests ejecutados | 6 |
| Pasados | 6 |
| Fallidos | 0 |
| Errores | 0 |
| Omitidos | 0 |
| Cobertura del harness | 93.94% |
| Gate cobertura minima | 80% |

### Quality Gates

| Gate | Condicion | Estado | Evidencia |
|---|---|---|---|
| Tests | 6 tests, 0 failures, 0 errors | PASS | passed=6, failures=0, errors=0 |
| Coverage | >= 80% | PASS | coverage=93.94% |
| DB bootstrap and seed | HU0-03 + HU1 sin errores | PASS | test_tc_hu0_03 + test_tc_hu1_01/02/03 |
| Concurrency | TC-HU0-01 sin race condition observable | PASS | SELECT FOR UPDATE bloquea actualizacion concurrente |

### Detalle de casos

| TC-ID | HU | Descripcion | Estado |
|---|---|---|---|
| TC-HU0-01 | HU0 | `SELECT FOR UPDATE` bloquea transaccion concurrente hasta COMMIT | PASS |
| TC-HU0-02 | HU0 | Rollback completo ante excepcion dentro de transaccion | PASS |
| TC-HU0-03 | HU0 | Bootstrap de API no altera datos ya sembrados | PASS |
| TC-HU1-01 | HU1 | Seeder crea hoteles y habitaciones validas en base vacia | PASS |
| TC-HU1-02 | HU1 | Seeder es idempotente; no genera duplicados en segunda ejecucion | PASS |
| TC-HU1-03 | HU1 | Seeder falla con mensaje util cuando la BD es inaccesible | PASS |

---

## 6. Resultados — Pruebas de Rendimiento (k6)

### 6.1 Smoke Test — 1 VU, ~1m45s

**Estado: PASS**

| Metrica | Valor |
|---|---|
| Total requests | 46 |
| Iteraciones | 15 |
| Tasa de error HTTP | 0.00% |
| http_req_duration avg | 10.26 ms |
| http_req_duration p(95) | 14.88 ms |
| http_req_duration p(99) | 31.56 ms |
| Checks pasados | 30/30 (100%) |

| Threshold | SLO | Resultado |
|---|---|---|
| checks rate | >= 95% | ✅ 100.00% |
| http_req_duration p(95) | < 800 ms | ✅ 14.88 ms |
| http_req_duration p(99) | < 1500 ms | ✅ 31.56 ms |
| http_req_failed | < 1% | ✅ 0.00% |

---

### 6.2 Load Test — 16 VUs, 3 minutos

**Estado: PASS**

| Metrica | Valor |
|---|---|
| Total requests | 1 967 |
| Requests/s | 10.87 |
| Iteraciones | 1 442 |
| Tasa de error HTTP | 0.00% |
| http_req_duration avg | 5.53 ms |
| http_req_duration p(95) | 10.23 ms |
| http_req_duration p(99) | 12.35 ms |
| Checks pasados | 3 963/3 963 (100%) |

| Threshold | SLO | Resultado |
|---|---|---|
| checks rate | >= 95% | ✅ 100.00% |
| http_req_duration p(95) | < 800 ms | ✅ 10.23 ms |
| http_req_duration p(99) | < 1500 ms | ✅ 12.35 ms |
| http_req_failed | < 1% | ✅ 0.00% |

| Endpoint | p(95) | SLO |
|---|---|---|
| get_availability | 10.17 ms | < 500 ms ✅ |
| create_hold | 8.21 ms | < 600 ms ✅ |
| process_payment | 14.17 ms | < 1000 ms ✅ |

Checks funcionales pasados (100%):

- TC-HU2-01 | status 200 y array JSON
- BookingFlow | disponibilidad → hold → pago → confirmacion
- TC-HU6-01/02/03 | confirmacion y liberacion de reserva
- TC-HU11-01/02/03 | rechazo de fechas invalidas (400)

---

### 6.3 Stress Test — 80 VUs, 12 minutos

**Estado: PASS (con degradacion controlada)**

| Metrica | Valor |
|---|---|
| Total requests | 45 764 |
| Requests/s | 63.52 |
| Iteraciones | 44 188 |
| VUs maximos | 80 (availability 40 + booking 50) |
| Tasa de error HTTP | 0.25% (118/45 764) |
| http_req_duration avg | 9.73 ms |
| http_req_duration p(95) | 19.83 ms |
| http_req_duration p(99) | 26.29 ms |
| Checks pasados | 91 679/91 850 (99.81%) |
| Checks fallidos | 171/91 850 (0.18%) |

| Threshold | SLO | Resultado |
|---|---|---|
| checks rate | >= 85% | ✅ 99.81% |
| http_req_duration p(95) | < 1500 ms | ✅ 19.83 ms |
| http_req_duration p(99) | < 3000 ms | ✅ 26.29 ms |
| http_req_failed | < 5% | ✅ 0.25% |

| Endpoint | p(95) | SLO |
|---|---|---|
| get_availability | 19.88 ms | < 500 ms ✅ |
| create_hold | 18.70 ms | < 600 ms ✅ |
| process_payment | 15.91 ms | < 1000 ms ✅ |

**Checks con fallo parcial bajo carga maxima:**

| Check | Tasa exito | Causa |
|---|---|---|
| BookingFlow — POST hold retorna 201 | 85% (328/385) | Inventario agotado bajo 80 VUs; 409/400 por solapamiento es respuesta valida del sistema |
| BookingFlow — hold status es PENDING | 85% (328/385) | Derivado del anterior |
| BookingFlow — hold tiene expires_at | 85% (328/385) | Derivado del anterior |

> El agotamiento de inventario bajo carga extrema es comportamiento esperado del sistema, no un defecto. Los logs confirman "Sin habitaciones para [rango] — iteracion omitida".

---

## 7. Resultados — Escaneo de Seguridad (OWASP ZAP)

**Estado: WARN**

| Nivel de riesgo | Alertas |
|---|---|
| High | 0 |
| Medium | 0 |
| Low | 2 |
| Informational | 3 |

### Alertas de riesgo bajo

| Alerta | Riesgo | Descripcion | Recomendacion |
|---|---|---|---|
| Server Leaks Information via `X-Powered-By` | Low | La respuesta expone `X-Powered-By: Express`, facilitando fingerprinting del stack | Suprimir el header en la configuracion del servidor |
| X-Content-Type-Options Header Missing | Low | El header de seguridad `X-Content-Type-Options` no se envia | Agregar `X-Content-Type-Options: nosniff` en cabeceras HTTP |

### Alertas informativas

| Alerta | Detalle |
|---|---|
| 99% de respuestas con codigo 4xx | Comportamiento esperado por los tests de validacion de fechas invalidas |
| 100% de endpoints con Content-Type application/json | Consistent API contract |
| 40 endpoints escaneados | Cobertura total del spec OpenAPI |

> El FAIL reportado en el pipeline de ZAP corresponde a un error de proceso (exit code 99), no a vulnerabilidades de alto riesgo. No se encontraron alertas Medium ni High.

---

## 8. Bugs Encontrados

> Esta seccion esta reservada para ser completada por el QA responsable con los defectos identificados durante la sesion de pruebas.

| ID | Endpoint | Severity | Descripcion | Estado |
|---|---|---|---|---|
| — | — | — | — | — |

---

## 9. Riesgos Detectados

| Riesgo | Nivel | Detalle |
|---|---|---|
| Header `X-Powered-By` expuesto | Bajo | Facilita fingerprinting del stack tecnologico (Express). Corregible con una linea de configuracion. |
| Header `X-Content-Type-Options` ausente | Bajo | Sin impacto directo en la API actual pero es una buena practica de seguridad obligatoria. |
| Agotamiento de inventario bajo carga extrema | Informativo | Bajo 80 VUs concurrentes el inventario de 2027 se agota; el sistema responde correctamente con 409/400. Se recomienda ampliar el pool de fechas de prueba o sembrar mas habitaciones para pruebas de stress. |
| Casos de HU8 sin cobertura automatizada | Medio | El worker de expiracion de holds no es activable desde suites externas. Riesgo de regresion silenciosa en esa logica. |
| Concurrencia TC-HU3-02 sin cobertura | Medio | El caso de dos usuarios intentando el mismo hold simultaneamente no esta validado de forma confiable. |

---

## 10. Cobertura de Pruebas

| Dimension | Valor |
|---|---|
| HUs cubiertas por al menos una suite | HU0, HU1, HU2, HU3, HU5, HU6, HU7, HU11 |
| HUs sin cobertura | HU4, HU8, HU9, HU10 (fuera del alcance acordado) |
| TCs funcionales activos (Karate) | 14 de 14 pasados |
| TCs de rendimiento activos (k6) | 15 activos, 0 fallidos en smoke/load |
| TCs de base de datos (pytest) | 6 de 6 pasados |
| Cobertura de codigo del harness BD | 93.94% |
| Endpoints cubiertos por ZAP | 40 (100% del spec OpenAPI) |
| Endpoints criticos cubiertos | 100% (disponibilidad, hold, pago, reserva) |

---

## 11. Conclusion de Calidad

**Veredicto: PASS WITH RISKS**

El sistema demuestra estabilidad solida en todos los flujos criticos del motor de reservas bajo condiciones normales y de carga sostenida.

- Los 14 casos funcionales activos de Karate pasan al 100%.
- Los 6 casos de base de datos pasan al 100% con 93.94% de cobertura del harness.
- El smoke y el load test pasan con 0% de error HTTP y 100% de checks.
- El stress test a 80 VUs concurrentes durante 12 minutos pasa todos los thresholds definidos; la degradacion observada en el flujo de booking (15% de iteraciones sin inventario) es comportamiento valido del sistema bajo carga extrema.
- No se encontraron vulnerabilidades de riesgo alto o medio en el escaneo OWASP ZAP.
- Las dos alertas de seguridad de nivel bajo son configurables sin cambios de logica de negocio.

**Recomendacion:** El sistema puede avanzar a una instancia de staging con las dos correcciones de headers de seguridad aplicadas y con la documentacion de los gaps de cobertura (HU8, TC-HU3-02) como deuda tecnica de QA explicitada.

---

## 12. Evidencias

### Reportes publicados en GitHub Pages

| Suite | Reporte |
|---|---|
| Karate + ZAP | [itsgabrieljt.github.io/HOTEL_QA_TEST_KARATE](https://itsgabrieljt.github.io/HOTEL_QA_TEST_KARATE/) |
| Base de datos (Allure) | [itsgabrieljt.github.io/HOTEL_QA_TEST_BD](https://itsgabrieljt.github.io/HOTEL_QA_TEST_BD/) |

### Repositorios de prueba

| Suite | Repositorio |
|---|---|
| Pruebas funcionales — Karate | [github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE) |
| Pruebas de rendimiento — k6 | [github.com/ItsgabrielJT/HOTEL_QA_TEST_K6](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_K6) |
| Pruebas de base de datos — pytest | [github.com/ItsgabrielJT/HOTEL_QA_TEST_BD](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_BD) |

### Pipeline de CI

| Suite | Pipeline |
|---|---|
| Karate + ZAP | [github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE/actions](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE/actions) |
| Base de datos | [github.com/ItsgabrielJT/HOTEL_QA_TEST_BD/actions](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_BD/actions) |

### Artefactos locales incluidos en este repositorio

| Artefacto | Descripcion |
|---|---|
| `db-qa-artifacts-24019646613/` | Resultados completos de la suite pytest: logs, reportes Allure, quality-gates, cobertura HTML |
| `karate-artifacts-24059252560/` | Resultados de Karate y ZAP: HTML, JUnit XML, execution-summary.json, karate-pages |
| `zap-artifacts-24059252560/` | Reporte OWASP ZAP en HTML, JSON y Markdown |
