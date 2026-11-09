# Práctica 5 (Ada) — Ejercicio 2

> **Enunciado.** Se quiere modelar el funcionamiento de un **banco**, al cual llegan clientes que deben realizar un pago y retirar un comprobante. Existe **un único empleado** en el banco, el cual atiende de acuerdo con **el orden de llegada**.
> **a)** Implemente una solución donde los clientes llegan y se retiran sólo después de haber sido atendidos.
> **b)** Implemente una solución donde los clientes se retiran si esperan **más de 10 minutos** para realizar el pago.
> **c)** Implemente una solución donde los clientes se retiran si **no son atendidos inmediatamente**.
> **d)** Implemente una solución donde los clientes esperan a lo sumo 10 minutos para ser atendidos. Si pasado ese lapso no fueron atendidos, entonces **solicitan atención una vez más** y se retiran si no son atendidos inmediatamente.

> **Cómo se resolvió:** el primer ejercicio de Ada. Antes se armó la sintaxis de a poco (§ 0). El (a), el (b) y el (c) los escribió Lucas, con detalles de sintaxis; en el (d) la idea estaba bien, pero se perdió con los `select` anidados.

---

## 0. La sintaxis, armada de a poco

### Un programa Ada es una caja con tres partes
```ada
Procedure Banco is
   -- PARTE 1: qué tareas hay y qué entries ofrecen (especificación)
   -- PARTE 2: qué hace cada tarea (task body)
begin
   -- PARTE 3: el programa principal (casi siempre null)
end Banco;
```

### `task` o `task type`: ¿cuántas hay?
| ¿Cuántas hay? | Usás | PMS equivalente |
|---|---|---|
| **Una sola** | `task Empleado` | `Process Empleado` |
| **Varias iguales** | `task type Cliente` + `arrClientes: array (1..N) of Cliente` | `Process Cliente[id: 1..N]` |

- **Dos iguales con nombre propio** también son `task type`: `EmpleadoA, EmpleadoB: Empleado;`.
- **Una tarea del arreglo no sabe su posición.** Si la necesita (porque alguien la tiene que llamar), se la pasa el programa principal con un `entry Ident`.
- **Varios empleados para cualquier cliente** → no se puede llamar a "cualquiera": hace falta un **Administrador** (molde R7). Recorrer los empleados con un `for` se traba en el primero ocupado.

### La llegada de los clientes no se programa
Las N tareas `Cliente` arrancan solas. **"Llegar al banco" es hacer la llamada** `Empleado.Pago(...)`: el cliente entra a **la cola del entry**, que es FIFO. **Esa cola es la fila del banco.**

### El `accept`: el empleado en la ventanilla
```ada
accept Pago (pago: IN texto; comprobante: OUT texto) do
   comprobante := atender(pago);
end Pago;
```
1. Espera hasta que haya alguien en la fila de `Pago`.
2. Toma al primero: lo que trajo llega en `pago` (`IN`).
3. Ejecuta el cuerpo **con el cliente parado adelante, esperando**.
4. En `end Pago`, el cliente se lleva el `comprobante` (`OUT`) y se va.

---

## 1. Ítem (a): R1

```ada
Procedure Banco is
   task Empleado is
      entry Pago (pago: IN texto; comprobante: OUT texto);
   end Empleado;

   task type Cliente;
   arrClientes: array (1..N) of Cliente;

   task body Empleado is
   begin
      loop
         accept Pago (pago: IN texto; comprobante: OUT texto) do
            comprobante := atender(pago);
         end Pago;
      end loop;
   end Empleado;

   task body Cliente is
      miPago, miComprobante: texto;
   begin
      Empleado.Pago(miPago, miComprobante);
   end Cliente;

begin
   null;
end Banco;
```

### Lo que hubo que corregir
| Error | Corrección |
|---|---|
| ⚠️ Declaró `entry Pago (...)` pero aceptó y llamó a `Atencion` | **El entry se llama igual en la declaración, en el `accept` y en la llamada.** Y en la declaración van los parámetros completos |
| `pedido; res: texto;` en el Empleado | **Sobran:** los parámetros del `accept` ya existen adentro del cuerpo. Sólo se declaran variables si un dato se usa **fuera** del `accept`. (Y la sintaxis es con coma) |
| Comentarios con `//` | En Ada son con `--` |

### El mismo banco en PMS (molde S2)
```
Process Cliente[id: 1..N] {
  Admin!pedido(id, miPago);
  Empleado?comprobante(miComprobante);
}
Process Admin {
  cola Fila; int idC; texto pago;
  do Cliente[*]?pedido(idC, pago)       -> push(Fila, (idC, pago));
  [] not empty(Fila); Empleado?listo() -> pop(Fila, (idC, pago));
                                          Empleado!trabajo(idC, pago);
  od
}
Process Empleado {
  while (true) {
    Admin!listo();
    Admin?trabajo(idC, pago);
    comprobante = atender(pago);
    Cliente[idC]!comprobante(comprobante);
  }
}
```

