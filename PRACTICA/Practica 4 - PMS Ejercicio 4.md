# Práctica 4 (PMS) — Ejercicio 4

> **Enunciado.** En una exposición aeronáutica hay un **simulador de vuelo** (que debe ser usado con **exclusión mutua**) y **un empleado** encargado de administrar su uso. Hay **P personas** que esperan a que el empleado las deje acceder al simulador **de a una por vez**, la usan por un rato y luego se retiran.
> **a)** Implemente una solución donde el empleado sólo se ocupa de garantizar la exclusión mutua (**sin importar el orden**).
> **b)** Modifique la solución anterior para que el empleado los deje acceder **según el orden de su identificador** (hasta que la persona i no lo haya usado, la persona i+1 debe esperar).
> **c)** Modifique la solución a) para que el empleado considere **el orden de llegada** para dar acceso al simulador.
>
> *Nota: **cada persona usa sólo una vez** el simulador.*

> **Cómo se resolvió:** en ping-pong. El planteo del (a) salió bien, pero el código no tenía exclusión mutua; se explicó el molde S4 pensado secuencial. El (b) y el (c) los escribió Lucas, con detalles de sintaxis y la terminación del Admin.

---

## 0. El planteo

- **El simulador es un recurso**, no un proceso. Lo maneja **el empleado**.
- **La persona y el empleado se comunican dos veces:** pedir pasar, y avisar que terminó.
- Es el **molde S4** del repaso, y los tres ítems son sus tres variantes. **Lo único que cambia es a quién atiende el empleado.**

### El molde S4, pensado secuencial
El empleado es **el que tiene la llave del simulador**, y hace siempre lo mismo:
1. **espera** a que alguien le pida pasar;
2. **le da la llave**;
3. **espera** a que esa persona se la devuelva;
4. vuelve al paso 1.

**La exclusión mutua sale del paso 3:** mientras alguien está adentro, el empleado está parado esperando que termine, así que no puede dejar pasar a otro. **No hace falta ninguna guarda**: hace los pasos siempre en el mismo orden.

Los que piden mientras el simulador está ocupado **quedan bloqueados en su envío**. En PMS esperar es gratis.

---

## 1. Ítem (a): sin orden

```
Process Empleado {
  int i, idP;
  for (i = 1 to P) {
    Persona[*]?pedido(idP);        // alguien pide (cualquiera)
    Persona[idP]!pasa();           // le doy la llave
    Persona[idP]?termino();        // espero que ESA persona termine
  }
}

Process Persona[id: 1..P] {
  Empleado!pedido(id);
  Empleado?pasa();
  usarSimulador();
  Empleado!termino();
}
```

- **En el `termino` no hace falta `[*]` ni id:** el empleado ya sabe quién está adentro, es `idP`.
- **`for` de P vueltas:** cada persona lo usa una vez, así que el empleado atiende exactamente P veces y termina.

### Lo que hubo que corregir
| Error | Por qué |
|---|---|
| ⚠️ El `termino` **afuera** del loop: `do P > 0; Persona[*]?pedido(idP) -> Persona[idP]!pasa(); od` | El empleado daba paso a uno, después a otro, después a otro… **sin esperar nunca que el de adentro termine**: entraban todos juntos. **No había exclusión mutua.** Y como `P` nunca bajaba, el `do` no terminaba |
| `!pasa` sin paréntesis | `Persona[idP]!pasa()` |
| `Persona[id: 1..N]` | El enunciado dice **P** personas |

---

## 2. Ítem (b): por orden de id

```
Process Empleado {
  int i;
  for (i = 1 to P) {
    Persona[i]?pedido();           // en la vuelta i, atiendo a la persona i
    Persona[i]!pasa();
    Persona[i]?termino();
  }
}

Process Persona[id: 1..P] {
  Empleado!pedido();               // sin id: el empleado ya sabe quién es
  Empleado?pasa();
  usarSimulador();
  Empleado!termino();
}
```

- **El `for` ya es el orden.** Es **la posta** de semáforos, pero la lleva el empleado.
- **Envío y recepción tienen que coincidir en los parámetros:** si el empleado recibe `pedido()` sin nada, la persona lo manda sin nada.
- **Variante válida:** el `pedido` sobra, porque el empleado ya sabe a quién le toca. Puede hacer directamente `Persona[i]!pasa()`, que **bloquea hasta que la persona `i` esté esperando**, y la persona arranca con `Empleado?pasa()`.

**¿Hacía falta un Admin?** **No.** La pregunta que lo decide: ***¿el orden se conoce antes de empezar, o depende de lo que pase mientras corre?***

| Orden | ¿Se sabe antes? | Qué usás |
|---|---|---|
| Por id (b) | **Sí**: 1, 2, 3… | El `for` alcanza: no hay nada que anotar |
| Por llegada (c) | **No**: depende de quién llega primero | **Alguien tiene que anotarlo** en una cola apenas pasa: un **Admin** |

Es el mismo par que en semáforos: el **(b) es la posta**, el **(c) es Passing the Baton**.

**Detalles que hubo que corregir:** `int i,` con coma de más · los comentarios copiados del (a) (*"cualquiera: sin orden"*).

---

## 3. Ítem (c): por orden de llegada — S4 + S2

**Por qué no alcanza con lo anterior:** el `[*]` del (a) elige entre los que esperan **sin respetar quién llegó primero**, y el `for` del (b) sigue el orden de los ids.

