# COLLAPS Docs

Documentación técnica de la suite COLLAPS (GitBook).

## Repositorios

| Carpeta | Rol | Stack |
|---|---|---|
| `collaps-C` | Motor + orquestador + worktables | Python, FastAPI, Pandas |
| `collaps-n8n-nodes` | Nodos Condenser + WorkTableGenerator | TypeScript, n8n |
| `collaps-docs` | Documentación GitBook | Markdown |

## Contrato actual (Refactor Core)

- Wire HTTP: **inglés + camelCase** (`tableA`, `calculationMethods`, `targetTable`, …)
- Cruces → tablas `c_results_*`
- WorkTables → tablas `w_table_*`
- Chunks 50k + `run_id` entero incremental
- Callbacks: `jobId`, `totalRows`, `onlyA`, `hasDuplicates`, …

## Flujo principal

```text
n8n → POST /api/v1/condenser/job (camelCase)
    → AnalysisEngine (chunks) → c_results_* → Directus + callback

n8n → POST /api/v1/worktables/create (camelCase)
    → WorktableEngine → w_table_*
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

### Motor matemático

- [Conceptos base](motor-matematico/conceptos-base.md) · [Numéricas](motor-matematico/numericas.md) · [Texto](motor-matematico/texto.md) · [Fechas](motor-matematico/fechas.md) · [Listas](motor-matematico/listas.md) · [Lógica](motor-matematico/logica.md) · [Legacy](motor-matematico/legacy.md)

### Guías

- [Directus end-to-end](guias-practicas/directus-end-to-end.md)

### Despliegue

- [Variables de entorno](despliegue/variables-entorno.md) · [Cloud Run](despliegue/cloud-run.md) · [Docker / Cloud Build](despliegue/docker-cloud-build.md)
