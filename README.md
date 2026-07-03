# VPN-IPSec-IKEv1-Route-Based-VTI

<img width="654" height="276" alt="image" src="https://github.com/user-attachments/assets/edfcc30b-66cc-4465-95dc-13ea7e5fda80" />

**LABORATORIO DE NETWORKING**

**VPN IPSec IKEv1 — Route-Based VTI**

_Documentación Técnica Profesional_

**Heldry Terrero**

Matrícula: 2025-0719

Materia: Seguridad de Redes

Fecha: Junio 2026

|   |
|---|
|⚠ AVISO IMPORTANTE — Este proyecto fue desarrollado únicamente con fines educativos y de laboratorio controlado, en el marco de la asignatura Seguridad de Redes del ITLA. Los scripts y técnicas documentadas SOLO deben ejecutarse en simuladores autorizados (PNetLab, GNS3, EVE-NG). QUEDA ESTRICTAMENTE PROHIBIDO usar este material en redes públicas o de terceros sin autorización explícita. El uso indebido puede constituir un delito conforme a las leyes de ciberseguridad de la República Dominicana.|

  

  

# 1. Objetivo

Implementar VPN Route-Based con Virtual Tunnel Interface (VTI) e IKEv1. A diferencia de Policy-Based, no se usan ACLs: el tráfico se cifra automáticamente al enrutarse hacia Tunnel0. La interfaz Tunnel0 aparece como interfaz real con IP propia, visible en show ip interface brief.

•        Configurar IPSec Profile (reemplaza al Crypto Map).

•        Crear interfaz Tunnel0 con tunnel mode ipsec ipv4.

•        Enrutar LAN remota hacia Tunnel0 con ip route.

•        Verificar Tunnel0 up/up con show ip interface brief.

•        Verificar QM_IDLE y contadores encaps/decaps.

# 2. Marco Teórico

## 2.1 Policy-Based vs Route-Based

|**Característica**|**Policy-Based (VPN 1-2)**|**Route-Based VTI (VPN 3)**|
|---|---|---|
|Selección tráfico|ACL de criptografía|Tabla de enrutamiento (ip route → Tunnel0)|
|Interfaz de túnel|No existe — transparente|Tunnel0 — interfaz real con IP propia|
|Crypto Map|Requerido en WAN|No se usa — reemplazado por IPSec Profile|
|Soporte routing dinámico|Limitado|Sí — OSPF/EIGRP/BGP sobre Tunnel0|
|Estado visible|No como interfaz|up/up en show ip interface brief|
|Multicast|No soportado|Soportado nativamente|

## 2.2 IPSec Profile vs Crypto Map

|**Elemento**|**Crypto Map**|**IPSec Profile**|
|---|---|---|
|Comando|crypto map MAP 10 ipsec-isakmp|crypto ipsec profile PROF_VTI|
|Peer remoto|set peer X.X.X.X|Definido en tunnel destination|
|Transform Set|set transform-set TS_VPN|set transform-set TS_VPN|
|ACL de tráfico|match address (requerida)|No requerida — todo Tunnel0 se cifra|
|Se aplica en|interface WAN (e0/0)|interface Tunnel0|
|Comando aplicación|crypto map MAP_VPN|tunnel protection ipsec profile|

## 2.3 Subred del Túnel VTI — 172.19.7.0/30

|**Router**|**IP Tunnel0**|**Descripción**|
|---|---|---|
|R1-PEER-A|172.19.7.1 /30|Extremo local del túnel VTI|
|R2-PEER-B|172.19.7.2 /30|Extremo remoto del túnel VTI|

#   
  
3. Topología

<img width="1195" height="152" alt="1-6" src="https://github.com/user-attachments/assets/95f746ec-6aa5-4d8d-ac1f-ca10c91a1e76" />

|---|

|**Dispositivo**|**Interfaz**|**IP**|**Máscara**|**Descripción**|
|---|---|---|---|---|
|R1-PEER-A|e0/0|20.25.1.1|255.255.255.252|WAN → ISP|
|R1-PEER-A|e0/1|20.25.7.1|255.255.255.240|LAN Sitio A|
|R1-PEER-A|Tunnel0|172.19.7.1|255.255.255.252|VTI extremo A|
|R-ISP|e0/1|20.25.1.2|255.255.255.252|Enlace → R1|
|R-ISP|e0/0|20.25.2.1|255.255.255.252|Enlace → R2|
|R2-PEER-B|e0/0|20.25.2.2|255.255.255.252|WAN → ISP|
|R2-PEER-B|e0/1|20.25.9.1|255.255.255.240|LAN Sitio B|
|R2-PEER-B|Tunnel0|172.19.7.2|255.255.255.252|VTI extremo B|
|PC-A|eth0|20.25.7.2|255.255.255.240|Host LAN A|
|PC-B|eth0|20.25.9.2|255.255.255.240|Host LAN B|

