# Visión general

## Qué es COLLAPS

**COLLAPS** (suite *Collaps BIM-OS*) cruza dos tablas PostgreSQL, aplica un catálogo de métodos matemáticos/lógicos y materializa resultados consumibles desde Directus. Opcionalmente genera **tablas de trabajo** (`w_table_*`) agrupadas.

El backend **Condenser CORE** (`collaps-C`) expone FastAPI en Cloud Run. n8n (`collaps-n8n-nodes`) construye el payload y dispara los jobs.

## Propósito

| Objetivo | Cómo lo resuelve |
|---|---|
| Cruzar dos fuentes | `FULL OUTER JOIN` por `joinKeyA` / `joinKeyB` |
| Comparar columnas | `collaps_engine` vía `calculationMethods` |
| No bloquear n8n | `202` + `BackgroundTasks` + `callbackUrl` |
| Escalar sin OOM | Lectura en chunks de 50.000 filas + `df.apply` |
| Nombrar resultados | `c_results_<camelCase>` (cruces) / `w_table_<camelCase>` (worktables) |
| Contrato estable | HTTP 100% inglés + **camelCase** |

## Repositorios

| Carpeta | Rol |
|---|---|
| `collaps-C` | Orquestador + motor + worktables |
| `collaps-n8n-nodes` | Nodos Condenser + WorkTableGenerator |
| `collaps-docs` | GitBook |

## Flujo conceptual

```text
n8n (Mapper → Methods → BttfTrigger)
  → POST camelCase AnalysisPayload
    → /api/v1/condenser/job → 202 { jobId }
      → AnalysisEngine (chunks 50k)
        → columnas indexadas + run_id int
        → Directus + callback camelCase

n8n (WorkTableGenerator)  [opcional / paralelo]
  → POST /api/v1/worktables/create → 202
      → WorktableEngine → w_table_*
```

## Principios de diseño

1. **Wire en inglés camelCase**; Python interno en snake_case.  
2. **Append por corrida** con `run_id` incremental entero (no UUID).  
3. **Fire-and-forget** con callback opcional.  
4. **Nombres de tabla generados** desde nombres amigables (`tableNameFormatter`).

## Lectura recomendada

| Documento | Contenido |
|---|---|
| [Componentes y límites](componentes-y-limites.md) | Quién hace qué |
| [Flujo end-to-end](flujo-end-to-end.md) | Ciclo de vida |
| [Payload y contratos](../orquestador/payload-y-contratos.md) | JSON oficial |
| [Guía Directus](../guias-practicas/directus-end-to-end.md) | Uso sin programar |
