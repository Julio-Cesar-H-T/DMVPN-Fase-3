# DMVPN Fase 2 — Hub and Spoke punto a multipunto (IKEv1)

## Información del Proyecto

| Campo           | Detalle                               |
| --------------- | ------------------------------------- |
| **Estudiante**  | Julio César Hernández Tibrey          |
| **Matrícula**   | 2025-0702                             |
| **Institución** | Instituto Tecnológico de Las Americas |
| **Asignatura**  | Seguridad de Redes                    |
| **Profesor**    | Jonathan Esteban Rondón Corniel       |

---

## Objetivo

Implementar una VPN **hub and spoke punto a multipunto** utilizando **DMVPN Fase 2** con **IPSec IKEv1**, integrando tres tecnologías: túneles **mGRE** (Multipoint GRE), el protocolo **NHRP** (Next Hop Resolution Protocol) y **EIGRP** como protocolo de enrutamiento dinámico.

El objetivo funcional es lograr que los tres sitios del laboratorio — Hub (R-HUB), Spoke-1 (R-SPOKE-1) y Spoke-2 (R-SPOKE-2) — puedan comunicar sus LANs privadas de forma cifrada a través de una red de tránsito simulada por el ISP, con la capacidad adicional de que los Spokes establezcan **túneles directos entre sí** sin que el tráfico deba transitar por el Hub de forma permanente.

---

## Contexto Teórico

### ¿Qué es DMVPN?

**DMVPN (Dynamic Multipoint VPN)** es una solución de Cisco que simplifica el despliegue y la gestión de VPNs a gran escala. A diferencia de las VPNs site-to-site tradicionales (donde cada par de sitios requiere una configuración de túnel estática dedicada), DMVPN permite que un único Hub atienda a múltiples Spokes con una configuración mínima, y que los Spokes construyan túneles directos entre sí de forma dinámica cuando es necesario.

### Ventajas sobre VPN site-to-site tradicional

| Aspecto                     | Site-to-Site tradicional                    | DMVPN                                   |
| --------------------------- | ------------------------------------------- | --------------------------------------- |
| Configuración de túneles    | Manual por cada par de sitios               | Una sola interfaz mGRE en el Hub        |
| Escalabilidad               | Baja — N sitios = N×(N-1)/2 túneles         | Alta — solo se configura el Hub una vez |
| Túneles Spoke—Spoke         | Requieren config estática en ambos extremos | Se construyen dinámicamente vía NHRP    |
| Soporte de routing dinámico | Limitado                                    | Total — OSPF, EIGRP, BGP sobre el túnel |
| IPs de Spoke                | Deben ser estáticas y conocidas             | Pueden ser dinámicas (DHCP del ISP)     |

### IKEv1 en DMVPN

En DMVPN los peers no son fijos ni conocidos de antemano (los Spokes pueden tener IPs dinámicas), por lo que la PSK no puede configurarse por IP específica. En IKEv1 se resuelve con una **PSK wildcard** (`address 0.0.0.0 0.0.0.0`): cualquier peer que presente la clave correcta es autenticado. Esto es funcional pero menos granular que IKEv2 (que permite PSK por peer dentro del keyring).

---

## Componentes de DMVPN

### mGRE (Multipoint GRE)

A diferencia de un túnel GRE point-to-point (que tiene un único destino fijo), una interfaz **mGRE** no tiene `tunnel destination` definido — acepta y origina tráfico hacia múltiples destinos dinámicamente. Esto permite que una sola interfaz `Tunnel0` en el Hub maneje comunicaciones con todos los Spokes simultáneamente, sin necesidad de crear una interfaz de túnel por cada Spoke.

### NHRP (Next Hop Resolution Protocol)

NHRP es el "directorio telefónico" de DMVPN. Funciona en dos roles:

- **NHS (Next Hop Server):** El Hub actúa como servidor NHRP. Mantiene una base de datos que mapea las IPs de túnel de cada Spoke con su IP pública (NBMA). Los Spokes se registran en el Hub al levantar su interfaz Tunnel0.
- **NHC (Next Hop Client):** Cada Spoke actúa como cliente NHRP. Al arrancar, registra su IP pública en el Hub. Cuando necesita comunicarse con otro Spoke, consulta al Hub la IP pública del Spoke destino y usa esa información para construir el túnel directo.

### IPSec sobre mGRE

El cifrado se aplica con `tunnel protection ipsec profile` directamente sobre la interfaz Tunnel0, en **modo transport** (GRE ya encapsula, IPSec solo protege el payload sin agregar una nueva cabecera IP). Todos los paquetes que entran y salen de Tunnel0 quedan cifrados automáticamente.

