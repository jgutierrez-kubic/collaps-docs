# Métodos de texto

Seis operaciones para comparar nombres, códigos, descripciones y cadenas “parecidas pero no idénticas”.

---

## Resumen

| method_id | En una frase | `result_value` | `is_match` | Options |
|---|---|---|---|---|
| `strict_equal` | ¿Son exactamente iguales? | `bool` | = resultado | — |
| `normalized_equal` | ¿Son iguales ignorando mayúsculas/acentos? | `bool` | = resultado | — |
| `fuzzy_levenshtein` | ¿Qué tan parecidas son (edits)? | `float` 0–1 | ≥ umbral | `threshold` |
| `fuzzy_jaro_winkler` | ¿Qué tan parecidas son (prefijos)? | `float` 0–1 | ≥ umbral | `threshold` |
| `contains_check` | ¿Una contiene a la otra? | `bool` | = resultado | — |
| `regex_match` | ¿Cumple un patrón? | `bool` | = resultado | `pattern`, `ignore_case` |

---

## `strict_equal`

### Explicación en lenguaje humano

Compara **tal cual**. Mayúsculas, acentos, espacios y hasta el tipo de dato importan. Es el “candado” más estricto.

**Ejemplos reales**

| A | B | ¿Iguales? | Por qué |
|---|---|---|---|
| `Hola` | `Hola` | Sí | Idénticos |
| `Café` | `Cafe` | No | Falta la tilde |
| `1` (número) | `"1"` (texto) | No | Tipos distintos |

Úsalo para códigos que deben coincidir carácter a carácter (SKU, GUID, clave exacta).

```python
execute_transformation("Hola", "Hola", "strict_equal")
# → true

execute_transformation(1, "1", "strict_equal")
# → false
```

---

## `normalized_equal`

### Explicación en lenguaje humano

Primero “limpia” ambos textos y luego compara:

1. Quita espacios al inicio/final  
2. Pasa a minúsculas  
3. Elimina acentos (`Café` → `cafe`)

**Ejemplo real:** en un listado alguien escribió `Café` y en otro `cafe`. Con igualdad estricta fallan; con `normalized_equal` **coinciden**.

Ideal para nombres de materiales, tipologías o descripciones digitadas a mano.

```python
execute_transformation("Café", "cafe", "normalized_equal")
# → true, is_match: true
```

---

## `fuzzy_levenshtein`

### Explicación en lenguaje humano

Mide similitud contando **cuántos cambios** (insertar, borrar o sustituir una letra) hacen falta para transformar una palabra en la otra. Luego convierte esa distancia en un score de 0 a 1:

- `1.0` = idénticas  
- cerca de `0` = muy distintas  

Piensa en typos: `Juan` vs `Jaun`, o `hormigón` vs `hormigon` con un error tipográfico.

La fórmula de similitud es:

$$
1 - \frac{distance(a,b)}{\max(|a|, |b|, 1)}
$$

donde $$distance(a,b)$$ es la distancia de Levenshtein (número de edits).

**Ejemplo clásico:** `kitten` → `sitting` necesita varios cambios; la similitud queda alrededor de `0.57`. Con el umbral por defecto (`0.85`), **no** se considera match.

| Caso | Retorno |
|---|---|
| Ambos vacíos | `1.0` (se tratan como iguales) |
| Resto | score entre 0 y 1 |

### Options

| Clave | Default | Uso |
|---|---|---|
| `threshold` | `0.85` | A partir de este score, `is_match = true` |

```python
execute_transformation("kitten", "sitting", "fuzzy_levenshtein")
# → ≈ 0.57, is_match: false (threshold 0.85)
```

**Cuándo usarlo:** detectar errores de tipeo o variantes cortas. Si el prefijo importa más que el final (nombres propios), prueba también `fuzzy_jaro_winkler`.

---

## `fuzzy_jaro_winkler`

### Explicación en lenguaje humano

Otra medida de “parecido”, pensada sobre todo para **nombres y cadenas que comparten el comienzo**. Premia que las primeras letras coincidan (hasta 4 caracteres de prefijo).

**Ejemplo real:** `martha` y `marhta` (letras traspuestas) obtienen alta similitud (~`0.96`) → con umbral `0.85` es **match**.

Diferencia práctica frente a Levenshtein:

| Situación | Suele comportarse mejor… |
|---|---|
| Typos / letras cambiadas en el medio | Levenshtein o Jaro-Winkler (probar ambos) |
| Nombres con mismo inicio (`Rodriguez` / `Rodrigues`) | Jaro-Winkler |
| Palabras muy distintas en longitud | Revisar ambos scores; bajar `threshold` con cuidado |

```python
execute_transformation("martha", "marhta", "fuzzy_jaro_winkler")
# → ≈ 0.96, is_match: true
```

### Options

| Clave | Default | Uso |
|---|---|---|
| `threshold` | `0.85` | Umbral de match |

---

## `contains_check`

### Explicación en lenguaje humano

Pregunta: **¿una cadena está dentro de la otra?** Da igual el orden (A dentro de B o B dentro de A).

**Ejemplo real:** descripción A `world` y B `hello world` → verdadero. También al revés.

Útil para tags, códigos embebidos en una descripción larga, o “¿el nombre corto aparece en el nombre largo?”.

```python
execute_transformation("world", "hello world", "contains_check")
# → true
```

---

## `regex_match`

### Explicación en lenguaje humano

Valida si el texto A **cumple un patrón** (expresión regular). El patrón puede venir en `options.pattern` o, si no lo envías, se usa el valor B como patrón.

**Ejemplos reales**

- ¿El código son solo letras mayúsculas? patrón `^[A-Z]+$`
- ¿Empieza por `PRJ-`? patrón `^PRJ-`

Con `ignore_case: true`, `abc` también pasa el patrón de solo letras.

### Options

| Clave | Default | Descripción |
|---|---|---|
| `pattern` | `val_b` | Expresión regular |
| `ignore_case` | `false` | Ignorar mayúsculas |

```python
execute_transformation("ABC123", "x", "regex_match", {"pattern": "^[A-Z]+$"})
# → false  (hay dígitos)

execute_transformation("abc", "x", "regex_match", {"pattern": "^[A-Z]+$", "ignore_case": True})
# → true
```
