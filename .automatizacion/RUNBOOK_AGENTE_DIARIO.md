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

1. `PANEL DIARIO/index.html` ← el panel nuevo.
2. La carpeta de la semana en curso, que debe quedar idéntica al panel diario.
   Hoy es `NOVIEMBRE/TERCERA SEMANA/index.html` (semana del 19 nov).
   Al cambiar de semana, usa la carpeta que corresponda al corte
   (`NOVIEMBRE/CUARTA SEMANA/`, luego `DICIEMBRE/PRIMERA SEMANA/`, etc.).
   Si la carpeta solo tiene la página de aviso, reemplázala por el panel.
3. En el `index.html` de la raíz, dentro de `window.PORTAL.informes`, pon la fecha
   del corte en el campo `actualizado` de la entrada `id: 'panel'`
   (formato `'AAAA-MM-DDTHH:MM'`) y también en la entrada del mes en curso si se
   actualizó su carpeta semanal.

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

## Paso 5 · Enviar el correo a gerencia

Solo si el Paso 3 terminó bien.

- **Para:** `josefelixlizalde@incomelec.com`
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