### EIGRP sobre el túnel

EIGRP se ejecuta sobre la red de túnel (`10.7.2.192/27`), permitiendo que el Hub anuncie las rutas de todos los sitios y que los Spokes aprendan cómo llegar a las LANs remotas. En Fase 2, el próximo salto de las rutas aprendidas por EIGRP **siempre apunta al Hub**, independientemente de si existe un túnel directo entre Spokes.

---

## Fases de DMVPN

| Fase  | Spoke—Spoke                                    | Próximo salto en ruta                      | Comandos exclusivos                                    |
| ----- | ---------------------------------------------- | ------------------------------------------ | ------------------------------------------------------ |
| **1** | No — todo pasa por Hub                         | Hub siempre                                | Ninguno                                                |
| **2** | Sí — túnel directo, pero ruta apunta al Hub    | Hub (CEF reenvía directo)                  | Ninguno extra                                          |
| **3** | Sí — túnel directo y ruta actualizada al Spoke | Spoke remoto (ruta /32 instalada por NHRP) | `ip nhrp redirect` (Hub) + `ip nhrp shortcut` (Spokes) |

**Este laboratorio implementa Fase 2:** los Spokes pueden comunicarse directamente gracias a NHRP, pero la ruta en la tabla de enrutamiento sigue apuntando al Hub como próximo salto. CEF (Cisco Express Forwarding) se encarga de reenviar el paquete al Spoke correcto consultando la tabla NHRP.

---

## Topología



```
                         HUB (R-HUB)
                       10.7.2.0/27 LAN
                       200.7.2.1/30 WAN
                       10.7.2.193/27 Tunnel0
                            |
                           ISP
                          /   \
               200.7.2.4/30   200.7.2.8/30
                    /               \
             R-SPOKE-1           R-SPOKE-2
          10.7.2.32/27 LAN    10.7.2.64/27 LAN
          200.7.2.6/30 WAN    200.7.2.10/30 WAN
          10.7.2.194/27 T0    10.7.2.195/27 T0
          <-------- tunel dinamico NHRP -------->
```

### Tabla de Direccionamiento

|Dispositivo|Interfaz|Dirección IP|Máscara|Descripción|
|---|---|---|---|---|
|ISP|e0/0|200.7.2.2|255.255.255.252|Enlace a R-HUB|
|ISP|e0/1|200.7.2.5|255.255.255.252|Enlace a R-SPOKE-1|
|ISP|e0/2|200.7.2.9|255.255.255.252|Enlace a R-SPOKE-2|
|R-HUB|e0/0|200.7.2.1|255.255.255.252|WAN (hacia ISP)|
|R-HUB|e0/1|10.7.2.1|255.255.255.224|LAN HUB|
|R-HUB|Tunnel0|10.7.2.193|255.255.255.224|mGRE — NHS|
|R-SPOKE-1|e0/0|200.7.2.6|255.255.255.252|WAN (hacia ISP)|
|R-SPOKE-1|e0/1|10.7.2.33|255.255.255.224|LAN SPOKE-1|
|R-SPOKE-1|Tunnel0|10.7.2.194|255.255.255.224|mGRE — NHC|
|R-SPOKE-2|e0/0|200.7.2.10|255.255.255.252|WAN (hacia ISP)|
|R-SPOKE-2|e0/1|10.7.2.65|255.255.255.224|LAN SPOKE-2|
|R-SPOKE-2|Tunnel0|10.7.2.195|255.255.255.224|mGRE — NHC|

### Red de túnel mGRE

|Red|Rango usable|Descripción|
|---|---|---|
|10.7.2.192/27|.193 → .222|Interfaz Tunnel0 de Hub y Spokes|

> La red de túnel usa /27 para evitar overlap con las LANs privadas (10.7.2.0/27, 10.7.2.32/27, 10.7.2.64/27) que también pertenecen al espacio 10.7.2.0/24.

### Pools DHCP

|Router|Pool|Red|Gateway|
|---|---|---|---|
|R-HUB|LAN-HUB|10.7.2.0/27|10.7.2.1|
|R-SPOKE-1|LAN-SPOKE1|10.7.2.32/27|10.7.2.33|
|R-SPOKE-2|LAN-SPOKE2|10.7.2.64/27|10.7.2.65|

`[ESPACIO PARA CAPTURA: Topología completa en PNETLab]`

---

## Configuración Base de la Infraestructura

### ISP

```
conf t
hostname ISP
int e0/0
 ip add 200.7.2.2 255.255.255.252
 no shut
int e0/1
 ip add 200.7.2.5 255.255.255.252
 no shut
int e0/2
 ip add 200.7.2.9 255.255.255.252
 no shut
exit
```

