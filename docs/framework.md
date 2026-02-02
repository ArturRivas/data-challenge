# Data Development Framework (Marco de desarrollo)

> **Objetivo:** acelerar el time-to-market sin sacrificar calidad, mediante un marco de trabajo que estandariza cómo diseñamos, construimos, probamos, desplegamos y operamos pipelines y productos de datos.  
> **Idea clave:** invertir temprano en este framework reduce **deuda técnica**, mejora la **confiabilidad** y habilita **escalabilidad del equipo** (más engineers entregando más rápido con menos regresiones).

> ℹ️ **Alcance:** este documento **no define el framework a detalle**. Su propósito es identificar **el mínimo indispensable** que debe cubrir y su **justificación**.

---

## 1. ¿Por qué es crítico definir un marco de trabajo?

En una plataforma de datos moderna (batch + streaming + ML + real-time), el riesgo principal no es “si funciona hoy”, sino:

- **Inconsistencia** entre pipelines (cada quien construye distinto)
- **Pérdida de tiempo** reinventando soluciones (retry, idempotencia, logging, DQ)
- **Errores silenciosos** (datos incorrectos sin alertas)
- **Operación costosa** (incidentes frecuentes, backfills manuales)
- **Dependencia de personas** (knowledge silos)

Este framework busca que el equipo tenga:
- **Velocidad** (plantillas y componentes listos)
- **Calidad** (testing y validaciones estándar)
- **Confiabilidad** (observabilidad y SLAs)
- **Mantenibilidad** (patrones y diseño consistentes)

---

## 2. Convenciones y estándares de nombrado

### 2.1 ¿Qué incluye?
Convenciones para:
- datasets, tablas, vistas y particiones; buckets y archivos
- topics/streams y subscriptions
- jobs de pipelines y DAGs
- repositorios, paquetes y módulos
- métricas, dimensiones y eventos

Ejemplos (ilustrativos):
- `bronze_events_raw`, `silver_sessions_clean`, `gold_product_usage_daily`
- `topic.user_events.v1`, `sub.user_events.analytics`
- `dlq.user_events.invalid`

### 2.2 ¿Por qué importa?
- Evita ambigüedad y fricción entre equipos (BI / ML / Backend)
- Reduce errores por malinterpretación
- Habilita automatización (catálogo, lineage, alertas, acceso)
- Acelera onboarding de nuevos engineers

---

## 3. Lineamientos y buenas prácticas de desarrollo

### 3.1 Prácticas base (mínimas)
- Code review obligatorio
- Branching strategy (trunk-based o GitFlow simplificado)
- Tests como gate (no deploy si falla)
- Documentación mínima por pipeline
- Definición de ownership (quién opera qué)

### 3.2 ¿Por qué importa?
- Reduce incidentes y regresiones
- Hace que los pipelines sean “operables” (no solo “funcionales”)
- Permite escalar el equipo sin degradar la calidad

---

## 4. Arquetipos importantes (plantillas estándar)

Los arquetipos son “blueprints” repetibles para construir rápido sin reinventar.

### 4.1 Arquetipos recomendados
- Batch ELT (dbt)
- Streaming pipeline (event-driven)
- CDC pipeline
- Serving dataset / Online state
- Dataset de entrenamiento ML

### 4.2 ¿Por qué importa?
- Reduce tiempo de diseño
- Crea consistencia entre pipelines
- Facilita soporte y debugging (mismos patrones)

---

## 5. Componentes reutilizables

La idea es construir librerías comunes (internas) para evitar copy/paste.

### 5.1 Componentes sugeridos (ejemplos)
- Validación y contratos
- Idempotencia y deduplicación
- Observabilidad (métricas/logs estándar)
- Manejo de errores y DLQ
- Conectores (fuentes/sinks comunes)

### 5.2 ¿Por qué importa?
- Menos bugs por soluciones ad-hoc
- Cambios se hacen una vez y benefician a todos
- Acelera entregas sin sacrificar estándares

---

## 6. Arquitectura y patrones de diseño comunes

### 6.1 Mínimo indispensable
- Patrones aprobados para:
  - Ingesta (eventos + CDC)
  - Procesamiento (batch incremental / streaming windows)
  - Publicación (datasets “serving-ready”)
- **ADRs** (Architecture Decision Records) para decisiones relevantes
- Un **árbol de decisión** para elegir tecnologías según caso de uso
- Casos de uso documentados para ingesta, procesamiento y consumo

### 6.2 ¿Por qué importa?
- Reduce discusiones repetidas
- Alinea decisiones y mejora consistencia técnica
- Evita deuda técnica por soluciones “únicas” y difíciles de operar

---

## 7. Esqueletos de proyectos y pipelines CI/CD

### 7.1 Estructura de repositorios (ejemplo)
- Opción A: monorepo
- Opción B: multi-repo por dominio

> Recomendación: empezar con **monorepo** si el equipo es pequeño/mediano y migrar si crece.

---

### 7.2 CI/CD mínimo viable (por tipo de pipeline)

**Para pipelines streaming/batch (Python)**
- lint + formatting (`ruff`, `black`)
- unit tests (`pytest`)
- integration tests (dataset sandbox) *(PR o nightly, según costo)*
- build artefact (container / package)
- deploy automatizado por ambiente (`dev` / `staging` / `prod`)

**Infra as Code**
- Terraform para recursos (topics, datasets, permisos)
- configuración versionada (no manual en consola)

---

### 7.3 Gates de calidad (policy checks)
- No merges sin tests
- Coverage mínimo (donde aplique)
- Escaneo de secretos
- Validación de naming conventions
- Guardrails de costos (ej. particionado obligatorio en tablas grandes)

### 7.4 ¿Por qué importa?
- Reduce fallas en producción
- Despliegues repetibles y auditables
- Acelera releases con confianza

---

## 8. Beneficios esperados (impacto real)

*(Estimado; depende de la madurez del equipo y adopción del framework)*

- **+30–50% velocidad de entrega** al reutilizar arquetipos y componentes
- **Menos incidentes** por validaciones estándar y observabilidad
- **Onboarding más rápido** (1–2 semanas vs 1–2 meses)
- **Costos controlados** por guardrails y mejores prácticas
- **Escala del equipo** sin pérdida de consistencia

---

## 9. Qué NO haría en etapa inicial (evitar sobre-ingeniería)

- Framework excesivamente rígido que frene entregas
- Tooling complejo sin necesidad (ej. catálogo enterprise desde día 1)
- “Perfect lineage” manual
- Feature store enterprise si aún no hay casos productivos claros
- Múltiples repositorios si el equipo aún es pequeño
