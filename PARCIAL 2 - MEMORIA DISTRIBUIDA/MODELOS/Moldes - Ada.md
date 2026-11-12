# Moldes — Ada (Parcial de Memoria Distribuida)

> **De dónde salen:** los 10 parciales de memoria distribuida de 2022 a 2025 (carpeta `Parciales y soluciones anteriores`, sin subir). Desde 2024 el parcial son **3 ejercicios: PMA, PMS y Ada**.

| # | Modelo | Veces en parciales | Dónde está escrito |
|---|---|---|---|
| **A1** | **Prioridad de 3 niveles con `'count`** | **5** (hospital, aeropuerto, maratón, cobros digitales, banco premium) | Acá abajo |
| A2 | Coordinador y workers: una vez, K rondas o infinito | 4 (WordCount, recomendaciones, sucursales, stock) | Acá abajo + [Práctica 5 — Ej. 7](../../PRACTICA/Practica%205%20-%20Ejercicio%207.md) |
| A3 | Recurso con capacidad + prioridad por tipo (128 núcleos de 1/2/3) | 1 | [Práctica 5 — Ej. 1](../../PRACTICA/Practica%205%20-%20Ejercicio%201.md) (el puente) |
| A4 | Cliente con `or delay` / `else` + contador de intentos (UP16) | 2 | [Práctica 5 — Ej. 2](../../PRACTICA/Practica%205%20-%20Ejercicio%202.md) dentro de un `while` |

---

## A1 — Prioridad de 3 niveles: la farmacia del hospital (parcial del 10-11-25)

> En la farmacia de un hospital hay una **única ventanilla** para dispensar medicamentos. Los pacientes internados deben retirar sus medicamentos presentando una **receta** (un único retiro por persona). Los pacientes son atendidos en el orden de llegada, pero **con prioridad**: los de **urgencia** tienen prioridad sobre los de **internación general**, y ambos sobre los **ambulatorios**. Cada paciente entrega su receta y recibe la medicación correspondiente. *Notas: `obtenerPrioridad()` retorna 0 urgencia, 1 internado general, 2 ambulatorio; `obtenerReceta()` retorna la receta del paciente; `dispensarMedicamento(receta)` le retorna al empleado el medicamento. El empleado atiende de a un paciente por vez, según la prioridad. **Todas las tareas deben finalizar.***

> **Cómo se resolvió:** en ping-pong. El Paciente salió con la lógica bien y errores de sintaxis; el Empleado con los `when` perfectos pero `dispensarMedicamento` **afuera** del `accept`.

### El planteo

| Pregunta | Respuesta |
|---|---|
| Tareas | `Empleado` (`task`) · `Paciente` (`task type`, arreglo de **P**) |
| Entries | **Tres, uno por prioridad.** El `when` no puede usar parámetros (como el puente) |
| ¿Y la medicación? | Vuelve **por la misma llamada** con un `OUT`. Un entry `DarMedicacion` aparte es pensar en PMS |
| Condiciones | Urgencia sin `when` · Internación `when Urgencia'count = 0` · Ambulatorio `when` las dos en 0 |
| Terminación | *"Todas deben finalizar"* + P pacientes que retiran una vez ⇒ **`for i in 1..P`**. Con `loop` el Empleado queda bloqueado para siempre |
| ¿Cómo elige el entry el Paciente? | Un **`if` común antes de llamar**. El `select` del cliente no lleva condiciones |

**¿Un `task type` o tres?** Si el enunciado da cuántos hay de cada tipo (*"A autos, B camionetas"*) ⇒ tres tipos con su arreglo (puente). Si da una **función** que devuelve el tipo ⇒ **un solo tipo con un `if`**.

### La solución

```ada
Procedure Farmacia is
   task Empleado is
      entry Urgencia    (rec: IN Receta; med: OUT Medicamento);
      entry Internacion (rec: IN Receta; med: OUT Medicamento);
      entry Ambulatorio (rec: IN Receta; med: OUT Medicamento);
   end Empleado;

   task type Paciente;
   arrPacientes: array (1..P) of Paciente;

   task body Empleado is
   begin
      for i in 1..P loop
         select
            accept Urgencia (rec: IN Receta; med: OUT Medicamento) do
               med := dispensarMedicamento(rec);
            end Urgencia;
         or
            when (Urgencia'count = 0) =>
               accept Internacion (rec: IN Receta; med: OUT Medicamento) do
                  med := dispensarMedicamento(rec);
               end Internacion;
         or
            when (Urgencia'count = 0) and (Internacion'count = 0) =>
               accept Ambulatorio (rec: IN Receta; med: OUT Medicamento) do
                  med := dispensarMedicamento(rec);
               end Ambulatorio;
         end select;
      end loop;
   end Empleado;

   task body Paciente is
      prio: integer;
      rec: Receta;
      med: Medicamento;
   begin
      prio := obtenerPrioridad;
      rec := obtenerReceta;
      if prio = 0 then
         Empleado.Urgencia(rec, med);
      elsif prio = 1 then
         Empleado.Internacion(rec, med);
      else
         Empleado.Ambulatorio(rec, med);
      end if;
   end Paciente;

begin
   null;
end Farmacia;
```

