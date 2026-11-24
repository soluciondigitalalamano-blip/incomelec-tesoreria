# Runbook · Agente automático del Panel Diario de Tesorería

Este documento es la instrucción completa para la tarea programada que revisa el
correo, actualiza el panel diario, lo publica, avisa a gerencia, envía el correo
a tesorería y marca la entrega en la Bitácora de Notion.
Cada ejecución arranca en una sesión nueva y sin memoria: todo lo necesario está aquí.

**Zona horaria de referencia: America/Bogota (UTC-05:00).**

---

## Paso 0 · Comprobar si el día ya está hecho (salida temprana)

1. Clona el repositorio:
   `git clone --depth 1 https://github.com/soluciondigitalalamano-blip/incomelec-tesoreria /home/claude/incomelec-tesoreria`
   (timeout generoso, ~10 min; una sola vez).
2. Mira el pie de página de `PANEL DIARIO/index.html`:
   `grep -o 'saldos de DIARIO[0-9]*\.xlsx' "PANEL DIARIO/index.html"`
3. Si el nombre ya corresponde al archivo **de hoy** (`DIARIO<DD><MM><AAAA>.xlsx`),
   el panel ya se actualizó en una corrida anterior del mismo día.
   **Termina sin hacer nada, sin avisar y sin enviar correo.**

### Regla del corte: nunca retroceder

Guarda la fecha que lee ese `grep` como **corte publicado**. Más adelante, cuando
tengas el Excel del correo, compara su fecha contra ella:

| Fecha del Excel | Qué hace |
|---|---|
| Posterior al corte publicado | Procesa y publica con normalidad |
| Igual al corte publicado | Ya está hecho: termina en silencio |
| **Anterior al corte publicado** | **No publiques nada.** Avisa por notificación que llegó un diario con fecha anterior a la del panel y que publicarlo haría retroceder el corte. Deja que el dueño decida |

Esto vale aunque el nombre del archivo y la celda `FECHA` de la Hoja1 no coincidan:
manda la celda `FECHA`, y si las dos difieren, repórtalo también. Un diario viejo
reprocesado borra los días ya publicados, y ese daño no se ve hasta que alguien
abre el portal.

## Paso 1 · Buscar el correo con el Excel del día

Busca en Gmail con la herramienta `search_threads`:

```
from:tesoreria@incomelec.com has:attachment newer_than:1d
```

Si no aparece nada, amplía una sola vez a:

```
filename:xlsx newer_than:1d
```

y quédate solo con el adjunto cuyo nombre empiece por `DIARIO` y termine con la
fecha de hoy (`DIARIO<DD><MM><AAAA>.xlsx`, p. ej. `DIARIO24112026.xlsx`).
Si también llega `RESUMEN_DE_PAGOS_SEMANAL.xlsx`, esa es la nueva solicitud semanal
y se procesa junto con el diario.

**Si no hay correo con el archivo de hoy: termina en silencio.** No avises, no envíes
correo, no toques el repositorio. La siguiente corrida del día volverá a mirar.

Si sí está: descarga los adjuntos con `download_gmail_attachment` a una carpeta
nueva y vacía (p. ej. `/home/claude/insumos/`), separada del código.

## Paso 2 · Actualizar el HTML

Lee y sigue **al pie de la letra** el archivo
`.automatizacion/PROMPT_ACTUALIZACION_HTML.md` de este mismo repositorio.
Son 8 pasos obligatorios (tabla intermedia, inventario de `data-pop`, grep de
valores viejos, reconciliación numérica con código, checklist de cierre y
revisión renderizada). No se omite ninguno aunque el cambio parezca pequeño.

El HTML de entrada es el `PANEL DIARIO/index.html` que ya está en el repositorio.

Recuerda las reglas duras de ese prompt: sin notas ni avisos dentro del HTML,
toda moneda extranjera sumada en COP, cada línea de la solicitud con su marca
✓ / ◐ / ✗, y los popovers con el detalle línea por línea.

## Paso 3 · Publicar en el repositorio

El sitio es estático y se publica con GitHub Pages desde la rama por defecto.
Actualiza **tres** cosas y en este orden:

1. `PANEL DIARIO/index.html` ← el panel nuevo. **Antes de guardarlo**, pon en su
   `<head>`, justo después de la etiqueta `viewport`, la marca de actualización
   con el día y la hora reales en que corriste la actualización:

   ```html
   <meta name="lidato:actualizado" content="AAAA-MM-DDTHH:MM-05:00">
   <meta name="lidato:periodo" content="Corte DD mes AAAA">
   ```

   Esto es lo que el portal lee primero para mostrar "Actualizado · hace X · 23 nov
   2026 · 09:00" en la tarjeta del Panel Diario. Si las etiquetas ya existen de un
   corte anterior, **reemplaza sus valores**, no agregues unas nuevas. La hora va en
   formato de 24 horas y zona `-05:00` (Bogotá), y es la hora real de la corrida,
   no una inventada.

   **Cuidado con la hora: el reloj de la máquina está en UTC, no en Bogotá.**
   Nunca pegues la hora de `date` con el sufijo `-05:00`: te adelantarías 5 horas
   y, pasada la medianoche UTC, publicarías el día siguiente. Saca la hora así:

   ```
   TZ=America/Bogota date '+%Y-%m-%dT%H:%M'
   ```

   Antes del commit, comprueba que la hora escrita **no sea posterior** a
   `TZ=America/Bogota date`. Si lo es, está mal: corrígela.
2. La carpeta de la semana en curso, que debe quedar idéntica al panel diario.

   **Cómo se elige la carpeta:** no por el número de semana del mes, sino por el
   bloque de días que el propio panel agrupa en su historial. Si el encabezado del
   historial dice "Semana desde el 19 nov", el corte pertenece a esa semana aunque
   caiga en lunes y parezca de la semana siguiente. Hoy esa carpeta es
   `NOVIEMBRE/TERCERA SEMANA/` (cubre del 19 al 25 de nov). Solo se pasa a
   `CUARTA SEMANA/` cuando el historial arranque en una fecha posterior al 25.
   Antes de escribir, abre la carpeta candidata: si contiene un panel de un corte
   de la misma semana, esa es; si contiene la página de aviso y la otra tiene el
   panel de esta semana, te equivocaste de carpeta.

   Deja la carpeta que no corresponde **tal como estaba**: no conviertas una
   página de aviso en panel ni dejes una semana con un corte viejo.
   Si la carpeta solo tiene la página de aviso, reemplázala por el panel.
   Lleva las mismas dos etiquetas `lidato:` del punto anterior.
3. En el `index.html` de la raíz, dentro de `window.PORTAL.informes`, pon la misma
   fecha y hora en el campo `actualizado` de la entrada `id: 'panel'`
   (formato `'AAAA-MM-DDTHH:MM'`, sin la zona), y también en la entrada del mes en
   curso si se actualizó su carpeta semanal. Este campo es el respaldo por si el
   portal no alcanza a leer el panel; los dos valores deben coincidir siempre.

**Verificación de la fecha antes del commit:**

```
grep -o 'lidato:actualizado[^>]*' "PANEL DIARIO/index.html"
grep -o 'lidato:actualizado[^>]*' "NOVIEMBRE/<SEMANA EN CURSO>/index.html"
grep -n "actualizado: '" index.html | head -2
```

Los tres deben mostrar el mismo día y la misma hora. Si no coinciden, corrígelo
antes de seguir: la tarjeta del portal mostraría una hora que no es.

Luego:

```
git -C /home/claude/incomelec-tesoreria add -A
git -C /home/claude/incomelec-tesoreria commit -m "Panel diario · corte <DD> <mes> 2026"
git -C /home/claude/incomelec-tesoreria -c push.negotiate=true push origin HEAD
```

Confirma que el push quedó bien (`git log --oneline -1` y salida del push sin error).
**Si el push falla, no envíes el correo a gerencia**: avisa el fallo y detente.

## Paso 4 · Avisar que la actualización se hizo

Manda una notificación push al dueño de la tarea con: fecha del corte, archivo
fuente usado, qué rutas se publicaron, y el resumen de la reconciliación
(totales que cuadran, líneas pendientes de la solicitud y cualquier hallazgo
raro: saldos repetidos, bloques sueltos, pagos sin línea en la solicitud).
Esos hallazgos van **solo aquí**, nunca dentro del HTML.

## Paso 5 · Enviar el correo al área de tesorería

**La tarea programada sí envía este correo, por sí sola.** Es el último paso del
flujo y se ejecuta **solo si el Paso 3 terminó bien**: si la actualización o el
push fallaron, no se envía nada, porque anunciaría algo que no se publicó.

- **Para:** `tesoreria@incomelec.com`
- **Asunto:** `Actualización informe tesoreria`
- **Cuerpo HTML:** el contenido de `.automatizacion/correo-aviso.html`.

### Los iconos van ADJUNTOS, no enlazados

El conector de Gmail **elimina del `htmlBody` toda imagen enlazada con una URL**.
Daba igual dónde estuvieran alojadas: se perdían siempre y el correo llegaba con
huecos. Por eso la plantilla ya no enlaza imágenes: usa `cid:att0` y `cid:att1`,
que apuntan a dos adjuntos incrustados.

En la llamada a `send_message` hay que pasar los dos archivos del repositorio,
**en este orden exacto**, porque `attN` es la posición dentro de `attachments`:

| Posición | Archivo | En el HTML |
|---|---|---|
| 0 | `.automatizacion/logo.png` | `cid:att0` — logo en la cabecera |
| 1 | `.automatizacion/usuario.png` | `cid:att1` — icono de Equipo Operaciones |

Cada adjunto se pasa con su contenido en base64, `mimeType: "image/png"` y
**`inline: true`**. Sin `inline: true` aparecerían como archivos adjuntos para
descargar en vez de verse dentro del mensaje. Si inviertes el orden, el logo
saldrá en el pie y el icono de persona en la cabecera.

### El botón siempre llevará un paso intermedio

El conector reescribe todo enlace del correo como
`https://www.google.com/url?q=<destino>&source=gmail&ust=...&sa=E`. Esos enlaces
no llevan firma, así que el destinatario ve la página "Aviso de redireccionamiento"
de Google y tiene que dar un clic más. **No es algo que puedas arreglar desde la
corrida**: no cambies el enlace ni la plantilla intentando evitarlo, y no lo
reportes como un fallo propio. Por eso el correo lleva, debajo del botón, la
dirección escrita en texto como alternativa.

### La fecha del correo se cambia SIEMPRE

La plantilla trae un único campo variable, `{{FECHA_CORTE}}`, en la frase
"Le informamos que al corte del día **{{FECHA_CORTE}}** se ha actualizado…".

Reglas, sin excepción:

1. **Nunca envíes la plantilla con `{{FECHA_CORTE}}` sin reemplazar.** Si el correo
   sale con las llaves visibles, el cliente ve el texto crudo del sistema.
2. La fecha que va ahí es **la del día en que se envía el correo**, tomada
   automáticamente del reloj, nunca escrita a mano ni copiada del correo anterior.
   Sácala así, porque el reloj de la máquina está en UTC y después de las 7 p. m.
   de Bogotá ya marca el día siguiente:

   ```
   python3 -c "
   import datetime, zoneinfo
   M=['Enero','Febrero','Marzo','Abril','Mayo','Junio','Julio','Agosto','Septiembre','Octubre','Noviembre','Diciembre']
   d=datetime.datetime.now(zoneinfo.ZoneInfo('America/Bogota'))
   print('%d de %s' % (d.day, M[d.month-1]))"
   ```

   **No uses `date '+%B'`:** la máquina no tiene el idioma español instalado y
   devolvería `November` en vez de `Noviembre`. Por eso el nombre del mes sale de
   la lista de arriba.

   En la operación normal esta fecha coincide con la del corte, porque el diario
   del día se procesa y se avisa el mismo día. Si algún día no coincidieran
   (por ejemplo, un lunes procesando el diario del viernes), avísalo en la
   notificación al dueño: el correo sale con la fecha del envío, pero conviene
   que él sepa que el corte era de otro día.
3. Formato: `<DD> de <Mes>`, día sin cero a la izquierda y mes en español con
   mayúscula inicial. Ejemplos: `24 de Noviembre`, `3 de Diciembre`, `1 de Enero`.
4. Antes de enviar, comprueba que el cuerpo ya no contenga la cadena
   `FECHA_CORTE`. Si la contiene, el reemplazo falló: corrígelo y no envíes hasta
   que la fecha aparezca escrita.

El resto de la plantilla no se toca: ni el asunto, ni el botón, ni el pie. Si hay
que cambiar el enlace del botón, se cambia en el archivo del repositorio, nunca
en el correo de una corrida.
- Se envía con la herramienta `send_message` de Gmail, campo `htmlBody`.

Envía el correo **una sola vez por día**. Si en el Paso 0 el panel ya estaba al día,
no se envía nada.

---

## Paso 6 · Marcar la entrega en la Bitácora de Notion

Después de enviar el correo, y **solo si el correo salió**, marca la entrega en la
Bitácora. Si el Paso 0 terminó en silencio, o si la actualización, la publicación
o el correo fallaron, **no escribas nada en Notion**.

**Nunca creas filas nuevas en la Bitácora.** El panel diario se marca subiendo el
contador de la fila del ciclo del mes, que ya existe. Eso es todo lo que hay que
hacer.

Base de datos: `LIDATO QUEST · Bitácora`
(ruta: Miguel Andres / Personal / Empresa / Panel informativo)

| Dato | Valor |
|---|---|
| `data_source_id` / tabla SQL | `collection://4204c6f8-c23f-8158-bae8-000bdfbb3f41` |
| `database_id` | `4204c6f8-c23f-817c-b300-d2072cbfa25e` |

### 6.1 · Busca la fila del ciclo del mes

```
python3 -c "
import datetime, zoneinfo
d=datetime.datetime.now(zoneinfo.ZoneInfo('America/Bogota'))
print(d.strftime('%Y-%m-%d'), d.strftime('%Y-%m'))"
```

