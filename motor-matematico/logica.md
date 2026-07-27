# Métodos lógicos

Dos operaciones para revisar nulos y combinar banderas verdadero/falso.

---

## Resumen

| method_id | En una frase | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `null_check` | ¿Hay valores vacíos (null)? | `dict` de flags | `true` si ninguno es null | — |
| `boolean_logic` | Combina dos sí/no con Y / O / XOR | `bool` | = resultado | `operator` |

---

## `null_check`

### Explicación en lenguaje humano

Antes de restar o comparar, a veces solo quieres saber si **falta un dato**. Este método no calcula una diferencia: diagnostica si A, B, ambos o alguno están en `null`.

Importante: solo mira `null` estricto. Un texto vacío `""` o un cero `0` **no** se consideran null.

**Ejemplo real:** cantidad en modelo = `null`, cantidad en contrato = `5` → hay un valor faltante en A (`any_null: true`). El sistema marca `is_match: false` porque “no están ambos presentes”.

### Retorno (`result_value`)

| Campo | Significado humano |
|---|---|
| `a_is_null` | ¿Falta el valor A? |
| `b_is_null` | ¿Falta el valor B? |
| `both_null` | ¿Faltan los dos? |
| `any_null` | ¿Falta al menos uno? |

`is_match = true` solo cuando **ninguno** es null.

```python
execute_transformation(None, 5, "null_check")
# → any_null: true, is_match: false

execute_transformation("x", "y", "null_check")
# → any_null: false, is_match: true
```

---

## `boolean_logic`

### Explicación en lenguaje humano

Toma dos valores “sí/no” (o cosas que se pueden interpretar como tales) y los combina:

| Operador | Pregunta en lenguaje humano |
|---|---|
| `AND` | ¿Se cumplen **las dos** condiciones? |
| `OR` | ¿Se cumple **al menos una**? |
| `XOR` | ¿Se cumple **exactamente una** (no ambas)? |

**Ejemplos reales**

- `AND`: “¿está aprobado en modelo **y** en contrato?”  
- `OR`: “¿tiene alerta en A **o** en B?”  
- `XOR`: “¿solo uno de los dos lados está marcado?” (inconsistencia)

Números: `0` = falso, distinto de cero = verdadero. Textos como `true`, `yes`, `si`, `1` también cuentan como verdadero.

### Options

| Clave | Default | Valores |
|---|---|---|
| `operator` | `"AND"` | `"AND"`, `"OR"`, `"XOR"` |

```python
execute_transformation(True, False, "boolean_logic", {"operator": "OR"})
# → true

execute_transformation(1, 0, "boolean_logic", {"operator": "AND"})
# → false

execute_transformation(True, False, "boolean_logic", {"operator": "XOR"})
# → true
```
