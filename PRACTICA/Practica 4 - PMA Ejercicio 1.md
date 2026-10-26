# Práctica 4 (PMA) — Ejercicio 1

> **Enunciado.** Suponga que **N clientes** llegan a la cola de un banco y que serán atendidos por sus empleados. Analice el problema y defina qué procesos, recursos y canales/comunicaciones serán necesarios/convenientes para resolverlo. Luego, resuelva considerando las siguientes situaciones:
> **a.** Existe **un único empleado**, el cual atiende por **orden de llegada**.
> **b.** Ídem a) pero considerando que hay **2 empleados** para atender. ¿Qué debe modificarse en la solución anterior?
> **c.** Ídem b) pero considerando que, **si no hay clientes para atender, los empleados realizan tareas administrativas durante 15 minutos**. ¿Se puede resolver sin usar procesos adicionales? ¿Qué consecuencias implicaría?

> **Cómo se resolvió:** en ping-pong. Lucas escribió el planteo y el código de cada ítem, y acá queda la versión corregida junto con lo que se le escapó.

---

## 0. Las consideraciones de PMA

| Regla | Qué implica |
|---|---|
| Los canales son **compartidos por todos** y son **colas FIFO** | El orden de llegada **sale gratis**: no hace falta una cola propia |
| `send` **no bloquea**; `receive` **sí** | El cliente manda y sigue; para esperar algo, tiene que hacer `receive` |
| `empty(canal)` **sí**, pero **no** cuántos mensajes hay | Ojo cuando el canal tiene **varios receptores** |
| `if`/`do` no determinístico | Si ninguna opción es verdadera, sigue de largo: **puede generar busy waiting** |
| Evitar busy waiting · el tiempo con `delay` | |

Y la regla de fondo: **no hay variables compartidas.** Lo que en semáforos era un arreglo compartido, acá es un mensaje.

---

## 1. El análisis: procesos, recursos y canales

**Procesos:** `Cliente[1..N]` y `Empleado` (uno en el (a), dos en el (b) y el (c)).
**Recursos compartidos:** ninguno. Los empleados no son recursos pasivos, son **procesos** que atienden.

**El molde es cliente/servidor (A2 del repaso)**, porque el cliente **espera** a que lo atiendan: no deja un pedido y se va.

La pregunta que define los canales es *¿cómo sabe el empleado a quién contestarle, y cómo recibe cada cliente SU respuesta?*

```
chan Pedidos(int);              // UN canal general; viaja el id del cliente
chan Respuestas[1..N](texto);   // UN canal PRIVADO por cliente
```

| Decisión | Por qué |
|---|---|
| **El id viaja en el pedido** | Sin él, el empleado no sabe a quién responder |
| **`Respuestas` es un arreglo** | Si fuera un solo canal, **cualquier cliente esperando en el `receive` se llevaría la respuesta de otro**: en PMA cualquiera puede recibir de cualquier canal |

> **La regla:** si **varios esperan una respuesta distinta cada uno**, va **un canal por cada uno**. Es el `sem atendido[id]` de semáforos, con el dato adentro.

---

## 2. Ítem (a): un empleado

```
chan Pedidos(int);
chan Respuestas[1..N](texto);

Process Cliente[id: 1..N] {
  texto comp;
  send Pedidos(id);                // me pongo en la cola
  receive Respuestas[id](comp);    // espero MI respuesta
}

Process Empleado {
  int idC; texto comp;
  while (true) {
    receive Pedidos(idC);          // el primero que llegó (FIFO)
    comp = atender(idC);
    send Respuestas[idC](comp);    // al cliente que atendí
  }
}
```

### Lo que hubo que corregir en la primera versión

| Error | Por qué está mal |
|---|---|
| `chan Respuestas[N];` sin tipo y `receive Respuestas[id];` sin variable | **Todo mensaje lleva un dato**, aunque sea un aviso. El canal se declara con tipo y el `receive` con variable |
| `Process Empleado[id: 1..M]` en el (a) | En el (a) hay **un** empleado. *(Sin querer, ya era la solución del (b).)* |
| `int idC; // Identificador del empleado` | Es el id del **cliente**. Un comentario equivocado confunde al corrector |
| `Respuestas[N]` con procesos `1..N` | Declarar `[1..N]` para que los índices coincidan |

---

## 3. Ítem (b): dos empleados

**No cambia nada**, salvo que hay dos procesos:

```
Process Empleado[id: 1..2] { ... el mismo código ... }
```

**Por qué alcanza:** los dos leen del **mismo** canal `Pedidos`, que es una cola FIFO. Cada `receive` saca un pedido distinto y el orden de llegada se respeta.

> **Un canal por empleado sólo** si el cliente tuviera que **elegir** a un empleado en particular. Acá lo atiende cualquiera.

---

## 4. Ítem (c): tareas administrativas cuando no hay clientes

### Sin procesos adicionales: se puede, pero con demora innecesaria

Lo natural es que cada empleado pregunte antes de recibir:

