# Payload y contratos

Contrato HTTP activo del orquestador: modelo Pydantic **`AnalysisPayload`** (`app/models/payload.py`).

## Características del modelo

| Propiedad | Valor |
|---|---|
| Config | `strict=True`, `extra="forbid"` |
| Forma | Objeto JSON **plano** (sin objetos anidados) |
| Campos desconocidos | Rechazados → `422` |
| Tipos incorrectos | Rechazados en modo estricto → `422` |

---

## Esquema de campos

| Campo | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `source` | `"directus"` \| `"n8n"` | No | `"directus"` | Origen del disparo |
| `analysis_id` | string \| null | No | `null` | Identificador lógico del análisis |
| `schema_name` | string | No | `"s00001_incancer"` | Schema PostgreSQL; vacío/`null` → default |
| `nombre_analisis` | string \| null | No | `null` | Etiqueta humana; se persiste en filas si viene |
| `tabla_a` | string | **Sí** | — | Tabla origen A (nombre puro o `schema.tabla`) |
| `tabla_b` | string | **Sí** | — | Tabla origen B |
| `llave_cruce_a` | string | **Sí** | — | Columna clave en A |
| `llave_cruce_b` | string | **Sí** | — | Columna clave en B |
| `columnas_a` | string (CSV) | **Sí** | — | Columnas de análisis en A, separadas por coma |
| `columnas_b` | string (CSV) | **Sí** | — | Columnas de análisis en B (misma cardinalidad) |
| `metodos_calculo` | string (CSV) | **Sí** | — | `method_id` o alias legacy por cada par |
| `tabla_destino` | string | **Sí** | — | Tabla de salida (append) |
| `callback_url` | string \| null | No | `null` | URL HTTP(S) de notificación al terminar |

### Ejemplo mínimo válido

```json
{
  "source": "n8n",
  "analysis_id": "n8n_1722096123456",
  "schema_name": "s00001_incancer",
  "nombre_analisis": "Cruce cantidades vs contrato",
  "tabla_a": "modelo",
  "tabla_b": "contrato",
  "llave_cruce_a": "codigo",
  "llave_cruce_b": "codigo",
  "columnas_a": "cantidad,precio",
  "columnas_b": "cantidad,precio",
  "metodos_calculo": "math_sub,strict_equal",
  "tabla_destino": "c_resultado_cruce",
  "callback_url": "https://n8n.example.com/webhook-waiting/abc123"
}
```

### Ejemplo con tablas calificadas

El cliente puede enviar `schema.tabla`; el validador **elimina el prefijo** y conserva solo el identificador de tabla. El schema efectivo sigue siendo `schema_name`.

```json
{
  "schema_name": "s00001_incancer",
  "tabla_a": "s00001_incancer.modelo",
  "tabla_b": "s00001_incancer.contrato",
  "tabla_destino": "s00001_incancer.c_resultado_cruce",
  "llave_cruce_a": "codigo",
  "llave_cruce_b": "codigo",
  "columnas_a": "cantidad",
  "columnas_b": "cantidad",
  "metodos_calculo": "DIFERENCIA"
}
```

Tras sanitizar: `tabla_a = "modelo"`, `tabla_b = "contrato"`, `tabla_destino = "c_resultado_cruce"`.

---

## Validadores

| Validador | Campos | Regla |
|---|---|---|
| `normalize_schema_name` | `schema_name` | `None` o blank → `"s00001_incancer"` |
| `validate_schema_name` | `schema_name` | Regex `^[a-zA-Z_][a-zA-Z0-9_]*$` |
| `sanitize_qualified_table_names` | `tabla_a`, `tabla_b`, `tabla_destino` | Quita `schema.`; valida identificador |
| `validate_join_keys` | `llave_cruce_a`, `llave_cruce_b` | Identificador SQL seguro |
| `validate_column_lists` | `columnas_a`, `columnas_b` | CSV no vacío; cada ítem es identificador |
| `validate_methods` | `metodos_calculo` | Cada ítem ∈ `OPERATIONS_REGISTRY` (lowercase) o `{DIFERENCIA, IGUALDAD}` |

