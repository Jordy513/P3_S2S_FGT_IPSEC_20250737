# VPN Site-to-Site — IPSec Route-Based con FortiGate (IKEv1)

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [¿Qué es una VPN IPSec Site-to-Site en FortiGate?](#21-qué-es-una-vpn-ipsec-site-to-site-en-fortigate)
   - [Route-Based vs. Policy-Based VPN](#22-route-based-vs-policy-based-vpn)
   - [Arquitectura Phase1-interface / Phase2-interface](#23-arquitectura-phase1-interface--phase2-interface)
   - [Flujo de establecimiento de la conexión](#24-flujo-de-establecimiento-de-la-conexión)
   - [Site-to-Site vs. Client-to-Site](#25-site-to-site-vs-client-to-site)
   - [Parámetros del laboratorio](#26-parámetros-del-laboratorio)
3. [Documentación de la Red](#3-documentación-de-la-red)
   - [Topología](#31-topología)
   - [Tabla de Dispositivos y Direccionamiento IP](#32-tabla-de-dispositivos-y-direccionamiento-ip)
4. [Scripts de Configuración](#4-scripts-de-configuración)
   - [FortiGate1 — Sitio A](#41-fortigate1--sitio-a)
   - [FortiGate2 — Sitio B](#42-fortigate2--sitio-b)
   - [Configuración de PCs (Hosts)](#43-configuración-de-pcs-hosts)
5. [Verificación de la Conexión (GUI)](#5-verificación-de-la-conexión-gui)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Consideraciones de Seguridad](#7-consideraciones-de-seguridad)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo del Laboratorio

El objetivo de este laboratorio es **implementar y verificar una VPN Site-to-Site punto a punto utilizando IPSec sobre IKEv1 en modo Route-Based (basado en interfaz virtual)** entre dos dispositivos FortiGate. A través de esta práctica se busca demostrar:

* La configuración de un túnel IPSec entre dos FortiGate mediante `phase1-interface` y `phase2-interface`, usando una interfaz VPN virtual (`net-device disable`) que permite enrutar el tráfico hacia el túnel mediante rutas estáticas, en lugar de depender de políticas de firewall que amarran subredes fijas (modo Policy-Based).
* La negociación de la Fase 1 (ISAKMP) con autenticación por Pre-Shared Key y la Fase 2 (IPSec SA) definiendo los selectores de tráfico (`src-subnet` / `dst-subnet`) entre las LAN de cada sitio.
* El enrutamiento del tráfico hacia la LAN remota mediante una ruta estática que apunta directamente a la interfaz virtual del túnel VPN (no a una IP de next-hop), característico del modelo Route-Based de FortiOS.
* La creación de políticas de firewall (`firewall policy`) en ambos sentidos (`LAN → VPN` y `VPN → LAN`) — paso obligatorio en FortiGate, ya que sin estas políticas el tráfico es descartado aunque el túnel IPSec esté completamente establecido (Fase 1 y Fase 2 activas).
* La verificación de conectividad extremo a extremo entre las LAN de ambos sitios (`20.25.37.0/25` y `20.25.37.128/25`) exclusivamente a través de la interfaz gráfica (GUI) de FortiOS.

---

## 2. Marco Teórico

### 2.1 ¿Qué es una VPN IPSec Site-to-Site en FortiGate?

Una VPN IPSec Site-to-Site conecta permanentemente dos redes fijas (LANs) a través de un túnel cifrado que atraviesa una red no confiable (Internet o, en este laboratorio, la nube simulada `192.168.1.0/24`). A diferencia de una VPN de acceso remoto (Client-to-Site), aquí **ambos extremos son gateways de red** (los FortiGate), no un usuario individual — ningún host final necesita instalar cliente VPN ni autenticarse con usuario/contraseña; el túnel es transparente para los hosts de cada LAN.

En FortiOS, la VPN IPSec se construye en dos bloques de configuración independientes que se enlazan entre sí:

* **`phase1-interface`**: negocia el canal IKE (ISAKMP) — autenticación del peer remoto (PSK), algoritmo de cifrado/hash, grupo Diffie-Hellman, y crea la interfaz de túnel virtual.
* **`phase2-interface`**: negocia las IPSec SA que protegen el tráfico real, y define qué subredes (`src-subnet` / `dst-subnet`) viajan cifradas por ese túnel.

### 2.2 Route-Based vs. Policy-Based VPN

FortiOS soporta dos modelos de VPN IPSec:

| Característica | Policy-Based (legacy) | Route-Based (este lab) |
|---|---|---|
| **Interfaz de túnel** | No crea interfaz — el túnel se referencia solo en la política de firewall | Crea una interfaz virtual dedicada (ej. `VPN-a-FGT2`) visible en `Network → Interfaces` |
| **Cómo se enruta el tráfico** | Una política de firewall con `action: ipsec` decide qué tráfico entra al túnel | Una ruta estática (`config router static`) apunta el tráfico hacia la interfaz del túnel |
| **Múltiples subredes por túnel** | Requiere una política por cada par de subredes | Una sola ruta estática adicional por cada subred nueva — más escalable |
| **Comando clave** | `set net-device enable` (comportamiento por defecto) | `set net-device disable` en el `phase1-interface` |
| **Recomendación de Fortinet** | Deprecado en builds recientes de FortiOS | Modelo recomendado actualmente |

Este laboratorio usa **Route-Based**, identificado por `set net-device disable` en el `phase1-interface` de ambos FortiGate — esto crea la interfaz virtual `VPN-a-FGT2` / `VPN-a-FGT1` que después se referencia directamente en `config router static` y en `srcintf` / `dstintf` de las políticas de firewall, exactamente como si fuera una interfaz física más.

### 2.3 Arquitectura Phase1-interface / Phase2-interface

```
FortiGate1 (Sitio A)                            FortiGate2 (Sitio B)
20.25.37.128/25                                 20.25.37.0/25
────────────────────────────────────────────────────────────────
  │                                                           │
  │  ┌─────────────────────────────────────────────────────┐  │
  │  │        IPSec ESP (cifrado — DES/SHA1, IKEv1)        │  │
  │  │  ┌───────────────────────────────────────────────┐  │  │
  │  │  │     Interfaz virtual VPN-a-FGT2 / VPN-a-FGT1  │  │  │
  │  │  │     (phase1-interface — net-device disable)   │  │  │
  │  │  │  ┌──────────────────────────────────────────┐ │  │  │
  │  │  │  │   Selectores Fase 2 (phase2-interface)   │ │  │  │
  │  │  │  │src: 20.25.37.128/25 ←→ dst: 20.25.37.0/25│ │  │  │
  │  │  │  └──────────────────────────────────────────┘ │  │  │
  │  │  └───────────────────────────────────────────────┘  │  │
  │  └─────────────────────────────────────────────────────┘  │
  │                                                           │

Puertos involucrados:
  UDP 500  → IKE (negociación Fase 1 y Fase 2)
  UDP 4500 → NAT-T (si hay NAT entre los FortiGate)
  ESP (protocolo IP 50) → tráfico cifrado de la Fase 2
```

### 2.4 Flujo de establecimiento de la conexión

| Paso | Bloque de configuración | Descripción |
|---|---|---|
| 1 | `phase1-interface` | Negocia IKE Fase 1 — autentica ambos FortiGate con la PSK `MiClaveVPN123`, crea la interfaz virtual del túnel |
| 2 | `phase2-interface` | Negocia la IPSec SA — define qué tráfico (selectores `src-subnet`/`dst-subnet`) se cifra |
| 3 | `router static` | Instala la ruta hacia la LAN remota apuntando a la interfaz virtual del túnel (no a una IP) |
| 4 | `firewall policy` | Permite explícitamente el tráfico en ambos sentidos: `LAN → VPN` y `VPN → LAN` |
| 5 | Datos | El tráfico entre ambas LAN viaja cifrado por IPSec a través de la interfaz virtual del túnel |

A diferencia de L2TP/IPSec (donde IPSec protege un túnel L2TP que a su vez autentica un usuario final), aquí **no existe autenticación de usuario** — la confianza se basa exclusivamente en que ambos FortiGate conozcan la PSK correcta y su IP pública mutua (`remote-gw`).

### 2.5 Site-to-Site vs. Client-to-Site

| Característica | Site-to-Site (este lab) | Client-to-Site (L2TP/IPSec) |
|---|---|---|
| **Extremos** | Dos gateways/firewalls fijos | Un usuario individual + un gateway |
| **IP de las LAN** | Fijas en ambos extremos, sin pool dinámico | Dinámica — asignada por un pool en el servidor |
| **Autenticación** | Solo autenticación de equipo (PSK entre FortiGate) | Usuario/contraseña (MS-CHAPv2) + autenticación de túnel |
| **Transparencia para el host final** | Total — PC1 y PC2 no saben que existe una VPN | Ninguna — el usuario debe iniciar la conexión VPN manualmente |
| **Modelo de enrutamiento** | Rutas estáticas hacia la interfaz del túnel | Pool de IPs asignado dinámicamente al cliente |
| **Escalabilidad** | Un túnel fijo por cada par de sitios | Un servidor atiende N usuarios simultáneos |

### 2.6 Parámetros del laboratorio

| Parámetro | Valor | Descripción |
|---|---|---|
| **Versión IKE** | IKEv1 | Negociación de Fase 1 |
| **Modo de VPN** | Route-Based (`net-device disable`) | Crea interfaz virtual de túnel |
| **Propuesta Fase 1 y Fase 2** | `des-sha1` | Cifrado DES, hash SHA-1 |
| **Grupo DH** | Grupo 2 (1024-bit) | Intercambio de clave en Fase 1 |
| **Tipo de peer** | `any` | No exige un Peer ID / certificado específico, solo la PSK |
| **Autenticación PSK** | `MiClaveVPN123` | Clave precompartida idéntica en ambos FortiGate |
| **Selectores Fase 2 (FGT1)** | src `20.25.37.128/25` → dst `20.25.37.0/25` | Tráfico protegido desde el punto de vista de FGT1 |
| **Selectores Fase 2 (FGT2)** | src `20.25.37.0/25` → dst `20.25.37.128/25` | Tráfico protegido desde el punto de vista de FGT2 (espejo del anterior) |
| **Interfaz de túnel FGT1** | `VPN-a-FGT2` | Nombre de la interfaz virtual creada por el `phase1-interface` |
| **Interfaz de túnel FGT2** | `VPN-a-FGT1` | Nombre de la interfaz virtual creada por el `phase1-interface` |
| **Logging** | `log memory setting` habilitado | Registro de tráfico para verificación en `Forward Traffic` |

> **Nota de seguridad:** Este laboratorio usa DES-SHA1 y Grupo DH 2 únicamente con fines didácticos para facilitar la comparación con los conceptos clásicos de IPSec IKEv1. En producción se recomienda migrar a AES-256/SHA-256 con Grupo DH 14 o superior — ver sección [7. Consideraciones de Seguridad](#7-consideraciones-de-seguridad).

---

## 3. Documentación de la Red

### 3.1 Topología

El laboratorio implementa una VPN Site-to-Site entre dos redes LAN separadas por una nube que simula el ISP/Internet (`192.168.1.0/24`). Cada FortiGate actúa como gateway de su propia LAN y como extremo del túnel IPSec hacia el FortiGate remoto.

```
                                    [ NET / ISP ]
                                    192.168.1.0/24
                                          │
                  ┌───────────────────────┴───────────────────────┐
                  │ port1: 192.168.1.10/24              port1: 192.168.1.20/24
          ┌───────┴───────┐                             ┌───────┴───────┐
          │  FortiGate1   │═══════════════════════════════│  FortiGate2   │
          │   Sitio A     │   Túnel IPSec (IKEv1, DES-SHA1)│   Sitio B     │
          │ VPN-a-FGT2    │◄──────────── PSK ─────────────►│ VPN-a-FGT1    │
          └───────┬───────┘                             └───────┬───────┘
                  │ port2: 20.25.37.129/25              port2: 20.25.37.1/25
                  │                                              │
          ┌───────┴───────┐                             ┌───────┴───────┐
          │      SW1      │                             │      SW2      │
          └───────┬───────┘                             └───────┬───────┘
                  │ e0/1                                        │ e0/2
                  │ eth1                                        │ eth0
          ┌───────┴───────┐                             ┌───────┴───────┐
          │      PC1      │                             │      PC2      │
          │ 20.25.37.130  │                             │  20.25.37.2   │
          └───────────────┘                             └───────────────┘
           20.25.37.128/25                                20.25.37.0/25

  ══════════════════════════════════════════════════════════════════
  Flujo de la conexión IPSec Site-to-Site:
    1. PC1 (20.25.37.130) hace ping/traceroute hacia PC2 (20.25.37.2).
    2. FortiGate1 recibe el paquete en port2 y consulta su tabla de
       rutas: la ruta hacia 20.25.37.0/25 apunta a la interfaz
       virtual VPN-a-FGT2.
    3. FortiGate1 negocia (o reutiliza, si ya está activa) la Fase 1
       y Fase 2 IPSec con FortiGate2 usando la PSK MiClaveVPN123.
    4. El paquete se cifra (ESP, DES-SHA1) y viaja por la nube NET
       hacia 192.168.1.20 (port1 de FortiGate2).
    5. FortiGate2 descifra el paquete, lo entrega por port2 hacia
       la LAN de Sitio B, y la política "VPN-a-LAN" lo permite pasar.
    6. PC2 recibe el paquete. El camino de regreso sigue el mismo
       proceso en sentido inverso (rutas y políticas espejo).
  ══════════════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Gateway | Rol |
|---|---|---|---|---|---|---|
| **NET** | Cloud/Switch simulado | — | 192.168.1.0/24 | /24 | — | Simulación de Internet/ISP (NBMA) |
| **FortiGate1** | Fortinet FortiGate (VM) | port1 | 192.168.1.10 | /24 | 192.168.1.2 | WAN — extremo VPN Sitio A |
| | | port2 | 20.25.37.129 | /25 | — | Gateway LAN — Sitio A |
| | | VPN-a-FGT2 | Interfaz virtual (túnel) | — | — | Interfaz de túnel Route-Based hacia FGT2 |
| **FortiGate2** | Fortinet FortiGate (VM) | port1 | 192.168.1.20 | /24 | 192.168.1.2 | WAN — extremo VPN Sitio B |
| | | port2 | 20.25.37.1 | /25 | — | Gateway LAN — Sitio B |
| | | VPN-a-FGT1 | Interfaz virtual (túnel) | — | — | Interfaz de túnel Route-Based hacia FGT1 |
| **SW1** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Sitio A |
| **SW2** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Sitio B |
| **PC1** | Host Linux / VPC | eth1 | 20.25.37.130 | /25 | 20.25.37.129 | Host en red interna de FortiGate1 |
| **PC2** | Host Linux / VPC | eth0 | 20.25.37.2 | /25 | 20.25.37.1 | Host en red interna de FortiGate2 |

---

## 4. Scripts de Configuración

### 4.1 FortiGate1 — Sitio A

```fortios
# ══════════════════════════════════════════════════════
# FortiGate1 — Sitio A | VPN Site-to-Site IPSec Route-Based (IKEv1)
# Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
# ══════════════════════════════════════════════════════

# ─── PASO 1: Interfaces físicas ────────────────────────
config system interface
    edit "port1"
        set mode static
        set ip 192.168.1.10 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "port2"
        set mode static
        set ip 20.25.37.129 255.255.255.128
        set allowaccess ping http ssh
        set role lan
    next
end

# ─── PASO 2: Ruta por defecto hacia la nube NET ───────
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 192.168.1.2
        set device "port1"
    next
end

# ══════════════════════════════════════════════════════
# PASO 3: IKEv1 Fase 1 — crea la interfaz virtual del túnel
# ══════════════════════════════════════════════════════
# net-device disable = modo Route-Based: el tráfico se dirige
# al túnel mediante una ruta estática, no mediante una política
# de tipo "ipsec" (Policy-Based).
config vpn ipsec phase1-interface
    edit "VPN-a-FGT2"
        set interface "port1"
        set ike-version 1
        set peertype any
        set net-device disable
        set proposal des-sha1
        set dhgrp 2
        set remote-gw 192.168.1.20
        set psksecret MiClaveVPN123
    next
end

# ══════════════════════════════════════════════════════
# PASO 4: IKEv1 Fase 2 — selectores de tráfico a cifrar
# ══════════════════════════════════════════════════════
config vpn ipsec phase2-interface
    edit "VPN-a-FGT2-p2"
        set phase1name "VPN-a-FGT2"
        set proposal des-sha1
        set src-subnet 20.25.37.128 255.255.255.128
        set dst-subnet 20.25.37.0 255.255.255.128
    next
end

# ══════════════════════════════════════════════════════
# PASO 5: Ruta estática hacia la LAN remota vía el túnel
# ══════════════════════════════════════════════════════
# El "device" es la interfaz virtual del túnel, no una IP
# de next-hop — así funciona el enrutamiento en Route-Based VPN.
config router static
    edit 2
        set dst 20.25.37.0 255.255.255.128
        set device "VPN-a-FGT2"
    next
end

# ══════════════════════════════════════════════════════
# PASO 6: Políticas de firewall — obligatorias en ambos
# sentidos, incluso con el túnel activo
# ══════════════════════════════════════════════════════
config firewall policy
    edit 1
        set name "LAN-a-VPN"
        set srcintf "port2"
        set dstintf "VPN-a-FGT2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 2
        set name "VPN-a-LAN"
        set srcintf "VPN-a-FGT2"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end

# ─── PASO 7: Logging — necesario para verificar en Forward Traffic
config log memory setting
    set status enable
end
```

### 4.2 FortiGate2 — Sitio B

```fortios
# ══════════════════════════════════════════════════════
# FortiGate2 — Sitio B | VPN Site-to-Site IPSec Route-Based (IKEv1)
# Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
# ══════════════════════════════════════════════════════

# ─── PASO 1: Interfaces físicas ────────────────────────
config system interface
    edit "port1"
        set mode static
        set ip 192.168.1.20 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "port2"
        set mode static
        set ip 20.25.37.1 255.255.255.128
        set allowaccess ping http ssh
        set role lan
    next
end

# ─── PASO 2: Ruta por defecto hacia la nube NET ───────
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 192.168.1.2
        set device "port1"
    next
end

# ══════════════════════════════════════════════════════
# PASO 3: IKEv1 Fase 1 — espejo de la configuración de FGT1
# ══════════════════════════════════════════════════════
config vpn ipsec phase1-interface
    edit "VPN-a-FGT1"
        set interface "port1"
        set ike-version 1
        set peertype any
        set net-device disable
        set proposal des-sha1
        set dhgrp 2
        set remote-gw 192.168.1.10
        set psksecret MiClaveVPN123
    next
end

# ══════════════════════════════════════════════════════
# PASO 4: IKEv1 Fase 2 — selectores en sentido inverso
# ══════════════════════════════════════════════════════
config vpn ipsec phase2-interface
    edit "VPN-a-FGT1-p2"
        set phase1name "VPN-a-FGT1"
        set proposal des-sha1
        set src-subnet 20.25.37.0 255.255.255.128
        set dst-subnet 20.25.37.128 255.255.255.128
    next
end

# ══════════════════════════════════════════════════════
# PASO 5: Ruta estática hacia la LAN remota vía el túnel
# ══════════════════════════════════════════════════════
config router static
    edit 2
        set dst 20.25.37.128 255.255.255.128
        set device "VPN-a-FGT1"
    next
end

# ══════════════════════════════════════════════════════
# PASO 6: Políticas de firewall — ambos sentidos
# ══════════════════════════════════════════════════════
config firewall policy
    edit 1
        set name "LAN-a-VPN"
        set srcintf "port2"
        set dstintf "VPN-a-FGT1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 2
        set name "VPN-a-LAN"
        set srcintf "VPN-a-FGT1"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end

# ─── PASO 7: Logging ────────────────────────────────────
config log memory setting
    set status enable
end
```

### 4.3 Configuración de PCs (Hosts)

**PC1 — LAN de Sitio A (`20.25.37.128/25`)**

```bash
ip 20.25.37.130 255.255.255.128 20.25.37.129
```

**PC2 — LAN de Sitio B (`20.25.37.0/25`)**

```bash
ip 20.25.37.2 255.255.255.128 20.25.37.1
```

> **Nota:** Si los hosts son VPCs de PNETLab se usa el comando `ip` como se muestra. Si son máquinas Linux se configura con `ip addr add` e `ip route add default via`.

---

## 5. Verificación de la Conexión (GUI)

Toda la verificación de este laboratorio se realiza desde la **interfaz gráfica (GUI)** de FortiOS, sin usar la CLI.

### 5.1 VPN → IPsec Tunnels (o Monitor → IPsec Monitor)

En esta sección se visualiza el túnel `VPN-a-FGT2` (o `VPN-a-FGT1` desde el otro extremo) con:

* Estado **Up** en verde — confirma que la Fase 1 y la Fase 2 están negociadas y activas.
* Contadores de **bytes/paquetes enviados y recibidos** que suben cuando hay tráfico cruzando el túnel.

> Esta es la prueba más directa de que el túnel IPSec está funcionando a nivel criptográfico, independientemente de si el tráfico de datos está siendo permitido por las políticas de firewall.

### 5.2 Log & Report → Forward Traffic

Después de generar tráfico (ping o `tracert` desde PC1 hacia PC2), revisar aquí. Se deben observar entradas con:

* `srcintf: port2` → `dstintf: VPN-a-FGT2` (o viceversa, según la dirección del tráfico).
* IP origen y destino correctas (`20.25.37.130` ↔ `20.25.37.2`).
* Acción **Accept**.

> Esto confirma que las políticas de firewall (`LAN-a-VPN` / `VPN-a-LAN`) están dejando pasar el tráfico específico por el túnel — no solo que el túnel está "up", sino que el tráfico realmente lo está cruzando.

### 5.3 Network → Static Routes

Confirma visualmente que la ruta hacia la LAN remota (`20.25.37.0/25` desde FGT1, o `20.25.37.128/25` desde FGT2) apunta al dispositivo/interfaz VPN correspondiente (`VPN-a-FGT2` o `VPN-a-FGT1`) y aparece como **activa**.

### 5.4 Dashboard → Network (widget de interfaces/VPN)

Algunos builds de FortiOS incluyen un widget en el Dashboard que resume el estado de los túneles VPN activos — útil como vista rápida adicional para confirmar de un vistazo que el túnel sigue `Up` sin tener que navegar a `VPN → IPsec Tunnels`.

### 5.5 Tabla de comprobaciones GUI

| Sección GUI | Dónde | Qué confirma |
|---|---|---|
| `VPN → IPsec Tunnels` / `Monitor → IPsec Monitor` | Ambos FortiGate | Túnel `Up`, contadores de bytes/paquetes subiendo — Fase 1 y Fase 2 activas |
| `Log & Report → Forward Traffic` | Ambos FortiGate | Tráfico real cruzando el túnel con acción `Accept` — políticas de firewall funcionando |
| `Network → Static Routes` | Ambos FortiGate | Ruta hacia la LAN remota apuntando a la interfaz de túnel correcta y activa |
| `Dashboard → Network` (widget VPN) | Ambos FortiGate | Vista rápida del estado general de los túneles activos |
| `ping` / `traceroute` desde PC1 hacia PC2 | Host final | Conectividad extremo a extremo real entre ambas LAN |

---

## 6. Capturas de Pantalla

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, ambos FortiGate, switches y PCs encendidos. |
| 2 | [`02_fgt1_phase1_phase2.png`](screenshots/02_fgt1_phase1_phase2.png) | GUI de FortiGate1 → `VPN → IPsec Tunnels → VPN-a-FGT2` mostrando la configuración de Fase 1 y Fase 2 (proposal `des-sha1`, remote-gw `192.168.1.20`). |
| 3 | [`03_fgt1_static_route.png`](screenshots/03_fgt1_static_route.png) | GUI de FortiGate1 → `Network → Static Routes` mostrando la ruta hacia `20.25.37.0/25` vía la interfaz `VPN-a-FGT2`. |
| 4 | [`04_fgt1_firewall_policies.png`](screenshots/04_fgt1_firewall_policies.png) | GUI de FortiGate1 → `Policy & Objects → Firewall Policy` mostrando las políticas `LAN-a-VPN` y `VPN-a-LAN`. |
| 5 | [`05_fgt2_phase1_phase2.png`](screenshots/05_fgt2_phase1_phase2.png) | GUI de FortiGate2 → `VPN → IPsec Tunnels → VPN-a-FGT1`, configuración espejo de la de FGT1. |
| 6 | [`06_ipsec_tunnel_up.png`](screenshots/06_ipsec_tunnel_up.png) | `VPN → IPsec Tunnels` (o `Monitor → IPsec Monitor`) mostrando el túnel en estado **Up** con contadores de bytes/paquetes activos. |
| 7 | [`07_forward_traffic_accept.png`](screenshots/07_forward_traffic_accept.png) | `Log & Report → Forward Traffic` mostrando el tráfico de PC1 hacia PC2 cruzando el túnel con acción `Accept`. |
| 8 | [`08_ping_pc1_a_pc2.png`](screenshots/08_ping_pc1_a_pc2.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC2 (`20.25.37.2`) a través del túnel Site-to-Site. |
| 9 | [`09_traceroute_pc1_a_pc2.png`](screenshots/09_traceroute_pc1_a_pc2.png) | Traceroute desde PC1 hacia PC2 mostrando el camino a través del túnel VPN. |
| 10 | [`10_dashboard_vpn_widget.png`](screenshots/10_dashboard_vpn_widget.png) | Widget del Dashboard mostrando el resumen de túneles VPN activos. |

---

## 7. Consideraciones de Seguridad

### 7.1 Debilidades de esta configuración

* **DES (proposal `des-sha1`):** DES es un algoritmo de cifrado de 56 bits, considerado **completamente inseguro** desde hace más de dos décadas — es factible romperlo por fuerza bruta con hardware moderno. Se usa aquí exclusivamente con fines didácticos para trabajar con la sintaxis clásica de IKEv1.
* **SHA-1 como hash de integridad:** Vulnerable a ataques de colisión conocidos (SHAttered, 2017). No debe usarse en entornos de producción.
* **Grupo DH 2 (1024-bit):** Considerado insuficiente desde 2015. El mínimo recomendado hoy es Grupo 14 (2048-bit) o superior (19/20/21 con curvas elípticas).
* **`peertype any`:** El FortiGate acepta la negociación de cualquier peer que presente la PSK correcta, sin validar un Peer ID específico — cualquier dispositivo que conozca `MiClaveVPN123` y alcance el puerto UDP 500 puede intentar iniciar la Fase 1.
* **PSK compartida en texto plano en la configuración:** En producción, las claves precompartidas deben rotarse periódicamente y, preferiblemente, sustituirse por certificados digitales (PKI).

### 7.2 Migración recomendada para producción

```fortios
# Fase 1 endurecida — IKEv2, AES-256, SHA-256, DH Grupo 14
config vpn ipsec phase1-interface
    edit "VPN-a-FGT2"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal aes256-sha256
        set dhgrp 14
        set remote-gw 192.168.1.20
        set psksecret <clave-larga-y-rotada-periodicamente>
    next
end

# Fase 2 endurecida — AES-256/SHA-256 + Perfect Forward Secrecy
config vpn ipsec phase2-interface
    edit "VPN-a-FGT2-p2"
        set phase1name "VPN-a-FGT2"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set src-subnet 20.25.37.128 255.255.255.128
        set dst-subnet 20.25.37.0 255.255.255.128
    next
end
```

* **IKEv2** en lugar de IKEv1 — negociación más simple, mejor resistencia a DoS y soporte nativo de EAP si en el futuro se requiere autenticación adicional.
* **PFS (Perfect Forward Secrecy)** habilitado — garantiza que comprometer una clave de sesión no compromete tráfico cifrado previamente capturado.
* Restringir el acceso `ping https ssh` de `port1` únicamente a IPs de administración conocidas, en lugar de dejarlo abierto a toda la interfaz WAN.

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](https://youtu.be/62Xyf6umglE)**

**Duración:** 7:47

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Configuración de FortiGate1 mostrando los bloques `phase1-interface`, `phase2-interface`, ruta estática y políticas de firewall.
* ✅ Configuración de FortiGate2 (espejo de FortiGate1).
* ✅ `VPN → IPsec Tunnels` mostrando el túnel en estado `Up` con contadores de tráfico.
* ✅ `Log & Report → Forward Traffic` mostrando el tráfico de PC1 a PC2 con acción `Accept`.
* ✅ `Network → Static Routes` mostrando la ruta activa hacia la LAN remota vía la interfaz VPN.
* ✅ Ping y traceroute exitosos desde PC1 hacia PC2 con el túnel activo.

---

## 9. Referencias

* Fortinet Inc. (2024). *FortiOS Administration Guide — IPsec VPN*.
* Fortinet Inc. (2024). *FortiOS Handbook — Route-Based vs. Policy-Based IPsec VPN*.
* Harkins, D. & Carrel, D. (1998). *RFC 2409 — The Internet Key Exchange (IKEv1)*. IETF.
* Kent, S. & Seo, K. (2005). *RFC 4301 — Security Architecture for the Internet Protocol*. IETF.
* Frankel, S. et al. (2005). *NIST SP 800-77 — Guide to IPsec VPNs*. National Institute of Standards and Technology.
* Doraswamy, N. & Harkins, D. (2003). *IPSec: The New Security Standard for the Internet, Intranets, and Virtual Private Networks (2nd Ed.)*. Prentice Hall.
