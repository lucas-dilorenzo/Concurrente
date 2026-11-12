# Simulacro 1 — Memoria Distribuida (11-nov)

> Tres ejercicios de `PRACTICA/Practica Repaso MD.pdf` (PM 1a, PM 1b, Ada 1), uno por mecanismo como el parcial. **Se hizo de a uno, en ping-pong.**

| Ej | Modelo | Resultado |
|---|---|---|
| 1 PMA | M1 | ✅ Bien. Detalles: `Process` y `texto doc;` |
| 2 PMS | S1 (PMS 4c) | ✅ Estructura bien. ⚠️ `!` en la guarda y falta el índice `Impresora[idI]` |
| 3 Ada | Cliente con `or delay` dentro de un `for` | ❌ Sintaxis no firme: se pasó la resolución |

**Diagnóstico:** **reconoció los tres modelos** (lo que más costó en el primer parcial). Lo que falla es **la sintaxis mezclada entre mecanismos** ⇒ [Hoja de sintaxis](Hoja%20de%20sintaxis.md).

---

## 1) PMA — impresoras

> En una oficina existen **100 empleados** que envían documentos para imprimir en **5 impresoras compartidas**. Los pedidos de impresión son procesados por orden de llegada y se asignan a la **primera impresora que se encuentre libre**.

```
chan enviarDoc(texto);

Process Empleado[id: 1..100] {
  texto doc;
  send enviarDoc(doc);
}

Process Impresora[id: 1..5] {
  texto doc;
  while (true) {
    receive enviarDoc(doc);
    imprimir(doc);
  }
}
```

- **"La primera libre" no se programa:** la libre está parada en el `receive`; la ocupada está imprimiendo. El canal compartido es la fila (una sola fila con varias cajas).
- **Sin canal de respuesta:** la impresora imprime, no devuelve nada. Por eso tampoco va el id en el mensaje.
- *Errores:* `Procedure` → `Process` · `Texto: doc` → `texto doc;` · faltaban `;`.

## 2) PMS — impresoras

```
Process Admin {
  cola Fila; texto doc; int idI;
  do Empleado[*]?pedido(doc) -> push(Fila, doc);
  [] not empty(Fila); Impresora[*]?listo(idI) -> pop(Fila, doc);
                                                 Impresora[idI]!imprimir(doc);
  od
}

Process Empleado[id: 1..100] {
  texto doc;
  Admin!pedido(doc);
}

Process Impresora[id: 1..5] {
  texto doc;
  while (true) {
    Admin!listo(id);
    Admin?imprimir(doc);
    imprimir(doc);
  }
}
```

- **El Admin no busca impresoras:** la libre va sola a avisar (el mozo que pasa por la cocina). El Admin la escucha sólo si hay algo en la fila: esa es la guarda.
- *Errores:* ⚠️ `Impresora[*]!listo` en la guarda → **`?`** · ⚠️ `Impresora!imprimir` → **`Impresora[idI]!`** · faltaba `od` · la impresora no llamaba a `imprimir(doc)` · `Procedure` · `doc: Texto`.

## 3) Ada — Banco Central

> La página web del Banco Central exhibe las cotizaciones del dólar oficial de **20 bancos**, compra y venta. Una **tarea programada** actualiza la página **en forma periódica** consultando a cada banco. Cada banco dispone de una **API**, cuya única función es procesar las solicitudes de aplicaciones externas. La tarea programada consulta **de a una API por vez**, esperando **a lo sumo 5 segundos** su respuesta. Si no respondió, se muestra **vacía** la información de ese banco.

```ada
Procedure BancoCentral is
   task Programada;

   task type API is
      entry Consulta (compra: OUT float; venta: OUT float);
   end API;
   arrAPI: array (1..20) of API;

   task body API is
   begin
      loop
         accept Consulta (compra: OUT float; venta: OUT float) do
            compra := cotizacionCompra;
            venta  := cotizacionVenta;
         end Consulta;
      end loop;
   end API;

   task body Programada is
      compras, ventas: array (1..20) of float;
   begin
      loop                                       -- "en forma periódica"
         for i in 1..20 loop                     -- "de a una API por vez"
            select
               arrAPI(i).Consulta(compras(i), ventas(i));
            or delay 5.0;                        -- "a lo sumo 5 segundos"
               compras(i) := -1;                 -- "se muestra vacía"
               ventas(i)  := -1;
            end select;
         end loop;
         actualizarPagina(compras, ventas);
      end loop;
   end Programada;

begin
   null;
end BancoCentral;
```

- **La Programada es el cliente** (el que pregunta); la API, la ventanilla.
- **Las APIs no necesitan `Ident`:** la Programada ya elige a quién llama con `arrAPI(i)`, y el `OUT` vuelve solo. El id hace falta sólo si **la tarea** tiene que decirle a otro quién es.
- **Acá el `for` no es la trampa del ej 7:** el enunciado lo pide, y el `or delay` evita que una API colgada frene al resto.
- `task Programada;` sin `is ... end`: no tiene entries.
- *Planteo:* tareas ✅ · quién llama ✅ · faltó **el `select` del cliente con `or delay 5.0`** · ❌ creyó que las APIs necesitaban su id.
