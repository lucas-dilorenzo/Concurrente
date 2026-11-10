# Práctica 5 (Ada) — Ejercicio 7

> **Enunciado.** Se debe calcular el **valor promedio** de un vector de **1 millón** de números enteros que se encuentra **distribuido entre 10 procesos Worker** (es decir, cada Worker tiene un vector de 100 mil números). Para ello, existe un **Coordinador** que determina el momento en que se debe realizar el cálculo de este promedio y que, además, **se queda con el resultado**.
> *Nota: maximizar la concurrencia; este cálculo se hace **una sola vez**.*

> **Cómo se resolvió:** en ping-pong. El planteo fue de Lucas, salvo **quién llama a quién**: eligió que el Coordinador llame a cada Worker, que es la trampa del ejercicio. El Worker le salió bien de entrada. Para el Coordinador copió el R9 del repaso sin adaptarlo (`Valor` con número, `for 1..10`), y en la segunda versión quedaron detalles de declaración.

---

## 0. El planteo — molde R9

Es **el WordCount del parcial de 2024** (alternativa 2) con otros nombres. Allá el admin mandaba una palabra; **acá no manda ningún dato: sólo da la señal de arranque.**

| Pregunta | Respuesta |
|---|---|
| Tareas | `Coordinador` (`task`) · `Worker` (`task type`, arreglo de 10) |
| ¿Qué devuelve cada Worker? | **La suma** de sus 100 mil números, no el promedio. El Coordinador suma las 10 sumas parciales y divide por 1.000.000 |
| ¿Quién llama a quién? | **Los Workers al Coordinador**, para arrancar y para entregar |
| ¿Cómo se entera el Worker de que arranca? | Llama a `Coordinador.Comenzar` y espera en la cola **hasta que el Coordinador lo acepta** (la puerta del micro del R8) |

### La trampa: el Coordinador recorriendo a los Workers
Hay **dos momentos**, y los decide gente distinta:

| Momento | ¿Quién decide? |
|---|---|
| **Arrancar** | El Coordinador: *"ya, empiecen"* |
| **Entregar el resultado** | **Cada Worker, cuando termina** |

Si el Coordinador hace `for i in 1..10 loop arrWorkers(i).Resultado(parcial); end loop;`, **queda clavado esperando al Worker 1**, aunque los otros 9 ya hayan terminado. Si **los Workers llaman**, el Coordinador hace `accept Resultado` y **atiende al primero que termine**.

> **Un examen:** el profesor decide cuándo empieza (*"pueden dar vuelta la hoja"*), pero no pasa banco por banco a retirar las hojas: **cada alumno la lleva cuando termina.** Si tuviera que retirar primero la del banco 1, todos esperarían al más lento.

### ¿Quién controla los 100 mil números?
**El Coordinador no los toca nunca.** Cada Worker recorre **su** vector. Al Coordinador le llegan 10 números; sólo necesita saber el total (1.000.000) para dividir. **Esa es la gracia del R9:** lo pesado lo hacen los 10 en paralelo.

---

## 1. La solución

```ada
Procedure Calcular is
   task Coordinador is
      entry Comenzar;
      entry Resultado (suma: IN integer);
   end Coordinador;

   task type Worker;
   arrWorkers: array (1..10) of Worker;

   task body Worker is
      v: vector;                -- sus 100 mil números, ya cargados
      suma: integer := 0;
   begin
      Coordinador.Comenzar;
      for i in 1..100000 loop
         suma := suma + v(i);
      end loop;
      Coordinador.Resultado(suma);
   end Worker;

   task body Coordinador is
      total: integer := 0;
      promedio: float;
   begin
      for i in 1..2*10 loop
         select
            accept Comenzar;
         or
            accept Resultado (suma: IN integer) do
               total := total + suma;
            end Resultado;
         end select;
      end loop;
      promedio := total / 1000000;
   end Coordinador;

begin
   null;
end Calcular;
```

### Por qué `for 1..2*N`
Cada Worker hace **2 llamadas** (`Comenzar` y `Resultado`) y son 10, así que llegan **20**. Con 10 vueltas el Coordinador termina a la mitad y deja Workers trabados en la cola.

### Por qué un `select` y no dos `for`
Con `for` de `Comenzar` y después `for` de `Resultado` también funciona, pero un Worker que ya terminó tendría que esperar a que el Coordinador acepte los 10 `Comenzar`. Acá la diferencia es mínima (los `Comenzar` son instantáneos), pero **la cátedra muestra el `select` como la mejor solución** (WordCount 2024, alternativa 2).

