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

### 🛠️ Troubleshooting y Dificultades Resueltas
* **Migración de despliegue SIEM:** Se inició el proyecto realizando una instalación manual de Wazuh sobre Ubuntu Server; sin embargo, tras experimentar incompatibilidades persistentes con la generación de certificados SSL y permisos de servicios, se optó por migrar a la *Virtual Appliance (OVA)* oficial. Esto garantizó la estabilidad del ecosistema e índices de Elasticsearch/OpenSearch.
* **Aseguramiento de Endpoints:** Se ajustaron las reglas del Firewall de Windows en la máquina víctima para permitir tráfico ICMPv4 entrante únicamente desde el segmento del laboratorio, equilibrando la visibilidad de red sin exponer el equipo a interfaces externas.
