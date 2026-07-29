# COLLAPS Docs

Documentación técnica de la suite COLLAPS (GitBook).

## Separation of Concerns (actual)

| Capa | Responsabilidad |
|---|---|
| **bttf-engine** (Python) | Pandas + chunking 50k + `run_id` + PostgreSQL + callback a n8n |
| **n8n** | Payload, orquestación, **Sync Visores** (NocoDB ∥ Directus) |
| **PostgreSQL** | Datos + exposición a NocoDB vía rol `nocodb_light` |
| **NocoDB / Directus** | UI; se refrescan por Meta Sync desde n8n, **no** desde Python |

## Repositorios

| Carpeta | Rol |
|---|---|
| `collaps-C` | Motor Condenser + WorkTables |
| `collaps-n8n-nodes` | Nodos Collaps* + workflow `sync-visores-nocodb-directus.json` |
| `collaps-docs` | Esta documentación |

## Flujo principal

```text
n8n → POST /api/v1/condenser/job
    → AnalysisEngine → PostgreSQL
    → callback { status: "success" }
        → Sync Visores (timeout 120s, 3 retries)
```

## Índice

Ver [`SUMMARY.md`](./SUMMARY.md).

### Arquitectura

- [Visión general](arquitectura/vision-general.md)
- [Componentes y límites](arquitectura/componentes-y-limites.md)
- [Flujo end-to-end](arquitectura/flujo-end-to-end.md)

### Orquestador

- [FastAPI / AnalysisEngine](orquestador/fastapi-analysisengine.md)
- [Payload y contratos](orquestador/payload-y-contratos.md)
- [Integración n8n](orquestador/integracion-n8n.md)
- [WorkTables](orquestador/worktables.md)
- [Sync de visores](orquestador/sync-visores.md)

### Infraestructura

- [NocoDB — Despliegue y mantención](infraestructura/nocodb.md)

### Guías

- [Directus end-to-end](guias-practicas/directus-end-to-end.md)

### Motor matemático · Despliegue

Ver `SUMMARY.md` para el listado completo.