### La regla de la llamada

| Dónde | Qué se escribe |
|---|---|
| `entry` | `Urgencia (rec: IN Receta; med: OUT Medicamento);` |
| `accept` | `accept Urgencia (rec: IN Receta; med: OUT Medicamento) do` |
| **llamada** | **`Empleado.Urgencia(rec, med);`**: sólo las variables |

### Lo que hubo que corregir

| Error | Corrección |
|---|---|
| ⚠️ Planteo: 2 entries, `Atencion(prioridad, receta)` + `DarMedicacion` | 3 entries por prioridad; la medicación vuelve por `OUT` |
| ⚠️ Planteo: `when prioridad = x` | El `when` no usa parámetros: **`'count`** |
| ⚠️ Planteo: `loop` "porque no sabemos el límite" | Son P y retiran una vez ⇒ `for i in 1..P` |
| ⚠️ `accept Urgencia(...);` y `dispensarMedicamento(rec);` afuera | Adentro del `do`: `med := dispensarMedicamento(rec);`. Afuera `rec` no existe y el paciente se va sin `med` |
| ⚠️ Llamada `Urgencia(rec: IN Receta; med: OUT Medicamento);` | `Empleado.Urgencia(rec, med);` |
| `med` sin declarar · `p` declarada y `prio` usada · `receta`/`rec` | Mismos nombres en todos lados, y declarar la variable del `OUT` |
| `prio = obtenerPrioridad()` | Asignación con `:=` |
| `if prio = 0;` · `else if` · `else;` | `if ... then` · `elsif` · `else` |
| `select` suelto antes del `for` · `For i in P` · `end for` | `for i in 1..P loop` ... `end loop;` |
| `when A'count = 0 & when B'count = 0` | Un solo `when`, unido con `and` |
| `array (1..N)` con `for 1..P` | El mismo P en los dos |

### Chequeos de sanidad
- **P llamadas, P vueltas:** cada paciente llama una vez. ✅
- **Con un urgente esperando**, los `when` de los otros dos son falsos. ✅
- **Orden de llegada dentro de cada prioridad:** cada cola de entry es FIFO. ✅
- **El `OUT` se asigna adentro del `do`.** ✅


### El esqueleto de la prioridad, con sus variantes

```ada
-- Cliente
[nada | loop]                                   -- ¿vuelve muchas veces?
   prio := obtenerPrioridad;                    -- sólo si el tipo sale de una función
   if prio = 0 then
      [select] Empleado.Alta(d, r);             -- [select] sólo si "se va si espera"
      [or delay X.0; null; end select;]
   elsif prio = 1 then
      Empleado.Media(d, r);
   else
      Empleado.Baja(d, r);
   end if;
[nada | end loop;]

-- Empleado
[for i in 1..P loop | loop]                     -- for: todos llegan a ser atendidos
   select
      accept Alta (d: IN T; r: OUT T) do r := f(d); end Alta;
   or
      when (Alta'count = 0) =>
         accept Media (d: IN T; r: OUT T) do r := f(d); end Media;
   or
      when (Alta'count = 0) and (Media'count = 0) =>
         accept Baja (d: IN T; r: OUT T) do r := f(d); end Baja;
   [or
      terminate;]                               -- sólo con loop + "todas deben terminar"
   end select;
end loop;
```

| Hueco | Opción | Cuándo |
|---|---|---|
| Afuera del Empleado | **`for i in 1..P`** | Cada cliente viene una vez y **todos** llegan a ser atendidos |
| | **`loop`** | *"Atiende continuamente"*, o **algún cliente se puede ir** (no hay P `accept`s seguros) |
| `or terminate` | Sí | Con `loop` y *"todas las tareas deben terminar"* |
| `select` en el cliente | `or delay X.0` / `else` | *"Se retira si espera más de X"* / *"si no lo atienden inmediatamente"* |
| `if` en el cliente | Con `if` | El tipo sale de una **función** (`obtenerPrioridad()`) |
| | Sin `if`: un `task type` por tipo | El enunciado da **cantidades** (*"R regulares y P premium"*) |
| `when` | Sólo `'count` | Prioridad pura |
| | **Capacidad `and` `'count`** + entries de salida sin `when` | Recurso con capacidad (núcleos, puente) |

**La trampa:** si algún cliente se puede ir, el `for 1..P` del Empleado queda bloqueado esperando a uno que ya no está ⇒ `loop` + `or terminate`.
---

## A2 — Coordinador y workers: WordCount (parcial del 4-11-24) y sus tres variantes

> El problema WordCount consiste en calcular la cantidad de veces que aparece una palabra en un texto. La base de datos de texto está **distribuida en N partes**. Un **administrador** determina la palabra a buscar y se la envía a las **N tareas contadoras** para que la busquen en su parte. Finalmente, el administrador **calcula el total** de apariciones. *Notas: los contadores tienen `contarPalabras(pal)`; el administrador tiene `obtenerPalabra()`. Se resuelve **una única vez** y hay que **maximizar la concurrencia**.*

