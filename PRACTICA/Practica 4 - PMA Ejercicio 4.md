# Práctica 4 (PMA) — Ejercicio 4

> **Enunciado.** Simular la atención en un locutorio con **10 cabinas telefónicas**, el cual tiene **un empleado** que se encarga de atender a **N clientes**. Al llegar, cada cliente **espera hasta que el empleado le indique a qué cabina ir**, la usa y luego **se dirige al empleado para pagarle**. El empleado atiende a los clientes **en el orden en que hacen los pedidos**. A cada cliente se le entrega un **ticket factura** por la operación.
> **a)** Implemente una solución para el problema descrito.
> **b)** Modifique la solución implementada para que el empleado **dé prioridad a los que terminaron de usar la cabina** sobre los que están esperando para usarla.
>
> *Nota: maximizar la concurrencia; suponga que hay una función `Cobrar()` llamada por el empleado que simula que el empleado le cobra al cliente.*

> **Cómo se resolvió:** en ping-pong. Lucas hizo el planteo; el (a) **se le complicó** ("me mató este") y se reconstruyó pensándolo secuencial. **El (b) lo escribió él** y salió bien.

---

## 0. Frase por frase

| Frase | Molde |
|---|---|
| *"10 cabinas"*, *"le indique a qué cabina ir"* | **A7**: administrador de recurso. Las cabinas **se usan**, no hacen nada: son recursos, y el empleado lleva cuáles están libres |
| *"espera hasta que le indique"*, *"se le entrega un ticket"* | **A2**: respuestas privadas por cliente |
| **(a)** *"en el orden en que hacen los pedidos"* | **Un solo canal** con tipo de pedido |
| **(b)** *"prioridad a los que terminaron"* | **A6**: dos canales + `and empty(...)` + `Aviso` |

---

## 1. La decisión que define el ejercicio: ¿uno o dos canales?

El cliente le habla al empleado **dos veces**: para pedir cabina y para pagar. ¿Por uno o por dos canales?

**Con dos canales se pierde el orden.** Llega el pedido de cabina del cliente 3 y **después** el pago del cliente 7. El empleado tiene que elegir entre los dos canales con un `if` no determinístico, que **elige al azar**: puede cobrarle al 7 primero.

> **Cada canal es FIFO por separado, pero entre canales no hay orden.**

Por eso:
- el **(a)**, que pide *"en el orden en que hacen los pedidos"*, va con **un solo canal** y un campo de tipo;
- el **(b)**, que pide *prioridad a los que terminaron*, **rompe ese orden a propósito**: ahí van **dos canales**.

**El molde A7 tiene las dos versiones** (Teoría 5, filminas 24 y 26). Lo que lo define no es la cantidad de canales, sino el **contador de libres + el conjunto de unidades + la cola de pendientes**.

---

## 2. ¿Y si están las 10 ocupadas?

El empleado **no puede quedarse esperando** una cabina libre: no le cobraría a nadie, nadie liberaría una cabina, y sería **deadlock**.

> **`Aviso` no resuelve esto.** `Aviso` sirve para que el empleado duerma cuando **no hay ningún mensaje**. Acá **sí hay** un mensaje (un pedido de cabina) que **todavía no puede atender**.

La solución: el empleado **guarda el pedido en una cola local** (`esperando`) y sigue. Cuando alguien paga:
- si hay alguien esperando, **le da esa cabina directamente**;
- si no, la cabina vuelve a quedar libre.

| Monitores | PMA |
|---|---|
| `wait(c)` | `push(esperando, idC)`: lo anoto y **no respondo** todavía |
| `signal(c)` | `pop(esperando, sig)` + `send Respuesta[sig](...)` |
| *"El recurso no se libera, se entrega"* | Si alguien espera, **`libres` no sube** |

---

## 3. Ítem (a): orden de llegada

**Pensado secuencial**, como atendería una persona de a un pedido por vez:
- **Si piden cabina:** si hay alguna libre, saco una y se la digo. Si no hay, **anoto al cliente** y sigo.
- **Si vienen a pagar:** cobro y le doy el ticket. La cabina que dejó: si hay alguien esperando, **se la doy directamente**; si no, queda libre.

```
chan Pedidos(texto, int, int);     // tipo, idC, idCabina
chan Respuesta[1..N](int);         // la cabina asignada, privada por cliente
chan Ticket[1..N](texto);          // el ticket, privado por cliente

Process Cliente[id: 1..N] {
  int miCabina; texto t;
  send Pedidos("cabina", id, -1);           // idCabina no se usa
  receive Respuesta[id](miCabina);
  usarCabina(miCabina);
  send Pedidos("pago", id, miCabina);       // la cabina para liberarla, el id para el ticket
  receive Ticket[id](t);
}

Process Empleado {
  int libres = 10;
  set cabinas = {1..10};
  cola esperando;                  // clientes que pidieron y no había cabina
  texto tipo, t; int idC, idCab, sig;

  while (true) {
    receive Pedidos(tipo, idC, idCab);       // el más viejo: respeta el orden
    if (tipo == "cabina") {
      if (libres > 0) {
        libres--;
        remove(cabinas, idCab);
        send Respuesta[idC](idCab);
      }
      else push(esperando, idC);             // no hay: lo anoto y sigo
    }
    else {                                   // "pago"
      t = Cobrar(idC);
      send Ticket[idC](t);
      if (not empty(esperando)) {
        pop(esperando, sig);
        send Respuesta[sig](idCab);          // la cabina SE ENTREGA
      }
      else {
        libres++;
        insert(cabinas, idCab);
      }
    }
  }
}
```

