# Arquitectura de Datos (Medallion + Lambda)

> **Objetivo:** habilitar analítica de producto, decisiones operativas y personalización en tiempo real con datos **confiables**, **reproducibles** y **operables** a escala.

> ℹ️ **Alcance:** este documento define la arquitectura y decisiones principales (no es un diseño exhaustivo por componente). El foco es establecer un diseño claro, escalable y pragmático.

---

## Arquitectura

![Diagram](/docs/diagrams/dataplatform_hla.svg)

---

## 1. Justificación de patrones de diseño

### 1.1 Medallion (almacenamiento): Bronze → Silver → Gold

**Por qué:** separa responsabilidades y evita que el sistema se convierta en un “warehouse monolítico” sin control.

- **Bronze:** historial crudo, auditabilidad, replay
- **Silver:** normalización por dominio, base reutilizable
- **Gold:** productos listos para consumo (BI/ML/product analytics)

**Trade-offs**
- (+) mejor trazabilidad, backfills y control de calidad
- (-) más pasos (pero con ownership claro se vuelve un acelerador, no un freno)

---

### 1.2 Lambda (procesamiento): Batch + Streaming

**Por qué:** el producto necesita simultáneamente:
- **Batch:** eficiencia, consistencia y reproducibilidad (reporting + ML offline)
- **Streaming:** baja latencia (personalización y señales online)

**Trade-offs**
- (+) separa “hot path” del “cold path” (menos riesgo operacional)
- (-) requiere claridad en el source of truth (normalmente Silver/Gold)

---

## 2. Capas de datos (Medallion)

> Principio: cada capa tiene **propósito**, **consumidores** y **reglas mínimas**.

---

### 2.1 Bronze / Raw (GCS)

**Objetivo**
- preservar datos tal como llegan (inmutables)
- habilitar replay/backfills
- desacoplar ingestión del warehouse

**Tecnología**
- **GCS** (raw principal)

**Características**
- inmutable (append-only)
- particionado por fecha de evento/ingesta
- formato recomendado: **Parquet** (batch) / **Avro** (CDC) / JSON (si aplica)

**Organización recomendada**
- por **fuente** y **tipo de ingesta**
  - `raw/events/...`
  - `raw/cdc/...`
  - `raw/external/...`

**Modelado**
- ninguno (data-as-received + metadata mínima)

**Trade-off**
- raw en GCS no es para consumo directo, es para **resiliencia + costo**.

---

### 2.2 Silver / Curated (BigQuery)

**Objetivo**
- estandarizar, deduplicar y normalizar
- crear entidades consistentes por dominio (reutilizables)

**Tecnología**
- **BigQuery** como capa curada principal

**Características**
- schema estable
- reglas de negocio mínimas
- idempotencia y dedupe por llaves (`event_id`, `session_id`, etc.)
- incremental por partición

**Organización**
- Dataset por **dominio de información**
  - `silver_identity`
  - `silver_learning`
  - `silver_product`
  - `silver_billing`

**Modelado**
- entidades normalizadas / snapshots
- SCD Type 2 cuando se requiere historial de estado

---

### 2.3 Gold / Data Products (BigQuery)

**Objetivo**
- exponer datasets listos para consumo (BI, producto, ML offline)
- estandarizar métricas y definiciones

**Tecnología**
- **BigQuery** (marts y capa semántica)

**Características**
- datasets documentados, estables y optimizados
- SLAs solo para productos críticos (no para todo)

**Organización**
- por **producto de datos / iniciativa**
  - `gold.product_usage_daily`
  - `gold.learning_funnel`
  - `gold.recommendations_metrics`
  - `gold.ab_testing_results`

**Modelado**
- star schema / tablas wide / agregados (según consumidor)

**Trade-off**
- si Gold crece sin ownership se duplica la lógica → por eso se organiza por “producto de datos”.

---

### 2.4 Serving / Online Layer (fuera de Medallion)

**Objetivo**
- habilitar lecturas low-latency para personalización en producción

**Tecnología**
- **Bigtable** como Online Store

**Qué guarda**
- estado actual por usuario/sesión
- features online (últimas señales)
- contadores y afinidades