| PMS | Ada | ¿Qué pasó? |
|---|---|---|
| `Process Admin` con `cola Fila` | — | **Desaparece:** la cola del entry es FIFO |
| `Admin!pedido(id, ...)` + `Empleado?comprobante(...)` | `Empleado.Pago(miPago, miComprobante)` | **Dos mensajes → una llamada** que va y vuelve |
| `id` en el pedido | — | **Desaparece:** el comprobante vuelve por el `OUT` |
| `Admin!listo()` + `Admin?trabajo(...)` | `accept Pago (...) do` | El empleado espera directo al próximo de su fila |
| `Cliente[idC]!comprobante(...)` | `end Pago;` | Al terminar el `accept`, el cliente se lleva el `OUT` |

**Con orden o sin orden, en Ada es la misma solución:** la cola del entry siempre es FIFO.

---

## 2. Ítem (b): esperar a lo sumo 10 minutos — R4 con `or delay`

**Cambia sólo el Cliente.** El que se cansa de esperar es él, y el empleado no sabe cuánto hace que espera cada uno ni puede sacarlo de la cola.

```ada
task body Cliente is
   miPago, miComprobante: texto;
begin
   select
      Empleado.Pago(miPago, miComprobante);
   or delay 600.0;
      null;                                  -- se fue sin pagar
   end select;
end Cliente;
```

- El cliente **llama enseguida y entra a la cola**, igual que en el (a). El `select ... or delay` no crea una cola: **le pone un límite de tiempo a la espera** en la cola del entry.
- Si el empleado lo acepta antes de los 600 segundos, hay rendezvous normal. Si no, **la llamada se cancela sola** y ejecuta lo que sigue al `or delay`.
- **Los 10 minutos cuentan hasta que lo empiezan a atender.** Si lo acepta en el minuto 9, ya está en el rendezvous y no se va.
- **No hay busy waiting:** el cliente queda bloqueado, esperando que lo acepten o que pase el tiempo.

**Detalles que hubo que corregir:** `600` → **`600.0`** (el tiempo es `Duration`, con decimal) · **después del `or delay` tiene que haber una sentencia**, aunque sea `null;` · comentario con `--`.

---

## 3. Ítem (c): sólo si lo atienden ya — R4 con `else`

```ada
task body Cliente is
   miPago, miComprobante: texto;
begin
   select
      Empleado.Pago(miPago, miComprobante);
   else
      null;                                  -- el empleado no estaba libre: se va
   end select;
end Cliente;
```

**"Inmediatamente"** significa que el empleado esté **parado en el `accept Pago` en ese instante**. Si está atendiendo a otro, el cliente se va aunque no haya nadie más en la fila.

### Cuándo hay busy waiting
| | ¿Espera bloqueado? | ¿Busy waiting? |
|---|---|---|
| Llamada simple | Sí, sin límite | No |
| `select ... or delay` | Sí, con límite | **No** |
| `select ... else` una vez | No espera | No |
| `select ... else` **en un loop sin espera** | No: reintenta enseguida | **Sí** ⚠️ (como el `if` con `empty` de PMA) |

---

## 4. Ítem (d): 10 minutos y otro intento inmediato — dos `select` anidados

**Cada `select` del cliente es una caja con dos compartimentos:** lo que intento, y el plan B. El (d) es **el (c) metido en el plan B del (b)**.

```ada
task body Cliente is
   miPago, miComprobante: texto;
begin
   select
      Empleado.Pago(miPago, miComprobante);        -- 1er intento: espero hasta 10 min
   or delay 600.0;
      select
         Empleado.Pago(miPago, miComprobante);     -- 2do intento: sólo si me atiende ya
      else
         null;                                     -- tampoco: me voy
      end select;
   end select;
end Cliente;
```

### Lo que hubo que corregir
| Su código | Problema |
|---|---|
| `select delay 600.0;` | No existe. El tiempo va **después de la llamada**: `or delay 600.0;` |
| La segunda llamada sin su propio `select` | Un `select` del cliente lleva **una sola** llamada: el segundo intento necesita su caja |
| El segundo intento con `or delay` | *"Inmediatamente"* → **`else`** |
| Un solo `end select;` | **Dos `select`, dos `end select`.** La sangría ayuda a verlo |

---

## 5. Los dos tipos de `select`

| | `select` del **cliente** (el que llama) | `select` del **servidor** (el que atiende) |
|---|---|---|
| Qué tiene adentro | **Una sola** llamada | **Varios** `accept`, con o sin `when` |
| Para qué sirve | Ponerle **límite a mi espera** | **Elegir a quién atender** según el estado |
| Opciones del final | `or delay` **o** `else` | `or delay`, `else` u `or terminate` |

**¿El cliente podía usar el `select` con `when`?** **No:** no tiene entries (nadie lo llama), el `when` sólo va antes de un `accept`, y no se mezclan `accept` y llamadas.

> **La regla:** *¿lo decide el que espera?* (*"se va si…"*, *"espera a lo sumo…"*) → `select` del **cliente**. *¿Lo decide el que atiende?* (*"atiende si hay lugar"*, *"prioridad a…"*) → `select` del **servidor**.

---

## 6. Chequeos de sanidad

- **El entry se llama igual** en la declaración, el `accept` y la llamada, con los mismos parámetros. ✅
- **Orden de llegada:** la cola del entry `Pago` es FIFO. ✅
- **El Empleado es el mismo en los cuatro ítems:** todo lo del tiempo lo decide el cliente. ✅
- **Cada `select` tiene una sola llamada, una sola opción y su `end select`.** ✅
