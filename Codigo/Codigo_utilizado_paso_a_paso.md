# Guía Completa de Instalación y Puesta en Marcha
## Estación Base GSM con OpenBTS + USRP NI 2901 (B210) + Asterisk

**Proyecto:** Comunicaciones Móviles — UNMSM/FIEE
**Hardware:** USRP NI 2901 (B210), laptop Acer Aspire A315-59 (firmware Insyde H2O)
**Software base:** Ubuntu 20.04.6 LTS, UHD 3.15, OpenBTS, Asterisk 16, Linphone

> Esta guía documenta, en orden cronológico, todos los comandos utilizados desde la instalación
> limpia del sistema operativo hasta dejar operativa la estación base GSM y el sistema de
> respaldo VoIP. Incluye los parches y ajustes que fueron necesarios para resolver los problemas
> de compatibilidad e infraestructura encontrados durante el desarrollo del proyecto.

---

## Índice

0. [Antes de empezar: requisitos y advertencias](#0-antes-de-empezar-requisitos-y-advertencias)
1. [Instalación del sistema operativo (Ubuntu 20.04.6 LTS)](#1-instalación-del-sistema-operativo-ubuntu-20046-lts)
2. [Ajustes de firmware / arranque (Acer Insyde H2O)](#2-ajustes-de-firmware--arranque-acer-insyde-h2o)
3. [Preparación del sistema base](#3-preparación-del-sistema-base)
4. [Instalación de UHD 3.15 (USRP Hardware Driver)](#4-instalación-de-uhd-315-usrp-hardware-driver)
5. [Verificación del hardware USRP B210](#5-verificación-del-hardware-usrp-b210)
6. [Compilación de OpenBTS desde código fuente](#6-compilación-de-openbts-desde-código-fuente)
7. [Parches aplicados al código fuente de OpenBTS](#7-parches-aplicados-al-código-fuente-de-openbts)
8. [Configuración de OpenBTS (GSMConfig)](#8-configuración-de-openbts-gsmconfig)
9. [Arranque de OpenBTS y primeras pruebas](#9-arranque-de-openbts-y-primeras-pruebas)
10. [Instalación y configuración de Asterisk 16](#10-instalación-y-configuración-de-asterisk-16)
11. [Configuración del respaldo VoIP con Linphone](#11-configuración-del-respaldo-voip-con-linphone)
12. [Monitoreo, diagnóstico y comandos de operación diaria](#12-monitoreo-diagnóstico-y-comandos-de-operación-diaria)
13. [Solución de problemas encontrados (troubleshooting)](#13-solución-de-problemas-encontrados-troubleshooting)
14. [Procedimiento de apagado seguro](#14-procedimiento-de-apagado-seguro)
15. [Checklist rápido de arranque (resumen)](#15-checklist-rápido-de-arranque-resumen)

---

## 0. Antes de empezar: requisitos y advertencias

- **Hardware necesario:** USRP NI 2901 (B210), cable USB 3.0 (directo al puerto del equipo, sin hub
  no alimentado), antena GSM banda 900, laptop con al menos 8 GB de RAM y disco de al menos 40 GB
  libres.
- **Advertencia legal:** la transmisión en banda GSM (900/1800 MHz) está regulada. Esta guía asume
  un entorno de laboratorio controlado, con potencia reducida y fines exclusivamente académicos,
  tal como se ejecutó en el proyecto del curso. No debe usarse fuera de un entorno autorizado.
- **Recomendación de almacenamiento:** instalar el sistema operativo en un **disco interno**
  (SATA/NVMe), no en un disco externo por adaptador SATA-USB. El proyecto identificó que la
  latencia de un adaptador SATA-USB provoca errores `STALE burst` en el Transceiver (ver
  sección 13.1).

---

## 1. Instalación del sistema operativo (Ubuntu 20.04.6 LTS)

1. Descargar la imagen ISO de Ubuntu 20.04.6 LTS desde el repositorio oficial de Canonical.
2. Crear un USB de arranque (desde otro equipo) con `dd` o herramientas como Rufus/BalenaEtcher:

```bash
# Desde un equipo Linux auxiliar, reemplazando /dev/sdX por el USB destino
sudo dd if=ubuntu-20.04.6-desktop-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

3. Arrancar el equipo Acer Aspire A315-59 desde el USB (ver sección 2 para el ajuste de firmware
   necesario en este modelo).
4. Seleccionar **"Instalación normal"** y marcar la opción de instalar software de terceros
   (controladores gráficos, códecs) durante el asistente.
5. **Elegir el disco interno del equipo como destino de instalación** (no un disco externo por
   USB), usando la opción "Borrar disco e instalar Ubuntu" o particionado manual según
   preferencia.
6. Completar el asistente (usuario, contraseña, zona horaria: America/Lima) y reiniciar al
   finalizar.

---

## 2. Ajustes de firmware / arranque (Acer Insyde H2O)

Durante el proyecto se presentaron conflictos entre el gestor de arranque **GRUB** y el firmware
**Insyde H2O** del Acer Aspire A315-59, que **no se resuelven únicamente con comandos de sistema
operativo** (por ejemplo `bcdedit`, que además es un comando de Windows y no aplica aquí). Es
necesario intervenir directamente el firmware:

1. Reiniciar el equipo y presionar repetidamente **F2** (o la tecla indicada en pantalla) para
   entrar a la configuración de BIOS/UEFI.
2. Ir a la pestaña **Boot**.
3. Ubicar la opción **Boot Priority Order** y mover **ubuntu** (o el gestor GRUB detectado) al
   primer lugar de la lista, por encima de Windows Boot Manager si existiera.
4. Desactivar **Secure Boot** si OpenBTS/UHD presentan problemas de carga de módulos (algunos
   controladores no firmados pueden requerir esto).
5. Guardar cambios y salir (**F10**, confirmar).

> Si tras este ajuste el equipo sigue arrancando el gestor incorrecto, repetir el reordenamiento
> desde el propio menú de arranque rápido (tecla **F12** en el POST) seleccionando manualmente la
> entrada de Ubuntu/GRUB.

---

## 3. Preparación del sistema base

Con Ubuntu ya instalado y arrancando correctamente:

```bash
# Actualizar el sistema
sudo apt update
sudo apt -y upgrade
sudo apt -y dist-upgrade

# Herramientas de compilación y dependencias generales
sudo apt install -y build-essential git cmake pkg-config \
    autoconf automake libtool \
    libboost-all-dev libusb-1.0-0-dev \
    doxygen python3-dev python3-pip \
    libczmq-dev libzmq3-dev \
    sqlite3 libsqlite3-dev \
    ncurses-dev \
    net-tools htop iotop iostat sysstat

# Herramientas de diagnóstico de disco (usadas más adelante en el troubleshooting de STALE bursts)
sudo apt install -y smartmontools e2fsprogs
```

---

## 4. Instalación de UHD 3.15 (USRP Hardware Driver)

OpenBTS requiere el driver UHD para comunicarse con el USRP B210. Se utilizó la versión **3.15**,
que exige los parches descritos en la sección 7.

```bash
# Clonar el repositorio de UHD
cd ~
git clone https://github.com/EttusResearch/uhd.git
cd uhd
git checkout v3.15.0.0

# Compilar UHD
cd host
mkdir build && cd build
cmake ../
make -j$(nproc)
sudo make install
sudo ldconfig

# Descargar las imágenes de FPGA/firmware necesarias para el B210
sudo uhd_images_downloader
```

Verificar que la instalación fue exitosa:

```bash
uhd_config_info --version
```

---

## 5. Verificación del hardware USRP B210

Con el USRP conectado por USB 3.0 directamente al equipo:

```bash
# Reglas udev para permisos de USB (evita tener que usar sudo para cada acceso al USRP)
cd ~/uhd/host/utils
sudo cp uhd-usrp.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger

# Verificar que el sistema reconoce el dispositivo USB
lsusb | grep -i "Ettus\|National Instruments"

# Verificar que UHD detecta el USRP
uhd_find_devices

# Prueba de auto-diagnóstico del hardware (opcional pero recomendado la primera vez)
uhd_usrp_probe
```

Salida esperada de `uhd_find_devices` (resumen):

```text
--------------------------------------------------
-- UHD Device 0
--------------------------------------------------
Device Address:
    serial: 3XXXXXX
    name:
    product: B210
    type: b200
```

Si el dispositivo no aparece, revisar la sección 13.2 (troubleshooting de reconocimiento USB).

---

## 6. Compilación de OpenBTS desde código fuente

```bash
cd ~
git clone https://github.com/RangeNetworks/openbts.git OpenBTS-src
cd OpenBTS-src

# (Aplicar aquí los parches de la sección 7 antes de compilar)

# Generar el sistema de compilación
autoreconf -i

# Configurar la compilación CON soporte UHD (crítico: sin este flag se compila
# la variante TransceiverRAD1, incompatible con el USRP B210)
./configure --with-uhd

# Compilar
make -j$(nproc)

# Instalar en el sistema
sudo make install
```

Verificar que el binario correcto fue generado:

```bash
ls -la apps/ | grep Transceiver
# Debe aparecer Transceiver52M (variante compatible con UHD/USRP)
# Si en su lugar aparece TransceiverRAD1, revisar que --with-uhd
# se haya pasado correctamente en el configure y volver a compilar
```

---

## 7. Parches aplicados al código fuente de OpenBTS

Antes de compilar (paso anterior), fue necesario aplicar los siguientes cambios manuales al
código fuente para lograr compatibilidad con UHD 3.15, dado que el código original de OpenBTS fue
escrito contra versiones anteriores de la API de UHD.

### 7.1 Eliminación de llamadas a `uhd::msg` (obsoleto en UHD 3.15)

Archivo afectado: `Transceiver52M/UHDDevice.cpp`

```bash
cd ~/OpenBTS-src/Transceiver52M
grep -n "uhd::msg" UHDDevice.cpp
```

Cada ocurrencia de `uhd::msg::register_handler(...)` o llamadas similares fue eliminada o
comentada, reemplazando el mecanismo de logging por el sistema de logging estándar vigente en
UHD 3.15 (basado en `uhd::log`):

```cpp
// ANTES (código original, incompatible con UHD 3.15):
// uhd::msg::register_handler(&uhd_msg_handler);

// DESPUÉS (eliminado / reemplazado por el logging nativo de UHD 3.15):
// (sin registro manual de handler; UHD 3.15 gestiona el logging internamente)
```

### 7.2 Renombramiento de `thread_priority.hpp`

UHD 3.15 reorganizó la ubicación de este header. Fue necesario ajustar la ruta de inclusión en
`UHDDevice.cpp`:

```bash
# Localizar la ruta real del header en la instalación de UHD 3.15
find /usr/local/include/uhd -iname "thread_priority*"
```

```cpp
// ANTES:
#include <uhd/utils/thread_priority.hpp>

// DESPUÉS (ruta ajustada según la ubicación real en UHD 3.15):
#include <uhd/utils/thread.hpp>
```

> La ruta exacta puede variar según la subversión de UHD 3.15 instalada; se recomienda confirmar
> con el comando `find` anterior antes de editar.

### 7.3 Incremento del timeout de comunicación con el Transceiver

Archivo afectado: configuración de timeouts dentro de `Transceiver52M` (constante de timeout de
inicialización).

```bash
grep -rn "TIMEOUT" ~/OpenBTS-src/Transceiver52M/*.cpp ~/OpenBTS-src/Transceiver52M/*.h
```

Se incrementó el valor por defecto (originalmente ~5 segundos) a aproximadamente **20 segundos**,
para acomodar el tiempo real de inicialización del USRP B210 (carga de FPGA, calibración de
front-end), que resultó mayor al esperado por el código original:

```cpp
// ANTES:
#define TRX_TIMEOUT_MS 5000

// DESPUÉS:
#define TRX_TIMEOUT_MS 20000
```

Tras aplicar los tres parches anteriores, volver a la sección 6 y ejecutar `./configure --with-uhd`
seguido de `make -j$(nproc)` para recompilar con los cambios incorporados.

---

## 8. Configuración de OpenBTS (GSMConfig)

OpenBTS almacena su configuración en una base de datos SQLite gestionada mediante la interfaz de
línea de comandos `OpenBTSCLI`. Los parámetros usados en el proyecto:

```bash
cd /OpenBTS
sudo ./OpenBTSCLI
```

Dentro de la consola interactiva de OpenBTSCLI:

```text
config GSM.Radio.Band 900
config GSM.Radio.C0 51
config GSM.Identity.MCC 716
config GSM.Identity.MNC 17
config GSM.Identity.LAC 1000
config GSM.Identity.CI 10
config GSM.Radio.PowerManager.MaxAttenDB 0
config GSM.Timer.T3212 6
```

Descripción rápida de cada parámetro:

| Parámetro | Valor | Significado |
|---|---|---|
| `GSM.Radio.Band` | 900 | Banda GSM 900 MHz |
| `GSM.Radio.C0` | 51 | ARFCN del canal de control (935.2 MHz en DL) |
| `GSM.Identity.MCC` | 716 | Código de país móvil (Perú) |
| `GSM.Identity.MNC` | 17 | Código de red móvil (experimental) |
| `GSM.Identity.LAC` | 1000 | Área de localización (valor de laboratorio) |
| `GSM.Identity.CI` | 10 | Identidad de celda (valor de laboratorio) |
| `GSM.Radio.PowerManager.MaxAttenDB` | 0 | Máxima potencia permitida por el hardware (sin atenuación adicional) |
| `GSM.Timer.T3212` | 6 | Periodo de actualización de localización (× 6 min) |

Salir de la consola con `exit` una vez aplicada la configuración (no es necesario reiniciar
OpenBTS si aún no se ha iniciado el proceso principal).

---

## 9. Arranque de OpenBTS y primeras pruebas

> **Importante:** OpenBTS debe ejecutarse siempre desde su propio directorio de instalación
> (`/OpenBTS`), **no** desde el directorio `home` del usuario, ya que el proceso busca sus
> archivos de recursos con rutas relativas a ese directorio.

```bash
# Verificar que el USRP está conectado
lsusb | grep -i "Ettus\|National Instruments"

# Ir al directorio correcto y arrancar OpenBTS
cd /OpenBTS
sudo ./OpenBTS
```

Esperar el mensaje de inicialización exitosa del `Transceiver52M` y la confirmación de enlace con
el USRP (puede tardar hasta ~20 segundos, ver parche de la sección 7.3).

En una segunda terminal, verificar el estado del sistema:

```bash
sudo ./OpenBTSCLI
```

Dentro de la consola:

```text
show
stats
```

**Prueba de campo:** en un teléfono móvil de prueba, realizar una búsqueda manual de operador.
La red debe aparecer identificada como **"71617"** (MCC 716 + MNC 17). Esto confirma que el BCCH
se está transmitiendo correctamente.

**Verificación de señalización bidireccional:** dentro de `OpenBTSCLI > stats`, revisar los
contadores `GSM.RACH.RequestsAccepted` y `GSM.RRLP.LUR`, que deben incrementarse al detectar el
terminal la red y realizar sus intentos de acceso/actualización de localización.

### Análisis espectral (opcional, para verificación de RF)

```bash
# Verificar visualmente la portadora en el ARFCN 51 (935.2 MHz)
uhd_fft -f 935.2M -s 200k
```

---

## 10. Instalación y configuración de Asterisk 16

Asterisk actúa como la capa de conmutación de llamadas, tanto para el núcleo GSM (SubscriberRegistry
+ interconexión) como, sobre todo en este proyecto, para el sistema de respaldo VoIP.

```bash
# Instalar dependencias de compilación de Asterisk
sudo apt install -y libxml2-dev libncurses5-dev libsqlite3-dev \
    libssl-dev libedit-dev uuid-dev libjansson-dev \
    subversion

# Descargar Asterisk 16 (rama LTS)
cd ~
wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-16-current.tar.gz
tar -xvzf asterisk-16-current.tar.gz
cd asterisk-16*/

# Instalar dependencias adicionales automáticamente
sudo contrib/scripts/install_prereq install

# Configurar, compilar e instalar
./configure
make menuselect.makeopts
make -j$(nproc)
sudo make install
sudo make samples
sudo make config
sudo ldconfig
```

Iniciar el servicio:

```bash
sudo systemctl enable asterisk
sudo systemctl start asterisk
sudo systemctl status asterisk
```

### Configuración de extensiones SIP para el respaldo VoIP

Editar el archivo de configuración SIP (según la versión de canal usada, `sip.conf` o
`pjsip.conf`):

```bash
sudo nano /etc/asterisk/sip.conf
```

Agregar al final del archivo (ejemplo con dos extensiones para pruebas):

```ini
[6001]
type=friend
context=internal
host=dynamic
secret=clave_segura_6001
disallow=all
allow=ulaw

[6002]
type=friend
context=internal
host=dynamic
secret=clave_segura_6002
disallow=all
allow=ulaw
```

Configurar el plan de marcado en `extensions.conf`:

```bash
sudo nano /etc/asterisk/extensions.conf
```

```ini
[internal]
exten => 6001,1,Dial(SIP/6001,20)
exten => 6002,1,Dial(SIP/6002,20)
```

Aplicar los cambios sin reiniciar todo el servicio:

```bash
sudo asterisk -rx "sip reload"
sudo asterisk -rx "dialplan reload"
```

---

## 11. Configuración del respaldo VoIP con Linphone

Este sistema de contingencia permite demostrar llamadas de voz en tiempo real de extremo a
extremo, independientemente del estado de la interfaz de radio GSM.

1. Instalar la aplicación **Linphone** en dos smartphones (disponible en Play Store / App Store).
2. Conectar ambos teléfonos a la misma red **WiFi local** que el servidor Asterisk.
3. En cada teléfono, dentro de Linphone, ir a **Configuración de cuenta SIP** y registrar:
   - **Usuario SIP:** `6001` (en el primer teléfono) / `6002` (en el segundo teléfono).
   - **Contraseña:** la definida en `sip.conf` (`clave_segura_6001` / `clave_segura_6002`).
   - **Dominio/servidor SIP:** la dirección IP local del equipo que corre Asterisk
     (verificar con `ip addr show` en el servidor).
   - **Transporte:** UDP (por defecto).
4. Confirmar que el estado de registro en la app muestra "conectado" (ícono verde).
5. Realizar una llamada de prueba marcando la extensión del otro teléfono (ej. desde el teléfono
   con extensión 6001, marcar `6002`).

Verificar el registro y las llamadas desde el servidor:

```bash
sudo asterisk -rx "sip show peers"
sudo asterisk -rx "core show channels"
```

---

## 12. Monitoreo, diagnóstico y comandos de operación diaria

```bash
# Consola interactiva de OpenBTS (estadísticas y estado)
cd /OpenBTS
sudo ./OpenBTSCLI
```

Comandos útiles dentro de la consola:

```text
stats
show
noise
```

Monitoreo de recursos del sistema durante la operación (útil para la prueba de estabilidad
prolongada):

```bash
# Uso de CPU y memoria en tiempo real
htop

# Actividad de disco (para diagnosticar STALE bursts asociados a I/O)
iostat -x 2
```

Verificación rápida de Asterisk:

```bash
sudo asterisk -rx "core show version"
sudo asterisk -rx "sip show peers"
sudo asterisk -rx "core show channels"
```

---

## 13. Solución de problemas encontrados (troubleshooting)

### 13.1 Errores `STALE burst` (Transceiver)

**Síntoma:** OpenBTS reporta repetidamente errores de tipo `STALE burst` en los logs del
Transceiver, indicando que las ráfagas llegan fuera de la ventana temporal esperada.

**Causa identificada:** latencia introducida por un adaptador **SATA-USB** usado como disco de
arranque, que impide al proceso `Transceiver52M` sostener el procesamiento de muestras I/Q en
tiempo real dentro de la ventana estricta de 577 µs por burst GSM.

**Diagnóstico:**

```bash
iostat -x 2
# Observar columna %util y await del disco durante la ejecución de OpenBTS
```

**Solución definitiva:** migrar la instalación del sistema operativo del disco externo (SATA-USB)
a un disco interno.

Pasos previos recomendados antes de reinstalar/migrar:

```bash
# Desde una unidad Live USB, verificar y reparar el sistema de archivos
# (reemplazar /dev/sdXN por la partición real, identificada con lsblk)
sudo umount /dev/sdXN
sudo fsck -f /dev/sdXN
```

Si aparecen errores de journal ext4 (frecuentes tras apagados forzados en sesiones previas),
`fsck` los repara automáticamente en modo interactivo (`-y` para aceptar todas las reparaciones
sugeridas si se ejecuta de forma no interactiva):

```bash
sudo fsck -fy /dev/sdXN
```

Después de la reparación, proceder con una instalación limpia en el disco interno siguiendo la
sección 1, o clonar el sistema existente al disco interno con herramientas como `dd` o `clonezilla`.

### 13.2 El USRP B210 no es reconocido por el sistema

```bash
lsusb
```

Si no aparece el dispositivo Ettus/National Instruments:

- Verificar que el cable sea USB 3.0 (los cables USB 2.0 o de solo carga no funcionan).
- Conectar directamente al puerto del equipo, evitando hubs USB no alimentados.
- Revisar las reglas udev (sección 5):

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
uhd_find_devices
```

### 13.3 Terminal no completa el registro (LUR rechazado)

**Causa:** el par IMSI/Ki de la SIM utilizada no está registrado en la base
`SubscriberRegistry` de OpenBTS, o la SIM es comercial (Ki no accesible).

**Mitigación durante el proyecto:** uso de una SIM programable (tipo SySmoSIM) con IMSI/Ki
conocidos, registrados manualmente en la base de datos, o habilitación temporal del modo de
autenticación abierta de OpenBTS **solo con fines de demostración académica**:

```bash
cd /OpenBTS
sudo ./OpenBTSCLI
```

```text
config Control.LUR.OpenRegistration 1
```

> Advertencia: este modo omite la verificación de Ki y **no debe usarse fuera de un entorno
> experimental controlado**.

### 13.4 Conflicto de arranque GRUB / firmware Insyde H2O

Ver sección 2. El ajuste de **Boot Priority** debe realizarse directamente desde el menú de
BIOS/UEFI (tecla F2 al inicio), no mediante comandos de sistema operativo.

### 13.5 Timeout al iniciar el Transceiver

**Síntoma:** OpenBTS cierra el proceso Transceiver antes de que el USRP termine de inicializarse.

**Solución:** confirmar que el parche de la sección 7.3 (timeout aumentado a ~20 segundos) fue
aplicado y que el binario fue recompilado después del cambio.

---

## 14. Procedimiento de apagado seguro

Dado que se identificó corrupción de journal ext4 asociada a apagados forzados, seguir siempre
este orden:

```bash
# 1. Detener OpenBTS de forma ordenada (Ctrl+C en la terminal donde corre,
#    o el comando de apagado si se ejecuta como servicio)

# 2. Detener Asterisk de forma ordenada
sudo systemctl stop asterisk

# 3. Confirmar que no quedan procesos del Transceiver activos
ps aux | grep -i transceiver

# 4. Desconectar el USRP B210 solo después del paso anterior

# 5. Apagar el sistema operativo de forma estándar
sudo shutdown now
```

**Nunca** desconectar la alimentación directamente ni forzar el apagado mientras OpenBTS o
Asterisk estén en ejecución.

---

## 15. Checklist rápido de arranque (resumen)

Para sesiones posteriores, una vez que el sistema ya fue instalado y configurado siguiendo todo
lo anterior, el arranque diario se reduce a:

```bash
# 1. Conectar el USRP B210 (USB 3.0 directo) y verificar reconocimiento
lsusb | grep -i "Ettus\|National Instruments"
uhd_find_devices

# 2. Iniciar OpenBTS (desde su propio directorio, obligatorio)
cd /OpenBTS
sudo ./OpenBTS

# 3. En otra terminal, iniciar/verificar Asterisk
sudo systemctl start asterisk
sudo systemctl status asterisk

# 4. Verificar estado de OpenBTS
sudo ./OpenBTSCLI
# dentro de la consola: stats / show

# 5. Prueba de campo: búsqueda manual de operador en el terminal de prueba
#    -> debe aparecer la red "71655"

# 6. Prueba de respaldo VoIP: abrir Linphone en ambos teléfonos y llamar
#    entre las extensiones 6001 y 6002
```

---

*Documento elaborado como parte del proyecto de Comunicaciones Móviles — UNMSM/FIEE. Los comandos
y parches aquí documentados reflejan el proceso real seguido durante el desarrollo del proyecto,
incluyendo los ajustes necesarios para resolver problemas de compatibilidad entre OpenBTS y UHD
3.15, así como las limitaciones de hardware encontradas (adaptador SATA-USB, disponibilidad de
SIM programables).*
