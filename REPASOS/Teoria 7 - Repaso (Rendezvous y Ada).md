# Teoría 7 — RPC, Rendezvous y Ada

> Repaso **por moldes**, como el de PMA y PMS. Sale de la Teoría 7, las consideraciones de la Práctica 5 y la Explicación práctica 5.
>
> **Rendezvous es el mecanismo; Ada es el lenguaje que lo implementa.** En la práctica y en el parcial se escribe todo en Ada. No son dos formas de resolver, como PMA y PMS: es un solo tema. RPC es el otro mecanismo bidireccional, pero se ve sólo en la teoría.

### Las consideraciones de la Práctica 5 (las que se corrigen)
1. **No se pueden usar variables compartidas.**
2. **Los parámetros del `accept` sólo existen adentro de su cuerpo.** Si se necesitan después, se copian a una variable de la tarea.
3. **`select` del cliente** (entry call): **una sola llamada** y **una sola opción** (`or delay` **o** `else`).
4. **`select` del servidor** (accept): varias alternativas de `accept`, con o sin `when`, **pero nunca mezcladas con entry calls**. Puede llevar un `else` **o** un `or delay`, no los dos.
5. **Después de un `when` siempre va un `accept`.** La condición **no puede usar los parámetros** del entry.
6. **`'count` sólo lo consulta la tarea dueña del entry.**
7. **La comunicación es bidireccional, y hay que aprovecharla** siempre que no reduzca la concurrencia.

---

## 0. Qué cambia respecto de PMS

**En PMS la comunicación va en un solo sentido:** si el cliente quiere una respuesta, son **dos mensajes** (pedido y respuesta), y por eso el id tiene que viajar.

**En Rendezvous es bidireccional:** **una sola llamada va y vuelve**. El cliente llama a un `entry` del servidor y **se queda bloqueado hasta que el servidor termina el `accept`**. Los parámetros `OUT` vuelven con la respuesta.

| | PMS | Ada (Rendezvous) |
|---|---|---|
| Pedido con respuesta | `S!pedido(id, d)` + `S?respuesta(r)` | **`S.Pedido(d, r)`**: una llamada, `r` es `OUT` |
| ¿Hace falta mandar el id para responder? | **Sí** | **No**: la respuesta vuelve por la misma llamada |
| Recibir | `Cliente[*]?pedido(...)` | `accept Pedido(...) do ... end Pedido;` |
| ¿Respeta el orden de llegada? | **No**: `[*]` elige cualquiera | **Sí**: cada `entry` tiene **su cola FIFO**, y el `accept` atiende al más viejo |
| Elegir entre varias comunicaciones | `do cond; recepción -> ... [] ... od` | `select when cond => accept ...; or ... end select;` |
| ¿Quién puede ser receptor? | El que tiene la guarda | **Sólo la tarea que declara el `entry`** |
| Consultar cuántos esperan | No se puede | **`Entry'count`** |

> **La clave:** en PMS pensabas *"¿a quién le contesto?"*. En Ada, **el que llama espera solo la respuesta** en la misma llamada.

**RPC vs Rendezvous:** los dos son bidireccionales. **RPC** crea un proceso nuevo por cada llamada (como un procedure remoto), y la sincronización interna hay que programarla aparte. **Rendezvous** atiende con **un proceso que ya existe**, que decide cuándo atender con `accept`. Ada usa Rendezvous.

---

## 1. Sintaxis mínima de Ada

```ada
Procedure Nombre is

   task Servidor is                          -- especificación: qué entries ofrece
      entry Pedido (dato: IN texto; res: OUT texto);
   end Servidor;

   task type Cliente;                        -- tipo, para tener un arreglo
   arrClientes: array (1..N) of Cliente;

   task body Servidor is                     -- cuerpo
      aux: texto;
   begin
      loop
         accept Pedido (dato: IN texto; res: OUT texto) do
            res := resolver(dato);
         end Pedido;
      end loop;
   end Servidor;

   task body Cliente is
      d, r: texto;
   begin
      Servidor.Pedido(d, r);                 -- llama y espera la respuesta
   end Cliente;

begin
   null;                                     -- o el reparto de ids (ver § 3, R6)
end Nombre;
```

