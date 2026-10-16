# Estado del Proyecto: Gateway Meshtastic MQTT & Malla de Asistencia / Detección de Caídas

**Fecha de exportación:** Julio de 2026  
**Objetivo del Proyecto:** Crear un ecosistema de emergencia/asistencia basado en red LoRa Meshtastic.  
- **Nodos Emisores/Calle:** Nodos comerciales (ej. SenseCAP T1000-E) que detectan eventos/caídas o envían alertas de asistencia.
- **Nodos Repetidores:** Malla Meshtastic estándar (retransmiten paquetes binarios de forma transparente por la ciudad).
- **Nodo Central Gateway:** Raspberry Pi Zero 2 W con HAT LoRa SX1262 executando `meshtasticd` (Portduino), que interpreta los paquetes binarios modificados y los sube a un Broker MQTT propio.

---

## 1. Hardware y Topología de Red Central

### Hardware Central
- **Single Board Computer:** Raspberry Pi Zero 2 W
- **HAT LoRa:** Waveshare SX1262 (EU_868 MHz)
- **Host / Conectividad:** Servidor Windows conectado por USB a la Pi en modo Gadget Ethernet (`192.168.137.X` mediante ICS).

### Configuración de Red e Internet (ICS Windows)
- **Regla de Oro IP:** Si `hostname -I` en la Raspberry Pi devuelve una IP del rango `192.168.137.X`, el *Internet Connection Sharing* (ICS) de Windows está activo y la Pi tiene acceso a internet y NTP.
- **Mantenimiento ICS:** Si la Pi se reconecta a Windows y cambia el adaptador dinámico (ej. de Ethernet 3 a Ethernet 4), el ICS en Windows debe apuntar a la nueva interfaz activa.
- **Configuración de Servicios de red en la Pi:**
  - Panel Web Local: `https://10.12.194.1:9443/` (SSL auto-firmado).
  - API TCP local para clientes/scripts: Puerto `4403`.

---

## 2. Ajustes de Tiempo y Zona Horaria (POSIX)

### Sincronización del Sistema Operativo
- El reloj del SO Linux se sincroniza mediante `systemd-timesyncd` (NTP) cuando hay salida a internet.
- Consulta de estado: `timedatectl`.

### Configuración de Zona Horaria POSIX en Meshtastic (Madrid / España)
Para asegurar que Meshtastic calcule internamente el desfase de horario de verano/invierno autónomamente:
- **Cadena POSIX Oficial:** `CET-1CEST,M3.5.0,M10.5.0/3`
  - `CET`: Horario estándar de invierno (UTC+1 en código POSIX `-1`).
  - `CEST`: Horario de verano (UTC+2).
  - `,M3.5.0`: Inicia el último domingo de marzo.
  - `,M10.5.0/3`: Termina el último domingo de octubre a las 03:00 am.
- **Inyección por comando desde el Host:**
  ```powershell
  meshtastic --host 10.12.194.1 --set device.tzdef "CET-1CEST,M3.5.0,M10.5.0/3"