# TeleStats: implementación detallada del estado de estaciones

Este documento explica cómo se construyó el panel **Estado de estaciones**, qué cambios se hicieron en la interfaz y cómo mantenerlo. La plantilla editable es:

```text
/home/julian/telestats-maintenance/ui/index.jade
```

La plantilla instalada se encuentra en:

```text
/usr/local/Apps/telestats/web/index.jade
```

La aplicación Java, el almacenamiento SNMP y la generación de PNG pertenecen al servicio TeleStats. Esta personalización solo consume el endpoint HTTP `/data` y presenta el resultado.

## 1. Objetivo del panel

El panel permite saber rápidamente si las estaciones tienen telemetría reciente. Cada estación se representa con una tarjeta que contiene:

- Nombre visible de la estación.
- Punto verde o rojo.
- Texto explicativo.
- Hora de la última evaluación.
- Enlaces a sus gráficas al hacer clic.

La pantalla inicial se identifica como **Estado de estaciones** y no carga una troncal ni una gráfica por defecto. Al elegir un grupo de monitoreo, el panel de estaciones se oculta y se muestran las gráficas del grupo elegido.

## 2. Estructura de la plantilla

`index.jade` combina tres partes:

1. **Marcado Jade:** selector de red, panel de estaciones, panel de gráficas, filtro y ventana ampliada.
2. **Estilos CSS:** tarjetas, puntos de estado, resumen, botones y diseño adaptable.
3. **JavaScript del navegador:** catálogo, comprobaciones HTTP, contador, filtros y ventanas de gráficas.

La plantilla no modifica el JAR ni los CSV. Solo cambia lo que el navegador muestra.

## 3. Selector y pantalla inicial

La opción del selector usa el valor interno `__none__` y el texto visible `Estado de estaciones`:

```jade
option(value='__none__' selected=(context.selectedTrunk == null)) Estado de estaciones
```

El valor interno se conserva para que el servidor pueda distinguir la pantalla inicial de una troncal real. Al enviar el formulario con ese valor, JavaScript cancela el envío y vuelve a `/`.

El panel de estaciones se muestra cuando `context.selectedTrunk == null`. Los paneles de gráficas y el mensaje para seleccionar una troncal se ocultan en ese caso.

## 4. `stationCatalog`: fuente de las tarjetas

El catálogo está definido en JavaScript como una lista de entradas:

```javascript
['Nombre', ['Grafica A.png', 'Grafica B.png']]
```

Cada entrada puede tener una tercera lista:

```javascript
['Nombre', ['Todas las gráficas.png'], ['Solo estas deciden el estado.png']]
```

Los elementos tienen este significado:

- Posición 0: nombre de la estación mostrado en la tarjeta.
- Posición 1: gráficas que se abren en la ventana de la estación.
- Posición 2: gráficas usadas exclusivamente para decidir verde/rojo.

Actualmente el catálogo contiene estaciones sismológicas, GNSS, radios, conversores, cámaras y enlaces PTZ. Entre ellas están Diamante, Santo Domingo, Cielo Roto, Quindío, Moral, Moral GNSS, Alejandría, Aguas Calientes, Camión, Gualí, Rubí, Rubí GNSS, Celandia, San Julian, Manizales, Desquite, Cerro Bravo, Aerocivil, La Cruz, Santa Marta, Nereidas, Nereidas 2, Olleta, Pitayo, Azufrado, Herveo, Bruma, Primavera, Cisne, Paramillo, África, Azufrera, Otún GNSS, Destierro, Guali4, Laguna, La Secreta, BIS, Río Claro, Balcones, Recio, Recio 3, Nido de Águila, Esmeralda, Laguna Verde, Totarito, Olleta 2, Alfombrales BA, Alfombrales Doas, Inclinómetro de Santa Isabel, Magnetómetro de Alfombrales, Molinos 1, La Siberia, Cerro Bravo CP, Aguacatal, El Águila, Glaciar, Gualí 3, Guali PTZ, Olleta PTZ, Desquite PTZ, Coralito PTZ, C. Bravo PTZ, Lisa, Inderena, Billar, CAM Lagunillas, Lagunillas Flujo de Lodos, Anillo, San Juan, Lajas, Cima, Toche, La Palma, Silencio, Rodeo, Tapias, San Lorenzo, Santa Ana, Magnetómetro Domo y Estatuas.

Los ePever no se agregan como estaciones porque son equipos auxiliares. Sus gráficas continúan en los grupos de monitoreo.

## 5. `buildHomeStations()`: creación de tarjetas

Esta función se ejecuta al cargar la página. Si no hay una troncal seleccionada, recorre `stationCatalog` y crea un elemento `article` por estación.

Para cada tarjeta establece:

- Clase CSS `station-card`.
- Accesibilidad con `tabindex` y `role="button"`.
- `data-station`: nombre de la estación.
- `data-graphs`: lista completa separada por `|`.
- `data-status-graphs`: lista de evaluación si existe.
- Un punto sin estado inicial.
- El texto `Comprobando telemetría…`.

Los atributos `data-*` permiten que las demás funciones trabajen sin volver a consultar el catálogo.

