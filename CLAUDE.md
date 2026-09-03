# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Dashboard Flota Pesquera — DGPCHDI (PRODUCE)

Tablero web de una sola página del registro de embarcaciones pesqueras con permiso de pesca.
Hermano del dashboard de plantas pesqueras. Trabajamos **un cambio a la vez**.

## Archivo y arquitectura
- Todo vive en un **único archivo**: `index.html` (HTML + CSS + JS en línea). No hay build, ni framework, ni backend, ni tests, ni gestor de paquetes.
- Librerías por CDN (cdnjs): PapaParse 5.4.1, Chart.js 4.4.1 y chartjs-plugin-datalabels 2.2.0. Montserrat por Google Fonts.
- Los datos se leen **en vivo** con `fetch()` y se procesan con PapaParse; todos los KPIs y gráficos se calculan en JavaScript a partir de las filas. **Nada de datos quemados/hardcodeados.**

## Comandos
- No hay build/lint/test. Editar el HTML es todo el ciclo de desarrollo.
- Previsualizar: `python3 -m http.server 8080` y abrir `http://localhost:8080/`. Abrirlo con `file://` no sirve.
- El archivo **debe llamarse `index.html`**: es lo que GitHub Pages sirve en la raíz del sitio. No lo renombres.
- Desplegar: `git push` a `main` en `andre250584/flota_pesquera` republica solo, vía `.github/workflows/pages.yml`. Sitio: https://andre250584.github.io/flota_pesquera/
- Probar siempre en la URL `*.github.io`, no en preview local.

## Fuente de datos
- CSV publicado de Google Sheets (hoja `Matriz`), en la constante `URL_MATRIZ` al inicio del `<script>`.
- Para cambiar la fuente o sumar la hoja de historial (`Crudo`), edita esa constante / agrega otra; no dupliques la lógica de fetch.
- La columna de embarcación es `MATRICULA`; una fila = una embarcación. Las filas sin `MATRICULA` se descartan al parsear.
- Columnas que consume el código: `MATRICULA`, `PERMISO PESCA`, `CASCO`, `ESLORA`, `REGIMEN`, `APAREJO`, `ARMADOR`, `CAPBOD_M3`, `POTENCIA MOTOR`, `FECHA RESOLUCION`, `INC. DEF` (columna U; el punto y el espacio del nombre son parte del encabezado), `ESPECIE CHD VIGENTES` (columna Z), `ESPECIE CHI VIGENTES` (columna AB). Renombrar una columna en la hoja rompe el gráfico o KPI correspondiente en silencio (queda `(sin dato)` o 0).

## Flujo de ejecución
1. `cargar()` — se llama al final del script y desde el botón «↻ Actualizar». Añade `?t=Date.now()` al URL y usa `cache:'no-store'` para evitar el CSV cacheado; llama a `destroyAll()` antes de recargar.
2. PapaParse con `header:true` → `DATA` (array global de filas, el CSV crudo).
3. `poblarPaneles()` arma las casillas de los filtros a partir de `DATA`.
4. `render()` → `aplicarFiltro()` (→ `VISTA`) + `kpis()` + `build(pestanaActiva())` + `pintarFiltros()`.
   Construye la pestaña visible, **no `'panorama'` fijo**: al filtrar se puede estar en cualquiera.
5. Los gráficos se construyen **perezosamente por pestaña**: `build(v)` corta si `BUILT[v]`, y cada instancia de Chart.js se guarda en el registro `CHARTS` para poder destruirla en la recarga.
- Para agregar un gráfico: canvas nuevo en la vista + una rama dentro de `build()` usando los helpers `bars(id,labels,data,color,horizontal)` o `donut(id,labels,data,colors)`, que ya traen la paleta, los data labels y el formato es-PE.
- Helpers de agregación reutilizables: `conteo(campo, mapfn)`, `ordenar(obj, topN)`, `num()` (limpia comas y devuelve `null` si no es número), `fmt()` (miles es-PE), `shorten()` (abrevia los nombres largos de `REGIMEN` para las etiquetas).
- **Las agregaciones leen `VISTA`, nunca `DATA`.** `DATA` es el CSV crudo; `VISTA` son las filas que
  pasan los filtros. Un gráfico nuevo que lea `DATA` se queda mudo ante los filtros y nadie se entera.

## Filtros
- Barra entre las tarjetas KPI y las pestañas, con dos desplegables de casillas (selección múltiple):
  **Régimen de pesca** y **Estado del permiso**. El recorte es global: alcanza los 5 KPIs, el contador
  de la cabecera y los gráficos de las tres pestañas.