## 3.1 Rutas estáticas hacia Tunnel0

|**Router**|**Comando ip route**|**Red destino**|**Salida**|
|---|---|---|---|
|R1-PEER-A|ip route 20.25.9.0 255.255.255.240 Tunnel0|20.25.9.0/28|Tunnel0|
|R2-PEER-B|ip route 20.25.7.0 255.255.255.240 Tunnel0|20.25.7.0/28|Tunnel0|
|En Route-Based VPN la ruta apunta a la interfaz Tunnel0 (no a un next-hop IP). El router encamina hacia Tunnel0 y tunnel protection cifra automáticamente con ESP.|   |   |   |

# 4. Parámetros IPSec

|**Parámetro**|**Valor**|**Descripción**|
|---|---|---|
|Cifrado Fase 1|AES-256|AES 256 bits|
|Hash Fase 1|SHA-256|HMAC-SHA256|
|PSK|cisco123|Pre-Shared Key|
|Grupo DH|14 (2048 bits)|NIST válido hasta 2030|
|Lifetime|86400 s|24 horas|
|Transform Set|TS_VPN|esp-aes 256 esp-sha256-hmac mode tunnel|
|IPSec Profile|PROF_VTI|set transform-set TS_VPN — reemplaza Crypto Map|
|tunnel mode|ipsec ipv4|Encapsula directamente en ESP sin GRE|

# 5. Scripts de Configuración

## 5.1 R1-PEER-A

enable  
configure terminal  
hostname R1-PEER-A  
!  
interface ethernet0/0  
 description WAN_ISP  
 ip address 20.25.1.1 255.255.255.252  
 no shutdown  
!  
interface ethernet0/1  
 description LAN_A  
 ip address 20.25.7.1 255.255.255.240  
 no shutdown  
!  
ip route 0.0.0.0 0.0.0.0 20.25.1.2  
!  
! === FASE 1: ISAKMP Policy ===  
crypto isakmp policy 10  
 encryption aes 256  
 hash sha256  
 authentication pre-share  
 group 14  
 lifetime 86400  
!  
crypto isakmp key cisco123 address 20.25.2.2  
!  
! === FASE 2: Transform Set ===  
crypto ipsec transform-set TS_VPN esp-aes 256 esp-sha256-hmac  
 mode tunnel  
!  
! === IPSec Profile (reemplaza Crypto Map) ===  
crypto ipsec profile PROF_VTI  
 set transform-set TS_VPN  
!  
! === Interfaz VTI Tunnel0 ===  
interface Tunnel0  
 description TUNEL_HACIA_R2  
 ip address 172.19.7.1 255.255.255.252  
 tunnel source ethernet0/0  
 tunnel destination 20.25.2.2  
 tunnel mode ipsec ipv4  
 tunnel protection ipsec profile PROF_VTI  
!  
! === Ruta estática hacia LAN-B por Tunnel0 ===  
ip route 20.25.9.0 255.255.255.240 Tunnel0  
!  
do write memory

## 5.2 R-ISP

enable  
configure terminal  
hostname R-ISP  
!  
interface ethernet0/1  
 ip address 20.25.1.2 255.255.255.252  
 no shutdown  
!  
interface ethernet0/0  
 ip address 20.25.2.1 255.255.255.252  
 no shutdown  
!  
do write memory

## 5.3 R2-PEER-B

enable  
configure terminal  
hostname R2-PEER-B  
!  
interface ethernet0/0  
 ip address 20.25.2.2 255.255.255.252  
 no shutdown  
!  
interface ethernet0/1  
 ip address 20.25.9.1 255.255.255.240  
 no shutdown  
!  
ip route 0.0.0.0 0.0.0.0 20.25.2.1  
!  
crypto isakmp policy 10  
 encryption aes 256  
 hash sha256  
 authentication pre-share  
 group 14  
 lifetime 86400  
!  
crypto isakmp key cisco123 address 20.25.1.1  
!  
crypto ipsec transform-set TS_VPN esp-aes 256 esp-sha256-hmac  
 mode tunnel  
!  
crypto ipsec profile PROF_VTI  
 set transform-set TS_VPN  
!  
interface Tunnel0  
 description TUNEL_HACIA_R1  
 ip address 172.19.7.2 255.255.255.252  
 tunnel source ethernet0/0  
 tunnel destination 20.25.1.1  
 tunnel mode ipsec ipv4  
 tunnel protection ipsec profile PROF_VTI  
!  
ip route 20.25.7.0 255.255.255.240 Tunnel0  
!  
do write memory

