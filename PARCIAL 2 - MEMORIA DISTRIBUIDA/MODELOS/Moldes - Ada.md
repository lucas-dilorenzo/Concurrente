# Moldes — Ada (Parcial de Memoria Distribuida)

> **De dónde salen:** los 10 parciales de memoria distribuida de 2022 a 2025 (carpeta `Parciales y soluciones anteriores`, sin subir). Desde 2024 el parcial son **3 ejercicios: PMA, PMS y Ada**.

| # | Modelo | Veces en parciales | Dónde está escrito |
|---|---|---|---|
| **A1** | **Prioridad de 3 niveles con `'count`** | **5** (hospital, aeropuerto, maratón, cobros digitales, banco premium) | Acá abajo |
| A2 | Coordinador y workers, una vez o con `loop` + barrera `Siguiente` | 4 (WordCount, recomendaciones, sucursales, stock) | [Práctica 5 — Ej. 7](../../PRACTICA/Practica%205%20-%20Ejercicio%207.md) |
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
