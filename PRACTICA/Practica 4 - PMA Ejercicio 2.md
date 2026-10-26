# Práctica 4 (PMA) — Ejercicio 2

> **Enunciado.** Se desea modelar el funcionamiento de un banco en el cual existen **5 cajas** para realizar pagos. Existen **P clientes** que desean hacer un pago. Para esto, cada uno **selecciona la caja donde hay menos personas esperando**; una vez seleccionada, espera a ser atendido. En cada caja, los clientes son atendidos **por orden de llegada** por los cajeros. Luego del pago, se les entrega un **comprobante**. *Nota: maximizar la concurrencia.*

> **Cómo se resolvió:** en ping-pong. Lucas hizo el planteo de procesos y canales solo; el Coordinador se armó por pasos, con un esqueleto que él completó.

---

## 0. Es un ejercicio conocido

Es la **planta verificadora del parcial de 2022** (y la terminal de micros de la Práctica 2) pasada a mensajes: un coordinador lleva cuántos hay en cada puesto y asigna el que tiene menos.

| Con semáforos | Con PMA |
|---|---|
| `int asignados[7]` compartido + su mutex | `int cant[1..5]` **local del Coordinador**. Sin mutex: sólo él lo toca |
| Cola + semáforo por estación | **Un canal por caja**, `Fila[1..5]`, que ya es FIFO |
| `sem verificado[150]` + `int resultado[150]` | **Un canal por cliente**, `Comprobante[1..N]`, con el dato adentro |

---

## 1. El planteo

**Procesos:** `Cliente[1..N]`, **un** `Coordinador` y `Caja[1..5]`.

**¿Quién sabe cuánta gente hay en cada caja?** No hay variables compartidas, así que el arreglo vive en **un proceso**: el Coordinador.

**¿Cuándo baja el contador?** Cuando la caja **termina** de atender y le avisa al Coordinador. El contador cuenta a los que están **en** la caja, esperando o siendo atendidos.

**Canales:**

| Canal | Dato | ¿Uno o arreglo? |
|---|---|---|
| `Llegada` (Cliente → Coord) | `idC` | Uno: todos los clientes mandan por ahí |
| `Termine` (Caja → Coord) | `idCaja` | Uno: el Coordinador necesita saber **qué** caja bajar |
| `Fila` (Coord → Caja) | `idC` | **Arreglo `[1..5]`**: el número de caja **es el índice** |
| `Comprobante` (Caja → Cliente) | el comprobante | **Arreglo `[1..N]`**: el id del cliente **es el índice** |
| `Aviso` (todos → Coord) | un entero cualquiera | Uno (ver § 2) |

> **La regla:** cuando un mensaje es **para alguien en particular**, ese alguien **es el índice del canal**, no un dato adentro del mensaje. En PMA, un dato adentro no impide que otro lo reciba.

**Un acierto del planteo:** el Coordinador le pasa el cliente **directo a la caja**. El cliente no necesita saber en qué caja está: manda su llegada y espera su comprobante.

---

## 2. La parte difícil: el Coordinador escucha a dos

El Coordinador recibe por **dos** canales: las llegadas de los clientes y los "terminé" de las cajas.

**Problema 1: si hace `receive Llegada` directo, queda bloqueado ahí.** Mientras no lleguen clientes, los "terminé" se acumulan sin procesar. Cuando llega el siguiente, **elige la caja con contadores desactualizados**.

**Problema 2: un `if` no determinístico con `empty` en los dos tiene dos riesgos.**
- **Busy waiting:** con los dos canales vacíos, el `if` no hace nada y el `while` vuelve a preguntar sin parar.
- **Elige al azar:** si hay una llegada y un "terminé" pendientes, puede asignar primero con el contador viejo.

**La solución es el molde A6 (prioridad):**
- **primero los "terminé"**, así al asignar los contadores ya están al día: la opción de las llegadas lleva `and empty(Termine)`;
- **un canal `Aviso`** para que el Coordinador **duerma** en `receive Aviso` en vez de dar vueltas.

### Las dos reglas del `Aviso`

`Aviso` **no** significa "llegó un cliente". Significa **"tenés un mensaje nuevo para procesar"**, venga de quien venga. Funciona como un semáforo contador:

> **Invariante:** mensajes en `Aviso` = mensajes pendientes en `Llegada` + en `Termine`.