## 6. `displayName()`: nombres visibles

`displayName(name)` transforma únicamente el texto presentado. No cambia el nombre del archivo, el dispositivo SNMP ni la URL de datos.

Cambios principales:

| Original | Texto visible |
|---|---|
| Alguacil | Guali4 |
| Manizales - Desquite C5X | Manizales PTZ - Desquite PTZ |
| Desquite - Manizales C5X | Desquite PTZ - Manizales PTZ |
| Desquite - C. Bravo C5X | Desquite PTZ - C. Bravo PTZ |
| C. Bravo - Desquite C5X | C. Bravo PTZ - Desquite PTZ |
| C. Bravo - Coralito C5X | C. Bravo PTZ - Coralito PTZ |
| Coralito - Cerro Bravo C5X | Coralito PTZ - Cerro Bravo PTZ |
| Guali - Manizales C5X | Guali PTZ - Manizales PTZ |
| Manizales - Guali C5X | Manizales PTZ - Guali PTZ |
| Manizales - Olleta C5X | Manizales PTZ - Olleta PTZ |
| Olleta - Manizales C5X | Olleta PTZ - Manizales PTZ |

Los reemplazos se aplican a títulos, nombres de tarjetas, filtros y ventanas emergentes.

## 7. `checkStations()`: evaluación de telemetría

`checkStations()` es la función principal del estado. Se ejecuta al cargar la página y luego mediante:

```javascript
window.setInterval(checkStations, 60000);
```

El intervalo es de 60 segundos.

Para cada tarjeta realiza estos pasos:

1. Quita las clases anteriores `ok` y `fail`.
2. Lee `data-status-graphs`; si no existe, usa `data-graphs`.
3. Divide la lista en nombres individuales.
4. Crea una consulta por cada gráfica:

```text
/data?device=NOMBRE_SIN_.png&count=3
```

5. Solicita los datos con `fetch(..., {cache: 'no-store'})`.
6. Rechaza respuestas HTTP no exitosas.
7. Divide el cuerpo en filas y elimina filas vacías.
8. Ignora la primera columna de fecha y la última columna de latencia para validar que las columnas de parámetros sean numéricas.
9. Busca la marca de tiempo más reciente.
10. Calcula su antigüedad.
11. Cuando terminan todas las consultas de la tarjeta, asigna el color y el mensaje.

Se usan dos contadores internos:

- `pending`: consultas aún pendientes.
- `failed` y `stale`: banderas de error o datos atrasados.

Una consulta fallida marca `failed`. Una fecha inválida también marca error. Una fecha con más de 15 minutos marca `stale`.

## 8. Significado de los estados

- `ok` + **Telemetría actualizada**: todas las gráficas seleccionadas tienen datos numéricos recientes.
- `fail` + **Telemetría no disponible**: una consulta falló, no devolvió filas o los valores no son numéricos.
- `fail` + **Telemetría atrasada**: hay datos, pero la muestra más reciente supera 15 minutos.
- `fail` + **Sin telemetría configurada**: la entrada no tiene gráficas.

El color no prueba que todos los sensores de una estación estén bien. Solo evalúa los enlaces incluidos en su lista de estado.

## 9. `updateStationSummary()`: contador superior

Esta función cuenta el DOM actual:

```javascript
var cards = document.querySelectorAll('.station-card');
var working = document.querySelectorAll('.station-card .status-dot.ok').length;
var failed = document.querySelectorAll('.station-card .status-dot.fail').length;
```

Después escribe:

```text
Total: N · Funcionando: V · Con problemas: R · Actualizado: HH:MM:SS
```

Se llama al terminar cada tarjeta y al terminar el recorrido inicial. Por eso el contador puede empezar en cero y completarse progresivamente durante la primera ronda.

## 10. Excepciones de evaluación configuradas

La segunda lista limita qué enlaces deciden el color, sin impedir que las demás gráficas se abran:

- Pitayo: solo enlaces con Cerro Bravo.
- Paramillo: solo Manizales–Paramillo.
- Cerro Bravo: solo enlaces con Desquite.
- África y Azufrera: enlaces con Paramillo.
- Recio: `Recio - CISNE`.
- Olleta 2: enlaces con Manizales.
- Inclinómetro de Santa Isabel: `Inclinometro Santa Isabel - OLLETA`.
- CAM Lagunillas: se excluye Pitayo–Lagunillas.
- Herveo: `Herveo - CERRO BRAVO`.
- La Palma: se excluyen `Toche - LA PALMA` y `Cima - LA PALMA`.
- Anillo: solo `Anillo - LA SECRETA`.

Para cambiar una excepción se edita la tercera lista de su entrada, no la función `checkStations()`.

## 11. Gráficas excluidas

Las condiciones Jade de la lista y del panel de imágenes ocultan, sin borrar del servidor:

- `Billar - PITAYO`.
- `Pitayo - Billar`.
- `Alguacil - MANIZALES`.
- `MANIZALES - Alguacil`.
- `Pitayo - C. Bravo C5X`.
- `Pitayo - Coralito C5X`.

La comparación elimina `.png` y convierte a minúsculas para evitar errores por diferencias de escritura.

