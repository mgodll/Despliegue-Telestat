# TeleStats — despliegue desde cero y referencia técnica

Esta guía explica cómo reconstruir, instalar, configurar y operar la instancia de TeleStats usada en `tororoi`. El proyecto de mantenimiento contiene configuración y scripts; el programa principal es el JAR Java.

## 1. Qué hace TeleStats

TeleStats recorre un árbol de equipos de red, consulta sus valores por SNMP, escribe muestras en CSV, genera imágenes PNG con Gnuplot y las publica mediante un servidor HTTP. El navegador nunca consulta directamente los radios: solicita a TeleStats los datos o las imágenes ya generadas.

Flujo completo:

```text
network/trunks/*.json
        │ árbol de equipos e IP
        ▼
  Poller SNMP ──► data/*.csv y web/temp/*
                         │
                         ▼
             Plotter + network/templates/*.plot
                         │
                         ▼
                    web/graphs/*.png
                         │
                         ▼
                   HTTP :8081
```

Cada dispositivo tiene un `model`. El modelo selecciona los OID SNMP y la plantilla `.plot` selecciona las columnas, unidades, escalas y colores de la gráfica.

## 2. Directorios en producción

La aplicación instalada está en `/usr/local/Apps/telestats`.

| Carpeta/archivo | Función |
|---|---|
| `telestats.jar` | Programa Java principal. Ejecuta sondeo, almacenamiento, gráficas y HTTP. |
| `conf/config.json` | Parámetros globales de ejecución. |
| `conf/logback.xml` | Niveles y destino de logs Java. |
| `network/trunks/` | Árboles JSON de cada red o troncal. |
| `network/models/` | Definición de modelos SNMP y sus parámetros. |
| `network/templates/` | Plantillas Gnuplot por modelo o enlace. |
| `data/` | Histórico CSV generado por el poller. |
| `web/temp/` | Datos de trabajo usados por el plotter. |
| `web/graphs/` | PNG publicados por el servidor web. |
| `logs/telestats.log` | Polling, respuestas SNMP y errores de dispositivos. |
| `logs/launcher.log` | Salida del proceso Java y mensajes de Gnuplot. |
| `launcher.sh` | Arranque, parada y consulta del PID. |
| `health.sh` | Comprobación externa de salud. |
| `cleanup.sh` | Limpieza controlada de temporales. |

La copia editable está en `/home/julian/telestats-maintenance`.

## 3. Archivos del proyecto de mantenimiento

- `config.json`: copia que se instala como `conf/config.json`.
- `monit-telestats`: reglas de Monit para el proceso y el health check.
- `launcher.sh`: script de ciclo de vida; sus funciones son leer el PID, comprobar `/proc`, arrancar Java con `nohup`, detener con `TERM` y reportar estado.
- `health.sh`: comprueba PID, puerto HTTP y antigüedad de PNG.
- `cleanup.sh`: elimina temporales antiguos sin borrar históricos PNG.
- `install.sh`: instalador general, crea respaldo, valida Monit, reinicia y monitoriza.
- `install-ui.sh`: publica `ui/index.jade` y reinicia solo para activar la interfaz nueva.
- `ui/index.jade`: plantilla de la interfaz, catálogo de estaciones, filtros, estados y ventanas de gráficas.
- `install-radio-links.sh`: instala redes/modelos/plantillas de FreeWave y enlaces modificados.
- `install-af5xhd-lacruz.sh`: instala configuraciones de radios Ubiquiti AF-5XHD.
- `install-rut200.sh`: instala la configuración del Teltonika RUT200.
- `install-zumlink-nereidas2.sh`: instala el enlace Zumlink de Nereidas 2.
- `install-moral-voltage.sh`: instala ajustes de voltaje de la red Moral.
- Los archivos `*.json` y `*.plot` de la raíz son fuentes de modelos, redes y plantillas para esos instaladores.

## 4. Requisitos de un servidor nuevo

Se necesita Linux con:

- Java compatible con el JAR.
- Monit 5.x.
- `curl`, `find`, `sort` y utilidades POSIX.
- `gnuplot` con terminal `pngcairo`.
- Cliente SNMP (`snmpwalk` es útil para pruebas).
- Acceso IP desde el servidor a los equipos y puertos SNMP permitidos.

El JAR debe provenir de una compilación verificada. El repositorio no contiene el código interno del ejecutable en una forma que permita sustituir sus dependencias automáticamente; por eso se conserva el JAR probado en `maintenance/bengie.jar` solo para Bengie y el JAR de TeleStats debe copiarse desde la distribución autorizada.

## 5. Crear la estructura desde cero

Ejecute como root en el servidor destino:

```bash
sudo mkdir -p /usr/local/Apps/telestats/{conf,network/{trunks,models,templates},data,web/{temp,graphs},logs/archived}
sudo chown -R julian:julian /usr/local/Apps/telestats
```

Copie los elementos de una distribución probada:

