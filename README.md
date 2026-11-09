# proyectoFinalRCD
# Documentación del Proyecto de Redes y Comunicación de Datos

## 1. Introducción y Topología

Este proyecto documenta el diseño e implementación de una red corporativa compuesta por dos sedes principales (Intranet Bogotá e Intranet Madrid) interconectadas a través de una red pública WAN (Nube de ISPs). La arquitectura cuenta con:

* Una red LAN en Bogotá administrada con un Router "Router on a Stick" (R1_BOG) y switches de capa 2.

* Una red LAN en Madrid administrada a través de un Multilayer Switch (MLSW1).

* Un área desmilitarizada (DMZ) centralizada conectada al R1_BOG que aloja servidores críticos (Web, DNS, Tracker).

* Una malla de 4 ISPs completamente en IPv4 que utilizan redistribución de rutas entre OSPF y EIGRP.

* Una VPN IPsec para encapsular tráfico IPv6 sobre la WAN IPv4, garantizando la comunicación segura entre las sucursales.

## 2. Diseño Estructurado

### 2.1 Preguntas de Diseño Estructurado

**¿Cuántas subredes son necesarias?**

* **IPv6 (Total: 12 subredes principales identificadas en la configuración):**

  * **DMZ:** 2 subredes configuradas en `R1_BOG` (Principal: `2001:1200:EC11:10::/64` y Secundaria: `2001:1200:EC11:1::/64`).

  * **Intranet BOG:** 4 subredes SVIs (Guest: `EA11:10::/64`, Internal: `EA11:20::/64`, Server: `EA11:30::/64`, Nativa: `EA11:99::/64`).

  * **Intranet MAD:** 4 subredes SVIs (Guest: `EB21:10::/64`, Internal: `EB21:20::/64`, Server: `EB21:30::/64`, Nativa: `EB21:99::/64`).

  * **Enlace de Tránsito MAD:** 1 subred `R2_ESP ↔ MLSW1` (`2001:1200:EB21:1::/64`).

  * **Túnel VPN (Frontera Lógica):** 1 subred punto a punto lógica (`2001:1200:ED11:1::/64`).

* **IPv4 (Total: 7 subredes /30):**

  * **Fronteras:** `R1_BOG ↔ ISP_BOG` y `ISP_ESP ↔ R2_ESP`.

  * **Malla ISP:** `ISP_BOG ↔ ISP_FL`, `ISP_FL ↔ ISP_ESP`, `ISP_BOG ↔ ISP_NET`, `ISP_NET ↔ ISP_ESP`, `ISP_FL ↔ ISP_NET`.

**Direcciones Privadas y Públicas:**

* **IPv6:** Se utilizan direcciones Global Unicast (GUA) para todas las redes. Las direcciones Link-Local (`fe80::/10`) son generadas para los gateways y vecindades de enrutamiento interno.

* **IPv4:** Todo el bloque ruteable del ISP y enlaces WAN corporativos utilizan direcciones públicas del segmento `86.106.77.0/24`.

## 3. Tablas de Subnetting 

### Subnetting IPv6

| Subred | Nro Hosts | Prefijo de Red | Long. | Primera Dirección | Método de Asignación | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| **DMZ (Main)** | 3 | `2001:1200:EC11:10::` | /64 | `2001:1200:EC11:10::1` | Estática | 
| **Intranet BOG: Guest** | 2 | `2001:1200:EA11:10::` | /64 | `2001:1200:EA11:10::1` | SLAAC / Estática | 
| **Intranet BOG: Internal** | 2 | `2001:1200:EA11:20::` | /64 | `2001:1200:EA11:20::1` | DHCPv6 (Stateful) | 
| **Intranet BOG: Server** | 4 | `2001:1200:EA11:30::` | /64 | `2001:1200:EA11:30::1` | Estática | 
| **Intranet BOG: Nativa** | 4 | `2001:1200:EA11:99::` | /64 | `2001:1200:EA11:99::1` | Estática | 
| **Intranet MAD: Guest** | 2 | `2001:1200:EB21:10::` | /64 | `2001:1200:EB21:10::1` | DHCPv6 / SLAAC | 
| **Intranet MAD: Internal** | 2 | `2001:1200:EB21:20::` | /64 | `2001:1200:EB21:20::1` | DHCPv6 / SLAAC | 
| **Intranet MAD: Server** | 3 | `2001:1200:EB21:30::` | /64 | `2001:1200:EB21:30::1` | Estática | 
| **Intranet MAD: Nativa** | 3 | `2001:1200:EB21:99::` | /64 | `2001:1200:EB21:99::1` | Estática | 
| **Transito MAD (R2 ↔ MLSW1)** | 2 | `2001:1200:EB21:1::` | /64 | `2001:1200:EB21:1::1` | Estática | 
| **Tunnel VPN (R1 ↔ R2)** | 2 | `2001:1200:ED11:1::` | /64 | `2001:1200:ED11:1::1` | Estática | 

### Subnetting IPv4 

| Descripción de Enlace | Dirección de Red / CIDR | Rango de Hosts (Usables) | Broadcast | 
| ----- | ----- | ----- | ----- | 
| **R1_BOG ↔ ISP_BOG** | `86.106.77.0/30` | `.1` - `.2` | `.3` | 
| **ISP_BOG ↔ ISP_FL** | `86.106.77.4/30` | `.5` - `.6` | `.7` | 
| **ISP_FL ↔ ISP_ESP** | `86.106.77.8/30` | `.9` - `.10` | `.11` | 
| **ISP_BOG ↔ ISP_NET** | `86.106.77.12/30` | `.13` - `.14` | `.15` | 
| **ISP_NET ↔ ISP_ESP** | `86.106.77.16/30` | `.17` - `.18` | `.19` | 
| **ISP_FL ↔ ISP_NET** | `86.106.77.20/30` | `.21` - `.22` | `.23` | 
| **ISP_ESP ↔ R2_ESP** | `86.106.77.24/30` | `.25` - `.26` | `.27` | 

