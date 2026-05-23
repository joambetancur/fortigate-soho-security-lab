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

````
## ⚙️ Configuración Paso a Paso (CLI de Referencia)
1. Inicialización y Direccionamiento de Interfaces
Configuración manual de los roles y direccionamiento IP para garantizar que el tráfico fluya exclusivamente por las interfaces correctas.
````
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
````
2. Configuración del Servidor DHCP (LAN)
Asignación automática de direccionamiento IP para el segmento de usuarios internos.
````
config system dhcp server
    edit 1
        set interface "port2"
        set default-gateway 192.168.10.1
        set netmask 255.255.255.0
        config ip-range
            edit 1
                set start-ip 192.168.10.2
                set end-ip 192.168.10.254
            next
        end
        set dns-server1 8.8.8.8
        set dns-server2 8.8.4.4
    next
end
````
3. Publicación de Servicios mediante IP Virtual (VIP)
Mapeo perimetral para exponer el servidor web interno (TCP 80) hacia internet a través de la interfaz WAN.
````
config firewall vip
    edit "VIP_Public_Web_Server"
        set extip 0.0.0.0
        set extintf "port1"
        set portforward enable
        set mappedip "172.16.10.10"
        set extport 80
        set mappedport 80
    next
end
````
4. Políticas de Seguridad (Firewall Policies)
Aplicación de reglas basadas en el principio de menor privilegio.
````
config firewall policy
    # 1. Permitir salida de LAN a Internet con NAT
    edit 1
        set name "Permitir_LAN_a_Internet"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
        set logtraffic all
    next
    # 2. Permitir acceso externo al Servidor Web (DMZ) vía VIP
    edit 2
        set name "Acceso_Publico_a_Servidor_Web"
        set srcintf "port1"
        set dstintf "port3"
        set srcaddr "all"
        set dstaddr "VIP_Public_Web_Server"
        set action accept
        set schedule "always"
        set service "HTTP"
        set logtraffic all
    next
    # 3. Bloqueo explícito de LAN a DMZ (Aislamiento de Seguridad)
    edit 3
        set name "Bloquear_LAN_a_DMZ"
        set srcintf "port2"
        set dstintf "port3"
        set srcaddr "all"
        set dstaddr "all"
        set action deny
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end
````
📊 Verificación y Resultados (Evidencias): Todo queda registrado en la carpeta **images**:

#### A. Tabla de Políticas de Seguridad (GUI)
A continuación se muestra la correcta jerarquía y estado de las políticas configuradas desde el entorno gráfico de FortiOS:
![Tabla de Politicas](images/politics1.jpg)
![Tabla de Politicas](images/politics2.jpg)

#### B. Monitoreo de Logs y Tráfico mediante CLI (Local Traffic)
*Nota de ingeniería: Debido a limitaciones de hardware para desplegar hosts finales en el hipervisor, las pruebas de conectividad se originaron directamente desde la CLI de FortiOS forzando las IPs de las interfaces como origen (`ping-options source`). Por arquitectura del sistema operativo, estas trazas se auditan bajo el módulo **Local Traffic**, demostrando el correcto enrutamiento y la aplicación de las políticas perimetrales:*
![Registro del Trafico](images/log_traffic.jpg)

📌 Conclusiones del Laboratorio 1:
Control Perimetral: El uso de las políticas NGFW segmentó exitosamente la red, impidiendo la comunicación directa no deseada entre la zona de usuarios y la zona de servidores.

Seguridad DMZ: Al utilizar una IP Virtual (VIP), se expone únicamente el puerto específico necesario (TCP 80) hacia internet, ocultando por completo el direccionamiento IP real de la infraestructura interna de ataques de escaneo.

## 🛡️ Laboratorio 2: Perfiles de Seguridad y Control de Aplicaciones (NGFW)

### 📝 Descripción
En esta fase se transformó el firewall perimetral básico en un **Firewall de Nueva Generación (NGFW)** mediante la implementación de inspección de Capa 7. El objetivo fue aplicar el principio de menor privilegio sobre la regla de salida a Internet (`Permitir_LAN_a_Internet`), restringiendo el acceso a categorías web de alto riesgo y controlando el uso de aplicaciones que comprometen la productividad y el ancho de banda.

### ⚙️ Configuración de Perfiles UTM (Arquitectura)

