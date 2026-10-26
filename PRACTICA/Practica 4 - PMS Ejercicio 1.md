# Práctica 4 (PMS) — Ejercicio 1

> **Enunciado.** Suponga que existe un antivirus distribuido que se compone de **R procesos robots Examinadores** y **1 proceso Analizador**. Los procesos Examinadores están buscando continuamente posibles sitios web infectados; **cada vez que encuentran uno avisan la dirección, esperan la respuesta del Analizador**, y luego continúan buscando. El proceso Analizador se encarga de hacer todas las pruebas necesarias con cada uno de los sitios encontrados por los robots para determinar si están o no infectados, para luego **responderle al Examinador**.
> **a)** Analice el problema y defina qué procesos, recursos y comunicaciones serán necesarios/convenientes para resolverlo.
> **b)** Implemente una solución con PMS donde el Analizador **no considere el orden** de los pedidos de los Examinadores.
> **c)** Implemente una solución con PMS donde el Analizador **sí considere el orden** de los pedidos de los Examinadores.
> **d)** Si los Examinadores **no deben esperar** la respuesta del Analizador, ¿cambiaría algo de la solución al inciso b)? ¿Por qué?

> **Cómo se resolvió:** en ping-pong. Antes del ejercicio se aclaró cómo funciona la comunicación guardada (§ 1). Lucas escribió el (b) y el (c) él solo; el (c) salió bien de entrada.

---

## 0. Las consideraciones de PMS

| PMA | PMS |
|---|---|
| Canales declarados, compartidos, FIFO | **No hay canales**: se nombra al proceso. `Destino!port(datos)` / `Origen?port(vars)` |
| `send` no bloquea | **El envío bloquea** hasta que el otro recibe |
| Hay `empty` de canales | **No hay `empty` de canales** (sí de colas locales) |
| Recibir de cualquiera: el canal compartido | **`Origen[*]?port(...)`**, que elige **sin orden** |
| Elegir entre canales: `if` con `empty` | **Comunicación guardada** (§ 1), **sólo con recepciones** |

**La regla que se armó Lucas para reconocer el molde:**
1. *"Sin importar el orden"* → **Servidor y Cliente** (molde S1).
2. *"En orden"*, *"no debe esperar"* o *"maximizar la concurrencia"* → **Admin** que lleva el orden con una cola, **Servidor** y **Cliente** (molde S2).

---

## 1. La comunicación guardada, explicada

```
do Cliente[*]?pedido(idC, R)        -> push(Fila, (idC, R));
[] not empty(Fila); Servidor?listo() -> pop(Fila, (idC, R));
                                        Servidor!trabajo(idC, R);
od
```

### La guarda
Cada opción de un `if` o `do` guardado tiene la forma `guarda -> acciones`:

```
not empty(Fila) ; Servidor?listo()   ->   pop(Fila, ...); Servidor!trabajo(...);
└────────────── guarda ─────────────┘     └──────────── acciones ────────────┘
  condición     ; recepción
```
- **Condición** (opcional, sólo variables locales): `not empty(Fila)` vale porque `Fila` es una **cola local** del Admin.
- **Recepción**: el mensaje que tiene que estar esperando.
- La opción se puede elegir cuando **la condición es verdadera y el mensaje ya está esperando**.

### `if` y `do`
- **`if` guardado:** elige una opción, una vez.
- **`do` guardado:** un `while` + `if` guardado. **Repite hasta que todas las condiciones sean falsas.** Si una opción no tiene condición, nunca falla, y el `do` funciona como un `while(true)`.

| | `if` de PMA (con `empty`) | `if` / `do` guardado de PMS |
|---|---|---|
| Ninguna opción se puede hacer **ahora** | Sale sin hacer nada → busy waiting | **Se queda esperando** |
| Todas las **condiciones** son falsas | Sale sin hacer nada | Sale (y el `do` termina) |

**En PMS esperar es gratis:** no hace falta el truco del `Aviso`.

**La analogía:** un recepcionista con un cuaderno. Viene un cliente → lo anota al final. El empleado avisa que está libre → si hay alguien anotado, le pasa al primero; si el cuaderno está vacío, ni lo escucha.

### Por qué el servidor tiene que **pedir** trabajo
Lo natural sería que el Admin le **mandara** el trabajo al servidor, con un envío en la guarda:
```
[] not empty(Fila); Servidor!trabajo(...) -> ...      // ✗ prohibido en la práctica
```
En la teoría (CSP) existe, pero **la cátedra lo prohíbe: en las guardas sólo van recepciones**. Y mandarlo **fuera** de la guarda tampoco sirve: el `!` bloquea, y si el servidor está ocupado, el Admin queda clavado y deja de recibir clientes.

**La vuelta:** el servidor avisa *"estoy libre"* (`Admin!listo()`), el Admin lo **recibe** en la guarda, y recién ahí **envía como acción**. Ese envío no bloquea, porque el servidor ya está parado en `Admin?trabajo(...)` esperando.

### ¿A quién afecta la regla?
**Sólo al proceso que tiene el `if` o `do` guardado**, el que elige entre varias comunicaciones. Los demás (clientes, servidores) envían y reciben en el orden que necesiten.