> El ISP no necesita rutas hacia las LANs privadas ni hacia la red de túnel. Solo enruta entre sus tres interfaces de tránsito directamente conectadas.

### R-HUB

```
conf t
hostname R-HUB
int e0/0
 ip add 200.7.2.1 255.255.255.252
 no shut
int e0/1
 ip add 10.7.2.1 255.255.255.224
 no shut
exit
ip route 0.0.0.0 0.0.0.0 200.7.2.2
ip dhcp excluded-address 10.7.2.1
ip dhcp pool LAN-HUB
 network 10.7.2.0 255.255.255.224
 default-router 10.7.2.1
 dns-server 8.8.8.8
exit
```

### R-SPOKE-1

```
conf t
hostname R-SPOKE-1
int e0/0
 ip add 200.7.2.6 255.255.255.252
 no shut
int e0/1
 ip add 10.7.2.33 255.255.255.224
 no shut
exit
ip route 0.0.0.0 0.0.0.0 200.7.2.5
ip dhcp excluded-address 10.7.2.33
ip dhcp pool LAN-SPOKE1
 network 10.7.2.32 255.255.255.224
 default-router 10.7.2.33
 dns-server 8.8.8.8
exit
```

### R-SPOKE-2

```
conf t
hostname R-SPOKE-2
int e0/0
 ip add 200.7.2.10 255.255.255.252
 no shut
int e0/1
 ip add 10.7.2.65 255.255.255.224
 no shut
exit
ip route 0.0.0.0 0.0.0.0 200.7.2.9
ip dhcp excluded-address 10.7.2.65
ip dhcp pool LAN-SPOKE2
 network 10.7.2.64 255.255.255.224
 default-router 10.7.2.65
 dns-server 8.8.8.8
exit
```

### SW-HUB / SW-SPOKE-1 / SW-SPOKE-2

```
conf t
hostname SW-HUB          ! (repetir para SW-SPOKE-1 y SW-SPOKE-2)
int e0/0
 switchport mode access
 switchport access vlan 1
 spanning-tree portfast
 no shut
int e0/1
 switchport mode access
 switchport access vlan 1
 spanning-tree portfast
 no shut
exit
```

### Verificación de la base

```
show ip interface brief
ping 200.7.2.2        ! desde R-HUB → ISP
ping 200.7.2.5        ! desde R-SPOKE-1 → ISP
ping 200.7.2.9        ! desde R-SPOKE-2 → ISP
```

`[ESPACIO PARA CAPTURA: ping exitoso de cada router hacia el ISP antes de aplicar DMVPN]`

---

## Configuración DMVPN Fase 2 con IKEv1

### R-HUB

```
conf t

! ----- FASE 1: IKEv1 (ISAKMP) -----
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
exit

! PSK wildcard: acepta cualquier peer con esta clave
crypto isakmp key 2025-0702 address 0.0.0.0 0.0.0.0

! ----- FASE 2: IPSec -----
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit

crypto ipsec profile DMVPN-PROF
 set transform-set TS-DMVPN
exit

! ----- INTERFAZ TUNNEL mGRE -----
interface Tunnel0
 ip address 10.7.2.193 255.255.255.224
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 ip nhrp authentication 2025-0702
 no ip split-horizon eigrp 100
 no ip next-hop-self eigrp 100
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
 no shutdown
exit

! ----- ENRUTAMIENTO DINÁMICO: EIGRP -----
router eigrp 100
 network 10.7.2.0 0.0.0.31
 network 10.7.2.32 0.0.0.31
 network 10.7.2.64 0.0.0.31
 network 10.7.2.192 0.0.0.31
 no auto-summary
exit

end
write memory
```

`[ESPACIO PARA CAPTURA: configuración DMVPN aplicada en R-HUB — terminal]`

### R-SPOKE-1

```
conf t

! ----- FASE 1: IKEv1 (ISAKMP) -----
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
exit

crypto isakmp key 2025-0702 address 0.0.0.0 0.0.0.0

! ----- FASE 2: IPSec -----
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit

crypto ipsec profile DMVPN-PROF
 set transform-set TS-DMVPN
exit

! ----- INTERFAZ TUNNEL mGRE -----
interface Tunnel0
 ip address 10.7.2.194 255.255.255.224
 ip nhrp network-id 1
 ip nhrp authentication 2025-0702
 ip nhrp nhs 10.7.2.193
 ip nhrp map 10.7.2.193 200.7.2.1
 ip nhrp map multicast 200.7.2.1
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
 no shutdown
exit

! ----- ENRUTAMIENTO DINÁMICO: EIGRP -----
router eigrp 100
 network 10.7.2.32 0.0.0.31
 network 10.7.2.192 0.0.0.31
 no auto-summary
exit

end
write memory
```

