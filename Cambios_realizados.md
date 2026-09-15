# Cambios de TeleStats y estados de estaciones

Este documento complementa `Despliegue_Teslestat.md` y describe las personalizaciones realizadas en la interfaz y el funcionamiento del panel **Estado de estaciones**.

## 1. Cambios realizados

### Interfaz y navegación

- Se creó una pantalla inicial sin gráficas.
- El grupo predeterminado pasó de **No seleccionar** a **Estado de estaciones**.
- La pantalla inicial muestra el estado general y no carga automáticamente una gráfica.
- Al seleccionar una red se oculta el panel de estaciones y se muestran las gráficas de esa red.
- Se añadió el botón **Inicio**.
- Cada estación se muestra como una tarjeta con indicador verde/rojo, mensaje y hora de comprobación.
- Al hacer clic en una tarjeta se abre una ventana con sus gráficas asociadas.
- Se añadieron filtro, contador de gráficas, ventana ampliada y nombres visuales personalizados.

### Resumen automático

Debajo del título aparece una nota con:

```text
Total: N · Funcionando: V · Con problemas: R · Actualizado: HH:MM:SS
```

El resumen cuenta las tarjetas del panel y se actualiza después de cada comprobación. La consulta completa se repite cada 60 segundos.

### Estaciones incorporadas

El catálogo incluye, entre otras, Moral GNSS, Aguas Calientes, San Julian, Alfombrales Doas, El Aguila, Glaciar, Anillo, San Juan, Lajas, La Secreta, Cima, Toche, La Palma, Silencio, Rodeo, Tapias, San Lorenzo, Santa Ana, Magnetómetro Domo, Estatuas, Río Claro, Balcones, Recio, Recio 3, Nido de Águila, Esmeralda, Laguna Verde, Totarito, Olleta 2, Alfombrales BA, Inclinómetro de Santa Isabel, Magnetómetro de Alfombrales, Molinos 1, La Siberia, Cerro Bravo CP, Aguacatal, Gualí 3, Lisa, Inderena, Billar, CAM Lagunillas y Lagunillas Flujo de Lodos.

También se agregaron las estaciones PTZ: Guali PTZ, Olleta PTZ, Desquite PTZ, Coralito PTZ y C. Bravo PTZ. Los equipos ePever no se usan como estaciones de estado; sus gráficas siguen disponibles dentro de las redes.

## 2. Criterio de estado

El código está en `ui/index.jade`, dentro de `checkStations()`. Para cada estación se usa su lista de gráficas o, si existe, una lista especial de gráficas de evaluación. Por cada enlace se solicita:

```text
/data?device=NOMBRE_DEL_ENLACE&count=3
```

Una gráfica es válida si el servidor responde, devuelve filas, los valores son numéricos y la última marca de tiempo es válida. La muestra más reciente debe tener menos de 15 minutos.

- **Verde — Telemetría actualizada:** todos los enlaces seleccionados tienen datos recientes.
- **Rojo — Telemetría no disponible:** algún enlace no responde o no tiene valores válidos.
- **Rojo — Telemetría atrasada:** los datos superan 15 minutos.
- **Rojo — Sin telemetría configurada:** no hay gráficas asociadas.

Cada tarjeta se actualiza de forma independiente y, al terminar las peticiones, se recalcula el contador general.

## 3. Excepciones de evaluación

Una tercera lista en `stationCatalog` permite mostrar todas las gráficas de una estación, pero usar solo algunas para decidir el color:

- Pitayo: solo enlaces con Cerro Bravo.
- Paramillo: solo enlaces Manizales–Paramillo.
- Cerro Bravo: solo enlaces con Desquite.
- África y Azufrera: enlaces con Paramillo.
- Recio: `Recio - CISNE`.
- Olleta 2: enlaces con Manizales.
- Inclinómetro de Santa Isabel: enlace del inclinómetro con Olleta.
- CAM Lagunillas: se excluye Pitayo–Lagunillas.
- Herveo: se usa `Herveo - CERRO BRAVO`.
- La Palma: no se consideran `Toche - LA PALMA` ni `Cima - LA PALMA`.
- Anillo: se usa `Anillo - LA SECRETA`; se excluye `LA SECRETA - Anillo`.

Estas excepciones deben revisarse si cambia la topología o el nombre de una gráfica.

## 4. Nombres visuales y PTZ

Los archivos originales no se renombran: `displayName()` cambia únicamente el texto mostrado. Entre los cambios están:

| Original | Mostrado |
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

`Pitayo - Coralito C5X` y `Pitayo - C. Bravo C5X` se excluyen de las vistas. La comparación ignora mayúsculas y la extensión `.png`.

## 5. Cómo implementar otra estación

1. Confirme el nombre exacto del enlace en `/usr/local/Apps/telestats/web/graphs`.
2. Compruebe que exista el mismo dispositivo en `web/temp` y que `/data?device=...` devuelva datos.
3. Edite `ui/index.jade` y agregue:

```javascript
,['Nombre de estación', ['Enlace A.png', 'Enlace B.png']]
```

Si solo ciertos enlaces deben decidir el color:

```javascript
,['Nombre de estación',
  ['Todos los enlaces.png'],
  ['Enlaces usados para el estado.png']]
```

4. Respete mayúsculas, minúsculas, espacios y tildes.
5. Valide y publique:

```bash
cd /home/julian/telestats-maintenance
bash -n install-ui.sh
sudo ./install-ui.sh
```

6. Recargue el navegador con `Ctrl+F5` y espere hasta un minuto para la primera evaluación.

## 6. Verificación

```bash
sudo monit status telestats
sudo monit status telestats-health
sudo /usr/local/Apps/telestats/launcher.sh status
```

Para probar un enlace:

```bash
curl -sS --get 'http://127.0.0.1:8081/data' \
  --data-urlencode 'device=Herveo - CERRO BRAVO' \
  --data-urlencode 'count=3'
```

Revise los registros si no hay datos:

```bash
tail -f /usr/local/Apps/telestats/logs/telestats.log
tail -f /usr/local/Apps/telestats/logs/launcher.log
```

## 7. Límites conocidos

El color representa la disponibilidad de la telemetría seleccionada, no garantiza que todos los sensores auxiliares de una estación estén operativos. Las listas de evaluación deben mantenerse alineadas con el criterio operativo de cada estación.
