# Moldes — PMS (Parcial de Memoria Distribuida)

> **La regla madre:** en PMS **no hay fila**: `Cliente[*]?` agarra a **cualquiera** de los que esperan. Si hay orden de llegada y alguien espera, **la fila la arma un Admin con una cola**. La única excepción es el orden por id.

| # | Modelo | Frase gatillo | Parciales | Práctica |
|---|---|---|---|---|
| **S1** | **Admin con cola + varios servidores** | *"alguno de los K empleados"*, *"la primera impresora libre"* | UP16 2024, estadio 5 boleterías 2025, impresoras repaso | PMS 4c |
| **S2** | **Recurso único, de a uno (Admin con `libre`)** | *"usa y deja al siguiente"*, *"de a una persona"* | Bicicleta 2025, SUBE 2023 y repaso | PMS 5 |
| **S3** | **Por id: `for i` con `Proceso[i]?`** | *"según su identificador"*, *"en orden de libreta"* | Médico 2024, bicicleta tema 2 2025 | PMS 4b |
| **S4** | **Prioridad: Admin con una cola por nivel** | *"tienen prioridad"* | (en PMS no salió; en PMA y Ada sí) | — |
| + | **Terminación** con varios servidores: Admin cuenta y manda `-1` | *"se retiran cuando no hay más"* | — | Repaso PMA/PMS |

Sintaxis: `!` de un lado, `?` del otro, **siempre de a pares** · en una guarda **sólo `?`** · con varios del mismo tipo, **siempre índice** en el `!` · `do ... od` termina cuando todas las guardas son falsas.


---

## ⭐ El esqueleto único: S1 y su variante S2 (un solo molde para memorizar)

> **El S2 con `bool libre` es el más difícil de PMS.** La cátedra aceptó (bicicleta 2025) una forma más fácil: **el S1 con un solo Encargado** que además espera `liberar`. Es el PMS 4c.

```
Process Cliente[id: 1..N] {
  Admin!pedido(id[, dato]);
  [S1]  Empleado[*]?respuesta(r);          -- o el empleado contesta Cliente[id]!
  [S2]  Encargado?usar();
        usarRecurso();
        Encargado!liberar();
}

Process Admin {                                         -- IGUAL en S1 y S2
  cola Fila; int idC[, idE];
  do Cliente[*]?pedido(idC[, dato]) -> push(Fila, idC);
  [] not empty(Fila); [S1] Empleado[*]?listo(idE) -> pop(Fila, idC);
                                                    Empleado[idE]!trabajo(idC);
                      [S2] Encargado?listo()      -> pop(Fila, idC);
                                                    Encargado!siguiente(idC);
  od
}

[S1] Process Empleado[id: 1..K] {                -- K empleados
       while (true) {
         Admin!listo(id);
         Admin?trabajo(idC);
         Cliente[idC]!respuesta(f(...));
       }
     }

[S2] Process Encargado {                         -- uno solo
       for i = 1 to N {
         Admin!listo();
         Admin?siguiente(idC);
         Cliente[idC]!usar();
         Cliente[idC]?liberar();                 -- espero que termine
       }
     }
```

| | S1 (varios servidores) | S2 (recurso de a uno) |
|---|---|---|
| ¿Quién avisa "libre"? | Cada **Empleado[id]**, con su id | **El Encargado**, uno solo: sin `[*]` ni id |
| ¿Qué hace con el cliente? | Lo atiende y le contesta | Le dice `usar` y **espera `liberar`** |
| ¿Cómo sabe que está libre? | El empleado vuelve a `Admin!listo` | **No hay `bool libre`:** mientras alguien usa, el Encargado está bloqueado en `?liberar` |
| Terminación | `while (true)` o `-1` con varios | `for i = 1 to N` (una vez cada uno) |
---

## S1 — Admin con cola + varios servidores

