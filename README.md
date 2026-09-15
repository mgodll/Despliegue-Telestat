# TeleStats: arquitectura, instalación y operación

Esta guía documenta la instalación que utiliza el equipo `tororoi`. TeleStats es el servicio que consulta equipos de telemetría por SNMP, guarda los datos, genera gráficas con Gnuplot y las publica mediante una interfaz web.

## 1. Arquitectura

```text
Equipos remotos (radios, routers, sensores)
              │ SNMP
              ▼
        telestats.jar
   ┌──────────┴──────────┐
   │                     │
   │ Poller              │ Plotter/Gnuplot
   │                     ▼
   │              web/temp y web/graphs/*.png
   ▼                     │
 network/trunks/*.json   ▼
              Servidor HTTP :8081
                       │
                       ▼
              Navegador del operador
```

El JAR ejecuta el sondeo y el procesamiento. Los archivos de modelos (`network/models`) indican qué OID SNMP leer para cada tipo de equipo; las plantillas (`network/templates`) convierten los datos en PNG. La interfaz se sirve desde `web/index.jade` y muestra las imágenes de `web/graphs`.

Monit supervisa dos elementos:

* `telestats`: proceso Java, respuesta HTTP en `127.0.0.1:8081` y reinicio automático.
* `telestats-health`: comprobación periódica del proceso, del puerto y de la antigüedad de las gráficas.

## 2. Rutas importantes

| Ruta | Uso |
|---|---|
| `/usr/local/Apps/telestats/telestats.jar` | Aplicación ejecutable |
| `/usr/local/Apps/telestats/conf/config.json` | Configuración global activa |
| `/usr/local/Apps/telestats/network/trunks/` | Redes y enlaces monitorizados |
| `/usr/local/Apps/telestats/network/models/` | Modelos SNMP por fabricante/equipo |
| `/usr/local/Apps/telestats/network/templates/` | Plantillas Gnuplot |
| `/usr/local/Apps/telestats/web/temp/` | Datos temporales por dispositivo |
| `/usr/local/Apps/telestats/web/graphs/` | Gráficas publicadas |
| `/usr/local/Apps/telestats/logs/telestats.log` | Registro de la aplicación |
| `/usr/local/Apps/telestats/logs/launcher.log` | Salida de generación de gráficas y arranque |
| `/etc/monit/conf-enabled/telestats` | Reglas de Monit |
| `/home/julian/telestats-maintenance/` | Copias editables y scripts de mantenimiento |

## 3. Configuración global

La copia de mantenimiento de `config.json` contiene:

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

`pollingPeriod` está expresado en segundos. `plottingPeriod` es el intervalo de actualización del generador de gráficas. `community`, las direcciones IP y los OID deben coincidir con la configuración de cada equipo remoto.

## 4. Instalación desde cero

Se requiere Linux, Java compatible con el JAR, Monit, `curl`, `gnuplot` y acceso SNMP a las redes de telemetría.

1. Instale las dependencias del sistema y Monit.
2. Cree la estructura `/usr/local/Apps/telestats` con permisos para el usuario que ejecutará la aplicación.
3. Copie `telestats.jar` en esa ruta y cree `conf`, `network`, `web` y `logs`.
4. Copie los JSON de redes, modelos SNMP y las plantillas `.plot` a sus directorios respectivos.
5. Ajuste `config.json`, las IP y la comunidad SNMP.
6. Copie `launcher.sh`, `health.sh`, `cleanup.sh` y `monit-telestats` desde este directorio.
7. Valide Monit y arranque:

```bash
cd /home/julian/telestats-maintenance
sudo ./install.sh
```

El instalador crea un respaldo en `/var/backups/telestats-AAAAmmdd-HHMMSS`, valida la sintaxis de Monit, detiene y arranca TeleStats y vuelve a habilitar su supervisión.

## 5. Operación diaria

```bash
sudo /usr/local/Apps/telestats/launcher.sh status
sudo /usr/local/Apps/telestats/launcher.sh stop
sudo /usr/local/Apps/telestats/launcher.sh start
sudo monit status telestats
sudo monit status telestats-health
```

La interfaz se consulta en:

```text
http://172.16.10.5:8081/
```

El grupo **Estado de estaciones** es la pantalla inicial. Las gráficas aparecen al seleccionar una red de monitoreo. El estado verde depende de telemetría reciente; el resumen de estaciones se actualiza cada minuto.

## 6. Agregar un equipo o enlace

1. Edite el JSON de la red correspondiente en `network/`.
2. Use el nombre exacto del enlace que tendrá el archivo de datos y la gráfica.
3. Seleccione el modelo SNMP apropiado en `network/models`.
4. Añada o ajuste la plantilla Gnuplot en `network/templates`.
5. Pruebe conectividad y SNMP desde el servidor:

```bash
ping -c 3 IP_DEL_EQUIPO
snmpwalk -v2c -c ovsm -t 3 -r 0 IP_DEL_EQUIPO 1.3.6.1.2.1.1
```

6. Instale el cambio con el script específico si existe (`install-radio-links.sh`, `install-rut200.sh`, `install-af5xhd-lacruz.sh`, etc.). Para cambios generales use `install.sh`.

Los nombres distinguen mayúsculas, minúsculas, espacios y tildes. Una diferencia entre el nombre del JSON, el archivo temporal y la plantilla suele producir una gráfica vacía.

## 7. Personalizar la interfaz

La plantilla editable es `ui/index.jade`. `install-ui.sh` la copia a la aplicación, reinicia TeleStats y vuelve a activar Monit:

```bash
cd /home/julian/telestats-maintenance
sudo ./install-ui.sh
```

El script guarda la versión anterior en `/var/backups/telestats-ui-*`. Después de instalar, recargue el navegador con `Ctrl+F5`.

## 8. Monit y health check

Verifique la configuración antes de recargar:

```bash
sudo monit -t
sudo monit reload
sudo monit monitor telestats
sudo monit monitor telestats-health
```

`health.sh` devuelve error si el proceso o el puerto no responden. Cuando el proceso lleva al menos 30 minutos, también comprueba que exista una gráfica PNG actualizada en los últimos 1800 segundos. Las advertencias de Gnuplot sobre archivos sin puntos pueden indicar que un equipo no responde o que el modelo/OID no coincide.

## 9. Diagnóstico rápido

* **Página no disponible:** comprobar `launcher.sh status`, `sudo monit status telestats` y `tail -f logs/launcher.log`.
* **Health en rojo por gráficas atrasadas:** comprobar SNMP, conectividad, hora del sistema y el último PNG con `find web/graphs -type f -name '*.png' -printf '%TY-%Tm-%Td %TH:%TM:%TS %p\n' | sort -r | head`.
* **Gráfica vacía:** revisar el archivo de `web/temp`, el nombre exacto del dispositivo y los OID del modelo.
* **Equipo sin datos:** ejecutar `ping` y `snmpwalk`; revisar comunidad, versión SNMP, firewall y ACL del equipo.
* **Cambio rechazado:** no sustituir manualmente el JAR activo sin respaldo. Usar los instaladores, que validan Monit y crean una copia de reversión.

## 10. Reversión y respaldos

Cada instalador crea un respaldo bajo `/var/backups`. Para volver atrás, restaure los archivos del respaldo correspondiente, valide `sudo monit -t`, recargue Monit y reinicie TeleStats. No elimine los directorios `web/temp` o `web/graphs` durante una investigación sin conservar primero una copia.

