Tienes dos insumos: `DIARIO_BANCOS.xlsx` (datos nuevos del día) y `index.html` (dashboard actual). Si llega también `RESUMEN_DE_PAGOS_SEMANAL.xlsx`, esa es la nueva solicitud semanal. Sigue este proceso exacto, en este orden. **No omitas ningún paso, incluso si el cambio parece pequeño.**

---

## **Reglas de contenido del dashboard (aplican a todo el proceso)**

1. **Sin notas en el dashboard.** Nada de avisos ⚠, etiquetas "por confirmar", "dato congelado", "fuera de totales", notas de la fuente ni comentarios de auditoría en tarjetas, encabezados, tablas o popovers. Solo valores y números. Cualquier duda, sospecha o hallazgo va **únicamente en la respuesta del chat**, nunca dentro del HTML.
2. **Saldos repetidos se dejan tal cual.** Si un saldo no cambia en varios días (ej. Banco de Bogotá en 999.999.626, Pichincha, Santander), se muestra igual y **sí suma** en todos los totales. Solo se menciona en el chat si parece raro; no se excluye de nada.
3. **Qué es del día y qué es de la semana** (no cambiar esto sin que se pida):
    - Saldos, tarjetas de bancos, flujos por banco, tarjetas de ingresos/egresos por concepto: **día de corte**, con "acumulado semana" como dato secundario.
    - Tarjeta **"Pagado"** (Solicitado vs. ejecutado) y su popover: **todos los pagos de la semana hasta hoy**, día por día y línea por línea.
    - Tarjeta **"Solicitado por tesorería (semana)"** y pestaña de solicitud: **semanal**, conservando las marcas de pagado de días anteriores.
    - Historial (tablas de detalle con filtro por día): todos los días de la semana, abierto en el día de corte.
4. **Toda moneda extranjera suma en pesos.** En cualquier total que mezcle monedas (pagado, solicitado, pendiente, avance), los USD y EUR se convierten a COP con la TRM de la solicitud semanal y se muestran las dos cosas: "Nacional COP X + internacional COP Y (USD Z a TRM T)". Ningún total puede mostrar solo los pesos nacionales dejando por fuera lo internacional. Cada pago en USD/EUR muestra su equivalente en COP.
5. **Estado explícito de cada línea de la solicitud** (nacional e internacional), en la pestaña y en los popovers:
    - ✓ pagado (verde) · ◐ abono con pendiente (naranja) · ✗ no pagado (rojo).
    - Una línea sin pago nunca queda "en blanco": siempre lleva la marca roja de no pagado.
6. **Cambia solo lo que se pidió.** No quites ni agregues secciones, acumulados o datos que no estén en la instrucción. Si algo de lo que existe contradice estas reglas, pregúntalo en el chat antes de tocarlo.

---

## **Paso 1 — Extrae TODOS los datos del Excel a una tabla intermedia antes de tocar el HTML**

Antes de editar nada, lista en una tabla simple (banco → saldo → ingresos → pagos → detalle por concepto) todos los valores de la hoja del día nuevo. **No abras el HTML todavía.** Esto evita que "arrastres" un valor viejo por accidente.

- **Nivel de detalle:** si la fuente trae los pagos línea por línea (un beneficiario, un valor), la tabla intermedia los mantiene línea por línea. Nunca agrupes.
- **Moneda de cada línea:** anota COP / USD / EUR / MXN en cada valor.
- **Bloques sueltos fuera de los totales del banco** (columnas H/I, listas de beneficiarios sin total, notas al lado de un saldo): compáralos contra los pagos de días anteriores. Si son idénticos a pagos ya registrados, son arrastre del día anterior y **no se cuentan como movimiento de hoy**. Repórtalo en el chat.
- **Saldos repetidos:** compara cada saldo contra los cortes anteriores del dashboard (el Excel suele traer una sola hoja). Si se repite, **se usa igual** (regla 2) y solo se menciona en el chat.

## **Paso 2 — Abre el HTML y construye el inventario cerrado de TODOS los lugares con datos**

Cataloga — solo estructura, no valores — cada lugar donde vive un dato:

- Cada `data-pop="..."` y su entrada en `POP_DATA` (`grep -oP 'data-pop="[^"]*"'` vs keys de `POP_DATA`; deben coincidir exactamente, sin huérfanos).
- Cada tarjeta de banco, cada tarjeta resumen (KPI), cada `<table>` por día o por beneficiario.
- Para cada tarjeta, anota si es **del día** o **de la semana** según la regla 3.
- Las tablas que **rastrean el mismo dinero desde ángulos distintos** (diario de pagos vs. solicitud semanal; pagos nacionales vs. internacionales) se marcan como pares que deben reconciliar.

Entrega esta lista antes de seguir. Es tu checklist del Paso 7.

## **Paso 3 — Identifica cada lugar donde un mismo dato aparece repetido**

