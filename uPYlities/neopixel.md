# Animación NeoPixel bandera trans en Raspberry Pi 5 con Ubuntu

Solicitado a ChatGPT

Esta documentación explica cómo instalar, configurar y ejecutar automáticamente un programa en Python para controlar una tira de **16 NeoPixels / WS2812** conectada a una **Raspberry Pi 5 con Ubuntu**, mostrando una secuencia suave con los colores de la **bandera trans** al **20% de brillo**.

El programa usa la librería `pi5neo`, que controla NeoPixels en Raspberry Pi 5 usando el periférico **SPI**. La tira NeoPixel no habla SPI de forma nativa, pero SPI permite generar una señal estable y suficientemente precisa para LEDs WS2812.

---

## 1. Descripción general

El programa genera una animación continua y suave en una tira NeoPixel de 16 LEDs.

La secuencia usa los colores de la bandera trans:

| Color | RGB |
|---|---|
| Azul claro | `91, 206, 250` |
| Rosa | `245, 169, 184` |
| Blanco | `255, 255, 255` |
| Rosa | `245, 169, 184` |
| Azul claro | `91, 206, 250` |

El brillo está limitado al **20%** para reducir consumo, calor y deslumbramiento.

---

## 2. Hardware utilizado

| Componente | Descripción |
|---|---|
| Raspberry Pi 5 | Corriendo Ubuntu |
| Tira NeoPixel / WS2812 | 16 LEDs direccionables |
| Cables Dupont | Para alimentación, tierra y datos |
| Fuente externa 5V | Recomendada si la tira consume más corriente |

---

## 3. Conexión recomendada

Aunque originalmente se intentó usar **GPIO17**, con Raspberry Pi 5 y la librería `pi5neo` se recomienda usar **SPI**, específicamente el pin **GPIO10 / MOSI**.

## Tabla de conexión

| NeoPixel | Raspberry Pi 5 |
|---|---|
| VCC / 5V | 3.3V para pruebas pequeñas o 5V externo recomendado |
| GND | GND de Raspberry Pi |
| DIN | GPIO10 / pin físico 19 |

## Nota sobre alimentación

Para una tira de **16 LEDs** al **20% de brillo**, puede funcionar usando 3.3V desde la Raspberry Pi si la tira lo tolera. Sin embargo, lo más correcto es alimentar la tira con una **fuente externa de 5V** y unir las tierras:

```text
GND fuente externa ─── GND Raspberry Pi ─── GND NeoPixel
5V fuente externa  ─── VCC NeoPixel
GPIO10 Pi          ─── DIN NeoPixel
```

No se recomienda alimentar tiras grandes directamente desde la Raspberry Pi.

---

## 4. Activar SPI en Ubuntu / Raspberry Pi 5

Primero revisa si SPI ya está disponible:

```bash
ls /dev/spidev*
```

Si aparece algo como esto:

```bash
/dev/spidev0.0  /dev/spidev0.1
```

SPI ya está activo.

Si no aparece, activa SPI con:

```bash
sudo raspi-config
```

Ruta dentro de `raspi-config`:

```text
Interface Options → SPI → Enable
```

Después reinicia:

```bash
sudo reboot
```

---

## 5. Instalación del entorno Python

Actualizar paquetes:

```bash
sudo apt update
```

Instalar herramientas necesarias:

```bash
sudo apt install -y python3-pip python3.14-venv python3-venv python3-wheel
```

Crear entorno virtual:

```bash
python3 -m venv ~/neopixel-env
```

Activar entorno:

```bash
source ~/neopixel-env/bin/activate
```

Actualizar `pip`:

```bash
pip install --upgrade pip
```

Instalar `pi5neo`:

```bash
pip install pi5neo
```

---

## 6. Estructura recomendada de archivos

Crear carpeta del proyecto:

```bash
mkdir -p ~/neopixel
```

El programa principal quedará en:

```bash
/home/luma/neopixel/trans_neopixels.py
```

---

## 7. Programa completo

Crear archivo:

```bash
nano ~/neopixel/trans_neopixels.py
```

Pegar el siguiente código:

