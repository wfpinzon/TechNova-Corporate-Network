# 🏢 Infraestructura de Red Corporativa y Seguridad Perimetral - TechNova Solutions

¡Bienvenido a mi portafolio de redes! Este proyecto demuestra el diseño, segmentación, implementación y endurecimiento de seguridad (*device hardening*) de una infraestructura de red LAN y WLAN para una empresa mediana ficticia llamada **TechNova Solutions**.

El proyecto fue desarrollado y validado utilizando **Cisco Packet Tracer**, aplicando los conocimientos consolidados en el curso de **Fundamentos de Redes (Networking Essentials)** de *Cisco Networking Academy*.

---
## 🚀 Características Principales
*   **Segmentación Avanzada (VLSM):** Optimización del espacio de direcciones IP privadas mitigar el desperdicio de hosts.
*   **Arquitectura Router-on-a-Stick:** Virtualización de interfaces lógicas para el enrutamiento inter-VLAN eficiente mediante un único enlace físico corporativo.
*   **Asignación Dinámica (DHCP Relay):** Centralización de servicios de red e implementación de agentes de retransmisión (*helper-address*).
*   **Seguridad y Control de Acceso (ACL):** Aislamiento de tráfico crítico corporativo frente a redes de invitados mediante listas de control de acceso extendidas.
*   **Mitigación de Riesgos (Device Hardening):** Configuración de acceso remoto seguro vía SSH v2, inhabilitación de protocolos vulnerables (Telnet) y encriptación de credenciales.

---

## 🗺️ 1. Topología de la Red
La red está organizada visualmente en cuatro zonas lógicas independientes (TI/Servidores, Ventas, Administración e Invitados).

<img src="Infraestructura%20de%20Red%20Corporativa%20y%20Seguridad%20Perimetral%20-%20TechNova%20Solutions.png" alt="Topología de Red Corporativa" width="800">



---

## 📐 2. Fase de Direccionamiento IP (Cálculo VLSM)
Tomando como base la red privada **`172.16.0.0 /22`** (Máscara: `255.255.252.0`), se realizó el cálculo matemático de subredes ordenado de mayor a menor requerimiento de hosts:

| Departamento / Red | Hosts Requeridos | IP de Red | Máscara (Decimal) | Primera IP Útil (Gateway) | Última IP Útil | Dirección de Broadcast |
| :--- | :---: | :--- | :--- | :--- | :--- | :--- |
| **Wi-Fi Invitados** | 50 | `172.16.0.0 /26` | `255.255.255.192` | `172.16.0.1` | `172.16.0.62` | `172.16.0.63` |
| **Ventas y Atención** | 40 | `172.16.0.64 /26` | `255.255.255.192` | `172.16.0.65` | `172.16.0.126` | `172.16.0.127` |
| **Administración** | 20 | `172.16.0.128 /27` | `255.255.255.224` | `172.16.0.129` | `172.16.0.158` | `172.16.0.159` |
| **TI y Servidores** | 15 | `172.16.0.160 /27` | `255.255.255.224` | `172.16.0.161` | `172.16.0.190` | `172.16.0.191` |

### Asignación de Infraestructura Estática (Zona de TI):
*   **Servidor DHCP:** `172.16.0.162` (Encargado de los pools corporativos).
*   **Servidor DNS:** `172.16.0.163` (Resolución local de dominios).
*   **Servidor Web Intranet:** `172.16.0.164` (Servicio HTTP interno corporativo).

---

## 💻 3. Configuración Lógica de Infraestructura (Cisco IOS)

A continuación, se detallan los bloques de comandos clave aplicados en la interfaz de línea de comandos (**CLI**) del **Router Central (ISR4321)**:

### Enrutamiento Inter-VLAN (Subinterfaces) y DHCP Relay
```ios
Router-Central# configure terminal

! --- INTERFAZ WI-FI INVITADOS ---
interface GigabitEthernet0/0/0
 ip address 172.16.0.1 255.255.255.192
 no shutdown
 exit

! --- ENLACE TRONCAL CORPORATIVO ---
interface GigabitEthernet0/0/1
 no shutdown
 exit

! --- SUBINTERFAZ VENTAS (VLAN 64) ---
interface GigabitEthernet0/0/1.64
 encapsulation dot1Q 64
 ip address 172.16.0.65 255.255.255.192
 ip helper-address 172.16.0.162
 exit

! --- SUBINTERFAZ ADMINISTRACIÓN (VLAN 128) ---
interface GigabitEthernet0/0/1.128
 encapsulation dot1Q 128
 ip address 172.16.0.129 255.255.255.224
 ip helper-address 172.16.0.162
 exit

! --- SUBINTERFAZ TI Y SERVIDORES (VLAN 160) ---
interface GigabitEthernet0/0/1.160
 encapsulation dot1Q 160
 ip address 172.16.0.161 255.255.255.224
 exit
```

### Seguridad de Perímetro (ACL Extendida)
Para aislar la red de invitados de los recursos privados corporativos, se desplegó una Lista de Control de Acceso que bloquea el tráfico de origen inalámbrico hacia los segmentos internos, permitiendo únicamente la comunicación con su puerta de enlace básica.

```ios
! --- DEFINICIÓN DE POLÍTICAS DE TRÁFICO ---
ip access-list extended FILTRO_INVITADOS
 deny ip 172.16.0.0 0.0.0.63 172.16.0.64 0.0.0.63    ! Bloqueo a Ventas
 deny ip 172.16.0.0 0.0.0.63 172.16.0.128 0.0.0.31   ! Bloqueo a Administración
 deny ip 172.16.0.0 0.0.0.63 172.16.0.160 0.0.0.31   ! Bloqueo a TI y Servidores
 permit ip any any                                   ! Permitir el resto (Salida WAN)
 exit



! --- APLICACIÓN EN LA INTERFAZ DE ENTRADA ---
interface GigabitEthernet0/0/0
 ip access-group FILTRO_INVITADOS in
 exit

```


---

## 🧪 4. Fase de Pruebas y Validación (Pings)
Las políticas de red y conectividad se validaron con éxito mediante pruebas de conectividad de extremo a extremo:

1.  **Validación DHCP:** Las estaciones de trabajo de Ventas y Administración obtienen correctamente parámetros de direccionamiento dinámico dentro de sus respectivos rangos de subred.
2.  **Validación Wi-Fi:** Los dispositivos de la zona de invitados se asocian de forma inalámbrica mediante seguridad robusta, obteniendo direccionamiento LAN local y traduciendo su tráfico vía NAT a través de la IP estática WAN (`172.16.0.2`) asignada en el puerto de Internet.
3.  **Auditoría de la ACL (Éxito de Seguridad):**
    *   Un comando `ping` desde el **Smartphone de Invitados** hacia el **Servidor Web Corporativo (172.16.0.164)** arroja el estado `Destination host unreachable`, validando el correcto funcionamiento del bloqueo.
    *   Un comando `ping` desde las estaciones de **Ventas** hacia los **Servidores** responde con total éxito, permitiendo la operatividad del negocio.

---

## 📂 Archivos en el Repositorio
*   `TechNova_Topology.pkt`: Archivo fuente ejecutable de Cisco Packet Tracer listo para ser abierto y simulado.
*   `README.md`: Documentación del proyecto.

---
### ✒️ Autor
*   **Tu Nombre** - [Tu Perfil de LinkedIn](https://www.linkedin.com/in/wilson-fernando-pinzon-guacaneme/)
*   Estudiante de Cisco Networking Academy - Curso: *Networking Essentials*.