- **`task X is ... end X;`** declara una sola tarea. **`task type X`** + `array (1..N) of X` declara N.
- **Parámetros:** `IN` (entra), `OUT` (sale), `IN OUT`.
- **`accept E (...) do S end E;`**: el que llamó **queda esperando hasta el `end E`**. Lo que va después del `end E` lo hace el servidor solo, mientras el cliente ya siguió.
- **`accept E;`** sin `do`: es pura sincronización, no lleva datos.
- **Variables:** `nombre: tipo := valor;`. Asignación con `:=`.

### `select` en el servidor: la comunicación guardada
```ada
select
   when cond1 => accept E1 (...) do ... end E1;
                 sentencias;
or
   when cond2 => accept E2 (...) do ... end E2;
or
   accept E3;                                -- sin when: siempre abierta
end select;
```
Es el `do ... od` de PMS, con dos diferencias: **no repite solo** (va adentro de un `loop`) y **puede usar `'count`**. Puede terminar con un `else` (si no hay nadie, hace otra cosa), un `or delay t` o un **`or terminate`**.

### `select` en el cliente: llamada condicional o con tiempo
```ada
select                                       -- espera como mucho 10 minutos
   Servidor.Pedido(d, r);
   -- (lo atendieron)
or delay 600.0;
   -- (se cansó y se va)
end select;

select                                       -- si no lo atienden YA, se va
   Servidor.Pedido(d, r);
else
   -- (no lo atendieron)
end select;
```
**El cliente sólo puede tener una llamada en su `select`**: no puede esperar a dos servidores a la vez.

---

## 2. La regla del `do ... end`

**¿Qué va adentro del `accept`?** **Solamente lo que el cliente tiene que esperar.**

| Situación | Dónde va |
|---|---|
| El cliente necesita el resultado (`OUT`) | **Adentro**: si no, el cliente se va sin la respuesta |
| Sólo copiar un dato que trae el cliente | Adentro: `aux := dato;` |
| Trabajo largo que el cliente **no** espera | **Afuera**, después del `end E`: si va adentro, el cliente queda bloqueado de más |

Es la **regla 3 de siempre**: la operación larga va afuera. Y además, **mientras está adentro del `do`, el servidor no atiende a nadie más**.

---

## 3. Los moldes

### R1 — Cliente / Servidor con respuesta ⭐
**Se reconoce:** *"espera la respuesta"*, *"le entrega el comprobante"*.

```ada
task body Empleado is
begin
   loop
      accept Atencion (pedido: IN texto; res: OUT texto) do
         res := atender(pedido);
      end Atencion;
   end loop;
end Empleado;

-- Cliente:
Empleado.Atencion(miPedido, res);
```
**Sin id, sin canal de respuesta y con orden de llegada gratis**: la cola del `entry` es FIFO. Es lo que en PMS eran S1 + S2 juntos.

### R2 — Varios entries con condiciones (el monitor activo)
**Se reconoce:** *"hay lugar para N"*, buffer, lectores/escritores.

```ada
loop
   select
      when cantidad < N => accept Depositar (x: IN T) do push(buf, x); end Depositar;
                           cantidad := cantidad + 1;
   or
      when cantidad > 0 => accept Retirar (x: OUT T) do pop(buf, x); end Retirar;
                           cantidad := cantidad - 1;
   end select;
end loop;
```
Es **el monitor con `wait`/`signal`**, pero sin las `cond`: la condición del `when` cumple ese papel. **Si es falsa, los que llaman a ese entry esperan en su cola.**

### R3 — Prioridad con `'count`
**Se reconoce:** *"tienen prioridad los ancianos"*, *"primero los urgentes"*.

