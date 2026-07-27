# COLLAPS Docs

Documentación de arquitectura de la suite COLLAPS.

## Repositorios

| Carpeta | Rol | Stack |
|---|---|---|
| `collaps-C` | Motor matemático + orquestador BTTF (FastAPI) | Python, FastAPI, SQLAlchemy, Pandas |
| `collaps-n8n-nodes` | Nodos custom n8n (payload → engine) | TypeScript, n8n-workflow |
| `collaps-docs` | Documentación de arquitectura y despliegue | Markdown / GitBook |

## Flujo principal

```text
n8n (nodos Collaps*)
  → POST JSON a Cloud Run
    → FastAPI `/api/v1/condenser/job`
      → AnalysisEngine
        → PostgreSQL + collaps_engine
        → callback opcional a n8n
```

## Índice

Ver [`SUMMARY.md`](./SUMMARY.md) para la estructura de la documentación.
