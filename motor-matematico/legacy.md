# Alias legacy — `DIFERENCIA` e `IGUALDAD`

Compatibilidad con workflows n8n / payloads antiguos. **No** son claves de `OPERATIONS_REGISTRY`; el orquestador los traduce antes de llamar a `execute_transformation`.

## Dónde se aceptan

| Capa | Comportamiento |
|---|---|
| `AnalysisPayload.validate_methods` | Permite los literales `DIFERENCIA` e `IGUALDAD` (además de los 21 `method_id`) |
| `AnalysisEngine._resolve_method` | Mapea al método canónico (+ flag de swap de operandos) |
| `CollapsMethodConfigurator` (n8n) | Expone ambos alias en el selector de métodos |
| `OPERATIONS_REGISTRY` | **No** contiene estas claves |

---

## Tabla de resolución

Definida en `app/core/analysis_engine.py`:

```python
_LEGACY_METHOD_MAP = {
    "DIFERENCIA": ("math_sub", True),   # swap_operands=True
    "IGUALDAD":   ("strict_equal", False),
}
```

| Alias (CSV en `metodos_calculo`) | method_id efectivo | Swap operandos | Semántica resultante |
|---|---|---|---|
| `DIFERENCIA` | `math_sub` | **Sí** | `val_b - val_a` (B − A) |
| `IGUALDAD` | `strict_equal` | No | `val_a == val_b` |

La comparación del alias es case-insensitive vía `.strip().upper()` en el resolver.

---

## `DIFERENCIA`

### Por qué existe el swap

Históricamente COLLAPS reportaba la diferencia como **B − A** (referencia menos modelo, o contrato menos cantidad, según el dominio). El método canónico `math_sub` calcula **A − B**. El flag `swap_operands=True` invierte los valores **antes** de llamar al engine:

```python
if swap_operands:
    val_a, val_b = val_b, val_a
execute_transformation(val_a, val_b, "math_sub")
```

### Ejemplo

| `val_a` (col A) | `val_b` (col B) | Alias | Cálculo efectivo | `result_value` |
|---|---|---|---|---|
| `10` | `3` | `DIFERENCIA` | `3 - 10` | `-7.0` |
| `10` | `3` | `math_sub` | `10 - 3` | `7.0` |

### Retorno / match

Igual que [`math_sub`](numericas.md#math_sub): `float` \| `null`, `is_match: null`.  
Nombre de columna de resultado usa el **method_id canónico** (`math_sub`), no el alias.

---

## `IGUALDAD`

Traducción directa a [`strict_equal`](texto.md#strict_equal) sin invertir operandos.

| `val_a` | `val_b` | `result_value` | `is_match` |
|---|---|---|---|
| `"abc"` | `"abc"` | `true` | `true` |
| `1` | `"1"` | `false` | `false` |

No aplica normalización de texto (para eso usar `normalized_equal`).

---

## Uso en el payload

```json
{
  "columnas_a": "cantidad,nombre",
  "columnas_b": "cantidad,nombre",
  "metodos_calculo": "DIFERENCIA,IGUALDAD"
}
```

Equivalente canónico (misma semántica de DIFERENCIA requiere conciencia del orden):

```json
{
  "metodos_calculo": "math_sub,strict_equal"
}
```

> Si migras de `DIFERENCIA` a `math_sub` **sin** ajustar el orden de columnas, el signo del resultado se invierte.

---

## Recomendación

| Escenario | Preferir |
|---|---|
| Workflows nuevos | `method_id` canónicos (`math_sub`, `strict_equal`, …) |
| Workflows legacy ya en producción | Mantener alias hasta migrar con prueba de regresión de signo |
| Documentación / tests | Nombrar siempre el método canónico y anotar el swap si aplica |
