# Métodos de fechas

Cuatro operaciones sobre marcas de tiempo. Toda conversión pasa por `parse_to_utc_datetime` (`collaps_engine/datetime_parser.py`), que normaliza a **UTC explícito**.

---

## Resumen

| method_id | Semántica | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `date_diff_seconds` | `\|Δt\|` en segundos | `float` | `null` | — |
| `date_diff_days` | `\|Δt\|` en días | `float` | `null` | — |
| `date_equal` | mismo día calendario UTC | `bool` | = resultado | — |
| `date_tolerance` | ¿dentro de ventana? | `dict` | `is_within_tolerance` | `tolerance_seconds` |

---

## Parseo a UTC

`parse_to_utc_datetime(value)` acepta:

| Tipo de entrada | Comportamiento |
|---|---|
| `datetime` con tz | Convierte a UTC |
| `datetime` naive | Asume UTC |
| `date` | Medianoche UTC de ese día |
| `int` / `float` | Epoch seconds; si `abs > 1e12`, trata como milisegundos |
| `str` ISO / `YYYY-MM-DD[ T]HH:MM:SS` | Parseo; sufijo `Z` → `+00:00`; naive → UTC |
| `str` solo dígitos | Se reinterpreta como epoch |
| `None` / vacío | `ValueError` |

Errores de parseo propagan excepción → `execute_transformation` devuelve `error` con el mensaje y `result_value: null`.

---

## `date_diff_seconds`

Diferencia **absoluta** en segundos entre A y B.

| | |
|---|---|
| **Options** | ninguna |
| **Retorno** | `float` (`abs((dt_a - dt_b).total_seconds())`) |
| **is_match** | `null` |

```python
execute_transformation(
    "2024-01-01T00:00:00Z",
    "2024-01-01T00:01:00Z",
    "date_diff_seconds",
)
# → result_value: 60.0
```

---

## `date_diff_days`

Equivalente a `date_diff_seconds / 86400.0`.

| | |
|---|---|
| **Options** | ninguna |
| **Retorno** | `float` (puede ser fraccionario) |
| **is_match** | `null` |

```python
execute_transformation("2024-01-01", "2024-01-03", "date_diff_days")
# → result_value: 2.0
```

---

## `date_equal`

Compara solo la **fecha calendario UTC** (`dt.date()`), ignorando la hora.

| | |
|---|---|
| **Options** | ninguna |
| **Retorno** | `bool` |
| **is_match** | igual al booleano |

```python
execute_transformation("2024-01-01 00:00", "2024-01-01 23:59", "date_equal")
# → result_value: true, is_match: true
```

Método *boolean-pure* en el orquestador.

---

## `date_tolerance`

¿Están A y B dentro de una ventana temporal?

### Options

| Clave | Default | Descripción |
|---|---|---|
| `tolerance_seconds` | `0` | Máximo `delta_seconds` permitido |

### Retorno (`result_value`)

| Campo | Tipo | Descripción |
|---|---|---|
| `is_within_tolerance` | `bool` | `delta_seconds <= tolerance_seconds` |
| `delta_seconds` | `float` | Diferencia absoluta en segundos |

`is_match` = `bool(is_within_tolerance)`.

```python
execute_transformation(
    "2024-01-01T00:00:00Z",
    "2024-01-01T00:30:00Z",
    "date_tolerance",
    {"tolerance_seconds": 3600},
)
# → result_value: { is_within_tolerance: true, delta_seconds: 1800.0 }
# → is_match: true
```