```ada
select
   accept Urgente (...) do ... end Urgente;
or
   when Urgente'count = 0 => accept Normal (...) do ... end Normal;
end select;
```
**Un entry por tipo de cliente**, y el de menor prioridad se abre **sólo si no hay nadie en la cola del otro**. Es el **A6** de PMA sin el canal `Aviso`: en Ada **esperar es gratis** y `'count` está permitido.

**¿Por qué no un solo entry con el tipo como parámetro?** Porque el `accept` atiende **al primero de la cola**, sea del tipo que sea. Si el primero es regular, lo atiende aunque haya prioritarios atrás (Explicación 5, ej. 3).

**¿El orden de las alternativas del `select` da prioridad?** **No.** Elige al azar entre las que se pueden ejecutar. La prioridad la da **sólo** el `when ...'count = 0`.

### ¿Cuándo se usa `'count`?
| Uso | Ejemplo |
|---|---|
| ✅ **Prioridad**: abrir un entry sólo si otro está vacío | `when Urgente'count = 0 => accept Normal ...` |
| ✅ Atender **a todos los que esperan** en una cola | `for i in 1..InicioLeer'count loop accept InicioLeer; end loop;` |
| ❌ **Nunca para ver si hay alguien esperando** | `when E'count > 0 => accept E;` **no tiene sentido**: si no hay nadie, igual no se puede aceptar. Y si se hace en todas las alternativas, **genera busy waiting** |

### R4 — El cliente que no espera para siempre
**Se reconoce:** *"si no es atendido en 10 minutos, se retira"* → `or delay`; *"si no lo atienden inmediatamente, se va"* → `else`. Va del **lado del cliente** (§ 1).

### R5 — El servidor que hace otra cosa si no hay nadie
**Se reconoce:** *"si no hay clientes, hace tareas administrativas"*.
```ada
select
   accept Atencion (...) do ... end Atencion;
else
   tareasAdministrativas;
end select;
```
Es el **A4** de PMA, sin busy waiting problemático y **sin el lío del A5**: como el `entry` es de una sola tarea, no hay otro receptor que robe el pedido.

### R6 — El servidor le habla a un cliente en particular (`Identificacion` + entry del cliente)
**Se reconoce:** el servidor no puede contestar **en el momento del `accept`**: tiene que **guardar el pedido y avisar más tarde** (orden de llegada con recurso ocupado, SJN, *"despertar al que…"*).

**Pieza nueva:** el cliente **también es una tarea con entries**. El servidor le avisa **llamándolo**: `arrClientes(id).Usar;`. Para eso cada cliente necesita saber su id, que se lo da el programa principal:

```ada
task type Cliente is
   entry Identificacion (i: IN integer);
   entry Usar;
end Cliente;
arrClientes: array (1..N) of Cliente;

task body Cliente is
   id: integer;
begin
   accept Identificacion (i: IN integer) do id := i; end Identificacion;
   Admin.Pedir(id);
   accept Usar;                              -- espero a que el Admin me llame
   usarRecurso;
   Admin.Liberar;
end Cliente;

task body Admin is                           -- el Alocador SJN de la Teoría 7
   libre: boolean := true;  espera: cola;  aux: integer;
begin
   loop
      aux := -1;
      select
         accept Pedir (id: IN integer) do
            if libre then libre := false; aux := id;
            else push(espera, id); end if;
         end Pedir;
      or
         accept Liberar;
         if empty(espera) then libre := true;
         else pop(espera, aux); end if;      -- se entrega, no se libera
      end select;
      if aux /= -1 then arrClientes(aux).Usar; end if;
   end loop;
end Admin;

begin                                        -- programa principal
   for i in 1..N loop arrClientes(i).Identificacion(i); end loop;
end;
```
**Es el PMS 5 (máquina expendedora) en Ada:** `bool libre` + cola + Passing the Baton.

**¿Por qué el `arrClientes(aux).Usar` va afuera del `select`?** Porque adentro del `accept Pedir` el cliente está bloqueado en la llamada: si el Admin lo llamara ahí, se trabarían los dos (deadlock). Se guarda en `aux` y se avisa después.

