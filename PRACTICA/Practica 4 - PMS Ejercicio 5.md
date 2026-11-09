# Práctica 4 (PMS) — Ejercicio 5

> **Enunciado.** En un estadio de fútbol hay una **máquina expendedora** de gaseosas que debe ser usada por **E Espectadores** de acuerdo con **el orden de llegada**. Cuando el espectador accede a la máquina en su turno, usa la máquina y luego se retira para dejar al siguiente.
> *Nota: cada espectador usa la máquina **sólo una vez**.*

> **Cómo se resolvió:** en ping-pong. Lucas encontró solo la diferencia con el 4(c) (no hay empleado) y escribió el `do` con las dos opciones y el traspaso al siguiente. Hubo que corregir el `libre`, la terminación y la sintaxis.

---

## 0. El planteo

| Pregunta | Respuesta |
|---|---|
| Procesos | `Espectador[1..E]` y `Admin` |
| Diferencia con el 4(c) | **No hay Empleado.** El Admin **anota** a los que llegan **y** da el acceso a la máquina |
| Terminación | Cada espectador, después de usarla. El Admin, cuando todos la usaron (con contadores) |

### Por qué el Admin no puede ser un loop de E vueltas
```
for (i = 1 to E) {
  Espectador[*]?pedido(id);
  Espectador[id]!pasa();
  Espectador[id]?termino();
}
```
Mientras alguien usa la máquina, el Admin está parado en el `?termino` **y no anota a nadie**. Después toma con `[*]` a cualquiera: **se pierde el orden de llegada**. Es el 4(a), no el 4(c).

En el 4(c) había **dos procesos**: uno que anotaba siempre (Admin) y otro que esperaba el `termino` (Empleado). Con uno solo, **tiene que poder recibir las dos cosas en cualquier momento**: un `do` con dos opciones.

### Pensado secuencial: es Passing the Baton
El Admin recuerda si la máquina está **libre**:

| Momento | Hace |
|---|---|
| Llega un pedido y `libre` | `libre = false` y le da el pase |
| Llega un pedido y está ocupada | Lo encola |
| Llega un `termino` y hay fila | Le pasa la máquina al siguiente. **`libre` sigue en `false`** |
| Llega un `termino` y no hay fila | `libre = true` |

> **Regla 4: el recurso se entrega, no se libera.** Si al terminar se pusiera `libre = true` y después se despertara al siguiente, un recién llegado podría colarse.

---

## 1. Solución

```
Process Admin {
  cola Fila; int idE; bool libre = true;
  int llegaron = 0, terminaron = 0;
  do llegaron < E; Espectador[*]?pedido(idE) ->
        llegaron++;
        if (libre) { libre = false; Espectador[idE]!pasa(); }
        else push(Fila, idE);
  [] terminaron < E; Espectador[*]?termino() ->
        terminaron++;
        if (empty(Fila)) libre = true;
        else { pop(Fila, idE); Espectador[idE]!pasa(); }   // se entrega, no se libera
  od
}

Process Espectador[id: 1..E] {
  Admin!pedido(id);
  Admin?pasa();
  usarMaquina();
  Admin!termino();
}
```

- **El `Espectador[idE]!pasa()` no bloquea al Admin:** el espectador ya está parado en `Admin?pasa()`, esperando.
- **En el `termino` no hace falta id:** el Admin no necesita saber quién terminó, sólo que la máquina se desocupó.
- **Cómo termina:** después de E pedidos la primera opción es falsa, y después de E `termino` también la segunda. **Todas falsas: el `do` termina.**

---

## 2. Lo que hubo que corregir

| Error | Por qué |
|---|---|
| ⚠️ `if (empty(Fila))` para saber si la máquina está libre | *"Nadie esperando"* **no es** *"nadie usando"*. Llega el 1 y pasa; mientras la usa llega el 2, la fila sigue vacía, y **pasa también**: dos en la máquina. Hace falta **`bool libre`** |
| Faltaba el caso *"termina y no hay fila"* | Sin `libre = true`, el próximo que llega queda encolado para siempre |
| ⚠️ Sólo contaba los pedidos | La opción del `termino` no tenía condición, así que **nunca fallaba**: el `do` no terminaba. Se cuentan también los `termino` |
| `do contador < E; [] Persona[*]?...` | La condición va **pegada a su recepción**, y la primera opción no lleva `[]` |
| `Persona`, `P` | El enunciado dice **Espectador** y **E** |

---

## 3. La comparación que vale para el parcial

| | 4(c): Admin + Empleado | 5: un solo Admin |
|---|---|---|
| Quién anota | Admin | Admin |
| Quién da el paso y espera el `termino` | Empleado (secuencial, con `for`) | **El mismo Admin** (por eso hace falta `libre`) |
| Molde de MC equivalente | Coordinador | **Passing the Baton** |

---

## 4. Chequeos de sanidad

- **Exclusión mutua:** sólo se da un pase si `libre`, o al recibir un `termino` (el que sale le deja el lugar al siguiente). ✅
- **Orden de llegada:** el Admin anota apenas llega, y siempre entrega al primero de la fila. ✅
- **Todos terminan:** cada espectador después de usarla; el Admin con los dos contadores en E. ✅
- **Cada `!` tiene su `?`:** `Admin!pedido` ↔ `Espectador[*]?pedido` · `Espectador[idE]!pasa` ↔ `Admin?pasa` · `Admin!termino` ↔ `Espectador[*]?termino`. ✅
