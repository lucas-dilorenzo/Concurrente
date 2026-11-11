# Moldes — PMA (Parcial de Memoria Distribuida)

> **PMA sale en todos los parciales de 2024-2025 como ejercicio 1.** Los cuatro modelos salen de los parciales viejos (carpeta `Parciales y soluciones anteriores`, sin subir). Las soluciones de la cátedra tienen erratas: estas están corregidas.

## Sintaxis mínima

| | PMA | PMS |
|---|---|---|
| Declaración | **`chan nombre(tipos);`**, global, arriba de todo | No hay canales |
| Mandar | `send canal(datos);`: **no espera** | `Proceso!msg(datos)`: espera |
| Recibir | `receive canal(vars);`: espera si está vacío | `Proceso?msg(vars)` |
| Respuesta a uno solo | **Arreglo de canales**: `chan respuesta[P](...)` + `send respuesta[id](...)` | `Cliente[id]!...` |
| Ver si hay algo | **`empty(canal)`**: sólo en PMA | No existe |

> **La idea clave:** un canal compartido **ya es una cola FIFO**. Si varios empleados hacen `receive` del mismo canal, se reparten a los clientes solos: no hace falta Admin.

## Cómo reconocerlos

| El enunciado dice… | Modelo |
|---|---|
| *"alguno de los K empleados"*, *"atienden infinitamente"* | **M1**: canal compartido, sin Admin |
| *"se retiran cuando no hay más"*, *"todos deben terminar"* | **M2**: Admin que cuenta y manda `false` |
| *"si no hay nadie, hace otra cosa"* | **M3**: `empty()`; con varios empleados, por un Admin |
| *"tienen prioridad"* | **M4**: canal por nivel + `señal` |

---

## M1 — Varios empleados con canal compartido

> **Enunciado (parcial del 10-11-25, tema 1).** En un estadio de fútbol se dará un recital al que asistirán P personas. Cada asistente debe retirar su entrada por alguna de las **3 boleterías** presentando su DNI (una entrada por persona). Los asistentes son atendidos **por orden de llegada por alguno de los 3 empleados** (cada uno atiende una boletería). Implemente una solución usando PMA, considerando que los empleados **atienden infinitamente**. *Notas: maximizar la concurrencia; `obtenerEntrada(dni)` le retorna al empleado la entrada para ese DNI; `obtenerDNI()` retorna el DNI de quien la invoca.*

*Mismo modelo:* UP16 (parcial del 4-11-24: N clientes, 3 empleados, QR → paquete).

```
chan pedido(int, int);              // (id, dni): compartido = la fila
chan respuesta[P](Entrada);         // uno por persona

Process Persona[id: 0..P-1] {
  Entrada e;
  send pedido(id, obtenerDNI());
  receive respuesta[id](e);
}

Process Empleado[id: 0..2] {
  int idP, dni;
  while (true) {
    receive pedido(idP, dni);       // el primero de la fila, lo agarre quien lo agarre
    send respuesta[idP](obtenerEntrada(dni));
  }
}
```

- **El id va en el pedido** para que el empleado sepa a qué `respuesta[idP]` contestar.
- *"Atienden infinitamente"* ⇒ `while (true)`.

---

## M2 — Terminación: el Admin manda `false`

> **Enunciado (parcial del 10-11-25, tema 2).** El mismo estadio, pero **los empleados se retiran cuando ya no hay personas por atender**. *Todos los procesos deben terminar.*

Con `while (true)` el empleado queda bloqueado en el `receive` para siempre. **Un Admin cuenta las P personas y después le avisa a cada empleado que termine.**

```
chan pedido(int, int);
chan respuesta[P](Entrada);
chan pedirTrabajo(int);
chan trabajo[3](bool, int, int);    // (hay?, idP, dni)

Process Admin {
  int idP, dni, idE;
  for i = 1 to P {                         // atiende P personas
    receive pedido(idP, dni);
    receive pedirTrabajo(idE);
    send trabajo[idE](true, idP, dni);
  }
  for i = 1 to 3 {                         // después, un "fin" a cada empleado
    receive pedirTrabajo(idE);
    send trabajo[idE](false, -1, -1);
  }
}

Process Empleado[id: 0..2] {
  bool hay; int idP, dni;
  send pedirTrabajo(id);
  receive trabajo[id](hay, idP, dni);
  while (hay) {
    send respuesta[idP](obtenerEntrada(dni));
    send pedirTrabajo(id);
    receive trabajo[id](hay, idP, dni);
  }
}
```

