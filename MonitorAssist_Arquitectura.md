# Arquitectura y Despliegue de MonitorAssist

Este documento describe la estrategia de comunicaciones, aprovisionamiento de nodos y arquitectura de backend para el proyecto MonitorAssist basado en Meshtastic.

## 1. Estrategia de Envío LoRa (Malla y Privacidad)

MonitorAssist necesita enviar datos médicos sensibles (Pulsaciones, caídas, GPS) por la red LoRa de forma cifrada, garantizando que los nodos "normales" de Meshtastic puedan retransmitir el mensaje sin poder leerlo.

**Solución elegida: Canal Secundario con PSK (Broadcast)**
- El módulo MonitorAssist emite los paquetes de emergencia como `Broadcast` (`0xFFFFFFFF`) pero dirigidos a un **canal secundario** específico (ej. `AssistChann`).
- Se utiliza la clave pre-compartida (PSK) AES-256 de este canal para cifrar el *payload*.
- **Comportamiento en la malla:** Cualquier nodo Meshtastic estándar retransmitirá el "blob" binario cifrado (flooding normal) para aumentar el alcance, aunque no tenga la clave y no pueda leer el contenido.
- **Eficiencia del código:** El firmware construye el paquete de forma nativa utilizando `router->allocForSending()`, el cual hereda automáticamente el `hop_limit` general del nodo y usa la variable nativa `localPosition` para la latitud/longitud, garantizando la coherencia y estabilidad de memoria. Se usa Nanopb (`pb_encode_to_bytes`) para la codificación.

## 2. Aprovisionamiento de Nodos (Intercambio de claves)

Configurar una clave PSK AES-256 manualmente es propenso a errores. Se utilizará la estrategia nativa de Meshtastic para enrolar los nodos del sistema:

1. **Generación inicial (Gateway):** El canal `AssistChann` y su PSK se configuran inicialmente en el nodo que actúa como Gateway (Raspberry Pi). Desde la CLI de Meshtastic (ej. `meshtastic --qr`), se genera el esquema de URL único (`https://meshtastic.org/e/#...`).
2. **Exportación a QR:** Esta URL se renderiza como código QR (en la terminal o en el panel de control web).
3. **Escaneo de clientes:** Los usuarios con el nodo Emisor (MonitorAssist) o los nodos Receptores solo tienen que abrir la aplicación móvil nativa de Meshtastic, ir a "Canales" -> "Escanear QR", y escanear el código. Esto importa el canal, los parámetros de radio y la PSK correcta de forma instantánea.

## 3. Infraestructura Backend (Broker MQTT Privado Cloud)

Para asegurar la recepción de la alerta en dispositivos alejados y evitar las restricciones del broker MQTT público de Meshtastic (`mqtt.meshtastic.org`), se adopta una arquitectura de nube privada:

* **Topología:**
  - **Gateway Transparente:** La Raspberry Pi Zero conectada al nodo LoRa actúa únicamente de enlace (uplink). Está configurada para subir el tráfico del canal `AssistChann` a un servidor de Internet.
  - **El Servidor Cloud (VPS):** Un servidor remoto ejecuta un broker MQTT (ej. Mosquitto) y un backend/servidor web.
  - **Receptores Móviles:** Los nodos o apps receptoras se conectan vía Wi-Fi o datos móviles a ese mismo broker MQTT en la nube, suscribiéndose al canal.

* **El Flujo del Mensaje:**
  1. MonitorAssist emite la alerta (RF).
  2. El Gateway la recibe, detecta el canal `AssistChann` y sube el paquete cifrado al broker MQTT cloud en el topic correspondiente (ej. `msh/EU_868/2/c/AssistChann/!nodo`).
  3. El MQTT cloud lo distribuye instantáneamente a todos los nodos suscritos a ese topic.
  4. El servidor web/backend local en el VPS lee también los mensajes de ese topic, los almacena en la base de datos y nutre el Dashboard.

* **Ventajas:**
  - **Sin Filtros:** Control absoluto sobre el tráfico. No hay filtros de comunidad que puedan bloquear paquetes de emergencia.
  - **Privacidad Total:** Los datos de telemetría médica jamás cruzan servidores públicos ni de terceros.
  - **Persistencia Temporal:** El servidor web permite guardar el histórico de pulsaciones y emergencias incluso si los nodos receptores estaban desconectados.
  - **Sincronización:** El uso del propio timestamp embebido en el paquete de telemetría (`msg.timestamp = getValidTime(...)`) garantiza que el momento exacto del incidente viaje seguro hasta la base de datos, ignorando los posibles retrasos de la malla RF o de colas MQTT (`rx_time` del nodo receptor).
