# DMVPN Fase 3 — Hub and Spoke punto a multipunto (IKEv2)

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

Implementar una VPN **hub and spoke punto a multipunto** utilizando **DMVPN Fase 3** con **IPSec IKEv2**, logrando que los Spokes establezcan túneles directos entre sí con **actualización dinámica de la tabla de rutas** — a diferencia de Fase 2, donde la ruta siempre apunta al Hub como próximo salto.

La combinación de Fase 3 con IKEv2 representa el estado del arte en DMVPN: mayor eficiencia en el plano de datos (el Hub queda completamente fuera del camino de reenvío entre Spokes), mayor seguridad en la negociación (IKEv2 en 4 mensajes, cookies anti-DoS, PSK por peer en keyring) y mayor escalabilidad.

---

## Contexto Teórico

### DMVPN — Recapitulación

**DMVPN (Dynamic Multipoint VPN)** combina tres tecnologías para crear una VPN escalable y dinámica:

- **mGRE:** Interfaz de túnel sin destino fijo que acepta múltiples peers simultáneamente
- **NHRP:** Protocolo de resolución que mapea IPs de túnel a IPs públicas, permitiendo que los Spokes se descubran entre sí
- **IPSec:** Capa de cifrado aplicada sobre el túnel mGRE con `tunnel protection`

En Fase 3, NHRP adquiere un rol adicional: no solo resuelve IPs, sino que también **instala rutas host `/32`** en la tabla de enrutamiento del Spoke origen cuando se detecta tráfico que debería ir directo al Spoke destino.

### El mecanismo de Fase 3 paso a paso

1. Spoke-1 envía tráfico hacia la LAN de Spoke-2 (`10.7.2.64/27`)
2. La ruta en Spoke-1 apunta al Hub (`10.7.2.193`) — igual que en Fase 2
3. El paquete llega al Hub
4. El Hub detecta que el destino final (`10.7.2.195`) es un Spoke registrado en NHRP
5. Gracias a `ip nhrp redirect`, el Hub envía un **NHRP Traffic Indication** a Spoke-1
6. Spoke-1 recibe el Traffic Indication, consulta NHRP y obtiene la IP pública de Spoke-2
7. NHRP instala una **ruta host `/32`** (`10.7.2.195/32 via 10.7.2.195`) en la tabla de rutas de Spoke-1
8. Spoke-1 construye el túnel IPSec directo hacia Spoke-2 (`200.7.2.10`)
9. El tráfico subsiguiente va **directamente de Spoke-1 a Spoke-2** — el Hub no interviene

La diferencia clave con Fase 2: en el paso 7, **la ruta se actualiza**. El próximo salto deja de ser el Hub y pasa a ser el Spoke remoto. Esto es posible gracias a `ip nhrp shortcut` en los Spokes, que autoriza a NHRP a instalar estas rutas en la tabla de enrutamiento.

---

## DMVPN Fase 3 vs Fase 2

### Diferencia operativa

| Aspecto                     | Fase 2 (IKEv1)                         | Fase 3 (IKEv2)                            |
| --------------------------- | -------------------------------------- | ----------------------------------------- |
| Túnel Spoke—Spoke           | Directo (nivel CEF)                    | Directo (nivel tabla de rutas)            |
| Próximo salto Spoke—Spoke   | Siempre el Hub                         | Se actualiza al Spoke remoto              |
| Cómo se redirige el tráfico | CEF consulta NHRP y reenvía            | NHRP instala ruta /32 en RIB              |
| Primer paquete Spoke—Spoke  | Pasa por el Hub                        | Pasa por el Hub                           |
| Paquetes subsiguientes      | Directo (CEF)                          | Directo (ruta /32 actualizada)            |
| Escalabilidad en el Hub     | Media — CEF procesa redireccionamiento | Alta — Hub completamente fuera del camino |
| Comando exclusivo Hub       | Ninguno                                | `ip nhrp redirect`                        |
| Comando exclusivo Spokes    | Ninguno                                | `ip nhrp shortcut`                        |

### ¿Cuándo usar cada fase?