📌 **Nota:** Bigtable no es “Gold”. Es “serving operational”.

---

## 3. Flujos Batch

> Batch prioriza **consistencia, reproducibilidad y costo**.

---

### 3.1 Batch T-1 (latencia diaria / horas)

**Cuándo aplica**
- reporting
- métricas de producto
- cohortes y funnels
- datasets de entrenamiento offline

**Diseño**
- **Composer (Airflow)** orquesta:
  - carga incremental a staging (si aplica)
  - `dbt run` (Silver → Gold)
  - `dbt test` como gate

**Trade-offs**
- (+) barato, estable, reproducible
- (-) no resuelve personalización inmediata

---

### 3.2 Batch para completar CDC (latencia baja aceptable)

**Cuándo aplica**
- cambios de entidades (suscripciones, progreso, perfiles)
- analítica operativa que tolera minutos

**Diseño**
- **Datastream → GCS (CDC logs)**
- Composer ejecuta micro-batches (cada 5/15 min):
  - carga incremental a BigQuery staging
  - dbt incremental para “current + history”

**Trade-offs**
- (+) operación más simple que streaming full
- (+) histórico raw auditable
- (-) no es sub-segundo

---

### 3.3 Batch para backfills / reprocesos

**Cuándo aplica**
- bugs de lógica
- cambios de negocio
- reconstrucción histórica

**Diseño**
- replay desde Bronze (GCS)
- publicación controlada (shadow + swap)

**Trade-offs**
- (+) resiliencia fuerte
- (-) requiere guardrails para no disparar costos

---

## 4. Flujos Streaming & Real-Time

> Streaming se usa solo cuando agrega valor claro (latencia y estado online).

---

### 4.1 Micro-batch con Dataflow (ventanas)

**Cuándo aplica**
- agregaciones por ventana (1m/5m)
- métricas near-real-time
- señales que no requieren evento inmediato

**Diseño**
- Pub/Sub → Dataflow (windowing) → BigQuery/Bigtable

**Trade-offs**
- (+) buen balance costo/latencia
- (-) agrega delay por ventana

---

### 4.2 Event-by-event con Dataflow (hot path)

**Cuándo aplica**
- features online sensibles al tiempo
- actualización inmediata de estado (ej. “última actividad”, contadores)

**Diseño**
- Pub/Sub → Dataflow → Bigtable (upsert por clave)
- idempotencia por `event_id` y dedupe strategy

**Trade-offs**
- (+) escalable y low-latency
- (-) más costoso y exige disciplina de deduplicación

---

### 4.3 Event-by-event con Cloud Run (consumers simples)

**Cuándo aplica**
- transformaciones ligeras
- integraciones rápidas
- volumen moderado

**Diseño**
- Pub/Sub → Cloud Run → Bigtable / APIs / logs

**Trade-offs**
- (+) simple, rápido, mantenible
- (-) no ideal para procesamiento pesado o windowing

---

## 5 Integración con ML / Analítica avanzada

### 5.1 Estrategia mínima (pragmática)

Separar claramente:
- **Offline ML:** entrenamiento con Silver/Gold (BigQuery)
- **Online ML:** inferencia en request-time con features online (Bigtable)

---

### 5.2 Datasets reproducibles de entrenamiento (offline)

**Fuente**
- Silver/Gold en BigQuery

**Recomendación mínima**
- versionar transformaciones (dbt git SHA)
- snapshot por ventana temporal (train/val/test)
- registrar metadata del dataset (fecha, query, feature set)

---

### 5.3 Feature Store (offline + online)

**Objetivo**
- consistencia entre entrenamiento e inferencia

**Propuesta**
- **Offline features:** BigQuery
- **Online features/state:** Bigtable

**Trade-offs**
- (+) escalable y claro por propósito
- (-) requiere ownership y definición mínima de features

---

### 5.4 Inferencia en tiempo real (serving)

**Cuándo ocurre**
- cuando el usuario solicita una decisión (home, “siguiente lección”, recomendaciones)

