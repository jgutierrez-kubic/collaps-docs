# Visión general

## Qué es COLLAPS

**COLLAPS** (suite *Collaps BIM-OS*) es un sistema de análisis de cruce de datos tabulares orientado a operaciones BIM / back-office. Compara dos tablas PostgreSQL bajo una misma clave de cruce, aplica un catálogo de métodos matemáticos y lógicos fila a fila, persiste el resultado y lo registra automáticamente en Directus para consumo operativo.

El servicio HTTP del motor se publica como **Condenser CORE** (`collaps-C`): FastAPI asíncrono desplegado en Google Cloud Run. La construcción del payload y el disparo del análisis se orquestan desde **n8n** mediante el paquete de nodos custom `n8n-nodes-collaps` (`collaps-n8n-nodes`).

## Propósito

| Objetivo | Cómo lo resuelve |
|---|---|
| Cruzar dos fuentes tabulares | `FULL OUTER JOIN` en PostgreSQL sobre `llave_cruce_a` / `llave_cruce_b` |
| Aplicar lógica de comparación reutilizable | `collaps_engine.execute_transformation` por par de columnas y método |
| No bloquear el flujo de n8n | Endpoint `202 Accepted` + `BackgroundTasks` + callback opcional a `$execution.resumeUrl` |
| Exponer resultados al portal | Append a tabla destino + auto-registro de colección en Directus |
| Evolucionar el esquema de salida | Migración automática de columnas (`ALTER TABLE ... ADD COLUMN IF NOT EXISTS`) |

## Alcance de la suite (repositorios)

| Repositorio | Rol | Stack |
|---|---|---|
| `collaps-C` | Orquestador HTTP + motor matemático (`collaps_engine`) + persistencia | Python 3.10, FastAPI, SQLAlchemy, Pandas |
| `collaps-n8n-nodes` | Nodos n8n que descubren esquema/tablas/columnas, arman el payload y llaman al engine | TypeScript, n8n-workflow, `pg` |
| `collaps-docs` | Documentación de arquitectura (GitBook) | Markdown |

## Flujo conceptual

```text
n8n (nodos Collaps*)
  → POST JSON AnalysisPayload
    → Cloud Run / FastAPI  POST /api/v1/condenser/job  → 202 Accepted
      → AnalysisEngine (background)
        → PostgreSQL FULL OUTER JOIN
        → collaps_engine (por fila / par de columnas)
        → append + auto-migración + PK para Directus
        → Directus POST /collections (idempotente)
        → callback HTTP opcional a n8n
```

## Principios de diseño

1. **Contrato plano y estricto.** El payload activo (`AnalysisPayload`) es un objeto JSON plano validado con Pydantic (`strict=True`, `extra="forbid"`). No admite campos desconocidos.
2. **Append-only.** La tabla destino nunca se reemplaza: cada ejecución añade filas con metadatos de corrida (`run_id`, `created_at`, etc.).
3. **Asincronía fire-and-forget.** El cliente recibe `job_id` de inmediato; el único canal de finalización hacia n8n es `callback_url` (si se envía).
4. **Separación de responsabilidades.** n8n construye y dispara; FastAPI orquesta; `collaps_engine` calcula; PostgreSQL almacena; Directus publica la colección.

## Fuera de alcance (Sprint 1)

- Detalle del catálogo `OPERATIONS_REGISTRY` y firma de cada método → sección **Motor matemático** (pendiente).
- Dockerfile, Cloud Build, secretos y variables de entorno de producción → sección **Despliegue** (pendiente).

## Lectura recomendada

| Documento | Contenido |
|---|---|
| [Componentes y límites](componentes-y-limites.md) | Límites de cada pieza del sistema |
| [Flujo end-to-end](flujo-end-to-end.md) | Ciclo de vida completo de una petición |
| [Payload y contratos](../orquestador/payload-y-contratos.md) | Esquema exacto de `AnalysisPayload` |
