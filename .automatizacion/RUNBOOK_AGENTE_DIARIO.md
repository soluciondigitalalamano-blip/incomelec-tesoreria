# Runbook · Agente automático del Panel Diario de Tesorería

Este documento es la instrucción completa para la tarea programada que revisa el
correo, actualiza el panel diario, lo publica, avisa a gerencia, envía el correo
a tesorería por Resend y marca la entrega en la Bitácora de Notion.
Cada ejecución arranca en una sesión nueva y sin memoria: todo lo necesario está aquí.

**Zona horaria de referencia: America/Bogota (UTC-05:00).**

---

## Paso 0 · Comprobar si el día ya está hecho (salida temprana)

1. Clona el repositorio:
   `git clone --depth 1 https://github.com/soluciondigitalalamano-blip/incomelec-tesoreria /home/claude/incomelec-tesoreria`
   (timeout generoso, ~10 min; una sola vez).

   **Si la carpeta ya existe** (a veces es el directorio de trabajo de la sesión),
   no la borres ni vuelvas a clonar: `rm -rf` sobre ella está bloqueado y la
   corrida se traba. Haz esto:

   ```
   git -C /home/claude/incomelec-tesoreria fetch origin --depth 1
   git -C /home/claude/incomelec-tesoreria reset --hard origin/main
   ```
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

### Un segundo archivo del mismo día manda sobre el primero

Tesorería a veces manda el diario incompleto y después lo corrige. **Antes de
hacer el commit, vuelve a buscar en Gmail** con la misma consulta: si hay un
adjunto más nuevo para hoy, trabaja con ese y descarta el anterior.

Si tienes dos archivos del mismo día y **no coinciden**, no publiques ninguno:
avisa por notificación cuáles celdas difieren. El 25-nov-2026 llegaron dos
versiones; la segunda traía una consignación de Parqueoo que la primera no
tenía, y además decía `FECHA` 24 en vez de 25.

**La celda `FECHA` manda sobre el nombre del archivo.** Un archivo llamado
`DIARIO25112026.xlsx` cuya celda `FECHA` dice 24 es el diario del 24, y aplica
la regla de no retroceder el corte: no se publica.

## Paso 1 bis · Validar el Excel antes de creerle

La fuente trae errores. Estos cuatro controles son obligatorios y van **con
código**, antes de tocar el HTML.

**1. Lee el libro dos veces: con valores y con fórmulas.**

```python
openpyxl.load_workbook(ruta, data_only=True)    # valores
openpyxl.load_workbook(ruta, data_only=False)   # fórmulas
```

**2. Cada TOTAL contra sus propias filas.** Si el rango de la fórmula deja
fuera una línea con valor, el total del Excel está mal: **vale la línea, no el
total**. Usa la suma real y repórtalo en la notificación.
El 25-nov-2026: `TOTAL PAGOS BANCOLOMBIA` era `=SUM(D49:D54)` y dejaba fuera la
retención de D46 por 17.793.000, así que el Excel mostraba 0.

**3. Saldos escritos a mano.** Revisa si algún saldo de la columna E es una
fórmula con el saldo del día anterior incrustado. Esas fórmulas suelen ignorar
las consignaciones. El 25-nov-2026 Parqueoo tenía `=118812724-D77`, que resta
pagos y no suma ingresos.

**4. Arrastre del día anterior.** Compara **cada línea de pago y de
consignación** contra el panel del día anterior, no solo los bloques sueltos.
Si una línea es idéntica (mismo beneficiario y mismo valor) a una ya publicada
ayer, es arrastre: **no cuenta como movimiento de hoy** y se reporta.
El 25-nov-2026 el pago de TKH por USD 10.063 venía repetido del 24.

**5. Saldo contra movimientos.** Para cada banco:
`saldo de apertura + consignaciones − pagos` debe dar el saldo del Excel.
Lista en la notificación todos los que no cierren, con la diferencia exacta.
No ajustes nada para que cuadre: se publica lo que dice la fuente y se reporta
la diferencia.

### Nunca publiques una cifra que no esté en la fuente

Si un número no se puede leer del archivo — viene cortado, falta, o dos
versiones se contradicen — **para y avisa**. No lo deduzcas por aritmética, no
lo infieras de una captura de pantalla y no lo completes «porque es el único
valor que cuadra». Un dígito inventado en un saldo de tesorería no se nota hasta
que alguien concilia contra el banco.