## 12. Ventana de estación y gráficas

`openStationWindow(card)` lee `data-station` y `data-graphs`, abre una ventana y genera un documento HTML independiente. Para cada archivo:

1. Codifica el nombre para la URL.
2. Conserva las barras necesarias.
3. Construye la ruta `graphs/<archivo>`.
4. Presenta el título mediante `displayName()`.
5. Inserta la imagen en una cuadrícula adaptable.

Si el navegador bloquea ventanas emergentes, muestra un aviso al operador.

## 13. Filtro de gráficas

`filter()` toma el texto del buscador y compara tanto el nombre original como el nombre visible transformado por `displayName()`. Oculta las tarjetas que no coinciden, actualiza el contador y muestra el mensaje de lista vacía cuando corresponde.

`clear-filter` borra el texto y vuelve a mostrar todas las gráficas. La tecla `/` enfoca el filtro y `Escape` cierra la ventana ampliada.

## 14. Cómo agregar una estación correctamente

1. Liste los PNG reales:

```bash
find /usr/local/Apps/telestats/web/graphs -maxdepth 1 -type f -name '*.png' -printf '%f\n' | sort
```

2. Confirme el dispositivo que atiende `/data`:

```bash
curl -sS --get http://127.0.0.1:8081/data \
  --data-urlencode 'device=NOMBRE_SIN_EXTENSION' \
  --data-urlencode 'count=3'
```

3. Añada la entrada a `stationCatalog` usando el nombre exacto:

```javascript
,['Nueva estación', ['Enlace A.png', 'Enlace B.png']]
```

4. Si un enlace no debe decidir el color, use una tercera lista:

```javascript
,['Nueva estación',
  ['Todos los enlaces.png'],
  ['Enlace fiable para estado.png']]
```

5. No coloque ePever, switches o cámaras como estación salvo que ese sea el criterio operativo.
6. Compruebe que no exista otra entrada con el mismo nombre.
7. Valide e instale la plantilla.

## 15. Cómo excluir o renombrar una gráfica

Para excluirla de los grupos renderizados, añada su nombre a las dos condiciones Jade (`station-panel` y `graphs`). Para ocultarla también en una ventana de estación, retírela de la lista `data-graphs` de esa estación.

Para cambiar solo el texto, agregue una sustitución a `displayName()`:

```javascript
.replace(/Nombre original/gi, 'Nombre mostrado')
```

No renombre el PNG ni el dispositivo salvo que también actualice el árbol de red, el histórico y la plantilla correspondiente.

## 16. Publicación de cambios

Desde el directorio de mantenimiento:

```bash
cd /home/julian/telestats-maintenance
bash -n install-ui.sh
sudo ./install-ui.sh
```

`install-ui.sh`:

1. Define la aplicación en `/usr/local/Apps/telestats`.
2. Crea un respaldo de `web/index.jade` bajo `/var/backups/telestats-ui-*`.
3. Instala la plantilla con permisos de lectura.
4. Detiene y arranca TeleStats.
5. Reactiva la supervisión Monit.

Después de publicar, recargue el navegador con `Ctrl+F5`. Espere hasta 60 segundos para completar una ronda de comprobaciones.

## 17. Verificación después del cambio

```bash
sudo monit status telestats
sudo monit status telestats-health
sudo /usr/local/Apps/telestats/launcher.sh status
curl -fsS http://127.0.0.1:8081/
```

Para una tarjeta concreta revise el endpoint `/data`. Si devuelve vacío, el estado rojo es correcto y el problema está en SNMP, el nombre del dispositivo, el CSV o la generación de datos; no en el contador.

## 18. Problemas frecuentes

**La página dice que no se pudo generar.** Revise la sintaxis Jade, especialmente las expresiones `if`; instale una versión anterior desde `/var/backups/telestats-ui-*` si el servicio no inicia.

**Una gráfica aparece aunque fue excluida.** Verifique mayúsculas, `.png` y que la exclusión esté en las dos condiciones Jade. Publique y use `Ctrl+F5`.

**Una tarjeta aparece roja.** Ejecute `/data` con el nombre exacto sin `.png`, revise la fecha de la última fila y consulte `telestats.log`.

**Una tarjeta aparece verde aunque falta un enlace.** Revise la tercera lista: solo los enlaces incluidos allí participan en el color.

**El contador no coincide.** El total cuenta tarjetas del DOM, no dispositivos del árbol SNMP. Revise duplicados en `stationCatalog`.

**La ventana muestra una imagen rota.** Compare el nombre exacto del PNG en `web/graphs`; Linux diferencia mayúsculas, minúsculas, espacios y tildes.

## 19. Reversión

El instalador conserva la plantilla anterior. Para revertir:

```bash
sudo cp -a /var/backups/telestats-ui-AAAAmmdd-HHMMSS/index.jade \
  /usr/local/Apps/telestats/web/index.jade
sudo /usr/local/Apps/telestats/launcher.sh stop
sudo /usr/local/Apps/telestats/launcher.sh start
sudo monit reload
sudo monit monitor telestats
```

Conserve el respaldo hasta verificar que los estados, gráficas y grupos vuelven a funcionar.
