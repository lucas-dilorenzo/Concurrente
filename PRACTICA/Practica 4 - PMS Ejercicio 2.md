# Práctica 4 (PMS) — Ejercicio 2

> **Enunciado.** En un laboratorio de genética veterinaria hay **3 empleados**. El **primero** de ellos continuamente prepara las muestras de ADN; cada vez que termina, **se la envía al segundo empleado y vuelve a su trabajo**. El **segundo** empleado toma cada muestra de ADN preparada, arma el set de análisis que se deben realizar con ella y **espera el resultado para archivarlo**. Por último, el **tercer** empleado se encarga de realizar el análisis y **devolverle el resultado al segundo** empleado.

> **Cómo se resolvió:** Lucas hizo el planteo (bien el tramo 2º–3º; en el 1º–2º creyó que alcanzaba con mandar directo). La solución la escribí yo a pedido suyo.

---

## 1. El planteo: una cadena de tres

| Tramo | Frase | Molde |
|---|---|---|
| 1º → 2º | *"se la envía y **vuelve a su trabajo**"* | **S2 como buffer**: hace falta un **Admin** |
| 2º → 3º → 2º | *"**espera el resultado**"* / *"devolverle el resultado"* | **S1**: pedido y respuesta directos |

**2º–3º:** ni siquiera hace falta `[*]`, porque hay **un solo** emisor y un solo receptor.

### Por qué no alcanza con que el 1º le mande directo al 2º

1. El **2º** le manda un set al **3º** y queda **esperando el resultado**.
2. Mientras tanto, el **1º** termina otra muestra y hace `Empleado2!muestra(m)`.
3. En PMS el `!` **bloquea hasta que el otro recibe**, y el 2º no está recibiendo muestras: está esperando el análisis.
4. **El 1º queda parado** hasta que el 2º archive. No *"vuelve a su trabajo"*.

> Es el **ítem (d) del ejercicio 1**: en PMS, no esperar una respuesta **no alcanza para no esperar**, porque el envío bloquea si el otro está ocupado. Hace falta un **Admin que reciba enseguida** (buffer).
>
> Es el **ejercicio 2 de la explicación de PMS** (Testeo y Mantenimiento), molde S2 del repaso.

**Son 4 procesos:** `Empleado1`, `Admin`, `Empleado2` y `Empleado3`.

---

## 2. Solución

```
Process Empleado1 {
  texto m;
  while (true) {
    m = prepararMuestra();
    Admin!muestra(m);                    // el Admin recibe enseguida: sigue trabajando
  }
}

Process Admin {
  cola buffer; texto m;
  do Empleado1?muestra(m)                 -> push(buffer, m);
  [] not empty(buffer); Empleado2?pedido() -> pop(buffer, m);
                                             Empleado2!muestra(m);
  od
}

Process Empleado2 {
  texto m, set, res;
  while (true) {
    Admin!pedido();                      // estoy libre: dame una muestra
    Admin?muestra(m);
    set = armarSet(m);
    Empleado3!analisis(set);
    Empleado3?resultado(res);            // espera el resultado
    archivar(res);
  }
}

Process Empleado3 {
  texto set, res;
  while (true) {
    Empleado2?analisis(set);
    res = analizar(set);
    Empleado2!resultado(res);
  }
}
```

| Pieza | Por qué |
|---|---|
| **La opción del Empleado1 no tiene condición** | El Admin siempre está dispuesto a recibir muestras: el `!` del 1º se completa enseguida |
| **`not empty(buffer)` en la otra opción** | Sólo atiende el pedido del 2º si tiene una muestra para darle. Si no, el 2º queda esperando en `Admin?muestra` |
| **El 2º pide** (`Admin!pedido()`) | Un envío no puede ir en una guarda: el Admin no puede "empujarle" la muestra, tiene que esperar a que se la pidan |
| **2º y 3º hablan directo** | Uno a uno, con respuesta: S1. No hace falta orden ni buffer |

**No hace falta mandar ids:** cada comunicación es entre procesos **únicos** y con nombre (`Empleado1`, `Empleado2`, `Empleado3`), así que siempre se sabe a quién responder.

---

## 3. Chequeos de sanidad

- **Cada `!` tiene su `?`:** `Admin!muestra` ↔ `Empleado1?muestra` · `Admin!pedido` ↔ `Empleado2?pedido` · `Empleado2!muestra` ↔ `Admin?muestra` · `Empleado3!analisis` ↔ `Empleado2?analisis` · `Empleado2!resultado` ↔ `Empleado3?resultado`. ✅
- **El 1º nunca espera al 2º:** sólo espera que el Admin lo reciba, y el Admin siempre puede. ✅
- **Sin buffer, el sistema funcionaría**, pero el 1º trabajaría al ritmo del análisis: no cumple *"vuelve a su trabajo"*. ✅