**1. Avisan todos los que le mandan algo al Coordinador**, incluidas las cajas. Si la caja no avisara:
1. La caja 2 manda `Termine(2)` sin aviso. El Coordinador sigue dormido.
2. Llega el cliente 7: `Llegada(7)` + `Aviso`. Hay **1 aviso y 2 mensajes**.
3. El Coordinador despierta y, por la prioridad, procesa el `Termine`. La llegada del 7 queda **sin aviso**.
4. Vuelve a dormir. **El cliente 7 no es atendido hasta que llegue otro.** Si era el último, espera para siempre.

**2. El aviso va DESPUÉS del mensaje principal.** Si fuera antes, el Coordinador podría despertar con los dos canales todavía vacíos, salir del `if` sin hacer nada y **consumir el aviso**. Cuando el mensaje llegara, quedaría sin aviso.

---

## 3. Solución

```
chan Llegada(int);                 // idC
chan Termine(int);                 // idCaja que terminó
chan Aviso(int);                   // "tenés un mensaje nuevo"
chan Fila[1..5](int);              // una por caja: idC
chan Comprobante[1..N](texto);     // una por cliente

Process Cliente[id: 1..N] {
  texto comp;
  send Llegada(id);
  send Aviso(1);                   // DESPUÉS del mensaje
  receive Comprobante[id](comp);
}

Process Coordinador {
  int cant[1..5] = ([5] 0);
  int ok, idC, idCaja, minID;
  while (true) {
    receive Aviso(ok);             // duermo hasta que haya algo
    if (not empty(Termine)) ->
          receive Termine(idCaja);
          cant[idCaja]--;
    [] (not empty(Llegada)) and (empty(Termine)) ->
          receive Llegada(idC);
          minID = cajaConMenos(cant);
          cant[minID]++;
          send Fila[minID](idC);
    fi
  }
}

Process Caja[id: 1..5] {
  int idC; texto comp;
  while (true) {
    receive Fila[id](idC);         // SÓLO mi fila, por orden de llegada
    comp = atenderPago(idC);
    send Comprobante[idC](comp);
    send Termine(id);
    send Aviso(1);                 // DESPUÉS del mensaje
  }
}
```

- **El orden de llegada a cada caja sale gratis:** `Fila[id]` es FIFO.
- **Se maximiza la concurrencia:** las 5 cajas atienden en paralelo y el Coordinador sólo reparte; nunca espera a que termine una atención.

---

## 4. Los errores que se cometieron en el camino

### ⚠️ Un solo canal de derivación con el número de caja adentro
```
send Derivacion(minID, idC);       // ✗ Coordinador
receive Derivacion(id, idC);       // ✗ Caja
```
**El `receive` no filtra por valor.** No significa "recibí sólo los mensajes para mi `id`": significa "sacá el primer mensaje y guardalo en `id` e `idC`". Entonces:
- **cualquier caja se lleva cualquier cliente**, y se pierde la elección del Coordinador;
- **la caja pisa su propio `id`** con el número que venía en el mensaje.

Lo mismo con el comprobante: el `idC` **es el índice** de `Comprobante`, no un dato adentro.

### Detalles
- El canal declarado como `Avsio` y usado como `Aviso`.
- `minID` sin declarar, y `calcularMin(min, cant)` con un parámetro de más: alcanza con `minID = cajaConMenos(cant)`.
- `Comprobante[N]` con clientes `1..N` → `[1..N]`.
- Comentarios equivocados: el `receive` del cliente espera **el comprobante**, no la derivación; `Aviso` es "mensaje nuevo", no "cliente esperando".

---

## 5. Chequeos de sanidad

- **Seguir el id del cliente con el dedo:** `Cliente` → `Llegada` → `Coordinador` → `Fila[minID]` → `Caja` → `Comprobante[idC]` → `Cliente`. ✅
- **El id de la caja:** `Caja` → `Termine` → `Coordinador` → `cant[idCaja]--`. ✅
- **Cada mensaje a `Llegada` o `Termine` tiene su `Aviso`, mandado después.** ✅
- **Cada `cant[c]++` tiene su `cant[c]--`:** el `++` al derivar, el `--` cuando esa caja avisa que terminó. ✅
- **Con 1 caja** colapsa al ejercicio 1 (a), con un intermediario de más. ✅