| Escenario                                                    | Fase recomendada                  |
| ------------------------------------------------------------ | --------------------------------- |
| Red pequeña, IPs estáticas                                   | Fase 1 o site-to-site tradicional |
| Spokes con IPs dinámicas, tráfico mayoritariamente Hub—Spoke | Fase 2                            |
| Alta frecuencia de comunicación Spoke—Spoke, muchos sitios   | **Fase 3**                        |
| IKEv2 requerido por política de seguridad                    | **Fase 3**                        |

---

## IKEv2 en DMVPN

En DMVPN, los Spokes pueden tener IPs públicas dinámicas o no conocidas de antemano. IKEv2 maneja esto con una **PSK wildcard en el keyring** (`address 0.0.0.0 0.0.0.0`), que permite autenticar cualquier peer que presente la clave correcta. Esto es funcionalmente equivalente a la PSK wildcard de IKEv1, pero con las ventajas propias de IKEv2:

|Aspecto|IKEv1 en DMVPN|IKEv2 en DMVPN|
|---|---|---|
|PSK wildcard|`crypto isakmp key KEY address 0.0.0.0 0.0.0.0`|`peer ANY / address 0.0.0.0 0.0.0.0` en keyring|
|Negociación|6–9 mensajes|4 mensajes|
|Protección DoS|Ninguna|Cookies IKEv2|
|Vinculación al perfil IPSec|Implícita|Explícita: `set ikev2-profile` en el IPSec profile|
|Verificación|`show crypto isakmp sa`|`show crypto ikev2 sa`|
|Estado esperado|`QM_IDLE`|`READY`|

---

## Topología



```
                         HUB (R-HUB)
                       10.7.2.0/27 LAN
                       200.7.2.1/30 WAN
                       10.7.2.193/27 Tunnel0
                       [ip nhrp redirect]
                            |
                           ISP
                          /   \
               200.7.2.4/30   200.7.2.8/30
                    /               \
             R-SPOKE-1           R-SPOKE-2
          10.7.2.32/27 LAN    10.7.2.64/27 LAN
          200.7.2.6/30 WAN    200.7.2.10/30 WAN
          10.7.2.194/27 T0    10.7.2.195/27 T0
          [ip nhrp shortcut]  [ip nhrp shortcut]
          <--- ruta /32 instalada por NHRP --->
          <====== tunel directo IKEv2 ========>
```

### Tabla de Direccionamiento

|Dispositivo|Interfaz|Dirección IP|Máscara|Descripción|
|---|---|---|---|---|
|ISP|e0/0|200.7.2.2|255.255.255.252|Enlace a R-HUB|
|ISP|e0/1|200.7.2.5|255.255.255.252|Enlace a R-SPOKE-1|
|ISP|e0/2|200.7.2.9|255.255.255.252|Enlace a R-SPOKE-2|
|R-HUB|e0/0|200.7.2.1|255.255.255.252|WAN (hacia ISP)|
|R-HUB|e0/1|10.7.2.1|255.255.255.224|LAN HUB|
|R-HUB|Tunnel0|10.7.2.193|255.255.255.224|mGRE — NHS + redirect|
|R-SPOKE-1|e0/0|200.7.2.6|255.255.255.252|WAN (hacia ISP)|
|R-SPOKE-1|e0/1|10.7.2.33|255.255.255.224|LAN SPOKE-1|
|R-SPOKE-1|Tunnel0|10.7.2.194|255.255.255.224|mGRE — NHC + shortcut|
|R-SPOKE-2|e0/0|200.7.2.10|255.255.255.252|WAN (hacia ISP)|
|R-SPOKE-2|e0/1|10.7.2.65|255.255.255.224|LAN SPOKE-2|
|R-SPOKE-2|Tunnel0|10.7.2.195|255.255.255.224|mGRE — NHC + shortcut|

### Red de túnel mGRE

|Red|Rango usable|Descripción|
|---|---|---|
|10.7.2.192/27|.193 → .222|Interfaz Tunnel0 de Hub y Spokes|

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

`[ESPACIO PARA CAPTURA: ping exitoso de cada router hacia el ISP antes de aplicar DMVPN]`

---

## Configuración DMVPN Fase 3 con IKEv2

