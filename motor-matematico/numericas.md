# Métodos numéricos

Seis operaciones para cantidades, precios, metros, porcentajes y márgenes de tolerancia. Si un valor no se puede leer como número, el resultado suele ser vacío (`null`), salvo en `math_tolerance`, que devuelve un detalle con flags.

---

## Resumen

| method_id | En una frase | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `math_add` | Suma A + B | `float` \| `null` | `null` | — |
| `math_sub` | Resta A − B | `float` \| `null` | `null` | — |
| `math_diff_abs` | Distancia numérica (siempre positiva) | `float` \| `null` | `null` | — |
| `math_diff_pct` | Diferencia en % respecto de A | `float` \| `inf` \| `null` | `null` | — |
| `math_tolerance` | ¿Están lo bastante cerca? | `dict` | sí/no | `epsilon`, `tolerance_pct` |
| `math_ratio` | Cociente A ÷ B | `float` \| `null` | `null` | — |

---

## `math_add`

### Explicación en lenguaje humano

Suma dos números. Útil cuando quieres un total: metros de modelo + metros de ajuste, o dos partidas que deben consolidarse.

**Ejemplo real:** cantidad modelo `120` + cantidad adicional `15` → `135`.

| | |
|---|---|
| **Entrada** | `val_a`, `val_b` numéricos |
| **Options** | ninguna |
| **Retorno** | `a + b` o `null` |

$$
a + b
$$

```python
execute_transformation(10, 5, "math_add")
# → result_value: 15.0, is_match: null
```

---

## `math_sub`

### Explicación en lenguaje humano

Resta **A menos B**. Es el método típico para ver cuánto “sobra” o “falta” del lado A respecto al B.

**Ejemplo real:** cantidad en modelo `100` y cantidad en contrato `95` → diferencia `5` (el modelo tiene 5 unidades más).

> El alias legacy `DIFERENCIA` hace lo contrario en el orden (B − A). Ver [Legacy](legacy.md).

| | |
|---|---|
| **Entrada** | `val_a`, `val_b` |
| **Options** | ninguna |
| **Retorno** | `a - b` o `null` |

$$
a - b
$$

```python
execute_transformation(10, 3, "math_sub")
# → result_value: 7.0
```

---

## `math_diff_abs`

### Explicación en lenguaje humano

Mide **qué tan lejos** están dos números, sin importar quién es mayor. El resultado siempre es positivo (o cero).

**Ejemplo real:** precio A `13` y precio B `10` → distancia `3`. Da igual si A es mayor o B.

$$
|a - b|
$$

```python
execute_transformation(10, 13, "math_diff_abs")
# → result_value: 3.0
```

---

## `math_diff_pct`

### Explicación en lenguaje humano

Responde: “¿en qué porcentaje se desvía B respecto de A?”. Usa A como referencia (el 100%).

**Ejemplo real:** presupuesto A `100` y ejecutado B `80` → `20` (un 20% menos que la referencia).

$$
\frac{a - b}{a} \times 100
$$

| Caso | Retorno |
|---|---|
| No se pueden leer como número | `null` |
| `a == 0` y `b == 0` | `null` |
| `a == 0` y `b != 0` | infinito (`inf`) |
| Resto | porcentaje `float` |

```python
execute_transformation(100, 80, "math_diff_pct")
# → result_value: 20.0
```

`is_match` siempre es `null`. Si necesitas un sí/no por margen, usa `math_tolerance`.

---

## `math_tolerance`

### Explicación en lenguaje humano

En obra o contratos casi nunca exigimos igualdad exacta: un redondeo de 0.01 o un 2% suele ser aceptable. Este método pregunta: **¿la diferencia cabe dentro del margen que yo definí?**

Puedes fijar:

- un margen **absoluto** (`epsilon`), p. ej. “hasta 5 unidades”, y/o
- un margen **porcentual** (`tolerance_pct`), p. ej. “hasta 2%”.

Si defines ambos, **deben cumplirse los dos**.

**Ejemplo real:** cantidad A `100`, B `102`, margen absoluto `5` → sí están dentro (±2 unidades).

### Options

| Clave | Tipo | Efecto |
|---|---|---|
| `epsilon` | number | Exige $$|a - b| \le \epsilon$$ |
| `tolerance_pct` | number | Exige que el % de desviación ≤ el valor; si `a = 0`, exige `b = 0` |

### Retorno (`result_value`)

| Campo | Tipo | Descripción |
|---|---|---|
| `is_within_tolerance` | `bool` | ¿Pasó el chequeo? |
| `delta_abs` | `float` \| `null` | Distancia absoluta |
| `delta_pct` | `float` \| `null` | Distancia en %; `null` si `a = 0` |

`is_match` copia `is_within_tolerance`.

```python
execute_transformation(100, 102, "math_tolerance", {"epsilon": 5})
# → is_within_tolerance: true, delta_abs: 2.0, delta_pct: 2.0
# → is_match: true
```

---

## `math_ratio`

### Explicación en lenguaje humano

Divide A entre B. Sirve para ratios: “¿cuántas veces cabe B en A?”, rendimiento, factor de conversión, etc.

**Ejemplo real:** 10 m² de acabado / 2 m² por unidad → ratio `5`.

Si B es cero, no se puede dividir → `null`.

$$
\frac{a}{b}
$$

```python
execute_transformation(10, 2, "math_ratio")
# → result_value: 5.0
```
