# Proyecto Mini-SOC Doméstico: Despliegue de Wazuh SIEM & Detección de Amenazas

## Descripción del Proyecto
Este proyecto consiste en el diseño, despliegue y configuración de un entorno de laboratorio corporativo simulado para la monitorización de seguridad (Blue Team). El objetivo principal es implementar un sistema SIEM/XDR (**Wazuh**) centralizado para recolectar eventos de sistemas finales, analizar la telemetría ante simulaciones de ataques dirigidos (Red Team) y desarrollar capacidades de detección y respuesta ante incidentes.

---

## Fase 1: Diseño de Red e Infraestructura

### 1. Objetivos de Infraestructura
* Crear un escenario virtualizado aislado del entorno doméstico pero con salida controlada a Internet para actualizaciones.
* Segmentar la comunicación en un direccionamiento dedicado `10.0.2.0/24` (Red NAT) en VirtualBox.
* Validar conectividad cruzada e intercambio de tráfico ICMP entre todos los nodos del laboratorio.

### 2. Inventario de Activos (Laboratorio)

| Nombre del Activo | Sistema Operativo | Rol / Función | IP Asignada (DHCP) | Recursos Asignados |
| :--- | :--- | :--- | :--- | :--- |
| **Wazuh v4.14.5 OVA** | Wazuh Appliance (Linux) | Cerebro del SOC (SIEM/XDR) | `10.0.2.5` | 2 vCPUs / 4 GB RAM |
| **SOC-WIN11-VICTIM** | Windows 11 Pro | Endpoint Víctima (Agente) | `10.0.2.3` | 2 vCPUs / 4 GB RAM |
| **SOC-KALI-ATTACKER**| Kali Linux 2025.4 | Plataforma de Auditoría (Red Team) | `10.0.2.4` | 2 vCPUs / 2 GB RAM |

### 3. Pruebas de Conectividad Cruzada
Se realizaron verificaciones de conectividad mediante tráfico ICMP (Ping) para asegurar el flujo de telemetría y escenarios de ataque:
* **Wazuh Server -> Internet (`8.8.8.8`):** ÉXITO (Resolución DNS y acceso a repositorios).
* **Kali Linux -> Windows 11:** ÉXITO (Alcance directo para simulaciones ofensivas).
* **Windows 11 -> Wazuh Server:** ÉXITO (Envío de eventos de auditoría mediante el puerto del agente `1514/1515`).

###  Troubleshooting y Dificultades Resueltas
* **Migración de despliegue SIEM:** Se inició el proyecto realizando una instalación manual de Wazuh sobre Ubuntu Server; sin embargo, tras experimentar incompatibilidades persistentes con la generación de certificados SSL y permisos de servicios, se optó por migrar a la *Virtual Appliance (OVA)* oficial. Esto garantizó la estabilidad del ecosistema e índices de Elasticsearch/OpenSearch.
* **Aseguramiento de Endpoints:** Se ajustaron las reglas del Firewall de Windows en la máquina víctima para permitir tráfico ICMPv4 entrante únicamente desde el segmento del laboratorio, equilibrando la visibilidad de red sin exponer el equipo a interfaces externas.

---

## Fase 2: Despliegue del Ecosistema de Defensas (Blue Team)

### 1. Objetivos del Ecosistema
* Configurar el nodo centralizador (**Wazuh Manager**) como cerebro del SOC.
* Desplegar sensores y agentes de auditoría en los endpoints del laboratorio (`Windows 11` y `Kali Linux`) para obtener visibilidad total de eventos.
* Endurecer la telemetría mediante **Sysmon** (System Monitor de Sysinternals) y políticas de auditoría avanzadas en Windows.

### 2. Arquitectura de Telemetría y Flujo de Datos
* **Transmisión de Logs:** Los agentes recopilan eventos locales de seguridad y los envían cifrados al Wazuh Manager a través del puerto seguro **1515/1514**.
* **Procesamiento e Ingesta:** El servidor decodifica los eventos recibidos, aplica reglas de correlación e indexa las alertas en **OpenSearch** para su análisis y visualización en tiempo real desde el **Wazuh Dashboard**.

### 3. Estado del Clúster de Agentes

| ID | Nombre del Agente | Sistema Operativo | IP | Estado | Cobertura |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **001** | `SOC-WIN11-VICTIM` | Windows 11 Pro | `10.0.2.3` | **ACTIVE** | Telemetría de Event Viewer + Sysmon |
| **002** | `SOC-KALI-ATTACKER`| Linux (Kali 2025.4)| `10.0.2.4` | **ACTIVE** | Auditoría de procesos Linux y red |

### 4. Configuración Avanzada de Telemetría (Hardening)

####  Monitorización de Procesos con Microsoft Sysmon
* **Despliegue:** Instalación y ejecución del servicio Sysmon64 en el endpoint Windows 11 (`Sysmon64.exe -i .\sysmonconfig.xml`).
* **Filtrado Avanzado:** Aplicación de reglas de filtrado mediante la plantilla `sysmonconfig.xml` orientada a la detección de técnicas de ataque (creación de procesos anómalos, inyección de código DLL y conexiones de red sospechosas).
* **Impacto:** Apertura de visibilidad de Nivel 1 de auditoría sobre acciones que el visor de eventos estándar de Windows no registra por defecto.

####  Monitorización de Integridad de Archivos (FIM - File Integrity Monitoring)
* **Objetivo:** Implementar la detección de modificaciones no autorizadas en archivos críticos del sistema operativo para proteger el pilar de **Integridad** de la tríada CIA frente a persistencias o ejecución de scripts maliciosos.
* **Configuración:** Personalización del bloque `<syscheck>` en el archivo `ossec.conf` del agente de Windows 11 para auditar directorios clave con una frecuencia de refresco periódica (300 segundos) y eventos en tiempo real:

```xml
<syscheck>
  <!-- Frecuencia de escaneo periódico cada 5 minutos (en segundos) -->
  <frequency>300</frequency>

  <!-- Monitorización en TIEMPO REAL de carpetas sensibles -->
  <directories realtime="yes" check_all="yes">C:\Windows\System32\drivers\etc</directories>
  <directories realtime="yes" check_all="yes">C:\Users\Public</directories>

  <!-- Evitar falsos positivos ignorando archivos temporales -->
  <ignore type="sregex">.tmp$</ignore>
</syscheck>
