# Flujo end-to-end

Ciclo de vida Release Stable: motor híbrido Pandas/Polars, alias SQL indexados, guardia `updateSchema`.

## Diagrama de secuencia

```mermaid
sequenceDiagram
  autonumber
  participant Ops as Operador / n8n
  participant Trig as CollapsBttfTrigger
  participant API as bttf-engine
  participant QB as QueryBuilder
  participant AE as AnalysisEngine
  participant PL as polars_transformer
  participant PG as PostgreSQL
  participant Wait as resumeUrl
  participant Sync as Sync Visores

  Ops->>Trig: Analysis Name + estructura + métodos
  Trig->>API: POST /condenser/job (camelCase)
  API-->>Trig: 202 { jobId }
  API->>AE: BackgroundTasks
  AE->>QB: build_analysis_sql (aliases 0_col_a, 1_col_a, …)

  loop Páginas LIMIT/OFFSET (SQL_CHUNK_SIZE)
    AE->>PG: read página → Pandas (conexión corta)
    AE->>PL: from_pandas → transform Polars → to_pandas
    AE->>PG: persist replace|append
  end

  AE->>Wait: callback { schema, targetTable, updateSchema, filas_insertadas, … }
  alt updateSchema == true
    Wait->>Sync: Edit Fields → Meta Sync
  else false
    Note over Wait: Sin Sync Visores
  end
```

## Fase 1 — n8n arma y dispara

Mapper → Methods → **CollapsBttfTrigger** (`c_results_*` + `callbackUrl`).  
El contrato JSON **no cambia** con Polars ni con los alias indexados.

## Fase 2 — Motor (datos)

1. `202 Accepted` inmediato (**requiere** `--no-cpu-throttling` en Cloud Run para no asfixiar el background).  
2. SQL con aliases `{pairIndex}_{col}_a` / `{pairIndex}_{col}_b` (anti-colisión).  
3. Por página: leer Pandas → Polars → transformar → Pandas → escribir.  
4. Pool fail-fast; sin llamadas a CMS.  
5. Callback con `updateSchema` / `filas_insertadas`.

## Fase 3 — Guardia de tráfico + Sync Visores

| `updateSchema` | Acción |
|---|---|
| `true` | Edit Fields (`schema`) → Sync Visores |
| `false` | Omitir Meta Sync |

## Puente de datos Pandas ↔ Polars

```mermaid
flowchart LR
  A[SQL chunk] --> B[Pandas]
  B --> C[pl.from_pandas]
  C --> D[Polars exprs / map_elements]
  D --> E[to_pandas]
  E --> F[SQLAlchemy insert]
```

`pyarrow` acelera el round-trip. Ver [FastAPI / AnalysisEngine](../orquestador/fastapi-analysisengine.md).

## Referencias

- [Payload y contratos](../orquestador/payload-y-contratos.md)  
- [Integración n8n](../orquestador/integracion-n8n.md)  
- [Cloud Run](../despliegue/cloud-run.md)
