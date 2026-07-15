# 🔒 VPN Lab — Seguridad de Redes

> *Ocho maneras de decirle a dos redes que confíen la una en la otra a través de una red pública que no controlas.*

**Autor:** Miguel Angel Peña Acevedo 

---

## Tabla de contenido

1. [De qué va este laboratorio](#de-qué-va-este-laboratorio)
2. [Mapa mental de las 8 variantes](#mapa-mental-de-las-8-variantes)
3. [Por qué existen tantas variantes de "lo mismo"](#por-qué-existen-tantas-variantes-de-lo-mismo)
4. [Direccionamiento base](#direccionamiento-base)
5. [Parámetros criptográficos por defecto](#parámetros-criptográficos-por-defecto)
6. [1–3. IPsec Site-to-Site: Policy-based, VTI y GRE (IKEv1/IKEv2)](#1–3-ipsec-site-to-site-policy-based-vti-y-gre-ikev1ikev2)
7. [4–5. DMVPN — Fase 2 (IKEv1) y Fase 3 (IKEv2)](#4–5-dmvpn--fase-2-ikev1-y-fase-3-ikev2)
8. [6. L2TP/IPsec — Client-to-Site](#6-l2tpipsec--client-to-site)
9. [7–8. FortiGate — Site-to-Site y Client-to-Site](#7–8-fortigate--site-to-site-y-client-to-site)
10. [Herramientas usadas](#herramientas-usadas-en-el-laboratorio)
11. [Notas finales](#notas-finales)

---

## De qué va este laboratorio

La pregunta de fondo detrás de cada escenario de este repo es siempre la
misma —**¿cómo hago que dos routers, dos FortiGates, o un router y un laptop
confíen el uno en el otro y cifren su tráfico a través de una red pública que
no controlo?**— pero la respuesta cambia según qué tan grande es la
topología, si el otro extremo tiene una IP fija, y si el tráfico necesita
enrutamiento dinámico o soporte multicast.

Todas las topologías comparten tres reglas de diseño:

- Al menos **2 routers/firewalls como peers**, cada uno con su propia LAN.
- Un **router ISP** intermedio que solo enruta entre subredes públicas
  `/30` — **nunca hace NAT** — para simular Internet de forma realista.
- Verificación explícita de conectividad extremo a extremo (`ping`,
  `traceroute`/`tracert`) una vez que el túnel está arriba.

Ocho variantes, un mismo principio subyacente: **autenticar, negociar
claves, cifrar, enrutar y verificar.**

| # | Escenario | Topología | Versión IKE | Enrutamiento |
|---|---|---|---|---|
| 1 | IPsec Site-to-Site basado en **políticas** | Punto a punto | v1 y v2 | Estático (ACL de tráfico interesante) |
| 2 | IPsec Site-to-Site basado en **enrutamiento (VTI)** | Punto a punto | v1 y v2 | Estático (ruta hacia `Tunnel0`) |
| 3 | IPsec Site-to-Site con **GRE sobre IPsec** | Punto a punto | v1 y v2 | **EIGRP dinámico** |
| 4 | **DMVPN Fase 2** | Hub + 2 Spokes | v1 | EIGRP + túnel spoke-to-spoke directo |
| 5 | **DMVPN Fase 3** | Hub + 2 Spokes | v2 | EIGRP + shortcuts NHRP |
| 6 | **L2TP/IPsec** | Client-to-Site | v1 | PPP + pool de IPs |
| 7 | **FortiGate** Site-to-Site | Punto a punto | v1 (o v2) | Rutas estáticas |
| 8 | **FortiGate** Client-to-Site | Dial-up + strongSwan | v2 | Mode Config + split-tunnel |

## Mapa mental de las 8 variantes

```
                          ¿Cuántos sitios se conectan?
                                     │
                 ┌───────────────────┼────────────────────┐
                 │                   │                    │
            Dos (punto           Un hub y varios      Un servidor y
             a punto)              spokes             clientes móviles
                 │                   │                    │
        ┌────────┼────────┐          │              ┌─────┴─────┐
        │        │        │       DMVPN            L2TP/IPsec   FortiGate
     Política   VTI      GRE     (mGRE+NHRP)       (Cisco IOS)  dial-up
   (crypto map) (Tunnel0) +EIGRP   Fase 2 / 3                  (IKEv2)
                                  IKEv1 / IKEv2

                    Site-to-Site entre FortiGates
                    (route-based, el mismo problema
                     de "dos sitios" resuelto en otro NOS)
```

## Por qué existen tantas variantes de "lo mismo"

No es redundancia — cada una resuelve una restricción distinta:

- **Policy-based** es la forma más simple de decir "cifra esto y solo esto"
  con una ACL asociada a un `crypto map`. Barato de montar, pero cada subred
  nueva obliga a tocar la ACL y el crypto map. Apropiado para dos sitios fijos
  y estables; no escala bien en hub-and-spoke.
- **VTI (route-based)** convierte el túnel en una interfaz real
  (`tunnel mode ipsec ipv4`): se le pueden aplicar rutas, ACL, QoS o NAT sin
  tocar la política criptográfica cada vez que cambia la red. Es el estándar
  que Cisco recomienda para reemplazar crypto maps en despliegues nuevos.
- **GRE sobre IPsec** existe porque un VTI IPsec puro **no transporta
  multicast/broadcast**, y sin eso no hay protocolos de enrutamiento dinámico
  clásicos (EIGRP, OSPF multiárea) funcionando de forma nativa sobre el
  túnel. GRE sí lo transporta, a cambio de mayor complejidad y overhead
  (cabecera GRE + IPsec).
- **DMVPN** lleva la idea de VTI a *N* sitios con un solo perfil de
  configuración por rol (hub o spoke), usando **NHRP** para resolver
  direcciones públicas bajo demanda en vez de declarar cada túnel a mano.
  La Fase 3 además sumariza rutas y usa *shortcuts*, escalando mejor que
  la Fase 2 en redes grandes.
- **L2TP/IPsec** resuelve un problema distinto: el otro extremo no es un
  router con IP fija, es **una persona con un laptop** que puede conectarse
  desde cualquier lugar. Por eso usa PSK comodín (`address 0.0.0.0 0.0.0.0`)
  y asigna una IP dinámica del pool corporativo vía PPP.
- **FortiGate** repite los escenarios site-to-site y client-to-site, pero en
  un sistema operativo de firewall (FortiOS) en vez de IOS, con sus propias
  particularidades de licenciamiento, GUI y modo dial-up con Mode Config.

## Direccionamiento base

Todas las LAN usan el bloque **`7.41.X.0/24`** (por la matrícula `2025-0741`)
y todos los enlaces WAN públicos son `/30` dentro de `200.0.0.0/24`.

| Escenario | LAN(s) | Overlay / Pool |
|---|---|---|
| Policy-based | 7.41.1.0/24 ↔ 7.41.2.0/24 | — |
| VTI | 7.41.3.0/24 ↔ 7.41.4.0/24 | 172.16.0.0/30 |
| GRE sobre IPsec | 7.41.5.0/24 ↔ 7.41.6.0/24 | 172.16.1.0/30 |
| DMVPN Fase 2 | Hub 7.41.10.0/24 · Spoke1 7.41.20.0/24 · Spoke2 7.41.30.0/24 | Tunnel 10.0.0.0/24 |
| DMVPN Fase 3 | Hub 7.41.40.0/24 · Spoke1 7.41.50.0/24 · Spoke2 7.41.60.0/24 | Tunnel 10.0.1.0/24 |
| L2TP/IPsec | Corporativa 7.41.7.0/24 | Pool 7.41.99.10–7.41.99.50 |
| FortiGate S2S | 7.41.8.0/24 ↔ 7.41.9.0/24 | — |
| FortiGate C2S | Corporativa 7.41.11.0/24 | Pool 7.41.99.10–7.41.99.50 |

## Parámetros criptográficos por defecto

A menos que la sección lo indique distinto (por ejemplo, la licencia trial de
FortiOS 7.6.2, que puede rechazar AES/3DES y obliga a bajar a DES), todos los
túneles del laboratorio usan:

```
Cifrado:        AES-256
Integridad:     SHA-256
Grupo DH:       14 (2048 bits)
Autenticación:  PSK (distinta por escenario)
Lifetime:       86400 s
```

---

## 1–3. IPsec Site-to-Site: Policy-based, VTI y GRE (IKEv1/IKEv2)

Tres formas distintas de construir el **mismo** túnel punto a punto entre
`R1` y `R2`, separadas por un router `ISP` sin NAT. Cada variante se
implementó primero con **IKEv1** y luego con **IKEv2**, para comparar la
configuración de Fase 1 en ambos protocolos.

```
   LAN A                                          LAN B
7.41.X.0/24                                    7.41.Y.0/24
     │                                              │
   ┌─R1─┐  200.0.0.1/30      200.0.0.5/30  ┌─R2─┐
   │e0/0│──e0/1────┐    ┌────e0/1──│e0/0│
   └────┘          │    │          └────┘
                  ┌─────┴─┴─────┐
                  │  ISP (sin    │
                  │  NAT, solo   │
                  │  rutas /30)  │
                  └──────────────┘
```

| Variante | Selección de tráfico | Interfaz de túnel | Enrutamiento dinámico |
|---|---|---|---|
| Policy-based | ACL + `crypto map` | ❌ No existe | ❌ No soporta |
| VTI (route-based) | Se enruta hacia `Tunnel0` | ✅ `tunnel mode ipsec ipv4` | ✅ (agregando rutas) |
| GRE sobre IPsec | Se enruta hacia `Tunnel0` | ✅ `tunnel mode gre ip` | ✅ EIGRP nativo (multicast) |

**Fase 1 (IKE) — IKEv1**
```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key cisco123 address <peer>
```

**Fase 1 (IKE) — IKEv2**
```
crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14
crypto ikev2 keyring KR
 peer <nombre>
  address <peer>
  pre-shared-key cisco123
crypto ikev2 profile IKEV2-PROF
 match identity remote address <peer> 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR
```

**Fase 2 (IPsec) — común a los tres**
```
crypto ipsec transform-set TSET esp-aes 256 esp-sha256-hmac
```
- *Policy-based:* `mode tunnel` + `crypto map` con `match address <ACL>`.
- *VTI:* `crypto ipsec profile` anclado con `tunnel protection ipsec profile`.
- *GRE:* igual que VTI, pero `tunnel mode gre ip` + `router eigrp 100`.

**Verificación**
```
show crypto isakmp sa         ! IKEv1: estado QM_IDLE
show crypto ikev2 sa          ! IKEv2: estado READY
show crypto ipsec sa          ! contadores #pkts encaps/decaps subiendo
show crypto session detail    ! UP-ACTIVE
ping <LAN remota> source <LAN local>
```

**Resolución de problemas**
- Fase 1 correcta pero no pasa tráfico → revisar selectores (ACL en
  policy-based; ruta hacia `Tunnel0` en VTI/GRE) y que exista ruta hacia la
  red remota.
- Túnel GRE con MTU/fragmentación → mantener `ip mtu 1400` y
  `ip tcp adjust-mss 1360` en la interfaz de túnel.

---

## 4–5. DMVPN — Fase 2 (IKEv1) y Fase 3 (IKEv2)

DMVPN (*Dynamic Multipoint VPN*) construye una topología **hub-and-spoke
punto a multipunto escalable** con un único perfil de configuración por rol,
combinando cuatro tecnologías:

1. **mGRE** (multipoint GRE): una sola interfaz `Tunnel0` habla con múltiples
   destinos sin declarar cada peer de forma estática.
2. **NHRP**: los spokes se registran contra el *Next-Hop Server* (el hub),
   que resuelve la IP pública de cada destino bajo demanda.
3. **IPsec**: protege el tráfico del túnel mGRE con `tunnel protection`.
4. **EIGRP**: distribuye dinámicamente las LAN de cada sitio sobre el overlay.

```
                    ┌────────┐
                    │  ISP   │  (3 enlaces, sin NAT)
                    └───┬┬┬──┘
            ┌───────────┘│└───────────┐
        ┌───┴───┐    ┌────┴───┐   ┌────┴────┐
        │  HUB  │    │ SPOKE1 │   │ SPOKE2  │
        │NHS/DR │    │        │   │         │
        └───┬───┘    └───┬────┘   └────┬────┘
         LAN Hub      LAN Spoke1    LAN Spoke2
```

| | Fase 2 · IKEv1 | Fase 3 · IKEv2 |
|---|---|---|
| Túneles spoke-to-spoke | Directos, aprendidos preservando next-hop | Directos vía **shortcuts NHRP** bajo demanda |
| Anuncio de rutas en el hub | Rutas específicas por spoke | **Sumario** `7.41.0.0/16` (menos carga en la tabla) |
| Mecanismo clave | `no ip next-hop-self` + `no ip split-horizon` | `ip nhrp redirect` (hub) + `ip nhrp shortcut` (spokes) |
| Fase 1 (IKE) | IKEv1 con PSK comodín | IKEv2 con `peer ANY` |

**Fase 2 vs Fase 3 — la diferencia que importa**

- *Fase 2:* el hub deja de reescribir el next-hop (`no ip next-hop-self`),
  así que cuando `SPOKE1` necesita hablar con `SPOKE2`, ambos conservan el
  next-hop original y negocian un túnel IPsec **directo** entre ellos, sin
  pasar por el hub.
- *Fase 3:* escala mejor. El hub sumariza (`ip summary-address`) y envía
  redirecciones NHRP; los spokes solo reciben un resumen de rutas e instalan
  el atajo (`ip nhrp shortcut`) automáticamente **cuando detectan tráfico**
  hacia el otro spoke — sin configuración manual de next-hop.

**Interfaz de túnel en el hub (Fase 2)**
```
interface Tunnel0
 ip nhrp authentication NHRPkey
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 no ip next-hop-self eigrp 100
 no ip split-horizon eigrp 100
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

**Interfaz de túnel en el hub (Fase 3)**
```
interface Tunnel0
 ip nhrp redirect
 no ip split-horizon eigrp 100
 ip summary-address eigrp 100 7.41.0.0 255.255.0.0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

**Verificación**
```
show dmvpn detail            ! Hub: peers estaticos. Spokes: NHS UP
show ip nhrp
show ip eigrp neighbors
show ip route eigrp          ! spoke aprende la LAN remota

! Probar el tunel directo spoke-to-spoke:
SPOKE1# ping <LAN de SPOKE2> source <LAN de SPOKE1>
SPOKE1# show dmvpn detail    ! entrada dinamica hacia el otro spoke
```

**Resolución de problemas**
- Sin vecinos EIGRP → revisar `ip nhrp map`/NHS, que el `tunnel key`
  coincida en todos los routers y `no ip split-horizon eigrp 100` en el hub.
- Spokes no registran → confirmar `ip nhrp authentication` idéntica y la IP
  pública del hub en `ip nhrp nhs`/`map`.
- No se forma el túnel directo (Fase 2) → revisar `no ip next-hop-self`.
- No hay atajo (Fase 3) → el hub necesita `ip nhrp redirect` y los spokes
  `ip nhrp shortcut`.

---

## 6. L2TP/IPsec — Client-to-Site

Acceso remoto clásico: un **empleado individual** (Windows o Linux), desde su
casa o un hotel, se conecta a la LAN corporativa como si estuviera
físicamente conectado a la oficina. A diferencia de DMVPN (que une sitios
completos), aquí el extremo remoto es **un solo cliente cuya IP pública no
se conoce de antemano**.

```
┌───────────────────────────────────────────────┐
│  PPP     → autentica (MS-CHAPv2) y asigna IP   │
│            del pool 7.41.99.10–7.41.99.50      │
├───────────────────────────────────────────────┤
│  L2TP    → crea el túnel de nivel 2            │
├───────────────────────────────────────────────┤
│  IPsec   → protege el tráfico (modo TRANSPORTE)│
└───────────────────────────────────────────────┘
        IKEv1 Fase 1/2  →  L2TP  →  PPP (MS-CHAPv2)
```

| Decisión de diseño | Por qué |
|---|---|
| IPsec en **modo transporte**, no túnel | L2TP ya aporta su propio encapsulado; el modo túnel sería doble encapsulado innecesario |
| PSK **comodín** (`address 0.0.0.0 0.0.0.0`) + `crypto dynamic-map` | La IP pública del cliente remoto no se conoce hasta que se conecta |
| PPP con `ms-chap-v2` contra usuarios locales (AAA) | Autenticación estándar soportada de forma nativa por el cliente VPN de Windows |

**Núcleo de la configuración del servidor**
```
vpdn enable
vpdn-group L2TP
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication

crypto isakmp key L2TPpsk address 0.0.0.0 0.0.0.0
crypto ipsec transform-set L2TP-TS esp-aes 256 esp-sha256-hmac
 mode transport
crypto dynamic-map DYN 10
 set transform-set L2TP-TS
crypto map CMAP 10 ipsec-isakmp dynamic DYN

interface Virtual-Template1
 ip unnumbered e0/0
 peer default ip address pool L2TP-POOL
 ppp authentication ms-chap-v2
```

**Configurar el cliente Windows**
1. Configuración → Red e Internet → VPN → Agregar una conexión VPN.
2. Tipo de VPN: **L2TP/IPSec con clave previamente compartida**.
3. Servidor: `200.0.0.1` · PSK: `L2TPpsk`.
4. Usuario/contraseña: `vpnuser` / `VpnPass123` · Autenticación: **MS-CHAP v2**.

Si el cliente está detrás de NAT, se necesita el valor de registro:
```
HKLM\SYSTEM\CurrentControlSet\Services\PolicyAgent\AssumeUDPEncapsulationContextOnSendRule = 2
```

**Verificación (en el servidor)**
```
show crypto isakmp sa
show crypto ipsec sa
show vpdn
show caller ip
```
Desde el cliente conectado: `ping 7.41.7.1` y `tracert 7.41.7.1`.

**Resolución de problemas**
- Conexión cae tras la Fase 1 → suele faltar `mode transport`, o hay
  problemas de NAT-T / registro de Windows.
- Autentica pero no navega a la LAN → verificar `ip unnumbered e0/0` en la
  `Virtual-Template` y que el pool esté asignado con `peer default ip address pool`.
- Depuración: `debug crypto isakmp`, `debug crypto ipsec`, `debug ppp negotiation`.

---

## 7–8. FortiGate — Site-to-Site y Client-to-Site

Dos escenarios sobre **FortiGate 7.6.2 (licencia de evaluación)**, atravesando
en ambos casos un router ISP Cisco sin NAT.

### ⚠️ Restricciones de la licencia trial

| Limitación | Solución aplicada |
|---|---|
| AES/3DES puede ser rechazado (error `-61`) | Fallback a `des-sha256` / `des-sha1`, manteniendo DH grupo 14 |
| SSL VPN modo túnel **eliminado** en FortiOS 7.6.x | Reemplazo: **IPsec dial-up** |
| FortiClient gratuito con limitaciones conocidas en IKEv1 | Usar **IKEv2** para el dial-up |
| FortiClient 7.0.x es el último compatible con Windows 7 | Confirmar versión antes de instalar |

### 7. Site-to-Site (route-based)

Une `7.41.8.0/24` (detrás de `FGT1`) y `7.41.9.0/24` (detrás de `FGT2`) con un
túnel IPsec **basado en interfaz**, atravesando el ISP sin NAT.

```
LAN A 7.41.8.0/24        200.0.0.1/30   200.0.0.5/30        LAN B 7.41.9.0/24
     └── FGT1 (port2/port1) ─── ISP (Cisco, sin NAT) ─── (port1/port2) FGT2 ──┘
```

| Parámetro | Valor |
|---|---|
| Tipo | Site-to-Site basado en interfaz (route-based) |
| Versión IKE | IKEv1 (o `set ike-version 2`) |
| Cifrado/Integridad/DH | AES-256 · SHA-256 · Grupo 14 (DES-SHA en trial) |
| Autenticación | PSK `VPNsecret123` |
| Selectores Fase 2 | `7.41.8.0/24` ↔ `7.41.9.0/24` (invertidos en FGT2) |

```
config vpn ipsec phase1-interface
    edit "to_FGT2"
        set interface "port1"
        set ike-version 1
        set proposal aes256-sha256
        set dhgrp 14
        set remote-gw 200.0.0.5
        set psksecret VPNsecret123
    next
end

config vpn ipsec phase2-interface
    edit "to_FGT2_p2"
        set phase1name "to_FGT2"
        set src-subnet 7.41.8.0 255.255.255.0
        set dst-subnet 7.41.9.0 255.255.255.0
    next
end
```

**Punto crítico:** si el FortiGate tiene una interfaz de gestión/cloud
(p. ej. `port3`) que obtiene IP por DHCP, esa ruta por defecto puede
**sobrescribir silenciosamente** la estática de `port1` y romper el retorno
del túnel:
```
config system interface
    edit port3
        set defaultgw disable
        set distance 254
```

**Verificación**
```
get vpn ipsec tunnel summary
diagnose vpn ike gateway list
execute traceroute 7.41.9.1     ! desde FGT1 hacia LAN B
tracert 7.41.9.10                ! Windows, desde un host de LAN A
traceroute 7.41.9.10             ! Linux
```

### 8. Client-to-Site (dial-up + strongSwan)

Un usuario remoto (Linux + strongSwan) accede a la LAN corporativa
`7.41.11.0/24` mediante un túnel **IPsec dial-up** con **IKEv2** y
**Mode Config** (el FortiGate entrega la IP virtual del pool).

| Parámetro | Valor |
|---|---|
| Tipo | IPsec dial-up (`type dynamic`) con `mode-cfg` |
| Versión IKE | IKEv2 |
| Cifrado/Integridad/DH | DES · SHA-256/SHA-1 · Grupo 14 (DES por licencia trial) |
| Autenticación | PSK `MiVpnLab2026` (`peertype any` + `leftid`) |
| Pool (mode-cfg) | 7.41.99.10 – 7.41.99.50 |
| Split-tunnel | Solo `7.41.11.0/24` viaja cifrado |

```
config vpn ipsec phase1-interface
    edit "dialup"
        set type dynamic
        set ike-version 2
        set peertype any
        set proposal des-sha256 des-sha1
        set dhgrp 14
        set mode-cfg enable
        set ipv4-start-ip 7.41.99.10
        set ipv4-end-ip 7.41.99.50
        set ipv4-split-include "LAN_corp"
        set psksecret MiVpnLab2026
    next
end
```

**Cliente Linux (`/etc/ipsec.conf`)**
```
conn fortigate-c2s
    keyexchange=ikev2
    ike=des-sha256-modp2048,des-sha1-modp2048!
    esp=des-sha256-modp2048,des-sha1-modp2048!
    leftauth=psk
    leftid=lab-client
    leftsourceip=%config
    right=200.0.0.1
    rightauth=psk
    rightsubnet=7.41.11.0/24
    auto=add
```

**Por qué Linux + strongSwan y no el FortiClient de Windows:** durante las
pruebas, el FortiClient de Windows no llegó a enviar paquetes IKE al
conectar (fallo del lado cliente), confirmado con
`diagnose sniffer packet port1 'host 200.0.0.5' 4` en el FortiGate. La
solución fue usar Linux + strongSwan (IKEv2), que sí completó la negociación.

**Verificación (cliente Linux)**
```
sudo ipsec statusall     # ESTABLISHED (Fase 1) / INSTALLED (Fase 2)
ip addr show             # IP virtual 7.41.99.x asignada por Mode Config
ping -c4 7.41.11.10
traceroute 7.41.11.10
```

**Resolución de problemas (ambos escenarios FortiGate)**
- Proposal rechazado → bajar a `des-sha256`/`des-sha1` en ambos extremos.
- Túnel up pero sin retorno de tráfico → revisar `defaultgw` en interfaces DHCP.
- strongSwan se queja de DES → instalar `libstrongswan-standard-plugins` y `sudo ipsec restart`.
- Campos mode-cfg ausentes en el GUI → guardar primero el túnel como Dial-up y volver a editarlo, o completar por CLI.

---

## Herramientas usadas en el laboratorio

- **Cisco Packet Tracer** — routers IOS (IPsec policy-based/VTI/GRE, DMVPN, L2TP).
- **FortiGate 7.6.2** (licencia de evaluación) — VPN IPsec site-to-site y dial-up.
- **strongSwan** (Linux) — cliente IPsec IKEv2 para el escenario dial-up.
- **Windows VPN nativo** — cliente L2TP/IPsec.

## Notas finales

- Las claves precompartidas (`cisco123`, `L2TPpsk`, `VPNsecret123`,
  `DMVPNkey`, `MiVpnLab2026`) son valores de laboratorio pensados para un
  entorno académico aislado — **no deben reutilizarse en producción**.
- Todos los comandos de verificación mostrados en cada sección son los que
  se usaron realmente para validar que el tráfico cruzaba el túnel (y no
  otra ruta), tal como exige el enunciado del laboratorio.
- Este documento resume el trabajo completo; las configuraciones línea por
  línea de cada router/firewall están disponibles bajo pedido, organizadas
  por escenario y por dispositivo.

---

<sub>Trabajo académico de la materia Seguridad de Redes — ITLA.</sub>
