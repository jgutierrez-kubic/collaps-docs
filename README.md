# COLLAPS Docs

Documentación técnica de la suite COLLAPS (GitBook).

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
        → Directus + callback opcional a n8n
```

## Índice publicado

Ver [`SUMMARY.md`](./SUMMARY.md).

### Arquitectura

- [Visión general](arquitectura/vision-general.md)
- [Componentes y límites](arquitectura/componentes-y-limites.md)
- [Flujo end-to-end](arquitectura/flujo-end-to-end.md)

### Orquestador

- [FastAPI / AnalysisEngine](orquestador/fastapi-analysisengine.md)
- [Payload y contratos](orquestador/payload-y-contratos.md)
- [Integración n8n](orquestador/integracion-n8n.md)

### Motor matemático

- [Conceptos base](motor-matematico/conceptos-base.md)
- [Numéricas](motor-matematico/numericas.md)
- [Texto](motor-matematico/texto.md)
- [Fechas](motor-matematico/fechas.md)
- [Listas](motor-matematico/listas.md)
- [Lógica](motor-matematico/logica.md)
- [Legacy](motor-matematico/legacy.md)

### Guías prácticas y ejemplos

- [Directus end-to-end](guias-practicas/directus-end-to-end.md)

### Despliegue

- [Variables de entorno](despliegue/variables-entorno.md)
- [Cloud Run](despliegue/cloud-run.md)
- [Docker / Cloud Build](despliegue/docker-cloud-build.md)