- Los dos comparten un solo mecanismo declarativo, el objeto `FILTROS`. Cada entrada dice de qué
  columna sale su valor (`valor`), cómo se etiqueta (`etiqueta`), en qué orden se listan sus casillas
  (`orden`, o `null` para ordenar por frecuencia), su selección inicial (`def`) y el prefijo de sus
  ids en el HTML (`pre`). **Para sumar un tercer filtro basta otra entrada aquí más su bloque en el
  HTML** — no dupliques `poblarPaneles()`, `refrescar()` ni `pintarFiltros()`.
- **Un conjunto vacío significa "todos", no "nada".** Así no existe el tablero en cero ni la división
  por cero de `kpis()` (`vig/total`). No lo inviertas.
- El estado arranca en **VIGENTE + SUSPENDIDO** (la flota operativa), así que el tablero **no abre
  mostrando las 2,234 embarcaciones** sino 1,511. Compara sobre `estadoPermiso()`, no sobre el valor
  crudo, o las variantes de "SUSPENDIDO …" quedan fuera de su propia casilla.
- Las casillas se arman desde `DATA`, **nunca** desde `VISTA`: si salieran de la vista filtrada, las
  demás opciones desaparecerían al marcar la primera.
- «Restablecer» devuelve la página a como carga (régimen vacío, estado en su defecto), no a "sin
  filtros"; se oculta cuando ya está en ese estado. Al recargar sobreviven las marcas cuyo valor siga
  existiendo en los datos frescos.
- El check **«Excluir INC. DEF»** saca del reporte las embarcaciones con `INC. DEF = SI` (187 hoy, y
  todas suspendidas: al marcarlo el defecto pasa de 1,511 a 1,324 sin tocar las vigentes). Va **fuera**
  de `FILTROS` — es un sí/no, no una lista de categorías — como el booleano `EXCL_INC` más el
  predicado `esIncDef()`; se combina con los otros filtros en `aplicarFiltro()`. Arranca marcado y
  «Restablecer» lo deja activado.
- Consecuencia esperada y aceptada: con un filtro puesto, «Embarcaciones por régimen» y «Capacidad de
  bodega por régimen» muestran solo las barras seleccionadas. No es un defecto.

## Reglas de negocio ya implementadas (no las rompas)
- `POT_MAX = 20000`: potencias por encima (o ≤ 0) se tratan como error de dato y se excluyen del total de HP.
- Estado del permiso: todas las variantes de "SUSPENDIDO …" se agrupan como `SUSPENDIDO` (`estadoPermiso()`); el orden fijo en el dona es VIGENTE · SUSPENDIDO · CANCELADO · ANULADO, el mismo que usan las casillas del filtro de estado.
- Fecha: `FECHA RESOLUCION` se parsea en formato `M/D/AAAA` y `AAAA-MM-DD` (`parseAnio()`); la evolución solo cuenta años entre 1990 y el año actual.
- Segmentos de eslora: <10 / 10-15 / 15-22.9 / 23-32.5 / >32.5 m; se ignoran esloras nulas o ≤ 0.
- `APAREJO` puede traer varios valores separados por `;`: solo se usa el primero (aparejo principal), top 8.
- Armadores: top 10, etiquetas truncadas a 26 caracteres.

## Identidad visual PRODUCE (Manual de Identidad) — obligatoria
- Color principal: **rojo `#B72727`**. Manda en cabecera, pestañas y los controles de filtro.
- Grises secundarios: texto `#5E5446`, `#ABA290`; fondos `#E3E2D8`, `#E0DBD7`.
- Tipografía **Montserrat** en todo (títulos Bold, cuerpo Regular).
- Las barras siempre muestran su valor (data labels).
- Cabecera con lockup **PRODUCE / Ministerio de la Producción** en blanco sobre rojo y el recurso gráfico de diagonales (pleca).

## Paleta de datos (decidida el 1 de septiembre de 2026)
El tablero era rojo monocromo y se veía plano. Ahora los gráficos usan acentos, igual que el
dashboard de plantas. **El rojo sigue siendo la identidad; el color en los gráficos es información.**
- Acentos: teal `#0891B2`, ámbar `#D97706`, verde `#047857`, violeta `#6D28D9`, pizarra `#475569`.
- Orden categórico fijo: `PAL = [ROJO, TEAL, AMBAR, VERDE, VIOLETA]`. **El orden es la garantía de
  daltonismo** (contiguos separados bajo protanopia/deuteranopia, ≥3:1 sobre panel blanco). No
  reordenar ni cambiar los hex sin volver a validar la paleta.
