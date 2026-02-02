# Reliability & Governance (Calidad, Observabilidad, Seguridad y Costos)

> **Objetivo:** asegurar datos **confiables, trazables, seguros y operables**, con control de **costos**.

> ℹ️ **Alcance:** este documento **no define la estrategia a detalle**. Solo lista **lo mínimo indispensable** que debe cubrir una estrategia de confiabilidad y gobierno.

---

## 1. Mínimo indispensable (MVP)

### 1. Calidad de datos
**Mínimo**
- Freshness (llegan a tiempo)
- Volumen (anomalías)
- Duplicados (por `event_id`)
- Nulls críticos (IDs/timestamps)
- Schema drift

**Ideal**
- Integridad referencial
- Validity por reglas de negocio

**Herramientas**
- dbt tests (core)
- BigQuery checks / INFORMATION_SCHEMA
- Great Expectations (solo si se requiere)

**Justificación**
- Evita datos “silenciosamente incorrectos” y protege BI/ML.

---

### 2. Contratos y esquemas
**Mínimo**
- Contrato base de eventos: `event_id`, `event_name`, `event_timestamp`, `user_id/anonymous_id`, `session_id`, `platform`, `app_version`
- Versionado simple (`schema_version`)

**Ideal**
- Catálogo de eventos con ownership y cambios controlados

**Herramientas**
- Documentación + validación en ingestión
- dbt docs

**Justificación**
- Reduce ambigüedad y evita romper consumidores.

---

### 3. Observabilidad y SLAs
**Mínimo**
- Alertas por: fallas de pipeline + breach de freshness + DLQ
- SLAs solo para datasets/pipelines críticos (3–5)

**Ideal**
- SLOs por latencia/disponibilidad por dominio

**Herramientas**
- Cloud Monitoring + Logging
- métricas Dataflow
- resultados de dbt

**Justificación**
- Menos tiempo en firefighting, más predictibilidad.

---

### 4. Lineage y ownership
**Mínimo**
- Owner por dataset/pipeline
- Trazabilidad básica: fuente → transformación → salida

**Ideal**
- Lineage automático end-to-end

**Herramientas**
- dbt lineage
- Dataplex / Data Catalog (tags)

**Justificación**
- Debug más rápido y menos dependencia de conocimiento tribal.

---

### 5. Seguridad y privacidad
**Mínimo**
- Clasificación: `Internal` / `Restricted (PII)`
- Acceso por rol (mínimo privilegio)
- PII con masking/hashing

**Ideal**
- Row-level security / policy tags

**Herramientas**
- IAM + permisos por dataset
- views de BigQuery + Audit Logs

**Justificación**
- Reduce riesgo sin frenar el uso de datos.

---

### 6. Costos (FinOps)
**Mínimo**
- Tablas grandes particionadas
- Budgets + alertas
- Revisión periódica de queries costosas

**Ideal**
- Guardrails por dominio + optimización continua

**Herramientas**
- BigQuery partitioning/clustering
- Budgets & alerts + job history

**Justificación**
- Evita crecimiento de costo por “accidentes” y mal uso.

---

### 7. Resiliencia y recuperación
**Mínimo**
- DLQ + retries con backoff
- Idempotencia (dedupe por `event_id`)
- Replay/backfill desde raw

**Ideal**
- Publicación “atomic” en marts (swap)

**Herramientas**
- Pub/Sub DLQ
- Dataflow retries/checkpoints
- GCS raw immutable + dbt backfills

**Justificación**
- Recuperación rápida sin intervención manual.

---
