# INCOMELEC · Portal de Informes de Tesorería

Sitio estático (HTML autocontenido, sin dependencias ni build) con el portal de
informes de tesorería de INCOMELEC, SUPLEC y PARQUEOO.

## Estructura

```
index.html                                  Portal de informes (página de entrada)
PANEL DIARIO/                               Panel diario vigente (semana del 19 nov)
SEPTIEMBRE/                                 Índice del mes
  PRIMERA SEMANA/   · corte 8 sep
  SEGUNDA SEMANA/   · corte 15 sep
  TERCERA SEMANA/   · corte 22 sep
  CUARTA SEMANA/    · 30 sep – 6 oct
OCTUBRE/
  PRIMERA SEMANA/   · 7 – 14 oct
  SEGUNDA SEMANA/   · desde 15 oct
  TERCERA SEMANA/   · desde 22 oct
  CUARTA SEMANA/    · desde 28 oct
NOVIEMBRE/
  PRIMERA SEMANA/   · sin informe publicado (página de aviso)
  SEGUNDA SEMANA/   · desde 12 nov
  TERCERA SEMANA/   · desde 19 nov
  CUARTA SEMANA/    · sin informe publicado (página de aviso)
GUIA DE CUADRO DE MANDO DE TESORERIA/       Manual de usuario del cuadro de mando
```

## Cómo actualizar

1. **Agregar una semana**: reemplaza el `index.html` de la carpeta que corresponda
   (por ejemplo `NOVIEMBRE/CUARTA SEMANA/index.html`). El nombre del archivo
   siempre debe ser `index.html` para que la carpeta abra sola.
2. **Actualizar el panel diario**: reemplaza `PANEL DIARIO/index.html`.
3. **Fechas del portal**: el portal lee la fecha de cada informe en este orden:
   - la etiqueta que publique el propio informe en su `<head>`:
     `<meta name="lidato:actualizado" content="2026-11-23T09:00-05:00">`
   - el campo `actualizado` del bloque `window.PORTAL` dentro de `index.html`
   - la fecha de modificación que entregue el servidor
4. **Agregar un mes**: copia una carpeta de mes existente, cambia el título y el
   texto, y añade el informe al arreglo `informes` de `window.PORTAL` en `index.html`.

El archivo `.nojekyll` evita que GitHub Pages procese el sitio con Jekyll, para que
las carpetas con espacios y mayúsculas se sirvan tal cual.