### R-HUB

```
conf t

! ----- FASE 1: IKEv2 -----
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN
exit

crypto ikev2 keyring KEY-DMVPN
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key 2025-0702
 exit
exit

crypto ikev2 profile PROF-DMVPN
 match identity remote address 0.0.0.0 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local KEY-DMVPN
exit

! ----- FASE 2: IPSec -----
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit

crypto ipsec profile DMVPN-PROF
 set transform-set TS-DMVPN
 set ikev2-profile PROF-DMVPN
exit

! ----- INTERFAZ TUNNEL mGRE -----
interface Tunnel0
 ip address 10.7.2.193 255.255.255.224
 ip nhrp network-id 1
 ip nhrp authentication 2025-0702
 ip nhrp map multicast dynamic
 ip nhrp redirect
 no ip split-horizon eigrp 100
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

`[ESPACIO PARA CAPTURA: configuración DMVPN Fase 3 aplicada en R-HUB — terminal]`

### R-SPOKE-1

```
conf t

! ----- FASE 1: IKEv2 -----
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN
exit

crypto ikev2 keyring KEY-DMVPN
 peer HUB
  address 0.0.0.0 0.0.0.0
  pre-shared-key 2025-0702
 exit
exit

crypto ikev2 profile PROF-DMVPN
 match identity remote address 0.0.0.0 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local KEY-DMVPN
exit

! ----- FASE 2: IPSec -----
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit

crypto ipsec profile DMVPN-PROF
 set transform-set TS-DMVPN
 set ikev2-profile PROF-DMVPN
exit

! ----- INTERFAZ TUNNEL mGRE -----
interface Tunnel0
 ip address 10.7.2.194 255.255.255.224
 ip nhrp network-id 1
 ip nhrp authentication 2025-0702
 ip nhrp nhs 10.7.2.193
 ip nhrp map 10.7.2.193 200.7.2.1
 ip nhrp map multicast 200.7.2.1
 ip nhrp shortcut
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

`[ESPACIO PARA CAPTURA: configuración DMVPN Fase 3 aplicada en R-SPOKE-1 — terminal]`

### R-SPOKE-2

