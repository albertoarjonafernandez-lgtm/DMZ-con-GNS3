# DMZ-con-GNS3
# Laboratorio: Despliegue de Seguridad Perimetral y DMZ con pfSense en GNS3

## 1. **Objetivo del Laboratorio**

Diseñar, desplegar y asegurar una infraestructura de red segmentada utilizando un firewall pfSense virtualizado. El objetivo principal es aislar un servicio expuesto al público (Servidor Web Nginx) en una Zona Desmilitarizada (DMZ) para proteger la red de área local (LAN) frente a posibles compromisos del servidor, permitiendo al mismo tiempo el acceso seguro a internet.

## 2. Conceptos Teóricos Clave

- **Zona Desmilitarizada (DMZ):** Es una subred aislada que se encuentra entre la red interna segura (LAN) y una red externa insegura (Internet/WAN). Los servidores públicos (Web, Correo, DNS) se colocan aquí. Si un atacante compromete la DMZ, el firewall le impedirá pivotar hacia la LAN.
- **Firewall de Inspección de Estado (Stateful):** pfSense analiza el estado de las conexiones activas. Si un equipo de la LAN inicia una conexión hacia la DMZ, el firewall permite automáticamente el tráfico de vuelta sin necesidad de crear una regla explícita.
- **Network Address Translation (NAT):** Permite enmascarar las direcciones IP privadas de los equipos internos detrás de la IP pública del firewall al salir a internet. En su variante **Port Forwarding (NAT 1:1 o DNAT)**, permite que las peticiones externas a un puerto específico del firewall se redirijan al servidor interno.
- **Reglas Top-Down:** Las listas de control de acceso (ACL) en el firewall se leen de arriba hacia abajo. La primera regla que coincide con el tráfico es la que se aplica ("First match wins").

---

## 3. Topología y Esquema de Direccionamiento IP

El entorno consta de tres zonas de red conectadas a las tres interfaces (tarjetas de red) del firewall pfSense:

- **Zona WAN (Internet):**
    - **Interfaz pfSense:** `em0` (WAN)
    - **Dirección IP:** Asignada por DHCP (Red virtual `192.168.122.0/24` provista por GNS3/Linux).
    - **Función:** Simula la conexión pública al proveedor de internet (ISP).
- **Zona LAN (Red Interna Segura):**
    - **Interfaz pfSense:** `em1` (LAN)
    - **Dirección IP pfSense (Gateway):** `192.168.10.1/24`
    - **Equipo Cliente:** VPCS (PC1) con IP `192.168.10.10`.
    - **Función:** Red de usuarios o administración que requiere máxima protección.
- **Zona DMZ (Servicios Expuestos):**
    - **Interfaz pfSense:** `em2` (OPT1)
    - **Dirección IP pfSense (Gateway):** `10.10.10.1/24`
    - **Servidor Web:** Contenedor Docker (Nginx) con IP `10.10.10.10`.
    - **Función:** Alojamiento del servicio web público.
     <img width="1920" height="1016" alt="image" src="https://github.com/user-attachments/assets/a617654c-e97f-4490-bc79-689de9f607ac" />

- **Tabla de Direccionamiento**
  <img width="1472" height="582" alt="image" src="https://github.com/user-attachments/assets/89452c16-9802-4468-99a6-8d852c46e720" />

     ## 4. Proceso de Implementación Técnica

### Fase 1: Configuración Base de Nodos y Enrutamiento

Para que los equipos puedan comunicarse a través de distintas subredes, es imperativo definir su puerta de enlace predeterminada (Gateway).

- **PC1 (LAN):** Se asigna la IP estática y se define el pfSense como su ruta de salida.
    - Comando VPCS: `ip 192.168.10.10/24 192.168.10.1`
- **Servidor Nginx (DMZ):** Al ser un contenedor, la configuración de red se define en su archivo `interfaces`.
    - Configuración aplicada:Plaintext
        
        `auto eth0
        iface eth0 inet static
            address 10.10.10.10
            netmask 255.255.255.0
            gateway 10.10.10.1`
        

### Fase 2: Despliegue y Ajustes Críticos de pfSense

Tras la instalación del sistema operativo FreeBSD subyacente de pfSense, se realizaron ajustes vitales para entornos de laboratorio:

- **Asignación de Interfaces:** Mapeo de tarjetas de red virtuales (`em0`, `em1`, `em2`) a las zonas lógicas.
- **Evasión de Bloqueo WAN (Anti-Lockout):** Por defecto, pfSense deniega el acceso a la interfaz de administración desde la WAN. Se ejecutó el comando de consola `pfSsh.php playback enableallowallwan` para permitir la administración inicial desde el host Linux.
- **Deshabilitación de RFC1918:** En *Interfaces > WAN*, se desactivó la opción `Block RFC1918 Private Networks`.
    - *Justificación técnica:* Dado que la "WAN" en este laboratorio es gestionada por el nodo NAT de GNS3 entregando IPs privadas (ej. 192.168.x.x), pfSense bloquearía legítimamente este tráfico asumiendo que es "Spoofing" (falsificación de IP) si esta opción permaneciera activa.

### Fase 3: Políticas de Seguridad y Apertura de Puertos

La lógica de seguridad aplicada define una postura de confianza cero (Zero Trust) para la DMZ.

**Reglas de Firewall (Pestaña OPT1 / DMZ):**

1. **Regla de Bloqueo a LAN:** Action: `Block` | Source: `OPT1 net` | Destination: `LAN net`.
    - *Propósito:* Evita movimientos laterales. Impide que un atacante que comprometa el Nginx alcance la base de datos o los ordenadores de la LAN.
2. **Regla de Permiso a Internet:** Action: `Pass` | Source: `OPT1 net` | Destination: `Any`.
    - *Propósito:* Permite al servidor web descargar actualizaciones del sistema operativo o paquetes desde repositorios externos.

**Reglas NAT Port Forwarding (Pestaña NAT):**

- **Regla:** Interface: `WAN` | Protocol: `TCP` | Dest. Port: `80 (HTTP)` | Target IP: `10.10.10.10` | Target Port: `80`.
- *Propósito:* Redirige cualquier petición HTTP entrante en la IP "pública" del firewall hacia la IP privada del contenedor Nginx de forma transparente para el usuario final.

---

## 5. Troubleshooting (Resolución de Problemas)

Durante la implementación, se utilizaron comandos estándar de administración de sistemas Linux para verificar la integridad del servicio en la DMZ:

- **Comprobación de conectividad ICMP:** `ping 10.10.10.10` (para verificar enrutamiento y reglas de capa 3).
- **Monitorización de Procesos (PID):** `ps aux` (para confirmar la ejecución del motor Nginx y sus procesos worker).
- **Análisis de Puertos Abiertos:** `netstat -tulpn | grep 80` (para asegurar que el daemon web estaba a la escucha (LISTEN) en las interfaces correctas).
- **Comprobación de Servicio Local:** `curl localhost` (para certificar que el servidor web entregaba contenido antes de probar el acceso remoto a través del firewall).
