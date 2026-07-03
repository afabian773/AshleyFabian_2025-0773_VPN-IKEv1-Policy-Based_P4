# VPN IPSec IKEv1 Site-to-Site Basado en Políticas

**Nombre:** Ashley Fabian Ortiz  
**Matrícula:** 2025-0773  
**Práctica:** P4  

---

## Objetivo

Configurar un túnel VPN IPSec IKEv1 Site-to-Site basado en políticas (Policy-Based) entre dos routers Cisco CSR1000v, permitiendo la comunicación cifrada entre dos redes LAN separadas a través de un router ISP que simula el enlace público de internet.

En una VPN basada en políticas, el tráfico interesante se define mediante una ACL (lista de control de acceso). Solo el tráfico que coincide con dicha ACL es cifrado y enviado a través del túnel IPSec. La negociación del túnel se realiza en dos fases:
- **Fase 1 (ISAKMP):** Establece un canal seguro para negociar los parámetros de seguridad (IKEv1 Main Mode).
- **Fase 2 (IPSec):** Negocia las Security Associations (SAs) para cifrar el tráfico de datos.

---

## Topología

```
[PC1]--[SW-A]--[R1]--------[ISP]--------[R2]--[SW-B]--[PC2]
        LAN-A   Gi3    Gi1       Gi2    Gi2    LAN-B
               25.7.73.9  25.7.73.1  25.7.73.5  25.7.73.6  25.7.73.17
```

---

## Direccionamiento IP

| Dispositivo | Interfaz       | Dirección IP    | Máscara         | Descripción       |
|-------------|----------------|-----------------|-----------------|-------------------|
| R1          | GigabitEthernet1 | 25.7.73.1     | 255.255.255.252 | WAN R1 → ISP      |
| ISP         | GigabitEthernet1 | 25.7.73.2     | 255.255.255.252 | WAN ISP → R1      |
| ISP         | GigabitEthernet2 | 25.7.73.5     | 255.255.255.252 | WAN ISP → R2      |
| R2          | GigabitEthernet2 | 25.7.73.6     | 255.255.255.252 | WAN R2 → ISP      |
| R1          | GigabitEthernet3 | 25.7.73.9     | 255.255.255.248 | Gateway LAN-A     |
| PC1         | eth0             | 25.7.73.10    | 255.255.255.248 | Host LAN-A        |
| R2          | GigabitEthernet3 | 25.7.73.17    | 255.255.255.248 | Gateway LAN-B     |
| PC2         | eth0             | 25.7.73.18    | 255.255.255.248 | Host LAN-B        |

### Subredes utilizadas (bloque 25.7.73.0/27)

| Subred            | Rango utilizable        | Uso            |
|-------------------|-------------------------|----------------|
| 25.7.73.0/30      | 25.7.73.1 – 25.7.73.2   | WAN R1 – ISP   |
| 25.7.73.4/30      | 25.7.73.5 – 25.7.73.6   | WAN ISP – R2   |
| 25.7.73.8/29      | 25.7.73.9 – 25.7.73.14  | LAN-A          |
| 25.7.73.16/29     | 25.7.73.17 – 25.7.73.22 | LAN-B          |

---

## Parámetros de configuración IPSec IKEv1

### Fase 1 — ISAKMP Policy

| Parámetro       | Valor              |
|-----------------|--------------------|
| Cifrado         | AES 256            |
| Hash            | SHA-256            |
| Autenticación   | Pre-shared key     |
| Grupo DH        | Group 14 (2048-bit)|
| Lifetime        | 86400 segundos     |
| Pre-shared key  | Cisco123!          |

### Fase 2 — IPSec Transform Set

| Parámetro       | Valor              |
|-----------------|--------------------|
| Cifrado ESP     | AES 256            |
| Integridad ESP  | SHA-256 HMAC       |
| Modo            | Tunnel             |

### Tráfico interesante (ACL)

| Router | Origen          | Destino         |
|--------|-----------------|-----------------|
| R1     | 25.7.73.8/29    | 25.7.73.16/29   |
| R2     | 25.7.73.16/29   | 25.7.73.8/29    |

---

## Scripts de configuración

### ISP

```
enable
configure terminal
hostname ISP

interface GigabitEthernet1
 ip address 25.7.73.2 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2
 ip address 25.7.73.5 255.255.255.252
 no shutdown
 exit

end
write memory
```

> El ISP no tiene rutas hacia las LANs internas. Solo conoce sus interfaces directamente conectadas, simulando el comportamiento real de un proveedor de internet.

### R1

