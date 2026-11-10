# Soluciones — Prácticas 1, 2, 3, 4 y 5

> Sólo enunciado y código. El razonamiento, las trazas y los errores que se penalizan están en los `.md` individuales de cada ejercicio.

**Índice**
- [Práctica 1 — Variables compartidas](#práctica-1--variables-compartidas): [1](#p1--1) · [2](#p1--2) · [3](#p1--3) · [4](#p1--4)
- [Práctica 2 — Semáforos](#práctica-2--semáforos): [1](#p2--1) · [2](#p2--2) · [3](#p2--3) · [4](#p2--4) · [5](#p2--5) · [6](#p2--6) · [7](#p2--7) · [8](#p2--8) · [9](#p2--9) · [10](#p2--10) · [11](#p2--11) · [12](#p2--12)
- [Práctica 3 — Monitores](#práctica-3--monitores): [4](#p3--4) · [5](#p3--5)
- [Práctica 4 — Pasaje de Mensajes](#práctica-4--pasaje-de-mensajes): PMA [1](#p4-pma--1) · [2](#p4-pma--2) · [3](#p4-pma--3) · [4](#p4-pma--4) · PMS [1](#p4-pms--1) · [2](#p4-pms--2) · [3](#p4-pms--3) · [4](#p4-pms--4) · [5](#p4-pms--5)
- [Práctica 5 — Rendezvous (Ada)](#práctica-5--rendezvous-ada): [1](#p5--1) · [2](#p5--2) · [7](#p5--7)

---

# Práctica 1 — Variables compartidas

## P1 — 1
*¿Puede `x` terminar en 56, 22 o 23?*

**Las tres son verdaderas.**

- **56** sale con granularidad gruesa (sentencias completas).
- **22 y 23** sólo con **granularidad fina**: `x := (x*3) + (x*2) + 1` tiene **dos referencias críticas**, viola la propiedad **A lo sumo una vez (ASV)**, y por eso su ejecución no es atómica.

---

## P1 — 2
*Contar cuántas veces aparece `N` en un arreglo de longitud `M`, con P procesos.*

```
// PRE: M > 0, P > 0, P <= M, M mod P == 0
//      A[0..M-1] inicializado antes de crear los procesos
//      A y N NO se modifican durante la ejecución
//      total == 0 antes de crear los procesos

int A[M];             // compartido, sólo lectura
int N;                // compartido, sólo lectura
int total = 0;        // compartido, escrito por los P

Process Buscador [id: 0..P-1]
{ int parcial = 0;                       // LOCAL
  int desde = id * (M / P);
  int hasta = desde + (M / P) - 1;

  for j = desde .. hasta
     { if (A[j] == N) parcial = parcial + 1; }    // local -> sin <>

  < total = total + parcial >;                     // una sola SC por proceso
}
```

**Variante sin la precondición** — reparto intercalado: `for j = id to M-1 step P`.

---

## P1 — 3
*Productor/Consumidor con buffer de tamaño N. (a) ¿funciona el código dado? (b) generalizar a P productores y C consumidores.*

**(a) El código dado NO funciona.** `cant++` ocurre **antes** de escribir el buffer, así que `cant` miente: el consumidor puede leer un slot no escrito (y simétricamente, el productor puede pisar un slot no leído).

**Solución (sirve para (a) y (b)):**

```
// PRE: N > 0, P > 0, C > 0
//      cant == 0, pri_vacia == 0, pri_ocupada == 0 antes de crear los procesos

int cant = 0;   int pri_ocupada = 0;   int pri_vacia = 0;   int buffer[N];

Process Productor [i: 1..P]                Process Consumidor [j: 1..C]
{ int elemento;              // LOCAL     { int elemento;              // LOCAL
  while (true)                              while (true)
   { produce elemento                        { <await (cant > 0);
     <await (cant < N);                          elemento = buffer[pri_ocupada];
        buffer[pri_vacia] = elemento;            pri_ocupada = (pri_ocupada + 1) mod N;
        pri_vacia = (pri_vacia + 1) mod N;       cant--
        cant++                                >
     >                                        consume elemento
   }                                        }
}                                         }
```

---

## P1 — 4
*5 instancias de un recurso en una cola: sacar, usar, devolver.*

```
// PRE: la cola contiene las 5 instancias antes de crear los procesos
//      N > 0;  cada proceso devuelve toda instancia que saca

cola recursos;      // compartida, inicializada con las 5 instancias

Process Usuario [i: 1..N]
{ int rec;                                                  // LOCAL
  while (true)
   { <await (not vacia(recursos)); rec = pop(recursos)>     // condicional
     usar(rec);                                             // AFUERA del <>
     <push(recursos, rec)>                                  // incondicional
   }
}
```

El `push` no lleva guarda: por el invariante `en_cola + en_uso = 5`, quien devuelve siempre tiene lugar.

---

# Práctica 2 — Semáforos

> **Reglas de la práctica:** semáforos declarados e inicializados siempre · prohibido consultar o setear el valor de un semáforo · evitar busy waiting · el tiempo va con `delay`.

## P2 — 1
*N personas pasan por un detector de metales.*

**(a) Análisis.** N procesos `Persona` (un solo tipo). El detector es un **recurso pasivo**, no un proceso. Un recurso con 1 unidad en (b) y 3 en (c). Una sincronización: competencia por unidades del recurso.

**(b) Un detector**

```
sem detector = 1;

Process Persona[id: 1..N]
{ P(detector);
  delay(tiempo);        // pasa por el detector
  V(detector);
}
```

**(c) Tres detectores** — sólo cambia el valor inicial:

```
sem detectores = 3;

Process Persona[id: 1..N]
{ P(detectores);
  delay(tiempo);
  V(detectores);
}
```

*Si hiciera falta saber **cuál** detector (el enunciado no lo pide):*

```
cola detectores;                      // ids 1, 2, 3
sem disponibles = 3, mutex = 1;

Process Persona[id: 1..N]
{ int d;
  P(disponibles);
  P(mutex); pop(detectores, d); V(mutex);
  delay(tiempo);                              // usa el detector d
  P(mutex); push(detectores, d); V(mutex);
  V(disponibles);
}
```

**(d) Cantidad aleatoria de pasadas**

```
sem detectores = 3;

Process Persona[id: 1..N]
{ int veces = random(1..K);        // LOCAL
  for j = 1 to veces
     { P(detectores);
       delay(tiempo);
       V(detectores);
     }
}
```

---

## P2 — 2
*4 procesos, historial de N fallos con `.ID` y `.gravedad` (0..3).*

**(a) Imprimir los ID de los críticos**

```
// PRE: N mod 4 == 0;  historial cargado antes de crear los procesos

fallo historial[N];        // compartido, SÓLO LECTURA -> sin protección
sem impresion = 1;         // la pantalla es un recurso compartido

Process Chequeador[id: 0..3]
{ int desde = id * (N/4);
  int hasta = desde + (N/4) - 1;
  for j = desde to hasta
     { if (historial[j].gravedad == 3)
          { P(impresion);
            mostrar(historial[j].ID);
            V(impresion);
          }
     }
}
```

**(b) Contar por nivel de gravedad**

```
// PRE: N mod 4 == 0;  gravedades inicializado en 0 antes de crear los procesos

fallo historial[N];                   // compartido, sólo lectura
int gravedades[0:3] = ([4] 0);        // compartido, escrito por los 4
sem acumulador = 1;

Process Chequeador[id: 0..3]
{ int parcial[0:3] = ([4] 0);          // LOCAL
  int desde = id * (N/4);
  int hasta = desde + (N/4) - 1;

  for j = desde to hasta
     parcial[historial[j].gravedad]++;  // local -> sin semáforo

  P(acumulador);                        // UNA sola SC por proceso
  for k = 0 to 3
     gravedades[k] = gravedades[k] + parcial[k];
  V(acumulador);
}
```

**(c) Cada proceso cuenta un nivel** — cambia el reparto y **desaparecen los semáforos**:

```
// PRE: historial cargado antes de crear los procesos
//      (NO hace falta N mod 4 == 0: nadie corta el arreglo)

fallo historial[N];               // compartido, sólo lectura
int gravedades[0:3];              // cada casillero tiene UN SOLO dueño

Process Chequeador[id: 0..3]
{ int total = 0;                  // LOCAL
  for j = 0 to N-1
     { if (historial[j].gravedad == id) total++; }
  gravedades[id] = total;         // asignación directa, no +=
}
```

---

## P2 — 3
*5 instancias de un recurso en una cola, P procesos.*

```
// PRE: la cola contiene las 5 instancias antes de crear los procesos;  P > 0

cola recursos;                    // contiene las 5 instancias
sem disponible = 5;               // contador de recursos
sem mutex = 1;                    // protege la cola

Process Usuario[id: 1..P]
{ int rec;                                          // LOCAL
  P(disponible);                                    // espero a que haya alguna libre
  P(mutex);  pop(recursos, rec);  V(mutex);         // SC 1: la saco
  delay(tiempo);                                    // USO — afuera de toda SC
  P(mutex);  push(recursos, rec);  V(mutex);        // SC 2: la devuelvo
  V(disponible);                                    // aviso que quedó libre
}
```

**Dos secciones críticas, no una**, con el uso en el medio. `P(disponible)` va **antes** que `P(mutex)`: al revés es deadlock.

---

## P2 — 4
*BD con máximo 6 usuarios, máximo 4 de prioridad alta y máximo 5 de prioridad baja. ¿Es adecuada la solución dada?*

**No es la más adecuada.** Es **correcta** (cumple los tres límites, no tiene deadlock) pero provoca **demora innecesaria**: un proceso bloqueado en `P(alta)` ya consumió una unidad de `total`, o sea que **reserva un lugar en la BD sin usarlo**.

*Contraejemplo:* 4 altas usando la BD (`alta=0`, `total=2`). Llegan 2 altas más: pasan `P(total)` (`total=0`) y se bloquean en `P(alta)`. Llega un baja: se bloquea en `P(total)`, aunque las tres restricciones lo permitían (5 ≤ 6, altas 4 ≤ 4, bajas 1 ≤ 5). La BD opera a 4 de sus 6 lugares.

**Mejora — invertir el orden de los dos `P`:**

```
Var  total: sem := 6;   alta: sem := 4;   baja: sem := 5;

Process Usuario-Alta [I: 1..L]::          Process Usuario-Baja [I: 1..K]::
 { P(alta);                                { P(baja);
   P(total);                                 P(total);
   //usa la BD                               //usa la BD
   V(total);                                 V(total);
   V(alta);                                  V(baja);
 }                                         }
```

Así el que no consigue cupo de su tipo se bloquea **sin retener un lugar de la BD**.

---

## P2 — 5
*Sala de N contenedores (1 paquete cada uno). Preparadores dejan paquetes, Entregadores los retiran.*

**Buffer acotado.** Dos esperas distintas ⇒ **dos semáforos contadores**, usados cruzados: cada bando hace `P` sobre lo que consume y `V` sobre lo que produce. Invariante: `vacios + llenos = N`.

```
sem vacios = N;      // contenedores libres   → lo espera el Preparador
sem llenos = 0;      // contenedores llenos   → lo espera el Entregador
```

### a) 1 Preparador, 1 Entregador — sin mutex

```
// PRE: N > 0;  la sala arranca vacía

paquete sala[0..N-1];
sem vacios = N;
sem llenos = 0;

Process Preparador                      Process Entregador
{ paquete p;                            { paquete p;
  int pos = 0;         // LOCAL           int pos = 0;         // LOCAL
  while (true) {                          while (true) {
    p = Preparar();    // delay(tp)         P(llenos);
    P(vacios);                              p = sala[pos];
    sala[pos] = p;                          pos = (pos + 1) mod N;
    pos = (pos + 1) mod N;                  V(vacios);
    V(llenos);                              Entregar(p);     // delay(te)
  }                                       }
}                                       }
```

**No hace falta mutex:** cada `pos` es local (un solo dueño), y los semáforos ya garantizan que ambos nunca tocan el mismo contenedor a la vez (cada acceso está precedido por el `V` del otro sobre esa celda). `Preparar()` y `Entregar()` van **afuera** del tramo que ocupa un contenedor.

### b) P Preparadores, 1 Entregador

`pos_prep` pasa a ser compartido ⇒ `mutexP`. **El Entregador no cambia.**

```
// PRE: N > 0, P > 0;  la sala arranca vacía
int pos_prep = 0;   sem mutexP = 1;      // + lo de (a)

Process Preparador[id: 1..P]
{ paquete p;
  while (true) {
    p = Preparar();                        // delay(tp) — AFUERA, en paralelo
    P(vacios);                             // contador PRIMERO
    P(mutexP);                             // candado DESPUÉS
      sala[pos_prep] = p;                  // la escritura va ADENTRO
      pos_prep = (pos_prep + 1) mod N;
    V(mutexP);
    V(llenos);
  }
}
```

**La escritura va adentro del mutex, no sólo el índice.** Si sólo reservás el índice adentro y escribís afuera, un Preparador puede hacer `V(llenos)` mientras otro todavía no escribió su celda anterior, y el Entregador —que recorre en orden circular— lee un contenedor vacío.

### c) 1 Preparador, E Entregadores

Espejo de (b): `pos_entr` compartido ⇒ `mutexE`, con la **lectura adentro**. **El Preparador no cambia.**

```
// PRE: N > 0, E > 0;  la sala arranca vacía
int pos_entr = 0;   sem mutexE = 1;      // + lo de (a)

Process Entregador[id: 1..E]
{ paquete p;                               // LOCAL
  while (true) {
    P(llenos);
    P(mutexE);
      p = sala[pos_entr];                  // la lectura va ADENTRO
      pos_entr = (pos_entr + 1) mod N;
    V(mutexE);
    V(vacios);
    Entregar(p);                           // delay(te) — AFUERA del mutex
  }
}
```

Con la lectura afuera, el Preparador puede pisar un contenedor que un Entregador todavía no leyó. Y `Entregar()` afuera del mutex, o los E entregadores trabajan como uno solo.

### d) P Preparadores, E Entregadores — dos mutex

```
// PRE: N > 0, P > 0, E > 0;  la sala arranca vacía

paquete sala[0..N-1];
sem vacios = N;                          // contenedores libres
sem llenos = 0;                          // contenedores con paquete
int pos_prep = 0;   sem mutexP = 1;      // punta de carga
int pos_entr = 0;   sem mutexE = 1;      // punta de descarga

Process Preparador[id: 1..P]            Process Entregador[id: 1..E]
{ paquete p;                            { paquete p;
  while (true) {                          while (true) {
    p = Preparar();                         P(llenos);
    P(vacios);                              P(mutexE);
    P(mutexP);                                p = sala[pos_entr];
      sala[pos_prep] = p;                     pos_entr = (pos_entr+1) mod N;
      pos_prep = (pos_prep+1) mod N;        V(mutexE);
    V(mutexP);                              V(vacios);
    V(llenos);                              Entregar(p);
  }                                       }
}                                       }
```

**Dos mutex, no uno.** `mutexP` arbitra entre Preparadores y `mutexE` entre Entregadores; entre un Preparador y un Entregador no hace falta candado, porque `vacios`/`llenos` ya garantizan que nunca tocan el mismo contenedor. Con un único mutex compartido, cargar y descargar se serializan — y con el orden de los `P` invertido, es **deadlock**.

*Sanidad:* con `P=1, E=1` los mutex están siempre libres y (d) colapsa exactamente en (a).

---

## P2 — 6
*N personas, un trabajo cada una, con una impresora compartida. Existe `Imprimir(documento)`.*

**Ninguna Persona lleva `while(true)`:** cada una imprime **una sola vez** y termina.

### a) Una impresora, sin importar el orden — sólo procesos Persona

```
// PRE: N > 0

sem mutex = 1;                    // la impresora

Process Persona[id: 1..N]
{ P(mutex);
  Imprimir(documento);            // ADENTRO: acá el candado ES la impresora
  V(mutex);
}
```

Acá **no** aplica "la operación larga va afuera": `Imprimir()` es el uso del recurso protegido. El orden queda indefinido, y el enunciado lo permite.

### b) Orden de llegada, sin procesos extra — Passing the Baton

```
// PRE: N > 0

sem e = 1;                      // protege el estado (libre + cola). NO es la impresora
sem priv[N] = ([N] 0);          // timbre con nombre de cada persona
bool libre = true;
cola espera;

Process Persona[id: 1..N]
{ P(e);
  if (libre) { libre = false;  V(e); }
  else       { push(espera, id);  V(e);  P(priv[id]); }   // V(e) ANTES de dormirse

  Imprimir(documento);          // AFUERA de e

  P(e);
  if (empty(espera)) libre = true;                        // sólo si no hay nadie
  else { pop(espera, sig);  V(priv[sig]); }               // TRASPASO: libre sigue false
  V(e);
}
```

**El recurso no se libera, se entrega.** `libre` queda en `false` durante todo el traspaso: si se pusiera en `true`, un recién llegado se colaría y se rompería el orden. El despertado **no vuelve a hacer `P(e)`**: ya tiene el permiso, y competir de nuevo destruiría el orden construido.

### c) Orden estricto por identificador — la posta

```
// PRE: N > 0;  procesos identificados 1..N

sem turno[N] = ([1] 1, [N-1] 0);     // sólo el primero arranca con permiso

Process Persona[id: 1..N]
{ P(turno[id]);
  Imprimir(documento);
  if (id < N) V(turno[id+1]);        // el último no avisa a nadie
}
```

**Ni cola ni mutex:** el orden es estático, la fila ya está escrita en los identificadores. Existe **un solo permiso** circulando ⇒ la exclusión mutua es consecuencia de la cadena. Costo: la impresora puede quedar **ociosa** esperando al que le toca.

*Ojo:* si el enunciado dijera *"de los procesos **que han solicitado** su uso, el de menor identificador"*, es prioridad **dinámica** → cola con selección del mínimo, no posta.

### d) Orden de llegada, con proceso Coordinador

```
// PRE: N > 0;  cada persona imprime una vez

sem mutexCola = 1;    sem hayPedido = 0;    sem libre = 0;
sem priv[N] = ([N] 0);
cola espera;

Process Persona[id: 1..N]              Process Coordinador
{ P(mutexCola);                        { int sig;
    push(espera, id);                    for i = 1 to N {
  V(mutexCola);                             P(hayPedido);
  V(hayPedido);                             P(mutexCola); pop(espera,sig); V(mutexCola);
                                            V(priv[sig]);        // "es tu turno"
  P(priv[id]);                              P(libre);            // ← espera que termine
  Imprimir(documento);                   }
  V(libre);                            }
}
```

**Desaparece el `bool libre`:** el estado de la impresora es el punto donde está parado el Coordinador. Entre `V(priv[sig])` y `P(libre)` está bloqueado, así que no puede repartir otro turno — **ahí está la exclusión mutua**. Sin ese `P(libre)`, las N personas imprimen a la vez.

El Coordinador itera **N veces y se retira** (nada de `while(true)`).

### e) 5 impresoras — el Coordinador indica cuándo y CUÁL

```
// PRE: N > 0;  'libres' contiene {1,2,3,4,5} antes de crear los procesos

sem mutexCola = 1;    sem hayPedido = 0;
sem hayLibre  = 5;    sem mutexLibres = 1;        // contador + pozo de impresoras
sem priv[N] = ([N] 0);
cola espera;          cola libres;
int asignada[N];                                  // CUÁL le tocó a cada persona

Process Persona[id: 1..N]                 Process Coordinador
{ int imp;                                { int sig, imp;
  P(mutexCola); push(espera,id); V(mutexCola);
  V(hayPedido);                             for i = 1 to N {
                                               P(hayPedido);
  P(priv[id]);                                 P(hayLibre);
  imp = asignada[id];                          P(mutexLibres); pop(libres,imp); V(mutexLibres);
  Imprimir(documento, imp);                    P(mutexCola);   pop(espera,sig); V(mutexCola);
                                               asignada[sig] = imp;     // el DATO primero
  P(mutexLibres);                              V(priv[sig]);            // la SEÑAL después
    push(libres, imp);                      }
  V(mutexLibres);                         }
  V(hayLibre);
}
```

**Un semáforo transmite un instante, no un dato:** el "cuándo" va por `priv[id]` y el "cuál" por `asignada[N]`, escrito **antes** del `V`. `asignada[]` no lleva mutex (un escritor, un lector, ordenados por el propio semáforo), pero **sí** es un arreglo: hay hasta 5 asignaciones en vuelo.

`hayLibre = 5` es un **contador de recursos** (no una señal) → hasta 5 imprimen en paralelo. La cola `libres` da la **identidad** que el contador no puede dar (no se puede consultar el valor de un semáforo). La persona devuelve la impresora al pozo ella misma, así el Coordinador nunca necesita saber cuál se liberó.

*Sanidad:* con `hayLibre = 1` y `libres = {1}` colapsa exactamente en el ítem (d).

---

## P2 — 7
*50 alumnos, 10 tareas (5 por tarea). Barrera al elegir; al terminar avisan al profesor y esperan el puntaje del grupo, que es el orden en que se completó esa tarea.*

**Clave del enunciado:** la nota final garantiza **5 alumnos por tarea** → un grupo está completo al contar 5, y el profesor **no espera a los demás grupos**.

```
// PRE: 50 alumnos, 10 tareas; elegir() reparte exactamente 5 por tarea
//      variables compartidas inicializadas antes de crear los procesos

int contador = 0;                 // cuántos ya eligieron
sem mutexBarrera = 1;             // candado del contador
sem barrera = 0;                  // señal de la barrera

cola avisos;                      // grupos que avisaron
sem mutexAviso = 1;               // candado de la cola (50 escritores)
sem yaAvisado = 0;                // señal: "hay un aviso nuevo"

sem grupos[10] = ([10] 0);        // señal: "tu grupo ya tiene nota"
int puntaje[10];                  // el DATO: la nota de cada grupo

Process Alumno [id: 1..50]
{ int mi_grupo, mi_nota;  bool ultimo;              // LOCALES

  mi_grupo = elegir();
  P(mutexBarrera);
    contador++;
    ultimo = (contador == 50);                      // decidido ADENTRO, en una local
  V(mutexBarrera);
  if (not ultimo) P(barrera);                       // los 49 esperan
  V(barrera);                                       // cascada

  delay(tarea);                                     // sin sincronización

  P(mutexAviso);  push(avisos, mi_grupo);  V(mutexAviso);   // el DATO
  V(yaAvisado);                                             // la SEÑAL

  P(grupos[mi_grupo]);                              // espero mi nota
  mi_nota = puntaje[mi_grupo];                      // la leo
}

Process Profesor
{ int terminados[10] = ([10] 0);                    // único escritor ⇒ SIN candado
  int nota = 1;  int grupo, i, j;

  for i = 1 to 50 {                                 // 50 avisos y me retiro
     P(yaAvisado);
     P(mutexAviso);  pop(avisos, grupo);  V(mutexAviso);
     terminados[grupo]++;
     if (terminados[grupo] == 5) {
        puntaje[grupo] = nota;                      // el DATO primero
        nota++;
        for j = 1 to 5  V(grupos[grupo]);           // la SEÑAL después — ojo: j, no i
     }
  }
}
```

**Barrera:** `ultimo` se calcula **dentro** del candado y se guarda en una local — consultar `contador` afuera es leer sin proteger una variable con 50 escritores. El `V(barrera)` lo hacen **todos**: es la cascada (queda 1 permiso suelto; la variante "el último hace 49 `V`" no deja ninguno).

**Aviso:** productor/consumidor de 50 productores y 1 consumidor. Sin semáforo de "lugares libres" porque cada alumno hace un solo `push` y la cola no desborda.

**Puntaje:** un semáforo transmite un **instante**, no un **dato** → el número viaja por `puntaje[]`, escrito **antes** de los `V`. Ni `puntaje[]` ni `terminados[]` llevan mutex: un solo escritor cada uno. Son **5 `V`** sobre el semáforo **de ese grupo**, no uno global (despertaría a alguien de un grupo que no terminó).

*Sanidad:* 50 push/50 pop · cada `terminados[k]` llega a 5 exactamente una vez · `nota` recorre 1..10 · 5 `V` contra 5 `P` por grupo.

---

## P2 — 8
*Fábrica de T piezas con E empleados. Arrancan cuando llegaron todos; mientras haya piezas toman una; tardan distinto. Al final, quién fabricó más.*

**Un solo proceso: `Empleado`.** La fábrica no decide nada, no es un proceso.

**La bolsa NO puede ser un semáforo.** No se puede consultar su valor, y `P` bloquea cuando está vacío en vez de avisar que lo está: los empleados quedarían demorados para siempre **para cualquier T y E**. Va **entero + candado**.

```
// PRE: T > E, E > 0;  variables compartidas inicializadas antes de crear los procesos

int contador = 0;                        // empleados que llegaron
sem mutexBarrera = 1;   sem barrera = 0;

int piezas = T;                          // LA BOLSA
sem mutexBolsa = 1;

int piezasFabricadas[1..E] = ([E] 0);    // cada uno su casillero

int terminaron = 0;                      // empleados que terminaron de fabricar
sem mutexFin = 1;

Process Empleado [id: 1..E]
{ boolean ultimo, hayParaMi;   int mejor, i;        // LOCALES

  // ── BARRERA ──
  P(mutexBarrera);
    contador++;
    ultimo = (contador == E);            // decidido ADENTRO, en una local
  V(mutexBarrera);
  if (not ultimo) P(barrera);
  V(barrera);                            // cascada

  // ── BOLSA DE TAREAS ──
  hayParaMi = true;
  while (hayParaMi) {
     P(mutexBolsa);
       hayParaMi = (piezas > 0);         // consulta y resta: UN SOLO acto
       if (hayParaMi) piezas--;          // sólo resto si había
     V(mutexBolsa);                      // el candado se suelta EN CADA VUELTA
     if (hayParaMi) {
        fabricarPieza();                 // AFUERA: si va adentro, trabajan de a uno
        piezasFabricadas[id]++;          // único dueño ⇒ sin candado
     }
  }

  // ── ¿SOY EL ÚLTIMO EN TERMINAR? ──
  P(mutexFin);
    terminaron++;
    ultimo = (terminaron == E);
  V(mutexFin);

  // ── INFORME: uno solo ──
  if (ultimo) {
     mejor = 1;
     for i = 2 to E
        if (piezasFabricadas[i] > piezasFabricadas[mejor]) mejor = i;
     Informe(mejor, piezasFabricadas[mejor]);
  }
}
```

**El truco del loop:** la consulta va **adentro** del candado y lo que sale afuera es la **copia local** de la respuesta (`hayParaMi`). Poner `while (piezas > 0)` lee la variable compartida sin proteger; poner el `P(mutexBolsa)` afuera del `while` retiene el candado mientras se fabrica.

**Informa el último en TERMINAR, no el que se llevó la última pieza** — ese recién empieza a fabricarla y otros siguen trabajando. No hace falta barrera antes del informe: el contador + el `if` ya dan el momento correcto y lo hace uno solo.

*"Cada empleado puede tardar distinto tiempo"* descarta el reparto fijo `for k = 1 to T/E`: el rápido terminaría su cuota y quedaría ocioso.

### b) Cualquier valor de T y E — **es la misma solución**

Con T = 3 y E = 5, los empleados 4 y 5 consultan, obtienen `hayParaMi = false`, **no restan**, sueltan el candado y salen del ciclo con 0 piezas. Con T = 0 salen los E. `piezas` nunca se vuelve negativo y nadie se bloquea esperando trabajo inexistente. El informe sigue siendo correcto porque lo emite el último en terminar, momento definido para cualquier T.

El ítem existe para detectar la solución con `sem piezas = T`: ahí los E−T sobrantes quedan demorados para siempre.

*Sanidad:* la suma de `piezasFabricadas[1..E]` da exactamente **T** · `piezas` termina en 0 · exactamente un empleado ve `terminaron == E`.

---

## P2 — 9
*Fábrica de ventanas: 4 carpinteros → depósito de 30 marcos; 1 vidriero → depósito de 50 vidrios; 2 armadores toman un marco y un vidrio (en ese orden) y arman la ventana. Todos continuamente.*

**Dos depósitos ⇒ dos pares de semáforos cruzados, independientes.** No hay orden pedido en ninguna parte: no es traspaso.

```
// PRE: ambos depósitos arrancan vacíos

marco  depMarcos[30];      vidrio depVidrios[50];

sem lugarMarcos  = 30;     sem lugarVidrios  = 50;    // → los esperan los PRODUCTORES
sem marcosListos = 0;      sem vidriosListos = 0;     // → los espera el ARMADOR

int posPonerMarco  = 0;  sem mutexPonerMarco  = 1;    // 4 carpinteros
int posSacarMarco  = 0;  sem mutexSacarMarco  = 1;    // 2 armadores
int posSacarVidrio = 0;  sem mutexSacarVidrio = 1;    // 2 armadores
                                    // el índice de carga de vidrios es LOCAL del vidriero

Process Carpintero [id: 1..4]              Process Vidriero
{ marco m;                                 { vidrio v;  int pos = 0;      // único cargador
  while (true) {                             while (true) {
    m = HacerMarco();      // SIN sincro        v = HacerVidrio();        // SIN sincro
    P(lugarMarcos);                             P(lugarVidrios);
    P(mutexPonerMarco);                         depVidrios[pos] = v;      // sin candado
      depMarcos[posPonerMarco] = m;             pos = (pos + 1) mod 50;
      posPonerMarco = (posPonerMarco+1) mod 30; V(vidriosListos);
    V(mutexPonerMarco);                       }
    V(marcosListos);                        }
  }
}

Process Armador [id: 1..2]
{ marco m;  vidrio v;
  while (true) {
     P(marcosListos);                                  // PRIMERO el marco
     P(mutexSacarMarco);
       m = depMarcos[posSacarMarco];
       posSacarMarco = (posSacarMarco + 1) mod 30;
     V(mutexSacarMarco);
     V(lugarMarcos);

     P(vidriosListos);                                 // DESPUÉS el vidrio
     P(mutexSacarVidrio);
       v = depVidrios[posSacarVidrio];
       posSacarVidrio = (posSacarVidrio + 1) mod 50;
     V(mutexSacarVidrio);
     V(lugarVidrios);

     ArmarVentana(m, v);                               // AFUERA de todo candado
  }
}
```

**`HacerMarco()` no se sincroniza con nada.** *"Cada marco es armado por un único carpintero"* = un carpintero **por marco**, no un carpintero **por vez**. Un `sem carpintero = 1` convertiría los 4 en 1.

**Tres mutex, no cuatro:** sólo donde hay varios del mismo rol. El índice de carga de vidrios es local del vidriero (único escritor). Donde hay candado, la celda se toca **adentro** junto con el avance del índice, o se rompe el orden circular que supone el otro extremo.

**El *"(en ese orden)"*:** los dos armadores piden marco→vidrio. En órdenes distintos, cada uno retiene la mitad de lo que necesita el otro → deadlock por espera circular.

*Sanidad:* `lugar + listos` = capacidad en cada depósito (30 y 50) · los `P`/`V` cruzados cierran en procesos **distintos** · con 1 carpintero y 1 armador los mutex sobran y colapsa al buffer simple.

---

## P2 — 10
*Cerealera: T camiones de trigo, M de maíz. Máximo 7 descargando, no más de 5 del mismo cereal. (a) con coordinador y orden de llegada, que se retira cuando todos descargaron. (b) sin procesos extra, sin orden.*

**Tres contadores, porque 5 + 5 > 7:** ni los límites por tipo garantizan el total ni el total garantiza los de tipo.

### b) Sin coordinador — *"no importa el orden"* ⇒ sin cola ni semáforos privados

```
// PRE: T >= 0, M >= 0
sem cupoTotal = 7;   sem cupoTrigo = 5;   sem cupoMaiz = 5;

Process CamionTrigo [id: 1..T]        Process CamionMaiz [id: 1..M]
{ P(cupoTrigo);   // específico        { P(cupoMaiz);    // específico
  P(cupoTotal);   // general             P(cupoTotal);   // general
  descargar();                           descargar();
  V(cupoTotal);  V(cupoTrigo);           V(cupoTotal);  V(cupoMaiz);
}                                      }
```

**Específico primero, general después.** Al revés, un camión sin cupo de su cereal queda bloqueado **reteniendo un lugar de descarga sin usarlo**. *Contraejemplo:* 5 de trigo descargando (`cupoTotal = 2`); llegan 2 de trigo → consumen los 2 lugares y se bloquean en `P(cupoTrigo)`; llega uno de maíz y **no entra**, aunque 6 ≤ 7, trigo 5 ≤ 5 y maíz 1 ≤ 5. Eso es *"maximice la concurrencia"*. Sin mutex: no hay estructura compartida.

### a) Con coordinador

```
// PRE: camiones numerados 1..T (trigo) y T+1..T+M (maíz)
u
sem cupoTotal = 7;   
sem cupoTrigo = 5;  
sem cupoMaiz = 5;

cola llegada;  

sem mutexLlegada = 1;  
sem hayPedido = 0;
sem permiso[T+M] = ([T+M] 0);          
sem termino = 0;

Process CamionTrigo [id: 1..T]                 // el de maíz es simétrico
{ P(mutexLlegada); push(llegada,{id,TRIGO}); V(mutexLlegada);   // DATO: id + tipo
  V(hayPedido);
  P(permiso[id]);                              // espero que me habiliten
  descargar();
  V(cupoTotal);  V(cupoTrigo);                 // libero YO los cupos
  V(termino);
}

Process CamionMaiz [id: 1..M]                
{ P(mutexLlegada); push(llegada,{id,MAIZ}); V(mutexLlegada); 
  V(hayPedido);
  P(permiso[id]);                  
  descargar();
  V(cupoTotal);  V(cupoMaiz);                
  V(termino);
}

Process Coordinador
{ int i;
  registro c;
  for i = 1 to T+M {                           // reparto permisos en orden
     P(hayPedido);
     P(mutexLlegada);  pop(llegada, c);  V(mutexLlegada);
     if (c.tipo == TRIGO) P(cupoTrigo);  else P(cupoMaiz);   // específico PRIMERO
     P(cupoTotal);                                           // me bloqueo YO
     V(permiso[c.id]);
  }
  for i = 1 to T+M  P(termino);                // me retiro cuando TODOS descargaron
}
```

**El coordinador hace los `P` de cupo en nombre del camión:** si no hay lugar se bloquea **él**, y por eso nadie detrás adelanta al primero de la fila — así se cumple el orden de llegada. **Dos `for`:** el primero reparte permisos, el segundo espera las descargas (*"retirarse cuando todos **han descargado**"*, que no es lo mismo que terminar de repartir).

---

## P2 — 11
*Vacunatorio: 1 empleado, 50 personas, orden de llegada, de a 5 por vez. Espera a que haya al menos 5, vacuna a las 5 primeras, las deja ir, y se retira al atender las 50.*

```
// PRE: 50 personas de a 5 ⇒ 10 tandas exactas (50 mod 5 == 0)

cola espera;                    sem mutexCola  = 1;      // 50 escritores
sem hayPersona = 0;             // señal: "llegó una más"
sem puedeIrse[50] = ([50] 0);   // señal: "ya te vacuné, andate"

Process Persona [id: 1..50]
{ P(mutexCola);  push(espera, id);  V(mutexCola);
  V(hayPersona);                       // toco el timbre
  P(puedeIrse[id]);                    // espero que me vacunen y me dejen ir
}

Process Empleado
{ int atendidos[1..5];  int tanda, k;            // atendidos[] es LOCAL
  for tanda = 1 to 10 {                          // 10 tandas y me retiro
     for k = 1 to 5  P(hayPersona);              // espero a que haya AL MENOS 5
     P(mutexCola);
       for k = 1 to 5  pop(espera, atendidos[k]);    // los 5 primeros, en orden
     V(mutexCola);
     for k = 1 to 5  VacunarPersona();           // AFUERA del candado
     for k = 1 to 5  V(puedeIrse[atendidos[k]]); // los dejo ir AL TERMINAR los 5
  }
}
```

**"Al menos 5" = cinco `P(hayPersona)` seguidos.** Cada persona emite un único `V`, así que cinco `P` exitosos significan cinco llegadas. El semáforo contador ya lleva la cuenta: no hace falta contador auxiliar ni mutex extra.

**Los `V(puedeIrse)` van después de las 5 vacunaciones**, no de a uno — el enunciado dice *"al terminar las deja ir"*. Y son privados: hay que liberar a **esos** cinco, no a cinco cualesquiera.

*Sanidad:* `hayPersona` recibe 50 `V` y 50 `P` (5 × 10) · `VacunarPersona()` corre 50 veces · todos terminan.

---

## P2 — 12
*Terminal: 3 puestos con una Enfermera cada uno, 150 pasajeros. El Recepcionista indica el puesto con menos gente esperando; la enfermera atiende por orden de llegada al puesto. (b) sin Recepcionista.*

**Orden de llegada AL PUESTO ⇒ tres colas independientes**, no una global.

```
// ── Recepción ──
cola recepcion;  sem mutexRecepcion = 1;  sem hayPasajero = 0;
sem asignado[150] = ([150] 0);   int puestoDe[150];       // señal + DATO

// ── Puestos ──
int esperando[1..3] = ([3] 0);   sem mutexEsperando = 1;  // lo tocan recep. y enfermeras
cola colaPuesto[1..3];           sem mutexPuesto[1..3] = ([3] 1);
sem hayEnPuesto[1..3] = ([3] 0); sem hisopado[150] = ([150] 0);

Process Pasajero [id: 1..150]
{ int mi_puesto;
  P(mutexRecepcion); push(recepcion,id); V(mutexRecepcion);   V(hayPasajero);
  P(asignado[id]);   mi_puesto = puestoDe[id];                // leo el DATO
  P(mutexPuesto[mi_puesto]); push(colaPuesto[mi_puesto],id); V(mutexPuesto[mi_puesto]);
  V(hayEnPuesto[mi_puesto]);
  P(hisopado[id]);                                            // me retiro
}

Process Recepcionista
{ int i, p, menor;
  for i = 1 to 150 {                                          // 150 y me retiro
     P(hayPasajero);
     P(mutexRecepcion);  pop(recepcion, p);  V(mutexRecepcion);
     P(mutexEsperando);
       menor = índice de 1..3 con menor esperando[];
       esperando[menor]++;                     // elegir y anotar: UN SOLO acto
     V(mutexEsperando);
     puestoDe[p] = menor;                      // el DATO primero
     V(asignado[p]);                           // la SEÑAL después
  }
}

Process Enfermera [p: 1..3]
{ int id;
  while (true) {                               // sin total propio conocido
     P(hayEnPuesto[p]);
     P(mutexPuesto[p]);  pop(colaPuesto[p], id);  V(mutexPuesto[p]);
     P(mutexEsperando);  esperando[p]--;  V(mutexEsperando);
     Hisopar();                                // AFUERA de todo candado
     V(hisopado[id]);
  }
}
```

### b) Sin Recepcionista — el pasajero elige solo

Desaparece toda la recepción (`recepcion`, `mutexRecepcion`, `hayPasajero`, `asignado[]`, `puestoDe[]`). **Las enfermeras no cambian una línea.**

```
Process Pasajero [id: 1..150]
{ int mi_puesto;
  P(mutexEsperando);
    mi_puesto = índice de 1..3 con menor esperando[];
    esperando[mi_puesto]++;                    // elegir y anotarme: UN SOLO acto
  V(mutexEsperando);
  P(mutexPuesto[mi_puesto]); push(colaPuesto[mi_puesto],id); V(mutexPuesto[mi_puesto]);
  V(hayEnPuesto[mi_puesto]);
  P(hisopado[id]);
}
```

**Elegir el mínimo e incrementarlo tienen que ser atómicos.** Separados, dos pasajeros ven el mismo puesto vacío y los dos van ahí — se rompe justo el balanceo que se quería. Es el mismo razonamiento que *"consultar y llevarse"* en la bolsa de tareas.

*Sanidad:* `Hisopar()` corre 150 veces · `esperando[1]+[2]+[3]` = asignados aún no llamados, nunca negativo · el recepcionista termina, las enfermeras no (el enunciado no lo exige).

---

# Práctica 3 — Monitores

> **Consideraciones de la práctica:** signal and continue · a una `cond` sólo `wait`/`signal`/`signalall` · **NO** `wait` con prioridades · **NO** se puede saber cuántos hay encolados en una `cond` ni si está vacía (una **cola propia** sí se puede consultar) · **los datos sólo viajan por parámetros de los procedures** · **no existen variables globales** · maximizar concurrencia · sin busy waiting · el tiempo con `delay`.

## P3 — 4
*N vehículos cruzan un puente **por orden de llegada**. El puente soporta hasta 50000 kg y cada vehículo tiene su propio peso (ninguno supera el máximo).*

**El estado es un número, no un booleano:** *"¿hay lugar?"* depende de cuánto pesa el que pregunta.

```
// PRE: N > 0;  ningún vehículo pesa más de MAX_PESO

monitor Puente {
   int peso_total = 0;                    // kg arriba del puente ahora
   int MAX_PESO   = 50000;
   cola colaDeEspera;                     // pares {id, peso}, en orden de llegada
   cond cruzar[N];                        // un timbre con nombre por vehículo

   procedure Entrar (int id; int peso)
   { if ((peso_total + peso > MAX_PESO) or (not empty(colaDeEspera)))
        { push(colaDeEspera, {id, peso});      // no entro, O hay fila: me anoto
          wait(cruzar[id]);                    // el que me despierte YA me sumó
        }
     else peso_total = peso_total + peso;      // entro directo: me sumo yo
   }

   procedure Salir (int peso)
   { registro v;
     peso_total = peso_total - peso;
     while ((not empty(colaDeEspera)) and
            (peso_total + verPrimero(colaDeEspera).peso <= MAX_PESO))
        { pop(colaDeEspera, v);
          peso_total = peso_total + v.peso;    // se lo sumo YO, en su nombre
          signal(cruzar[v.id]);
        }
   }
}

Process Vehiculo [id: 1..N]
{ int mi_peso;
  Puente.Entrar(id, mi_peso);
  delay(cruce);                            // cruza — AFUERA del monitor
  Puente.Salir(mi_peso);
}
```

**Las tres claves:**

1. **Te anotás aunque entres por peso, si la fila no está vacía.** Sin el `or (not empty(...))` un auto liviano se cuela delante de un camión que espera. Costo asumido: el puente puede quedar subutilizado — **eso es el orden de llegada estricto**.
2. **Passing the Condition ⇒ `if`, y la suma va en el `else`.** El que sale suma el peso **en nombre** del que despierta, así que el despertado no toca nada. Si sumaras también después del `wait`, el peso se cuenta **dos veces**. Y `peso_total` nunca vuelve a "disponible", por eso nadie se cuela en el hueco del `signal`.
3. **`Salir` despierta a varios y frena en el primero que no entra — no lo saltea.** Saltearlo rompería el orden.

**`delay(cruce)` fuera del monitor**, o pasa uno por vez y los 50000 kg no sirven de nada. **Ningún mutex adentro:** la exclusión mutua del monitor es implícita.

*Sanidad:* `peso_total` = suma de los que están cruzando, nunca > MAX ni < 0 · cada `cruzar[i]` recibe a lo sumo un `signal` · si A se anotó antes que B, A despierta antes.

---

## P3 — 5
*Corralón: N clientes atendidos **por orden de llegada**. El cliente llamado entrega una lista y espera que **alguno** de los empleados le dé el comprobante. (a) 1 empleado · (b) E empleados que no terminan · (c) que terminen al atender a todos.*

**Cuatro procedures, dos por actor.** La regla: *cada vez que un proceso tiene que hacer algo **afuera** del monitor en el medio de su secuencia, hay que cortar en dos llamadas*. Acá el corte es `procesar(L)`.

```
// PRE: N > 0, E >= 1;  cada cliente trae su lista desde el inicio

monitor Corralon {
   cola colaDeEspera;              // pares {id, lista}, en orden de llegada
   cond esperaCliente;             // UNA sola: cualquier empleado libre sirve
   cond esperaEmpleado[N];         // arreglo: hay que despertar a ESE cliente
   comprobante compDe[N];          // el DATO
   bool listo[N] = ([N] false);    // la MEMORIA
   cond compListo[N];              // la ESPERA

   procedure hacerFila (int id; lista L)              // CLIENTE
   { push(colaDeEspera, id, L);
     signal(esperaCliente);
     wait(esperaEmpleado[id]);           // PELADO: rendezvous, signal garantizado
   }

   procedure llamarCliente (out int id; out lista L)  // EMPLEADO
   { while (empty(colaDeEspera))         // WHILE: otro empleado puede ganarme
        wait(esperaCliente);
     pop(colaDeEspera, id, L);
     signal(esperaEmpleado[id]);
   }

   procedure entregarComprobante (int id; comprobante C)   // EMPLEADO
   { compDe[id] = C;
     listo[id] = true;                   // ← lo REGISTRO: evita perder la señal
     signal(compListo[id]);
   }

   procedure recibirComprobante (int id; out comprobante C)  // CLIENTE
   { if (not listo[id]) wait(compListo[id]);     // IF + bandera
     C = compDe[id];
   }
}

Process Cliente [id: 1..N]              Process Empleado [id: 1..E]
{ lista L;  comprobante C;              { int idCliente;            // ≠ mi id
  Corralon.hacerFila(id, L);              lista L;  comprobante C;
  Corralon.recibirComprobante(id, C);     while (true) {
}                                            Corralon.llamarCliente(idCliente, L);
                                             C = procesar(L);       // AFUERA del monitor
                                             Corralon.entregarComprobante(idCliente, C);
                                          }
                                        }
```

**Los tres tipos de `wait`, los tres en este ejercicio:**

| Situación | Forma |
|---|---|
| *"esperá a que se cumpla una condición"* (hace `pop` después) | **`while`** |
| *"te paso el turno, ya te dejé todo hecho"* | **`if`** (Passing the Condition) |
| *"nos esperamos y tengo un `signal` garantizado para mí"* | **`wait` pelado** (rendezvous) |

**La trampa de la señal perdida:** el cliente **sale del monitor** entre sus dos llamadas. Si el empleado procesa rápido y hace `signal(compListo[id])` antes de que el cliente llame a `recibirComprobante`, **la señal se pierde** (una `cond` no tiene memoria) y el cliente duerme para siempre. Por eso la bandera `listo[]`: el booleano **recuerda**, la `cond` **duerme y despierta** — no son alternativas.

*(En `hacerFila` no pasa: el `push` y el `wait` ocurren sin soltar el monitor, porque con signal-and-continue el `signal` no lo libera.)*

**Criterio de `cond` arreglo vs. sola — y de si pasar el `id`:** es la misma pregunta, *¿el monitor tiene que distinguirlo?* El cliente sí (hay que despertarlo a él y guardar su comprobante) ⇒ arreglo + `id`. El empleado no (*"alguno de los empleados"*) ⇒ una sola `cond`, sin `id`.

### b) E empleados — **es la misma solución**

El `while` del empleado (no `if`) y tener todo indexado **por cliente** es lo que la hace funcionar con E > 1: si otro empleado se lleva al cliente en el hueco del `signal`, el despertado re-verifica y se vuelve a demorar. Dos empleados nunca sacan el mismo cliente porque el `pop` ocurre bajo la exclusión mutua del monitor.

### c) Con terminación — contador + `signalall` + bandera de salida

**La trampa:** al atender al último cliente pueden quedar **empleados dormidos** en `wait(esperaCliente)` y nadie va a llegar más. *El mismo estado final es **correcto** en el (b) y es **el error** en el (c): cambió el requisito, no el código.*

**`id == N` NO es "el último":** los clientes llegan en orden arbitrario. El único que puede saberlo es el **monitor, contando** — igual que en una barrera, donde el último es el N-ésimo en llegar, no el que se llama N.

Sólo cambian `llamarCliente` y el `Process Empleado`:

```
int clientesTomados = 0;              // NUEVO: cuántos ya salieron de la cola

procedure llamarCliente (out int id; out lista L; out bool hay)
{ while (empty(colaDeEspera) and (clientesTomados < N))   // AND: dos motivos para dejar de esperar
     wait(esperaCliente);

  if (clientesTomados == N) { hay = false;  return; }     // ya no queda nada

  pop(colaDeEspera, id, L);
  clientesTomados++;
  if (clientesTomados == N)
     signalall(esperaCliente);        // ← despierto a TODOS para que salgan
  signal(esperaEmpleado[id]);
  hay = true;
}

Process Empleado [id: 1..E]
{ int idCliente;  lista L;  comprobante C;  bool hay;
  hay = true;
  while (hay) {
     Corralon.llamarCliente(idCliente, L, hay);
     if (hay) { C = procesar(L);
                Corralon.entregarComprobante(idCliente, C); }
  }
}                                     // TERMINA
```

**Se cuenta al SACAR de la cola, no al entregar el comprobante.** Con N=1 y E=2: A saca al cliente y sale a procesar; B ve la cola vacía y el contador en 0 → espera; A entrega y pone el contador en 1, **pero nadie le avisa a B** → duerme para siempre. Contando los `pop`, el que toma al último **está adentro del monitor** y puede despertar a los demás ahí mismo.

**`signalall` y no `signal`:** a todos los empleados demorados se les acabó el trabajo al mismo tiempo, y el que despertara con `signal` ya no vuelve a entrar a despertar a los otros. **Es el error más común del ítem y es silencioso: el programa da resultados correctos y simplemente no termina.**

### La regla general de terminación

> Cuando el enunciado dice *"debe retirarse / deben terminar"*, preguntate: **¿cada proceso sabe de antemano cuántas vueltas le tocan?**

| | Mecanismo | Ejemplos |
|---|---|---|
| **Sí, reparto fijo** | **`for` acotado** | Vacunatorio (10 tandas) · Cerealera (T+M) · Recepcionista (150) |
| **No, reparto dinámico** | **contador + `signalall` + bandera de salida** | **Corralón (c)**: si uno atiende 30 y otro 20, ninguno puede escribir un `for` |

*Sanidad:* N `push` y N `pop` · cada `esperaEmpleado[i]` recibe un solo `signal` · `listo[i]` pasa a `true` una sola vez · con E = 1 el (b) colapsa al (a) · `while(true)` **siempre** en el proceso, nunca adentro del monitor.

---

# Práctica 4 — Pasaje de Mensajes

> **Consideraciones de PMA:** canales compartidos por todos y FIFO · `send` no bloquea, `receive` sí · `empty` sí, pero no la cantidad de mensajes · `if`/`do` no determinístico (si ninguna opción es verdadera, sigue de largo) · evitar busy waiting · el tiempo con `delay`.

## P4 PMA — 1
*N clientes llegan a la cola de un banco y los atienden sus empleados, por orden de llegada.*

**Cliente/servidor:** el id viaja en el pedido y cada cliente tiene su canal de respuesta privado. El orden de llegada lo da el canal, que es FIFO.

### a) Un empleado

```
chan Pedidos(int);              // el id del cliente
chan Respuestas[1..N](texto);   // privado por cliente

Process Cliente[id: 1..N]
{ texto comp;
  send Pedidos(id);
  receive Respuestas[id](comp);
}

Process Empleado
{ int idC; texto comp;
  while (true) {
     receive Pedidos(idC);
     comp = atender(idC);
     send Respuestas[idC](comp);
  }
}
```

### b) Dos empleados — **es la misma solución**

`Process Empleado[id: 1..2]` con el mismo código: los dos leen del mismo canal FIFO. Un canal por empleado sólo si el cliente eligiera a uno.

### c) Si no hay clientes, tareas administrativas 15 min — Coordinador

**Sin procesos adicionales se puede, pero con demora innecesaria:** con un solo pedido, los dos empleados ven `not empty(Pedidos)`, uno se lo lleva y **el otro queda bloqueado en el `receive`** en vez de hacer tareas administrativas. Hace falta que `Pedidos` tenga **un solo receptor**.

```
chan Pedidos(int);              // el id del cliente
chan PedidoEmple(int);          // el empleado libre pide trabajo
chan Siguiente[1..2](int);      // id del cliente, o -1 si no hay nadie
chan Respuestas[1..N](texto);

Process Cliente[id: 1..N]
{ texto comp;
  send Pedidos(id);
  receive Respuestas[id](comp);
}

Process Coordinador
{ int idE, idC;
  while (true) {
     receive PedidoEmple(idE);
     if (empty(Pedidos)) idC = -1;
     else receive Pedidos(idC);
     send Siguiente[idE](idC);       // SIEMPRE responde
  }
}

Process Empleado[id: 1..2]
{ int idC; texto comp;
  while (true) {
     send PedidoEmple(id);
     receive Siguiente[id](idC);
     if (idC <> -1) { comp = atender(idC);
                      send Respuestas[idC](comp); }
     else delay(15 min);
  }
}
```

*Sanidad:* el id del cliente viaja Cliente → `Pedidos` → Coordinador → `Siguiente[idE]` → Empleado → `Respuestas[idC]` · el empleado **nunca** lee `Pedidos` directo · el Coordinador responde siempre, si no el empleado se duerme en `Siguiente`.

## P4 PMA — 2
*5 cajas y P clientes. Cada cliente va a la caja con menos personas esperando; en cada caja se atiende por orden de llegada y se entrega un comprobante. Maximizar la concurrencia.*

**Coordinador con `cant[5]` local.** Escucha llegadas y "terminé" **con prioridad a los "terminé"** (para asignar con contadores al día) y duerme en `Aviso` para no hacer busy waiting. **El destinatario es el índice del canal:** `Fila[caja]`, `Comprobante[cliente]`.

```
chan Llegada(int);                 // idC
chan Termine(int);                 // idCaja que terminó
chan Aviso(int);                   // "tenés un mensaje nuevo"
chan Fila[1..5](int);              // una por caja: idC
chan Comprobante[1..N](texto);     // una por cliente

Process Cliente[id: 1..N]
{ texto comp;
  send Llegada(id);
  send Aviso(1);                   // DESPUÉS del mensaje
  receive Comprobante[id](comp);
}

Process Coordinador
{ int cant[1..5] = ([5] 0);
  int ok, idC, idCaja, minID;
  while (true) {
     receive Aviso(ok);
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

Process Caja[id: 1..5]
{ int idC; texto comp;
  while (true) {
     receive Fila[id](idC);
     comp = atenderPago(idC);
     send Comprobante[idC](comp);
     send Termine(id);
     send Aviso(1);                // DESPUÉS del mensaje
  }
}
```

*Sanidad:* **avisan todos** los que le mandan algo al Coordinador, y **después** del mensaje (invariante: avisos = mensajes pendientes) · un `receive` **no filtra por valor**: un único canal `Derivacion(caja, idC)` deja que cualquier caja se lleve cualquier cliente · cada `cant[c]++` tiene su `cant[c]--`.

## P4 PMA — 3
*Comida rápida: C clientes, 3 vendedores, 2 cocineros. Cualquier vendedor toma el pedido y se lo pasa a los cocineros; si no hay pedidos, reponen bebidas (1 a 3 min). El cocinero le entrega el plato directo al cliente. Maximizar la concurrencia.*

**Tres moldes:** vendedores que reponen ⇒ **Coordinador único lector** de `Pedidos` (como el 1 c) · vendedor → cocineros: deja la comanda **y sigue**, un solo canal para los 2 cocineros · cocinero → cliente: respuesta privada `Producto[idC]`.

```
chan Pedidos(int, texto);          // idC, pedido
chan PedidoVend(int);              // idV
chan Siguiente[1..3](int, texto);  // idC (o -1), pedido
chan Comanda(int, texto);          // idC, pedido
chan Producto[1..C](texto);        // privado por cliente

Process Cliente[id: 1..C]
{ texto pedido, plato;
  pedido = elegirPedido();
  send Pedidos(id, pedido);
  receive Producto[id](plato);
}

Process Coordinador
{ texto pedido; int idV, idC;
  while (true) {
     receive PedidoVend(idV);
     if (empty(Pedidos)) { idC = -1; pedido = ""; }
     else receive Pedidos(idC, pedido);
     send Siguiente[idV](idC, pedido);    // SIEMPRE responde
  }
}

Process Vendedor[id: 1..3]
{ int idC; texto pedido;
  while (true) {
     send PedidoVend(id);
     receive Siguiente[id](idC, pedido);
     if (idC <> -1) send Comanda(idC, pedido);   // y sigue
     else delay(random(1, 3) min);               // repone bebidas
  }
}

Process Cocinero[id: 1..2]
{ int idC; texto pedido, plato;
  while (true) {
     receive Comanda(idC, pedido);
     plato = cocinar(pedido);
     send Producto[idC](plato);
  }
}
```

*Sanidad:* los cocineros **no** necesitan Coordinador: si no hay comandas no tienen otra cosa que hacer · con 1 vendedor el Coordinador sobra (alcanza `empty`) · el `idC` viaja Cliente → `Pedidos` → Coordinador → `Siguiente` → Vendedor → `Comanda` → Cocinero → `Producto[idC]`.

## P4 PMA — 4
*Locutorio con 10 cabinas y un empleado que atiende a N clientes. El cliente espera que el empleado le indique la cabina, la usa y después le paga; recibe un ticket. El empleado atiende en el orden en que hacen los pedidos. Maximizar la concurrencia.*

**Administrador de recurso:** `libres` + `cabinas` + **cola de pendientes** (= `wait`). Al pagar, si alguien espera, **la cabina se entrega** (= `signal`) y `libres` no sube. **Entre canales no hay orden** ⇒ el (a) va con un solo canal con tipo.

### a) Orden de llegada — un canal con tipo

```
chan Pedidos(texto, int, int);     // tipo, idC, idCabina
chan Respuesta[1..N](int);
chan Ticket[1..N](texto);

Process Cliente[id: 1..N]
{ int miCabina; texto t;
  send Pedidos("cabina", id, -1);
  receive Respuesta[id](miCabina);
  usarCabina(miCabina);
  send Pedidos("pago", id, miCabina);
  receive Ticket[id](t);
}

Process Empleado
{ int libres = 10; set cabinas = {1..10}; cola esperando;
  texto tipo, t; int idC, idCab, sig;
  while (true) {
     receive Pedidos(tipo, idC, idCab);
     if (tipo == "cabina") {
        if (libres > 0) { libres--; remove(cabinas, idCab);
                          send Respuesta[idC](idCab); }
        else push(esperando, idC);
     }
     else {
        t = Cobrar(idC);
        send Ticket[idC](t);
        if (not empty(esperando)) { pop(esperando, sig);
                                    send Respuesta[sig](idCab); }
        else { libres++; insert(cabinas, idCab); }
     }
  }
}
```

### b) Prioridad a los que terminaron — dos canales + `Aviso`

Las ramas son las del (a). Cambia sólo cómo elige.

```
chan PedidoCabina(int);            // idC
chan Pago(int, int);               // idC, idCabina
chan Respuesta[1..N](int);
chan Ticket[1..N](texto);
chan Aviso(int);

Process Cliente[id: 1..N]
{ int miCabina; texto t;
  send PedidoCabina(id);  send Aviso(1);
  receive Respuesta[id](miCabina);
  usarCabina(miCabina);
  send Pago(id, miCabina);  send Aviso(1);
  receive Ticket[id](t);
}

Process Empleado
{ int libres = 10; set cabinas = {1..10}; cola esperando;
  texto t; int ok, idC, idCab, sig;
  while (true) {
     receive Aviso(ok);
     if (not empty(Pago)) ->
           receive Pago(idC, idCab);
           t = Cobrar(idC);
           send Ticket[idC](t);
           if (not empty(esperando)) { pop(esperando, sig);
                                       send Respuesta[sig](idCab); }
           else { libres++; insert(cabinas, idCab); }
     [] (not empty(PedidoCabina)) and (empty(Pago)) ->
           receive PedidoCabina(idC);
           if (libres > 0) { libres--; remove(cabinas, idCab);
                             send Respuesta[idC](idCab); }
           else push(esperando, idC);
     fi
  }
}
```

*Sanidad:* sólo espera el que **pide** cabina (un pago siempre se atiende) · después del `pop` **no** se hace otro `receive` · `disponible` dice cuántas, `remove` dice cuál · cada mensaje del (b) tiene su `Aviso`, después.

---

> **Consideraciones de PMS:** no hay canales, se nombra al proceso (`Destino!port(...)`, `Origen?port(...)`) · envío y recepción bloquean · no hay `empty` de canales · `[*]` sólo en la recepción, **sin orden** · guardas `condición; recepción -> acciones`, **sólo con recepciones**, condiciones sobre variables locales.

## P4 PMS — 1
*R Examinadores encuentran sitios web, se los avisan a 1 Analizador, esperan su respuesta y siguen buscando.*

### b) Sin orden — `[*]` alcanza

```
Process Examinador[id: 1..R]
{ texto sitio, res;
  while (true) {
     sitio = buscarSitio();
     Analizador!pedido(id, sitio);
     Analizador?respuesta(res);
  }
}

Process Analizador
{ int idE; texto sitio, res;
  while (true) {
     Examinador[*]?pedido(idE, sitio);
     res = analizar(sitio);
     Examinador[idE]!respuesta(res);
  }
}
```

### c) Con orden — Admin con cola; el Analizador pide trabajo

```
Process Admin
{ cola fila; int idE; texto sitio;
  do Examinador[*]?pedido(idE, sitio)    -> push(fila, (idE, sitio));
  [] not empty(fila); Analizador?listo() -> pop(fila, (idE, sitio));
                                            Analizador!trabajo(idE, sitio);
  od
}

Process Examinador[id: 1..R]
{ texto sitio, res;
  while (true) {
     sitio = buscarSitio();
     Admin!pedido(id, sitio);
     Analizador?respuesta(res);
  }
}

Process Analizador
{ int idE; texto sitio, res;
  while (true) {
     Admin!listo();
     Admin?trabajo(idE, sitio);
     res = analizar(sitio);
     Examinador[idE]!respuesta(res);
  }
}
```

### d) Sin esperar la respuesta

**Sí cambia:** en PMS el envío bloquea, así que sin respuesta el Examinador **igual espera** en `Analizador!pedido` a que el Analizador se desocupe. Hace falta un **Admin buffer** (siempre dispuesto a recibir): queda como el (c) sin la respuesta.

*Sanidad:* en las guardas sólo recepciones (el servidor **pide** trabajo porque un envío no puede ir en una guarda) · el id viaja en el mensaje porque en el `!` no hay `[*]` · el Admin aparece en (c) para **ordenar** y en (d) para **no bloquear al emisor**.

## P4 PMS — 2
*Laboratorio: el 1º prepara muestras y se las manda al 2º sin esperar; el 2º arma el set, se lo pasa al 3º, espera el resultado y lo archiva; el 3º analiza y le devuelve el resultado al 2º.*

**1º → 2º: Admin buffer** (en PMS el envío bloquea y el 2º está ocupado esperando el análisis) · **2º ↔ 3º: directo** (S1).

```
Process Empleado1
{ texto m;
  while (true) { m = prepararMuestra(); Admin!muestra(m); }
}

Process Admin
{ cola buffer; texto m;
  do Empleado1?muestra(m)                  -> push(buffer, m);
  [] not empty(buffer); Empleado2?pedido() -> pop(buffer, m);
                                              Empleado2!muestra(m);
  od
}

Process Empleado2
{ texto m, set, res;
  while (true) {
     Admin!pedido();
     Admin?muestra(m);
     set = armarSet(m);
     Empleado3!analisis(set);
     Empleado3?resultado(res);
     archivar(res);
  }
}

Process Empleado3
{ texto set, res;
  while (true) {
     Empleado2?analisis(set);
     res = analizar(set);
     Empleado2!resultado(res);
  }
}
```

*Sanidad:* sin ids, porque todos son procesos únicos · la opción del Empleado1 sin condición = el Admin siempre recibe muestras.

## P4 PMS — 3
*N alumnos entregan su examen y esperan la nota; P profesores corrigen en el orden de entrega. Todos los procesos deben terminar.*

**Admin con cola** (como PMS 1 c). **Terminación:** el `do` termina cuando todas sus condiciones son falsas · con P > 1, el Admin le manda un **`-1`** a cada profesor cuando ya no queda nada, y no termina hasta avisarles a los P.

### a) P = 1

```
Process Admin
{ cola Fila; texto E; int idA; int contador = 0;
  do contador < N; Alumno[*]?examen(idA, E) -> push(Fila, (idA, E)); contador++;
  [] not empty(Fila); Profesor?listo()      -> pop(Fila, (idA, E));
                                               Profesor!trabajo(idA, E);
  od
}

Process Profesor
{ int i, idA, res; texto E;
  for (i = 1 to N) {
     Admin!listo();
     Admin?trabajo(idA, E);
     res = puntuar(E);
     Alumno[idA]!nota(res);
  }
}

Process Alumno[id: 1..N]
{ texto examen; int res;
  examen = resolverExamen();
  Admin!examen(id, examen);
  Profesor?nota(res);
}
```

### b) P > 1 — el Admin avisa con `-1`

```
Process Admin
{ cola Fila; texto E; int idA, idP; int contador = 0, terminados = 0;
  do contador < N; Alumno[*]?examen(idA, E) -> push(Fila, (idA, E)); contador++;
  [] not empty(Fila); Profesor[*]?listo(idP) -> pop(Fila, (idA, E));
                                                Profesor[idP]!trabajo(idA, E);
  [] (contador == N) and empty(Fila) and (terminados < P);
        Profesor[*]?listo(idP) -> Profesor[idP]!trabajo(-1, ""); terminados++;
  od
}

Process Profesor[id: 1..P]
{ int idA, res; texto E; bool seguir = true;
  while (seguir) {
     Admin!listo(id);
     Admin?trabajo(idA, E);
     if (idA == -1) seguir = false;
     else { res = puntuar(E); Alumno[idA]!nota(res); }
  }
}

Process Alumno[id: 1..N]
{ texto examen; int res;
  examen = resolverExamen();
  Admin!examen(id, examen);
  Profesor[*]?nota(res);
}
```

### c) Barrera al principio — antes del `do`

```
// En el Admin, antes del do del (b):
for (i = 1 to N) Alumno[*]?llegue();
for (i = 1 to N) Alumno[i]!empezar();

// El Alumno, antes de resolver:
Admin!llegue();
Admin?empezar();
```

*Sanidad:* para cada proceso, *"¿quién le manda el mensaje que lo hace salir?"* · `[*]` sólo para recibir: para avisar a todos, un `for` (no hay broadcast) · cada alumno avisa que llegó **una** vez.

## P4 PMS — 4
*Un simulador de vuelo con exclusión mutua, un empleado que lo administra y P personas que lo usan una vez cada una.*

**Molde S4:** el empleado da el paso y **espera el `termino` antes de dejar pasar a otro** (de ahí sale la exclusión mutua) · `for` de P vueltas porque cada persona lo usa una vez · lo único que cambia entre ítems es **a quién atiende**.

### a) Sin orden

```
Process Empleado
{ int i, idP;
  for (i = 1 to P) {
     Persona[*]?pedido(idP);
     Persona[idP]!pasa();
     Persona[idP]?termino();
  }
}

Process Persona[id: 1..P]
{ Empleado!pedido(id);
  Empleado?pasa();
  usarSimulador();
  Empleado!termino();
}
```

### b) Por orden de id — la posta

```
Process Empleado
{ int i;
  for (i = 1 to P) {
     Persona[i]?pedido();
     Persona[i]!pasa();
     Persona[i]?termino();
  }
}

Process Persona[id: 1..P]
{ Empleado!pedido();
  Empleado?pasa();
  usarSimulador();
  Empleado!termino();
}
```

### c) Por orden de llegada — Admin con cola

```
Process Admin
{ cola Fila; int idP; int contador = 0;
  do contador < P; Persona[*]?pedido(idP) -> push(Fila, idP); contador++;
  [] not empty(Fila); Empleado?proximo()  -> pop(Fila, idP);
                                             Empleado!siguiente(idP);
  od
}

Process Empleado
{ int i, idP;
  for (i = 1 to P) {
     Admin!proximo();
     Admin?siguiente(idP);
     Persona[idP]!pasa();
     Persona[idP]?termino();
  }
}

Process Persona[id: 1..P]
{ Admin!pedido(id);
  Empleado?pasa();
  usarSimulador();
  Empleado!termino();
}
```

*Sanidad:* el orden por id se conoce antes de empezar (alcanza el `for`); el de llegada no (hace falta alguien que anote: el Admin) · **con límite** → Admin con contador y servidor con `for`; **sin límite** → `while (true)`; **varios servidores con límite** → `-1` (PMS 3 b).

## P4 PMS — 5
*Una máquina expendedora usada por E espectadores en orden de llegada, una vez cada uno. **No hay empleado.***

**Un solo Admin que anota y da el acceso** ⇒ `do` con dos opciones + **`bool libre`** · es **Passing the Baton**: al terminar, si hay fila la máquina **se entrega** al siguiente (`libre` sigue en `false`) · se cuentan pedidos **y** `termino` para que el `do` termine.

```
Process Admin
{ cola Fila; int idE; bool libre = true; int llegaron = 0, terminaron = 0;
  do llegaron < E; Espectador[*]?pedido(idE) ->
        llegaron++;
        if (libre) { libre = false; Espectador[idE]!pasa(); }
        else push(Fila, idE);
  [] terminaron < E; Espectador[*]?termino() ->
        terminaron++;
        if (empty(Fila)) libre = true;
        else { pop(Fila, idE); Espectador[idE]!pasa(); }
  od
}

Process Espectador[id: 1..E]
{ Admin!pedido(id);
  Admin?pasa();
  usarMaquina();
  Admin!termino();
}
```

*Sanidad:* `empty(Fila)` **no** dice si la máquina está libre (nadie esperando ≠ nadie usando) · con Empleado aparte (4 c) es un coordinador; con un solo Admin es PtB.

---

# Práctica 5 — Rendezvous (Ada)

> **Consideraciones de la Práctica 5:** no hay variables compartidas · los parámetros del `accept` sólo existen en su cuerpo · `select` del cliente: **una** llamada y **una** opción (`or delay` o `else`) · `select` del servidor: varios `accept` con o sin `when`, sin mezclar con llamadas · después de un `when` siempre un `accept`, y la condición no usa los parámetros · `'count` sólo lo consulta la tarea dueña del entry · aprovechar la comunicación bidireccional.

## P5 — 1
*Puente de un sentido que soporta 5 unidades: auto 1, camioneta 2, camión 3. (b) prioridad a los camiones.*

**R2:** un entry por tipo (el `when` no puede usar parámetros) + `LiberarPuente(peso)` sin condición · suma afuera del `accept`, resta adentro (usa el parámetro) · (b) **R3**: `EntrarCamion'count = 0` en autos y camionetas.

### a) Misma prioridad

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
   -- Camioneta y Camion: iguales, con su entry y su peso (2 y 3)

begin
   null;
end Transito;
```

### b) Prioridad a los camiones

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

*Sanidad:* un `or` entre cada par de alternativas · las declaraciones antes del `begin` se ejecutan una vez · sin paréntesis en llamadas sin parámetros.

## P5 — 2
*Banco con un único empleado que atiende por orden de llegada; clientes que pagan y retiran el comprobante.*

**R1:** el comprobante vuelve por el `OUT` de la misma llamada · el orden de llegada es gratis (cola FIFO del entry) · los tiempos los maneja **el cliente**; el Empleado es el mismo en los cuatro ítems.

### a) Normal

```ada
Procedure Banco is
   task Empleado is
      entry Pago (pago: IN texto; comprobante: OUT texto);
   end Empleado;

   task type Cliente;
   arrClientes: array (1..N) of Cliente;

   task body Empleado is
   begin
      loop
         accept Pago (pago: IN texto; comprobante: OUT texto) do
            comprobante := atender(pago);
         end Pago;
      end loop;
   end Empleado;

   task body Cliente is
      miPago, miComprobante: texto;
   begin
      Empleado.Pago(miPago, miComprobante);
   end Cliente;

begin
   null;
end Banco;
```

### b) A lo sumo 10 minutos

```ada
task body Cliente is
   miPago, miComprobante: texto;
begin
   select
      Empleado.Pago(miPago, miComprobante);
   or delay 600.0;
      null;
   end select;
end Cliente;
```

### c) Sólo si lo atienden inmediatamente

```ada
task body Cliente is
   miPago, miComprobante: texto;
begin
   select
      Empleado.Pago(miPago, miComprobante);
   else
      null;
   end select;
end Cliente;
```

### d) 10 minutos y otro intento inmediato

```ada
task body Cliente is
   miPago, miComprobante: texto;
begin
   select
      Empleado.Pago(miPago, miComprobante);
   or delay 600.0;
      select
         Empleado.Pago(miPago, miComprobante);
      else
         null;
      end select;
   end select;
end Cliente;
```

*Sanidad:* tiempo con decimal (`600.0`) · después de `or delay`/`else` siempre una sentencia (`null;`) · dos `select`, dos `end select`.

## P5 — 7
*Promedio de un vector de 1 millón repartido entre 10 Workers (100 mil cada uno). El Coordinador decide cuándo arrancar y se queda con el resultado. Una sola vez.*

**R9:** los **Workers llaman** (`Comenzar`, `Resultado`) · el Coordinador **no** recorre a los Workers (se clavaría en el lento) · `for 1..2*N` con `select` · cada Worker devuelve la **suma**, se divide al final.

```ada
Procedure Calcular is
   task Coordinador is
      entry Comenzar;
      entry Resultado (suma: IN integer);
   end Coordinador;

   task type Worker;
   arrWorkers: array (1..10) of Worker;

   task body Worker is
      v: vector;
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

*Sanidad:* 10 × 2 llamadas = 20 vueltas · entry declarado con modo y tipo igual al `accept` · el Coordinador sólo suma 10 números.