Un mismo número (ej. el saldo de Davivienda) aparece en varios sitios: tarjeta del banco, total de la pestaña, popover, cobertura bancaria. Busca con `grep` todas las apariciones del valor viejo y confirma que cada una corresponde a una entrada del inventario y que todas se reemplazaron.

## **Paso 4 — Nunca reemplaces solo por texto suelto — usa contexto único**

Incluye siempre contexto (nombre del banco, etiqueta del campo) para que el reemplazo sea inequívoco. Preferible: generar todo desde un solo modelo de datos para que tarjetas, tablas y popovers no puedan desincronizarse.

## **Paso 4 bis — El popover de un banco abre con el día del corte, no con la semana**

Es la regla 3 aplicada a los popovers, y es el error que más ha reclamado el
cliente. Si la primera fila de KPI dice "ingresos de la semana", el cliente abre
la tarjeta el día del corte y ve el movimiento **del día anterior** como si fuera
el de hoy. El 26-nov-2026 pasó dos veces: Colpatria mostraba 142.217.361 de
ingresos cuando ese día no entró nada, y en Bancolombia los cuatro pagos de
SUPLEC del día quedaban debajo del pago del día anterior, fuera de la pantalla.

Cómo se arma el popover de banco:

1. **Primera fila de KPI, los cuatro del mismo día:** `Saldo de apertura · <día
   anterior>` (el saldo al corte del panel publicado ayer) · `Ingresos del <día
   del corte>` · `Pagos del <día del corte>` · `Saldo al corte · <día del corte>`.
   Un día sin movimiento muestra 0, nunca el valor de otro día.
2. **Segunda fila de KPI, la semana como dato secundario:** `Saldo antes de la
   semana` (el saldo del día anterior al inicio de la semana) · `Ingresos de la
   semana` · `Pagos de la semana`. Las dos filas tienen que cuadrar cada una por
   su lado; nunca se mezclan una apertura de un periodo con los movimientos de otro.
3. **Las tablas llevan columna `Fecha` y arrancan por el día del corte**, con un
   subtotal por día y, si la semana tiene más de un día, un total de la semana al
   final. Los días anteriores van debajo, nunca encima.

La resta puede no dar exacta si la fuente trae diferencias (Paso 1 bis punto 5
del runbook). Esa diferencia se reporta en la notificación y **no** se tapa
ajustando la apertura ni recortando movimientos.

## **Paso 5 — Mismo nivel de detalle en cada ubicación**

Para cada dato que aparezca en más de un lugar, el desglose debe ser **idéntico** en todas sus apariciones. Prohibido agrupar "para ahorrar espacio": es preferible un popover largo a ocultar un pago.

## **Paso 6 — Verificación cruzada obligatoria antes de entregar**

**6a. Grep de valores viejos.** Busca cada valor viejo clave (saldos anteriores, totales del día anterior) y confirma que solo quedan donde corresponde (saldo anterior, historial).

**6b. Reconciliación numérica.** Con código, no a ojo:

1. Suma todas las filas de cada tabla y popover y compárala con su total.
2. Reconciliación de pares: pagos del día (tarjetas por concepto = flujos por banco = historial); pagos de la semana (tarjeta "Pagado" = suma de los días del historial, con lo internacional convertido a COP).
3. **Avance de la solicitud:** recalcúlalo línea por línea (pagado imputado / solicitado, todo en COP) y lista en el chat **todas las líneas pendientes** con su valor (en COP y en moneda original). Si el avance parece bajo o alto, esa lista es la que lo explica.
4. Si dos totales no coinciden, explica la diferencia línea por línea en el chat (pagos sin línea en la solicitud, excedentes sobre lo solicitado, pagos de días anteriores a la semana).

Pega el resultado de 6a y 6b en tu respuesta.

## **Paso 7 — Checklist de cierre contra el inventario del Paso 2**

Para cada ítem confirma:

- [ ]  Su valor es del corte nuevo (o de la semana, si es semanal según la regla 3).
- [ ]  Si es un popover: cada línea del Excel aparece individualmente.
- [ ]  Si mezcla monedas: lo internacional está sumado en COP y se ve la moneda original.
- [ ]  Si es una línea de la solicitud: tiene su marca de pagado / abono / no pagado.
- [ ]  No hay ninguna nota, ⚠ ni etiqueta de auditoría (`grep -c "⚠\|por confirmar\|congelado\|fuera de"` debe dar 0).
- [ ]  El `data-pop` existe y no quedó huérfano.

## **Paso 8 — Nunca entregues sin haber visto el resultado renderizado**

Abre el HTML con Playwright y toma screenshot de **cada pestaña y cada popover**. Revisa además que las tarjetas KPI de los popovers no se corten (máximo 3-4 por fila).

---

### **Regla permanente**

**Todas las tarjetas deben tener el detalle completo al hacer clic**, con el mismo nivel de desglose que la fuente — sin resúmenes, sin líneas agrupadas, y sin popovers vacíos o desincronizados del dato que representan.