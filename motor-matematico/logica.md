# Métodos lógicos

Dos operaciones de estructura y lógica booleana.

---

## Resumen

| method_id | Semántica | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `null_check` | Diagnóstico de nulos | `dict` de flags | `not any_null` | — |
| `boolean_logic` | AND / OR / XOR | `bool` | = resultado | `operator` |

Ambos están en `_BOOLEAN_PURE_METHODS` del orquestador: no se materializa columna `is_match__` adicional (para `null_check` el indicador queda en el dict / en `is_match` del resultado de `execute_transformation`).

---

## `null_check`

Inspecciona nulidad de ambos operandos (`is None` estricto; no trata `""` ni `NaN` como null).

### Retorno (`result_value`)

| Campo | Tipo | Significado |
|---|---|---|
| `a_is_null` | `bool` | `val_a is None` |
| `b_is_null` | `bool` | `val_b is None` |
| `both_null` | `bool` | Ambos null |
| `any_null` | `bool` | Al menos uno null |

### `is_match`

```text
is_match = not result_value["any_null"]
```

Es decir: **match** solo si ninguno es null.

```python
execute_transformation(None, 5, "null_check")
# → result_value: {
#      a_is_null: true, b_is_null: false,
#      both_null: false, any_null: true
#    }
# → is_match: false

execute_transformation("x", "y", "null_check")
# → any_null: false, is_match: true
```

---

## `boolean_logic`

Combina A y B como booleanos con el operador elegido.

### Coerción `_to_bool`

| Entrada | Resultado |
|---|---|
| `bool` | Tal cual |
| `int` / `float` | `!= 0` |
| `str` | `true` si (lower/strip) ∈ `{1, true, t, yes, y, si, sí}` |
| Otro | `bool(value)` |

### Options

| Clave | Default | Valores |
|---|---|---|
| `operator` | `"AND"` | `"AND"`, `"OR"`, `"XOR"` (case-insensitive; otro valor cae en AND) |

### Retorno

| Operador | Expresión |
|---|---|
| `AND` | `a and b` |
| `OR` | `a or b` |
| `XOR` | `a ^ b` |

`is_match` = el booleano resultante.

```python
execute_transformation(True, False, "boolean_logic", {"operator": "OR"})
# → result_value: true, is_match: true

execute_transformation(1, 0, "boolean_logic", {"operator": "AND"})
# → result_value: false

execute_transformation(True, False, "boolean_logic", {"operator": "XOR"})
# → result_value: true
```
