# Práctica 4 (PMA) — Ejercicio 3

> **Enunciado.** Se debe modelar el funcionamiento de una casa de comida rápida, en la cual trabajan **2 cocineros** y **3 vendedores**, y que debe atender a **C clientes**. El modelado debe considerar que:
> - Cada cliente realiza un pedido y luego **espera a que se lo entreguen**.
> - Los pedidos que hacen los clientes son tomados por **cualquiera de los vendedores** y se los pasan a los cocineros para que realicen el plato. **Cuando no hay pedidos para atender, los vendedores aprovechan para reponer un pack de bebidas** de la heladera (tardan entre 1 y 3 minutos para hacer esto).
> - Repetidamente cada cocinero toma un pedido pendiente dejado por los vendedores, lo cocina y **se lo entrega directamente al cliente** correspondiente.
>
> *Nota: maximizar la concurrencia.*

> **Cómo se resolvió:** en ping-pong. Lucas hizo el planteo frase por frase y escribió el código mirando los modelos del repaso. **Lo que más le costó: distinguir el rol del Vendedor del rol del Coordinador** (ver § 2).

---

## 0. Frase por frase: tres moldes que ya conocés

| Frase | Molde |
|---|---|
| *"Cualquiera de los 3 vendedores toma el pedido; si no hay, reponen bebidas"* | **A5**: varios servidores que hacen otra cosa → **Coordinador**. Es el ejercicio 1 (c) con otros nombres |
| *"Los vendedores se los pasan a los cocineros"* | **A1** (el vendedor deja la comanda y sigue) + **A3** (los 2 cocineros leen del mismo canal) |
| *"El cocinero se lo entrega directamente al cliente"* | **La respuesta privada del A2**: el cliente espera **su** plato en `Producto[idC]` |

---

## 1. El planteo

| Proceso | Qué hace |
|---|---|
| `Cliente[1..C]` | Manda su pedido y espera **su** plato |
| `Coordinador` | **Único lector** de `Pedidos`. Le responde a cada vendedor con un cliente o con `-1` |
| `Vendedor[1..3]` | Pide trabajo. Si hay cliente, le pasa la comanda a los cocineros **y sigue**. Si no hay, repone bebidas |
| `Cocinero[1..2]` | Toma una comanda, cocina y **le entrega el plato directo al cliente** |

```
chan Pedidos(int, texto);          // idC, pedido
chan PedidoVend(int);              // idV: el vendedor libre pide trabajo
chan Siguiente[1..3](int, texto);  // idC (o -1), pedido
chan Comanda(int, texto);          // idC, pedido: un solo canal para los 2 cocineros
chan Producto[1..C](texto);        // el plato, privado por cliente
```

**El camino de un pedido:**
1. **Cliente:** `send Pedidos(id, pedido)` y espera en `Producto[id]`.
2. **Vendedor:** cuando está libre, `send PedidoVend(idV)` y espera en `Siguiente[idV]`.
3. **Coordinador:** recibe el pedido de trabajo y mira `Pedidos`. Si hay un pedido, lo saca y se lo manda al vendedor; si no, le manda `-1`. **El pedido espera en el canal hasta que un vendedor pide trabajo**, no se lee cuando el cliente llega.
4. **Vendedor:** si recibió un cliente, `send Comanda(idC, pedido)` y **vuelve al paso 2 sin esperar el plato**. Si recibió `-1`, repone bebidas.
5. **Cocinero:** recibe la comanda, cocina y hace `send Producto[idC](plato)`.

---

## 2. ¿Para qué el Coordinador, si el Vendedor ya reparte?

Es la duda que apareció, y es razonable: el vendedor **sí** coordina algo, porque recibe el pedido y lo deriva. Pero el Coordinador no está para eso. **Está para que `Pedidos` tenga un solo lector.**

Sin Coordinador, con los vendedores leyendo directo:

```
Process Vendedor[id: 1..3] {
  while (true) {
    if (not empty(Pedidos)) { receive Pedidos(idC, pedido); send Comanda(idC, pedido); }
    else delay(random(1, 3) min);
  }
}
```