`[ESPACIO PARA CAPTURA: configuración DMVPN aplicada en R-SPOKE-1 — terminal]`

### R-SPOKE-2

```
conf t

! ----- FASE 1: IKEv1 (ISAKMP) -----
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
exit

crypto isakmp key 2025-0702 address 0.0.0.0 0.0.0.0

! ----- FASE 2: IPSec -----
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit

crypto ipsec profile DMVPN-PROF
 set transform-set TS-DMVPN
exit

! ----- INTERFAZ TUNNEL mGRE -----
interface Tunnel0
 ip address 10.7.2.195 255.255.255.224
 ip nhrp network-id 1
 ip nhrp authentication 2025-0702
 ip nhrp nhs 10.7.2.193
 ip nhrp map 10.7.2.193 200.7.2.1
 ip nhrp map multicast 200.7.2.1
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
 no shutdown
exit

! ----- ENRUTAMIENTO DINÁMICO: EIGRP -----
router eigrp 100
 network 10.7.2.64 0.0.0.31
 network 10.7.2.192 0.0.0.31
 no auto-summary
exit

end
write memory
```

`[ESPACIO PARA CAPTURA: configuración DMVPN aplicada en R-SPOKE-2 — terminal]`

---

## Explicación de los Componentes

### Fase 1 — ISAKMP (IKEv1)

|Parámetro|Valor|Descripción|
|---|---|---|
|`encryption aes 256`|AES-256|Algoritmo de encriptación|
|`hash sha256`|SHA-256|Algoritmo de integridad|
|`authentication pre-share`|PSK|Método de autenticación|
|`group 14`|DH Group 14 (2048-bit)|Grupo Diffie-Hellman|
|`lifetime 86400`|24 horas|Tiempo de vida de la SA|
|`key 2025-0702 address 0.0.0.0 0.0.0.0`|PSK wildcard|Acepta cualquier peer con esta clave — necesario en DMVPN ya que los Spokes pueden tener IPs dinámicas o desconocidas|

### Fase 2 — IPSec

|Parámetro|Valor|Descripción|
|---|---|---|
|`esp-aes 256`|AES-256|Encriptación del payload|
|`esp-sha256-hmac`|SHA-256|Integridad del payload|
|`mode transport`|Transport|GRE ya encapsula — IPSec solo protege sin agregar nueva cabecera IP|

### Interfaz Tunnel0 — mGRE

|Parámetro|Rol|Descripción|
|---|---|---|
|`tunnel mode gre multipoint`|Hub y Spokes|Define el túnel como mGRE — sin `tunnel destination` fijo, acepta múltiples peers dinámicamente|
|`ip nhrp network-id 1`|Hub y Spokes|ID de red NHRP — debe ser idéntico en todos los nodos del DMVPN|
|`ip nhrp authentication 2025-0702`|Hub y Spokes|Contraseña NHRP — protege el proceso de registro y resolución|
|`ip nhrp map multicast dynamic`|Solo Hub|Permite a los Spokes registrar sus IPs de forma dinámica; el Hub distribuye actualizaciones de routing por multicast a todos los registrados|
|`ip nhrp nhs 10.7.2.193`|Solo Spokes|Apunta al Hub como Next Hop Server — destino del registro NHRP|
|`ip nhrp map 10.7.2.193 200.7.2.1`|Solo Spokes|Mapeo estático: IP de túnel del Hub → IP pública del Hub (necesario para iniciar el registro antes de que NHRP esté activo)|
|`ip nhrp map multicast 200.7.2.1`|Solo Spokes|Envía tráfico multicast (actualizaciones EIGRP) hacia la IP pública del Hub|
|`tunnel protection ipsec profile DMVPN-PROF`|Hub y Spokes|Cifra todo el tráfico de Tunnel0 con IPSec|

### EIGRP sobre el túnel

|Parámetro|Descripción|
|---|---|
|`network 10.7.2.192 0.0.0.31`|Anuncia la red de túnel mGRE — permite que Hub y Spokes se vean entre sí a nivel de túnel|
|`network 10.7.2.X 0.0.0.31`|Anuncia la LAN de cada sitio — permite que los Spokes aprendan las rutas de las LANs remotas|
|`no auto-summary`|Desactiva la sumarización automática — necesario para que las subredes /27 se anuncien correctamente|

### Comportamiento Spoke—Spoke en Fase 2