> **¿Hace falta R6 para "orden de llegada"?** **No, si el recurso lo maneja el servidor de a uno:** en R1 la cola del entry ya da el orden. Hace falta cuando **el servidor tiene que atender otras cosas mientras el cliente espera**.

### R7 — Varios servidores (empleados) y clientes → Admin intermedio
**Se reconoce:** *"hay 3 empleados que atienden a los clientes"*.

**Un `entry` es de una sola tarea:** el cliente no puede llamar a *"cualquier empleado libre"*, y dos empleados no pueden aceptar el mismo entry. La salida es la de PMS: **un Admin en el medio, y los empleados le piden trabajo**.

**La versión de la cátedra (Explicación 5, ej. 4): un `accept` adentro de otro, sin cola.**
```ada
task body Administrador is
begin
   loop
      accept Siguiente (id: OUT integer; dc: OUT texto) do      -- un empleado está libre
         accept Pedido (idC: IN integer; d: IN texto) do        -- espero al cliente más viejo
            id := idC;
            dc := d;
         end Pedido;
      end Siguiente;
   end loop;
end Administrador;

task body Empleado is
   res, dat: texto;  idC: integer;
begin
   loop
      Administrador.Siguiente(idC, dat);
      res := resolverPedido(dat);
      arrClientes(idC).Respuesta(res);                         -- le contesto al cliente
   end loop;
end Empleado;

task body Cliente is
   resultado: texto;  id: integer;
begin
   accept Ident (pos: IN integer) do id := pos; end Ident;
   Administrador.Pedido(id, "datos");
   accept Respuesta (r: IN texto) do resultado := r; end Respuesta;
end Cliente;
```
- **No hace falta una cola explícita:** el Admin acepta un pedido de cliente **recién cuando un empleado le pide**. Mientras tanto, los clientes esperan **en la cola del entry `Pedido`**, que ya respeta el orden.
- **El cliente no puede recibir la respuesta en la llamada al Admin**, porque el que resuelve es el empleado. Por eso son dos comunicaciones, y el cliente necesita su **id**, que le da el programa principal con `Ident`.
- **La regla:** *¿quién llama a quién?* Siempre llama el que **sabe a quién** llamar.

### R8 — Barrera
**Se reconoce:** *"no empiezan hasta que lleguen todos"*.
```ada
-- En el Admin:
for i in 1..N loop accept Llegada; end loop;
for i in 1..N loop accept Comenzar; end loop;      -- o arrClientes(i).Comenzar

-- Cliente:
Admin.Llegada;
Admin.Comenzar;                                     -- queda en la cola hasta que lleguen todos
```
**Más fácil que en PMS:** los clientes se quedan esperando en la cola del entry `Comenzar` sin hacer nada especial.

### R9 — Coordinador que reparte un cálculo entre N workers
**Se reconoce:** *"un vector distribuido entre 10 workers"*, *"envía la huella a los 8 servidores"*, *"cada sucursal calcula"*.

**La trampa (Explicación 5, ej. 5):** si el coordinador hace `for i: Worker(i).Contar(num, parcial)` con el resultado como `OUT`, **es secuencial**: el worker `i+1` no empieza hasta que el `i` devolvió su resultado.

**La versión buena: el worker pide el dato cuando está listo, y después entrega el resultado.**
```ada
task body Coordinador is
   numero: integer := elegirNumero;  total: integer := 0;
begin
   for i in 1..2*N loop                       -- N pedidos de dato + N resultados
      select
         accept Valor (num: OUT integer) do num := numero; end Valor;
      or
         accept Resultado (res: IN integer) do total := total + res; end Resultado;
      end select;
   end loop;
end Coordinador;

task body Worker is
   valor, cant: integer := 0;
begin
   Coordinador.Valor(valor);                  -- pido el dato
   cant := contar(valor);                     -- trabajo en paralelo con los demás
   Coordinador.Resultado(cant);               -- entrego
end Worker;
```
- **Por qué se invierte quién llama:** si el coordinador llamara en orden, uno lento demora a todos. Si llama el worker, cada uno arranca apenas puede.
- **`for 1..2*N`:** el coordinador sabe exactamente cuántos mensajes recibe en total.