- Cada color hace un solo trabajo: `COLOR_ESTADO` es escala de estado (vigente→anulado, el color dice
  gravedad); `RAMPA_ESLORA` es una sola tinta de claro a oscuro porque los tramos van en orden;
  `colorCat()` reparte identidad. Las categorías sin información van siempre a los grises (`GRISES`).
- **Una serie = un color.** Los gráficos de barras de serie única (régimen, aparejo, armador,
  capacidad) llevan un solo tono cada uno, no un arcoíris por barra — la longitud ya codifica el
  valor. Régimen y capacidad-por-régimen comparten teal por ser la misma dimensión.
- Las cifras de los KPIs van en **negro**; el color de lo que mide cada tarjeta vive en su borde
  superior (y en el porcentaje de «Permiso vigente»), no en el número.
- Los colores existen dos veces: variables CSS en `:root` y constantes JS para Chart.js. Tocar ambos.
- `tintaSobre()` decide blanco o tinta oscura en las etiquetas dentro de la dona según el relleno; no
  poner blanco fijo. Las porciones bajo 3.5% no llevan etiqueta (no cabe, la lee el tooltip).

## Estructura del tablero
- Pestañas: **Panorama · Evolución · Reportes**. **No hay mapa** (decisión tomada; no agregar uno).
- Panorama concentra los ocho gráficos; Evolución tiene la serie por año; **Reportes** son cuadros de
  cifras (tablas `.tabla`, no gráficos), así que su rama de `build()` no toca `CHARTS`.

## Reportes (pestaña)
- «Flota pesquera por régimen y tipo de acceso» cruza régimen (filas) × especie (columnas), con fila
  TOTAL. `Anch. CHI` sale de `ESPECIE CHI VIGENTES`; el resto de las columnas, de `ESPECIE CHD
  VIGENTES`. Los regímenes fuera de `ORDEN_REG` se agregan al final en vez de desaparecer: el reporte
  oficial en papel solo lista cinco, pero el cuadro no puede esconder embarcaciones.
- Las cifras del cuadro reproducen el reporte oficial con diferencias de 1 a 3 embarcaciones, porque
  la hoja es viva. El recorte que lo reproduce es justamente el de arranque: vigentes + suspendidas,
  sin `INC. DEF`.
- **La fila TOTAL lleva `:not(.total)` en la regla de cebreado.** Sin eso, `:nth-child(even)` le gana
  en especificidad al fondo oscuro y la deja en blanco sobre casi blanco — y solo cuando cae en
  posición par, o sea según cuántos regímenes traiga el filtro. Es un error que se esconde solo.
- «Menor escala por especie» cuenta embarcaciones de menor escala que llevan una especie entre sus
  CHD vigentes: Anchoveta (`ANCH`), Bacalao (`BAC`), Anguila (`ANGL`) y Merluza (`MERLZ`). Crece
  agregando una fila más al arreglo de la rama `build('reportes')`.
- Menor escala son las **tres** variantes de `REGIMEN` (`MENOR ESCALA`, `... (ANCHOVETA)` y
  `... - ARTESANAL`), por eso `esMenorEscala()` compara con `includes('MENOR ESCALA')`.
- Las columnas de especie son listas separadas por `/` (ej. `AB/ANCH/R.H.`). `conEspecie()` compara
  el **código completo**, nunca por subcadena: la columna tiene códigos que contienen a otro, y buscar
  el trozo los mezcla. `PER` (perico) convive con `PERL`, `ANG` con `ANGL`, y `MERLZ` (merluza) con
  `MERLI` y `MERO`. Cuando esto se hacía con `includes()`, perico contaba 1 embarcación de más.
- El cuadro lee `VISTA`: responde a los filtros como el resto del tablero. Con el arranque por defecto
  (vigentes + suspendidas, sin `INC. DEF`) da 338 · 9 · 19 · 1; sobre las 2,234 sin filtrar,
  352 · 11 · 28 · 1.

## Trampa conocida (no es un bug que arreglar)
- En algunos entornos de previsualización local, el `fetch` al CSV falla por **CORS**. El código ya detecta ese caso y muestra un mensaje claro en `#errbox`.
- Se resuelve solo al servir desde un **origen https real** (GitHub Pages). No cambies el mecanismo de fetch para "arreglar" esto.