**Diseño recomendado**
- App → **Cloud Run (Personalization API)**
  - lookup de features en Bigtable
  - llamada al modelo en **Vertex AI Endpoint**
  - reglas de negocio + fallback
  - respuesta al usuario

📌 **Nota:** evitar que el pipeline streaming sea el path crítico de inferencia (reduce fragilidad y costo).

---

## 6. Decisiones clave (resumen ejecutivo)

- Medallion para separar raw/curated/products y habilitar replay + gobernanza
- Lambda para balancear costo/consistencia (batch) y latencia (streaming)
- GCS como raw principal (histórico + bajo costo)
- BigQuery como Silver/Gold (transformación + analítica)
- Bigtable como Online Store (serving low-latency)
- Cloud Run + Vertex AI para inferencia real-time (request-driven)

---

## 7. Qué NO haría en etapa inicial

- Feature store enterprise complejo sin casos productivos claros
- Inferencia en Dataflow como path crítico (alto costo y riesgo)
- Gold sin ownership (métricas duplicadas y marts inconsistentes)
- Lineage manual perfecto desde día 1 (priorizar automatización)

---

## 8. Stack propuesto (servicios / herramientas)

> Principio: usar **lo mínimo necesario**, priorizando herramientas administradas, reproducibles y con buena operabilidad.

| Componente | Servicio / Herramienta | Rol en la arquitectura | Justificación breve |
|---|---|---|---|
| Data Lake (Raw/Bronze) | **GCS** | Almacenamiento histórico crudo e inmutable | Bajo costo, escalable, ideal para replay/backfills y auditoría. |
| Data Warehouse (Silver/Gold) | **BigQuery** | Curación, modelado, serving analítico | Motor serverless, rápido para ELT, integra bien con dbt y BI. |
| Transformaciones | **dbt** | ELT, tests, documentación, incremental models | Acelera desarrollo con estándares, CI/CD, modularidad y data quality gates. |
| Orquestación Batch | **Composer (Airflow)** | Scheduling, dependencias, retries | Patrón estándar, control de ejecución y observabilidad de pipelines batch. |
| CDC (Change Data Capture) | **Datastream** | Captura cambios desde BD transaccionales | Minimiza impacto en origen, habilita replicación y sincronización continua. |
| Event Bus | **Pub/Sub** | Transporte de eventos en tiempo real | Desacopla productores/consumidores, escala alto volumen con baja fricción. |
| Stream Processing | **Dataflow** | Enriquecimiento streaming, windowing, stateful processing | Escalable, robusto para streaming real, soporta exactly-once según diseño. |
| Serving low-latency | **Bigtable** | Estado online, contadores, features online | Baja latencia, alto throughput, ideal para “key-value access patterns”. |
| APIs / Consumers ligeros | **Cloud Run** | Consumo simple de eventos y serving API | Serverless, simple de operar, despliegue rápido, buen fit para microservicios. |
| Model Serving | **Vertex AI Endpoint** | Inferencia en tiempo real administrada | Facilita despliegue/operación de modelos, autoscaling, versionamiento. |
| Observabilidad (mínimo) | **Cloud Logging/Monitoring** | Logs, métricas, alertas | Base operativa para SRE, troubleshooting y SLAs. |
| Gobierno (mínimo) | **IAM + Data Catalog/Dataplex (opcional)** | Accesos, clasificación, ownership | Control de acceso y base para gobierno sin sobrediseñar al inicio. |

---

## 9. Equivalencias GCP ↔ AWS (alternativa)

| Necesidad | GCP (propuesta) | AWS (alternativa) |
|---|---|---|
| Event bus | Pub/Sub | Kinesis / MSK |
| Stream processing | Dataflow | Kinesis Data Analytics / Flink / Glue Streaming |
| Raw data lake | GCS | S3 |
| Warehouse | BigQuery | Redshift / Athena + Iceberg |
| Orquestación batch | Composer (Airflow) | MWAA (Airflow) |
| Online store | Bigtable | DynamoDB |
| Serving API | Cloud Run | ECS Fargate / Lambda |
| Model serving | Vertex AI Endpoint | SageMaker Endpoint |