---

## 4. Terminación

| Caso | Cómo |
|---|---|
| Se sabe cuántas veces atiende | **`for i in 1..N loop`** en vez de `loop` |
| El servidor no sabe cuándo terminar | **`or terminate`** en su `select`: la tarea termina cuando todas las que la podrían llamar terminaron |
| Varios servidores con límite | Igual que PMS 3(b): el Admin les avisa con un valor especial (`-1`) cuando ya no hay nada |

---

## 5. Cómo reconocer el molde (machete)

| Si el enunciado dice… | Molde | PMS equivalente |
|---|---|---|
| *"espera la respuesta"* | **R1**: un `accept` con `OUT` | S1 |
| *"en orden de llegada"* (un servidor) | **R1**: gratis, la cola del entry es FIFO | S2 (Admin con cola) |
| *"N lugares"*, buffer, *"si hay espacio"* | **R2**: `select when` | Guardas con condición |
| *"prioridad"* | **R3**: un entry por tipo + `'count` | Guardas + variables locales |
| *"si no lo atienden en X minutos, se va"* | **R4**: `select ... or delay` del lado del cliente | — |
| *"si no hay clientes, hace otra cosa"* | **R5**: `select ... else` en el servidor | A4 / A5 |
| *"el recurso se lo da cuando se libera"* | **R6**: `Identificacion` + entry del cliente | S4 / PMS 5 |
| *"K empleados atienden"* | **R7**: Admin intermedio | S3 |
| *"no empiezan hasta que estén todos"* | **R8**: dos `for` de `accept` | S5 |
| *"distribuido entre N workers"*, *"le envía a los N servidores"* | **R9**: el worker pide el dato y entrega el resultado | — |

**Las reglas que valen siempre:**
1. **Una llamada va y vuelve:** si el cliente necesita respuesta, es un parámetro `OUT` de la misma llamada.
2. **Adentro del `do` sólo va lo que el cliente tiene que esperar.**
3. **Un `entry` es de una sola tarea.** Para varios servidores, un Admin.
4. **Nunca llames a una tarea que te está llamando a vos** (deadlock): guardá el id y llamalo después del `end`.
5. **El orden de llegada es gratis** en la cola de cada entry.
6. **`'count` sólo para prioridad**, nunca para preguntar si hay alguien.

---

## 6. Los ejercicios de la Práctica 5 según su molde

| Ejercicio | Molde |
|---|---|
| 1. Puente de 5 unidades (autos, camionetas, camiones); (b) prioridad a camiones | **R2** (un entry por tipo con `when peso + x <= 5`) · (b) + **R3** |
| 2. Banco, un empleado: (a) normal, (b) 10 min, (c) inmediato, (d) 10 min y otra vez inmediato | **R1** + **R4**: `or delay` y `else` del lado del cliente |
| 3. Central y 2 periféricos con tiempos | **R4** en los periféricos + **R2/R5** con tiempos en la central |
| 4. Médico, enfermeras y personas | **R3** (prioridad a las personas) + **R4** (personas: `or delay`; enfermeras: `else`) + **R5** (notas cuando está libre) |
| 5. Servidor de acreditación, U usuarios | **R1** + **R4** (`or delay` de 2 min y reintento) |
| 6. 5 equipos de 4 en la playa | **R8** por equipo + **R9** para sumar y comparar |
| 7. Promedio de un vector entre 10 workers | **R9** |
| 8. 8 servidores de huellas y un especialista | **R9** (repetido indefinidamente) |
| 9. 3 camiones, P personas con reclamos | **R7** + elegir **al que más reclamos hizo** + **R4** (15 min) |

**La Práctica de repaso de MD** (en `PRACTICA/`) trae 3 de PM y 3 de Ada, con formato de parcial: **Ada 1** = R4 desde el que llama · **Ada 2** = R3 con tres niveles de prioridad · **Ada 3** = R9.
