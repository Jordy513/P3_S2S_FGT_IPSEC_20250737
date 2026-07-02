# Capturas de pantalla — VPN Site-to-Site IPSec Route-Based con FortiGate (IKEv1)

Capturas del laboratorio en orden de demostración.

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](/screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, ambos FortiGate, switches y PCs encendidos. |
| 2 | [`02_fgt1_phase1_phase2.png`](/screenshots/02_fgt1_phase1_phase2.png) | GUI de FortiGate1 → `VPN → IPsec Tunnels → VPN-a-FGT2` mostrando la configuración de Fase 1 y Fase 2 (proposal `des-sha1`, remote-gw `192.168.1.20`). |
| 3 | [`03_fgt1_static_route.png`](/screenshots/03_fgt1_static_route.png) | GUI de FortiGate1 → `Network → Static Routes` mostrando la ruta hacia `20.25.37.0/25` vía la interfaz `VPN-a-FGT2`. |
| 4 | [`04_fgt1_firewall_policies.png`](/screenshots/04_fgt1_firewall_policies.png) | GUI de FortiGate1 → `Policy & Objects → Firewall Policy` mostrando las políticas `LAN-a-VPN` y `VPN-a-LAN`. |
| 5 | [`05_fgt2_phase1_phase2.png`](/screenshots/05_fgt2_phase1_phase2.png) | GUI de FortiGate2 → `VPN → IPsec Tunnels → VPN-a-FGT1`, configuración espejo de la de FGT1. |
| 6 | [`06_ipsec_tunnel_up.png`](/screenshots/06_ipsec_tunnel_up.png) | `VPN → IPsec Tunnels` (o `Monitor → IPsec Monitor`) mostrando el túnel en estado **Up** con contadores de bytes/paquetes activos. |
| 7 | [`07_forward_traffic_accept.png`](/screenshots/07_forward_traffic_accept.png) | `Log & Report → Forward Traffic` mostrando el tráfico de PC1 hacia PC2 cruzando el túnel con acción `Accept`. |
| 8 | [`08_ping_pc1_a_pc2.png`](/screenshots/08_ping_pc1_a_pc2.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC2 (`20.25.37.2`) a través del túnel Site-to-Site. |
| 9 | [`09_traceroute_pc1_a_pc2.png`](/screenshots/09_traceroute_pc1_a_pc2.png) | Traceroute desde PC1 hacia PC2 mostrando el camino a través del túnel VPN. |
