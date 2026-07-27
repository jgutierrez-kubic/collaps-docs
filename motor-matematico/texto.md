# Métodos de texto

Seis operaciones de igualdad, similitud y patrón sobre cadenas.

---

## Resumen

| method_id | Semántica | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `strict_equal` | `a == b` (Python) | `bool` | = resultado | — |
| `normalized_equal` | igualdad tras normalizar | `bool` | = resultado | — |
| `fuzzy_levenshtein` | similitud 0–1 | `float` | `>= threshold` | `threshold` (default `0.85`) |
| `fuzzy_jaro_winkler` | similitud 0–1 | `float` | `>= threshold` | `threshold` (default `0.85`) |
| `contains_check` | substring en cualquiera dirección | `bool` | = resultado | — |
| `regex_match` | `re.search(pattern, a)` | `bool` | = resultado | `pattern`, `ignore_case` |

Los métodos con `result_value` booleano son *boolean-pure* en el orquestador (no se añade columna `is_match__` extra).

---

## `strict_equal`

Igualdad estricta de Python (`val_a == val_b`), sin coerción de tipos.

| | |
|---|---|
| **Options** | ninguna |
| **Retorno** | `bool` |
| **is_match** | igual al booleano |

```python
execute_transformation("Hola", "Hola", "strict_equal")
# → result_value: true, is_match: true

execute_transformation(1, "1", "strict_equal")
# → result_value: false  (tipos distintos)
```

---

## `normalized_equal`

Compara tras `_normalize_text`:

1. `str(value)` (`None` → `""`)
2. `strip()` + `lower()`
3. Normalización Unicode NFKD y eliminación de diacríticos

| | |
|---|---|
| **Options** | ninguna |
| **Retorno** | `bool` |

```python
execute_transformation("Café", "cafe", "normalized_equal")
# → result_value: true, is_match: true
```

---

## `fuzzy_levenshtein`

Similitud basada en distancia de Levenshtein:

\[
1 - \frac{distance(a,b)}{\max(|a|,|b|,1)}
\]

| Caso | Retorno |
|---|---|
| Ambos vacíos | `1.0` |
| Resto | `float` en \([0, 1]\) |

### Options

| Clave | Default | Uso |
|---|---|---|
| `threshold` | `0.85` | `is_match = result_value >= threshold` |

```python
execute_transformation("kitten", "sitting", "fuzzy_levenshtein")
# → result_value ≈ 0.57, is_match: false (con threshold 0.85)
```

---

## `fuzzy_jaro_winkler`

Similitud Jaro-Winkler (implementación interna, `prefix_scale=0.1`, prefijo máx. 4).

| | |
|---|---|
| **Retorno** | `float` 0.0–1.0 |
| **Options** | `threshold` (default `0.85`) para `is_match` |

```python
execute_transformation("martha", "marhta", "fuzzy_jaro_winkler")
# → result_value ≈ 0.96, is_match: true
```

---

## `contains_check`

Verdadero si `str(a)` está contenido en `str(b)` **o** viceversa.

| | |
|---|---|
| **Options** | ninguna |
| **Retorno** | `bool` |

```python
execute_transformation("world", "hello world", "contains_check")
# → result_value: true, is_match: true
```

---

## `regex_match`

Ejecuta `re.search(pattern, str(val_a), flags)`.

### Options

| Clave | Default | Descripción |
|---|---|---|
| `pattern` | `val_b` | Patrón regex; si no se pasa, se usa `val_b` como patrón |
| `ignore_case` | `false` | Si `true`, aplica `re.IGNORECASE` |

```python
execute_transformation("ABC123", "x", "regex_match", {"pattern": "^[A-Z]+$"})
# → result_value: false

execute_transformation("abc", "x", "regex_match", {"pattern": "^[A-Z]+$", "ignore_case": True})
# → result_value: true
```
