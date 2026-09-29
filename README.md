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
3. [Configuración de la Nube PNET](#3-configuración-de-la-nube-pnet)
4. [Configuraciones del FortiGate-A (Sitio Usuarios) por la GUI](#4-configuraciones-del-fortigate-a-sitio-usuarios-por-la-gui)
   - [4.0 Acceso Inicial — CLI](#40-acceso-inicial--cli)
   - [4.1 Configuración de Interfaces](#41-configuración-de-interfaces)
   - [4.2 DHCP en VLAN10 (Usuarios)](#42-dhcp-en-vlan10-usuarios)
5. [Configuraciones del FortiGate-B (Sitio Servidor) por la GUI](#5-configuraciones-del-fortigate-b-sitio-servidor-por-la-gui)
   - [5.0 Acceso Inicial — CLI](#50-acceso-inicial--cli)
   - [5.1 Configuración de Interfaces](#51-configuración-de-interfaces)
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

Esta práctica implementa un **túnel VPN Site-to-Site (IPsec)** entre dos FortiGates que representan dos sitios distintos, conectados entre sí y con la PC local a través de una **Nube de PNETLab** en la red `203.0.113.0/29`. El objetivo es que el **Usuario** (VLAN 10, detrás de FortiGate-A) pueda comunicarse con el **Web Server** (detrás de FortiGate-B) **únicamente a través del enlace VPN** — no existe ninguna ruta ni política que permita ese tráfico fuera del túnel, así que si el túnel cae, la comunicación se cae con él. Esto se demuestra con `traceroute` desde el Usuario hacia el servidor, tanto con el túnel activo como caído.

Toda la configuración y demostración de ambos FortiGates se realiza **por GUI**, accedida desde el navegador de la PC local. El laboratorio no requiere salida a Internet.

---

## 2. Topología y Direccionamiento

> Direccionamiento de las LAN internas derivado de la matrícula **2025-0730** → base `20.25.30.0/24`. La red de la nube es `203.0.113.0/29` (rango reservado para documentación, RFC 5737): la `.1` es para la PC local y las demás para los FortiGates.

### 2.1 Diagrama de Topología

```
                      ┌───────────────────────────┐
   PC local ──────────┤       Nube PNET (Cloud)   │
   203.0.113.1        │       203.0.113.0/29      │
                      └─────┬───────────────┬─────┘
                            │               │
                     ┌──────┴───────┐ ┌─────┴────────┐
                     │  FortiGate-A │ │  FortiGate-B │
                     │ port1 (WAN)  │ │ port1 (WAN)  │
                     │ 203.0.113.2  │ │ 203.0.113.3  │
                     │ port2 (LAN)  │ │ port2 (LAN)  │
                     └──────┬───────┘ └─────┬────────┘
                            │ 20.25.30.0/25 │ 20.25.30.128/28
                     ┌──────┴───────┐ ┌─────┴────────┐
                     │   Usuario    │ │  Web Server  │
                     │   (DHCP)     │ │  (Estática)  │
                     │   VLAN 10    │ │    HTTPS     │
                     └──────────────┘ └──────────────┘

              ┄┄┄┄┄┄┄┄┄┄┄┄┄ VPN Site-to-Site (IPsec) ┄┄┄┄┄┄┄┄┄┄┄┄┄
                    túnel entre 203.0.113.2 ↔ 203.0.113.3

  Política de comunicación:
  ┌───────────────────────────────────────────────────────────────────┐
  │ Usuario → Web Server : SOLO a través del túnel VPN (IPsec)        │
  │ No existe ruta ni política que permita ese tráfico fuera del túnel│
  │ Si el túnel cae, el traceroute deja de completar hacia el servidor│
  └───────────────────────────────────────────────────────────────────┘
```

### 2.2 Tabla de Interfaces

**Nube PNET:**

| Elemento | Rol | Dirección IP | Máscara |
|---|---|---|---|
| **Red de la nube** | Segmento compartido PC + FortiGates | 203.0.113.0 | /29 |
| **PC local** | Acceso a la GUI de ambos FortiGates | 203.0.113.1 | /29 |

**FortiGate-A (Sitio Usuarios):**

| Interfaz | Alias | Rol | Dirección IP | Máscara |
|---|---|---|---|---|
| **port1** | WAN-NUBE | WAN | 203.0.113.2 | /29 |
| **port2** | LAN-USUARIOS | LAN | 20.25.30.2 | /25 |

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
| **FortiGate-A** | port2 | 20.25.30.2 | /25 | — | Estática | Gateway VLAN 10 (Usuarios) |
| **FortiGate-B** | port1 | 203.0.113.3 | /29 | — | Estática | WAN, extremo remoto de la VPN |
| **FortiGate-B** | port2 | 20.25.30.130 | /28 | — | Estática | Gateway LAN Servidor |
| **Usuario** | eth0 | 20.25.30.3 (rango) | /25 | 20.25.30.2 | **DHCP** | Cliente en VLAN 10 |
| **Web Server** | eth0 | 20.25.30.131 | /28 | 20.25.30.130 | **Estática** | Servidor HTTPS |

> El rango DHCP de VLAN 10 es `20.25.30.3 – 20.25.30.126`. En la red de la nube no hay gateway porque los tres equipos (PC y ambos FortiGates) están en el mismo segmento y no se necesita salida a Internet.

---

## 3. Configuración de la Nube PNET

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
5. Conectar `port2` de cada FortiGate a su LAN (Usuario / Web Server).

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

Acceder luego desde el navegador de la PC a `https://203.0.113.2` con las credenciales por defecto (`admin` / contraseña vacía) y definir una contraseña segura.

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

> **Nota:** ya no se configura ruta por defecto. Los dos FortiGates están en el mismo segmento `/29` y se ven directamente; la única ruta que necesitan es la del túnel (sección 6.5).

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

Acceder desde el navegador de la PC a `https://203.0.113.3`.

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
| Pre-shared Key | *Arlene3003.* |
| IKE Version | `2` |

> **NAT configuration → `No NAT between sites`:** ninguno de los dos FortiGates está detrás de un dispositivo que haga NAT. Ambos están en el mismo segmento `203.0.113.0/29` de la Nube PNET y se ven con su IP real, así que no hace falta NAT-Traversal. Se usa la misma opción en FortiGate-B (sección 6.3).

### 6.2 Fase 2 — FortiGate-A

Paso 3 del asistente (**Policy & Routing**):

| Campo | Valor |
|---|---|
| Local interface | `port2` (LAN-USUARIOS) |
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
| Pre-shared Key | *Arlene3003.* |
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

Repetir el `traceroute` y el `curl` desde el Usuario — ambos deben **fallar/quedar colgados**, confirmando que no existe ninguna ruta alterna hacia el servidor sin el túnel.

Volver a levantar el túnel (`Bring Up`) y repetir la prueba para mostrar que se recupera.

> Ver evidencia: [13_traceroute_tunel_activo.png](screenshots/13_traceroute_tunel_activo.png), [14_traceroute_tunel_caido.png](screenshots/14_traceroute_tunel_caido.png), [15_ipsec_monitor.png](screenshots/15_ipsec_monitor.png)

---

## 9. Capturas de Pantalla

| # | Archivo | Descripción |
|---|---|---|
| 00 | [`00_nube_pnet_config.png`](screenshots/00_nube_pnet_config.png) | Topología en PNETLab con la Nube conectada a `port1` de ambos FortiGates, y ping exitoso desde la PC a `203.0.113.2` y `203.0.113.3`. |
| 01 | [`01_cli_acceso_fga.png`](screenshots/01_cli_acceso_fga.png) | Terminal CLI de FortiGate-A mostrando la config inicial de `port1` (203.0.113.2/29) y el login de la GUI. |
| 02 | [`02_interfaces_fga.png`](screenshots/02_interfaces_fga.png) | `Network → Interfaces` de FortiGate-A: port1 WAN y port2 LAN-USUARIOS configuradas. |
| 03 | [`03_dhcp_fga.png`](screenshots/03_dhcp_fga.png) | Servidor DHCP en port2 de FortiGate-A, rango `20.25.30.3–126`. |
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

---

## 10. Estructura del Repositorio

```
/
├── README.md                  ← este documento
├── screenshots/                ← capturas numeradas de cada configuración
├── running-configs/
│   ├── fortigate-a-running-config.conf
│   └── fortigate-b-running-config.conf
└── entregable/
    └── ArleneFernandez_20250730_P3.txt
```

> Ajustar el número de práctica (`P3`) según lo indicado por el profesor. El video debe subirse al principio del repositorio (enlace ya colocado arriba en este README).