#### 1. Filtrado Web (Web Filter)
Se creó el perfil personalizado `WF_LAN_Corporativo` bajo la base de datos de **FortiGuard**, aplicando políticas de bloqueo estricto a las categorías de **Adult/Mature Content** para evitar contenido no deseado o de alto impacto.

![Perfil Web Filter](images/perfil_web.jpg)

#### 2. Control de Aplicaciones (Application Control)
Mediante el análisis de firmas profundas de Capa 7, se desplegó el perfil `AC_LAN_Corporativo` para interceptar y denegar de forma heurística el tráfico de las categorías **Social.Media** (firmas de Facebook, Instagram, TikTok) y herramientas **P2P** (descargas BitTorrent), previniendo la fuga de información y el abuso del canal de datos.

![Perfil Application Control](images/perfil_apps.jpg)

### 🔗 Vinculación y Motores de Inspección en la Política
Ambos escudos de seguridad fueron acoplados directamente dentro de la política de firewall principal. Para asegurar la continuidad del laboratorio sin alteración de llaves criptográficas en esta etapa, se asoció el método de inspección en modo **Certificate Inspection**:

![Política NGFW Acoplada](images/politica_ngfw.jpg)

## 📌 Conclusiones del Laboratorio 2
* **Seguridad de Capa 7:** La activación de perfiles UTM demuestra que el firewall ya no solo inspecciona IPs y puertos (Capas 3 y 4), sino que analiza el comportamiento real del tráfico y el contenido de las peticiones.
* **Mitigación del Riesgo:** El bloqueo preventivo de categorías de reputación no deseada disminuye la superficie de ataque de la red SOHO de manera drástica, aislando los endpoints de servidores de comando y control (C2) o sitios de phishing conocidos.

---

## 🔍 Laboratorio 3: Inspección SSL/TLS Profunda (Deep SSL Inspection) y AntiVirus Perimetral

### 📝 Descripción
Este laboratorio aborda la eliminación del "punto ciego" del tráfico corporativo cifrado (HTTPS), el cual representa más del 95% de la navegación actual. Se implementó una arquitectura de tipo *Man-in-the-Middle* (MitM) controlada, delegando en el FortiGate la función de Proxy SSL para descifrar el tráfico entrante, inspeccionarlo con el motor criptográfico de FortiGuard en busca de código malicioso, y volverlo a cifrar de forma transparente antes de su entrega al endpoint.

### ⚙️ Configuración del Motor de Inspección y Seguridad

#### 1. Perfil Antivirus Corporativo en Modo Proxy
Se configuró el perfil `default` bajo un conjunto de funciones basadas en **Proxy (Proxy-based)**. Este modo es un requisito de diseño de FortiOS para realizar un análisis completo de archivos pesados sobre los protocolos estándar de transferencia (`HTTP`, `FTP`, `CIFS`, etc.), garantizando la interrupción y el bloqueo de binarios sospechosos o firmas de malware conocidas.

![Configuración Perfil Antivirus](images/perfil_av.jpg) 

#### 2. Despliegue de Deep Inspection en la Política Perimetral
Se modificó la política de control de acceso a internet para alternar el análisis superficial de certificados por el perfil de **Deep Inspection**. Al acoplar el descifrado SSL junto al motor de AntiVirus, el NGFW adquiere visibilidad total sobre la Capa 7 profunda, analizando el payload de las conexiones TLS dirigidas a dominios permitidos.

![Configuración Deep Inspection](images/politica_deep_inspection.jpg)

### Evidencia de Inspección Activa vía CLI (Sniffer)

Para verificar que el firewall está procesando las solicitudes de seguridad y comunicándose con FortiGuard para el análisis de firmas, se ejecutó un sniffer de paquetes en tiempo real:

![Log Trafico Lab 3](images/log_sniffer.jpg)

---

## 📌 Conclusiones del Laboratorio 3
* **Eliminación del Punto Ciego HTTPS:** Sin la inspección profunda (`deep-inspection`), los perfiles de seguridad como el Web Filter o el AntiVirus quedan completamente ciegos ante ataques modernos que utilizan canales HTTPS cifrados para distribuir malware.
* **Consideraciones de Producción:** Debido a que el firewall genera certificados dinámicos firmados por su propia CA interna (`Fortinet_CA_SSL`), en un entorno empresarial real es obligatorio distribuir este certificado de manera masiva en todos los dispositivos.
