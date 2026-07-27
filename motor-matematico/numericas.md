# Métodos numéricos

Seis operaciones aritméticas y de tolerancia del `OPERATIONS_REGISTRY`. Todas convierten operandos con `_to_float`; si alguno no es convertible, el resultado suele ser `null` (excepto `math_tolerance`, que devuelve un dict con flags en `false`/`null`).

---

## Resumen

| method_id | Fórmula / semántica | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `math_add` | `a + b` | `float` \| `null` | `null` | — |
| `math_sub` | `a - b` | `float` \| `null` | `null` | — |
| `math_diff_abs` | `\|a - b\|` | `float` \| `null` | `null` | — |
| `math_diff_pct` | `((a - b) / a) * 100` | `float` \| `inf` \| `null` | `null` | — |
| `math_tolerance` | ¿dentro de margen? | `dict` | `is_within_tolerance` | `epsilon`, `tolerance_pct` |
| `math_ratio` | `a / b` | `float` \| `null` | `null` | — |

---

## `math_add`

Suma aritmética.

| | |
|---|---|
| **Entrada** | `val_a`, `val_b` numéricos (o coercibles) |
| **Options** | ninguna |
| **Retorno** | `a + b` o `null` si falla la coerción |

```python
execute_transformation(10, 5, "math_add")
# → result_value: 15.0, is_match: null
```

---

## `math_sub`

Resta aritmética: **A − B**.

| | |
|---|---|
| **Entrada** | `val_a`, `val_b` |
| **Options** | ninguna |
| **Retorno** | `a - b` o `null` |

```python
execute_transformation(10, 3, "math_sub")
# → result_value: 7.0
```

> El alias legacy `DIFERENCIA` invierte operandos antes de llamar a `math_sub` (calcula B − A). Ver [Legacy](legacy.md).

---

## `math_diff_abs`

Diferencia absoluta.

| | |
|---|---|
| **Entrada** | `val_a`, `val_b` |
| **Options** | ninguna |
| **Retorno** | `abs(a - b)` o `null` |

```python
execute_transformation(10, 13, "math_diff_abs")
# → result_value: 3.0
```

---

## `math_diff_pct`

Diferencia porcentual relativa a A:

\[
\frac{a - b}{a} \times 100
\]

| Caso | Retorno |
|---|---|
| Coerción fallida | `null` |
| `a == 0` y `b == 0` | `null` |
| `a == 0` y `b != 0` | `math.inf` |
| Resto | porcentaje `float` |

```python
execute_transformation(100, 80, "math_diff_pct")
# → result_value: 20.0
```

`is_match` siempre `null` (no hay umbral embebido; usar `math_tolerance` para match).

---

## `math_tolerance`

Evalúa si A y B están dentro de un margen absoluto y/o porcentual.

### Options

| Clave | Tipo | Efecto |
|---|---|---|
| `epsilon` | number | Exige `abs(a-b) <= epsilon` |
| `tolerance_pct` | number | Exige `abs((a-b)/a)*100 <= tolerance_pct` (si `a==0`, exige `b==0`) |

Si se pasan ambas, **ambas** deben cumplirse (`within = within and ...`).

### Retorno (`result_value`)

| Campo | Tipo | Descripción |
|---|---|---|
| `is_within_tolerance` | `bool` | Resultado del chequeo |
| `delta_abs` | `float` \| `null` | `abs(a-b)` |
| `delta_pct` | `float` \| `null` | `%` relativo a A; `null` si `a==0` |

`is_match` = `bool(is_within_tolerance)`.

```python
execute_transformation(100, 102, "math_tolerance", {"epsilon": 5})
# → result_value: { is_within_tolerance: true, delta_abs: 2.0, delta_pct: 2.0 }
# → is_match: true
```

Si la coerción falla:

```json
{
  "is_within_tolerance": false,
  "delta_abs": null,
  "delta_pct": null
}
```

---

## `math_ratio`

División A / B.

| Caso | Retorno |
|---|---|
| Coerción fallida | `null` |
| `b == 0` | `null` |
| OK | `a / b` |

```python
execute_transformation(10, 2, "math_ratio")
# → result_value: 5.0
```