```
enable
configure terminal
hostname R1

interface GigabitEthernet1
 ip address 25.7.73.1 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.9 255.255.255.248
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.2

! --- Fase 1: ISAKMP Policy ---
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
 exit

crypto isakmp key Cisco123! address 25.7.73.6

! --- Fase 2: Transform Set ---
crypto ipsec transform-set TS-P1 esp-aes 256 esp-sha256-hmac
 mode tunnel
 exit

! --- ACL de tráfico interesante ---
access-list 101 permit ip 25.7.73.8 0.0.0.7 25.7.73.16 0.0.0.7

! --- Crypto Map ---
crypto map CMAP-P1 10 ipsec-isakmp
 set peer 25.7.73.6
 set transform-set TS-P1
 match address 101
 exit

! --- Aplicar crypto map a interfaz WAN ---
interface GigabitEthernet1
 crypto map CMAP-P1
 exit

end
write memory
```

### R2

```
enable
configure terminal
hostname R2

interface GigabitEthernet2
 ip address 25.7.73.6 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.17 255.255.255.248
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.5

! --- Fase 1: ISAKMP Policy ---
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
 exit

crypto isakmp key Cisco123! address 25.7.73.1

! --- Fase 2: Transform Set ---
crypto ipsec transform-set TS-P1 esp-aes 256 esp-sha256-hmac
 mode tunnel
 exit

! --- ACL de tráfico interesante ---
access-list 101 permit ip 25.7.73.16 0.0.0.7 25.7.73.8 0.0.0.7

! --- Crypto Map ---
crypto map CMAP-P1 10 ipsec-isakmp
 set peer 25.7.73.1
 set transform-set TS-P1
 match address 101
 exit

! --- Aplicar crypto map a interfaz WAN ---
interface GigabitEthernet2
 crypto map CMAP-P1
 exit

end
write memory
```

### SW-A y SW-B (IOSvL2)

```
enable
configure terminal

interface GigabitEthernet0/0
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

end
write memory
```

### PC1 (VPCS)

```
ip 25.7.73.10 255.255.255.248 25.7.73.9
save
```

### PC2 (VPCS)

```
ip 25.7.73.18 255.255.255.248 25.7.73.17
save
```

---

## Verificación y pruebas

### Comandos de verificación

```
! Verificar fase 1 (ISAKMP SA)
show crypto isakmp sa

! Verificar fase 2 (IPSec SA) y contadores de paquetes
show crypto ipsec sa

! Verificar crypto map aplicado
show crypto map

! Verificar ACL y sus matches
show access-lists 101
```

### Resultados esperados

**show crypto isakmp sa:**
```
dst             src             state          conn-id status
25.7.73.6       25.7.73.1       QM_IDLE           1001 ACTIVE
```

**show crypto ipsec sa (fragmento):**
```
#pkts encaps: 6, #pkts encrypt: 6, #pkts digest: 6
#pkts decaps: 6, #pkts decrypt: 6, #pkts verify: 6
Status: ACTIVE(ACTIVE)
```

**Ping PC1 → PC2:**
```
PC1> ping 25.7.73.18
84 bytes from 25.7.73.18 icmp_seq=1 ttl=62 time=147.758 ms
84 bytes from 25.7.73.18 icmp_seq=2 ttl=62 time=61.479 ms
```

**Ping PC2 → PC1:**
```
PC2> ping 25.7.73.10
84 bytes from 25.7.73.10 icmp_seq=1 ttl=62 time=99.120 ms
84 bytes from 25.7.73.10 icmp_seq=2 ttl=62 time=75.428 ms
```

---

## Capturas de pantalla

> Insertar aquí las capturas de:
> 1. Topología completa en GNS3 con nombre y matrícula visible

![](./Topologia.png)
> 2. `show crypto isakmp sa` en R1 mostrando QM_IDLE ACTIVE

![](./Foto-1-R1.png)
> 3. `show crypto ipsec sa` en R1 mostrando contadores de paquetes cifrados/descifrados

![](./Foto-2-R1.png)
> 4. Ping exitoso de PC1 a PC2

![](./Ping-PC1.png)
> 5. Ping exitoso de PC2 a PC1

![](./Ping-PC2.png)
---

## Conclusión

Se configuró exitosamente una VPN IPSec IKEv1 Site-to-Site basada en políticas entre R1 y R2. La fase 1 (ISAKMP) negoció el canal seguro usando AES-256, SHA-256 y DH Group 14. La fase 2 (IPSec) estableció las Security Associations usando el transform set ESP-AES-256/SHA-256 en modo túnel. El tráfico entre LAN-A (25.7.73.8/29) y LAN-B (25.7.73.16/29) viaja cifrado a través del ISP, verificado mediante los contadores de encaps/decaps y los pings exitosos entre PC1 y PC2.
