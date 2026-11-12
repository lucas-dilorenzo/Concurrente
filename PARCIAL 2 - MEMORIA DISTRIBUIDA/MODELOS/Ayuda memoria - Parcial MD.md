# Ayuda memoria — Parcial de Memoria Distribuida

> **¿Quién es la fila?** PMA: el canal · Ada: la cola del entry · PMS: no hay ⇒ **la arma el Admin**.
> **Leer:** qué mecanismo pide · subrayar *"orden"*, *"terminan"*, *"prioridad"*, *"si no hay nadie"*, *"a lo sumo"* · anotar el molde al margen.

## PMA — `chan`, `send` (no espera), `receive`, `empty`

```
chan pedido(int, Dato);   chan respuesta[N](Resp);

Process Cliente[id: 1..N] {
  send pedido(id, dato);
  receive respuesta[id](r);
}
Process Empleado[id: 1..K] {                 // M1: el canal reparte solo
  while (true) {
    receive pedido(idC, dato);
    send respuesta[idC](f(dato));
  }
}
```
| Si dice… | Agregar |
|---|---|
| *"terminan"* (varios) — **M2** | `chan pedirTrabajo(int); chan trabajo[K](bool,int,Dato);` · Admin: `for N { receive pedido; receive pedirTrabajo(idE); send trabajo[idE](true,..) }` + `for K { receive pedirTrabajo(idE); send trabajo[idE](false,..) }` · Empleado: `while (hay)` |
| *"si no hay nadie"* — **M3** | 1 empleado: `if (not empty(pedido)) {...} else otraCosa();` · varios: Admin con `if (empty(pedido)) send trabajo[idE](-1,..) else {...}` |
| *"prioridad"* — **M4** | Un canal por nivel + `chan señal();` · cliente: `send <nivel>(id,d); send señal();` · empleado: `receive señal(); if (not empty(alta)) receive alta(..) else if ... else receive baja(..);` |
| *"de a uno"* (recurso) | Admin: `receive pedido(id); send usar[id](); receive liberar();` |
| *"por id"* | `for i { send pasa[i](); receive termine(); }` |

## PMS — `Proc!msg` / `Proc?msg`, `do ... od`

```
Process Cliente[id: 1..N] {
  Admin!pedido(id);
  [S1] Empleado[*]?respuesta(r);
  [S2] Encargado?usar(); usar(); Encargado!liberar();
}
Process Admin {
  cola Fila; int idC, idE;
  do Cliente[*]?pedido(idC) -> push(Fila, idC);
  [] not empty(Fila); Empleado[*]?listo(idE) -> pop(Fila, idC); Empleado[idE]!trabajo(idC);   // S1
  [] not empty(Fila); Encargado?listo()      -> pop(Fila, idC); Encargado!siguiente(idC);     // S2
  od
}
[S1] Process Empleado[id: 1..K] { while (true) { Admin!listo(id); Admin?trabajo(idC); Cliente[idC]!respuesta(..); } }
[S2] Process Encargado { for i = 1 to N { Admin!listo(); Admin?siguiente(idC); Cliente[idC]!usar(); Cliente[idC]?liberar(); } }
```
- **`[*]` + id al recibir → `[id]` al mandar.** Uno solo: sin `[*]` ni índice.
- En la guarda **sólo `?`** · `!` y `?` siempre de a pares · cierra con **`od`**.
- **Por id (S3):** `for i = 1 to N { Cliente[i]?pedido(d); Cliente[i]!resp(..); }` — sin cola.
- **Prioridad (S4):** una cola por nivel; al `pop`, primero la más importante.
- **Terminan con varios:** contador `atendidos < N` en la guarda + `for K { Empleado[*]?listo(idE); Empleado[idE]!trabajo(-1); }`.

## Ada — `task`, `entry`, `accept`, `select`

```ada
Procedure X is
   task Empleado is
      entry Alta (d: IN T; r: OUT T);     -- nombre: MODO tipo; separados por ;
   end Empleado;
   task type Cliente;
   arrClientes: array (1..P) of Cliente;

   task body Empleado is ... begin ... end Empleado;
   task body Cliente is  ... begin ... end Cliente;
begin
   null;
end X;
```

**1. Prioridad** — *"tienen prioridad"*
```ada
for i in 1..P loop                                   -- loop + or terminate si alguno se va
   select
      accept Alta (d: IN T; r: OUT T) do r := f(d); end Alta;
   or
      when (Alta'count = 0) =>
         accept Media (d: IN T; r: OUT T) do r := f(d); end Media;
   or
      when (Alta'count = 0) and (Media'count = 0) =>
         accept Baja (d: IN T; r: OUT T) do r := f(d); end Baja;
   end select;
end loop;
-- Cliente:  prio := obtenerPrioridad;
--           if prio = 0 then Empleado.Alta(d, r); elsif prio = 1 then Empleado.Media(d, r); else Empleado.Baja(d, r); end if;
```

**2. Workers** — *"distribuido en N"*, *"calcula el total"*
```ada
-- Worker:  [loop]  Admin.Pedir(dato); res := calcular(dato); Admin.Resultado(res); [Admin.Siguiente;]  [end loop;]
-- Admin:
[loop]
   dato := generar; total := 0;
   for i in 1..2*N loop
      select
         accept Pedir (d: OUT T) do d := dato; end Pedir;
      or
         accept Resultado (r: IN T) do total := total + r; end Resultado;
      end select;
   end loop;
   [for i in 1..N loop accept Siguiente; end loop;]   -- sólo si se repite
[end loop;]
```

**3. Timeout** — *"a lo sumo X"*, *"si no lo atienden inmediatamente"*
```ada
select
   Empleado.Pago(p, c);           -- UNA sola llamada
or delay 600.0;                   -- o: else  ("inmediatamente")
   null;                          -- siempre una sentencia
end select;
-- En serie ("de a uno"): for i in 1..N loop select arrServ(i).Consulta(r(i)); or delay 5.0; r(i) := vacio; end select; end loop;
```

**Reglas de Ada:** el `OUT` se asigna **adentro del `do`** · la llamada lleva **variables**: `Empleado.Alta(d, r);` · mismo nombre en `entry`/`accept`/llamada · `:=` para asignar · `if ... then` / `elsif` / `end if;` · `for i in 1..N loop ... end loop;` · `when (X'count = 0) =>` con `and` · `task T;` si no tiene entries · tiempo con decimal · `--` comentarios.
