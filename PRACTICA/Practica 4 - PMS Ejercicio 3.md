# Práctica 4 (PMS) — Ejercicio 3

> **Enunciado.** En un examen final hay **N alumnos** y **P profesores**. Cada alumno resuelve su examen, **lo entrega y espera a que alguno de los profesores lo corrija y le indique la nota**. Los profesores corrigen los exámenes **respetando el orden en que los alumnos van entregando**.
> **a)** Considerando que **P = 1**.
> **b)** Considerando que **P > 1**.
> **c)** Ídem b) pero considerando que los alumnos **no comienzan a realizar su examen hasta que todos hayan llegado** al aula.
>
> *Nota: maximizar la concurrencia; no generar demora innecesaria; **todos los procesos deben terminar su ejecución**.*

> **Cómo se resolvió:** en ping-pong. El (a) lo escribió Lucas completo. En el (b) le costó **el corte de los profesores** y se explicó con la solución; en el (c) planteó bien la barrera, con dos errores de PMS.

---

## 0. El planteo

Es el **ejercicio 1 (c)** (antivirus con orden) con otros nombres: Admin con cola, el profesor **pide** trabajo y la nota va **directo** al alumno.

**Lo nuevo es la terminación:** *"todos los procesos deben terminar"*. Hasta este ejercicio, todo era `while (true)`.

| Proceso | ¿Cómo termina? |
|---|---|
| Alumno | Entrega **un** examen y recibe **una** nota: termina solo |
| Profesor | (a) `for` de N vueltas · (b) cuando el Admin le manda un `-1` |
| Admin | Su `do` termina cuando **todas las condiciones son falsas** |

---

## 1. Ítem (a): un profesor

```
Process Admin {
  cola Fila; texto E; int idA; int contador = 0;
  do contador < N; Alumno[*]?examen(idA, E) -> push(Fila, (idA, E));
                                               contador++;
  [] not empty(Fila); Profesor?listo()      -> pop(Fila, (idA, E));
                                               Profesor!trabajo(idA, E);
  od
}

Process Profesor {
  int i, idA, res; texto E;
  for (i = 1 to N) {                         // corrige exactamente N
    Admin!listo();
    Admin?trabajo(idA, E);
    res = puntuar(E);
    Alumno[idA]!nota(res);
  }
}

Process Alumno[id: 1..N] {
  texto examen; int res;
  examen = resolverExamen();
  Admin!examen(id, examen);
  Profesor?nota(res);
}
```

### Cómo termina el Admin
La primera versión tenía la opción de los alumnos **sin condición**: una opción sin condición **nunca falla**, así que el `do` era un `while(true)`.

**La clave: el `do` termina cuando todas las condiciones son falsas.** Con `contador < N` en la primera opción:
- cuando recibió los N exámenes, la primera es falsa;
- cuando le pasa al profesor el último, `not empty(Fila)` también;
- **el `do` termina solo.**

No deja a nadie colgado: el último `Admin!listo()` del profesor es el de la vuelta N, y el Admin todavía tiene ese examen en la fila.

---

## 2. Ítem (b): P profesores

### El problema del corte
- Los profesores **no pueden hacer `for (i = 1 to N)`**: no saben cuántos exámenes les tocan (con 3 profesores y 10 exámenes, uno puede corregir 5, otro 3 y otro 2).
- Con `while (true)` **nunca terminan**. Y cuando el Admin termina su `do`, el próximo `Admin!listo(id)` de cada profesor **queda bloqueado para siempre**: nadie lo recibe.

### La idea: el Admin les dice "no hay más, andate"
**Quien sabe cuándo se terminó todo es el Admin**, porque cuenta los exámenes. Cuando un profesor pide trabajo y ya no queda nada (ni por llegar ni en la fila), le responde un examen falso con **`idA = -1`**. Es el mismo truco del `-1` de PMA.

**Tiene que avisarles a los P**, uno por uno cuando cada uno pide. Lleva un contador `terminados` y **no termina su `do` hasta haberles avisado a todos**.

```
Process Admin {
  cola Fila; texto E; int idA, idP;
  int contador = 0, terminados = 0;

  do contador < N; Alumno[*]?examen(idA, E) ->
        push(Fila, (idA, E));
        contador++;

  [] not empty(Fila); Profesor[*]?listo(idP) ->           // hay examen para darle
        pop(Fila, (idA, E));
        Profesor[idP]!trabajo(idA, E);

  [] (contador == N) and empty(Fila) and (terminados < P);
        Profesor[*]?listo(idP) ->                          // no queda nada: "andate"
        Profesor[idP]!trabajo(-1, "");
        terminados++;
  od
}

Process Profesor[id: 1..P] {
  int idA, res; texto E;
  bool seguir = true;
  while (seguir) {
    Admin!listo(id);
    Admin?trabajo(idA, E);
    if (idA == -1) seguir = false;            // no hay más: termino
    else {
      res = puntuar(E);
      Alumno[idA]!nota(res);
    }
  }
}

Process Alumno[id: 1..N] {
  texto examen; int res;
  examen = resolverExamen();
  Admin!examen(id, examen);
  Profesor[*]?nota(res);                    // no sabe qué profesor lo corrigió
}
```

