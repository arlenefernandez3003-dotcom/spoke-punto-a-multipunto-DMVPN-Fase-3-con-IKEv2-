# 🔐 DMVPN Fase 2 — Hub and Spoke con IPSec IKEv1 y Enrutamiento Dinámico

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![DMVPN](https://img.shields.io/badge/Tecnología-DMVPN%20Fase%202-blue?style=for-the-badge&logo=cisco)
![IKEv1](https://img.shields.io/badge/Protocolo-IPSec%20IKEv1-orange?style=for-the-badge)
![EIGRP](https://img.shields.io/badge/Enrutamiento-EIGRP-brightgreen?style=for-the-badge)
![Platform](https://img.shields.io/badge/Plataforma-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Verificación del Túnel](#6-verificación-del-túnel)
7. [Capturas de Pantalla](#7-capturas-de-pantalla)
8. [Video Demostrativo](#8-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una **VPN Hub-and-Spoke punto a multipunto DMVPN Fase 2** utilizando **IPSec IKEv1** con **enrutamiento dinámico EIGRP** en routers Cisco IOS dentro de PNetLab. La práctica cubre:

- Configuración del **Hub (R1)** como nodo central DMVPN con interfaz `mGRE` (multipoint GRE) y servidor **NHRP** que resuelve las IPs WAN de los Spokes.
- Configuración de **Spoke1 (R2)** y **Spoke2 (R3)** con interfaz `mGRE` registrándose en el Hub como NHS (Next-Hop Server).
- Protección del tráfico con **IPSec IKEv1** usando Crypto Profile aplicado sobre `Tunnel0` con `tunnel protection`.
- **DMVPN Fase 2**: los Spokes se comunican directamente entre sí (Spoke-to-Spoke) sin pasar por el Hub, una vez que NHRP resuelve la IP NBMA del Spoke destino.
- Propagación de redes LAN mediante **EIGRP** sobre el túnel DMVPN.

### DMVPN Fase 1 vs Fase 2

| Característica | Fase 1 | Fase 2 (este lab) |
|---|---|---|
| Tráfico Spoke→Spoke | Siempre pasa por Hub | Túnel directo Spoke↔Spoke vía NHRP |
| Next-Hop en Hub | Se reemplaza (Hub como NH) | **Se preserva** (Spoke original como NH) |
| `no ip next-hop-self` | No aplica | **Obligatorio en el Hub** |
| Escalabilidad | Menor | Mayor — menos carga en el Hub |

---

## 2. Topología

```
                                  [ Neto ]
                               192.168.19.0/24
                              (Cloud/NAT: 192.168.19.2)
                          ┌──────────┼──────────┐
               e0/0       │     e0/0 │           │ e0/0
          192.168.19.5    │  192.168.19.6        │ 192.168.19.30
          ┌───────────────┤  ┌───────────────┐  ┌┴───────────────┐
          │   R1 (HUB)   │  │  R2 (Spoke1)  │  │  R3 (Spoke2)   │
          └───────┬───────┘  └──────┬────────┘  └──────┬─────────┘
               e0/1                e0/1                e0/1
        202.50.73.193/26    202.50.73.129/26     202.50.73.65/26
               │                   │                   │
          ┌────┴─────┐        ┌────┴─────┐        ┌────┴─────┐
          │   SW1    │        │   SW2    │        │   SW3    │
          │  (e0/0)  │        │  (e0/0)  │        │  (e0/0)  │
          └────┬─────┘        └────┬─────┘        └────┬─────┘
             e0/1                e0/1                 e0/1
              │                   │                    │
          ┌───┴──────┐        ┌───┴──────┐        ┌───┴──────┐
          │   VPC1   │        │   VPC2   │        │   VPC3   │
          │.194/26   │        │.130/26   │        │ .66/26   │
          └──────────┘        └──────────┘        └──────────┘

  Red Tunnel DMVPN (mGRE): 10.0.0.0/24
  Hub  R1 → Tunnel0: 10.0.0.1
  Spoke1 R2 → Tunnel0: 10.0.0.2
  Spoke2 R3 → Tunnel0: 10.0.0.3

  Fase 2 — túnel directo entre Spokes:
  R2 ◄══════════════════════════════════► R3
     (NHRP resuelve IP NBMA del destino)
```

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo     | Interfaz    | Dirección IP       | Máscara | Gateway        | Rol                         |
|-----------------|-------------|--------------------|---------|----------------|-----------------------------|
| Neto (Cloud)    | —           | 192.168.19.2       | /24     | —              | Gateway Cloud NAT (PNetLab) |
| **R1 (Hub)**    | **e0/0**    | **192.168.19.5**   | **/24** | 192.168.19.2   | WAN Hub → Neto              |
| **R1 (Hub)**    | **e0/1**    | **202.50.73.193**  | **/26** | —              | Gateway LAN Hub → SW1       |
| **R1 (Hub)**    | **Tunnel0** | **10.0.0.1**       | **/24** | —              | mGRE DMVPN Hub              |
| **R2 (Spoke1)** | **e0/0**    | **192.168.19.6**   | **/24** | 192.168.19.2   | WAN Spoke1 → Neto           |
| **R2 (Spoke1)** | **e0/1**    | **202.50.73.129**  | **/26** | —              | Gateway LAN Spoke1 → SW2    |
| **R2 (Spoke1)** | **Tunnel0** | **10.0.0.2**       | **/24** | —              | mGRE DMVPN Spoke1           |
| **R3 (Spoke2)** | **e0/0**    | **192.168.19.30**  | **/24** | 192.168.19.2   | WAN Spoke2 → Neto           |
| **R3 (Spoke2)** | **e0/1**    | **202.50.73.65**   | **/26** | —              | Gateway LAN Spoke2 → SW3    |
| **R3 (Spoke2)** | **Tunnel0** | **10.0.0.3**       | **/24** | —              | mGRE DMVPN Spoke2           |
| SW1             | e0/0        | —                  | —       | —              | Uplink → R1 e0/1            |
| SW1             | e0/1        | —                  | —       | —              | Downlink → VPC1             |
| SW2             | e0/0        | —                  | —       | —              | Uplink → R2 e0/1            |
| SW2             | e0/1        | —                  | —       | —              | Downlink → VPC2             |
| SW3             | e0/0        | —                  | —       | —              | Uplink → R3 e0/1            |
| SW3             | e0/1        | —                  | —       | —              | Downlink → VPC3             |
| VPC1            | eth0        | 202.50.73.194      | /26     | 202.50.73.193  | Host LAN Hub                |
| VPC2            | eth0        | 202.50.73.130      | /26     | 202.50.73.129  | Host LAN Spoke1             |
| VPC3            | eth0        | 202.50.73.66       | /26     | 202.50.73.65   | Host LAN Spoke2             |

### Tabla de Subredes

| Subred             | Rango Utilizable              | Broadcast       | Uso                     |
|--------------------|-------------------------------|-----------------|-------------------------|
| `192.168.19.0/24`  | 192.168.19.1 – 192.168.19.254 | 192.168.19.255  | Segmento WAN/Neto       |
| `202.50.73.64/26`  | 202.50.73.65 – 202.50.73.126  | 202.50.73.127   | LAN Spoke2 (R3)         |
| `202.50.73.128/26` | 202.50.73.129 – 202.50.73.190 | 202.50.73.191   | LAN Spoke1 (R2)         |
| `202.50.73.192/26` | 202.50.73.193 – 202.50.73.254 | 202.50.73.255   | LAN Hub (R1)            |
| `10.0.0.0/24`      | 10.0.0.1 – 10.0.0.254         | 10.0.0.255      | Red Tunnel DMVPN (mGRE) |

---

## 4. Parámetros Configurados

### IPSec IKEv1 — ISAKMP Policy (Fase 1)

| Parámetro          | Valor               | Descripción                                          |
|--------------------|---------------------|------------------------------------------------------|
| Número de política | `10`                | Prioridad de la política ISAKMP                      |
| Cifrado            | AES-256             | Cifrado simétrico del canal IKE                      |
| Hash / Integridad  | SHA-256             | Verificación de integridad                           |
| Autenticación      | Pre-Shared Key      | Clave compartida entre Hub y Spokes                  |
| Grupo DH           | Group 14 (2048-bit) | Intercambio Diffie-Hellman                           |
| Lifetime SA        | 86400 s (24 h)      | Duración del canal IKE antes de renegociar           |
| Pre-Shared Key     | `ITLA2025Arlene`    | Configurada con wildcard `0.0.0.0` en todos         |

> La PSK se configura con `address 0.0.0.0` (wildcard) para que el Hub acepte cualquier Spoke dinámicamente y los Spokes puedan negociar entre sí en Fase 2.

### IPSec IKEv1 — Transform Set y Crypto Profile (Fase 2)

| Parámetro      | Valor              | Descripción                                         |
|----------------|--------------------|-----------------------------------------------------|
| Transform Set  | `TS_AES256_SHA256` | ESP-AES-256 + ESP-SHA256-HMAC                       |
| Modo           | Transport          | mGRE ya encapsula — IPSec usa transport             |
| Crypto Profile | `DMVPN_PROFILE`    | Aplicado en `Tunnel0` con `tunnel protection`       |
| Lifetime SA    | 3600 s (1 h)       | Duración antes de renegociar                        |

### DMVPN — NHRP y mGRE

| Parámetro           | Hub R1          | Spoke1 R2       | Spoke2 R3       |
|---------------------|-----------------|-----------------|-----------------|
| Tunnel IP           | 10.0.0.1/24     | 10.0.0.2/24     | 10.0.0.3/24     |
| Tunnel Mode         | gre multipoint  | gre multipoint  | gre multipoint  |
| Tunnel Key          | 1000            | 1000            | 1000            |
| NHRP Network-ID     | 1               | 1               | 1               |
| NHRP Authentication | `ITLA2025`      | `ITLA2025`      | `ITLA2025`      |
| NHRP NHS            | — (es el NHS)   | 10.0.0.1        | 10.0.0.1        |
| NHRP NHS NBMA       | — (es el NHS)   | 192.168.19.5    | 192.168.19.5    |
| `ip nhrp redirect`  | ✅ Fase 2        | —               | —               |
| `ip nhrp shortcut`  | —               | ✅ Fase 2        | ✅ Fase 2        |

### EIGRP sobre DMVPN

| Parámetro              | Valor | Descripción                                                              |
|------------------------|-------|--------------------------------------------------------------------------|
| AS Number              | `100` | Sistema autónomo compartido por Hub y Spokes                             |
| `no ip split-horizon`  | Hub   | Permite reenviar rutas EIGRP entre Spokes                                |
| `no ip next-hop-self`  | Hub   | **Crítico en Fase 2** — preserva el NH del Spoke para túneles directos   |

---

## 5. Scripts de Configuración

### R1 — Hub

```cisco
! ══════════════════════════════════════════════════════════════
!  R1 — Hub | DMVPN Fase 2 + IPSec IKEv1 + EIGRP
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R1-Hub

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Neto
 ip address 192.168.19.5 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-Hub-hacia-SW1
 ip address 202.50.73.193 255.255.255.192
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── IPSec IKEv1 — Fase 1 ────────────────────────────────────
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! Wildcard: acepta PSK de cualquier Spoke
crypto isakmp key ITLA2025Arlene address 0.0.0.0

! ── IPSec IKEv1 — Fase 2 ────────────────────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile DMVPN_PROFILE
 set transform-set TS_AES256_SHA256
 set security-association lifetime seconds 3600

! ── Interfaz Tunnel DMVPN (mGRE) ────────────────────────────
interface Tunnel0
 description DMVPN-Hub
 ip address 10.0.0.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 no ip next-hop-self eigrp 100
 no ip split-horizon eigrp 100
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 1000
 tunnel protection ipsec profile DMVPN_PROFILE
 ip nhrp network-id 1
 ip nhrp authentication ITLA2025
 ip nhrp redirect
 no shutdown

! ── EIGRP ───────────────────────────────────────────────────
router eigrp 100
 network 10.0.0.0 0.0.0.255
 network 202.50.73.192 0.0.0.63
 no auto-summary
```

### R2 — Spoke1

```cisco
! ══════════════════════════════════════════════════════════════
!  R2 — Spoke1 | DMVPN Fase 2 + IPSec IKEv1 + EIGRP
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R2-Spoke1

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Neto
 ip address 192.168.19.6 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-Spoke1-hacia-SW2
 ip address 202.50.73.129 255.255.255.192
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── IPSec IKEv1 — Fase 1 ────────────────────────────────────
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ITLA2025Arlene address 0.0.0.0

! ── IPSec IKEv1 — Fase 2 ────────────────────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile DMVPN_PROFILE
 set transform-set TS_AES256_SHA256
 set security-association lifetime seconds 3600

! ── Interfaz Tunnel DMVPN (mGRE) ────────────────────────────
interface Tunnel0
 description DMVPN-Spoke1
 ip address 10.0.0.2 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 1000
 tunnel protection ipsec profile DMVPN_PROFILE
 ip nhrp network-id 1
 ip nhrp authentication ITLA2025
 ip nhrp nhs 10.0.0.1
 ip nhrp map 10.0.0.1 192.168.19.5
 ip nhrp map multicast 192.168.19.5
 ip nhrp shortcut
 no shutdown

! ── EIGRP ───────────────────────────────────────────────────
router eigrp 100
 network 10.0.0.0 0.0.0.255
 network 202.50.73.128 0.0.0.63
 no auto-summary
```

### R3 — Spoke2

```cisco
! ══════════════════════════════════════════════════════════════
!  R3 — Spoke2 | DMVPN Fase 2 + IPSec IKEv1 + EIGRP
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R3-Spoke2

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Neto
 ip address 192.168.19.30 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-Spoke2-hacia-SW3
 ip address 202.50.73.65 255.255.255.192
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── IPSec IKEv1 — Fase 1 ────────────────────────────────────
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ITLA2025Arlene address 0.0.0.0

! ── IPSec IKEv1 — Fase 2 ────────────────────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile DMVPN_PROFILE
 set transform-set TS_AES256_SHA256
 set security-association lifetime seconds 3600

! ── Interfaz Tunnel DMVPN (mGRE) ────────────────────────────
interface Tunnel0
 description DMVPN-Spoke2
 ip address 10.0.0.3 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 1000
 tunnel protection ipsec profile DMVPN_PROFILE
 ip nhrp network-id 1
 ip nhrp authentication ITLA2025
 ip nhrp nhs 10.0.0.1
 ip nhrp map 10.0.0.1 192.168.19.5
 ip nhrp map multicast 192.168.19.5
 ip nhrp shortcut
 no shutdown

! ── EIGRP ───────────────────────────────────────────────────
router eigrp 100
 network 10.0.0.0 0.0.0.255
 network 202.50.73.64 0.0.0.63
 no auto-summary
```

### VPCs en PNetLab

```bash
# VPC1 — LAN Hub (R1)
ip 202.50.73.194 255.255.255.192 202.50.73.193

# VPC2 — LAN Spoke1 (R2)
ip 202.50.73.130 255.255.255.192 202.50.73.129

# VPC3 — LAN Spoke2 (R3)
ip 202.50.73.66 255.255.255.192 202.50.73.65
```

---

## 6. Verificación del Túnel

### Entradas NHRP en el Hub

```cisco
R1-Hub# show ip nhrp
```

Salida esperada:

```
10.0.0.2/32 via 10.0.0.2
   Tunnel0 created 00:02:10, expire 01:57:50
   Type: dynamic, Flags: registered used nhop
   NBMA address: 192.168.19.6

10.0.0.3/32 via 10.0.0.3
   Tunnel0 created 00:01:45, expire 01:58:15
   Type: dynamic, Flags: registered used nhop
   NBMA address: 192.168.19.30
```

> Ambos Spokes deben aparecer como entradas `dynamic` con su IP NBMA real registrada.

---

### Vecinos EIGRP activos

```cisco
R1-Hub# show ip eigrp neighbors
```

Salida esperada:

```
EIGRP-IPv4 Neighbors for AS(100)
H   Address    Interface   Hold  Uptime   SRTT  RTO  Q  Seq
0   10.0.0.2   Tu0          14   00:03:20  10   100  0   8
1   10.0.0.3   Tu0          13   00:02:55  12   100  0   6
```

---

### Tabla de rutas en Spoke1 — verificar Fase 2

```cisco
R2-Spoke1# show ip route eigrp
```

Salida esperada:

```
D    202.50.73.64/26  [90/27008000] via 10.0.0.3, 00:01:30, Tunnel0
D    202.50.73.192/26 [90/27008000] via 10.0.0.1, 00:03:00, Tunnel0
```

> **Clave de Fase 2:** la ruta hacia la LAN de Spoke2 muestra `via 10.0.0.3` (Spoke2 directamente), **no** `via 10.0.0.1` (Hub). Si aparece el Hub como next-hop, la Fase 2 no está activa.

---

### NHRP dinámico Spoke-to-Spoke

Después de un ping entre VPC2 y VPC3, en R2 debe aparecer:

```cisco
R2-Spoke1# show ip nhrp
```

```
10.0.0.3/32 via 10.0.0.3
   Tunnel0 created 00:00:05, expire 00:01:55
   Type: dynamic, Flags: router nhop rib
   NBMA address: 192.168.19.30
```

> Esta entrada dinámica confirma que R2 resolvió la IP NBMA de R3 y estableció el túnel directo Spoke-to-Spoke.

---

### Estado IPSec IKEv1

```cisco
R1-Hub# show crypto isakmp sa
```

Salida esperada:

```
dst              src              state     conn-id  status
192.168.19.6     192.168.19.5     QM_IDLE   1001     ACTIVE
192.168.19.30    192.168.19.5     QM_IDLE   1002     ACTIVE
```

---

### Conectividad entre sitios

```bash
# VPC2 → VPC3 (Spoke-to-Spoke directo — Fase 2)
VPC2> ping 202.50.73.66

# VPC1 → VPC2 y VPC3 (Hub → Spokes)
VPC1> ping 202.50.73.130
VPC1> ping 202.50.73.66
```

---

### Tabla de Comandos de Verificación

| Comando | Qué verifica |
|---|---|
| `show ip nhrp` | Entradas NHRP con IPs NBMA de los Spokes registradas. |
| `show ip nhrp detail` | Detalle de cada entrada NHRP con flags y tipo. |
| `show dmvpn` | Resumen del estado de todos los peers DMVPN. |
| `show ip eigrp neighbors` | Vecinos EIGRP activos sobre el túnel. |
| `show ip route eigrp` | NH debe ser el Spoke directamente (Fase 2), no el Hub. |
| `show crypto isakmp sa` | SAs IKEv1 activas entre Hub y cada Spoke. |
| `show crypto ipsec sa` | Contadores de paquetes cifrados por sesión. |

---



---

## 7. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/1.png) | Topología en PNetLab con nombre **Arlene Fernández Herrera**, matrícula **2025-0730**, Hub R1, Spoke1 R2 y Spoke2 R3 visibles. |
| 2 | [Config Hub – Tunnel DMVPN](evidencias/2.png) | Consola R1: `Tunnel0` mGRE con NHRP, `no next-hop-self eigrp 100` y `tunnel protection` configurados. |
| 3 | [NHRP + EIGRP activos](evidencias/3.png) | Salida de `show ip nhrp` con ambos Spokes registrados y `show ip eigrp neighbors` con vecinos activos. |
| 4 | [Ping Spoke-to-Spoke](evidencias/4.png) | Ping exitoso de VPC2 (`202.50.73.130`) a VPC3 (`202.50.73.66`) confirmando túnel directo Fase 2. |

---

## 8. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](#)**



---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab 07: DMVPN Fase 2 IKEv1 con EIGRP*

</div>