## 5.4 Hosts

PC-A> ip 20.25.7.2/28 20.25.7.1  
PC-B> ip 20.25.9.2/28 20.25.9.1

# 6. Verificación

## 6.1 Verificación exclusiva Route-Based — show ip interface brief

Este comando confirma que Tunnel0 está up/up — exclusivo del modelo Route-Based:

R1-PEER-A# show ip interface brief  
   
Interface              IP-Address      OK? Method Status    Protocol  
Ethernet0/0            20.25.1.1       YES NVRAM  up        up  
Ethernet0/1            20.25.7.1       YES NVRAM  up        up  
Tunnel0                172.19.7.1      YES NVRAM  up        up  
   
! Tunnel0 up/up = tunel VTI activo

<img width="1023" height="187" alt="show brief" src="https://github.com/user-attachments/assets/c87eda05-0128-4906-b7d1-5fc46323bbb0" />

| ------------------- |

## 6.2 Fase 1 — show crypto isakmp sa (QM_IDLE)

R1-PEER-A# show crypto isakmp sa  
   
dst             src             state    conn-id status  
20.25.2.2       20.25.1.1       QM_IDLE     1001 ACTIVE

<img width="842" height="194" alt="isakmp" src="https://github.com/user-attachments/assets/05618b21-966c-4057-ada3-2c2979cd3a30" />

| --------------- |

## 6.3 Fase 2 — show crypto ipsec sa (interfaz Tunnel0)

R1-PEER-A# show crypto ipsec sa  
   
interface: Tunnel0  
  #pkts encaps: 5,  #pkts encrypt: 5,  #pkts digest: 5  
  #pkts decaps: 5,  #pkts decrypt: 5,  #pkts verify: 5  
  ! La interfaz es Tunnel0, no Ethernet0/0

<img width="629" height="51" alt="encrypt IPSEC" src="https://github.com/user-attachments/assets/58b5a783-ad23-4569-9761-fa2c76780d1e" />

| ---------------------- |

## 6.4 Verificar ruta estática

R1-PEER-A# show ip route  
S    20.25.9.0/28 is directly connected, Tunnel0

## 6.5 Ping PC-A a PC-B

PC-A> ping 20.25.9.2

<img width="675" height="190" alt="Ping" src="https://github.com/user-attachments/assets/ff7be59a-0a5d-4180-a83a-612fe12792c2" />

| ------------- |

# 7. Troubleshooting

|**Síntoma**|**Causa**|**Solución**|
|---|---|---|
|Tunnel0 down/down|tunnel destination inalcanzable|ping 20.25.2.2 source e0/0 — verificar ISP|
|Tunnel0 up/down|IPSec SA no establecida|debug crypto isakmp — verificar PSK|
|Ping falla / túnel up|Ruta estática falta|ip route 20.25.9.0 255.255.255.240 Tunnel0|
|#pkts encaps=0|IPSec profile no asociado|Verificar tunnel protection ipsec profile PROF_VTI|

# 8. Buenas Prácticas

•        Routing dinámico: configurar OSPF directamente sobre Tunnel0 — ip ospf 1 area 0.

•        QoS sobre túnel: aplicar políticas de calidad de servicio en Tunnel0.

•        MTU: configurar ip mtu 1400 en Tunnel0 para evitar fragmentación.

•        Monitoreo: Tunnel0 up/up es indicador de salud integrable con SNMP.

•        PFS: agregar set pfs group14 en PROF_VTI.

|**Marco**|**Relevancia**|
|---|---|
|NIST SP 800-77|Guía IPSec — valida ESP-AES256|
|RFC 4301|Arquitectura IPSec — modo túnel|
|ISO/IEC 27033-5|Seguridad en comunicaciones VPN|
|Ley 172-13 RD|Protección de datos — justifica VPN|

# 9. Conclusión

La VPN Route-Based VTI con IKEv1 elimina las ACLs de criptografía, simplificando la configuración. La triple verificación (Tunnel0 up/up, QM_IDLE, encaps/decaps > 0) confirma el funcionamiento en interfaz, control y datos. La principal ventaja operativa es la visibilidad del túnel como interfaz real y el soporte nativo de routing dinámico.

|   |
|---|
|RESULTADO: Tunnel0 up/up, QM_IDLE confirmado, encaps/decaps > 0. VTI IKEv1 activo entre PC-A y PC-B.|

Heldry Terrero — Matrícula: 2025-0719 — Seguridad de Redes — Junio 2026

Link del Video: [https://youtu.be/zJ_Ucfv95u4](https://youtu.be/zJ_Ucfv95u4)  
  

Link del GitHub: https://github.com/HeldryRoot/VPN-IPSec-IKEv1-Route-Based-VTI
