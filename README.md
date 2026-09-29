# FortiGate — VPN Site-to-Site entre dos FortiGates

### Arlene Fernández Herrera · Matrícula: 2025-0730

**Seguridad de Redes · ITLA**

---

## 🎥 Video Demostrativo

**[▶ Ver video de demostración](https://youtu.be/REEMPLAZAR-CON-TU-ID)**

---

> ⚠️ **Aclaración importante:** FortiGate-A y FortiGate-B **no tienen la misma versión de FortiOS** (FortiGate-A: `v7.0.3` · FortiGate-B: `v7.6.2`). Por eso el asistente de VPN se ve distinto en cada equipo: en FortiGate-A son pasos numerados y en FortiGate-B es una sola pantalla con tres bloques. Los valores de configuración son los mismos (en espejo) en ambos equipos.

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Topología y Direccionamiento](#2-topología-y-direccionamiento)
3. [Procedimiento paso a paso](#3-procedimiento-paso-a-paso)
   - [Paso 1. Nube PNET y PC local](#paso-1-nube-pnet-y-pc-local)
   - [Paso 2. Switch de Usuarios (VLAN 10)](#paso-2-switch-de-usuarios-vlan-10)
   - [Paso 3. Acceso inicial de FortiGate-A (CLI)](#paso-3-acceso-inicial-de-fortigate-a-cli)
   - [Paso 4. Acceso inicial de FortiGate-B (CLI)](#paso-4-acceso-inicial-de-fortigate-b-cli)
   - [Paso 5. Interfaces y DHCP de FortiGate-A](#paso-5-interfaces-y-dhcp-de-fortigate-a)
   - [Paso 6. Interfaces de FortiGate-B](#paso-6-interfaces-de-fortigate-b)
   - [Paso 7. VPN IPsec en FortiGate-A](#paso-7-vpn-ipsec-en-fortigate-a)
   - [Paso 8. VPN IPsec en FortiGate-B](#paso-8-vpn-ipsec-en-fortigate-b)
   - [Paso 9. Verificar rutas estáticas hacia el túnel](#paso-9-verificar-rutas-estáticas-hacia-el-túnel)
   - [Paso 10. Verificar políticas de firewall de la VPN](#paso-10-verificar-políticas-de-firewall-de-la-vpn)
   - [Paso 11. Políticas de NAT hacia la WAN](#paso-11-políticas-de-nat-hacia-la-wan)
   - [Paso 12. Web Server (HTTPS)](#paso-12-web-server-https)
   - [Paso 13. Pruebas de verificación](#paso-13-pruebas-de-verificación)
4. [Capturas de Pantalla](#4-capturas-de-pantalla)
5. [Estructura del Repositorio](#5-estructura-del-repositorio)

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

## 3. Procedimiento paso a paso

Los pasos están en el orden en que se ejecutan. Cada uno depende de los anteriores.

---

### Paso 1. Nube PNET y PC local

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
5. Conectar `port2` de FortiGate-A a `e0/0` de `SW-USUARIOS` (Paso 2); `e0/1` del switch va al Usuario.
6. Conectar `port2` de FortiGate-B al Web Server.

---

### Paso 2. Switch de Usuarios (VLAN 10)

Se agrega un switch L2 (`Cisco IOL L2` en PNETLab) entre FortiGate-A y el Usuario para que la VLAN 10 sea real: el puerto hacia el FortiGate es un **trunk 802.1Q** y el puerto del Usuario es un **access en VLAN 10**. Configuración en la consola del switch (script: [`scripts/sw-usuarios.txt`](scripts/sw-usuarios.txt)):

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

> Ver evidencia: [01_switch_vlan10.png](screenshots/01_switch_vlan10.png)

---

### Paso 3. Acceso inicial de FortiGate-A (CLI)

Desde la consola de FortiGate-A (script: [`scripts/fortigate-a-cli.txt`](scripts/fortigate-a-cli.txt)):

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

> Ver evidencia: [02_cli_acceso_fga.png](screenshots/02_cli_acceso_fga.png)

---

### Paso 4. Acceso inicial de FortiGate-B (CLI)

Desde la consola de FortiGate-B (script: [`scripts/fortigate-b-cli.txt`](scripts/fortigate-b-cli.txt)):

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

Acceder desde el navegador de la PC a `https://203.0.113.3` y definir una contraseña segura.

> Ver evidencia: [03_cli_acceso_fgb.png](screenshots/03_cli_acceso_fgb.png)

---

### Paso 5. Interfaces y DHCP de FortiGate-A

Todo por GUI en `https://203.0.113.2`.

#### 5.1 Interfaces físicas

**Ruta:** `Network → Interfaces`

**port1 — WAN-NUBE** (ya quedó con IP en el Paso 3; se completa el resto):

| Campo | Valor |
|---|---|
| Alias | `WAN-NUBE` |
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

> Ver evidencia: [04_interfaces_fga.png](screenshots/04_interfaces_fga.png)

#### 5.2 Interfaz VLAN 10

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

> Esta interfaz debe existir **antes** del asistente IPsec del Paso 7, porque es la `Local interface` de la VPN.

> Ver evidencia: [05_interfaz_vlan10_fga.png](screenshots/05_interfaz_vlan10_fga.png)

#### 5.3 DHCP en VLAN 10 (Usuarios)

**Ruta:** `Network → Interfaces → VLAN10 → Edit → DHCP Server`

| Campo | Valor |
|---|---|
| Status | `Enable` |
| Address Range | `20.25.30.3 – 20.25.30.126` |
| Netmask | `255.255.255.128` |
| Default Gateway | `20.25.30.2` |
| DNS Server | `8.8.8.8` / `8.8.4.4` |
| Lease Time | `1 day` |

> Ver evidencia: [06_dhcp_fga.png](screenshots/06_dhcp_fga.png)

---

### Paso 6. Interfaces de FortiGate-B

Todo por GUI en `https://203.0.113.3`.

**Ruta:** `Network → Interfaces`

**port1 — WAN-NUBE:**

| Campo | Valor |
|---|---|
| Alias | `WAN-NUBE` |
| Role | `WAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `203.0.113.3 / 255.255.255.248` |
| Administrative access | `HTTPS, SSH, Ping` |

**port2 — LAN-SERVIDOR:**

| Campo | Valor |
|---|---|
| Alias | `LAN-SERVIDOR` |
| Role | `LAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `20.25.30.130 / 255.255.255.240` |
| Administrative access | `Ping` |

> Ver evidencia: [07_interfaces_fgb.png](screenshots/07_interfaces_fgb.png)

---

### Paso 7. VPN IPsec en FortiGate-A

Se configura un túnel **IPsec Site-to-Site** entre `203.0.113.2` (FortiGate-A) y `203.0.113.3` (FortiGate-B) con el asistente de FortiGate en modo **Site to Site**, que crea automáticamente la interfaz de túnel, Fase 1, Fase 2, la ruta estática y las políticas asociadas.

#### 7.1 Fase 1

**Ruta (FortiOS 7.0.3):** `VPN → IPsec Wizard → Create New`

| Campo | Valor |
|---|---|
| Name | `VPN-A-to-B` |
| Template type | `Site to Site` |
| NAT configuration | `No NAT between sites` |
| Remote Device Type | `FortiGate` |
| Remote IP Address | `203.0.113.3` |
| Outgoing Interface | `port1` |
| Authentication Method | `Pre-shared Key` |
| Pre-shared Key | *Arlene123.* |
| IKE Version | `2` |

> **NAT configuration → `No NAT between sites`:** ninguno de los dos FortiGates está detrás de un dispositivo que haga NAT. Ambos están en el mismo segmento `203.0.113.0/29` y se ven con su IP real, así que no hace falta NAT-Traversal. No confundir con la política de NAT del Paso 11, que es solo para el tráfico hacia la WAN.

> Ver evidencia: [08_ipsec_fase1_fga.png](screenshots/08_ipsec_fase1_fga.png)

#### 7.2 Fase 2

Paso 3 del asistente (**Policy & Routing**):

| Campo | Valor |
|---|---|
| Local interface | `VLAN10` (LAN-USUARIOS) |
| Local subnets | `20.25.30.0/25` (red de Usuarios) |
| Remote Subnets | `20.25.30.128/28` (red del Servidor) |
| Internet Access | `None` |

> **Internet Access → `None`:** el laboratorio no usa Internet. `Share Local` y `Use Remote` agregarían políticas y rutas para sacar tráfico a Internet a través del túnel, y aquí solo debe viajar por él el tráfico entre las dos LAN.

> Ver evidencia: [09_ipsec_fase2_fga.png](screenshots/09_ipsec_fase2_fga.png)

---

### Paso 8. VPN IPsec en FortiGate-B

Configuración espejo de la de FortiGate-A.

**Ruta (FortiOS 7.6.2):** `VPN → VPN Wizard`

En 7.6.2 el asistente es una sola pantalla con tres bloques (**VPN Tunnel**, **Remote Site**, **Local FortiGate**), no pasos numerados. Nombre del asistente: `VPN-B-to-A`.

**Bloque VPN Tunnel:**

| Campo | Valor |
|---|---|
| Authentication method | `Pre-shared key` |
| Pre-shared key | *Arlene123.* |
| IKE | `Version 2` |
| Transport | `Auto` |
| Use Fortinet encapsulation | Desactivado |
| NAT traversal | `Disable` |

> **NAT traversal → `Disable`:** equivale al `No NAT between sites` de FortiGate-A.

> Ver evidencia: [10_ipsec_tunel_fgb.png](screenshots/10_ipsec_tunel_fgb.png)

**Bloque Remote Site:**

| Campo | Valor |
|---|---|
| Remote site device type | `FortiGate` |
| Remote site device | `Accessible and static` |
| IP/FQDN | `203.0.113.2` |
| Route this device's internet traffic through the remote site | Desactivado |
| Remote site subnets that can access VPN | `20.25.30.0/25` (red de Usuarios) |

**Bloque Local FortiGate:**

| Campo | Valor |
|---|---|
| Outgoing interface that binds to tunnel | `port1` |
| Create and add interface to zone | Activado (valor por defecto) |
| Local interface | `port2` (LAN-SERVIDOR) |
| Local subnets that can access VPN | `20.25.30.128/28` (red del Servidor) |
| Allow remote site's internet traffic through this device | Desactivado |

> **Tráfico de Internet desactivado en ambos sentidos:** equivale a `Internet Access = None` de FortiGate-A. Por el túnel solo debe viajar el tráfico entre las dos LAN.

> Ver evidencia: [11_ipsec_remoto_local_fgb.png](screenshots/11_ipsec_remoto_local_fgb.png)

---

### Paso 9. Verificar rutas estáticas hacia el túnel

**Ruta:** `Network → Static Routes`

Los asistentes crean estas rutas automáticamente. Si alguna no existe, crearla con `Create New`:

| Equipo | Destination | Interface |
|---|---|---|
| FortiGate-A | `20.25.30.128/28` | `VPN-A-to-B` |
| FortiGate-B | `20.25.30.0/25` | `VPN-B-to-A` |

> No se configura ruta por defecto: `203.0.113.0/29` es una red directamente conectada y las LAN internas solo tienen ruta a través del túnel.

> Ver evidencia: [12_rutas_estaticas.png](screenshots/12_rutas_estaticas.png)

---

### Paso 10. Verificar políticas de firewall de la VPN

**Ruta:** `Policy & Objects → Firewall Policy`

Los asistentes suelen crear estas políticas; verificar y ajustar si hace falta.

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

> Ver evidencia: [13_politicas_vpn_fga.png](screenshots/13_politicas_vpn_fga.png)

**En FortiGate-B:** políticas espejo, `VPN-B-to-A ↔ port2 (LAN-SERVIDOR)`, con origen y destino invertidos.

> Ver evidencia: [14_politicas_vpn_fgb.png](screenshots/14_politicas_vpn_fgb.png)

---

### Paso 11. Políticas de NAT hacia la WAN

Se crean **después** de las políticas de la VPN, para que queden debajo de ellas en la lista.

**Ruta:** `Policy & Objects → Firewall Policy → Create New`

**FortiGate-A:**

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

> Ver evidencia: [15_politica_nat_fga.png](screenshots/15_politica_nat_fga.png)

**FortiGate-B:**

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

> Ver evidencia: [16_politica_nat_fgb.png](screenshots/16_politica_nat_fgb.png)

> **Importante:** el tráfico Usuario→Servidor no queda cubierto por estas políticas de NAT: `20.25.30.131` no es alcanzable por `port1` porque no existe ruta por defecto. Así se garantiza que la única forma de llegar al servidor es a través de la VPN.

---

### Paso 12. Web Server (HTTPS)

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

### Paso 13. Pruebas de verificación

**13.1 — Con el túnel activo**

Desde el Usuario (VLAN 10, con IP por DHCP):
```
traceroute 20.25.30.131
```
Debe completar en pocos saltos, atravesando la interfaz `VPN-A-to-B`.

```
curl -k https://20.25.30.131/
```
Debe responder con el contenido del Web Server.

> Ver evidencia: [17_traceroute_tunel_activo.png](screenshots/17_traceroute_tunel_activo.png)

**13.2 — Con el túnel caído (no hay ruta alterna)**

En cualquiera de los dos FortiGates: `VPN → IPsec Tunnels → VPN-A-to-B → Bring Down` (o deshabilitar temporalmente la Fase 1).

Repetir el `traceroute` y el `curl` desde el Usuario: ambos deben **fallar o quedar colgados**. Luego volver a levantar el túnel (`Bring Up`) y repetir la prueba para mostrar que se recupera.

> Ver evidencia: [18_traceroute_tunel_caido.png](screenshots/18_traceroute_tunel_caido.png), [19_ipsec_monitor.png](screenshots/19_ipsec_monitor.png)

**13.3 — Verificación de NAT**

Desde el Usuario, hacer ping a la PC local (permitir ICMP en el firewall de Windows si hace falta):
```
ping 203.0.113.1
```
En `Log & Report → Forward Traffic` de FortiGate-A se ve el tráfico con la política `Usuarios-to-WAN` y la IP de origen traducida a `203.0.113.2`. Repetir desde el Web Server en FortiGate-B (origen traducido a `203.0.113.3`).

> Ver evidencia: [20_prueba_nat.png](screenshots/20_prueba_nat.png)

---

## 4. Capturas de Pantalla

Numeradas en el orden en que se toman durante el procedimiento.

| # | Archivo | Paso | Descripción |
|---|---|---|---|
| 01 | [`01_switch_vlan10.png`](screenshots/01_switch_vlan10.png) | 2 | Consola de SW-USUARIOS con `show vlan brief` y `show interfaces trunk`. |
| 02 | [`02_cli_acceso_fga.png`](screenshots/02_cli_acceso_fga.png) | 3 | CLI de FortiGate-A con la config inicial de `port1` (203.0.113.2/29). |
| 03 | [`03_cli_acceso_fgb.png`](screenshots/03_cli_acceso_fgb.png) | 4 | CLI de FortiGate-B con la config inicial de `port1` (203.0.113.3/29). |
| 04 | [`04_interfaces_fga.png`](screenshots/04_interfaces_fga.png) | 5.1 | `Network → Interfaces` de FortiGate-A: port1 WAN, port2 físico y VLAN10. |
| 05 | [`05_interfaz_vlan10_fga.png`](screenshots/05_interfaz_vlan10_fga.png) | 5.2 | Interfaz VLAN10 (ID 10 sobre port2) con IP `20.25.30.2/25`. |
| 06 | [`06_dhcp_fga.png`](screenshots/06_dhcp_fga.png) | 5.3 | Servidor DHCP en VLAN10, rango `20.25.30.3–126`. |
| 07 | [`07_interfaces_fgb.png`](screenshots/07_interfaces_fgb.png) | 6 | `Network → Interfaces` de FortiGate-B: port1 WAN y port2 LAN-SERVIDOR. |
| 08 | [`08_ipsec_fase1_fga.png`](screenshots/08_ipsec_fase1_fga.png) | 7.1 | Fase 1 de la VPN en FortiGate-A, remote gateway `203.0.113.3`. |
| 09 | [`09_ipsec_fase2_fga.png`](screenshots/09_ipsec_fase2_fga.png) | 7.2 | Fase 2 de la VPN en FortiGate-A, subredes local/remota. |
| 10 | [`10_ipsec_tunel_fgb.png`](screenshots/10_ipsec_tunel_fgb.png) | 8 | Asistente de FortiGate-B (7.6.2), bloque VPN Tunnel. |
| 11 | [`11_ipsec_remoto_local_fgb.png`](screenshots/11_ipsec_remoto_local_fgb.png) | 8 | Asistente de FortiGate-B (7.6.2), bloques Remote Site y Local FortiGate. |
| 12 | [`12_rutas_estaticas.png`](screenshots/12_rutas_estaticas.png) | 9 | `Network → Static Routes` con la ruta hacia el túnel. |
| 13 | [`13_politicas_vpn_fga.png`](screenshots/13_politicas_vpn_fga.png) | 10 | Políticas de firewall de la VPN en FortiGate-A. |
| 14 | [`14_politicas_vpn_fgb.png`](screenshots/14_politicas_vpn_fgb.png) | 10 | Políticas de firewall de la VPN en FortiGate-B. |
| 15 | [`15_politica_nat_fga.png`](screenshots/15_politica_nat_fga.png) | 11 | Política `Usuarios-to-WAN` con NAT habilitado. |
| 16 | [`16_politica_nat_fgb.png`](screenshots/16_politica_nat_fgb.png) | 11 | Política `Servidor-to-WAN` con NAT habilitado. |
| 17 | [`17_traceroute_tunel_activo.png`](screenshots/17_traceroute_tunel_activo.png) | 13.1 | Traceroute exitoso del Usuario al Web Server con el túnel activo. |
| 18 | [`18_traceroute_tunel_caido.png`](screenshots/18_traceroute_tunel_caido.png) | 13.2 | Traceroute fallido con el túnel caído: no hay ruta alterna. |
| 19 | [`19_ipsec_monitor.png`](screenshots/19_ipsec_monitor.png) | 13.2 | `Monitor → IPsec Monitor` con el túnel `Up` y luego `Down`. |
| 20 | [`20_prueba_nat.png`](screenshots/20_prueba_nat.png) | 13.3 | Ping del Usuario a la PC y log de Forward Traffic con la IP traducida. |

---

## 5. Estructura del Repositorio

```
/
├── README.md                  ← este documento
├── screenshots/               ← capturas numeradas de cada configuración
├── scripts/
│   ├── sw-usuarios.txt        ← configuración del switch (VLAN 10, trunk/access)
│   ├── fortigate-a-cli.txt    ← acceso inicial FortiGate-A
│   ├── fortigate-b-cli.txt    ← acceso inicial FortiGate-B
│   └── webserver-https.sh     ← Apache + certificado autofirmado
├── running-configs/
│   ├── sw-usuarios-running-config.txt
│   ├── fortigate-a-running-config.conf
│   └── fortigate-b-running-config.conf
```