```
Process Empleado[id: 1..2] {
  while (true) {
    if (not empty(Pedidos)) { receive Pedidos(idC); ... }
    else delay(15 min);
  }
}
```

**El problema aparece porque `Pedidos` tiene dos receptores:**

1. Hay **un solo** pedido en el canal.
2. Los dos empleados hacen `empty(Pedidos)` casi al mismo tiempo y **los dos ven que no está vacío**.
3. Los dos hacen `receive`. Uno se lleva el pedido y **el otro queda bloqueado en el `receive`** hasta que llegue otro cliente.
4. Ese empleado debería estar haciendo tareas administrativas y está dormido: **demora innecesaria**.

> **Respuesta a la pregunta del enunciado:** sí, se puede resolver sin procesos adicionales, pero **genera demora innecesaria**. No es deadlock (el empleado se despierta cuando llega otro cliente), pero no cumple lo que pide el enunciado.

### La solución: un Coordinador que es el único lector

Si **un solo proceso** lee `Pedidos`, el `empty` deja de tener carrera. Los empleados le **piden trabajo** al Coordinador, y el Coordinador **siempre les responde**: con un cliente, o con `-1` si no hay nadie.

```
chan Pedidos(int);                 // el id del cliente
chan PedidoEmple(int);             // el empleado libre pide trabajo
chan Siguiente[1..2](int);         // id del cliente a atender, o -1
chan Respuestas[1..N](texto);      // privado por cliente

Process Cliente[id: 1..N] {
  texto comp;
  send Pedidos(id);
  receive Respuestas[id](comp);
}

Process Coordinador {
  int idE, idC;
  while (true) {
    receive PedidoEmple(idE);
    if (empty(Pedidos)) idC = -1;
    else receive Pedidos(idC);
    send Siguiente[idE](idC);      // SIEMPRE responde
  }
}

Process Empleado[id: 1..2] {
  int idC; texto comp;
  while (true) {
    send PedidoEmple(id);
    receive Siguiente[id](idC);
    if (idC <> -1) {
      comp = atender(idC);
      send Respuestas[idC](comp);  // al CLIENTE, no al empleado
    }
    else delay(15 min);            // tareas administrativas
  }
}
```

### Por qué cada pieza está donde está

| Pieza | Por qué |
|---|---|
| **El Coordinador siempre responde** | Si sólo respondiera cuando hay cliente, el empleado quedaría bloqueado en `receive Siguiente` y volvería el problema original |
| **`Siguiente` es privado por empleado** | Si fuera uno solo, un empleado podría llevarse la respuesta pensada para el otro |
| **El `-1` como "no hay nadie"** | Lo que viaja es un id (`int`); `-1` no es un id válido |
| **El id del cliente viaja por `Siguiente`** | Es la única forma de que el empleado sepa a quién responder |
| **El `send Respuestas` va adentro del `if`** | Si no hubo cliente, no hay a quién responder |

### Los errores que se cometieron en el camino

**1. Responder al id del empleado.**
`send RespuestasEmple[id](Rep)`, con el `id` del proceso `Empleado`, le responde al cliente 1 o al 2, sin importar a quién atendió.

**2. Que el empleado lea directo del canal de los clientes para conseguir el `idC`.**
```
receive Pedidos(idC, Rep);     // ✗ el empleado lee directo de los clientes
send PedidoEmple(id);
receive Siguiente[id](Rep);
```
Rompe las tres cosas que resolvía el Coordinador:
- **Si no hay clientes, el empleado queda bloqueado** en ese `receive` y nunca hace tareas administrativas.
- **Cada atención consume dos pedidos:** uno lo lee el empleado y otro se lo pasa el Coordinador.
- **El Coordinador deja de ser el único lector** del canal.

> **La regla del molde A5:** si hay Coordinador, **el empleado no recibe nunca directo de los clientes**. Todo le llega por el Coordinador, **incluido el id del cliente**.

**3. Detalles:** `Siguiente[3]` con 2 empleados · canales sin tipo · nombres que no coinciden entre declaración y uso (`Pedidos` / `PedidosCli`) · `delay(600)` arrastrado de la explicación, donde eran 10 minutos (15 minutos son 900).

---

## 5. Chequeos de sanidad

> **Seguí con el dedo cada id desde quien lo genera hasta quien lo usa para responder.** Si en algún canal del camino no viaja, se perdió.

- **El id del cliente:** `Cliente` → `Pedidos` → `Coordinador` → `Siguiente[idE]` → `Empleado` → `Respuestas[idC]` → `Cliente`. ✅
- **El id del empleado:** `Empleado` → `PedidoEmple` → `Coordinador` → `Siguiente[idE]`. ✅
- **Cada `receive` tiene quién le mande:** el `receive Siguiente[id]` siempre recibe, porque el Coordinador responde siempre; el `receive Respuestas[id]` recibe una vez, del empleado que lo atendió. ✅
- **Con 1 empleado, el (b) colapsa en el (a).** ✅
- **Un solo receptor por cada canal donde se usa `empty`:** `Pedidos` sólo lo lee el Coordinador. ✅