```text
/usr/local/Apps/telestats/telestats.jar
/usr/local/Apps/telestats/conf/config.json
/usr/local/Apps/telestats/conf/logback.xml
/usr/local/Apps/telestats/network/trunks/*.json
/usr/local/Apps/telestats/network/models/*
/usr/local/Apps/telestats/network/templates/*
```

Instale los scripts con permisos adecuados:

```bash
sudo install -o julian -g julian -m 0755 launcher.sh /usr/local/Apps/telestats/launcher.sh
sudo install -o root -g root -m 0700 health.sh /usr/local/Apps/telestats/health.sh
sudo install -o root -g root -m 0755 cleanup.sh /usr/local/Apps/telestats/cleanup.sh
sudo install -o root -g root -m 0700 monit-telestats /etc/monit/conf-enabled/telestats
```

## 6. Configuración global

`config.json` actual:

```json
{
  "interface": "172.16.10.5",
  "webPort": 8081,
  "community": "ovsm",
  "pollingPeriod": 600,
  "requestGap": 250,
  "timeout": 3000,
  "retries": 0,
  "poolSize": 5,
  "plottingPeriod": 30,
  "plottingWindow": 5,
  "maxPoints": 105120
}
```

Funciones de cada opción:

- `interface`: dirección donde se publica la interfaz.
- `webPort`: puerto HTTP.
- `community`: comunidad SNMPv2c usada en las consultas.
- `pollingPeriod`: intervalo de sondeo en segundos.
- `requestGap`: espera entre solicitudes SNMP, en milisegundos.
- `timeout`: tiempo máximo de una solicitud SNMP, en milisegundos.
- `retries`: reintentos SNMP.
- `poolSize`: cantidad de tareas de consulta concurrentes.
- `plottingPeriod`: frecuencia de generación de imágenes.
- `plottingWindow`: ventana de datos usada para cada renderizado.
- `maxPoints`: máximo de muestras conservadas por serie.

## 7. Cómo funciona una red (`trunks/*.json`)

Cada archivo describe una troncal:

```json
{
  "name": "Red ejemplo",
  "rootDevice": {
    "name": "MANIZALES - Desquite C5X",
    "ip": "172.16.42.12",
    "model": "mimosa-c5x",
    "children": []
  }
}
```

`name` es el texto del grupo de monitoreo. `rootDevice` inicia el árbol. Cada dispositivo tiene:

- `name`: identificador exacto usado en CSV, URL `/data` y nombre de PNG.
- `ip`: dirección de administración.
- `model`: nombre del archivo de modelo sin `.json`.
- `children`: siguientes saltos de la red.

El recorrido visita el dispositivo y después sus hijos. El nombre es sensible a mayúsculas, espacios, tildes y signos; una diferencia produce archivos separados o gráficas sin datos.

## 8. Cómo funciona un modelo SNMP

Un modelo, por ejemplo `network/models/zumlink.json`, define el tipo de gestión y los parámetros:

```json
{
  "name": "zumlink",
  "managementType": "snmpv2",
  "parameters": [
    {"name": "noise", "oid": "1.3.6.1.4.1.29956.3.2.10.40.0"},
    {"name": "vswr", "oid": "1.3.6.1.4.1.29956.3.2.10.41.0"},
    {"name": "tx", "oid": "1.3.6.1.4.1.29956.3.2.10.42.0"},
    {"name": "rx", "oid": "1.3.6.1.4.1.29956.3.2.10.44.0"},
    {"name": "voltage", "oid": "1.3.6.1.4.1.29956.3.2.10.45.0"}
  ]
}
```

El poller consulta cada OID y escribe una fila con fecha, parámetros y latencia. Para agregar un modelo, cree el JSON, confirme los OID con `snmpwalk` y asigne su nombre en el troncal.

## 9. Cómo funciona una plantilla Gnuplot

La plantilla recibe el archivo de datos y un título/salida definidos por TeleStats. Sus funciones son:

1. Elegir terminal y tamaño PNG.
2. Definir separador CSV y formato de fecha.
3. Configurar rangos y ejes.
4. Seleccionar columnas con `using`.
5. Aplicar conversiones de unidades.
6. Dibujar series y cerrar la salida.

En `zumlink.plot`, por ejemplo, el voltaje se convierte con `($6*0.001)`. Si el OID entrega milivoltios y se omite esa conversión, la gráfica queda con valores incorrectos. Un error de columnas, nombre o escala produce advertencias como `Skipping data file with no valid points` o `x range is invalid`.

## 10. Datos, gráficas y servidor HTTP

El poller actualiza el CSV histórico en `data/` y una copia de trabajo en `web/temp/`. El plotter ejecuta las plantillas y publica el PNG final en `web/graphs/`. El servidor HTTP expone:

- `/`: interfaz Jade renderizada.
- `/graphs/<nombre>`: imagen PNG.
- `/data?device=<nombre>&count=<N>`: últimas muestras del dispositivo.

Prueba básica:

```bash
curl -fsS http://127.0.0.1:8081/
curl -fsS --get http://127.0.0.1:8081/data \
  --data-urlencode 'device=Herveo - CERRO BRAVO' \
  --data-urlencode 'count=3'
```

