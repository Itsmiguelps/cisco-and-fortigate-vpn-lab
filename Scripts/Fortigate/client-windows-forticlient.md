[client-windows-forticlient.md](https://github.com/user-attachments/files/30032839/client-windows-forticlient.md)
# Cliente Windows — FortiClient VPN

## Requisito de la licencia trial del FortiGate
La licencia de evaluación del FortiGate restringe el cifrado IPsec a **DES**
(AES no disponible). Todos los proposals de fase 1/2 en este laboratorio
usan `des-sha256` / `des-sha1` + DH group 14 para poder negociar.

## Opción A — IPsec VPN (IKEv2)

1. FortiClient → Add New VPN → pestaña **IPsec VPN**.
2. Connection Name: `lab-c2s` · Remote Gateway: `200.0.0.1`.
3. Authentication Method: Pre-shared key → `DialupSecret123`.
4. Authentication (XAuth): Prompt on login.
5. Advanced Settings:
   - IKE: **Version 2**, Mode: Aggressive (solo aplica a IKEv1),
     Address Assignment: Mode Config.
   - Phase 1 y Phase 2: Encryption **DES**, Authentication **SHA256** / **SHA1**,
     DH Group: **solo 14** (no marcar más de un grupo).
6. Connect → usuario `vpnuser` / clave `VpnPass123`.

> **Nota de compatibilidad:** algunas versiones recientes de FortiClient
> (p. ej. "Zero Trust Fabric Agent" sin licencia) no soportan **IKEv1**
> y marcan la conexión como `[IKEv1 Unsupported]`. Verificar que el túnel
> del FortiGate esté en **IKE Version 2** si el cliente lo exige.

## Opción B — SSL-VPN (recomendada, ver docs/08-fortigate-client-to-site.md)

1. FortiClient → Add New VPN → pestaña **SSL-VPN**.
2. Connection Name: `labssl` · Remote Gateway: `200.0.0.1`.
3. Customize port: `10443`.
4. Authentication: Prompt on login. Aceptar advertencia del certificado
   (es el certificado de fábrica, no un CA válido).
5. Connect → usuario `vpnuser` / clave `VpnPass123`.

## Verificación (ambas opciones)
```
ipconfig             (busca el adaptador virtual de Fortinet con IP 7.41.99.x)
tracert 7.41.11.1     (LAN corporativa detrás del FortiGate)
```
