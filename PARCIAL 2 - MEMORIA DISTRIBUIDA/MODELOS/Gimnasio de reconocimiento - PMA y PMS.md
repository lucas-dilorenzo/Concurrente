# Gimnasio de reconocimiento — PMA y PMS (y el traductor a Ada)

> **Para qué:** el 12-nov, antes del parcial. Lucas temía **enunciados espejo**: el mismo problema en PMA/PMS y en Ada, o tema 1 y tema 2 con la misma historia. **Los parciales lo confirman**, así que la clave es saber traducir un problema a los tres mecanismos.

## Los espejos que ya pasaron

| Tipo | Ejemplos |
|---|---|
| **El mismo problema en PMA y PMS** (mismo parcial) | 1ra fecha 2024: UP16 (*"resuelva el mismo problema pero con PMS"*) · repaso 2026: impresoras a y b |
| **El mismo problema en PMx y en Ada** | **Cobros digitales con prioridad:** PMA 2023, Ada repaso 2026 · **Maratón con prioridad:** PMA 2024 fecha 2, Ada 2024 fecha 3 |
| **Tema 1 y tema 2: misma historia con variante** | 1ra fecha 2025: estadio infinito vs. que termina · bicicleta por llegada vs. por id · hospital vs. aeropuerto |

## El traductor: un problema, tres mecanismos

| Problema | PMA | PMS | Ada |
|---|---|---|---|
| **Varios empleados para cualquier cliente** | **Canal compartido**, sin Admin | **Admin con cola** + `listo(idE)` | Admin (R7) |
| **Un servidor, orden de llegada** | Un canal + `respuesta[id]` | Admin con cola | `accept` en un `loop` (banco a) |
| **Prioridad** | **Canal por nivel + `señal`** | **Admin con una cola por nivel** | **Entry por nivel + `'count`** |
| **Si no hay nadie, hace otra cosa** | `empty(canal)` (con varios servidores, por un Admin) | No se puede sin bloquearse: no lo toman | `select ... else` en el servidor |
| **Terminar** | Admin cuenta y manda `false` / `-1` | `for` si es uno; con varios, Admin con contador que manda `-1` | `for 1..P` u `or terminate` |
| **Recurso único, de a uno** | Admin **simple**: `receive pedido(id); send usar[id](); receive liberar();` | Admin **con cola y `bool libre`** (PMS 5, SUBE) | `accept Usar` / R6 |
| **Por id, en orden** | `for i`: `send pasa[i](); receive termine();` | `for i` con `Cliente[i]?` | `for i` con `arrClientes(i).Turno` |
| **Timeout** (*"espera a lo sumo"*) | — no se pide | — no se pide | `select` del cliente con `or delay` |

### ¿Cuándo hace falta Admin?

| | Regla |
|---|---|
| **PMA** | Casi nunca: **el canal es la fila FIFO**. Sólo para: **recurso de a uno** (Admin simple), **terminar con varios servidores** (`false`), **`empty` con varios servidores** |
| **PMS** | **Siempre que haya orden de llegada y alguien espere**: no hay fila, `Cliente[*]?` agarra a cualquiera. **Excepción: orden por id** ⇒ `for` con `Proceso[i]?` |

> *En PMA la fila viene gratis (el canal); en PMS la armás vos (Admin con cola).*

| El enunciado dice… | Se resuelve con… |
|---|---|
| *"orden de llegada"* | **Cola**: el canal en PMA, el Admin con cola en PMS |
| *"orden de id / de número / de libreta"* | **`for i`**: sin cola ni Admin |

## Las preguntas fijas para PMA/PMS

1. **¿Hay varios servidores para cualquier cliente?** PMA: canal compartido · PMS: Admin con cola.
2. **¿Dice *"prioridad"*?** PMA: canal por nivel + `señal` · PMS: Admin con varias colas.
3. **¿Hacen otra cosa si no hay nadie?** `empty`, sólo PMA (con varios servidores, por un Admin).
4. **¿Tienen que terminar?** ¿Quién manda el `false` / `-1`?
5. **¿Por id o recurso de a uno?** `for i` / Admin con `libre`.

> **Subrayar cada frase con *"orden"*, *"terminan"*, *"prioridad"* o *"si no hay nadie"*:** cada una agrega una pieza. *Una frase, un molde.*

---

## Ronda 1 (4/4, con una duda clave en la b)

| | Enunciado | PMA | PMS |
|---|---|---|---|
| a | *4 bibliotecarios; los socios piden un libro y cualquier bibliotecario libre se lo entrega.* | ✅ Canal compartido | ✅ Admin con cola |
| b | *Una máquina de café que usan E empleados de a uno, por orden de llegada.* | Admin **simple** (el canal es la fila) | Admin **con cola + `libre`** |
| c | *Un empleado; si no hay nadie esperando, ordena papeles.* | ✅ `empty` | ✅ No se toma en PMS |
| d | *Un médico; los de urgencia tienen prioridad.* | ✅ Canal por nivel + `señal` | ✅ Admin con cola por nivel (ver abajo) |

**La duda de la b:** *"pensé que el admin siempre es necesario si necesitamos orden, pero al ser de a uno tal vez no"*. **En los dos hace falta alguien que dé turnos** (la máquina no dice *"estoy libre"* sola); lo que cambia es si el Admin necesita cola: en PMA no, en PMS sí.

**PMS con prioridad (d):**
```
do Paciente[*]?urgente(id) -> push(colaU, id);
[] Paciente[*]?normal(id)  -> push(colaN, id);
[] (not empty(colaU) or not empty(colaN)); Medico?listo() ->
      if (not empty(colaU)) pop(colaU, id) else pop(colaN, id);
      Medico!atender(id);
od
```

## Ronda 2 — las dos fallaron por saltearse una frase

| | Enunciado | Lo que respondió | Lo que faltaba |
|---|---|---|---|
| e | *3 cajeros; orden de llegada; **los cajeros se van cuando ya no quedan clientes**.* | PMA canal compartido · PMS Admin con cola | **Terminación.** PMA: **M2**, Admin que cuenta y manda `false` · PMS: Admin cuenta y contesta `-1`; cajeros con `while (seguir)` |
| f | *20 alumnos rinden un oral de a uno, **en orden de su número de libreta**.* | ❌ PMS `Cliente[*]?` + cola | **Por id ⇒ `for i`, sin cola ni Admin.** `[*]` es "cualquiera": lo contrario de lo que pide |

**f en PMS:**
```
Process Profesor {
  for i = 1 to 20 {
    Alumno[i]?llegue();      // exactamente el alumno i
    // toma el oral
    Alumno[i]?termine();
  }
}
Process Alumno[id: 1..20] {
  Profesor!llegue();
  rendir();
  Profesor!termine();
}
```

**f en PMA** (cada alumno espera en su canal propio):
```
chan pasa[20]();
chan termine();

Process Profesor {
  for i = 1 to 20 {
    send pasa[i]();
    receive termine();
  }
}
Process Alumno[id: 1..20] {
  receive pasa[id]();
  rendir();
  send termine();
}
```

Ver [Moldes — PMA](Moldes%20-%20PMA.md), [Gimnasio — Ada](Gimnasio%20de%20reconocimiento%20-%20Ada.md) y la [Hoja de sintaxis](Hoja%20de%20sintaxis.md).
