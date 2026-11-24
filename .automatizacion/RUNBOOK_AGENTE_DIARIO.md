# Runbook · Agente automático del Panel Diario de Tesorería

Este documento es la instrucción completa para la tarea programada que revisa el
correo, actualiza el panel diario, lo publica y avisa a gerencia.
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

## Paso 5 · El correo a gerencia

**Estado actual: la tarea programada NO envía ni redacta este correo.** No tiene
permiso para escribir correo a terceros sin el dueño presente, y ese permiso se
habilita en la configuración de la cuenta, no cambiando este texto. No intentes
enviarlo ni dejarlo como borrador: la llamada será rechazada y la corrida quedará
a medias.

Lo que sí hace la tarea: en la notificación del Paso 4, incluir al final una línea
exactamente así, para que el dueño lo envíe desde el chat con una sola indicación:

```
CORREO LISTO · Para: tesoreria@incomelec.com · Asunto: Actualización informe tesoreria · Corte: <DD> de <Mes>
```

Cuando el envío automático quede habilitado, esto es lo que hay que ejecutar, y
solo si el Paso 3 terminó bien.

- **Para:** `tesoreria@incomelec.com`
- **Asunto:** `Actualización informe tesoreria`
- **Cuerpo HTML:** el contenido de `.automatizacion/correo-aviso.html`, sustituyendo
  `{{FECHA_CORTE}}` por la fecha del corte en texto (p. ej. `24 de Noviembre`).
- Se envía con la herramienta `send_message` de Gmail, campo `htmlBody`.

Envía el correo **una sola vez por día**. Si en el Paso 0 el panel ya estaba al día,
no se envía nada.

---

## Resumen de la condición de parada

| Situación | Qué hace |
|---|---|
| Panel ya tiene el corte de hoy | Termina en silencio |
| No llegó el correo con el Excel de hoy | Termina en silencio |
| Llegó el Excel | Actualiza, publica, avisa y envía el correo |
| Falla la actualización o el push | Avisa el error y **no** envía el correo a gerencia |
