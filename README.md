# Technical Challenge - Plataforma de datos


> **Objetivo:** Diseñar plataforma de datos end-to-end para una plataforma digital con usuarios activos a escala: desde la captura de eventos y cambios de estado, hasta los pipelines que alimentan analítica, modelos y sistemas en producción, con foco en calidad, escalabilidad y confiabilidad.
---

## 1. Introducción

Esta propuesta describe una arquitectura end-to-end para capturar y procesar:

- **Eventos de comportamiento** (sesiones, interacciones, progreso, evaluaciones)
- **Cambios de estado** (suscripción, completado, avance consolidado, etc.)
- Flujos **batch** y **streaming** para habilitar:
  - Analítica (BI / Product Analytics)
  - Modelos de recomendación y personalización
  - Experimentación (A/B testing) y métricas de aprendizaje
  - Casos de uso operativos en near real-time

**Principios de diseño**
- **Confiabilidad primero**: datos correctos y trazables > velocidad
- **Streaming donde aporta valor**: real-time solo para casos que lo requieren
- **Reproducibilidad**: datasets de analítica y ML deben reconstruirse
- **Cost-aware by design**: particionado, retención y backfills controlados
- **Data as a Product**: ownership, contratos y SLAs por dataset

---

## 2. Supuestos

Los siguientes supuestos guían las decisiones de arquitectura:

- Producto digital con **usuarios activos a escala** y **alto volumen de eventos**
- Se requieren distintos niveles de latencia:
  - **Personalización en tiempo real**: segundos
  - **Analítica near real-time**: minutos
  - **Modelos batch**: diario/semanal + backfills
- Los consumidores de datos incluyen:
  - Producto / BI / Analytics Engineering
  - ML/AI (recomendaciones, personalización, LLMs)
  - Operaciones (dashboards y alertas)
- Se necesita soporte para:
  - **evolución de esquemas**
  - **idempotencia y deduplicación**
  - **reintentos y recuperación**
  - **reprocesos/backfills**
  - **seguridad y PII** (RBAC / masking)

---

## 3. Arquitectura a Alto Nivel

### 3.1 Principales componentes

A continuación se muestra una visión general de la plataforma (GCP):

![Diagram](/docs/diagrams/dataplatform_hla.svg)

**Productores**
- Web / Mobile (SDK de eventos)
- Backend services (eventos server-side)
- Sistemas operativos (pagos, suscripciones, evaluaciones)

**Ingesta**
- **Pub/Sub** como backbone de eventos (event-driven)
- **CDC** para cambios de estado desde sistemas transaccionales (cuando aplique)
- **DataFlow Batch** Cuando en costo beneficio de mas valor que el CDC y la latencia no sea un problema

**Procesamiento**
- **Streaming (near real-time):** Dataflow (Apache Beam)
- **Batch (ELT/ETL):** dbt + BigQuery

**Orquestación**
- **Composer (Airflow)**: para orquestación de process batch

**Almacenamiento**
- **Raw inmutable (Bronze):** GCS (Parquet) para auditoría y reprocesos
- **Curado (Silver/Gold):** BigQuery para modelos analíticos y marts
- (Opcional) snapshots/exports para datasets de ML en GCS

**Serving / Real-time**
- Store de baja latencia para features y agregados:
  - **Bigtable** o **Memorystore (Redis)** según el patrón de lectura/escritura
- API/servicio de personalización consume features online

**Consumo**
- BI (Looker/Metabase)
- Producto (métricas near real-time, segmentación)
- ML/AI (training datasets reproducibles + features)

---

### 3.2 Decisiones clave y trade-offs (alto nivel)

A continuación, las decisiones más relevantes con sus trade-offs y mitigaciones:

1) **Backbone event-driven con Pub/Sub**
- **Por qué:** desacopla productores/consumidores, escalable y managed
- **Trade-off:** delivery at-least-once → posibles duplicados
- **Mitigación:** `event_id` + dedupe downstream + escrituras idempotentes

2) **Separación explícita entre streaming y batch**
- **Por qué:** optimiza costo/operación y alinea latencia con el caso de uso
- **Trade-off:** arquitectura híbrida = más componentes
- **Mitigación:** estándares de diseño + componentes reutilizables + observabilidad unificada

3) **Raw inmutable en object storage (GCS)**
- **Por qué:** auditoría, trazabilidad y backfills baratos
- **Trade-off:** gestión de lifecycle + doble storage (raw + curado)
- **Mitigación:** políticas de retención, compresión y tiering

4) **BigQuery como core para analítica (Silver/Gold)**
- **Por qué:** velocidad de entrega, performance analítica, menor carga operativa
- **Trade-off:** costo si no se optimizan consultas/modelos
- **Mitigación:** particionado, clustering, incremental models (dbt), budgets/alerts

5) **Exactly-once “efectivo” en lugar de exactly-once absoluto**
- **Por qué:** en sistemas distribuidos, el exactly-once end-to-end es costoso y complejo
- **Trade-off:** requiere diseño explícito de idempotencia
- **Mitigación:** claves determinísticas, dedupe, checkpoints y replays controlados

6) **Feature Store Lite en fase inicial**
- **Por qué:** acelera time-to-value para personalización
- **Trade-off:** menos automatización que un feature store completo
- **Mitigación:** definiciones de features en código + evolución a Vertex AI Feature Store

7) **Data Quality y SLAs desde el inicio (MVP)**
- **Por qué:** evita pérdida de confianza y re-trabajo
- **Trade-off:** más esfuerzo inicial vs “solo mover datos”
- **Mitigación:** empezar con checks críticos (freshness/volumen/duplicados) e iterar

---

## 4 Índice (detalle de diseño)

La documentación detallada se encuentra en la carpeta [`/docs`](./docs):

- 📄 **Arquitectura end-to-end (Capa de Datos + Flujo batch + Flujo streaming)**
  - [`docs/arquitecture.md`](./docs/architecture.md)

- ✅ **Confiabilidad: Data Quality, Gobierno, Seguridad y Monitoreo**
  - [`docs/data_quality_governance.md`](./docs/reliability_governance.md)

- 🧰 **Marco de desarrollo (estándares + CI/CD + Arquetipos)**
  - [`docs/data_development_framework.md`](./docs/framework.md)

- 🗺️ **Roadmap por fases**
  - [`docs/roadmap.md`](./docs/roadmap.md)

---

### Alternativa multi-cloud (AWS)

Si se requiere una implementación equivalente en AWS, el mapeo por capacidades sería:

- Pub/Sub → Kinesis / MSK (Kafka)
- Dataflow → Flink / Glue / EMR Spark
- GCS → S3
- BigQuery → Redshift / Snowflake
- Bigtable → DynamoDB
- Memorystore → ElastiCache (Redis)
- Airflow → MWAA

> Nota: la arquitectura se mantiene; cambian los servicios managed.

---