1. Hay **un solo** pedido en `Pedidos`.
2. Los vendedores 1 y 2 preguntan `empty(Pedidos)` casi al mismo tiempo. **Los dos ven que no está vacío.**
3. Los dos hacen `receive`. El 1 se lleva el pedido y **el 2 queda bloqueado** hasta que llegue otro cliente.
4. El vendedor 2 debería estar reponiendo bebidas y está dormido: **demora innecesaria**.

> **La regla para reconocerlo:** **varios** procesos leen del **mismo** canal **y** tienen algo que hacer si está vacío ⇒ Coordinador que sea el único lector.
>
> | Situación | Molde |
> |---|---|
> | **Un** vendedor que repone bebidas | `empty` alcanza (A4) |
> | **3** vendedores que **no** hacen otra cosa | Leen directo con `receive` y se bloquean (A3) |
> | **3** vendedores que reponen bebidas | **Coordinador** (A5) |

**Los cocineros no necesitan Coordinador**, aunque también son varios leyendo de `Comanda`: si no hay comandas, **no tienen otra cosa que hacer**, así que está bien que esperen bloqueados en el `receive`.

---

## 3. Solución

```
chan Pedidos(int, texto);          // idC, pedido
chan PedidoVend(int);              // idV
chan Siguiente[1..3](int, texto);  // idC (o -1), pedido
chan Comanda(int, texto);          // idC, pedido
chan Producto[1..C](texto);        // privado por cliente

Process Cliente[id: 1..C] {
  texto pedido, plato;
  pedido = elegirPedido();
  send Pedidos(id, pedido);
  receive Producto[id](plato);
}

Process Coordinador {
  texto pedido; int idV, idC;
  while (true) {
    receive PedidoVend(idV);
    if (empty(Pedidos)) { idC = -1; pedido = ""; }
    else receive Pedidos(idC, pedido);
    send Siguiente[idV](idC, pedido);     // SIEMPRE responde
  }
}

Process Vendedor[id: 1..3] {
  int idC; texto pedido;
  while (true) {
    send PedidoVend(id);
    receive Siguiente[id](idC, pedido);
    if (idC <> -1) send Comanda(idC, pedido);   // y sigue: no espera el plato
    else delay(random(1, 3) min);               // repone bebidas
  }
}

Process Cocinero[id: 1..2] {
  int idC; texto pedido, plato;
  while (true) {
    receive Comanda(idC, pedido);
    plato = cocinar(pedido);
    send Producto[idC](plato);            // directo al cliente
  }
}
```

**Por qué se maximiza la concurrencia:**
- El vendedor **no espera el plato**: deja la comanda y vuelve a pedir trabajo. Como `send` no bloquea, eso sale solo.
- Los 2 cocineros cocinan en paralelo y los 3 vendedores atienden en paralelo.
- El Coordinador sólo reparte; nunca espera una atención.

---

## 4. Lo que hubo que corregir

- **En el planteo:** faltaba el **Coordinador** como proceso (se había elegido el molde A5, que lo trae); `Producto(idC)` era un solo canal y tiene que ser **`Producto[1..C]`** (el cliente es el índice); la entrega del cocinero se había marcado como A4, cuando es **la respuesta privada del A2**.
- **En el código, sólo escritura:** `recive` → `receive` · `producto` sin declarar en el Cocinero y `;` faltantes · `Texto` en mayúscula · `delay(random(1,3))` sin unidad: sin `min` se lee como segundos · el cliente mandaba `pedido` sin asignarle valor.
- **Detalle fino:** cuando el Coordinador responde `-1`, el `pedido` que viaja tiene el valor anterior. No rompe nada (el vendedor no lo usa), pero queda más prolijo vaciarlo.

---

## 5. Chequeos de sanidad

- **Seguir el `idC` con el dedo:** `Cliente` → `Pedidos` → `Coordinador` → `Siguiente[idV]` → `Vendedor` → `Comanda` → `Cocinero` → `Producto[idC]` → `Cliente`. ✅
- **`Pedidos` tiene un solo receptor** (el Coordinador), así que su `empty` no tiene carrera. ✅
- **El Coordinador responde siempre**, si no el vendedor se dormiría en `Siguiente`. ✅
- **Con 1 vendedor** el Coordinador sobra: alcanza con `empty` en el vendedor (A4). ✅