## 11. Estados de estaciones de la interfaz

`ui/index.jade` construye `stationCatalog`, crea una tarjeta por estación y ejecuta `checkStations()` cada 60 segundos. Cada tarjeta consulta `/data` con sus enlaces. Se revisan respuesta HTTP, filas numéricas y antigüedad de la última marca de tiempo. Verde significa telemetría actualizada en los últimos 15 minutos; rojo indica datos ausentes, inválidos o atrasados.

Una tercera lista en una entrada del catálogo permite excluir enlaces que no deben decidir el estado, aunque continúen disponibles al abrir la estación. El resumen superior cuenta tarjetas verdes y rojas después de cada ronda.

Para agregar una estación:

```javascript
,['Nueva estación', ['Enlace A.png', 'Enlace B.png']]
```

Con excepción de evaluación:

```javascript
,['Nueva estación', ['Todos los enlaces.png'], ['Enlace usado para estado.png']]
```

Los nombres deben coincidir exactamente con los PNG y dispositivos publicados. Después ejecute `sudo ./install-ui.sh` y recargue con `Ctrl+F5`.

## 12. Scripts y sus funciones

### `launcher.sh`

- `read_pid`: lee el archivo PID.
- `running_pid`: confirma que `/proc/PID/cmdline` corresponde a `telestats.jar`.
- `start`: elimina un PID obsoleto, inicia Java en segundo plano y verifica el proceso.
- `stop`: envía `TERM`, espera hasta 20 segundos y limpia el PID.
- `status`: informa PID activo o salida detenida.

### `health.sh`

Valida el PID, la respuesta HTTP y, después de 30 minutos de vida, que el PNG más reciente no tenga más de 1800 segundos. Devuelve 0 si todo está correcto y 1/2 cuando falla proceso, puerto o gráficas.

### `install.sh`

Respalda archivos activos, instala configuración, valida `monit -t`, limpia temporales, detiene y arranca la aplicación, recarga Monit y muestra el estado.

### `install-ui.sh`

Respalda `web/index.jade`, copia la plantilla editada, reinicia TeleStats y reactiva el proceso en Monit.

### `cleanup.sh`

Elimina temporales de trabajo según la política definida y registra el número de archivos eliminados. No debe usarse para borrar históricos sin respaldo.

## 13. Monit y arranque automático

`monit-telestats` contiene:

```text
check process telestats matching "telestats.jar"
    start = "/usr/local/Apps/telestats/launcher.sh start"
    stop = "/usr/local/Apps/telestats/launcher.sh stop"
    if failed host 127.0.0.1 port 8081 protocol http request "/" with timeout 10 seconds then restart
    if 5 restarts within 10 cycles then alert

check program telestats-health with path "/usr/local/Apps/telestats/health.sh"
    every 5 cycles
    if status != 0 then alert
```

Validación y operación:

```bash
sudo monit -t
sudo monit reload
sudo monit monitor telestats
sudo monit status telestats
sudo monit status telestats-health
```

## 14. Despliegue completo con el mantenimiento

Desde una copia del proyecto:

```bash
cd /home/julian/telestats-maintenance
bash -n install.sh
sudo ./install.sh
```

El instalador crea `/var/backups/telestats-AAAAMMDD-HHMMSS`. Nunca copie manualmente una configuración sobre producción sin respaldo. Para cambios de radios use el instalador específico; para interfaz use `install-ui.sh`.

## 15. Diagnóstico ordenado

1. **Proceso:** `sudo /usr/local/Apps/telestats/launcher.sh status`.
2. **Monit:** `sudo monit status telestats`.
3. **HTTP:** `curl -fsS http://127.0.0.1:8081/`.
4. **Datos:** revisar `web/temp`, `data` y `/data?device=...`.
5. **SNMP:** `snmpwalk -v2c -c ovsm -t 3 -r 0 IP 1.3.6.1.2.1.1`.
6. **Gráficas:** revisar fecha del PNG y `logs/launcher.log`.
7. **Plantilla:** comprobar columnas, separador, unidades y rango x.
8. **Red/firewall:** confirmar ruta, ACL y UDP/161.

Mensajes frecuentes:

- `gráficas atrasadas`: el proceso puede estar vivo, pero el plotter no genera PNG recientes.
- `Skipping data file with no valid points`: CSV vacío, OID sin respuesta o nombre equivocado.
- `x range is invalid`: no hay fechas válidas para dibujar.
- Puerto HTTP sin respuesta: proceso detenido, error de arranque o escucha en otra interfaz.

## 16. Reversión

Cada instalador imprime la ruta de respaldo. Restaure los archivos necesarios, valide Monit y reinicie:

```bash
sudo monit -t
sudo monit reload
sudo /usr/local/Apps/telestats/launcher.sh restart
```

Si `restart` no está implementado en el launcher, use `stop` y luego `start`. Mantenga los CSV y PNG hasta terminar la investigación.
