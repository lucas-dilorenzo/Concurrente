# Práctica 5 (Ada) — Ejercicio 1

> **Enunciado.** Se requiere modelar un **puente de un único sentido** que soporta hasta **5 unidades de peso**. El peso de los vehículos depende del tipo: cada **auto pesa 1** unidad, cada **camioneta pesa 2** unidades y cada **camión 3** unidades. Suponga que hay una cantidad innumerable de vehículos (A autos, B camionetas y C camiones). Analice el problema y defina qué tareas, recursos y sincronizaciones serán necesarios/convenientes para resolverlo.
> **a.** Realice la solución suponiendo que todos los vehículos tienen la misma prioridad.
> **b.** Modifique la solución para que tengan mayor prioridad los camiones que el resto de los vehículos.

> **Cómo se resolvió:** en ping-pong. El planteo fue de Lucas (tareas, un entry por tipo, las condiciones). Le costó ver **la salida del puente**, que se explicó con la caseta de peaje. Después escribió el programa completo, con detalles de sintaxis, y dedujo solo la condición del (b).

---

## 0. El planteo

Es el **puente de monitores** (P3 ej. 4) traducido a Ada: el monitor con `peso_total` pasa a ser **una tarea `Puente`**.

| Pregunta | Respuesta |
|---|---|
| Tareas | `Puente` (`task`, una sola) · `Auto`, `Camioneta`, `Camion` (`task type`, con un arreglo cada uno) |
| Entries del Puente | **Uno por tipo para entrar** (`EntrarAuto`, `EntrarCamioneta`, `EntrarCamion`) + **uno para salir** con el peso como parámetro |
| Condiciones | `when pesoTotal + peso <= 5` en cada entrada; salir **sin condición** |

### ¿Por qué un entry por tipo y no un solo `Entrar(peso)`?
**El `when` no puede usar los parámetros del entry.** La condición depende del peso, así que el puente tiene que saberlo **antes** de aceptar. Con un entry por tipo, el entry al que llamás ya dice cuánto pesás.

**Para salir alcanza con uno**, porque no tiene condición: el peso va como parámetro y se usa adentro del cuerpo.

### Cada vehículo hace dos llamadas, como con el monitor
**La caseta de peaje:** al entrar le pedís permiso al empleado (si no hay lugar, esperás en la fila de tu tipo); cruzás mientras él atiende a otros; al salir le avisás *"salí, pesaba 1"* y él resta. **Sin el aviso de salida, el contador nunca baja** y, después de unos pocos vehículos, no entra nadie más.

---

## 1. Ítem (a): misma prioridad — molde R2

```ada
Procedure Transito is
   task Puente is
      entry EntrarAuto;
      entry EntrarCamioneta;
      entry EntrarCamion;
      entry LiberarPuente (peso: IN integer);
   end Puente;

   task type Auto;        arrAutos: array (1..A) of Auto;
   task type Camioneta;   arrCamionetas: array (1..B) of Camioneta;
   task type Camion;      arrCamiones: array (1..C) of Camion;

   task body Puente is
      pesoTotal: integer := 0;
   begin
      loop
         select
            when pesoTotal + 1 <= 5 => accept EntrarAuto;
                                       pesoTotal := pesoTotal + 1;
         or
            when pesoTotal + 2 <= 5 => accept EntrarCamioneta;
                                       pesoTotal := pesoTotal + 2;
         or
            when pesoTotal + 3 <= 5 => accept EntrarCamion;
                                       pesoTotal := pesoTotal + 3;
         or
            accept LiberarPuente (peso: IN integer) do
               pesoTotal := pesoTotal - peso;
            end LiberarPuente;
         end select;
      end loop;
   end Puente;

   task body Auto is
   begin
      Puente.EntrarAuto;
      cruzar;
      Puente.LiberarPuente(1);
   end Auto;

   task body Camioneta is
   begin
      Puente.EntrarCamioneta;
      cruzar;
      Puente.LiberarPuente(2);
   end Camioneta;

   task body Camion is
   begin
      Puente.EntrarCamion;
      cruzar;
      Puente.LiberarPuente(3);
   end Camion;

begin
   null;
end Transito;
```

