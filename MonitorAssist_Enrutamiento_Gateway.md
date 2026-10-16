# Enrutamiento y Gestión de Gateways para MonitorAssist

Este documento recoge las decisiones y aclaraciones arquitectónicas sobre cómo los nodos de MonitorAssist interactúan con la red Mesh, los Gateways, y la capa de MQTT para el enrutamiento de las alertas.

## 1. El Rol del Gateway (Transparencia del Payload)

No es necesario modificar el firmware ni recompilar el demonio `meshtasticd` de la Raspberry Pi (Gateway) para que reconozca el nuevo Protobuf de MonitorAssist.

- **Descifrar vs Entender:** El Gateway recibe el paquete cifrado por radio (AES-256) y, al tener la PSK del canal `AssistChann`, puede quitar ese "candado externo".
- **Agnosticismo:** Dentro del paquete se encuentra el campo `payload` en bruto con un `portnum = PRIVATE_APP`. El Gateway no necesita conocer la estructura `.proto` interna de MonitorAssist. Simplemente convierte ese bloque de bytes a Base64 y lo empuja como un paquete JSON al broker MQTT.
- **Decodificación Final:** La decodificación real de las variables (pulsaciones, GPS, caídas) ocurre exclusivamente en el servidor Backend Cloud o en la aplicación del teléfono receptor.

Dcoumentacion aqui: https://meshtastic.org/docs/software/integrations/mqtt/

Ejemplo de NodeRed con MQTT aqui: https://meshtastic.org/docs/software/integrations/mqtt/nodered/

## 2. Enrutamiento de MQTT hacia LoRa (Downlink)

Para garantizar que el mensaje pueda llegar a nodos sin conexión a internet (situados en otras ubicaciones geográficas), se utiliza la función nativa de enlace cruzado (Uplink/Downlink) a través de un **Broker MQTT Privado Cloud**. No se recomienda el broker público de Meshtastic debido a sus fuertes filtros anti-spam y rate-limiting.

**Flujo de Comunicación:**
1. **Emisión (Radio):** La pulsera de MonitorAssist emite la alerta en Broadcast al canal `AssistChann`.
2. **Uplink:** El Gateway "A" local escucha el canal (configurado con `uplink_enabled = true`) y sube el paquete a tu broker privado.
3. **Distribución en la Nube:** Tu servidor Cloud almacena el mensaje en base de datos. Simultáneamente, el Broker MQTT redistribuye el mensaje al instante.
4. **Downlink:** Un Gateway "B" remoto (configurado con `downlink_enabled = true` en `AssistChann`) recibe el mensaje desde MQTT, lo empaqueta de nuevo y lo emite por su antena LoRa.
5. **Recepción Final (Radio):** El nodo del receptor (familiar), que no tiene internet, captura la señal del Gateway B, la descifra usando su clave PSK y lanza la alerta.

## 3. Destinatarios y Filtro de Nodos

**No se añade ningún campo de "receptor destino" al Protobuf de MonitorAssist.** La filosofía de Meshtastic ya proporciona herramientas nativas mucho más limpias y eficientes para gestionar a quién le llega el mensaje:

- **Segregación por Canal (Recomendada):** Cada paciente/familia tiene su propia PSK y su propio canal (ej. `JuanSOS`, `MariaSOS`). El envío se hace en `Broadcast (0xFFFFFFFF)`. Si Juan se cae, la alerta la reciben exclusivamente los cuidadores que han importado el QR de `JuanSOS`. Todos los dispositivos de esos cuidadores suenan a la vez (redundancia ideal en emergencias).
- **Identidad del Remitente:** El "sobre" `MeshPacket` de Meshtastic ya incluye automáticamente el ID único del emisor en el campo `from`. Si hay varios pacientes en el mismo canal (p.ej. una residencia), el enfermero o servidor web solo necesita mirar el ID de origen para saber qué paciente exacto es el que tiene la emergencia.
- **Direccionamiento Directo:** En caso de necesitar mandar una alerta en silencio solo a un cuidador específico (sin alertar a los demás familiares), se puede sustituir el Broadcast por la MAC/NodeNum exacto del cuidador en el código C++ (`p->to = 0x1234ABCD;`). El resto de nodos del canal lo ignorarán por hardware/software.

## 4. Interfaces de Configuración y Roles de la Flota

La versión de Linux para Raspberry Pi (`meshtasticd`) es un nodo completo y no obliga al uso exclusivo de la línea de comandos (CLI).

