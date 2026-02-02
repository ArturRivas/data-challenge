# Roadmap (Plan por fases)

> **Objetivo:** entregar valor incremental mientras maduramos en paralelo  
> (1) la **plataforma de datos** y (2) el **marco de desarrollo** que la hace escalable y confiable.  
> **Principio:** cada fase agrega capacidades, pero también **endurece** estándares, calidad y operación.

---

## Fase 0 — Fundamentos mínimos

### Enfoque
Establecer el **mínimo común** para construir rápido sin perder control.

### Plataforma de datos (mínimo viable)
- Definir capas (Bronze/Silver/Gold) y criterios de uso
- Estructura base para ingesta batch/stream (sin sobre-ingeniería)
- Primeros pipelines “hello world” para validar el flujo end-to-end

### Marco de desarrollo (mínimo viable)
- Convenciones de nombrado y organización de repositorios
- CI/CD básico (tests + validaciones como gate)
- Observabilidad mínima (logs/alertas de fallas)
- Seguridad mínima (ambientes y permisos)

**Resultado:** el equipo puede entregar sin improvisar cada pipeline.

---

## Fase 1 — Primer valor (Analítica confiable)

### Enfoque
Habilitar analítica de producto con datasets confiables y reutilizables.

### Plataforma de datos (maduración)
- Ingesta estable (eventos + fuentes core)
- Silver por dominios prioritarios
- Primeros data products en Gold (métricas y reportes críticos)
- Backfills básicos y re-ejecuciones controladas

### Marco de desarrollo (maduración)
- Estándares aplicados en todos los pipelines (no solo “recomendaciones”)
- Data Quality mínimo para datasets críticos (freshness, duplicados, schema)
- Ownership claro por dataset/pipeline
- Runbooks básicos para operación

**Resultado:** insights útiles + datos con garantías mínimas.

---

## Fase 2 — Escalamiento (Cobertura + Operación)

### Enfoque
Aumentar cobertura de datos y confiabilidad operacional a escala.

### Plataforma de datos (maduración)
- Expansión de dominios y data products
- Incorporación gradual de CDC donde agregue valor
- Pipelines incrementales y optimización de performance/costos
- Mayor estandarización del consumo (BI/Producto/ML)

### Marco de desarrollo (maduración)
- Observabilidad completa (latencia, fallas, anomalías, SLAs)
- Gobierno pragmático (clasificación, permisos, trazabilidad mínima)
- Guardrails de costos y performance
- Reprocesos/backfills como capacidad “de rutina”, no “de emergencia”

**Resultado:** plataforma estable, operable y preparada para crecer.

---

## Fase 3 — Real-Time & ML en producción (Personalización)

### Enfoque
Cerrar el loop entre comportamiento → features → inferencia → feedback.

### Plataforma de datos (maduración)
- Streaming para señales críticas (hot path)
- Serving/Online store para estado y features en tiempo real
- Integración formal con ML (datasets reproducibles + serving)
- Data products orientados a experimentación (A/B testing)

### Marco de desarrollo (maduración)
- SLAs/SLOs para data products críticos y features online
- Estrategias de resiliencia (fallbacks, degradación controlada)
- Auditoría y seguridad avanzada para datos sensibles
- Mejora continua basada en incidentes y métricas operativas

**Resultado:** personalización real-time confiable + plataforma lista para ML a escala.

---

## Resumen (idea clave)

- Las fases no solo agregan “más pipelines”.  
- Cada fase madura **dos cosas al mismo tiempo**:
  1) la **plataforma de datos** (capacidades)  
  2) el **marco de desarrollo** (velocidad + calidad + operación)

Esto evita que el crecimiento del producto se traduzca en deuda técnica o pérdida de confianza en los datos.