La primera salida es la fecha de hoy; la segunda es el ciclo. **No uses el reloj
del sistema sin la zona**: está en UTC y de noche ya marca el día siguiente.

```sql
SELECT url, "Hecho", "Total", "Estado", "date:Fecha:start"
FROM "collection://4204c6f8-c23f-8158-bae8-000bdfbb3f41"
WHERE "Registro" = 'INCOMELEC · Informe Diario de Tesorería · <YYYY-MM>'
```

En noviembre de 2026 esa es la fila **N.º 44**, con `Hecho` 15 y `Total` 19.

Si la fila no existe —mes nuevo y nadie la creó todavía— **no la crees y no
inventes el `Total`.** Avisa por notificación al dueño que falta la fila del ciclo
`<YYYY-MM>` para que la cree. El correo ya salió y eso no se deshace.

### 6.2 · El campo `Fecha` de esa fila dice cuál fue el último día marcado

Compáralo con la fecha de hoy antes de tocar nada:

| `Fecha` de la fila | Qué haces |
|---|---|
| Igual a hoy | **Nada.** Hoy ya está marcado. No toques `Hecho` |
| Distinta de hoy, o vacía | Márcalo como dice el 6.3 |

Esta comparación es OBLIGATORIA y es lo único que evita contar dos veces el mismo
día: los cinco agentes comparten la misma jornada y una corrida puede repetirse.
**Nunca subas `Hecho` sin haber hecho esta comparación.**

### 6.3 · Marcar la entrega

`notion-update-page` con `command: "update_properties"` sobre esa fila:

| Propiedad | Valor |
|---|---|
| `Hecho` | el valor que acabas de leer **más 1** |
| `date:Fecha:start` | la fecha de hoy (`<YYYY-MM-DD>`) |
| `date:Fecha:is_datetime` | `0` (el número cero, sin comillas) |

Sobre `Hecho`: es **+1 y solamente +1**, a partir del valor leído en el 6.1. No lo
recalcules contando días hábiles del calendario, no lo deduzcas de la fecha y no
lo pongas en un número «que debería ser». Si el mes tuvo un día sin entrega, el
contador se queda atrás a propósito: eso es justamente lo que se quiere ver.

Además:

- Si el nuevo `Hecho` queda **igual** a `Total`, el ciclo está completo: pon
  `Estado` = `Entregado a tiempo` y `Acuse enviado` = `__YES__`.
- Si el nuevo `Hecho` sigue **por debajo** de `Total`, deja `Estado` en `Parcial`
  y no toques `Acuse enviado`.
- Si el nuevo `Hecho` **pasaría** de `Total`, **no escribas nada**: algo no cuadra
  (un festivo de más, un `Total` mal puesto). Deja la fila como está y avísalo.

No cambies ninguna otra propiedad de esa fila.

### 6.4 · Verifica y repórtalo

Vuelve a leer la fila y comprueba dos cosas: que `Hecho` subió exactamente uno
respecto al valor que leíste, y que `Fecha` quedó en la de hoy.

En la notificación del Paso 4 (o en un aviso corto después) di en una línea qué
pasó: `contador 15 → 16`, `ya estaba marcado hoy, no se tocó`, o el error si Notion
no respondió.

Si Notion no está disponible o la escritura falla, **no reintentes en bucle ni
deshagas el correo**: el correo ya salió y eso es lo importante. Avisa al dueño que
falta marcar el día `<YYYY-MM-DD>` en la fila del ciclo para que lo suba a mano.

### 6.5 · Estado al momento de dejar esto escrito

Al 24 de noviembre de 2026 la fila N.º 44 quedó en `Hecho` 15, `Total` 19,
`Estado` `Parcial` y `Fecha` `2026-11-24`. El 24 es el día hábil número 15 del mes
(descontando el 2 y el 16, que son festivos), así que ese 15 ya incluye la entrega
del 24. La siguiente entrega, la del 25 de noviembre, es la que sube el contador a
16.

---

## Resumen de la condición de parada

| Situación | Qué hace |
|---|---|
| Panel ya tiene el corte de hoy | Termina en silencio |
| No llegó el correo con el Excel de hoy | Termina en silencio |
| Llegó el Excel con fecha posterior al corte publicado | Actualiza, publica, avisa, envía el correo y sube el contador de la Bitácora |
| El Excel tiene fecha anterior al corte publicado | No publica. Avisa al dueño y **no** envía el correo |
| Falla la actualización o el push | Avisa el error, **no** envía el correo y **no** escribe en la Bitácora |
| El correo salió pero Notion falla | Avisa que falta marcar el día. No reintenta en bucle ni reenvía el correo |
| La fila del ciclo ya tiene la fecha de hoy | Hoy ya estaba marcado. No vuelve a contar |
