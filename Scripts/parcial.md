 #Configuración VPN — Laboratorio R1 (ISP) + R2/R3/R4

## Esquema de direccionamiento

| Segmento | Red | Router / Gateway | Host |
|---|---|---|---|
| VLAN 10 (Windows10-1, detrás de R2) | 10.13.67.0/24 | R2 f0/1 = 10.13.67.1 | Windows10-1 = 10.13.67.10 |
| VLAN 20 (WindowsServer2022-1, detrás de R3) | 20.13.67.0/24 | R3 f0/1 = 20.13.67.1 | Server = 20.13.67.10 |
| VLAN 30 (Pc1/VPCS, detrás de R4) | 30.13.67.0/24 | R4 f0/1 = 30.13.67.1 | Pc1 = 30.13.67.10 |
| Público (ISP, R1) | 200.13.67.0/24 subneteado en /30 | R1↔️R2: 200.13.67.0/30, R1↔️R3: 200.13.67.4/30, R1↔️R4: 200.13.67.8/30 | — |

*Distribución de las 2 VPN pedidas:*
- *VPN Client-to-Site L2TP/IPsec: el propio **Windows10-1* se conecta como cliente VPN nativo de Windows hacia *R3* (que actúa como servidor LNS), para entrar a la red del Server (VLAN 20) igual que un usuario remoto.
- *VPN Site-to-Site IKEv1: túnel permanente entre **R3* y *R4*, para que la VLAN 20 (Server) y la VLAN 30 (Pc1) se comuniquen de forma transparente (Pc1 es un VPCS y no puede correr un cliente VPN).

R1 solo actúa como ISP/tránsito, no es punto final de ninguna VPN.

---

## R1 — ISP (solo enrutamiento, sin VPN)


enable
configure terminal
hostname R1

interface f0/0
 ip address 200.13.67.1 255.255.255.252
 no shutdown

interface f0/1
 ip address 200.13.67.5 255.255.255.252
 no shutdown

interface f1/0
 ip address 200.13.67.9 255.255.255.252
 no shutdown

end
write memory


R1 no necesita rutas adicionales: las tres redes públicas están directamente conectadas a él.

---

## R2 — Router del cliente (Windows10-1 / VLAN 10)

Solo da salida a Internet (NAT) para que Windows10-1 pueda llegar a R3 y levantar el túnel L2TP. No participa en ninguna VPN él mismo.


enable
configure terminal
hostname R2

interface f0/0
 ip address 200.13.67.2 255.255.255.252
 ip nat outside
 no shutdown

interface f0/1
 ip address 10.13.67.1 255.255.255.0
 ip nat inside
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.13.67.1

access-list 1 permit 10.13.67.0 0.0.0.255
ip nat inside source list 1 interface f0/0 overload

! ===================== DHCP para Windows10-1 (VLAN 10) =====================
ip dhcp excluded-address 10.13.67.1

ip dhcp pool POOL_WIN10
 network 10.13.67.0 255.255.255.0
 default-router 10.13.67.1

end
write memory


En Windows10-1: Configuración → Red → Ethernet → Propiedades IP → *Obtener dirección IP automáticamente (DHCP)*.

---

## R3 — Server (VLAN 20) + Servidor L2TP (LNS) + extremo IKEv1 hacia R4


enable
configure terminal
hostname R3

interface f0/0
 ip address 200.13.67.6 255.255.255.252
 no shutdown

interface f0/1
 ip address 20.13.67.1 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.13.67.5

! ===================== VPN SITE-TO-SITE IKEv1 (R3 <-> R4) =====================
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 2
crypto isakmp key Clave_S2S_123 address 200.13.67.10

crypto ipsec transform-set TS_S2S esp-aes 256 esp-sha-hmac

access-list 101 permit ip 20.13.67.0 0.0.0.255 30.13.67.0 0.0.0.255

crypto map MAPA_VPN 10 ipsec-isakmp
 set peer 200.13.67.10
 set transform-set TS_S2S
 match address 101

! ===================== VPN CLIENT-TO-SITE L2TP/IPsec (Windows10-1 -> R3) ======
aaa new-model
aaa authentication ppp VPDN_AUTH local
aaa authorization network VPDN_AUTH local
username 2025 password 1367

vpdn enable
vpdn-group L2TP_GRUPO
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication

crypto isakmp policy 20
 encr 3des
 hash sha
 authentication pre-share
 group 2
crypto isakmp key ClaveL2TP_123 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS_L2TP esp-3des esp-sha-hmac
 mode transport

crypto dynamic-map MAPA_DIN 5
 set transform-set TS_L2TP

! El mismo crypto map lleva la entrada estática (site-to-site) y la dinámica (L2TP)
crypto map MAPA_VPN 20 ipsec-isakmp dynamic MAPA_DIN

interface f0/0
 crypto map MAPA_VPN

ip local pool POOL_L2TP 20.13.67.100 20.13.67.150

interface Virtual-Template1
 ip unnumbered f0/1
 peer default ip address pool POOL_L2TP
 ppp authentication ms-chap-v2 VPDN_AUTH

end
write memory


---

## R4 — Cliente2 (VLAN 30 / Pc1) — extremo IKEv1 hacia R3


enable
configure terminal
hostname R4

interface f0/0
 ip address 200.13.67.10 255.255.255.252
 no shutdown

