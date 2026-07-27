# Métodos de fechas

Cuatro operaciones para plazos, hitos y marcas de tiempo. Todo se normaliza primero a **UTC**, para que “el mismo instante” no dependa de la zona horaria del servidor.

---

## Resumen

| method_id | En una frase | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `date_diff_seconds` | ¿Cuántos segundos hay entre ambas? | `float` | `null` | — |
| `date_diff_days` | ¿Cuántos días hay entre ambas? | `float` | `null` | — |
| `date_equal` | ¿Caen el mismo día (UTC)? | `bool` | = resultado | — |
| `date_tolerance` | ¿Están dentro de una ventana de tiempo? | `dict` | sí/no | `tolerance_seconds` |

---

## Parseo a UTC

### Explicación en lenguaje humano

Antes de comparar, el motor traduce lo que le llega (texto ISO, fecha sola, epoch…) a un reloj único en UTC. Así `2024-01-01T00:00:00Z` y un timestamp equivalente se tratan igual.

| Tipo de entrada | Comportamiento |
|---|---|
| `datetime` con zona | Se convierte a UTC |
| `datetime` sin zona | Se asume UTC |
| Solo fecha (`date`) | Medianoche UTC de ese día |
| Número (epoch) | Segundos; si es muy grande, milisegundos |
| Texto ISO / `YYYY-MM-DD HH:MM:SS` | Se parsea; `Z` = UTC |
| Vacío / `null` | Error de parseo |

Si el formato no se entiende, la transformación falla con `error` y `result_value: null`.

---

## `date_diff_seconds`

### Explicación en lenguaje humano

Devuelve cuántos **segundos** separan dos momentos (siempre valor positivo: no importa cuál es anterior).

**Ejemplo real:** inicio `00:00` y fin `00:01` del mismo día → `60` segundos.

$$
|\Delta t|_{\text{segundos}}
$$

```python
execute_transformation(
    "2024-01-01T00:00:00Z",
    "2024-01-01T00:01:00Z",
    "date_diff_seconds",
)
# → 60.0
```

---

## `date_diff_days`

### Explicación en lenguaje humano

Igual que la anterior, pero expresada en **días** (puede ser fraccionario: medio día = `0.5`).

**Ejemplo real:** `2024-01-01` vs `2024-01-03` → `2` días.

$$
\frac{|\Delta t|_{\text{segundos}}}{86400}
$$

```python
execute_transformation("2024-01-01", "2024-01-03", "date_diff_days")
# → 2.0
```

---

## `date_equal`

### Explicación en lenguaje humano

Pregunta solo: **¿es el mismo día calendario en UTC?** La hora se ignora.

**Ejemplo real:** `2024-01-01 00:00` y `2024-01-01 23:59` → **sí** (mismo día).  
`2024-01-01` y `2024-01-02` → **no**.

Útil para hitos de entrega (“¿se reportó el mismo día?”) sin exigir la misma hora exacta.

```python
execute_transformation("2024-01-01 00:00", "2024-01-01 23:59", "date_equal")
# → true
```

---

## `date_tolerance`

### Explicación en lenguaje humano

En la práctica dos sistemas pueden registrar el mismo evento con unos minutos de diferencia (sincronización, husos, redondeo). Este método pregunta: **¿la diferencia cabe en mi ventana?**

**Ejemplo real:** evento A a las `00:00` y evento B a las `00:30`, tolerancia 1 hora (`3600` s) → **sí** (solo 1800 s de diferencia).

### Options

| Clave | Default | Descripción |
|---|---|---|
| `tolerance_seconds` | `0` | Máximo de segundos permitidos (0 = deben ser exactos) |

### Retorno (`result_value`)

| Campo | Tipo | Descripción |
|---|---|---|
| `is_within_tolerance` | `bool` | ¿Dentro de la ventana? |
| `delta_seconds` | `float` | Distancia real en segundos |

`is_match` = `is_within_tolerance`.

```python
execute_transformation(
    "2024-01-01T00:00:00Z",
    "2024-01-01T00:30:00Z",
    "date_tolerance",
    {"tolerance_seconds": 3600},
)
# → is_within_tolerance: true, delta_seconds: 1800.0
```
