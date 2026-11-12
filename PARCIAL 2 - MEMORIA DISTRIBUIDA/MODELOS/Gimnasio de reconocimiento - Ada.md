# Gimnasio de reconocimiento — Ada

> **Para qué:** el 12-nov, antes del parcial, Lucas dijo que le costaba leer el enunciado y saber si era Workers, Timeout u otra cosa (Prioridad ya la reconocía por la palabra). Se armaron **tres preguntas fijas** y dos rondas de enunciados cortos, **sin código**. Resultado: **7 de 7**.

## Las tres preguntas, en orden (la primera que da "sí" decide)

| # | Pregunta | Frases que la disparan | Modelo |
|---|---|---|---|
| 1 | ¿Alguien **se va** o **deja de esperar** si tarda? | *"a lo sumo X"*, *"si no lo atienden inmediatamente"*, *"si pasado ese lapso…"*, *"espera X y vuelve a intentar"* | **Timeout**: `select` del cliente con `or delay` / `else` |
| 2 | ¿Hay **un trabajo grande que se parte** entre varios y después **se junta**? | *"distribuido en N"*, *"cada uno tiene su parte"*, *"calcula el total / promedio / máximo"*, *"le envía a los N"* | **Workers**: `for 2*N` con `select` |
| 3 | ¿Varios **compiten por ser atendidos** por uno? | *"tienen prioridad"* (si no, es el banco (a): `accept` en un `loop`) | **Prioridad**: `when X'count = 0` |

> **La imagen:** Timeout = alguien **mira el reloj** · Workers = **un grupo con una tarea partida** (un TP: cada uno hace su parte y uno junta) · Prioridad = **una fila con preferencias**.

## Ronda 1 — un modelo por enunciado (4/4)

| | Enunciado | Modelo | La frase | Detalle |
|---|---|---|---|---|
| a | *Una central consulta a 15 sensores, de a uno; si uno no responde en 2 segundos lo marca como fallado.* | Timeout | *"si no responde en 2 segundos"* | **En serie** (Banco Central): `for` + `select` |
| b | *8 sucursales con sus ventas del mes; un gerente quiere saber cuál vendió más.* | Workers | *"cada una con sus ventas"* + *"cuál vendió más"* | Juntar el **máximo** (recomendaciones 2025) |
| c | *Un veterinario; los perros tienen prioridad sobre los gatos.* | Prioridad | *"tienen prioridad"* | **2 niveles**: un solo `when` |
| d | *Una cabina de locutorio; los clientes esperan a lo sumo 15 minutos y si no, se van.* | Timeout | *"a lo sumo 15 minutos"* | Muchos clientes, un servidor: banco (b) |

**Los dos sabores de Timeout:** (a) **uno consulta a muchos**, en serie, con `for` · (d) **muchos van a uno**, sin `for`, cada cliente con su `select`.

## Ronda 2 — se mezclan (3/3)

| | Enunciado | Modelo | Lo que agrega |
|---|---|---|---|
| e | *Un empleado de banco; los premium tienen prioridad sobre los regulares; los regulares se retiran si no los atienden en 20 minutos.* | **Prioridad + Timeout** | `select` con `or delay 1200.0` **sólo en los regulares**. **El Empleado no puede contar P** (algunos se van) ⇒ `loop` + **`or terminate`** |
| f | *Un servidor de 64 núcleos; tareas chicas (1) y grandes (4); las grandes tienen prioridad.* | **Prioridad con capacidad** | Puente / 128 núcleos: los `when` llevan **capacidad `and` `'count`**, más entries de **salida sin `when`** |
| g | *Un coordinador le envía a 10 robots una zona; cada uno limpia y avisa cuántos residuos juntó; después manda la zona siguiente.* | **Workers + `Siguiente`** | *"la zona siguiente"* ⇒ se repite ⇒ barrera. Sin cantidad de zonas ⇒ `loop` en los dos |

## La secuencia para el parcial

1. **¿Qué mecanismo pide?** (PMA, PMS o Ada)
2. **¿Alguien mira el reloj?** → Timeout.
3. **¿Trabajo partido que se junta?** → Workers. **¿Se repite?** → + `Siguiente`.
4. **¿Dice *"prioridad"*?** → Prioridad. **¿Hay capacidad?** → los `when` llevan además la capacidad.
5. **Anotar el o los modelos al margen** y recién después escribir.

Ver [Moldes — Ada](Moldes%20-%20Ada.md) para los esqueletos.
