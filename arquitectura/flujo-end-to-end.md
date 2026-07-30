# Flujo end-to-end

Ciclo de vida completo: motor desacoplado de CMS + sync de visores en n8n.

## Diagrama de secuencia

```mermaid
sequenceDiagram
  autonumber
  participant Ops as Operador / n8n
  participant Trig as CollapsBttfTrigger
  participant API as bttf-engine
  participant AE as AnalysisEngine
  participant PG as PostgreSQL
  participant Wait as resumeUrl
  participant Sync as Sync Visores
  participant NC as NocoDB
  participant DX as Directus

  Ops->>Trig: Analysis Name + estructura + métodos
  Trig->>API: POST /condenser/job (camelCase)
  API-->>Trig: 202 { jobId }
  API->>AE: BackgroundTasks

  loop Chunks 50.000 filas
    AE->>PG: read / transform / replace|append
  end

  AE->>Wait: POST callback { status, schema, targetTable, updateSchema, filas_insertadas, summary }
  alt updateSchema == true
    Wait->>Sync: Edit Fields → Execute Workflow
    par Meta Sync
      Sync->>NC: POST meta-diff (timeout 120s, 3 retries)
      Sync->>DX: POST schema/diff (timeout 120s, 3 retries)
    end
  else updateSchema == false
    Note over Wait: Sin Sync Visores (solo datos)
  end
```

## Fase 1 — n8n arma y dispara

Mapper → Methods → **CollapsBttfTrigger** (auto `c_results_*` + `callbackUrl`).  
Opcional: **WorkTableGenerator** → `/worktables/create`.

## Fase 2 — Motor (solo datos)

1. `202 Accepted` inmediato.  
2. Chunks de 50k, `run_id` entero, columnas indexadas.  
3. Persistencia PostgreSQL.  
4. **No** hay llamadas a Directus/NocoDB.  
5. Callback HTTP a n8n.

## Fase 3 — Guardia de tráfico + Sync de visores (n8n)

Tras el webhook, n8n evalúa `updateSchema`:

| Valor | Significado | Acción |
|---|---|---|
| `true` | Tabla nueva o columnas añadidas por `_auto_migrate_table` | Edit Fields (`schema`) → Sync Visores |
| `false` | Solo append | Omitir Meta Sync |

Cuando corre el sync: NocoDB ∥ Directus, timeout **120.000 ms**, **3** reintentos.

Ver [Sync de visores](../orquestador/sync-visores.md) y [NocoDB](../infraestructura/nocodb.md).

## Observabilidad

| Artefacto | Dónde |
|---|---|
| Filas de resultado | PostgreSQL (`c_results_*` / `w_table_*`) |
| Señal de fin | Callback camelCase a n8n |
| Catálogo UI | NocoDB / Directus **después** del Meta Sync |

## Referencias

- [FastAPI / AnalysisEngine](../orquestador/fastapi-analysisengine.md)  
- [Payload y contratos](../orquestador/payload-y-contratos.md)  
- [Integración n8n](../orquestador/integracion-n8n.md)
