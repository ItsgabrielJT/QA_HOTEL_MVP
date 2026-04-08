# QA Hotel MVP — Suite Maestra de Pruebas

**Proyecto:** Travel Hotel — Motor de Reservas  
**Version:** MVP v1.0.0  
**QA responsable:** Joel Tates  
**Fecha:** 2026-04-06

Este repositorio es el punto central de la estrategia de QA para el motor de reservas de Travel Hotel. Consolida los resultados, artefactos y reportes de las suites de prueba independientes ejecutadas sobre el sistema, incluyendo cobertura funcional de API, rendimiento, base de datos y un flujo UI E2E complementario.

---

## Repositorios de Prueba

Cada suite vive en un repositorio separado para mantener el harness desacoplado del backend objetivo.

| Suite | Tecnologia | HUs cubiertas | Repositorio |
|---|---|---|---|
| Pruebas funcionales de API | Karate + JUnit 5 + Gradle | HU2, HU3, HU5, HU6, HU11 | [HOTEL_QA_TEST_KARATE](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE) |
| Pruebas de rendimiento | k6 (Grafana) | HU2, HU3, HU5, HU6, HU7, HU11 | [HOTEL_QA_TEST_K6](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_K6) |
| Pruebas UI E2E | Java 17 + Gradle + Serenity BDD + Screenplay | HU2, HU3, HU6, HU7 | [HOTEL_QA_TEST_SERENITY](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_SERENITY) |
| Pruebas de base de datos | pytest + psycopg + Allure | HU0, HU1 | [HOTEL_QA_TEST_BD](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_BD) |

---

## Donde se Genero Cada Prueba

### Pruebas Funcionales — Karate

Generadas en [HOTEL_QA_TEST_KARATE](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE).

Validan el contrato observable de la API mediante runners de dominio organizados por historia de usuario: disponibilidad, holds, pagos, reservas y validacion de fechas. Se ejecutan con Gradle en paralelo (5 hilos) sobre un checkout del backend levantado en Docker Compose.

Tambien incluyen el escaneo de seguridad automatizado con OWASP ZAP que corre en paralelo en el mismo pipeline de CI.

### Pruebas de Rendimiento — k6

Generadas en [HOTEL_QA_TEST_K6](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_K6).

Validan que los flujos criticos del motor de reservas cumplan los SLOs definidos bajo distintos niveles de carga:

| Test | VUs | Duracion | Proposito |
|---|---|---|---|
| `smoke-test.js` | 1 VU | ~1m45s | Verificacion basica de script y sistema |
| `load-test.js` | 16 VUs | 3 min | Carga normal sostenida |
| `stress-test.js` | 80 VUs | 12 min | Limite de capacidad del sistema |
| `idempotency-test.js` | 1 VU | ~2 min | Idempotencia de pagos (HU5) |

Adicionalmente incluye `double-booking-test.js`, una prueba concurrente focalizada que valida que no se confirmen reservas duplicadas para la misma habitacion y rango bajo 20 VUs simultaneos.

### Pruebas UI E2E — Serenity BDD + Screenplay

Generadas en [HOTEL_QA_TEST_SERENITY](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_SERENITY).

Automatizan el flujo visible para el usuario final sobre la aplicacion Hotel Booking MVP (`http://localhost:5173`) con Serenity BDD, Cucumber y el patron Screenplay. Cubren busqueda de habitaciones, reserva end-to-end con reintento tras pago rechazado y validacion de bloqueo concurrente en la UI.

Cobertura principal documentada en el README de la suite:

| Feature | Cobertura |
|---|---|
| `busqueda_habitaciones.feature` | Busqueda exitosa con fechas validas, validacion sin fechas y escenarios parametrizados |
| `reserva_habitacion.feature` | Happy path completo de reserva, redireccion a checkout, mensaje de pago rechazado y confirmacion con codigo |
| `bloqueo_habitacion.feature` | La habitacion elegida por un usuario deja de estar disponible para un segundo usuario |

### Pruebas de Base de Datos — pytest

Generadas en [HOTEL_QA_TEST_BD](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_BD).

