# QA Hotel MVP — Suite Maestra de Pruebas

**Proyecto:** Travel Hotel — Motor de Reservas  
**Version:** MVP v1.0.0  
**QA responsable:** Joel Tates  
**Fecha:** 2026-04-06

Este repositorio es el punto central de la estrategia de QA para el motor de reservas de Travel Hotel. Consolida los resultados, artefactos y reportes de las tres suites de prueba independientes ejecutadas sobre el sistema.

---

## Repositorios de Prueba

Cada suite vive en un repositorio separado para mantener el harness desacoplado del backend objetivo.

| Suite | Tecnologia | HUs cubiertas | Repositorio |
|---|---|---|---|
| Pruebas funcionales de API | Karate + JUnit 5 + Gradle | HU2, HU3, HU5, HU6, HU11 | [HOTEL_QA_TEST_KARATE](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_KARATE) |
| Pruebas de rendimiento | k6 (Grafana) | HU2, HU3, HU5, HU6, HU7, HU11 | [HOTEL_QA_TEST_K6](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_K6) |
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

### Pruebas de Rendimiento — k6

Los resultados se exportan en consola y en JSON. No tiene GitHub Pages dedicado; los resultados de cada corrida quedan disponibles como artefactos de GitHub Actions y pueden exportarse a Grafana Cloud k6.

**Pipeline CI:** [github.com/ItsgabrielJT/HOTEL_QA_TEST_K6/actions](https://github.com/ItsgabrielJT/HOTEL_QA_TEST_K6/actions)

---

## Reporte Final de QA

El reporte consolidado con todos los resultados, metricas, riesgos y conclusion de calidad esta en:

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
| OWASP ZAP | ⚠️ WARN | 2 alertas Low (headers), 0 alertas High/Medium |