### Cardinalidad de listas CSV

Aunque Pydantic no impone la igualdad de longitudes en el modelo, **`AnalysisEngine` / `build_analysis_sql` exigen** que:

```text
len(columnas_a) == len(columnas_b) == len(metodos_calculo)
```

tras el split por comas. Un desajuste provoca error en background (no en la validación 422 del endpoint, salvo que un método individual sea inválido).

En n8n, `CollapsMethodConfigurator` sí valida esa igualdad antes del POST.

---

## Métodos permitidos en `metodos_calculo`

### Alias legacy

| Alias | Resolución en engine |
|---|---|
| `DIFERENCIA` | `math_sub` con operandos invertidos (`val_b - val_a`) |
| `IGUALDAD` | `strict_equal` |

### method_id canónicos (nodos n8n)

| Categoría | IDs |
|---|---|
| Math | `math_add`, `math_sub`, `math_diff_abs`, `math_diff_pct`, `math_tolerance`, `math_ratio` |
| Text | `strict_equal`, `normalized_equal`, `fuzzy_levenshtein`, `fuzzy_jaro_winkler`, `contains_check`, `regex_match` |
| Date | `date_diff_seconds`, `date_diff_days`, `date_equal`, `date_tolerance` |
| Array | `array_intersection`, `array_difference`, `array_jaccard` |
| Logic | `null_check`, `boolean_logic` |

> El detalle semántico de cada método se documentará en la sección **Motor matemático**. Aquí solo importa el contrato de nombres aceptados.

---

## Helpers de calificación (no son campos JSON)

```python
payload.qualified_table_a()   # "{schema_name}.{tabla_a}"
payload.qualified_table_b()   # "{schema_name}.{tabla_b}"
payload.qualified_destino()   # "{schema_name}.{tabla_destino}"
```

---

## Contratos de respuesta

### Aceptación síncrona (`202`)

```json
{
  "status": "accepted",
  "job_id": "<uuid4>",
  "analysis_id": "<string|null>",
  "message": "Análisis encolado exitosamente"
}
```

### Callback asíncrono (POST a `callback_url`)

```json
{
  "status": "success",
  "analysis_id": "n8n_1722096123456",
  "schema": "s00001_incancer",
  "summary": {
    "total_rows": 120,
    "matches": 100,
    "only_a": 12,
    "only_b": 8,
    "has_duplicates": false
  }
}
```

| Campo `summary` | Significado |
|---|---|
| `total_rows` | Filas del DataFrame de cruce |
| `matches` | Filas con `estado_cruce = Match` |
| `only_a` / `only_b` | Filas solo en un lado del JOIN |
| `has_duplicates` | Indica si el total de filas sugiere llaves no únicas |

### Error de validación (`422`)

Formato estándar FastAPI:

```json
{
  "detail": [
    {
      "type": "...",
      "loc": ["body", "metodos_calculo"],
      "msg": "Métodos no soportados: foo. Use un method_id de collaps_engine o alias legacy DIFERENCIA/IGUALDAD.",
      "input": "..."
    }
  ]
}
```

---

## Columnas persistidas (contrato de salida en BD)

Además de las columnas del JOIN y de resultado (`{col}__{method}`, `is_match__...`), el engine añade:

| Columna | Origen |
|---|---|
| `run_id` | UUID de la ejecución |
| `created_at` | Timestamp UTC |
| `analysis_id` | Si venía en el payload |
| `nombre_analisis` | Si venía en el payload |
| `source` | Valor del payload |
| `id` | `SERIAL PRIMARY KEY` (si no existía) |

---

## Formatos que **no** son el contrato activo

| Formato | Estado |
|---|---|
| `JobPayload` / módulos `module_00_on`… | Legacy en `bttf_engine.py`; **no** expuesto |
| `column_comparisons[]` (`dynamoMatching.ts`) | Helper interno n8n; el POST real usa CSV plano |

## Ver también

- [FastAPI / AnalysisEngine](fastapi-analysisengine.md)
- [Integración n8n](integracion-n8n.md)
