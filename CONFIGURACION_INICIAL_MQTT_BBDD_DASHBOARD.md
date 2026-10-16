# Configuración y Despliegue del Backend: MonitorAssist

Este documento recoge todas las decisiones arquitectónicas, componentes de software y fases de despliegue para el ecosistema en la nube del TFM MonitorAssist.

---

## 1. Visión General de la Arquitectura (Fases de Desarrollo)

Para asegurar un desarrollo rápido y sin costes, el proyecto se divide en dos fases:
- **Fase 1 (Desarrollo Local):** Todo el ecosistema se ejecuta en el ordenador del desarrollador. Se utilizan contenedores locales o emuladores (LocalStack) para programar la lógica sin depender de internet ni consumir créditos cloud.
- **Fase 2 (Producción Cloud-Native AWS):** Migración de la lógica local probada hacia los servicios gestionados de Amazon Web Services (AWS IoT Core, Amazon Aurora, EC2).

**Flujo de Telemetría:**
`Nodo LoRa` ➡️ `Gateway Pi` ➡️ `Broker MQTT (Local o AWS)` ➡️ `Node-RED` ➡️ `BBDD SQL (Local o Aurora)` ➡️ `Grafana`

---

## 2. Fase 1: Entorno de Desarrollo Local

Durante esta fase, el ordenador personal actuará como el servidor de nube simulado.

### Opciones de Entorno Local:
1. **LocalStack (Pro):** Emula directamente los servicios de AWS (IoT Core, RDS). Ideal si se quiere tener compatibilidad 100% nativa con las APIs de AWS antes de migrar.
2. **Docker Compose Estándar:** Levanta contenedores locales de Mosquitto (Broker MQTT), PostgreSQL, Node-RED y Grafana. Es el método más rápido para empezar a programar la lógica de decodificación y el dashboard hoy mismo sin configurar las credenciales emuladas de AWS.

### El Reto Principal: La Decodificación (Node-RED)
El foco absoluto de esta fase es programar el "Cerebro" (Node-RED) para que resuelva la lógica principal de la aplicación:
1. Conectarse al Broker MQTT local.
2. Recibir el JSON del Gateway Meshtastic y extraer el campo `payload` en Base64.
3. Desencriptar el Base64 y usar el archivo `monitor_assist.proto` para extraer la latitud, longitud y pulsaciones.
4. Ejecutar la lógica de Geofencing y alertas (Telegram).
5. Hacer un `INSERT` en la base de datos local estructurada.

---

## 3. Fase 2: Producción (Nativa AWS - Enterprise)

Una vez el flujo de Node-RED local sea perfecto, se realizará el salto a la nube final:

- **Pilar 1 (Ingesta):** Se reemplazará el broker MQTT local por **AWS IoT Core**.
- **Pilar 2 (Base de Datos):** Se reemplazará la BBDD local por un clúster gestionado de **Amazon Aurora** (con esquema tradicional SQL).
- **Pilar 3 (Visualización):** Se desplegará Grafana (ya sea en EC2 o Amazon Managed Grafana) conectado a Aurora.
- **Pilar 4 (Lógica):** El archivo de flujos (`flows.json`) programado en la Fase 1 se exportará y se cargará directamente a un Node-RED alojado en una instancia **AWS EC2**.

---

## 4. Siguientes Pasos
El siguiente hito del proyecto es levantar el entorno local (Broker + Node-RED) para comenzar a inyectar mensajes de prueba reales de la red LoRa (usando los scripts de Python disponibles) y programar el decodificador de Protobuf.
