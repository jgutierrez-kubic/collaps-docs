# Payload y contratos

Contrato HTTP activo del orquestador: modelo Pydantic **`AnalysisPayload`** (`app/models/payload.py`).

## Convención de nombres (Refactor Core)

| Capa | Convención | Ejemplo |
|---|---|---|
| Wire HTTP (n8n ↔ Python) | **Inglés + `camelCase`** | `tableA`, `targetTable`, `calculationMethods` |
| Python interno (atributos) | **Inglés + `snake_case`** | `table_a`, `target_table`, `calculation_methods` |

Pydantic usa `alias_generator=to_camel` con `populate_by_name=True`: acepta camelCase (preferido) y también snake_case por compatibilidad. Las respuestas y callbacks salen en **camelCase**.

> Los nombres españoles antiguos (`tabla_a`, `llave_cruce_a`, `metodos_calculo`, …) **ya no son el contrato oficial**.

---

## Características del modelo

| Propiedad | Valor |
|---|---|
| Config | `alias_generator=to_camel`, `populate_by_name=True`, `strict=True`, `extra="forbid"` |
| Forma | Objeto JSON **plano** |
| Campos desconocidos | Rechazados → `422` |

---

## Esquema de campos (wire = camelCase)

| Campo JSON | Atributo Python | Tipo | Req. | Default | Descripción |
|---|---|---|---|---|---|
| `source` | `source` | `"directus"` \| `"n8n"` | No | `"directus"` | Origen del disparo |
| `analysisId` | `analysis_id` | string \| null | No | `null` | ID lógico del análisis |
| `schemaName` | `schema_name` | string | No | `"s00001_incancer"` | Schema PostgreSQL |
| `analysisName` | `analysis_name` | string \| null | No | `null` | Nombre legible; en n8n también genera `targetTable` |
| `tableA` | `table_a` | string | **Sí** | — | Tabla origen A |
| `tableB` | `table_b` | string | **Sí** | — | Tabla origen B |
| `joinKeyA` | `join_key_a` | string | **Sí** | — | Llave de cruce en A |
| `joinKeyB` | `join_key_b` | string | **Sí** | — | Llave de cruce en B |
| `columnsA` | `columns_a` | string (CSV) | **Sí** | — | Columnas de análisis en A |
| `columnsB` | `columns_b` | string (CSV) | **Sí** | — | Columnas de análisis en B |
| `calculationMethods` | `calculation_methods` | string (CSV) | **Sí** | — | `method_id` o alias legacy por par |
| `targetTable` | `target_table` | string | **Sí** | — | Tabla destino (append por corrida; ver nomenclatura) |
| `callbackUrl` | `callback_url` | string \| null | No | `null` | URL HTTP(S) de notificación |

### Mapa de migración (legacy → actual)

| Antiguo (no usar) | Nuevo (wire) |
|---|---|
| `tabla_a` / `tabla_b` | `tableA` / `tableB` |
| `llave_cruce_a` / `llave_cruce_b` | `joinKeyA` / `joinKeyB` |
| `columnas_a` / `columnas_b` | `columnsA` / `columnsB` |
| `metodos_calculo` | `calculationMethods` |
| `tabla_destino` | `targetTable` |
| `nombre_analisis` | `analysisName` |
| `callback_url` | `callbackUrl` |
| `analysis_id` | `analysisId` |
| `schema_name` | `schemaName` |

### Ejemplo mínimo válido

```json
{
  "source": "n8n",
  "analysisId": "n8n_1722096123456",
  "schemaName": "s00001_incancer",
  "analysisName": "Precio Frutas",
  "tableA": "modelo",
  "tableB": "contrato",
  "joinKeyA": "codigo",
  "joinKeyB": "codigo",
  "columnsA": "cantidad,precio",
  "columnsB": "cantidad,precio",
  "calculationMethods": "math_sub,strict_equal",
  "targetTable": "c_results_precioFrutas",
  "callbackUrl": "https://n8n.example.com/webhook-waiting/abc123"
}
```

### Nomenclatura de `targetTable`

| Origen | Convención |
|---|---|
| Nodo Condenser (n8n) | Auto: `c_results_` + camelCase del `analysisName` (vía `tableNameFormatter`) |
| Cliente manual / Flow Directus | Debe enviar un identificador SQL válido; se recomienda el mismo prefijo `c_results_*` |

Ejemplo: Analysis Name `Precio Frutas` → `targetTable: "c_results_precioFrutas"`.

---

## Validadores

