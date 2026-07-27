# Alias legacy — `DIFERENCIA` e `IGUALDAD`

Nombres antiguos que todavía aceptan Directus/n8n por compatibilidad. Por detrás se traducen a métodos canónicos del motor.

---

## Explicación en lenguaje humano

Durante mucho tiempo los workflows hablaban de “hacer una DIFERENCIA” o “verificar IGUALDAD”. El motor moderno usa IDs técnicos (`math_sub`, `strict_equal`), pero **sigue entendiendo** esos dos nombres clásicos para no romper configuraciones viejas.

No necesitas programar: si en **Métodos Calculo** escribes `DIFERENCIA` o `IGUALDAD`, el sistema sabe qué hacer. Para análisis nuevos, preferimos los `method_id` modernos (son más claros y hay más opciones).

---

## Dónde se aceptan

| Capa | Comportamiento |
|---|---|
| Validación del payload | Permite `DIFERENCIA` e `IGUALDAD` además de los 21 métodos |
| Orquestador | Los traduce al método real (y, en DIFERENCIA, invierte el orden) |
| Selector n8n | Siguen apareciendo en la lista |
| `OPERATIONS_REGISTRY` | **No** los incluye como claves propias |

---

## Tabla de resolución

```python
_LEGACY_METHOD_MAP = {
    "DIFERENCIA": ("math_sub", True),   # invierte A y B
    "IGUALDAD":   ("strict_equal", False),
}
```

| Alias | Se convierte en | ¿Invierte A y B? | Resultado práctico |
|---|---|---|---|
| `DIFERENCIA` | `math_sub` | **Sí** | Calcula **B − A** |
| `IGUALDAD` | `strict_equal` | No | ¿A es exactamente igual a B? |

---

## `DIFERENCIA`

### Explicación en lenguaje humano

Históricamente COLLAPS reportaba la diferencia como **referencia menos modelo** (B menos A): “¿cuánto tiene el contrato respecto al modelo?”. El método moderno `math_sub` hace **A menos B**. Por eso, al usar el alias, el sistema **cambia el orden** internamente para conservar el signo de siempre.

**Ejemplo real**

| Cantidad modelo (A) | Cantidad contrato (B) | Con `DIFERENCIA` | Con `math_sub` |
|---|---|---|---|
| 10 | 3 | $$3 - 10 = -7$$ | $$10 - 3 = 7$$ |

Si migras un análisis de `DIFERENCIA` a `math_sub` sin cambiar el orden de columnas, **el signo se invierte**.

La columna de resultado en la tabla final se nombra con el método canónico (`...__math_sub`), no con la palabra `DIFERENCIA`.

---

## `IGUALDAD`

### Explicación en lenguaje humano

Es simplemente “¿son iguales tal cual?”, igual que `strict_equal`. No limpia acentos ni mayúsculas.

**Ejemplo real:** código `ABC` vs `ABC` → sí. `Café` vs `Cafe` → no (para eso usa `normalized_equal`).

---

## Uso en el payload / Directus

```text
Metodos Calculo: DIFERENCIA,IGUALDAD
```

Equivalente moderno (ojo al signo de la resta):

```text
Metodos Calculo: math_sub,strict_equal
```

---

## Recomendación

| Escenario | Preferir |
|---|---|
| Análisis nuevos | `method_id` canónicos (`math_sub`, `strict_equal`, …) |
| Flujos legacy en producción | Mantener alias hasta validar el signo del resultado |
| Documentar para el equipo | Nombrar el método canónico y avisar si hay inversión B−A |