### La suma afuera, la resta adentro
| | Dónde | Por qué |
|---|---|---|
| **Suma** al entrar | **Afuera** del `accept` | El vehículo no necesita esperarla (no le devuelve nada). **No se puede colar nadie** entre el `accept` y la suma: el puente es **una sola tarea** que hace una cosa por vez, y `pesoTotal` es local |
| **Resta** al salir | **Adentro** del `do` | Usa el parámetro `peso`, que **sólo existe adentro del cuerpo** del `accept` |

Poner la suma adentro también es correcto: el auto espera un instante más, sin necesidad. **Adentro del `do` va sólo lo que el que llama tiene que esperar** (o lo que usa los parámetros).

### `pesoTotal: integer := 0;`: ¿no se reinicia?
**Lo que está entre `is` y `begin` se ejecuta una sola vez**, al crear la tarea. Lo que se repite es sólo lo que está entre `loop` y `end loop`.

### Lo que hubo que corregir
| Error | Corrección |
|---|---|
| ⚠️ Un solo `or`, antes de la última alternativa | **Va un `or` entre cada par** de alternativas |
| ⚠️ `entry LiberarPuente;` sin parámetro | La declaración tiene que coincidir con el `accept`: `(peso: IN integer)` |
| `Procedure Puente` y `task Puente` | Mismo nombre: se pisan. El programa se llama distinto |
| `pesoTotal: int;` sin inicializar | `integer := 0` |
| `Puente.EntrarAuto();` | **Sin paréntesis** si el entry no tiene parámetros |
| `< 5` en el primer planteo | *"Soporta hasta 5"* → **`<= 5`** (camión + camioneta = 5 entran juntos) |

**Orden:** dentro de cada tipo hay orden de llegada (cola FIFO de cada entry); entre tipos no, y el enunciado no lo pide.

---

## 2. Ítem (b): prioridad a los camiones — molde R3

```ada
select
   when (pesoTotal + 1 <= 5) and (EntrarCamion'count = 0) => accept EntrarAuto;
                                                             pesoTotal := pesoTotal + 1;
or
   when (pesoTotal + 2 <= 5) and (EntrarCamion'count = 0) => accept EntrarCamioneta;
                                                             pesoTotal := pesoTotal + 2;
or
   when pesoTotal + 3 <= 5 => accept EntrarCamion;
                              pesoTotal := pesoTotal + 3;
or
   accept LiberarPuente (peso: IN integer) do
      pesoTotal := pesoTotal - peso;
   end LiberarPuente;
end select;
```

- **El camión y la salida no cambian.**
- **Con un camión esperando y `pesoTotal = 3`:** el camión no entra (3 + 3 > 5) y **un auto tampoco**, aunque entraría (3 + 1 ≤ 5), porque hay un camión en la fila. **Eso es la prioridad.**
- **No se traba:** los de arriba van a salir, y `LiberarPuente` no tiene `when`.
- **`'count` y no un parámetro:** el `when` no puede usar parámetros; `EntrarCamion'count` pregunta cuántos camiones esperan sin aceptar a ninguno.

**Con el monitor de la P3**, la prioridad pedía cola, contadores y `signal` en orden. **Acá es una condición.**

---

## 3. Chequeos de sanidad

- **Nunca se pasa de 5:** cada entrada se acepta sólo si `pesoTotal + peso <= 5`. ✅
- **El peso siempre baja:** cada vehículo llama a `LiberarPuente` con su peso, y esa alternativa no tiene `when`. ✅
- **Cada entry declarado tiene su `accept`**, con los mismos parámetros. ✅
- **Con un solo tipo de vehículo**, el (a) y el (b) se comportan igual. ✅