### `accept Comenzar;` pelado
Es **sólo una señal**: no tiene parámetros ni `do` (como el `accept Usar;` del R6).

---

## 2. Lo que hubo que corregir

| Error | Corrección |
|---|---|
| ⚠️ El Coordinador llama a cada Worker (planteo) | **Los Workers llaman**: el Coordinador atiende al que termina primero |
| ⚠️ Coordinador copiado del R9: `accept Valor (num: OUT integer)` | Acá no se manda ningún dato: `accept Comenzar;` |
| ⚠️ `for i in 1..10` | **`1..2*N`**: dos llamadas por Worker |
| `entry Resultado(suma);` | En la declaración van **modo y tipo**, iguales al `accept`: `(suma: IN integer)` |
| `numero: integer := elegirNumero;` | Sobraba (resto del R9 copiado) |
| Faltaba el promedio | `promedio := total / 1000000;` después del `for` |
| `v, suma: integer := 0` (Worker) | `v` es un **vector**, no un integer · falta `;` |
| La suma promediaba | Cada Worker devuelve la **suma**; se divide una sola vez al final |

---

## 3. Chequeos de sanidad

- **Llamadas = accepts:** 10 Workers × 2 llamadas = 20 = vueltas del `for`. ✅
- **Si el Worker 3 es lento**, el `select` sigue aceptando los `Resultado` de los demás. ✅
- **El Coordinador no recorre ningún vector:** sólo suma 10 números y divide. ✅
- **Cada entry declarado tiene su `accept`**, con los mismos parámetros. ✅
- **Todas las tareas terminan:** los Workers después de entregar, el Coordinador después de las 20 vueltas. ✅

---

## 4. Variante con `loop` (parciales 2023 sucursales y 2025 recomendaciones)

*"Cuando termina de procesar un artículo, comienza con el siguiente."* Todo se repite para siempre, y el Coordinador le **manda** el dato a buscar.

### El bug si sólo se agrega el `loop`
**El `for` de 2N vueltas es una planilla con 2N casilleros**, y el `select` anota a cualquiera que esté en la cola sin mirar quién es. Un Worker rápido termina, entrega y **vuelve a llamar a `Comenzar` en la misma ronda**: ocupa el casillero de otro, la ronda se cierra sin el más lento, y su resultado **cae en el total del artículo siguiente**. No es un problema de eficiencia: **el resultado da mal.**

En el ej 7 no pasaba porque cada Worker llamaba exactamente 2 veces en su vida.

### La solución: barrera `Siguiente` (R8) al cierre de cada ronda
**Afuera** del `for` de 2N, un `for` de **N** `accept Siguiente`. El Worker rápido queda en esa cola hasta que entregaron todos.

```ada
Procedure Sucursales is
   task Coordinador is
      entry Comenzar (a: OUT integer);
      entry Resultado (c: IN integer);
      entry Siguiente;
   end Coordinador;

   task type Worker;
   arrWorkers: array (1..N) of Worker;

   task body Worker is
      articulo, cant: integer;
   begin
      loop
         Coordinador.Comenzar(articulo);
         cant := ObtenerVentas(articulo);
         Coordinador.Resultado(cant);
         Coordinador.Siguiente;
      end loop;
   end Worker;

   task body Coordinador is
      articulo, total: integer;
   begin
      loop
         articulo := generarArticulo;
         total := 0;
         for i in 1..2*N loop
            select
               accept Comenzar (a: OUT integer) do
                  a := articulo;
               end Comenzar;
            or
               when (Comenzar'count = 0) =>
                  accept Resultado (c: IN integer) do
                     total := total + c;
                  end Resultado;
            end select;
         end loop;
         for i in 1..N loop
            accept Siguiente;
         end loop;
      end loop;
   end Coordinador;

begin
   null;
end Sucursales;
```

### `when (Comenzar'count = 0)` en `Resultado`
No es para la corrección (eso lo resuelve la barrera): es para **maximizar la concurrencia**. Si uno espera para arrancar y otro para entregar, primero se despacha al que va a trabajar. Es `'count` usado para dar prioridad, por la tarea dueña del entry (R3).

| | Ej 7 (una vez) | Variante con `loop` |
|---|---|---|
| Worker | 2 llamadas en su vida | `loop`, **3** llamadas por ronda |
| `Comenzar` | Señal pelada | Devuelve el dato (`OUT`) |
| Barrera `Siguiente` | No hace falta | **Obligatoria** |
| `when Comenzar'count = 0` | Opcional | La cátedra lo pone |
