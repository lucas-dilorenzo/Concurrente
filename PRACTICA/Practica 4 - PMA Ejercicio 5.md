# Práctica 4 (PMA) — Ejercicio 5

> **Enunciado.** Resolver la administración de **3 impresoras** de una oficina. Las impresoras son usadas por **N administrativos**, los cuales están continuamente trabajando y **cada tanto envían documentos a imprimir**. Cada impresora, **cuando está libre, toma un documento y lo imprime**, de acuerdo con el **orden de llegada**.
> **a)** Implemente una solución para el problema descrito.
> **b)** Modifique la solución implementada para que considere la presencia de un **director de oficina** que también usa las impresoras, el cual **tiene prioridad** sobre los administrativos.
> **c)** Modifique la solución (a) considerando que cada administrativo imprime **10 trabajos** y que **todos los procesos deben terminar** su ejecución.
> **d)** Modifique la solución (b) considerando que tanto el director como cada administrativo imprimen **10 trabajos** y que **todos los procesos deben terminar** su ejecución.
> **e)** Si la solución al ítem d) implica realizar **busy waiting**, modifíquela para evitarlo.
>
> *Nota: **ni los administrativos ni el director deben esperar a que se imprima el documento**.*

> **Estado: EN CURSO.** Resueltos el (a) y el (b). **Faltan (c), (d) y (e)**, que tratan de terminación.
>
> **Cómo se resolvió hasta acá:** el (a) lo escribió Lucas. En el (b) propuso la prioridad directo en las impresoras; se analizaron los dos problemas de esa versión y se armó la solución con Coordinador.

---

## 0. El planteo

| Frase | Molde |
|---|---|
| *"Cada impresora, cuando está libre, **toma** un documento y **lo imprime**"* | Las impresoras **hacen** algo: son **procesos**, no recursos |
| *"Ni los administrativos ni el director deben esperar a que se imprima"* | **A1, dejar algo y seguir:** `send` y continúa. Sin canal de respuesta |
| *"3 impresoras … de acuerdo con el orden de llegada"* | **A3, varios servidores idénticos:** las 3 leen del **mismo** canal, que es FIFO |
| **(b)** *"el director … tiene prioridad"* | **A6, prioridad**, con un Coordinador por haber varios receptores |

### No es el molde A7

| | A7 (administrador de recurso) | Este ejercicio |
|---|---|---|
| Ejemplo | Las cabinas del locutorio | Las impresoras |
| Qué hace el cliente | **Pide** una unidad, **la usa él** y la devuelve | **Deja** el documento y **sigue trabajando** |
| La unidad | **Recurso**: no hace nada | **Proceso**: toma e imprime |
| ¿Respuesta? | Sí: qué unidad le tocó | **No**: nadie espera nada |

Es el ejercicio 1 de la explicación de PMA (personas que dejan reportes) con 3 empleados, que es el ejercicio 3 de esa explicación.

---

## 1. Ítem (a)

```
chan Pedido(texto);

Process Administrativo[id: 1..N] {
  texto doc;
  while (true) {
    doc = generarDoc();
    send Pedido(doc);              // y sigue: no espera la impresión
  }
}

Process Impresora[id: 1..3] {
  texto doc;
  while (true) {
    receive Pedido(doc);           // la libre toma el siguiente, FIFO
    imprimir(doc);
  }
}
```

- **El `send` no bloquea**, así que el administrativo no espera la impresión: la nota se cumple sola.
- **No hace falta el id del administrativo** en el mensaje: nadie le tiene que responder.
- **Una impresora libre toma el siguiente y una ocupada no lee.** El orden de llegada sale del canal.

**Lo que hubo que corregir:** `recieve(doc)` → `receive Pedido(doc);`. Además del tipeo, **al `receive` le faltaba el nombre del canal**: siempre se recibe de un canal. Y faltaban los `;`.

---

## 2. Ítem (b): el director con prioridad

### La primera idea: prioridad directo en las impresoras

```
Process Impresora[id: 1..3] {                     // ✗
  texto doc;
  while (true) {
    if (not empty(PedidoD)) ->
          receive PedidoD(doc); imprimir(doc);
    [] (not empty(PedidoA)) and (empty(PedidoD)) ->
          receive PedidoA(doc); imprimir(doc);
    fi
  }
}
```

