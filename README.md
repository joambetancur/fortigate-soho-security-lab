# Laboratorio Fortinet FCA: Implementación de Topología SOHO Segura con FortiGate VM

## 📝 Descripción del Proyecto
Este laboratorio práctico demuestra el despliegue y la configuración de un Firewall de Nueva Generación (NGFW) **FortiGate VM** en un entorno simulado de oficina pequeña o contingencia doméstica (SOHO). El objetivo principal es aplicar los conceptos fundamentales del nivel **Fortinet Certified Associate (FCA)**, garantizando la segmentación estricta de la red, control de acceso perimetral, traducción de direcciones de red (NAT) y la exposición segura de servicios críticos en una Zona Desmilitarizada (DMZ).

---

## 🗺️ Arquitectura y Topología de Red

El laboratorio se compone de tres segmentos de red lógicos conectados directamente a las interfaces de la máquina virtual FortiGate:

*   **WAN (Port 1):** Conectividad hacia el exterior (Internet) mediante asignación dinámica (DHCP).
*   **LAN (Port 2):** Red interna de usuarios de confianza. Segmento: `192.168.10.0/24`.
*   **DMZ (Port 3):** Zona aislada para servidores públicos. Segmento: `172.16.10.0/24`.

```text
       [ INTERNET / WAN ]
               |
         (Port 1 - DHCP)
         +-----------+
         | FortiGate |
         |    VM     |
         +-----------+
         (Port 2) (Port 3)
            |        |
            |        +--- [ Servidor Web DMZ ] (172.16.10.10:80)
            |
    [ Usuarios LAN ] (192.168.10.0/24)

# Configuración de Interfaz WAN
config system interface
    edit "port1"
        set alias "WAN"
        set mode dhcp
        set role wan
        set allowaccess ping https
    next
end

# Configuración de Interfaz LAN
config system interface
    edit "port2"
        set alias "LAN_Users"
        set mode static
        set ip 192.168.10.1 255.255.255.0
        set role lan
        set allowaccess ping https ssh
    next
end

# Configuración de Interfaz DMZ
config system interface
    edit "port3"
        set alias "Web_Server_DMZ"
        set mode static
        set ip 172.16.10.1 255.255.255.0
        set role dmz
        set allowaccess ping
    next
end