### Los popovers son del día del corte y de nada más

KPI del día (apertura del día anterior, ingresos, pagos y saldo al corte) y tablas
con solo los movimientos de ese día. Los días anteriores se ven únicamente en
«Detallado de ingresos recibidos» y «Detallado de pagos ejecutados», que tienen su
filtro, y en la tarjeta «Pagado en la semana». Detalle en el Paso 4 bis de
`PROMPT_ACTUALIZACION_HTML.md`. Es el reclamo más repetido del cliente.

### Convención de las filas de las tablas de detalle

Las filas nuevas del historial copian la convención del panel de la semana
anterior, o los filtros por día dejan de funcionar:

- ingresos: `<tr data-ingrow="25nov" data-week="4">`
- pagos: `<tr data-dayrow="25nov" data-week="4">`
- el total del día lleva los mismos atributos; el total acumulado no lleva ninguno
- el banco va en `<span class="chip banco">`, el beneficiario en `<td class="strong">`

---

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
git -C /home/claude/incomelec-tesoreria -c push.negotiate=true push origin HEAD:refs/heads/main
```

Confirma que el push quedó bien (`git log --oneline -1` y salida del push sin error).
**Si el push falla, no envíes el correo a gerencia**: avisa el fallo y detente.

## Paso 4 · Avisar que la actualización se hizo

Manda una notificación push al dueño de la tarea con: fecha del corte, archivo
fuente usado, qué rutas se publicaron, y el resumen de la reconciliación
(totales que cuadran, líneas pendientes de la solicitud y cualquier hallazgo
raro: saldos repetidos, bloques sueltos, pagos sin línea en la solicitud).
Esos hallazgos van **solo aquí**, nunca dentro del HTML.

La notificación incluye además, siempre, lo que **no** cuadró: totales del Excel
corregidos por fórmula mala, líneas tomadas como arrastre, saldos que no cierran
con la diferencia exacta, pagos sin línea en la solicitud y excedentes. Si todo
cuadró, dilo en una línea. Un silencio no se distingue de un control que no se
hizo.

## Paso 5 · Enviar el correo al área de tesorería

El correo **no sale por Gmail**. Sale por **Resend**, con el dominio propio
`lidato.site`, usando la herramienta `mcp__Resend__send-email`.

No uses `send_message` de Gmail para este correo. Se probó y no sirve: el
conector de Gmail reescribe toda dirección web del mensaje y el cliente termina
viendo una pantalla de advertencia de Google antes de llegar al panel. La sección
«Por qué no se manda por Gmail» al final de este paso tiene el detalle.

### Qué enviar

| Campo | Valor |
|---|---|
| `from` | `LIDATO <operaciones@lidato.site>` |
| `to` | `["tesoreria@incomelec.com"]` |
| `bcc` | `["soluciondigitalalamano@gmail.com"]` |
| `replyTo` | `["operaciones@lidato.site"]` |
| `subject` | `Actualización informe tesorería` |
| `html` | el contenido de `.automatizacion/correo-aviso.html`, con `{{FECHA_CORTE}}` reemplazado |
| `text` | el contenido de `.automatizacion/correo-aviso.txt`, con `{{FECHA_CORTE}}` reemplazado |

`text` es obligatorio cuando mandas `html`. No lo omitas.

La copia oculta a `soluciondigitalalamano@gmail.com` va en **todos** los envíos: es el
único registro del aviso que queda fuera de Resend. No la quites ni la pongas en `cc`.

### Los iconos van como adjuntos incrustados

En `attachments`, exactamente estos dos y en este orden:

| `filename` | `url` | `contentId` |
|---|---|---|
| `logo.png` | `https://raw.githubusercontent.com/soluciondigitalalamano-blip/mis-imagenes/refs/heads/main/logo-white.png` | `logolidato` |
| `usuario.png` | `https://raw.githubusercontent.com/soluciondigitalalamano-blip/mis-imagenes/refs/heads/main/user-icon-teal.png` | `iconousuario` |

Resend descarga esas imágenes de GitHub y las incrusta en el correo. La plantilla
las llama por `cid:logolidato` y `cid:iconousuario`: esos nombres tienen que
coincidir con los `contentId` o los iconos salen rotos.

### El botón se deja tal cual