## 4. Implementación de Servicios y Enrutamiento

### 4.1 Enrutamiento en Nube ISP (IPv4)

La nube de proveedores de servicio utiliza un enrutamiento híbrido IPv4 con redistribución mutua para garantizar el ruteo completo de las IPs públicas:

* **OSPFv2 (Process ID 1):** Configurado en `ISP_NET`, con enlaces hacia ISP_BOG e ISP_ESP.

* **EIGRP (AS 1):** Configurado en `ISP_FL` y partes de `ISP_BOG` e `ISP_ESP`.

* **Redistribución:** Implementada en los Routers de Borde (`ISP_BOG` y `ISP_ESP`) inyectando las métricas requeridas de EIGRP a OSPF y viceversa.

* **Frontera Empresa:** Se emplea enrutamiento por defecto (`ip route 0.0.0.0 0.0.0.0`) desde `R1_BOG` apuntando a `ISP_BOG`, y desde `R2_ESP` apuntando a `ISP_ESP`.

### 4.2 Enrutamiento Corporativo (IPv6)

* **Intranet BOG y DMZ:** El router `R1_BOG` ejecuta `OSPFv3 (Process 1)` y se declara directamente en las interfaces lógicas (VLANs) y en la Fa0/0 (DMZ).

* **Intranet MAD:** El Multilayer Switch enruta internamente (SVI gateways). `MLSW1` apunta su ruta por defecto a `R2_ESP` (`ipv6 route ::/0 2001:1200:EB21:1::1`). A su vez, `R2_ESP` cuenta con rutas estáticas apuntando el tráfico corporativo y DMZ hacia el Túnel VPN (`2001:1200:ED11:1::1`).

### 4.3 Implementación de VPN IPsec e IPv6-in-IPv4

Para interconectar las sedes IPv6 de forma segura a través de Internet (IPv4), se configuró una VPN de sitio a sitio (Site-to-Site IPsec) sobre interfaces de tipo Túnel.

1. **Fase 1 (ISAKMP):** Políticas con encriptación AES 256, Hash seguro y autenticación con llave pre-compartida (`ClaveSecretaRCD2026`).

2. **Fase 2 (IPsec):** Transform-set (`TS-RCD`) aplicando `esp-aes 256` y `esp-sha-hmac`.

3. **Crypto Map (MAP-VPN):** Vincula las políticas IPSec y se aplica en las interfaces WAN IPv4 (`Serial0/0/0` en R1_BOG y R2_ESP).

4. **Túnel Lógico (Tunnel 0):** Configurado con `tunnel mode ipv6ip`. Este túnel permite que los paquetes IPv6 originados en las sucursales sean empaquetados dentro de las cabeceras IPv4 que cruzan la Nube del ISP. Luego, el IPsec encripta todo este payload (Tunnel payload), cumpliendo la migración IPv6/IPv4 y añadiendo total confidencialidad.

## 5. Implementación de Políticas de Seguridad (ACLs)

Las listas de control de acceso se aplican de manera `inbound` en los gateways (SVI y Subinterfaces). Según los requerimientos:

* **Intranet Bogotá (R1_BOG):**

  * `ACL_BOG_GUEST`: Deniega HTTPS (443) hacia el Servidor Web (IP `.10`), permite HTTP (80/www).

  * `ACL_BOG_INTERNAL`: Deniega HTTP (80/www) hacia el Servidor Web, permite HTTPS (443).

* **Intranet Madrid (MLSW1):**

  * `ACL_MAD_GUEST`: Bloquea HTTPS al Web Server DMZ, pero permite HTTP.

  * `ACL_MAD_INTERNAL`: Bloquea HTTP al Web Server DMZ, pero permite HTTPS de forma estricta para la corporación.

## 6. Comunicación de Datos (App de Seguimiento)

El sistema **Employee Tracker** implementa el seguimiento de ubicación mediante un modelo basado en sockets TCP y telemetría simulada (Beacon):

1. **Captura en Edge (BOG - PC4):** Un host en la intranet BOG (`PC4`, VLAN Internal) ejecuta la **Tracker App**. Este host simulado actúa como cliente detectando dispositivos Bluetooth (Beacons), empaquetando métricas como la coordenada `X,Y` y el estado físico del empleado.

2. **Transmisión a DMZ:** A través del enrutamiento OSPFv3, la Tracker App envía un stream persistente a través de la LAN y el Router `R1_BOG` hacia el `Tracker Server` alojado de forma segura en la DMZ.

3. **Relay del Tracker Server:** El Servidor procesa la señal en la aplicación **Tracker Relay**. Su función es desdoblar la telemetría y transmitirla hacia la intranet de monitoreo, comportándose como un Middleware de puente seguro.

4. **Visualización en MAD (Tracker Dashboard - PC7):** La información atraviesa el router `R1_BOG`, se encripta e inyecta en el túnel IPsec, pasa por el ISP en formato IPv4, se desencripta en `R2_ESP` y se entrega a `PC7` en la red MAD. El Dashboard presenta variables en tiempo real disparando alarmas de tiempo/posición si corresponden.

Esto comprueba el funcionamiento end-to-end de la topología: Conectividad local LAN, políticas de firewall restrictivas, enrutamiento a través de un túnel superpuesto IPsec e integración directa con capa de aplicación.
