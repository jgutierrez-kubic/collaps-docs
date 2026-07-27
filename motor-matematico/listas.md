# Métodos de listas / arreglos

Tres operaciones para comparar conjuntos: materiales, códigos, tags, IDs múltiples, etc.

Antes de operar, el motor convierte la entrada a lista:

| Entrada | Se interpreta como |
|---|---|
| Lista / tupla | Tal cual |
| `null` | Lista vacía |
| Texto JSON `[...]` | Lista parseada |
| Texto CSV `"a,b,c"` | Lista separada por comas |
| Un solo valor | Lista de un elemento |

---

## Resumen

| method_id | En una frase | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `array_intersection` | ¿Qué tienen en común? | `list` | `null` | — |
| `array_difference` | ¿Qué tiene A que B no? | `list` | `null` | — |
| `array_jaccard` | ¿Qué tan parecidos son los conjuntos? | `float` 0–1 | ≥ umbral | `threshold` |

---

## `array_intersection`

### Explicación en lenguaje humano

Devuelve los elementos que aparecen en **ambos** lados. Es el “terreno común”.

**Ejemplo real:** modelo tiene materiales `[acero, hormigón, vidrio]` y contrato `[hormigón, madera, vidrio]` → intersección `[hormigón, vidrio]`.

Conserva el orden de A y no repite elementos.

```python
execute_transformation([1, 2, 3], [2, 3, 4], "array_intersection")
# → [2, 3]
```

---

## `array_difference`

### Explicación en lenguaje humano

Lista lo que está en A y **no** está en B. Sirve para detectar faltantes o sobrantes unilaterales.

**Ejemplo real:** códigos en modelo `[A, B, C]` y en contrato `[B]` → diferencia `[A, C]` (están en el modelo pero no en el contrato).

```python
execute_transformation([1, 2, 3], [2], "array_difference")
# → [1, 3]
```

---

## `array_jaccard`

### Explicación en lenguaje humano

El **índice de Jaccard** resume en un solo número (0 a 1) qué tan parecidos son dos conjuntos:

- `1.0` = son el mismo conjunto  
- `0.0` = no comparten nada  
- Valores intermedios = solapan parcialmente  

Fórmula:

$$
J(A,B) = \frac{|A \cap B|}{|A \cup B|}
$$

**Ejemplo real:** A = `{1, 2}` y B = `{2, 3}`

- Intersección: `{2}` → 1 elemento  
- Unión: `{1, 2, 3}` → 3 elementos  
- Jaccard: $$1/3 \approx 0.333$$  

Con el umbral por defecto (`0.85`) **no** es match. Si bajas el umbral a `0.3`, sí lo sería.

| Caso especial | Retorno |
|---|---|
| Ambos vacíos | `1.0` (se consideran iguales) |
| Unión vacía | `0.0` |

### Options

| Clave | Default | Uso |
|---|---|---|
| `threshold` | `0.85` | Score mínimo para `is_match = true` |

```python
execute_transformation([1, 2], [2, 3], "array_jaccard")
# → ≈ 0.333, is_match: false

execute_transformation([1, 2], [2, 3], "array_jaccard", {"threshold": 0.3})
# → is_match: true
```

**Cuándo usarlo:** comparar listas de tags, especialidades o códigos donde importa el *solapamiento relativo*, no solo “¿hay alguno en común?”.