La plantilla trae `href="https://tesoreria-prueba.pages.dev/"`. Resend entrega el
enlace sin tocarlo, así que el cliente da un clic y entra directo. **No cambies el
enlace, no lo envuelvas en nada y no le agregues parámetros.**

### La fecha del correo se cambia SIEMPRE

La plantilla trae un único campo variable, `{{FECHA_CORTE}}`, en la frase
"Le informamos que al corte del día **{{FECHA_CORTE}}** se ha actualizado…", tanto
en el HTML como en el texto plano.

Reglas, sin excepción:

1. **Nunca envíes con `{{FECHA_CORTE}}` sin reemplazar**, ni en el HTML ni en el
   texto. Si sale con las llaves visibles, el cliente ve el texto crudo del sistema.
2. La fecha es **la del día en que se envía el correo**, tomada automáticamente del
   reloj, nunca escrita a mano ni copiada del correo anterior. Sácala así, porque el
   reloj de la máquina está en UTC y después de las 7 p. m. de Bogotá ya marca el
   día siguiente:

   ```
   python3 -c "
   import datetime, zoneinfo
   M=['Enero','Febrero','Marzo','Abril','Mayo','Junio','Julio','Agosto','Septiembre','Octubre','Noviembre','Diciembre']
   d=datetime.datetime.now(zoneinfo.ZoneInfo('America/Bogota'))
   print('%d de %s' % (d.day, M[d.month-1]))"
   ```

   **No uses `date '+%B'`:** la máquina no tiene el idioma español instalado y
   devolvería `November` en vez de `Noviembre`.

   En la operación normal esta fecha coincide con la del corte. Si algún día no
   coincidieran (un lunes procesando el diario del viernes), el correo sale con la
   fecha del envío, pero avísalo en la notificación al dueño.
3. Formato: `<DD> de <Mes>`, día sin cero a la izquierda y mes en español con
   mayúscula inicial. Ejemplos: `24 de Noviembre`, `3 de Diciembre`, `1 de Enero`.
4. Antes de enviar, comprueba que ni el HTML ni el texto contengan ya la cadena
   `FECHA_CORTE`.

El resto de la plantilla no se toca: ni el asunto, ni el botón, ni el pie. Si hay
que cambiar el enlace del botón, se cambia en el archivo del repositorio, nunca en
el correo de una corrida.

### Después de enviar, verifica

`mcp__Resend__send-email` devuelve un `id`. Con ese id llama a
`mcp__Resend__get-email` y comprueba dos cosas en el HTML que quedó enviado:

- que aparezca `href="https://tesoreria-prueba.pages.dev/"` **sin** `google.com/url`
  ni ningún otro envoltorio;
- que no quede la cadena `FECHA_CORTE`.

Si algo de eso falla, dilo en la notificación al dueño.

### Los estados de Resend, y qué prueba cada uno

Verificado contra la documentación de Resend
(`resend.com/guides/email/users-never-got-login-code`,
`resend.com/docs/webhooks/event-types`):

| Estado | Qué significa |
|---|---|
| `sent` | Resend lo procesó y lo está entregando al servidor del destinatario |
| `delivered` | **El servidor del destinatario aceptó el mensaje.** Nada más |
| `delivery_delayed` | Rechazo temporal (buzón lleno, límite de tasa). Reintenta hasta 14 h antes de marcarlo rebotado |
| `bounced` | El servidor lo rechazó de forma permanente |
| `complained` | Se entregó y el destinatario lo marcó como spam |
| `suppressed` | No se envió: la dirección está en la lista de supresión |
| `failed` | Falló el envío: destinatario inválido, API key, verificación de dominio o cuota |

La documentación es explícita: *«'Delivered' only means the recipient's mail
server accepted the email»*, y lo que pase después — bandeja, spam, cuarentena —
lo decide el proveedor del destinatario y **no se le reporta al remitente**.

**Nunca escribas que el correo llegó.** En la notificación va
«aceptado por el servidor del destinatario». El 25-nov-2026 tres envíos
figuraban `delivered` y la destinataria no vio ninguno: estaban dentro de
INCOMELEC, retenidos.

### Comprobación obligatoria después de enviar

Con el `id` que devuelve `send-email`, llama a `mcp__Resend__get-email` y:

1. Si el estado es `bounced`, `failed` o `suppressed`, **el correo no salió**:
   avísalo como fallo, no como entrega.