La Persona es igual que en M1. **Es el `-1` de la tabla de terminación de PMS:** *"¿quién le manda el mensaje que lo hace salir?"*

---

## M3 — Tareas administrativas si no hay nadie: `empty()`

> **Enunciado, un empleado (parcial del 15-12-25).** El estadio con **una** boletería y **un** empleado, que **realiza tareas administrativas por 15 minutos cuando no hay asistentes para atender**. *Nota: `realizarTareasAdmin()` representa esas tareas.*

```
Process Empleado {
  int idP, dni;
  while (true) {
    if (not empty(pedido)) {
      receive pedido(idP, dni);
      send respuesta[idP](obtenerEntrada(dni));
    } else
      realizarTareasAdmin();
  }
}
```

> **Enunciado, varios empleados (parcial del 1-12-25).** Lo mismo con **5 boleterías y 5 empleados**.

Con varios, si dos ven `not empty` y hay una sola persona, **uno queda bloqueado en el `receive`**. El que pregunta `empty` tiene que ser **uno solo**: un Admin.

```
chan pedirTrabajo(int);
chan trabajo[5](int, int);

Process Admin {
  int idE, idP, dni;
  while (true) {
    receive pedirTrabajo(idE);
    if (empty(pedido))
      send trabajo[idE](-1, -1);           // no hay nadie
    else {
      receive pedido(idP, dni);
      send trabajo[idE](idP, dni);
    }
  }
}

Process Empleado[id: 0..4] {
  int idP, dni;
  while (true) {
    send pedirTrabajo(id);
    receive trabajo[id](idP, dni);
    if (idP == -1) realizarTareasAdmin();
    else send respuesta[idP](obtenerEntrada(dni));
  }
}
```

---

## M4 — Prioridad de 3 niveles: canal por nivel + `señal`

> **Enunciado (parcial del 25-11-24).** La UNLP organiza su maratón y P corredores deben retirar remera y chip en Rectorado, donde los atiende **un organizador**. Se atienden por orden de llegada, con **prioridad de los ancianos sobre jóvenes y adultos**, y de **los corredores con discapacidad sobre todos**. Entregan su DNI y reciben remera y chip. *Notas: `obtenerPrioridad()` retorna 0 discapacidad, 1 anciano, 2 otro; `obtenerRyC(dni)` retorna remera y chip; `obtenerDNI()` el DNI de quien la invoca.*

*Mismo modelo:* cobros digitales (parcial 3 de 2023 y Práctica de repaso MD, PMA 3). **Es el hermano en PMA del hospital en Ada.**

```
chan disc(int, int);  chan anciano(int, int);  chan general(int, int);
chan señal();
chan respuesta[P](Remera, Chip);

Process Corredor[id: 0..P-1] {
  int prio = obtenerPrioridad(), dni = obtenerDNI();
  Remera r; Chip c;
  if (prio == 0) send disc(id, dni);
  else if (prio == 1) send anciano(id, dni);
  else send general(id, dni);
  send señal();                       // DESPUÉS del pedido
  receive respuesta[id](r, c);
}

Process Organizador {
  int idC, dni;
  while (true) {
    receive señal();                  // duermo hasta que haya alguien
    if (not empty(disc)) receive disc(idC, dni);
    else if (not empty(anciano)) receive anciano(idC, dni);
    else receive general(idC, dni);
    send respuesta[idC](obtenerRyC(dni));
  }
}
```

- **La señal va después del pedido:** cuando el Organizador la recibe, el pedido ya está en algún canal.
- **Sin `señal`** habría que preguntar `empty` en un `while (true)`: busy waiting. La cátedra lo acepta, pero lo marca.
- El `if / else if` **ya es la prioridad**: mira primero el canal más importante.
