# Cliente L2TP/IPsec — Windows

1. Configuración > Red > VPN > Agregar conexión.
2. Tipo de VPN: **L2TP/IPSec con clave previamente compartida**.
3. Dirección del servidor: `200.0.0.1` · Clave compartida (PSK): `L2TPpsk`.
4. Usuario/contraseña: `vpnuser` / `VpnPass123` · Autenticación: **MS-CHAP v2**.
5. Si el cliente está detrás de NAT, crear el registro:
   ```
   HKLM\SYSTEM\CurrentControlSet\Services\PolicyAgent\AssumeUDPEncapsulationContextOnSendRule = 2   (DWORD)
   ```
   y reiniciar.