```
conf t

! ----- FASE 1: IKEv2 -----
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN
exit

crypto ikev2 keyring KEY-DMVPN
 peer HUB
  address 0.0.0.0 0.0.0.0
  pre-shared-key 2025-0702
 exit
exit

crypto ikev2 profile PROF-DMVPN
 match identity remote address 0.0.0.0 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local KEY-DMVPN
exit

! ----- FASE 2: IPSec -----
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit

crypto ipsec profile DMVPN-PROF
 set transform-set TS-DMVPN
 set ikev2-profile PROF-DMVPN
exit

! ----- INTERFAZ TUNNEL mGRE -----
interface Tunnel0
 ip address 10.7.2.195 255.255.255.224
 ip nhrp network-id 1
 ip nhrp authentication 2025-0702
 ip nhrp nhs 10.7.2.193
 ip nhrp map 10.7.2.193 200.7.2.1
 ip nhrp map multicast 200.7.2.1
 ip nhrp shortcut
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

`[ESPACIO PARA CAPTURA: configuración DMVPN Fase 3 aplicada en R-SPOKE-2 — terminal]`

---

## Explicación de los Componentes

### IKEv2 Proposal

|Parámetro|Valor|Descripción|
|---|---|---|
|`encryption aes-cbc-256`|AES-CBC 256 bits|Algoritmo de encriptación|
|`integrity sha256`|SHA-256|Algoritmo de integridad|
|`group 14`|DH Group 14 (2048-bit)|Grupo Diffie-Hellman|

### IKEv2 Keyring — wildcard para DMVPN

|Parámetro|Descripción|
|---|---|
|`address 0.0.0.0 0.0.0.0`|Acepta cualquier IP como peer — necesario porque los Spokes pueden tener IPs dinámicas o no conocidas|
|`pre-shared-key 2025-0702`|PSK definida dentro del keyring por peer (ventaja sobre IKEv1 donde era global)|

### IKEv2 Profile

|Parámetro|Descripción|
|---|---|
|`match identity remote address 0.0.0.0 0.0.0.0`|Coincide con cualquier peer — equivalente a la wildcard del keyring|
|`authentication remote/local pre-share`|Ambos extremos se autentican con PSK|
|`keyring local KEY-DMVPN`|Referencia el keyring con la PSK wildcard|

### IPSec Profile — diferencia clave con Fase 2

|Parámetro|Fase 2 IKEv1|Fase 3 IKEv2|
|---|---|---|
|Contenido del perfil|Solo `set transform-set`|`set transform-set` + `set ikev2-profile PROF-DMVPN`|
|Modo de transporte|`mode transport`|`mode transport` (igual)|

### Interfaz Tunnel0 — comandos exclusivos de Fase 3

|Comando|Rol|Descripción|
|---|---|---|
|`ip nhrp redirect`|Solo Hub|Cuando el Hub recibe tráfico entre Spokes, envía un NHRP Traffic Indication al Spoke origen para que construya la ruta directa|
|`ip nhrp shortcut`|Solo Spokes|Autoriza a NHRP a instalar rutas host `/32` en la tabla de enrutamiento cuando recibe un Traffic Indication del Hub|

### Resto de comandos Tunnel0 (idénticos a Fase 2)

|Comando|Descripción|
|---|---|
|`tunnel mode gre multipoint`|Túnel mGRE sin destino fijo|
|`ip nhrp network-id 1`|ID de red NHRP — idéntico en todos los nodos|
|`ip nhrp authentication 2025-0702`|Contraseña NHRP|
|`ip nhrp map multicast dynamic`|Solo Hub — registro dinámico de Spokes|
|`ip nhrp nhs 10.7.2.193`|Solo Spokes — apunta al Hub como NHS|
|`ip nhrp map 10.7.2.193 200.7.2.1`|Solo Spokes — mapeo estático del Hub|
|`tunnel protection ipsec profile DMVPN-PROF`|Cifrado IKEv2 sobre todo el tráfico del túnel|

---

## Comparativa Fase 2 vs Fase 3

| Aspecto                            | Fase 2 (IKEv1)                               | Fase 3 (IKEv2)                          |
| ---------------------------------- | -------------------------------------------- | --------------------------------------- |
| Protocolo IKE                      | IKEv1                                        | IKEv2                                   |
| Mensajes negociación               | 6–9                                          | 4                                       |
| PSK wildcard                       | `crypto isakmp key ... 0.0.0.0 0.0.0.0`      | `keyring / address 0.0.0.0 0.0.0.0`     |
| IPSec profile                      | Solo transform-set                           | Transform-set + `set ikev2-profile`     |
| Comando Hub exclusivo              | Ninguno                                      | `ip nhrp redirect`                      |
| Comando Spoke exclusivo            | Ninguno                                      | `ip nhrp shortcut`                      |
| Ruta Spoke—Spoke                   | Próximo salto = Hub                          | Próximo salto = Spoke remoto (ruta /32) |
| Hub en camino de datos Spoke—Spoke | Sí (primer y único paquete vía Hub para CEF) | No (desde el segundo paquete)           |
| Verificación IKE                   | `show crypto isakmp sa` → `QM_IDLE`          | `show crypto ikev2 sa` → `READY`        |
| Entrada NHRP Spoke—Spoke           | `implicit`                                   | `implicit` + `shortcut`                 |
| Escalabilidad                      | Media                                        | Alta                                    |

---

## Verificación

### Estado general del DMVPN

```
show dmvpn
```

`[ESPACIO PARA CAPTURA: show dmvpn en R-HUB — Spokes en estado UP]`

**Explicación:** Igual que en Fase 2, los Spokes aparecen con atributo `D` (dinámico). Tras un ping entre Spokes, aparece una entrada adicional con atributo `DT3` (dynamic, Fase 3 Traffic Indication procesado), confirmando que el mecanismo de Fase 3 está activo.

### Tabla NHRP — diferencia clave con Fase 2

```
show ip nhrp
```

`[ESPACIO PARA CAPTURA: show ip nhrp en R-SPOKE-1 antes del ping Spoke—Spoke]`

`[ESPACIO PARA CAPTURA: show ip nhrp en R-SPOKE-1 después del ping — entrada con flag shortcut]`

**Explicación:** Después del ping entre Spokes, `show ip nhrp` en R-SPOKE-1 muestra una entrada para `10.7.2.195` con flags `implicit` y `shortcut`. El flag `shortcut` confirma que NHRP instaló la ruta directa en la tabla de enrutamiento — esto es lo que diferencia Fase 3 de Fase 2.

### Estado IKEv2

```
show crypto ikev2 sa
```

`[ESPACIO PARA CAPTURA: show crypto ikev2 sa — estado READY entre Hub y Spokes]`

**Explicación:** El estado `READY` (en lugar del `QM_IDLE` de IKEv1) confirma que las SAs IKEv2 están activas. Tras el primer ping Spoke—Spoke aparece una SA adicional directamente entre los Spokes.

### Estado IPSec

```
show crypto ipsec sa
```

`[ESPACIO PARA CAPTURA: show crypto ipsec sa — contadores de paquetes encapsulados]`

### Vecinos EIGRP

```
show ip eigrp neighbors
```

`[ESPACIO PARA CAPTURA: show ip eigrp neighbors — vecindades activas]`

### Tabla de rutas — la diferencia definitiva entre Fase 2 y Fase 3

```
show ip route
```

`[ESPACIO PARA CAPTURA: show ip route en R-SPOKE-1 ANTES del ping Spoke—Spoke]`

`[ESPACIO PARA CAPTURA: show ip route en R-SPOKE-1 DESPUÉS del ping — ruta /32 instalada por NHRP]`

**Explicación:** Esta es la verificación más importante de Fase 3. Antes del primer ping, la ruta hacia la LAN de Spoke-2 apunta al Hub (`via 10.7.2.193`). Después del primer ping, aparece una nueva entrada instalada por NHRP: `10.7.2.195/32 via 10.7.2.195` — el próximo salto ahora es el propio Spoke-2. Esta entrada **no existe en Fase 2**, donde el próximo salto siempre es el Hub.

### Prueba de conectividad

```
ping 10.7.2.194       ! desde R-HUB hacia Tunnel0 de Spoke-1
ping 10.7.2.195       ! desde R-HUB hacia Tunnel0 de Spoke-2
ping 10.7.2.34        ! desde LAN HUB hacia host en LAN SPOKE-1
ping 10.7.2.65        ! desde LAN HUB hacia host en LAN SPOKE-2
ping 10.7.2.65        ! desde LAN SPOKE-1 hacia LAN SPOKE-2 (fuerza el Traffic Indication)
```

`[ESPACIO PARA CAPTURA: ping exitoso entre las 3 LANs]`

`[ESPACIO PARA CAPTURA: show ip nhrp y show ip route en R-SPOKE-1 después del ping Spoke—Spoke — confirmar ruta /32 y flag shortcut]`

---

## Notas y Consideraciones

- `ip nhrp redirect` y `ip nhrp shortcut` son los únicos dos comandos que diferencian Fase 3 de Fase 2 en la interfaz Tunnel0. El resto de la configuración es idéntica.
- La ruta `/32` instalada por NHRP es temporal (tiene un timeout configurable con `ip nhrp holdtime`). Si no hay tráfico entre los Spokes, la ruta expira y el siguiente paquete vuelve a pasar por el Hub para re-activar el mecanismo.
- El flag `DT3` en `show dmvpn` es exclusivo de Fase 3 — su ausencia tras un ping entre Spokes indica que `ip nhrp redirect` no está activo en el Hub o `ip nhrp shortcut` no está activo en los Spokes.
- `show crypto ikev2 sa` reemplaza a `show crypto isakmp sa` de IKEv1 — el estado esperado es `READY`, no `QM_IDLE`.
- La PSK wildcard en el keyring (`address 0.0.0.0 0.0.0.0`) es funcional para laboratorio. En producción, se recomienda usar certificados digitales o PSK por peer con IPs estáticas conocidas.
- Nombres y clave personalizados con la matrícula del autor (`2025-0702`) para trazabilidad del laboratorio.
