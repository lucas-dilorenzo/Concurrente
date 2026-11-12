# Hoja de sintaxis — PMA · PMS · Ada

> **Para mirar la noche antes y antes de entrar.** El riesgo más grande es **mezclar la sintaxis de los tres**.

## 1. Lo básico, lado a lado

| | PMA | PMS | Ada |
|---|---|---|---|
| Proceso único | `Process Admin { ... }` | `Process Admin { ... }` | `task Admin is entry ...; end Admin;` + `task body Admin is ... begin ... end Admin;` |
| Varios iguales | `Process Cliente[id: 1..N]` | `Process Cliente[id: 1..N]` | `task type Cliente;` + `arrClientes: array (1..N) of Cliente;` |
| ¿Sabe su id? | Sí, `id` | Sí, `id` | **No.** Sólo si hace falta, con `entry Ident(id: IN integer)` |
| Variables | **`int x; texto doc;`** (tipo primero) | **`int x; texto doc;`** | **`x: integer; doc: texto;`** (nombre primero) |
| Asignar | `x = 5;` | `x = 5;` | **`x := 5;`** (`=` sólo compara) |
| Comentario | `//` | `//` | **`--`** |
| Programa | — | — | `Procedure X is ... begin null; end X;` |

## 2. Comunicarse

| | PMA | PMS | Ada |
|---|---|---|---|
| Declarar | `chan pedido(int, texto);` global | Nada | `entry Pedido (d: IN texto; r: OUT texto);` en la especificación |
| Mandar | `send pedido(id, d);` (no espera) | `Admin!pedido(id, d);` (espera) | `Admin.Pedido(d, r);` (va **y vuelve**) |
| Recibir | `receive pedido(idC, d);` | `Cliente[*]?pedido(idC, d);` | `accept Pedido (d: IN texto; r: OUT texto) do r := f(d); end Pedido;` |
| Contestarle a uno | `chan resp[N](...)` + `send resp[idC](...)` | `Cliente[idC]!resp(...)` | **No hace falta**: vuelve por el `OUT` |
| ¿Hay algo? | `empty(canal)` | No existe | `Pedido'count` (sólo el dueño del entry) |

## 3. Las estructuras que deciden

**PMS — `do` con guardas (el Admin):**
```
do Cliente[*]?pedido(idC) -> push(Fila, idC);
[] not empty(Fila); Empleado[*]?listo(idE) -> pop(Fila, idC);
                                              Empleado[idE]!atender(idC);
od
```
`!` de un lado, `?` del otro, **siempre de a pares** · en la guarda **sólo `?`** · con varios del mismo tipo, **siempre índice** (`Empleado[idE]!`) · `[*]` sólo para recibir.

**Ada — `select` del servidor (elige a quién atender):**
```ada
for i in 1..P loop
   select
      accept A (...) do ... end A;
   or
      when (A'count = 0) =>
         accept B (...) do ... end B;
   end select;
end loop;
```
`or` **entre cada** alternativa · un solo `when` por alternativa, condiciones con **`and`** · el `when` **no usa parámetros** del entry · puede terminar con `or delay`, `else` u `or terminate`.

**Ada — `select` del cliente (límite a mi espera):**
```ada
select
   Empleado.Pago(p, c);          -- UNA sola llamada
or delay 600.0;                  -- o: else
   null;                         -- siempre una sentencia
end select;
```
Tiempo **con decimal** · `else` = *"inmediatamente"* · `or delay` = *"a lo sumo X"* · nunca mezclar llamadas y `accept`.

## 4. Las reglas de Ada que más se olvidan

1. **El parámetro sólo existe adentro del `do ... end`.** El `OUT` se asigna ahí, o el cliente se va sin nada.
2. **La llamada lleva variables, no declaraciones:** `Empleado.Urgencia(rec, med);`, nunca `(rec: IN Receta; ...)`.
3. **Mismo nombre y mismos parámetros** en `entry`, `accept` y llamada.
4. **Sin paréntesis** si el entry no tiene parámetros: `Admin.Siguiente;`, `accept Siguiente;`.
5. **Tarea sin entries:** `task Programada;` (sin `is ... end`).
6. `if ... then` · `elsif` · `else` · `end if;` — `for i in 1..N loop ... end loop;` — `loop ... end loop;`
7. **`'count` para prioridad, nunca `when E'count > 0`.**
8. *"Todas deben terminar"* + cantidad conocida ⇒ **`for`**, no `loop`.

## 5. Terminación (los tres)

| | Con límite (N conocidos, una vez) | Sin límite (*"infinitamente"*) |
|---|---|---|
| Un servidor | `for i = 1 to N` | `while (true)` / `loop` |
| Varios servidores | Admin cuenta y manda `false` / `-1` | `while (true)` |

*¿Quién le manda el mensaje que lo hace salir?*

## 6. Checklist antes de entregar

- [ ] ¿Usé el **mecanismo que pide** el enunciado?
- [ ] PMA/PMS: `Process`, `tipo nombre;` · Ada: `task`, `nombre: tipo;`, `:=`
- [ ] PMS: cada `!` tiene su `?` · índice en `Proceso[id]!`
- [ ] Ada: cada `entry` tiene su `accept`, igual nombre y parámetros · `OUT` adentro del `do`
- [ ] ¿Cada proceso termina (o el enunciado dice que no)?
- [ ] ¿Respeté orden de llegada / prioridad?
