# Análisis de Comunicaciones y Duty Cycle - Monitor Assist

Este documento detalla el comportamiento de red del dispositivo operando bajo el firmware de Meshtastic con el módulo Monitor Assist habilitado. Se analiza tanto el volumen de paquetes emitidos como los mecanismos de bajo nivel para asegurar el acceso ordenado al medio compartido (CSMA/CA).

## 1. Estimación de Mensajes y Duty Cycle (Peor Escenario Continuo)

Se calcula el número de paquetes emitidos en una ventana de 1 hora en el **peor caso posible**: estado de emergencia/alerta ininterrumpida durante los 60 minutos.

### Mensajes Base de Meshtastic (Configuración Client)
Bajo los ajustes por defecto de Meshtastic para evitar congestionar la red MESH, un nodo estándar emitirá:
*   **Device Telemetry**: 1 mensaje/hora (`default_telemetry_broadcast_interval_secs` = 3600s).
*   **Position (GPS)**: 1 mensaje/hora (`default_broadcast_interval_secs` = 3600s).
*   **NodeInfo**: 1 mensaje cada 3 horas (`default_node_info_broadcast_secs` = 10800s). En la peor hora posible, asumiendo coincidencia, se contabiliza como 1 mensaje.
*   **Total Meshtastic Base**: **3 mensajes / hora**.

### Mensajes de Monitor Assist (Estado de Emergencia)
La cadencia de envío está gobernada por la máquina de estados y las constantes de alerta del módulo:
*   **Primeros 2 minutos** (`INTERVALO_ENVIO_ALERTA_SECUNDARIA_MS` = 2 min): 
    * Se emite 1 paquete cada 30 segundos.
    * Subtotal = **4 mensajes**.
*   **Siguientes 58 minutos** (hasta completar la hora):
    * Se emite 1 paquete cada 10 minutos (`INTERVALO_ENVIO_ALERTA_SOSTENIDA_MS`).
    * Subtotal = 58 / 10 = **5.8 mensajes** (redondeado a 6).
*   **Total Monitor Assist**: **10 mensajes / hora**.

### Conclusión de Tráfico
El número total de paquetes emitidos será de **13 mensajes / hora**. Esta cifra garantiza que el dispositivo se mantenga sobradamente dentro de los límites legales del ciclo de trabajo (Duty Cycle, típicamente del 1% en la banda Europea 868 MHz), evitando la saturación del canal MESH. Además, los paquetes de Monitor Assist se envían como `broadcast` privado sin solicitar ACK explícito (`want_ack = false`), minimizando el tráfico de confirmación.

---

## 2. Gestión de la Transmisión de Radio (CSMA / Prevención de Colisiones)

Cuando el módulo de usuario invoca la transmisión de un mensaje (`service->sendToMesh()`), el sistema de Meshtastic no acciona el transceptor inmediatamente. Se implementa un mecanismo de prevención de colisiones análogo al CSMA/CA clásico:

### 2.1 Validación y Encolado
1. **Verificación de Región y Duty Cycle:** El router evalúa el límite regulatorio (`dutyCycle`). Si el canal está bajo una restricción de uso, el mensaje es denegado preventivamente e informa al usuario del tiempo de backoff obligatorio ("Duty cycle limit exceeded").
2. **Cola de Transmisión (`txQueue`):** Si se permite la emisión, el mensaje se introduce en una cola FIFO en memoria.

### 2.2 Retardo de Contienda (Contention Window)
El hilo controlador de la radio (`RadioLibInterface`) extrae el paquete de la cola, pero no emite inmediatamente. Invoca `setTransmitDelay()`, el cual asigna un retraso aleatorio:
*   Para mensajes locales (propios), se genera un retraso al azar de varios `slotTimes`.
*   El tamaño del tiempo base aumenta dinámicamente si el canal está congestionado (Channel Utilization).
*   Si el mensaje a retransmitir pertenece a otro nodo, el retraso se pondera por la relación señal a ruido (SNR) para dar prioridad temporal al nodo mejor posicionado.

### 2.3 Channel Activity Detection (CAD)
Al vencer el temporizador del paso anterior, el hardware LoRa entra en escucha activa (CAD) comprobando el medio electromagnético:
*   **Canal Libre:** El transmisor se enciende y el paquete se envía físicamente al aire (`startSend()`).
*   **Canal Ocupado:** Si el hardware escucha tráfico, la transmisión se **aborta**. La radio entra en modo recepción para capturar el paquete ajeno y reinicia todo el ciclo de retardo aleatorio (backoff) posponiendo su turno original.

### 2.4 Bloqueos Hardware (Radio Busy)
En la circunstancia donde la propia radio ya está enviando (o recibiendo) otro paquete, la invocación de la API de Meshtastic resulta simplemente en la acumulación de la petición en la memoria `txQueue`. El hilo principal de radio irá drenando la cola mediante los pasos de contienda y CAD a medida que el transmisor hardware vaya liberándose, lo cual puede alargar la latencia de envío desde milisegundos hasta múltiples segundos dependiendo del volumen del tráfico aéreo en la malla.