interface f0/1
 ip address 30.13.67.1 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.13.67.9

crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 2
crypto isakmp key Clave_S2S_123 address 200.13.67.6

crypto ipsec transform-set TS_S2S esp-aes 256 esp-sha-hmac

access-list 101 permit ip 30.13.67.0 0.0.0.255 20.13.67.0 0.0.0.255

crypto map MAPA_VPN 10 ipsec-isakmp
 set peer 200.13.67.6
 set transform-set TS_S2S
 match address 101

interface f0/0
 crypto map MAPA_VPN

! ===================== DHCP para Pc1 / VPCS (VLAN 30) =====================
ip dhcp excluded-address 30.13.67.1

ip dhcp pool POOL_PC1
 network 30.13.67.0 255.255.255.0
 default-router 30.13.67.1

end
write memory


En Pc1 (VPCS), obtener IP con:

PC1> ip dhcp


---

## Switch1 — detrás de R2 (segmento VLAN 10 / Windows10-1)


enable
configure terminal
hostname Switch1

vlan 10
 name VLAN10_Windows10-1

interface e0/0
 switchport mode access
 switchport access vlan 10
 no shutdown

interface e0/1
 switchport mode access
 switchport access vlan 10
 no shutdown

end
write memory


---

## Switch2 — detrás de R3 (segmento VLAN 20 / WindowsServer2022-1)


enable
configure terminal
hostname Switch2

vlan 20
 name VLAN20_Server

interface e0/0
 switchport mode access
 switchport access vlan 20
 no shutdown

interface e0/1
 switchport mode access
 switchport access vlan 20
 no shutdown

end
write memory


---

## Switch3 — detrás de R4 (segmento VLAN 30 / Pc1 - VPCS)


enable
configure terminal
hostname Switch3

vlan 30
 name VLAN30_Pc1

interface e0/0
 switchport mode access
 switchport access vlan 30
 no shutdown

interface e0/1
 switchport mode access
 switchport access vlan 30
 no shutdown

end
write memory


> *Nota:* e0/0 conecta hacia el router y e0/1 hacia el host (Windows10-1, Server o Pc1 según el switch). Ambos puertos de cada switch quedan en la misma VLAN (10, 20 o 30 respectivamente) para reflejar el nombre del segmento en el diagrama. Como el enlace hacia el router va en modo access (no trunk), el router no necesita saber nada de la VLAN — simplemente ve su interfaz f0/1 conectada a ese segmento igual que antes.

---

## PCs / hosts

- *Windows10-1: obtiene IP por **DHCP* desde el pool POOL_WIN10 de R2 (rango 10.13.67.2–254, gateway 10.13.67.1).
- *WindowsServer2022-1*: IP estática 20.13.67.10/24, gateway 20.13.67.1.
- *Pc1 (VPCS): obtiene IP por **DHCP* desde el pool POOL_PC1 de R4 (rango 30.13.67.2–254, gateway 30.13.67.1):

PC1> ip dhcp


---

## Conexión L2TP en el cliente Windows (Windows10-1)

Pasos para crear la conexión VPN L2TP/IPsec en Windows10-1, apuntando a R3 (200.13.67.6):

1. *Configuración → Red e Internet → VPN → Agregar una conexión VPN*.
2. *Proveedor de VPN*: Windows (integrado).
3. *Nombre de la conexión*: Miguel-L2PT.
4. *Nombre o dirección del servidor*: 200.13.67.6.
5. *Tipo de VPN*: L2TP/IPsec con clave previamente compartida.
6. *Clave precompartida*: ClaveL2TP_123 (debe coincidir con el crypto isakmp key de R3).
7. *Tipo de información de inicio de sesión*: Nombre de usuario y contraseña.
   - Usuario: 2025
   - Contraseña: 1367
8. Guardar y luego ir a *Configuración → Red e Internet → VPN, seleccionar Miguel-L2PT → **Conectar*.

Si prefieres el método clásico (Panel de control):
Panel de control → Redes e Internet → Centro de redes y recursos compartidos → Configurar una nueva conexión → Conectarse a un área de trabajo → Usar mi conexión a Internet (VPN) → 200.13.67.6 → tipo L2TP/IPsec + clave precompartida → usuario/contraseña.

Al conectar, Windows10-1 recibirá una IP del pool 20.13.67.100–150 (asignada por R3) y podrá alcanzar el Server como si estuviera dentro de la VLAN 20.

---

## Verificación rápida


show crypto isakmp sa       ! en R3 y R4 (debe verse QM_IDLE para el túnel S2S)
show crypto ipsec sa        ! ver paquetes encriptados/desencriptados
show vpdn session           ! en R3, ver la sesión L2TP activa de Windows10-1
show ip interface brief     ! confirmar IPs en todos los routers
show interfaces status      ! en cada switch, confirmar que los puertos están up/up en modo access


> *Importante:* la lista AAA usada por ppp authentication ms-chap-v2 debe crearse con aaa authentication ppp VPDN_AUTH local (no aaa authentication login), de lo contrario aparece el warning AAA: authentication list "VPDN_AUTH" is not defined for PPP y el túnel L2TP no autenticará correctamente.

Desde Windows10-1 (una vez conectada la VPN): ping 20.13.67.10
Desde Pc1: ping 20.13.67.10 (debe funcionar automáticamente por el túnel site-to-site, sin cliente VPN).
