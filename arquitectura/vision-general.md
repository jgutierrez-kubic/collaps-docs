# Visión general

## Qué es COLLAPS

**COLLAPS** (suite *Collaps BIM-OS*) cruza dos tablas PostgreSQL, aplica un catálogo de métodos matemáticos/lógicos y materializa resultados en Postgres. Los **visores** (NocoDB / Directus) se actualizan por separado desde n8n.

El backend **Condenser CORE** (`bttf-engine` / `collaps-C`) se dedica al cómputo y la persistencia. n8n orquesta el payload, recibe el callback y sincroniza las UIs.

## Separation of Concerns (actual)

| Responsabilidad | Quién |
|---|---|
| Procesar Pandas, `run_id`, chunking 50k, escribir PG | Motor Python |
| Webhook de retorno `status: success` | Motor → n8n |
| Meta Sync / introspección de visores | n8n (`sync-visores-nocodb-directus`) |
| Exponer subconjunto de tablas al UI | PostgreSQL (`nocodb_light` + política) |

El motor **no** habla con Directus ni NocoDB (sin auto-registro de colecciones, sin `DIRECTUS_URL`).

## Propósito

| Objetivo | Cómo lo resuelve |
|---|---|
| Cruzar dos fuentes | `FULL OUTER JOIN` por `joinKeyA` / `joinKeyB` |
| Comparar columnas | `collaps_engine` vía `calculationMethods` |
| No bloquear n8n | `202` + `BackgroundTasks` + `callbackUrl` |
| Evitar OOM | Chunks de 50.000 filas |
| Nombrar resultados | `c_results_*` / `w_table_*` |
| Refrescar UIs | Sub-workflow de sync de visores |

## Flujo conceptual

```text
n8n → POST /api/v1/condenser/job
    → AnalysisEngine (chunks) → PostgreSQL
    → callback { status, schema, targetTable, updateSchema, filas_insertadas, ... }
        → IF updateSchema: Edit Fields → Sync Visores (NocoDB ∥ Directus)
```

## Lectura recomendada

| Documento | Contenido |
|---|---|
| [Componentes y límites](componentes-y-limites.md) | Quién hace qué |
| [Flujo end-to-end](flujo-end-to-end.md) | Ciclo de vida |
| [Sync de visores](../orquestador/sync-visores.md) | Meta Sync en n8n |
| [NocoDB](../infraestructura/nocodb.md) | Despliegue y rol `nocodb_light` |