2. Comprueba en el HTML enviado que esté `href="https://tesoreria-prueba.pages.dev/"`
   sin `google.com/url`, y que no quede la cadena `FECHA_CORTE`.
3. Reporta el `id` y el estado textual, sin interpretarlo.

### Si el cliente dice que no le llegó

No reenvíes el mismo correo: si figura `delivered`, un envío idéntico cae en el
mismo sitio. Sigue este orden, que es el que recomienda Resend:

1. `mcp__Resend__get-suppression <dirección>` — si está suprimida, ahí está la causa.
2. Mira el estado real del envío. `delivered` sin rebote significa que el correo
   **entró al dominio del destinatario** y quedó retenido de su lado.
3. Pide que busquen **por el asunto exacto**, no por el remitente, en todo el
   correo incluido spam. Resend dice que buscar por asunto lo encuentra antes.
4. Correo corporativo: la cuarentena está **fuera** del buzón. En Microsoft 365
   se revisa en `security.microsoft.com/quarantine`; el usuario puede liberar y
   marcar «Allow sender», pero lo clasificado como phishing de alta confianza
   solo lo libera un administrador. Liberar un mensaje libera **ese** mensaje:
   para los siguientes hace falta que TI ponga el dominio en la lista de permitidos.
5. A TI se le da el **dominio de envío**, no direcciones IP: las IP del proveedor
   cambian.

### Por qué pasa, y qué lo arregla de raíz

`lidato.site` se dio de alta en Resend el 24-nov-2026. Un dominio nuevo no tiene
reputación, y la documentación advierte que el spam es más probable *«for new
domains, for the first email a user gets from you»*. El 25-nov fue la primera vez
que este dominio escribió a `incomelec.com`.

Pendientes, en orden de impacto:

1. **Publicar DMARC.** Hoy `lidato.site` tiene SPF y DKIM verificados pero **no
   tiene registro DMARC**. Resend lo recomienda y advierte que Gmail y Yahoo lo
   exigen a remitentes masivos desde 2024. El valor para empezar, textual de su
   documentación:

   ```
   Nombre: _dmarc.lidato.site
   Tipo:   TXT
   Valor:  v=DMARC1; p=none; rua=mailto:operaciones@lidato.site;
   ```

   Se empieza en `p=none` — *«we suggest starting with a policy of p=none;
   before moving to a stricter policy»* — y solo se sube a `p=quarantine` y
   después a `p=reject` cuando todo el correo legítimo pase DMARC.
2. **Lista de permitidos en INCOMELEC.** Que TI añada `lidato.site`. Es lo único
   que garantiza la bandeja en un dominio corporativo.
3. **Que la destinataria responda una vez** el correo: es la señal más fuerte
   para su propio filtro.
4. **Webhooks.** La cuenta no tiene ninguno configurado. Con un endpoint que
   escuche `email.bounced`, `email.complained`, `email.delivery_delayed` y
   `email.suppressed` se detectaría el problema sin esperar a que el cliente
   reclame. Deduplicar por la cabecera `svix-id`.
5. **Volumen.** El correo transaccional «se calienta solo» según Resend, pero un
   dominio nuevo igual necesita rodaje. No sirve de nada mover los envíos a otro
   dominio para esquivar el filtro.

Envía el correo **una sola vez por día**, y solo si el push terminó bien. Si en el
Paso 0 el panel ya estaba al día, no se envía nada.

### Por qué no se manda por Gmail

Se probó de tres formas distintas y las tres fallaron igual: botón con etiqueta de
enlace, dirección como texto suelto dentro del HTML, y correo de puro texto plano.
En los tres casos el conector de Gmail reemplazó la dirección por
`https://www.google.com/url?q=...&source=gmail&ust=...&sa=E`, sin la firma `usg`
que Gmail sí agrega cuando el envío sale de su propia pantalla, y por eso el
destinatario veía una pantalla de advertencia. El conector además borra toda
imagen con `src` externo.

Nada de eso se arregla desde el HTML. Por eso el correo salió de Gmail y se pasó a
Resend. **No vuelvas a intentar arreglarlo cambiando la plantilla.**

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
| Falla el envío por Resend | Avisa el error. **No** reintenta por Gmail y **no** marca la Bitácora |
| El correo salió pero Notion falla | Avisa que falta marcar el día. No reintenta en bucle ni reenvía el correo |
| La fila del ciclo ya tiene la fecha de hoy | Hoy ya estaba marcado. No vuelve a contar |