```python
from pi5neo import Pi5Neo
import time
import signal
import sys

NUM_LEDS = 16
BRIGHTNESS = 0.20

# Pi5Neo usa SPI en Raspberry Pi 5.
# Normalmente corresponde a GPIO10 / pin físico 19.
strip = Pi5Neo("/dev/spidev0.0", NUM_LEDS, 800)

TRANS_COLORS = [
    (91, 206, 250),   # Azul claro
    (245, 169, 184),  # Rosa
    (255, 255, 255),  # Blanco
    (245, 169, 184),  # Rosa
    (91, 206, 250),   # Azul claro
]


def apply_brightness(color, brightness):
    r, g, b = color
    return (
        int(r * brightness),
        int(g * brightness),
        int(b * brightness),
    )


def blend(c1, c2, t):
    return (
        int(c1[0] + (c2[0] - c1[0]) * t),
        int(c1[1] + (c2[1] - c1[1]) * t),
        int(c1[2] + (c2[2] - c1[2]) * t),
    )


def smoothstep(t):
    return t * t * (3 - 2 * t)


def clear_strip():
    for i in range(NUM_LEDS):
        strip.set_led_color(i, 0, 0, 0)
    strip.update_strip()


def exit_gracefully(sig=None, frame=None):
    clear_strip()
    sys.exit(0)


signal.signal(signal.SIGINT, exit_gracefully)
signal.signal(signal.SIGTERM, exit_gracefully)

offset = 0.0

while True:
    for i in range(NUM_LEDS):
        position = ((i / NUM_LEDS) * len(TRANS_COLORS) + offset) % len(TRANS_COLORS)

        index_a = int(position)
        index_b = (index_a + 1) % len(TRANS_COLORS)

        t = position - index_a
        t = smoothstep(t)

        color = blend(TRANS_COLORS[index_a], TRANS_COLORS[index_b], t)
        color = apply_brightness(color, BRIGHTNESS)

        r, g, b = color
        strip.set_led_color(i, r, g, b)

    strip.update_strip()

    offset += 0.012
    time.sleep(0.035)
```

Guardar en `nano`:

```text
Ctrl + O
Enter
Ctrl + X
```

---

## 8. Probar el programa manualmente

Activar entorno virtual:

```bash
source ~/neopixel-env/bin/activate
```

Ejecutar:

```bash
python ~/neopixel/trans_neopixels.py
```

Para detener:

```text
Ctrl + C
```

Al detenerse, el programa apaga la tira.

---

## 9. Crear servicio para arranque automático

Para que la animación arranque con el sistema, se crea un servicio de `systemd`.

Crear archivo del servicio:

```bash
sudo nano /etc/systemd/system/neopixel-trans.service
```

Pegar:

```ini
[Unit]
Description=NeoPixel trans flag animation
After=multi-user.target

[Service]
Type=simple
User=luma
WorkingDirectory=/home/luma/neopixel
ExecStart=/home/luma/neopixel-env/bin/python /home/luma/neopixel/trans_neopixels.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Guardar:

```text
Ctrl + O
Enter
Ctrl + X
```

---

## 10. Activar el servicio

Recargar `systemd`:

```bash
sudo systemctl daemon-reload
```

Activar arranque automático:

```bash
sudo systemctl enable neopixel-trans.service
```

Iniciar servicio:

```bash
sudo systemctl start neopixel-trans.service
```

Revisar estado:

```bash
systemctl status neopixel-trans.service
```

Debe aparecer:

```text
active (running)
```

---

## 11. Ver logs del servicio

Para ver errores o mensajes del programa:

```bash
journalctl -u neopixel-trans.service -f
```

Salir:

```text
Ctrl + C
```

---

## 12. Comandos útiles

Detener animación:

```bash
sudo systemctl stop neopixel-trans.service
```

Iniciar animación:

```bash
sudo systemctl start neopixel-trans.service
```

Reiniciar animación:

```bash
sudo systemctl restart neopixel-trans.service
```

Desactivar arranque automático:

```bash
sudo systemctl disable neopixel-trans.service
```

Volver a activar arranque automático:

```bash
sudo systemctl enable neopixel-trans.service
```

---

## 13. Permisos de SPI

Si el programa funciona manualmente pero falla como servicio, puede ser un problema de permisos sobre `/dev/spidev0.0`.

Agregar el usuario `luma` al grupo `spi`:

```bash
sudo usermod -aG spi luma
```

Después reiniciar:

```bash
sudo reboot
```

Si Ubuntu no tiene grupo `spi` o sigue fallando, se puede editar el servicio para correr como `root`.

Editar:

```bash
sudo nano /etc/systemd/system/neopixel-trans.service
```

Cambiar:

```ini
User=luma
```

por:

```ini
User=root
```

Luego:

```bash
sudo systemctl daemon-reload
sudo systemctl restart neopixel-trans.service
```

---

## 14. Diagnóstico rápido

Verificar SPI:

```bash
ls /dev/spidev*
```

Resultado esperado:

```bash
/dev/spidev0.0  /dev/spidev0.1
```

Verificar servicio:

```bash
systemctl status neopixel-trans.service
```

Ver logs recientes:

```bash
journalctl -u neopixel-trans.service -n 50
```

Ejecutar manualmente con el Python del entorno:

```bash
/home/luma/neopixel-env/bin/python /home/luma/neopixel/trans_neopixels.py
```

Ejecutar manualmente con permisos elevados:

```bash
sudo /home/luma/neopixel-env/bin/python /home/luma/neopixel/trans_neopixels.py
```

---

## 15. Problemas comunes

| Problema | Posible causa | Solución |
|---|---|---|
| No prende nada | DIN conectado a GPIO17 | Mover DIN a GPIO10 / pin físico 19 |
| No existe `/dev/spidev0.0` | SPI desactivado | Activar SPI con `raspi-config` |
| Funciona con `sudo` pero no normal | Permisos de SPI | Agregar usuario a grupo `spi` o usar `User=root` |
| Parpadea o colores raros | Alimentación inestable | Usar fuente externa 5V y GND común |
| Colores incorrectos | Orden RGB distinto | Ajustar orden de color en código |
| El servicio no arranca | Ruta incorrecta | Revisar `ExecStart` y `WorkingDirectory` |
| Se apaga al cerrar terminal | Se ejecutó manual, no como servicio | Usar `systemctl enable/start` |

---

## 16. Resumen de instalación completa

Este bloque resume el proceso completo desde cero:

```bash
sudo apt update
sudo apt install -y python3-pip python3.14-venv python3-venv python3-wheel

python3 -m venv ~/neopixel-env
source ~/neopixel-env/bin/activate
pip install --upgrade pip
pip install pi5neo

mkdir -p ~/neopixel
nano ~/neopixel/trans_neopixels.py
```

Luego crear servicio:

```bash
sudo nano /etc/systemd/system/neopixel-trans.service
```

Contenido del servicio:

```ini
[Unit]
Description=NeoPixel trans flag animation
After=multi-user.target

[Service]
Type=simple
User=luma
WorkingDirectory=/home/luma/neopixel
ExecStart=/home/luma/neopixel-env/bin/python /home/luma/neopixel/trans_neopixels.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Activar:

```bash
sudo systemctl daemon-reload
sudo systemctl enable neopixel-trans.service
sudo systemctl start neopixel-trans.service
```

Revisar:

```bash
systemctl status neopixel-trans.service
```

---

## 17. Notas finales

Este programa está pensado para una tira pequeña de **16 NeoPixels**, usando una animación suave y continua con bajo brillo.

Para más LEDs, conviene usar:

- fuente externa de 5V,
- tierra común entre fuente y Raspberry Pi,
- conversor de nivel lógico de 3.3V a 5V para la señal de datos,
- cableado más robusto.

Configuración final recomendada:

```text
Raspberry Pi 5 + Ubuntu
NeoPixel DIN → GPIO10 / pin físico 19
SPI activado
Python venv en ~/neopixel-env
Programa en ~/neopixel/trans_neopixels.py
Servicio systemd neopixel-trans.service
Arranque automático habilitado
```

Resultado: al encender la Raspberry Pi, la tira NeoPixel arranca automáticamente con una animación suave en colores de la bandera trans al 20% de brillo.