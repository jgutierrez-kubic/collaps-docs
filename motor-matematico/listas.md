# Métodos de listas / arreglos

Tres operaciones sobre colecciones. La coerción `_to_list` acepta:

| Entrada | Resultado |
|---|---|
| `list` / `tuple` | Lista |
| `None` | `[]` |
| `str` JSON array (`[...]`) | Lista parseada |
| `str` CSV (`"a,b,c"`) | Split por coma |
| Otro | `[value]` |

---

## Resumen

| method_id | Semántica | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `array_intersection` | A ∩ B (orden de A, sin dupes) | `list` | `null` | — |
| `array_difference` | A − B (orden de A, sin dupes) | `list` | `null` | — |
| `array_jaccard` | \|A∩B\| / \|A∪B\| | `float` 0–1 | `>= threshold` | `threshold` (default `0.85`) |

---

## `array_intersection`

Elementos de A que también están en B.

| Detalle | Comportamiento |
|---|---|
| Igualdad de ítems | Clave serializada: `json.dumps(..., sort_keys=True)` para dict/list; `str` para el resto |
| Orden | Preserva el de A |
| Duplicados | Elimina repeticiones en el resultado |

```python
execute_transformation([1, 2, 3], [2, 3, 4], "array_intersection")
# → result_value: [2, 3], is_match: null
```

---

## `array_difference`

Elementos en A que **no** están en B (`item not in list_b`), sin duplicar en el resultado.

| Nota | |
|---|---|
| Comparación | Usa `in` de Python sobre la lista B (no la serialización JSON de intersection) |
| Orden | Preserva el de A |

```python
execute_transformation([1, 2, 3], [2], "array_difference")
# → result_value: [1, 3]
```

---

## `array_jaccard`

Índice de Jaccard sobre conjuntos:

\[
J(A,B) = \frac{|A \cap B|}{|A \cup B|}
\]

| Caso | Retorno |
|---|---|
| Ambos vacíos | `1.0` |
| Unión vacía (defensivo) | `0.0` |
| Resto | `float` en \([0, 1]\) |

### Options

| Clave | Default | Uso |
|---|---|---|
| `threshold` | `0.85` | `is_match = result_value >= threshold` |

```python
execute_transformation([1, 2], [2, 3], "array_jaccard")
# → result_value: 0.333..., is_match: false (threshold 0.85)

execute_transformation([1, 2], [2, 3], "array_jaccard", {"threshold": 0.3})
# → is_match: true
```

> Los sets de Python requieren elementos hashables. Listas/dicts anidados como ítems pueden fallar en Jaccard (a diferencia de intersection, que serializa).