Validan el ecosistema de datos de PostgreSQL: concurrencia transaccional (`SELECT FOR UPDATE`), rollback ante excepciones, bootstrap de la API sin alteracion de datos sembrados, y correcta inicializacion del inventario por el seeder.

---

## Reportes por Suite

### Pruebas Funcionales — Karate + ZAP

Los reportes se publican automaticamente en GitHub Pages tras cada ejecucion de CI.

**GitHub Pages:** [itsgabrieljt.github.io/HOTEL_QA_TEST_KARATE](https://itsgabrieljt.github.io/HOTEL_QA_TEST_KARATE/)

Contiene:
- Reporte HTML de Karate con detalle por escenario
- Reporte HTML de Gradle
- Resultados JUnit XML
- Reporte OWASP ZAP en HTML y Markdown

**Pipeline CI:** [github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE/actions](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE/actions)

---

### Pruebas de Base de Datos — pytest + Allure

Los reportes Allure se publican automaticamente en GitHub Pages tras cada ejecucion de CI.

**GitHub Pages:** [itsgabrieljt.github.io/HOTEL_QA_TEST_BD](https://itsgabrieljt.github.io/HOTEL_QA_TEST_BD/)

Contiene:
- Reporte Allure interactivo con trazabilidad por TC-ID
- Reporte HTML de pytest
- Cobertura de codigo del harness (93.94%)
- Quality gates con estado PASS/FAIL
- Trazabilidad BDD entre TC-ID y validaciones automatizadas

**Pipeline CI:** [github.com/ItsgabrielJT/HOTEL_QA_TEST_BD/actions](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_BD/actions)

---

### Pruebas UI E2E — Serenity BDD + Screenplay

El reporte HTML se genera localmente con Serenity tras ejecutar `./gradlew test aggregate`.

**Reporte local:** `target/site/serenity/index.html`

Contiene:
- Capturas de pantalla por paso
- Resumen ejecutivo de escenarios aprobados y fallidos
- Documentacion viva del flujo en lenguaje natural
- Linea de tiempo de ejecucion

---

### Pruebas de Rendimiento — k6

Los resultados se exportan en consola y en JSON. No tiene GitHub Pages dedicado; los resultados de cada corrida quedan disponibles como artefactos de GitHub Actions y pueden exportarse a Grafana Cloud k6.

**Pipeline CI:** [github.com/ItsgabrielJT/HOTEL_QA_TEST_K6/actions](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_K6/actions)

---

## Reporte Final de QA

El reporte consolidado con los resultados, metricas, riesgos y conclusion de calidad actualmente documentados esta en:

**[REPORT_FINAL.md](./REPORT_FINAL.md)**

Cubre:
1. Informacion general del ciclo de pruebas
2. Alcance y estrategia
3. Resultados funcionales (Karate)
4. Resultados de base de datos (pytest)
5. Resultados de rendimiento — Smoke, Load y Stress (k6)
6. Resultados de seguridad (OWASP ZAP)
7. Riesgos detectados
8. Cobertura total
9. Conclusion de calidad y veredicto

La suite UI E2E con Serenity queda enlazada en este README como cobertura complementaria del flujo observable de reserva. Su reporte operativo vive en el repositorio dedicado de Serenity.

**Veredicto:** `PASS WITH RISKS`

---

## Resumen de Resultados

| Suite | Estado | Detalle |
|---|---|---|
| Karate (funcional) | ✅ PASS | 14/14 TCs activos pasados, 100% |
| pytest (base de datos) | ✅ PASS | 6/6 tests, cobertura 93.94% |
| k6 Smoke | ✅ PASS | 30/30 checks, 0% error HTTP |
| k6 Load (16 VUs) | ✅ PASS | 3963/3963 checks, 0% error HTTP |
| k6 Stress (80 VUs) | ✅ PASS | 99.81% checks, 0.25% error HTTP |
| Serenity UI E2E | ℹ️ INFO | Suite complementaria documentada; resultados se consultan en su reporte Serenity dedicado |
| OWASP ZAP | ⚠️ WARN | 2 alertas Low (headers), 0 alertas High/Medium |