| Validador | Campos | Regla |
|---|---|---|
| `normalize_schema_name` | `schema_name` | `None`/blank → default |
| `validate_schema_name` | `schema_name` | Identificador SQL |
| `sanitize_qualified_table_names` | `table_a`, `table_b`, `target_table` | Quita `schema.`; valida identificador |
| `validate_join_keys` | `join_key_a`, `join_key_b` | Identificador SQL |
| `validate_column_lists` | `columns_a`, `columns_b` | CSV no vacío de identificadores |
| `validate_methods` | `calculation_methods` | Cada ítem ∈ registry o `{DIFERENCIA, IGUALDAD}` |

**Cardinalidad:** `len(columnsA) == len(columnsB) == len(calculationMethods)` (exigido al construir el SQL / en n8n Method Configurator).

---

## Contratos de respuesta

### Aceptación síncrona (`202`)

```json
{
  "status": "accepted",
  "jobId": "<uuid4>",
  "analysisId": "<string|null>",
  "message": "Analysis job queued successfully"
}
```

### Callback asíncrono (POST a `callbackUrl`)

Formato exacto enviado por `AnalysisEngine._send_callback`:

```json
{
  "status": "success",
  "analysisId": "n8n_1722096123456",
  "schema": "s00001_incancer",
  "targetTable": "c_results_precioFrutas",
  "updateSchema": false,
  "filas_insertadas": 120,
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "summary": {
    "totalRows": 120,
    "matches": 100,
    "onlyA": 12,
    "onlyB": 8,
    "hasDuplicates": false
  }
}
```

| Campo | Tipo | Notas |
|---|---|---|
| `status` | string | `"success"` \| `"failed"` |
| `analysisId` | string \| null | Eco del payload |
| `schema` | string | `schemaName` del job (clave para Sync Visores) |
| `targetTable` | string | Tabla destino materializada |
| `updateSchema` | **boolean** | `true` solo si hubo cambio de esquema (ver abajo) |
| `filas_insertadas` | int | Filas escritas en todos los chunks del job |
| `jobId` | string | UUID del encolado HTTP (si existe) |
| `summary.*` | object | Acumulado sobre chunks (`totalRows`, `matches`, `onlyA`, `onlyB`, `hasDuplicates`) |

#### Lógica de `updateSchema`

Se inicializa en `False` al crear el engine y se **resetea a `False`** al inicio de cada `run()`. Solo pasa a `True` en estas dos condiciones:

| Condición | Detección en código |
|---|---|
| **Creación de tabla nueva** | Primer chunk con `if_exists="replace"` (la tabla destino no existía) |
| **Adición de columnas** | `_auto_migrate_table()` retorna `True` (ejecutó `ALTER TABLE ... ADD COLUMN`) |

Si el job solo hace `append` sobre un esquema ya alineado → `updateSchema: false` y n8n **no** debe lanzar Meta Sync.

---

## Columnas persistidas en la tabla destino

### Pares indexados (datos)

Por cada par `(columnsA[i], columnsB[i], calculationMethods[i])`:

| Columna | Ejemplo | Significado |
|---|---|---|
| `{i}_{col}A` | `0_cantidadA` | Valor origen A del par |
| `{i}_{col}B` | `0_cantidadB` | Valor origen B del par |
| `{i}_metodo_aplicado` | `0_metodo_aplicado` | Texto del método pedido |
| `{i}_{method}` | `0_math_sub` o `0_diferencia` | Resultado del cálculo |
| `{i}_is_match` | `0_is_match` | Match inferido (si aplica y no es boolean-pure) |

Las columnas SQL intermedias `{col}_a` / `{col}_b` se eliminan tras materializar el bloque indexado.

### Metadatos (siempre al final, derecha)

Orden fijo:

`run_id`, `created_at`, `timestamp`, `job_id`, `estado_cruce`, `analysis_id`, `analysis_name`, `source`

| Campo | Tipo / notas |
|---|---|
| `run_id` | **Entero incremental** por tabla (`MAX(run_id)+1`), único por job, compartido entre chunks |
| `job_id` | UUID del encolado HTTP |
| `estado_cruce` | `Match` \| `Only A` \| `Only B` |
| `timestamp` / `created_at` | Marca UTC de la corrida |

El motor **no** registra la tabla en Directus/NocoDB; el refresh de visores lo hace n8n ([Sync de visores](sync-visores.md)).

---

## Contrato WorkTables (relacionado)

Endpoint aparte: `POST /api/v1/worktables/create` — ver [WorkTables](worktables.md).

## Ver también

- [FastAPI / AnalysisEngine](fastapi-analysisengine.md)
- [Integración n8n](integracion-n8n.md)