> **Cómo se resolvió:** el 12-nov, antes del parcial. Lucas lo confundió con Passing the Baton y pidió la resolución.

### No es Passing the Baton

| | Passing the Baton | Coordinador (R9) |
|---|---|---|
| ¿Qué pasa? | **Un recurso** que usan **de a uno** y se van pasando | **Un trabajo** que se **reparte** entre N que trabajan **a la vez** |
| Ejemplos | Bicicleta, SUBE, PMS 5, R6 | Promedio (ej 7), WordCount, sucursales, recomendaciones |
| Se reconoce por | *"de a una por vez"*, *"usa y deja al siguiente"* | *"distribuido en N partes"*, *"envía a las N"*, *"calcula el total"* |

### La solución (una vez)

```ada
Procedure WordCount is
   task Admin is
      entry Palabra (pal: OUT string);
      entry Cantidad (cant: IN integer);
   end Admin;

   task type Contador;
   arrContadores: array (1..N) of Contador;

   task body Contador is
      pal: string;
      cant: integer;
   begin
      Admin.Palabra(pal);              -- pido la palabra
      cant := contarPalabras(pal);     -- busco en mi parte
      Admin.Cantidad(cant);            -- entrego mi parcial
   end Contador;

   task body Admin is
      palabra: string;
      total: integer := 0;
   begin
      palabra := obtenerPalabra;
      for i in 1..2*N loop
         select
            accept Palabra (pal: OUT string) do
               pal := palabra;
            end Palabra;
         or
            accept Cantidad (cant: IN integer) do
               total := total + cant;
            end Cantidad;
         end select;
      end loop;
   end Admin;

begin
   null;
end WordCount;
```

- **Los contadores llaman al Admin** (si el Admin los recorre con un `for`, queda clavado en el lento). La palabra es **`OUT`** para el contador: la recibe.
- **`2*N`:** cada contador hace 2 llamadas. **El `for` de 2N ya es el corte:** el Admin termina solo.
- Comparado con el ej 7: `accept Comenzar;` pelado → `accept Palabra (pal: OUT string) do pal := palabra; end Palabra;`. Lo demás, igual.

### Las tres variantes según el enunciado

| El enunciado dice… | Contador | Admin | ¿Barrera `Siguiente`? |
|---|---|---|---|
| **1. *"Una única vez"*** (WordCount, ej 7) | **Sin loop** | `for 1..2*N` con `select` | **No** |
| **2. *"K palabras"*** / *"todas deben terminar"* con varias | **`for k in 1..K loop`** | **`for k in 1..K loop`** con nueva palabra, `for 2*N` y `for N accept Siguiente` | **Sí** |
| **3. *"Cuando termina, sigue con la siguiente"*** (sucursales, recomendaciones) | **`loop`** | **`loop`** con lo mismo que la 2 | **Sí** |

**La 2 y la 3 son el mismo código: sólo cambia `for k in 1..K loop` por `loop`.**

```ada
-- Contador
[nada | for k in 1..K loop | loop]
   Admin.Palabra(pal);
   cant := contarPalabras(pal);
   Admin.Cantidad(cant);
   Admin.Siguiente;                    -- sólo en la 2 y la 3
[nada | end loop;]

-- Admin
[nada | for k in 1..K loop | loop]
   palabra := obtenerPalabra;
   total := 0;
   for i in 1..2*N loop
      select
         accept Palabra (pal: OUT string) do pal := palabra; end Palabra;
      or
         when (Palabra'count = 0) =>   -- opcional: despachar primero al que arranca
            accept Cantidad (cant: IN integer) do total := total + cant; end Cantidad;
      end select;
   end loop;
   for i in 1..N loop                  -- sólo en la 2 y la 3
      accept Siguiente;
   end loop;
[nada | end loop;]
```

- **¿Por qué la barrera?** Un contador rápido termina, entrega y vuelve a pedir la palabra **en la misma ronda**: ocupa el lugar de otro y los resultados se mezclan entre palabras.
- **Regla:** Contador y Admin tienen **siempre la misma estructura exterior** (nada / `for 1..K` / `loop`). Si no coinciden, alguno queda bloqueado para siempre.

### N y K no son lo mismo

| | Cuenta… | Sale de… |
|---|---|---|
| **N** | **Cuántos trabajan a la vez** | *"distribuida en N partes"*, *"100 sucursales"*, `array (1..N)` |
| **K** | **Cuántas rondas** | *"K palabras"*, *"una única vez"* (K = 1), *"sigue con el siguiente"* (K = ∞) |

El `for 2*N` es **una ronda** (cada uno de los N llama 2 veces); el `for K` repite la ronda entera. *Un profesor con 30 alumnos y 3 exámenes:* adentro 60 vueltas, afuera 3.