**Opciones de Administración del Gateway:**
1. **Web UI Nativa:** `meshtasticd` expone un servidor web local. Accediendo a la IP de la Raspberry Pi desde un navegador en la misma red local (`http://<IP_DE_LA_RASPBERRY>`) se dispone de un panel gráfico completo para administrar el nodo cómodamente.
2. **App Móvil (vía TCP/IP):** Desde la aplicación oficial de Meshtastic, en los ajustes de conexión, se puede elegir conectarse por red (TCP/IP) a la IP de la Raspberry, logrando el mismo control que si estuviera enlazada por Bluetooth.

### Organización y Configuración Ideal de los Nodos

Para asegurar la máxima eficiencia y escalabilidad, cada actor en la malla asume un rol y configuración específicos para el canal secundario (`AssistChann`):

*   **El Gateway (La Raspberry Pi):**
    *   **Canal 0:** `LongFast` (para beneficiarse del rebote en nodos de la comunidad).
    *   **Canal 1 (`AssistChann`):** Se genera la PSK. Se deben activar obligatoriamente **`uplink_enabled`** (subida) y **`downlink_enabled`** (bajada).
    *   **Módulo MQTT:** Se configura apuntando a la IP/dominio del broker privado en la nube.
    *   **Rol:** `ROUTER` o `CLIENT` (para maximizar la capacidad de enrutamiento y mantener la radio encendida).
    *   **Aprovisionamiento:** Utiliza la Web UI para mostrar en pantalla el código QR de configuración del Canal 1.

*   **El Nodo Emisor (Pulsera MonitorAssist):**
    *   **Canal 0:** `LongFast`.
    *   **Canal 1 (`AssistChann`):** Se configura al instante **escaneando el QR** del Gateway con el teléfono conectado por Bluetooth.
    *   **MQTT:** Desactivar explícitamente el `uplink` y `downlink` para ahorrar batería extrema, ya que no tiene/usa internet propio.
    *   **Rol:** `SENSOR` o `TRACKER` (optimizado para enviar ráfagas de datos y volver a dormir).

*   **El Nodo Receptor (Dispositivo del Familiar):**
    *   **Canal 0:** `LongFast`.
    *   **Canal 1 (`AssistChann`):** Escanear el mismo QR del Gateway.
    *   **MQTT:** Dejar desactivado el enrutamiento para que no intente subir tráfico usando el internet del teléfono, cediendo esa responsabilidad al Gateway.
    *   **Notificaciones:** App del teléfono configurada para alertar visual y sonoramente al recibir mensajes por este canal.
    *   **Rol:** `CLIENT`.

## 5. Lógica del Nodo Receptor (Notificaciones Locales de Hardware)

El módulo MonitorAssist utiliza una única base de firmware portable tanto para el emisor como para el receptor. Si un nodo receptor (como un *Wio Tracker 1110* o *Heltec*) no tiene la pulsera BLE conectada, la máquina de estados permanecerá pasiva, esperando y procesando únicamente las alertas que lleguen por radio.

**Recepción Automática y Transparente:**
Al registrar el `portnum = meshtastic_PortNum_PRIVATE_APP` en el constructor del módulo, el Core de Meshtastic intercepta automáticamente los paquetes dirigidos a este puerto y canal (tras descifrarlos), e invoca el método `handleReceivedProtobuf()`. No es necesario implementar bucles de lectura ni filtros manuales.

**Portabilidad Acústica (La API `buzz.h`):**
Para alertar físicamente al cuidador cuando el nodo receptor captura el mensaje, MonitorAssist interactúa con el hardware de sonido. 
*   **La trampa del hardware directo:** Usar funciones tipo `tone(PIN, FREQ)` es una mala práctica en Meshtastic, ya que rompe la portabilidad (los pines cambian según la placa, ESP32 vs nRF52) e ignora las preferencias de "No Molestar" que el usuario haya establecido en la app.
*   **La solución nativa:** Se utiliza la cabecera de abstracción `buzz/buzz.h` de Meshtastic. Dentro de `handleReceivedProtobuf()`, se extraen las banderas de emergencia y se reacciona usando la API oficial:
    *   Se usa `playComboTune()` para reproducir una melodía compleja y llamativa al detectar una **Caída**.
    *   Se utiliza un bucle de `playLongBeep()` para simular un monitor de constantes vitales al detectar un fallo de **Pulso Cardíaco**.

De esta forma, el firmware se mantiene agnóstico a la placa base, compilando limpiamente y generando respuestas acústicas distintas sin riesgo de cortocircuitos o cuelgues por usar pines no mapeados.