> **Regla general:** todo lo que **le llega** al proceso del medio es una **recepción en una guarda**. Si **tiene que mandarle algo a alguien**, ese alguien **se lo pide primero**.

---

## 2. Ítem (a): el análisis

- **Procesos:** `Examinador[1..R]` y `Analizador`. En el (c) y el (d) se suma un `Admin`.
- **Recursos compartidos:** ninguno.
- **Comunicaciones:** el Examinador le manda **su id y el sitio**; el Analizador le responde **al examinador que le escribió**.

---

## 3. Ítem (b): sin orden — molde S1

```
Process Examinador[id: 1..R] {
  texto sitio, res;
  while (true) {
    sitio = buscarSitio();
    Analizador!pedido(id, sitio);
    Analizador?respuesta(res);
  }
}

Process Analizador {
  int idE; texto sitio, res;
  while (true) {
    Examinador[*]?pedido(idE, sitio);     // de cualquiera, sin orden
    res = analizar(sitio);
    Examinador[idE]!respuesta(res);       // al que me escribió
  }
}
```

- **`[*]` recibe de cualquiera** de los que están esperando, al azar. Como el enunciado dice *"no considere el orden"*, alcanza.
- **El id viaja en el mensaje**: en el envío no se puede usar `[*]`, hay que nombrar a uno.

### Lo que hubo que corregir
| Error | Corrección |
|---|---|
| ⚠️ `Examinador[id]!respuesta(res)` en el Analizador | En el Analizador **no existe `id`**: el que escribió está en **`idE`**. Es el error del id de siempre |
| Planteo: `Analizador[*]?...` | En la recepción se nombra a **quien envía**: `Examinador[*]?...` |
| `Analizador!.recibe(...)` | Sin punto: `Analizador!pedido(...)`. Y un port que diga **qué viaja**, no `recibe` |
| Examinador sin `while` | Busca **continuamente** |
| `Processs`, `Analizador()`, `[1..P]` | `Process`, sin paréntesis, `[1..R]` |

---

## 4. Ítem (c): con orden — molde S2

```
Process Admin {
  cola fila; int idE; texto sitio;
  do Examinador[*]?pedido(idE, sitio)       -> push(fila, (idE, sitio));
  [] not empty(fila); Analizador?listo()    -> pop(fila, (idE, sitio));
                                               Analizador!trabajo(idE, sitio);
  od
}

Process Examinador[id: 1..R] {
  texto sitio, res;
  while (true) {
    sitio = buscarSitio();
    Admin!pedido(id, sitio);
    Analizador?respuesta(res);              // la respuesta viene del ANALIZADOR
  }
}

Process Analizador {
  int idE; texto sitio, res;
  while (true) {
    Admin!listo();                          // estoy libre
    Admin?trabajo(idE, sitio);
    res = analizar(sitio);
    Examinador[idE]!respuesta(res);
  }
}
```

- **El Admin sólo ordena:** la respuesta va **directo** del Analizador al Examinador.
- **Por qué respeta el orden:** el `[*]` del Admin toma en cualquier orden **entre los que están esperando a la vez**, pero apenas recibe uno **lo anota**, y el Analizador recibe **siempre el más viejo** de la fila. **El orden lo construye la cola.**
- **El `idE` viaja todo el camino:** Examinador → Admin → `fila` → Analizador → `Examinador[idE]`.

**Detalles que hubo que corregir:** faltaba el **`;` entre la condición y la recepción** de la guarda (`not empty(fila); Analizador?...`) · `text` → `texto` · `fila.push(...)` → `push(fila, ...)`, la notación de la cátedra.

---

## 5. Ítem (d): si el Examinador no espera la respuesta

**Sacar la respuesta no alcanza.** En el (b), sin las dos líneas de respuesta:
```
Process Examinador[id: 1..R] {
  while (true) {
    sitio = buscarSitio();
    Analizador!pedido(id, sitio);       // ← bloquea igual
  }
}
```
**En PMS el `!` bloquea hasta que el otro recibe.** El Analizador recibe recién cuando termina de analizar el sitio anterior. Mientras tanto, el Examinador queda parado en el envío, **esperando a que el Analizador se desocupe**, y con muchos examinadores puede esperar varios análisis ajenos.

> **Respuesta:** **sí cambia.** En PMA alcanzaría con sacar la respuesta, porque el `send` no bloquea. En PMS el envío es sincrónico: para que el Examinador mande y siga hace falta **un Admin que funcione como buffer**, con la opción de recibir examinadores **sin condición** (siempre dispuesto a recibir). La solución queda como la del (c), **sin la respuesta**.

**El Admin aparece por dos motivos distintos:**
| Ítem | Para qué |
|---|---|
| (c) | **Ordenar** los pedidos |
| (d) | **No bloquear al emisor** (buffer) |

---

## 6. Chequeos de sanidad

- **Cada `!` tiene su `?` del otro lado**, en el orden correcto: `Admin!listo()` ↔ `Analizador?listo()`; `Analizador!trabajo` ↔ `Admin?trabajo`. ✅
- **El `idE` viaja de punta a punta.** ✅
- **En las guardas sólo hay recepciones**, y las condiciones sólo miran variables locales (`fila`). ✅
- **Con R = 1**, el (b) y el (c) se comportan igual. ✅