```
Process Cliente[id: 1..N] {
  Admin!pedido(id, dato);
  Empleado[*]?respuesta(r);              // o: el empleado le contesta Cliente[id]!
}

Process Admin {
  cola Fila; int idC, idE;
  do Cliente[*]?pedido(idC, dato) -> push(Fila, (idC, dato));
  [] not empty(Fila); Empleado[*]?listo(idE) -> pop(Fila, (idC, dato));
                                                Empleado[idE]!trabajo(idC, dato);
  od
}

Process Empleado[id: 1..K] {
  while (true) {
    Admin!listo(id);                     // "estoy libre"
    Admin?trabajo(idC, dato);
    Cliente[idC]!respuesta(f(dato));
  }
}
```
- **El Admin no busca empleados:** el libre va solo a avisar (el mozo que pasa por la cocina). El Admin lo escucha sólo si hay algo en la fila: esa es la guarda.
- Si no hay respuesta (impresoras), el cliente sólo hace `Admin!pedido(doc)` y no hace falta el id.
- **Con límite** (P clientes, una vez): el `do` lleva contadores (`llegaron < P`, `atendidos < P`).

## S2 — Recurso único, de a uno: Admin con `bool libre` (Passing the Baton)

```
Process Admin {
  cola Fila; int idP; bool libre = true;
  do Persona[*]?pedido(idP) ->
        if (libre) { libre = false; Persona[idP]!usar(); }
        else push(Fila, idP);
  [] Persona[*]?liberar() ->
        if (empty(Fila)) libre = true;
        else { pop(Fila, idP); Persona[idP]!usar(); }      // se entrega, no se libera
  od
}

Process Persona[id: 1..P] {
  Admin!pedido(id);
  Admin?usar();
  // usa el recurso
  Admin!liberar();
}
```
- **`empty(Fila)` no es "libre":** puede estar la fila vacía y el recurso ocupado.
- **El recurso no se libera: se entrega** al primero de la fila.
- **Alternativa de la cátedra (bicicleta 2025):** Buffer con cola + Entrenador con `for` que pide el siguiente, usa y espera `liberar`.

## S3 — Por id: `for i`, sin cola ni Admin

```
Process Medico {
  for i = 1 to P {
    Paciente[i]?pedido(est);             // exactamente el i, no [*]
    Paciente[i]!resultado(revisar(est));
  }
}

Process Paciente[id: 1..P] {
  Medico!pedido(obtenerEstudio());
  Medico?resultado(res);
}
```
- El orden **ya se sabe de antemano** ⇒ no hace falta cola. `[*]` sería lo contrario: "cualquiera".

## S4 — Prioridad: Admin con una cola por nivel

```
Process Admin {
  cola colaU, colaN; int id;
  do Paciente[*]?urgente(id) -> push(colaU, id);
  [] Paciente[*]?normal(id)  -> push(colaN, id);
  [] (not empty(colaU) or not empty(colaN)); Medico?listo() ->
        if (not empty(colaU)) pop(colaU, id) else pop(colaN, id);
        Medico!atender(id);
  od
}
```
- La prioridad está en el `if` del `pop`: **siempre se mira primero la cola más importante**.

## + Terminación con varios servidores: el `-1`

```
Process Admin {
  cola Fila; int idC, idE, atendidos = 0;
  do Cliente[*]?pedido(idC) -> push(Fila, idC);
  [] atendidos < P; not empty(Fila); Empleado[*]?listo(idE) ->
        pop(Fila, idC); Empleado[idE]!trabajo(idC); atendidos++;
  od
  for i = 1 to K { Empleado[*]?listo(idE); Empleado[idE]!trabajo(-1); }
}

Process Empleado[id: 1..K] {
  int idC; bool seguir = true;
  while (seguir) {
    Admin!listo(id);
    Admin?trabajo(idC);
    if (idC == -1) seguir = false;
    else // atender a idC
  }
}
```
*¿Quién le manda el mensaje que lo hace salir?* El Admin, con el `-1`.
