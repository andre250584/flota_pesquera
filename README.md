# Tablero de la flota pesquera

Tablero web del registro de embarcaciones pesqueras con permiso de pesca — Dirección General de Pesca para Consumo Humano Directo e Indirecto (DGPCHDI), Ministerio de la Producción. Es un único archivo HTML autónomo que se conecta a Google Sheets y se dibuja solo en el navegador.

**Ver en vivo:** https://andre250584.github.io/flota_pesquera/

## Qué muestra

Cinco indicadores en la cabecera: embarcaciones registradas (y armadores únicos), permisos vigentes y su porcentaje de la flota, capacidad de bodega acumulada en m³, potencia instalada en HP y permisos no vigentes.

- **Panorama** — estado del permiso de pesca, material de casco, segmento por eslora, embarcaciones por régimen y arte de pesca (aparejo principal).
- **Evolución** — resoluciones de permiso por año, según la fecha de resolución.
- **Registro** — concentración por armador (top 10) y capacidad de bodega por régimen.

## Fuente de datos

Lee una hoja publicada en la web desde Google Sheets en formato CSV: la hoja **Matriz**, el registro maestro de embarcaciones. La URL publicada está configurada dentro del archivo (constante `URL_MATRIZ`). Al abrir la página la descarga automáticamente; el botón **↻ Actualizar** vuelve a leerla sin recargar.

Una fila = una embarcación, identificada por su `MATRICULA`. Nada está quemado en el código: todos los indicadores y gráficos se calculan en el navegador a partir de las filas del CSV.

> Los cambios que se hagan en la hoja de Google se reflejan en el tablero unos minutos después (tiempo que tarda Google en refrescar la versión publicada).

## Reglas de negocio aplicadas

- **Potencias atípicas excluidas**: los valores de `POTENCIA MOTOR` mayores a 20 000 HP (o iguales o menores a cero) se tratan como error de dato y no suman al total de potencia instalada.
- **Estados de suspensión agrupados**: todas las variantes de "SUSPENDIDO …" se consolidan en un único estado `SUSPENDIDO`.
- **Segmentos por eslora**: menos de 10 m, 10–15, 15–22.9, 23–32.5 y más de 32.5 m. Las esloras vacías o iguales o menores a cero se ignoran.
- **Aparejo principal**: cuando una embarcación declara varias artes de pesca, se contabiliza solo la primera.
- **Evolución acotada**: la serie por año considera únicamente resoluciones entre 1990 y el año en curso.

## Cómo está hecho

Un solo archivo `index.html` con el HTML, el CSS y el JavaScript en línea. Sin build, sin framework y sin backend. Las librerías se cargan por CDN: PapaParse para leer el CSV, Chart.js con el plugin de data labels para los gráficos, y Montserrat desde Google Fonts. Los gráficos de cada pestaña se construyen la primera vez que se abre.

La identidad visual sigue el Manual de Identidad de PRODUCE: rojo `#B72727` como color principal con su rampa de rojos, grises secundarios y tipografía Montserrat.

## Cómo publicarlo (GitHub Pages)

El despliegue está automatizado con GitHub Actions (`.github/workflows/pages.yml`): cada push a `main` publica el sitio.

1. Crea un repositorio **público** y sube el archivo como `index.html` en la raíz.
2. En **Settings → Pages**, elige como *Source* la opción **GitHub Actions**.
3. Haz push a `main`. En un par de minutos el tablero queda disponible en `https://<usuario>.github.io/<repositorio>/`.

Hay que servirlo desde un origen https real: abriendo el archivo directamente con `file://` el navegador bloquea por CORS la descarga del CSV. Para probar en local, `python3 -m http.server 8080` y abrir `http://localhost:8080/`.
