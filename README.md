# Portón XHouse EGA1800 → local con ESPHome

*English: [README.en.md](README.en.md)*

Convierte un portón batiente **XHouse EGA1800** (placa `EGA-18`, "WiFi+BLE Swing
Gate") de un dispositivo **dependiente de la nube** a uno **100% local**,
controlado desde Home Assistant con [ESPHome](https://esphome.io).

Sin cambiar hardware: el módulo "Smart" que trae el portón **ya es un ESP32-C2**,
así que se **reflashea en el lugar** con ESPHome. Deja de hablar con la nube del
fabricante (Alibaba Cloud IoT) y pasa a hablar directamente, por UART, con el
STM32 que maneja los motores.

## Por qué

- **Rapidez.** El comando viaja ESP ↔ STM32 por cable; abre al instante. Nada de
  ida y vuelta a un servidor en China.
- **Independencia de la nube.** Funciona aunque te quedes sin internet, aunque el
  fabricante apague el servicio o cambie la app. Tus datos no salen de tu red.
- **Integración nativa.** Aparece en Home Assistant como un `cover` (portón),
  botones de peatonal/vehículo, voltaje de la batería de respaldo y estado de
  cada hoja. Automatizable como cualquier entidad local.

## ⚠️ Antes de empezar: hacé un backup del firmware stock

Flashear ESPHome **borra** el firmware de fábrica del módulo. Si algún día querés
volver atrás (o algo sale mal), **necesitás ese backup**. No hay descarga pública
del firmware del ESP, así que **esta es tu única copia**. Hacela primero.

Con un adaptador USB-UART de **3.3 V** conectado al ESP32-C2 (TXD0/RXD0, EN/reset,
3V3, GND) y **GPIO9 a GND** para entrar en modo descarga:

```bash
esptool.py --port /dev/ttyUSB0 read_flash 0 0x200000 esp_stock_backup.bin
```

Guardá `esp_stock_backup.bin` (2 MB) en un lugar seguro, fuera del equipo. Para
restaurar el firmware original más adelante:

```bash
esptool.py --port /dev/ttyUSB0 write_flash 0 esp_stock_backup.bin
```

> El detalle físico del tap de pines, el modo descarga y las fotos de la placa van
> en la documentación de hardware (próximamente).

## Cómo funciona (software)

```
Home Assistant  ──API ESPHome──►  ESP32-C2 (este firmware)  ──UART 115200──►  STM32  ──►  motores
```

- El **STM32** es el cerebro del portón: maneja motores, finales de carrera,
  batería de respaldo (24 V), display y los controles RF. **No se toca.**
- El **ESP32-C2** (módulo "Smart") normalmente agrega WiFi+BLE y habla con la
  nube. Lo reemplazamos por ESPHome, que habla el **mismo protocolo UART** que
  esperaba el STM32.
- Comando de control: `55 AA 01 <canal> 02` (emula una pulsación del control,
  paso a paso). Estado y batería: el ESP sondea con `55 AA 32` cada 3 s y parsea
  la respuesta.

El protocolo completo (tramas, bytes, códigos de estado) está en
[`docs/protocolo-uart.md`](docs/protocolo-uart.md). Se obtuvo por ingeniería
inversa y se verificó sobre un EGA1800 real.

## Lo que ves en Home Assistant

| Entidad                  | Tipo            | Qué hace                                        |
|--------------------------|-----------------|-------------------------------------------------|
| Portón                   | `cover` (gate)  | abrir / cerrar / parar, con % de apertura       |
| Peatonal                 | `button`        | pulsa el canal peatonal (una hoja)              |
| Vehículo                 | `button`        | pulsa el canal del portón completo              |
| Batería                  | `sensor` (V)    | voltaje de la batería de respaldo               |
| Batería presente         | `binary_sensor` | si hay batería de respaldo conectada            |
| Modelo / Firmware        | `text_sensor`   | lo que reporta el STM32 (p. ej. EGA1800 / 1.3.0)|

## Instalación

1. **Instalá ESPHome** (add-on de Home Assistant, o el
   [CLI](https://esphome.io/guides/installing_esphome)).
2. **Backup del firmware stock** (ver arriba). No te lo saltees.
3. Copiá los archivos de ejemplo:
   ```bash
   cp esp-ega1800.example.yaml esp-ega1800.yaml
   cp secrets.yaml.example secrets.yaml
   ```
4. Editá `secrets.yaml` con tu WiFi y una clave de API (hay instrucciones en el
   propio archivo).
5. Compilá y flasheá. La **primera vez** tiene que ser por **cable serie**
   (adaptador USB-UART 3.3 V al ESP, GPIO9 a GND):
   ```bash
   esphome run esp-ega1800.yaml
   ```
   Las siguientes actualizaciones ya van **por WiFi (OTA)**, sin abrir la caja.
6. Agregá el dispositivo en Home Assistant: **Ajustes → Dispositivos y servicios**
   → debería aparecer por descubrimiento automático (ESPHome).

## Notas importantes

- El módulo es un **ESP32-C2** con **cristal de 26 MHz**: el YAML ya trae los
  `sdkconfig_options` necesarios. Sin eso, el WiFi no engancha.
- Dejá `power_save_mode: none` en WiFi: el C2 es de un solo core y el power-save
  interrumpe la UART.
- El control es **paso a paso** (cada pulso cicla abrir→parar→cerrar). El `cover`
  del YAML ya tiene la lógica para que abrir/cerrar/parar de HA hagan lo esperado.
- El STM32 **no reporta estado solo**: hay que sondearlo. El YAML sondea cada 3 s.

## Otros modelos

El comando de control suele servir igual, pero el **layout de la trama de estado
puede cambiar** entre variantes (EGA18, EGB18, EGA15, EGB1900…). Si convertís
otro modelo, ver la sección final de [`docs/protocolo-uart.md`](docs/protocolo-uart.md)
para capturar y ajustar los offsets.

## Contribuir

¿Lo hiciste andar en tu portón, convertiste otra variante o encontraste algo para
mejorar? Bienvenido: **abrí un issue** para reportar un problema, hacer una
pregunta o compartir capturas de tramas de otro modelo, o **mandá un pull
request** con tus cambios. Todo aporte suma para que más gente pueda pasar su
portón a local.

## Descargo de responsabilidad

Esto implica reflashear hardware que controla un portón motorizado. Lo hacés bajo
tu propia responsabilidad: podés dejar el módulo inutilizable (de ahí el backup) y
un portón mal configurado puede ser peligroso. Probá siempre con el portón
despejado y a mano el corte de energía. No está afiliado a XHouse/Giigle.