**Por qué el Admin es un proceso aparte:** el empleado está **ocupado** mientras alguien usa el simulador (esperando su `termino`). Si las personas le hablaran directo, nadie las anotaría mientras tanto. Hace falta alguien **siempre libre para recibir**.

### Planteo
| Pregunta | Respuesta |
|---|---|
| Procesos | `Persona[1..P]`, `Admin`, `Empleado` |
| ¿Qué le manda la Persona al Admin? | **Su id**, para que el Admin la encole y **el Empleado sepa a quién darle el `pasa`** |
| ¿Cómo consigue el Empleado a la próxima? | **Se la pide al Admin**, como el Profesor del ej. 3 |
| ¿A quién va el `termino`? ¿Quién da el `pasa`? | El `termino` va **al Empleado**, y el `pasa` lo da **el Empleado**. Pedir la siguiente al Admin **ya funciona como aviso** de *"estoy libre"*: no hace falta otro mensaje |

**El id hace este camino:** Persona → Admin → `Fila` → Empleado → `Persona[idP]!pasa()`.

```
Process Admin {
  cola Fila; int idP; int contador = 0;
  do contador < P; Persona[*]?pedido(idP) -> push(Fila, idP);
                                             contador++;
  [] not empty(Fila); Empleado?proximo()  -> pop(Fila, idP);
                                             Empleado!siguiente(idP);
  od
}

Process Empleado {
  int i, idP;
  for (i = 1 to P) {
    Admin!proximo();               // ¿quién sigue?
    Admin?siguiente(idP);
    Persona[idP]!pasa();
    Persona[idP]?termino();
  }
}

Process Persona[id: 1..P] {
  Admin!pedido(id);                // me anoto
  Empleado?pasa();                 // espero que el empleado me deje pasar
  usarSimulador();
  Empleado!termino();
}
```

- **El Empleado es el del (a)** con un solo cambio: en lugar de recibir de `Persona[*]`, **le pide la siguiente al Admin**.
- **Por qué respeta el orden:** el Admin anota a cada persona apenas llega, y el Empleado recibe siempre la más vieja de la `Fila`. **El orden lo construye la cola.**

### Cómo termina el Admin
- Cuando llegaron las P personas, `contador == P` y la primera opción es falsa.
- Cuando le pasa la última al Empleado, la `Fila` queda vacía y la segunda también es falsa.
- **Todas falsas: el `do` termina.** No deja a nadie colgado: el Empleado pide exactamente P veces y el Admin despacha exactamente P.

### Lo que hubo que corregir
| Error | Por qué |
|---|---|
| `do true; [] Persona[*]?... [] ...` | El `do` **no lleva `true;`** y **la primera opción no lleva `[]`**: el `[]` sólo *separa* opciones |
| ⚠️ El Admin sin contador | Una opción sin condición **nunca falla**: el `do` era un `while(true)` y el Admin no terminaba. Cada persona pide **una vez** ⇒ se cuentan los pedidos y se corta en P (como el ej. 3 a) |
| `Adimin!proximo()`, falta `;` tras `Admin?siguiente(idP)` | Typos |

---

## 4. La regla de terminación

**¿Por qué acá hay contador y en el PMS 1 no?** Porque **acá sabemos el límite**: cada persona lo usa una vez, son P pedidos en total. En el PMS 1 los examinadores buscan *"continuamente"*: los pedidos son infinitos y nadie termina, y está bien.

| | **Un servidor** | **Varios servidores** |
|---|---|---|
| **Con límite** (*"una vez"*, *"10 trabajos"*) | Admin con contador · Servidor con `for` (PMS 3 a, PMS 4) | Admin con contador **y la opción del `-1`** · Servidor con `while (seguir)` (PMS 3 b) |
| **Sin límite** (*"continuamente"*) | Admin sin contador · Servidor con `while (true)` (PMS 1) | Igual: todos con `while (true)` |

- **Con varios servidores no sirve el `for`:** se sabe que en total son N trabajos, pero **no cuántos le tocan a cada uno**. El que cuenta (el Admin) avisa con un `-1` a cada servidor cuando pide, y no termina hasta avisarles a todos.
- **¿`while (true)` está mal si hay límite?** El programa funciona, pero el proceso **nunca termina**: queda bloqueado esperando un mensaje que no llega. Si el enunciado da la cantidad, lo seguro es el `for`. Si dice *"todos los procesos deben terminar"*, el `while (true)` sí está mal.

> **Chequeo final:** por cada proceso, *"¿quién le manda el mensaje que lo hace salir?"*. Si es *"nadie"*, queda colgado.

---

## 5. Chequeos de sanidad

- **Exclusión mutua:** el empleado no da otro `pasa` hasta recibir el `termino` del que está adentro. ✅
- **Cada `!` tiene su `?`:** `Admin!pedido` ↔ `Persona[*]?pedido` · `Admin!proximo` ↔ `Empleado?proximo` · `Empleado!siguiente` ↔ `Admin?siguiente` · `Persona[idP]!pasa` ↔ `Empleado?pasa` · `Empleado!termino` ↔ `Persona[idP]?termino`. ✅
- **Todos terminan:** cada persona después de usarlo; el empleado tras P vueltas; el Admin con todas las condiciones falsas. ✅
- **Con P = 1**, los tres ítems se comportan igual. ✅