| Condición de la 3ª opción | Por qué |
|---|---|
| `contador == N` | Ya entregaron todos: no llega ninguno más |
| `empty(Fila)` | No queda nada sin repartir |
| `terminados < P` | Todavía falta avisarle a algún profesor |

- **El `do` termina** cuando `contador == N`, la fila está vacía y `terminados == P`: **después** de avisarles a todos.
- **Si la fila está vacía pero faltan exámenes por llegar**, ninguna opción de profesor es verdadera: el Admin no recibe ese `listo` y el profesor espera hasta que llegue un examen. Está bien: no hay nada para darle todavía.

> **La regla de terminación con varios servidores:** el que **cuenta** (el Admin) es el que sabe cuándo se terminó. Avisa con un **valor especial** a **cada** servidor cuando éste pide trabajo, y no termina hasta haberles avisado a **todos**.
>
> Es la misma idea que el corralón (c) de monitores: allá era el `signalall` con la bandera de salida.

### El error del primer intento
El Profesor esperaba el `-1`, **pero el Admin no tenía la opción que lo manda**. El Admin terminaba su `do` al repartir los N exámenes, y los profesores quedaban bloqueados en `Admin!listo(id)` para siempre.

> **Chequeo para cualquier ejercicio con terminación:** para cada proceso, preguntate *"¿quién le manda el mensaje que lo hace salir?"*. Si la respuesta es *"nadie"*, queda colgado.

---

## 3. Ítem (c): con barrera al principio

**Ningún examen puede llegar antes de que estén todos**, así que el Admin no necesita atender las dos cosas a la vez: primero hace la barrera, **en orden y sin guardas**, y después el `do` del (b) tal cual. Es el **molde S5**: N recepciones y después N envíos.

```
Process Admin {
  cola Fila; texto E; int idA, idP, i;
  int contador = 0, terminados = 0;

  // --- barrera: espero a los N y después los largo ---
  for (i = 1 to N) Alumno[*]?llegue();
  for (i = 1 to N) Alumno[i]!empezar();

  // --- igual que el (b) ---
  do contador < N; Alumno[*]?examen(idA, E) ->
        push(Fila, (idA, E));
        contador++;
  [] not empty(Fila); Profesor[*]?listo(idP) ->
        pop(Fila, (idA, E));
        Profesor[idP]!trabajo(idA, E);
  [] (contador == N) and empty(Fila) and (terminados < P);
        Profesor[*]?listo(idP) ->
        Profesor[idP]!trabajo(-1, "");
        terminados++;
  od
}

Process Alumno[id: 1..N] {
  texto examen; int res;
  Admin!llegue();                   // llegué (una vez)
  Admin?empezar();                  // espero a que estén todos
  examen = resolverExamen();
  Admin!examen(id, examen);
  Profesor[*]?nota(res);
}

// Process Profesor: igual que en el (b)
```

- **En `llegue` no hace falta el id:** el Admin sólo cuenta.
- **El segundo `for` no se traba:** cuando el Admin empieza a enviar, los N alumnos ya están parados en `Admin?empezar()`.

### Los errores del primer intento
| Error | Por qué |
|---|---|
| `for (i = 1 to N) Admin!llegue(id)` en el **alumno** | Cada alumno llega **una vez**: avisa una vez. Con el `for` serían N × N avisos y los alumnos quedarían bloqueados en los que sobran |
| `Alumno[*]!todosLlegaron()` | **El `[*]` es sólo para recibir.** Para avisar a todos: `for (i = 1 to N) Alumno[i]!empezar()`. **En PMS no hay broadcast** |
| La barrera como una cuarta opción del `do` | Funciona, pero complica: como ningún examen llega antes, la barrera va **antes** del `do`, en orden |
| `contadorAlumnos` sin declarar | |

---

## 4. Chequeos de sanidad

- **Todos terminan:** cada alumno con su nota; cada profesor con su `-1`; el Admin cuando `terminados == P`. ✅
- **Cada `!` tiene quien lo reciba:** los `listo` que llegan después del último examen los recibe la 3ª opción. ✅
- **Con P = 1**, el (b) funciona igual que el (a), con un `-1` de más al final. ✅
- **El `-1` nunca es un alumno real:** los ids van de 1 a N. ✅