La prioridad está bien puesta (el `and empty(PedidoD)`), pero tiene **dos problemas**:

**1. Busy waiting.** Con los dos canales vacíos, ninguna opción es verdadera, el `if` sale sin hacer nada y el `while` vuelve a preguntar. Las 3 impresoras quedan dando vueltas. En PMA está permitido, pero hay que evitarlo (el ítem (e) lo pide).

**2. Carrera entre impresoras.**
1. Hay **un** documento del director y **uno** de un administrativo.
2. Las impresoras 1 y 2 preguntan `not empty(PedidoD)` casi al mismo tiempo. **Las dos ven que hay algo.**
3. Las dos eligen la rama del director. La 1 se lo lleva y **la 2 queda bloqueada en `receive PedidoD`**.
4. **El documento del administrativo queda esperando** con una impresora libre que no lo toma: **demora innecesaria**.

> Es **lo mismo que los vendedores del ejercicio 3**: varios receptores usando `empty` sobre el mismo canal. **La solución también es la misma: un Coordinador que sea el único lector**, y las impresoras le piden trabajo.

### La solución con Coordinador

```
chan PedidoA(texto);               // documentos de los administrativos
chan PedidoD(texto);               // documentos del director
chan Aviso(int);                   // "hay un documento nuevo"
chan Listo(int);                   // idI: una impresora libre pide trabajo
chan Doc[1..3](texto);             // el documento para esa impresora

Process Administrativo[id: 1..N] {
  texto doc;
  while (true) {
    doc = generarDoc();
    send PedidoA(doc);
    send Aviso(1);                 // DESPUÉS del documento
  }
}

Process Director {
  texto doc;
  while (true) {
    doc = generarDoc();
    send PedidoD(doc);
    send Aviso(1);                 // DESPUÉS del documento
  }
}

Process Coordinador {
  texto doc; int idI, ok;
  while (true) {
    receive Listo(idI);            // 1. espero una impresora libre
    receive Aviso(ok);             // 2. espero que haya un documento
    if (not empty(PedidoD)) ->     // 3. primero el director
          receive PedidoD(doc);
    [] (not empty(PedidoA)) and (empty(PedidoD)) ->
          receive PedidoA(doc);
    fi
    send Doc[idI](doc);            // 4. a la impresora que lo pidió
  }
}

Process Impresora[id: 1..3] {
  texto doc;
  while (true) {
    send Listo(id);                // estoy libre
    receive Doc[id](doc);
    imprimir(doc);
  }
}
```

| Pieza | Por qué |
|---|---|
| **El Coordinador es el único que lee** `PedidoD` y `PedidoA` | El `empty` dice la verdad: nadie más puede sacar el documento entre la pregunta y el `receive`. Se acaba la carrera |
| **Primero `Listo`, después `Aviso`** | Elige el documento **recién cuando hay una impresora para imprimirlo**. Si eligiera antes, un documento del director que llega mientras espera la impresora quedaría detrás de uno de un administrativo ya elegido |
| **`Aviso` después de cada documento** | Cada vez que pasa el `receive Aviso`, **seguro** hay un documento. El `if` nunca sale vacío |
| **`and empty(PedidoD)`** | La prioridad: un documento de administrativo sólo si no hay del director |
| **`Doc[1..3]` privado por impresora** | El documento va a **la impresora que pidió**, no a cualquiera |

- **Ni el director ni los administrativos esperan** la impresión: los `send` no bloquean.
- **No hay busy waiting:** el Coordinador duerme en `Listo` y en `Aviso`; las impresoras, en `Doc[id]`.

**Comparado con la primera idea:** las ramas del `if` son las mismas. Cambia **quién** lo ejecuta: antes las 3 impresoras (carrera), ahora **un solo** Coordinador.

**Detalles de escritura de la primera versión:** `send pedido(doc)` con el canal declarado como `pedidoA` · `recieve` → `receive` · la guarda se escribe `(not empty(PedidoD)) ->`, sin `if is`.

---

## 3. Ítems (c), (d) y (e): pendientes

**Pregunta para arrancar el (c):** si cada administrativo imprime 10 documentos y después termina, **¿cómo se entera una impresora de que ya no va a llegar ningún documento más?**