1. Spoke-1 envía tráfico hacia la LAN de Spoke-2 (`10.7.2.64/27`)
2. La tabla de rutas de Spoke-1 indica que el próximo salto es el Hub (`10.7.2.193`)
3. Spoke-1 consulta NHRP: "¿cuál es la IP pública de `10.7.2.195`?"
4. El Hub responde con la IP pública de Spoke-2 (`200.7.2.10`)
5. Spoke-1 construye un túnel IPSec directo hacia `200.7.2.10`
6. CEF reenvía el paquete directamente a Spoke-2 — **sin pasar por el Hub**
7. La **ruta en la tabla de enrutamiento sigue apuntando al Hub** (diferencia con Fase 3)

---

## Verificación

### Estado general del DMVPN

```
show dmvpn
```

`[ESPACIO PARA CAPTURA: show dmvpn en R-HUB — Spokes en estado UP con atributo D]`

**Explicación:** La columna `Attrb` muestra `S` (estático) para entradas configuradas manualmente y `D` (dinámico) para los Spokes que se registraron automáticamente. El estado debe ser `UP` para todos los peers activos.

### Tabla NHRP

```
show ip nhrp
```

`[ESPACIO PARA CAPTURA: show ip nhrp en R-HUB — mapeo de IPs de túnel a IPs públicas]`

**Explicación:** Cada entrada muestra la IP de túnel del Spoke, su IP pública (NBMA) y el flag `dynamic` indicando que fue registrado automáticamente. Después de un ping entre Spokes, aparecen entradas adicionales con flag `implicit` correspondientes al túnel dinámico negociado.

### Estado IPSec (Fase 1 y 2)

```
show crypto isakmp sa
show crypto ipsec sa
```

`[ESPACIO PARA CAPTURA: show crypto isakmp sa — estado QM_IDLE entre Hub y Spokes]`

`[ESPACIO PARA CAPTURA: show crypto ipsec sa — contadores de paquetes encapsulados]`

### Vecinos EIGRP

```
show ip eigrp neighbors
```

`[ESPACIO PARA CAPTURA: show ip eigrp neighbors — R-HUB con Spoke-1 y Spoke-2 como vecinos]`

**Explicación:** El Hub debe mostrar dos vecinos EIGRP (los dos Spokes). Cada Spoke debe mostrar un vecino (el Hub). La adyacencia EIGRP se forma sobre la red de túnel (`10.7.2.192/27`).

### Tabla de rutas

```
show ip route
```

`[ESPACIO PARA CAPTURA: show ip route en R-SPOKE-1 — rutas hacia LAN HUB y LAN SPOKE-2 vía Hub]`

**Explicación clave de Fase 2:** Las rutas hacia las LANs remotas muestran el Hub (`10.7.2.193`) como **próximo salto**, aunque el tráfico finalmente viaje directo al Spoke destino por CEF. Esta es la característica distintiva de Fase 2 frente a Fase 3, donde el próximo salto se actualiza al Spoke remoto.

### Prueba de conectividad

```
ping 10.7.2.194       ! desde R-HUB hacia Tunnel0 de Spoke-1
ping 10.7.2.195       ! desde R-HUB hacia Tunnel0 de Spoke-2
ping 10.7.2.34        ! desde LAN HUB hacia host en LAN SPOKE-1
ping 10.7.2.65        ! desde LAN HUB hacia host en LAN SPOKE-2
ping 10.7.2.65        ! desde LAN SPOKE-1 hacia host en LAN SPOKE-2 (túnel directo)
```

`[ESPACIO PARA CAPTURA: ping exitoso entre las 3 LANs]`

`[ESPACIO PARA CAPTURA: show ip nhrp después del ping Spoke—Spoke — entrada dinámica del peer remoto]`

---

## Notas y Consideraciones

- La PSK wildcard (`0.0.0.0 0.0.0.0`) es necesaria porque en DMVPN las IPs de los Spokes no son conocidas de antemano por el Hub. En IKEv2 (Fase 3) esto se maneja de forma más elegante con el `keyring`.
- El `tunnel mode gre multipoint` no tiene `tunnel destination` — este es el cambio fundamental respecto a un túnel GRE point-to-point; NHRP se encarga de resolver los destinos dinámicamente.
- Si `show dmvpn` muestra los Spokes en estado `NHRP` en lugar de `UP`, verificar que la autenticación NHRP (`ip nhrp authentication`) sea idéntica en todos los nodos.
- La red de túnel usa /27 (en lugar de /24) para evitar solapamiento con las LANs privadas que también pertenecen al espacio `10.7.2.0`.
- Nombres y clave personalizados con la matrícula del autor (`2025-0702`) para trazabilidad del laboratorio.
