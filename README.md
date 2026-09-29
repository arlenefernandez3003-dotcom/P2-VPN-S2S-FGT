# FortiGate — VPN Site-to-Site entre dos FortiGates

### Arlene Fernández Herrera · Matrícula: 2025-0730

**Seguridad de Redes · ITLA**

---

## 🎥 Video Demostrativo

**[▶ Ver video de demostración](https://youtu.be/REEMPLAZAR-CON-TU-ID)**

---

> ⚠️ **Aclaración importante:** FortiGate-A y FortiGate-B **no tienen la misma versión de FortiOS** (FortiGate-A: `v7.0.3` · FortiGate-B: `v7.6.2` *← completar*). Por eso, al configurar uno y otro pueden aparecer **pequeñas variaciones en la GUI** (nombres de menús o campos, orden de las opciones, pasos del asistente). Los valores de configuración son los mismos en ambos equipos.

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Topología y Direccionamiento](#2-topología-y-direccionamiento)
   - [Diagrama de Topología](#21-diagrama-de-topología)
   - [Tabla de Interfaces](#22-tabla-de-interfaces)
   - [Tabla de Dispositivos](#23-tabla-de-dispositivos)
3. [Nube PNET y Switch de Usuarios](#3-nube-pnet-y-switch-de-usuarios)
   - [3.1 Configuración de la Nube PNET](#31-configuración-de-la-nube-pnet)
   - [3.2 Switch de Usuarios (VLAN 10)](#32-switch-de-usuarios-vlan-10)
4. [Configuraciones del FortiGate-A (Sitio Usuarios) por la GUI](#4-configuraciones-del-fortigate-a-sitio-usuarios-por-la-gui)
   - [4.0 Acceso Inicial — CLI](#40-acceso-inicial--cli)
   - [4.1 Configuración de Interfaces](#41-configuración-de-interfaces)
   - [4.2 Interfaz VLAN 10](#42-interfaz-vlan-10)
   - [4.3 DHCP en VLAN 10 (Usuarios)](#43-dhcp-en-vlan-10-usuarios)
   - [4.4 Política de NAT hacia la WAN](#44-política-de-nat-hacia-la-wan)
5. [Configuraciones del FortiGate-B (Sitio Servidor) por la GUI](#5-configuraciones-del-fortigate-b-sitio-servidor-por-la-gui)
   - [5.0 Acceso Inicial — CLI](#50-acceso-inicial--cli)
   - [5.1 Configuración de Interfaces](#51-configuración-de-interfaces)
   - [5.2 Política de NAT hacia la WAN](#52-política-de-nat-hacia-la-wan)
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

Esta práctica implementa un **túnel VPN Site-to-Site (IPsec)** entre dos FortiGates que representan dos sitios distintos, conectados entre sí y con la PC local a través de una **Nube de PNETLab** en la red `203.0.113.0/29` que simula al ISP con IPs públicas. El objetivo es que el **Usuario** (VLAN 10, `/25`, detrás de FortiGate-A y un switch) pueda comunicarse con el **Web Server** (`/28`, detrás de FortiGate-B) **únicamente a través del enlace VPN** — no existe ninguna ruta ni política que permita ese tráfico fuera del túnel, así que si el túnel cae, la comunicación se cae con él. Esto se demuestra con `traceroute` desde el Usuario hacia el servidor, tanto con el túnel activo como caído.

Ambos FortiGates además tienen **NAT** hacia la WAN (política con NAT habilitado en `port1`), que se comprueba mostrando que el tráfico saliente hacia la nube lleva la IP pública del FortiGate.

Toda la configuración y demostración de ambos FortiGates se realiza **por GUI**, accedida desde el navegador de la PC local. El laboratorio no requiere salida a Internet.

---

## 2. Topología y Direccionamiento

> Direccionamiento de las LAN internas derivado de la matrícula **2025-0730** → base `20.25.30.0/24`. La red de la nube es `203.0.113.0/29` (rango reservado para documentación, RFC 5737): la `.1` es para la PC local y las demás para los FortiGates.

### 2.1 Diagrama de Topología

```
                      ┌───────────────────────────┐
   PC local ──────────┤   Nube PNET (ISP / Cloud) │
   203.0.113.1        │       203.0.113.0/29      │
                      └─────┬───────────────┬─────┘
                            │               │
                     ┌──────┴───────┐ ┌─────┴────────┐
                     │  FortiGate-A │ │  FortiGate-B │
                     │ port1 (WAN)  │ │ port1 (WAN)  │
                     │ 203.0.113.2  │ │ 203.0.113.3  │
                     │ port2 (trunk)│ │ port2 (LAN)  │
                     │ └ VLAN10     │ │ 20.25.30.130 │
                     │  20.25.30.2  │ │              │
                     └──────┬───────┘ └─────┬────────┘
                            │ trunk VLAN 10 │ 20.25.30.128/28
                     ┌──────┴───────┐ ┌─────┴────────┐
                     │ SW-USUARIOS  │ │  Web Server  │
                     │ e0/0 trunk   │ │  (Estática)  │
                     │ e0/1 acc.V10 │ │    HTTPS     │
                     └──────┬───────┘ └──────────────┘
                            │ VLAN 10 · 20.25.30.0/25
                     ┌──────┴───────┐
                     │   Usuario    │
                     │   (DHCP)     │
                     └──────────────┘

              ┄┄┄┄┄┄┄┄┄┄┄┄┄ VPN Site-to-Site (IPsec) ┄┄┄┄┄┄┄┄┄┄┄┄┄
                    túnel entre 203.0.113.2 ↔ 203.0.113.3

  Política de comunicación:
  ┌───────────────────────────────────────────────────────────────────┐
  │ Usuario → Web Server : SOLO a través del túnel VPN (IPsec)        │
  │ No existe ruta ni política que permita ese tráfico fuera del túnel│
  │ Si el túnel cae, el traceroute deja de completar hacia el servidor│
  │ NAT: el tráfico hacia la nube (203.0.113.0/29) sale con la IP de  │
  │      port1 de cada FortiGate                                      │
  └───────────────────────────────────────────────────────────────────┘
```

### 2.2 Tabla de Interfaces

**Nube PNET:**

| Elemento | Rol | Dirección IP | Máscara |
|---|---|---|---|
| **Red de la nube** | Segmento compartido PC + FortiGates (ISP) | 203.0.113.0 | /29 |
| **PC local** | Acceso a la GUI de ambos FortiGates | 203.0.113.1 | /29 |

**SW-USUARIOS (switch L2):**

| Interfaz | Modo | VLAN | Conectado a |
|---|---|---|---|
| **e0/0** | Trunk (802.1Q) | 10 permitida | FortiGate-A `port2` |
| **e0/1** | Access | 10 | Usuario |

**FortiGate-A (Sitio Usuarios):**

| Interfaz | Alias | Rol | Dirección IP | Máscara |
|---|---|---|---|---|
| **port1** | WAN-NUBE | WAN | 203.0.113.2 | /29 |
| **port2** | TRUNK-SW | (físico, sin IP) | — | — |
| **VLAN10** (sobre port2, ID 10) | LAN-USUARIOS | LAN | 20.25.30.2 | /25 |

**FortiGate-B (Sitio Servidor):**

| Interfaz | Alias | Rol | Dirección IP | Máscara |
|---|---|---|---|---|
| **port1** | WAN-NUBE | WAN | 203.0.113.3 | /29 |
| **port2** | LAN-SERVIDOR | LAN | 20.25.30.130 | /28 |

### 2.3 Tabla de Dispositivos

| Dispositivo | Interfaz | Dirección IP | Máscara | Gateway | Método | Rol |
|---|---|---|---|---|---|---|
| **PC local** | Adaptador VMnet | 203.0.113.1 | /29 | — | Estática | Acceso a la GUI de los FortiGates |
| **FortiGate-A** | port1 | 203.0.113.2 | /29 | — | Estática | WAN, extremo local de la VPN |
| **FortiGate-A** | VLAN10 (port2) | 20.25.30.2 | /25 | — | Estática | Gateway VLAN 10 (Usuarios) |
| **FortiGate-B** | port1 | 203.0.113.3 | /29 | — | Estática | WAN, extremo remoto de la VPN |
| **FortiGate-B** | port2 | 20.25.30.130 | /28 | — | Estática | Gateway LAN Servidor |
| **SW-USUARIOS** | e0/0 · e0/1 | — | — | — | — | Switch L2: trunk hacia FGT-A, access VLAN 10 al Usuario |
| **Usuario** | eth0 | 20.25.30.3 (rango) | /25 | 20.25.30.2 | **DHCP** | Cliente en VLAN 10 |
| **Web Server** | eth0 | 20.25.30.131 | /28 | 20.25.30.130 | **Estática** | Servidor HTTPS |

> El rango DHCP de VLAN 10 es `20.25.30.3 – 20.25.30.126`. En la red de la nube no hay gateway porque los tres equipos (PC y ambos FortiGates) están en el mismo segmento y no se necesita salida a Internet.

---

## 3. Nube PNET y Switch de Usuarios

### 3.1 Configuración de la Nube PNET

Los dos FortiGates y la PC local se conectan al mismo nodo **Cloud** de PNETLab, que funciona como un switch: todos quedan en la red `203.0.113.0/29`.

**Adaptador de la PC:** en el adaptador virtual que usa la VM de PNETLab (por ejemplo VMnet8 o Host-only), configurar una IP estática:

| Campo | Valor |
|---|---|
| IP | `203.0.113.1` |
| Máscara | `255.255.255.248` |
| Gateway | *(vacío)* |

**En el laboratorio (PNETLab):**

1. Clic derecho en el área de trabajo → `Add an object → Network`.
2. Type: `Management(Cloud0)`, nombre `Nube-PNET`.
3. Conectar `port1` de FortiGate-A a `Nube-PNET`.
4. Conectar `port1` de FortiGate-B a `Nube-PNET`.
5. Conectar `port2` de FortiGate-A a `e0/0` de `SW-USUARIOS` (sección 3.2); `e0/1` del switch va al Usuario.
6. Conectar `port2` de FortiGate-B al Web Server.

### 3.2 Switch de Usuarios (VLAN 10)

Se agrega un switch L2 (`Cisco IOL L2` en PNETLab) entre FortiGate-A y el Usuario para que la VLAN 10 sea real: el puerto hacia el FortiGate es un **trunk 802.1Q** y el puerto del Usuario es un **access en VLAN 10**. Configuración en consola del switch (script: [`scripts/sw-usuarios.txt`](scripts/sw-usuarios.txt)):

```bash
enable
configure terminal

hostname SW-USUARIOS

vlan 10
 name USUARIOS
exit

interface Ethernet0/0
 description Trunk hacia FortiGate-A port2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10
 no shutdown
exit

interface Ethernet0/1
 description Usuario - VLAN 10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit

end
write memory
```

**Verificación:**
```bash
show vlan brief
show interfaces trunk
```
Debe mostrar VLAN 10 `USUARIOS` con `Et0/1` y el trunk `Et0/0` activo con VLAN 10 permitida.

> Ver evidencia: [00_switch_vlan10.png](screenshots/00_switch_vlan10.png)

---

## 4. Configuraciones del FortiGate-A (Sitio Usuarios) por la GUI

### 4.0 Acceso Inicial — CLI

```bash
config system interface
    edit "port1"
        set mode static
        set ip 203.0.113.2 255.255.255.248
        set allowaccess https ssh ping
        set role wan
    next
end
```

Acceder luego desde el navegador de la PC a `https://203.0.113.2` con las credenciales por defecto (`admin` / contraseña vacía) y definir una contraseña segura. (Script: [`scripts/fortigate-a-cli.txt`](scripts/fortigate-a-cli.txt))

> Ver evidencia: [01_cli_acceso_fga.png](screenshots/01_cli_acceso_fga.png)

### 4.1 Configuración de Interfaces

**Ruta:** `Network → Interfaces`

**port1 — WAN-NUBE:**

| Campo | Valor |
|---|---|
| Role | `WAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `203.0.113.2 / 255.255.255.248` |
| Administrative access | `HTTPS, SSH, Ping` |

**port2 — TRUNK-SW (físico, sin IP):**

| Campo | Valor |
|---|---|
| Alias | `TRUNK-SW` |
| Role | `Undefined` |
| Addressing mode | `Manual` (`0.0.0.0/0.0.0.0`) |

> `port2` solo transporta las tramas etiquetadas de la VLAN 10 hacia el switch; la IP del gateway va en la interfaz VLAN (siguiente sección).

> Ver evidencia: [02_interfaces_fga.png](screenshots/02_interfaces_fga.png)

### 4.2 Interfaz VLAN 10

**Ruta:** `Network → Interfaces → Create New → Interface`

| Campo | Valor |
|---|---|
| Name | `VLAN10` |
| Alias | `LAN-USUARIOS` |
| Type | `VLAN` |
| Interface | `port2` |
| VLAN ID | `10` |
| Role | `LAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `20.25.30.2 / 255.255.255.128` |
| Administrative access | `Ping` |

> Esta interfaz debe existir **antes** de correr el asistente IPsec, porque es la `Local interface` de la sección 6.2.

> Ver evidencia: [17_interfaz_vlan10_fga.png](screenshots/17_interfaz_vlan10_fga.png)

### 4.3 DHCP en VLAN 10 (Usuarios)

**Ruta:** `Network → Interfaces → VLAN10 → Edit → DHCP Server`

| Campo | Valor |
|---|---|
| Status | `Enable` |
| Address Range | `20.25.30.3 – 20.25.30.126` |
| Netmask | `255.255.255.128` |
| Default Gateway | `20.25.30.2` |
| DNS Server | `8.8.8.8` / `8.8.4.4` |
| Lease Time | `1 day` |

> Ver evidencia: [03_dhcp_fga.png](screenshots/03_dhcp_fga.png)

### 4.4 Política de NAT hacia la WAN

Crear **después** de la VPN (secciones 6.x), para que quede debajo de las políticas del túnel en la lista.

**Ruta:** `Policy & Objects → Firewall Policy → Create New`

| Campo | Valor |
|---|---|
| Name | `Usuarios-to-WAN` |
| Incoming Interface | `VLAN10 (LAN-USUARIOS)` |
| Outgoing Interface | `port1 (WAN-NUBE)` |
| Source | `all` |
| Destination | `all` |
| Schedule | `always` |
| Service | `ALL` |
| Action | `ACCEPT` |
| NAT | ✅ Enabled — `Use Outgoing Interface Address` |

> No se configura ruta por defecto: `203.0.113.0/29` es una red directamente conectada, por lo que el tráfico hacia la nube (PC local, otro FortiGate) sale por `port1` con la IP `203.0.113.2`. El tráfico hacia `20.25.30.128/28` solo tiene ruta a través del túnel.

> Ver evidencia: [18_politica_nat_fga.png](screenshots/18_politica_nat_fga.png)

---

## 5. Configuraciones del FortiGate-B (Sitio Servidor) por la GUI

### 5.0 Acceso Inicial — CLI

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

Acceder desde el navegador de la PC a `https://203.0.113.3`. (Script: [`scripts/fortigate-b-cli.txt`](scripts/fortigate-b-cli.txt))

### 5.1 Configuración de Interfaces

**port1 — WAN-NUBE:**

| Campo | Valor |
|---|---|
| Role | `WAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `203.0.113.3 / 255.255.255.248` |
| Administrative access | `HTTPS, SSH, Ping` |

**port2 — LAN-SERVIDOR:**

| Campo | Valor |
|---|---|
| Role | `LAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `20.25.30.130 / 255.255.255.240` |
| Administrative access | `Ping` |

> Ver evidencia: [05_interfaces_fgb.png](screenshots/05_interfaces_fgb.png)

### 5.2 Política de NAT hacia la WAN

Crear **después** de la VPN, debajo de las políticas del túnel.

**Ruta:** `Policy & Objects → Firewall Policy → Create New`

| Campo | Valor |
|---|---|
| Name | `Servidor-to-WAN` |
| Incoming Interface | `port2 (LAN-SERVIDOR)` |
| Outgoing Interface | `port1 (WAN-NUBE)` |
| Source | `all` |
| Destination | `all` |
| Schedule | `always` |
| Service | `ALL` |
| Action | `ACCEPT` |
| NAT | ✅ Enabled — `Use Outgoing Interface Address` |

> Ver evidencia: [19_politica_nat_fgb.png](screenshots/19_politica_nat_fgb.png)

---

## 6. VPN Site-to-Site (IPsec)

Se configura un túnel **IPsec Site-to-Site** entre `203.0.113.2` (FortiGate-A) y `203.0.113.3` (FortiGate-B), usando el asistente de FortiGate (`VPN → IPsec Wizard`) en modo **Site to Site**, que crea automáticamente la interfaz de túnel, Fase 1, Fase 2 y las rutas asociadas.

### 6.1 Fase 1 — FortiGate-A

**Ruta:** `VPN → IPsec Wizard → Create New`

| Campo | Valor |
|---|---|
| Name | `VPN-A-to-B` |
| Template type | `Site to Site` |
| NAT configuration | `No NAT between sites` |
| Remote Device Type | `FortiGate` |
| Remote IP Address | `203.0.113.3` |
| Outgoing Interface | `port1` |
| Authentication Method | `Pre-shared Key` |
| Pre-shared Key | *(clave fuerte, la misma en ambos extremos)* |
| IKE Version | `2` |

> **NAT configuration → `No NAT between sites`:** ninguno de los dos FortiGates está detrás de un dispositivo que haga NAT. Ambos están en el mismo segmento `203.0.113.0/29` de la Nube PNET y se ven con su IP real, así que no hace falta NAT-Traversal. Se usa la misma opción en FortiGate-B (sección 6.3). No confundir con la política de NAT de las secciones 4.4 y 5.2, que es solo para el tráfico hacia la WAN.

### 6.2 Fase 2 — FortiGate-A

Paso 3 del asistente (**Policy & Routing**):

| Campo | Valor |
|---|---|
| Local interface | `VLAN10` (LAN-USUARIOS) |
| Local subnets | `20.25.30.0/25` (red de Usuarios) |
| Remote Subnets | `20.25.30.128/28` (red del Servidor) |
| Internet Access | `None` |

> **Internet Access → `None`:** el laboratorio no usa Internet. `Share Local` y `Use Remote` agregarían políticas y rutas para sacar tráfico a Internet a través del túnel, y aquí solo debe viajar por él el tráfico entre las dos LAN.

> El wizard crea automáticamente la interfaz virtual `VPN-A-to-B` (tipo tunnel) y una ruta estática hacia `20.25.30.128/28` a través de ella — verificar en `Network → Static Routes` que quedó creada.

> Ver evidencia: [07_ipsec_fase1_fga.png](screenshots/07_ipsec_fase1_fga.png), [08_ipsec_fase2_fga.png](screenshots/08_ipsec_fase2_fga.png)

### 6.3 Fase 1 — FortiGate-B

Configuración espejo, apuntando de vuelta hacia FortiGate-A:

| Campo | Valor |
|---|---|
| Name | `VPN-B-to-A` |
| Template type | `Site to Site` |
| NAT configuration | `No NAT between sites` |
| Remote Device Type | `FortiGate` |
| Remote IP Address | `203.0.113.2` |
| Outgoing Interface | `port1` |
| Authentication Method | `Pre-shared Key` |
| Pre-shared Key | *(la misma clave configurada en FortiGate-A)* |
| IKE Version | `2` |

### 6.4 Fase 2 — FortiGate-B

Paso 3 del asistente (**Policy & Routing**):

| Campo | Valor |
|---|---|
| Local interface | `port2` (LAN-SERVIDOR) |
| Local subnets | `20.25.30.128/28` (red del Servidor) |
| Remote Subnets | `20.25.30.0/25` (red de Usuarios) |
| Internet Access | `None` |

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
| Incoming Interface | `VLAN10 (LAN-USUARIOS)` |
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
| Outgoing Interface | `VLAN10 (LAN-USUARIOS)` |
| Source | `Web Server (20.25.30.131)` |
| Destination | `all` |
| Service | `ALL` |
| Action | `ACCEPT` |
| NAT | ❌ Disabled |

**En FortiGate-B:** políticas espejo, `VPN-B-to-A ↔ port2 (LAN-SERVIDOR)`, mismos criterios de origen/destino invertidos.

> **Importante:** el tráfico Usuario→Servidor solo tiene ruta por el túnel y política por el túnel. La política de NAT (`Usuarios-to-WAN`) no lo cubre en la práctica: `20.25.30.131` no es alcanzable por `port1`, ya que no existe ruta por defecto. Así se garantiza que la única forma de llegar al servidor es a través de la VPN.

> Ver evidencia: [11_politicas_vpn_fga.png](screenshots/11_politicas_vpn_fga.png), [12_politicas_vpn_fgb.png](screenshots/12_politicas_vpn_fgb.png)

---

## 7. Web Server (HTTPS)

Servidor Ubuntu con Apache + certificado autofirmado (script completo: [`scripts/webserver-https.sh`](scripts/webserver-https.sh)):

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

Desde el Usuario (VLAN 10, con IP por DHCP):
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

Repetir el `traceroute` y el `curl` desde el Usuario — ambos deben **fallar/quedar colgados**, confirmando que no existe ninguna ruta alterna hacia el servidor sin el túnel.

Volver a levantar el túnel (`Bring Up`) y repetir la prueba para mostrar que se recupera.

**8.3 — Verificación de NAT:**

Desde el Usuario, hacer ping a la PC local (permitir ICMP en el firewall de Windows si hace falta):
```
ping 203.0.113.1
```
En `Log & Report → Forward Traffic` de FortiGate-A se ve el tráfico con la política `Usuarios-to-WAN` y la IP de origen traducida a `203.0.113.2`. Repetir desde el Web Server en FortiGate-B (origen traducido a `203.0.113.3`).

> Ver evidencia: [13_traceroute_tunel_activo.png](screenshots/13_traceroute_tunel_activo.png), [14_traceroute_tunel_caido.png](screenshots/14_traceroute_tunel_caido.png), [15_ipsec_monitor.png](screenshots/15_ipsec_monitor.png), [20_prueba_nat.png](screenshots/20_prueba_nat.png)

---

## 9. Capturas de Pantalla

| # | Archivo | Descripción |
|---|---|---|
| 00 | [`00_switch_vlan10.png`](screenshots/00_switch_vlan10.png) | Consola de SW-USUARIOS con `show vlan brief` y `show interfaces trunk`. |
| 01 | [`01_cli_acceso_fga.png`](screenshots/01_cli_acceso_fga.png) | Terminal CLI de FortiGate-A mostrando la config inicial de `port1` (203.0.113.2/29) y el login de la GUI. |
| 02 | [`02_interfaces_fga.png`](screenshots/02_interfaces_fga.png) | `Network → Interfaces` de FortiGate-A: port1 WAN, port2 físico y VLAN10. |
| 03 | [`03_dhcp_fga.png`](screenshots/03_dhcp_fga.png) | Servidor DHCP en la interfaz VLAN10 de FortiGate-A, rango `20.25.30.3–126`. |
| 05 | [`05_interfaces_fgb.png`](screenshots/05_interfaces_fgb.png) | `Network → Interfaces` de FortiGate-B: port1 WAN y port2 LAN-SERVIDOR configuradas. |
| 07 | [`07_ipsec_fase1_fga.png`](screenshots/07_ipsec_fase1_fga.png) | Fase 1 de la VPN en FortiGate-A, remote gateway `203.0.113.3`. |
| 08 | [`08_ipsec_fase2_fga.png`](screenshots/08_ipsec_fase2_fga.png) | Fase 2 de la VPN en FortiGate-A, subredes local/remota. |
| 09 | [`09_ipsec_fase1_fgb.png`](screenshots/09_ipsec_fase1_fgb.png) | Fase 1 de la VPN en FortiGate-B, remote gateway `203.0.113.2`. |
| 10 | [`10_ipsec_fase2_fgb.png`](screenshots/10_ipsec_fase2_fgb.png) | Fase 2 de la VPN en FortiGate-B, subredes local/remota. |
| 11 | [`11_politicas_vpn_fga.png`](screenshots/11_politicas_vpn_fga.png) | Políticas de firewall en FortiGate-A para el tráfico hacia/desde la VPN. |
| 12 | [`12_politicas_vpn_fgb.png`](screenshots/12_politicas_vpn_fgb.png) | Políticas de firewall en FortiGate-B para el tráfico hacia/desde la VPN. |
| 13 | [`13_traceroute_tunel_activo.png`](screenshots/13_traceroute_tunel_activo.png) | Traceroute exitoso desde el Usuario al Web Server con el túnel activo. |
| 14 | [`14_traceroute_tunel_caido.png`](screenshots/14_traceroute_tunel_caido.png) | Traceroute fallido desde el Usuario al Web Server con el túnel caído — confirma que no hay ruta alterna. |
| 15 | [`15_ipsec_monitor.png`](screenshots/15_ipsec_monitor.png) | `Monitor → IPsec Monitor` mostrando el túnel `Up` y luego `Down` durante la prueba. |
| 16 | [`16_interfaz_vlan10_fga.png`](screenshots/16_interfaz_vlan10_fga.png) | Interfaz VLAN10 (ID 10 sobre port2) en FortiGate-A con IP `20.25.30.2/25`. |
| 18 | [`17_politica_nat_fga.png`](screenshots/17_politica_nat_fga.png) | Política `Usuarios-to-WAN` con NAT habilitado en FortiGate-A. |
| 19 | [`18_politica_nat_fgb.png`](screenshots/18_politica_nat_fgb.png) | Política `Servidor-to-WAN` con NAT habilitado en FortiGate-B. |
| 20 | [`19_prueba_nat.png`](screenshots/19_prueba_nat.png) | Ping desde el Usuario a la PC y log de Forward Traffic mostrando la IP traducida. |

---

## 10. Estructura del Repositorio

```
/
├── README.md                  ← este documento
├── screenshots/                ← capturas numeradas de cada configuración
├── scripts/
│   ├── sw-usuarios.txt        ← configuración del switch (VLAN 10, trunk/access)
│   ├── fortigate-a-cli.txt    ← acceso inicial FortiGate-A
│   ├── fortigate-b-cli.txt    ← acceso inicial FortiGate-B
│   └── webserver-https.sh     ← Apache + certificado autofirmado
├── running-configs/
│   ├── sw-usuarios-running-config.txt
│   ├── fortigate-a-running-config.conf
│   └── fortigate-b-running-config.conf
└── entregable/
    └── ArleneFernandez_20250730_P3.txt
```

> Ajustar el número de práctica (`P3`) según lo indicado por el profesor. El video debe subirse al principio del repositorio (enlace ya colocado arriba en este README).
