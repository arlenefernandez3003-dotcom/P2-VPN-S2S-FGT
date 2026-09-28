# FortiGate — VPN Site-to-Site entre dos FortiGates

### Arlene Fernández Herrera · Matrícula: 2025-0730

**Seguridad de Redes · ITLA**

---

## 🎥 Video Demostrativo

**[▶ Ver video de demostración](https://youtu.be/REEMPLAZAR-CON-TU-ID)**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Topología y Direccionamiento](#2-topología-y-direccionamiento)
   - [Diagrama de Topología](#21-diagrama-de-topología)
   - [Tabla de Interfaces](#22-tabla-de-interfaces)
   - [Tabla de Dispositivos](#23-tabla-de-dispositivos)
3. [Configuración del Router ISP (Cisco)](#3-configuración-del-router-isp-cisco)
4. [Configuraciones del FortiGate-A (Sitio Usuarios) por la GUI](#4-configuraciones-del-fortigate-a-sitio-usuarios-por-la-gui)
   - [4.0 Acceso Inicial — CLI](#40-acceso-inicial--cli)
   - [4.1 Configuración de Interfaces](#41-configuración-de-interfaces)
   - [4.2 DHCP en VLAN10 (Usuarios)](#42-dhcp-en-vlan10-usuarios)
   - [4.3 Ruta por Defecto](#43-ruta-por-defecto)
5. [Configuraciones del FortiGate-B (Sitio Servidor) por la GUI](#5-configuraciones-del-fortigate-b-sitio-servidor-por-la-gui)
   - [5.0 Acceso Inicial — CLI](#50-acceso-inicial--cli)
   - [5.1 Configuración de Interfaces](#51-configuración-de-interfaces)
   - [5.2 Ruta por Defecto](#52-ruta-por-defecto)
6. [VPN Site-to-Site (IPsec)](#6-vpn-site-to-site-ipsec)
   - [6.1 Fase 1 — FortiGate-A](#61-fase-1--fortigate-a)
   - [6.2 Fase 2 — FortiGate-A](#62-fase-2--fortigate-a)
   - [6.3 Fase 1 — FortiGate-B](#63-fase-1--fortigate-b)
   - [6.4 Fase 2 — FortiGate-B](#64-fase-2--fortigate-b)
   - [6.5 Rutas estáticas hacia el túnel](#65-rutas-estáticas-hacia-el-túnel)
   - [6.6 Políticas de Firewall para el tráfico VPN](#66-políticas-de-firewall-para-el-tráfico-vpn)
7. [Web Server (HTTPS)](#7-web-server-https)
8. [Pruebas de Verificación](#8-pruebas-de-verificación)
9. [Capturas de Pantalla](#9-capturas-de-pantalla)
10. [Estructura del Repositorio](#10-estructura-del-repositorio)

---

## 1. Objetivo del Laboratorio

Esta práctica implementa un **túnel VPN Site-to-Site (IPsec)** entre dos FortiGates que representan dos sitios distintos, conectados a través de un **router Cisco que simula al ISP** con IPs públicas. El objetivo es que el **Usuario** (VLAN 10, detrás de FortiGate-A) pueda comunicarse con el **Web Server** (detrás de FortiGate-B) **únicamente a través del enlace VPN** — no existe ninguna política que permita ese tráfico por la ruta pública/NAT normal, así que si el túnel cae, la comunicación se cae con él. Esto se demuestra con `traceroute` desde el Usuario hacia el servidor, tanto con el túnel activo como caído.

Toda la configuración y demostración de ambos FortiGates se realiza **por GUI**.

---

## 2. Topología y Direccionamiento

> Direccionamiento derivado de la matrícula **2025-0730** → base `20.25.30.0/24` para las LAN internas, y `203.0.113.0/24` (rango reservado para documentación, RFC 5737) repartido en dos enlaces punto a punto `/29` para el ISP. Como en los labs anteriores, **ninguna interfaz usa la primera IP utilizable de su subred**.

### 2.1 Diagrama de Topología

```
                              ┌─────────────────┐
                              │   Router ISP    │
                              │                 │
                              │  e0/0: .2 /29   │
                              │  e0/1: .10 /29  │
                              └───┬──────────┬──┘
                    203.0.113.0/29│          │203.0.113.8/29
                                  │          │
                     ┌────────────┘          └───────────────┐
                     │                                       │
              ┌──────┴───────┐                       ┌───────┴──────┐
              │  FortiGate-A │                       │  FortiGate-B │
              │ port1 (WAN)  │                       │ port1 (WAN)  │
              │ 203.0.113.3  │                       │ 203.0.113.11 │
              │ port2 (LAN)  │                       │ port2 (LAN)  │
              └──────┬───────┘                       └───────┬──────┘
                     │ 20.25.30.0/25         20.25.30.128/28 │
              ┌──────┴───────┐                       ┌───────┴──────┐
              │   Usuario    │                       │  Web Server  │
              │   (DHCP)     │                       │  (Estática)  │
              │   VLAN 10    │                       │    HTTPS     │
              └──────────────┘                       └──────────────┘

              ┄┄┄┄┄┄┄┄┄┄┄┄┄ VPN Site-to-Site (IPsec) ┄┄┄┄┄┄┄┄┄┄┄┄┄
                     túnel entre 203.0.113.3 ↔ 203.0.113.11
                  (el tráfico VPN transita a través del router ISP,
                   que solo enruta entre sus dos redes directamente
                   conectadas — no requiere salida real a Internet)

  Política de comunicación:
  ┌───────────────────────────────────────────────────────────────────┐
  │ Usuario → Web Server : SOLO a través del túnel VPN (IPsec)        │
  │ No existe ruta pública/NAT que permita ese tráfico fuera del túnel│
  │ Si el túnel cae, el traceroute deja de completar hacia el servidor│
  └───────────────────────────────────────────────────────────────────┘
```

> El router ISP no necesita NAT ni ruta por defecto hacia un Internet real — su único trabajo es enrutar entre las dos redes `/29` a las que está directamente conectado, exactamente lo que necesita la VPN para establecerse entre las dos IPs públicas.

### 2.2 Tabla de Interfaces

**Router ISP (Cisco):**

| Interfaz | Rol | Dirección IP | Máscara |
|---|---|---|---|
| **e0/0** | Hacia FortiGate-A | 203.0.113.2 | /29 |
| **e0/1** | Hacia FortiGate-B | 203.0.113.10 | /29 |

**FortiGate-A (Sitio Usuarios):**

| Interfaz | Alias | Rol | Dirección IP | Máscara |
|---|---|---|---|---|
| **port1** | WAN-ISP | WAN | 203.0.113.3 | /29 |
| **port2** | LAN-USUARIOS | LAN | 20.25.30.2 | /25 |

**FortiGate-B (Sitio Servidor):**

| Interfaz | Alias | Rol | Dirección IP | Máscara |
|---|---|---|---|---|
| **port1** | WAN-ISP | WAN | 203.0.113.11 | /29 |
| **port2** | LAN-SERVIDOR | LAN | 20.25.30.130 | /28 |

### 2.3 Tabla de Dispositivos

| Dispositivo | Interfaz | Dirección IP | Máscara | Gateway | Método | Rol |
|---|---|---|---|---|---|---|
| **Router ISP** | e0/0 | 203.0.113.2 | /29 | — | Estática | Enlace hacia FortiGate-A |
| **Router ISP** | e0/1 | 203.0.113.10 | /29 | — | Estática | Enlace hacia FortiGate-B |
| **FortiGate-A** | port1 | 203.0.113.3 | /29 | 203.0.113.2 | Estática | WAN, extremo local de la VPN |
| **FortiGate-A** | port2 | 20.25.30.2 | /25 | — | Estática | Gateway VLAN 10 (Usuarios) |
| **FortiGate-B** | port1 | 203.0.113.11 | /29 | 203.0.113.10 | Estática | WAN, extremo remoto de la VPN |
| **FortiGate-B** | port2 | 20.25.30.130 | /28 | — | Estática | Gateway LAN Servidor |
| **Usuario** | eth0 | 20.25.30.3 (rango) | /25 | 20.25.30.2 | **DHCP** | Cliente en VLAN 10 |
| **Web Server** | eth0 | 20.25.30.131 | /28 | 20.25.30.130 | **Estática** | Servidor HTTPS |

> El rango DHCP de VLAN 10 es `20.25.30.3 – 20.25.30.126`. Cada enlace ISP↔FortiGate es un `/29` (6 IPs utilizables: `.1–.6` y `.9–.14`) — se usa la tercera IP de cada uno para el FortiGate y la segunda para el router, dejando la primera libre en ambos casos.

---

## 3. Configuración del Router ISP (Cisco)

Simula el proveedor de Internet: solo enruta entre las dos redes `/29` a las que está directamente conectado. No necesita NAT, ACLs de salida a Internet, ni protocolo de enrutamiento dinámico — con las interfaces `up/up` y sus IPs, el router ya tiene ambas rutas en su tabla como "directly connected", suficiente para que las dos FortiGate se vean entre sí.

```bash
enable
configure terminal

hostname ISP

interface Ethernet0/0
 description Enlace hacia FortiGate-A
 ip address 203.0.113.2 255.255.255.248
 no shutdown
exit

interface Ethernet0/1
 description Enlace hacia FortiGate-B
 ip address 203.0.113.10 255.255.255.248
 no shutdown
exit

end
write memory
```

**Verificación:**
```bash
show ip interface brief
show ip route
```
Debe mostrar ambas interfaces `up/up` y las dos redes `203.0.113.0/29` y `203.0.113.8/29` como `C` (directly connected) en la tabla de rutas.

> Ver evidencia: [00_isp_router_config.png](screenshots/00_isp_router_config.png)

---

## 4. Configuraciones del FortiGate-A (Sitio Usuarios) por la GUI

### 4.0 Acceso Inicial — CLI

```bash
config system interface
    edit "port1"
        set mode static
        set ip 203.0.113.3 255.255.255.248
        set allowaccess https ssh ping
        set role wan
    next
end
```

Acceder luego desde el navegador a `https://203.0.113.3` (o por la consola/mgmt del hipervisor) con las credenciales por defecto (`admin` / contraseña vacía) y definir una contraseña segura.

> Ver evidencia: [01_cli_acceso_fga.png](screenshots/01_cli_acceso_fga.png)

### 4.1 Configuración de Interfaces

**Ruta:** `Network → Interfaces`

**port1 — WAN-ISP:**

| Campo | Valor |
|---|---|
| Role | `WAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `203.0.113.3 / 255.255.255.248` |
| Administrative access | `HTTPS, SSH, Ping` |

**port2 — LAN-USUARIOS:**

| Campo | Valor |
|---|---|
| Role | `LAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `20.25.30.2 / 255.255.255.128` |
| Administrative access | `Ping` |

> Ver evidencia: [02_interfaces_fga.png](screenshots/02_interfaces_fga.png)

### 4.2 DHCP en VLAN10 (Usuarios)

**Ruta:** `Network → Interfaces → port2 → Edit → DHCP Server → Create New`

| Campo | Valor |
|---|---|
| Status | `Enable` |
| Address Range | `20.25.30.3 – 20.25.30.126` |
| Netmask | `255.255.255.128` |
| Default Gateway | `20.25.30.2` |
| DNS Server | `8.8.8.8` / `8.8.4.4` |
| Lease Time | `1 day` |

> Ver evidencia: [03_dhcp_fga.png](screenshots/03_dhcp_fga.png)

### 4.3 Ruta por Defecto

**Ruta:** `Network → Static Routes → Create New`

| Campo | Valor |
|---|---|
| Destination | `0.0.0.0 / 0.0.0.0` |
| Gateway | `203.0.113.2` (Router ISP, interfaz Gi0/0) |
| Interface | `port1` |
| Distance | `10` |

> Esta ruta apunta al router ISP como salto siguiente — necesaria para que FortiGate-A sepa cómo llegar a `203.0.113.11` (FortiGate-B) y así poder levantar la Fase 1 de la VPN. No implica salida real a Internet: el router ISP solo conoce sus dos redes `/29` directamente conectadas.

> Ver evidencia: [04_ruta_default_fga.png](screenshots/04_ruta_default_fga.png)

---

## 5. Configuraciones del FortiGate-B (Sitio Servidor) por la GUI

### 5.0 Acceso Inicial — CLI

```bash
config system interface
    edit "port1"
        set mode static
        set ip 203.0.113.11 255.255.255.248
        set allowaccess https ssh ping
        set role wan
    next
end
```

### 5.1 Configuración de Interfaces

**port1 — WAN-ISP:**

| Campo | Valor |
|---|---|
| Role | `WAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `203.0.113.11 / 255.255.255.248` |
| Administrative access | `HTTPS, SSH, Ping` |

**port2 — LAN-SERVIDOR:**

| Campo | Valor |
|---|---|
| Role | `LAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `20.25.30.130 / 255.255.255.240` |
| Administrative access | `Ping` |

> Ver evidencia: [05_interfaces_fgb.png](screenshots/05_interfaces_fgb.png)

### 5.2 Ruta por Defecto

**Ruta:** `Network → Static Routes → Create New`

| Campo | Valor |
|---|---|
| Destination | `0.0.0.0 / 0.0.0.0` |
| Gateway | `203.0.113.10` (Router ISP, interfaz Gi0/1) |
| Interface | `port1` |
| Distance | `10` |

> Igual que en FortiGate-A: es lo que le permite a FortiGate-B alcanzar `203.0.113.3` para levantar la VPN — no implica salida a Internet real.

> Ver evidencia: [06_ruta_default_fgb.png](screenshots/06_ruta_default_fgb.png)

---

## 6. VPN Site-to-Site (IPsec)

Se configura un túnel **IPsec Site-to-Site** entre `203.0.113.3` (FortiGate-A) y `203.0.113.11` (FortiGate-B), a través del router ISP, usando el asistente de FortiGate (`VPN → IPsec Wizard`) en modo **Site to Site**, que crea automáticamente la interfaz de túnel, Fase 1, Fase 2 y las rutas asociadas.

### 6.1 Fase 1 — FortiGate-A

**Ruta:** `VPN → IPsec Wizard → Create New`

| Campo | Valor |
|---|---|
| Name | `VPN-A-to-B` |
| Template | `Site to Site` |
| Remote Device Type | `FortiGate` |
| Remote IP Address | `203.0.113.11` |
| Outgoing Interface | `port1` |
| Authentication Method | `Pre-shared Key` |
| Pre-shared Key | *(clave fuerte, la misma en ambos extremos)* |
| IKE Version | `2` |

### 6.2 Fase 2 — FortiGate-A

| Campo | Valor |
|---|---|
| Local Address | `20.25.30.0/25` (red de Usuarios) |
| Remote Address | `20.25.30.128/28` (red del Servidor) |

> El wizard crea automáticamente la interfaz virtual `VPN-A-to-B` (tipo tunnel) y una ruta estática hacia `20.25.30.128/28` a través de ella — verificar en `Network → Static Routes` que quedó creada.

> Ver evidencia: [07_ipsec_fase1_fga.png](screenshots/07_ipsec_fase1_fga.png), [08_ipsec_fase2_fga.png](screenshots/08_ipsec_fase2_fga.png)

### 6.3 Fase 1 — FortiGate-B

Configuración espejo, apuntando de vuelta hacia FortiGate-A:

| Campo | Valor |
|---|---|
| Name | `VPN-B-to-A` |
| Template | `Site to Site` |
| Remote Device Type | `FortiGate` |
| Remote IP Address | `203.0.113.3` |
| Outgoing Interface | `port1` |
| Authentication Method | `Pre-shared Key` |
| Pre-shared Key | *(la misma clave configurada en FortiGate-A)* |
| IKE Version | `2` |

### 6.4 Fase 2 — FortiGate-B

| Campo | Valor |
|---|---|
| Local Address | `20.25.30.128/28` (red del Servidor) |
| Remote Address | `20.25.30.0/25` (red de Usuarios) |

> Ver evidencia: [09_ipsec_fase1_fgb.png](screenshots/09_ipsec_fase1_fgb.png), [10_ipsec_fase2_fgb.png](screenshots/10_ipsec_fase2_fgb.png)

### 6.5 Rutas estáticas hacia el túnel

Si el wizard no las crea automáticamente, agregarlas manualmente:

**En FortiGate-A:** `Network → Static Routes → Create New`

| Campo | Valor |
|---|---|
| Destination | `20.25.30.128/28` |
| Interface | `VPN-A-to-B` |

**En FortiGate-B:** `Network → Static Routes → Create New`

| Campo | Valor |
|---|---|
| Destination | `20.25.30.0/25` |
| Interface | `VPN-B-to-A` |

### 6.6 Políticas de Firewall para el tráfico VPN

El wizard suele crear estas políticas automáticamente; verificar/ajustar en `Policy & Objects → Firewall Policy`.

**En FortiGate-A:**

| Campo | Valor |
|---|---|
| Name | `Usuarios-to-VPN` |
| Incoming Interface | `port2 (LAN-USUARIOS)` |
| Outgoing Interface | `VPN-A-to-B` |
| Source | `all` |
| Destination | `Web Server (20.25.30.131)` |
| Service | `ALL` |
| Action | `ACCEPT` |
| NAT | ❌ Disabled *(el tráfico VPN site-to-site no se NATea, así ambos extremos ven la IP real del otro)* |

| Campo | Valor |
|---|---|
| Name | `VPN-to-Usuarios` |
| Incoming Interface | `VPN-A-to-B` |
| Outgoing Interface | `port2 (LAN-USUARIOS)` |
| Source | `Web Server (20.25.30.131)` |
| Destination | `all` |
| Service | `ALL` |
| Action | `ACCEPT` |
| NAT | ❌ Disabled |

**En FortiGate-B:** políticas espejo, `VPN-B-to-A ↔ port2 (LAN-SERVIDOR)`, mismos criterios de origen/destino invertidos.

> **Importante:** no se crea ninguna política que permita el tráfico Usuario→Servidor por `port1` (fuera del túnel) — así se garantiza que la única forma de llegar al servidor es a través de la VPN, cumpliendo el objetivo del lab.

> Ver evidencia: [11_politicas_vpn_fga.png](screenshots/11_politicas_vpn_fga.png), [12_politicas_vpn_fgb.png](screenshots/12_politicas_vpn_fgb.png)

---

## 7. Web Server (HTTPS)

Servidor Ubuntu con Apache + certificado autofirmado, igual que en el lab anterior:

```bash
sudo apt update && sudo apt install -y apache2 openssl
sudo openssl req -x509 -nodes -days 825 -newkey rsa:2048 \
  -keyout /etc/ssl/private/webserver.key \
  -out /etc/ssl/certs/webserver.crt \
  -subj "/C=DO/ST=SantoDomingo/L=SantoDomingo/O=ITLA/CN=20.25.30.131" \
  -addext "basicConstraints=critical,CA:FALSE" \
  -addext "keyUsage=critical,digitalSignature,keyEncipherment" \
  -addext "subjectAltName=IP:20.25.30.131"
sudo a2enmod ssl
# apuntar SSLCertificateFile / SSLCertificateKeyFile a los archivos generados en default-ssl.conf
sudo a2ensite default-ssl
sudo systemctl restart apache2
```

Direccionamiento estático: `20.25.30.131/28`, gateway `20.25.30.130` (FortiGate-B, `port2`).

---

## 8. Pruebas de Verificación

**8.1 — Con el túnel activo:**

Desde el Usuario (VLAN 10):
```
traceroute 20.25.30.131
```
Debe completar en pocos saltos, mostrando el tráfico atravesando la interfaz `VPN-A-to-B`.

```
curl -k https://20.25.30.131/
```
Debe responder con el contenido del Web Server.

**8.2 — Con el túnel caído (para demostrar que NO hay ruta alterna):**

En cualquiera de los dos FortiGates: `VPN → IPsec Tunnels → VPN-A-to-B → Bring Down` (o deshabilitar temporalmente la Fase 1).

Repetir el `traceroute` y el `curl` desde el Usuario — ambos deben **fallar/quedar colgados**, confirmando que no existe ninguna ruta pública que permita llegar al servidor sin el túnel.

Volver a levantar el túnel (`Bring Up`) y repetir la prueba para mostrar que se recupera.

> Ver evidencia: [13_traceroute_tunel_activo.png](screenshots/13_traceroute_tunel_activo.png), [14_traceroute_tunel_caido.png](screenshots/14_traceroute_tunel_caido.png), [15_ipsec_monitor.png](screenshots/15_ipsec_monitor.png)

---

## 9. Capturas de Pantalla

| # | Archivo | Descripción |
|---|---|---|
| 00 | [`00_isp_router_config.png`](screenshots/00_isp_router_config.png) | Terminal CLI del router ISP mostrando `show ip interface brief` y `show ip route` con ambas interfaces `up/up` y las dos redes `/29` directamente conectadas. |
| 01 | [`01_cli_acceso_fga.png`](screenshots/01_cli_acceso_fga.png) | Terminal CLI de FortiGate-A mostrando la config inicial de `port1` (203.0.113.3/29) y el login de la GUI. |
| 02 | [`02_interfaces_fga.png`](screenshots/02_interfaces_fga.png) | `Network → Interfaces` de FortiGate-A: port1 WAN y port2 LAN-USUARIOS configuradas. |
| 03 | [`03_dhcp_fga.png`](screenshots/03_dhcp_fga.png) | Servidor DHCP en port2 de FortiGate-A, rango `20.25.30.3–126`. |
| 04 | [`04_ruta_default_fga.png`](screenshots/04_ruta_default_fga.png) | Ruta estática por defecto en FortiGate-A apuntando al router ISP (`203.0.113.2`). |
| 05 | [`05_interfaces_fgb.png`](screenshots/05_interfaces_fgb.png) | `Network → Interfaces` de FortiGate-B: port1 WAN y port2 LAN-SERVIDOR configuradas. |
| 06 | [`06_ruta_default_fgb.png`](screenshots/06_ruta_default_fgb.png) | Ruta estática por defecto en FortiGate-B apuntando al router ISP (`203.0.113.10`). |
| 07 | [`07_ipsec_fase1_fga.png`](screenshots/07_ipsec_fase1_fga.png) | Fase 1 de la VPN en FortiGate-A, remote gateway `203.0.113.11`. |
| 08 | [`08_ipsec_fase2_fga.png`](screenshots/08_ipsec_fase2_fga.png) | Fase 2 de la VPN en FortiGate-A, subredes local/remota. |
| 09 | [`09_ipsec_fase1_fgb.png`](screenshots/09_ipsec_fase1_fgb.png) | Fase 1 de la VPN en FortiGate-B, remote gateway `203.0.113.3`. |
| 10 | [`10_ipsec_fase2_fgb.png`](screenshots/10_ipsec_fase2_fgb.png) | Fase 2 de la VPN en FortiGate-B, subredes local/remota. |
| 11 | [`11_politicas_vpn_fga.png`](screenshots/11_politicas_vpn_fga.png) | Políticas de firewall en FortiGate-A para el tráfico hacia/desde la VPN. |
| 12 | [`12_politicas_vpn_fgb.png`](screenshots/12_politicas_vpn_fgb.png) | Políticas de firewall en FortiGate-B para el tráfico hacia/desde la VPN. |
| 13 | [`13_traceroute_tunel_activo.png`](screenshots/13_traceroute_tunel_activo.png) | Traceroute exitoso desde el Usuario al Web Server con el túnel activo. |
| 14 | [`14_traceroute_tunel_caido.png`](screenshots/14_traceroute_tunel_caido.png) | Traceroute fallido desde el Usuario al Web Server con el túnel caído — confirma que no hay ruta alterna. |
| 15 | [`15_ipsec_monitor.png`](screenshots/15_ipsec_monitor.png) | `Monitor → IPsec Monitor` mostrando el túnel `Up` y luego `Down` durante la prueba. |

---

## 10. Estructura del Repositorio

```
/
├── README.md                  ← este documento
├── screenshots/                ← capturas numeradas de cada configuración
├── running-configs/
│   ├── isp-router-running-config.txt
│   ├── fortigate-a-running-config.conf
│   └── fortigate-b-running-config.conf
└── entregable/
    └── ArleneFernandez_20250730_P3.txt
```

> Ajustar el número de práctica (`P3`) según lo indicado por el profesor. El video debe subirse al principio del repositorio (enlace ya colocado arriba en este README).

---