**No hace falta `if` no determinístico ni `Aviso`:** con un solo canal, el empleado hace `receive` y se bloquea si no hay nada. Eso mismo es lo que respeta el orden.

**Se maximiza la concurrencia:** el empleado nunca espera a nadie (si no hay cabina, anota y sigue cobrando), y el cliente usa la cabina **afuera**.

### Lo que se mezcló en el primer intento

El primer intento combinó las dos versiones del molde: un solo canal **y** un `if` con guardas, con colas de espera para los dos tipos de pedido.

| Error | Por qué está mal |
|---|---|
| `if` no determinístico sobre un único canal | Con un solo canal alcanza con `receive`. El `if` con guardas es para elegir entre **varios** canales |
| **Cola de espera para los pagos** | Un pago **siempre** se puede atender: no necesita cabina libre. Sólo espera el que **pide** cabina |
| **`receive` después del `pop`** | El pedido ya se había recibido al guardarlo en la cola. Un segundo `receive` saca **otro** pedido del canal y lo pierde |
| Canal `ticket` usado por el cliente para avisar que terminó | El ticket va **del empleado al cliente**, privado: `Ticket[idC]`. El cliente avisa que terminó **con el pedido de pago** |
| `idU = disponible` | `disponible` dice **cuántas** quedan, no **cuál**. La cabina concreta la da `remove(cabinas, idU)` |

---

## 4. Ítem (b): prioridad a los que terminaron

**Las ramas no cambian.** Pedir cabina y pagar se atienden igual que en el (a). **Cambia sólo cómo elige el empleado cuál atender:**

| | (a) orden de llegada | (b) prioridad a los pagos |
|---|---|---|
| Canales | **Uno**, con tipo | **Dos**: `PedidoCabina` y `Pago` |
| Cómo elige | `receive` directo: el más viejo | `receive Aviso` + `if` con `and empty(Pago)` |

```
chan PedidoCabina(int);            // idC
chan Pago(int, int);               // idC, idCabina
chan Respuesta[1..N](int);
chan Ticket[1..N](texto);
chan Aviso(int);                   // "tenés un mensaje nuevo"

Process Cliente[id: 1..N] {
  int miCabina; texto t;
  send PedidoCabina(id);
  send Aviso(1);                   // DESPUÉS del mensaje
  receive Respuesta[id](miCabina);
  usarCabina(miCabina);
  send Pago(id, miCabina);
  send Aviso(1);                   // DESPUÉS del mensaje
  receive Ticket[id](t);
}

Process Empleado {
  int libres = 10;
  set cabinas = {1..10};
  cola esperando;
  texto t; int ok, idC, idCab, sig;

  while (true) {
    receive Aviso(ok);                       // duermo hasta que haya algo
    if (not empty(Pago)) ->
          receive Pago(idC, idCab);
          t = Cobrar(idC);
          send Ticket[idC](t);
          if (not empty(esperando)) {
            pop(esperando, sig);
            send Respuesta[sig](idCab);
          }
          else {
            libres++;
            insert(cabinas, idCab);
          }
    [] (not empty(PedidoCabina)) and (empty(Pago)) ->
          receive PedidoCabina(idC);
          if (libres > 0) {
            libres--;
            remove(cabinas, idCab);
            send Respuesta[idC](idCab);
          }
          else push(esperando, idC);
    fi
  }
}
```

**Por qué funciona la prioridad:** si hay un pago y un pedido de cabina, la segunda opción es falsa por el `and empty(Pago)`, así que el `if` sólo puede elegir el pago. Y como cada mensaje tiene su aviso, cada vez que el empleado despierta tiene algo que atender.

**Las dos piezas de la prioridad, del ejercicio 2:**
1. **`and empty(...)` en la opción menos prioritaria.** Sin eso, el `if` elige al azar.
2. **`Aviso`** para dormir en vez de dar vueltas, mandado por **todos** y **después** del mensaje. Acá el cliente avisa **dos veces**.

**Detalles que hubo que corregir en el (b):** canales sin tipo · `notEmpty(...)` en lugar de `not empty(...)` · faltaban el `fi` y las llaves de cierre · `ok` sin declarar.

---

## 5. Chequeos de sanidad

- **Cada cabina está en exactamente un lugar:** en `cabinas` (libre), en manos de un cliente, o viajando en `Respuesta`. Cuando se entrega a un pendiente, **no pasa por `cabinas`**. ✅
- **`libres` = tamaño de `cabinas`** en todo momento. ✅
- **Con 1 cabina** colapsa a exclusión mutua con cola de espera. ✅
- **En el (b), cada mensaje a `PedidoCabina` o `Pago` tiene su `Aviso`.** ✅
