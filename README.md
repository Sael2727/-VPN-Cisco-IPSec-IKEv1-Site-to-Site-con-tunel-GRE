# 🔐 VPN Cisco IPSec IKEv1 Site-to-Site — Túnel GRE

<div align="center">

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![IPSec](https://img.shields.io/badge/IPSec-IKEv1-green?style=for-the-badge)
![GRE](https://img.shields.io/badge/Tunnel-GRE-purple?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-orange?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-red?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Configuración y validación de una **VPN Cisco IPSec IKEv1 Site-to-Site con túnel GRE** para permitir comunicación segura entre dos LANs remotas a través de un router ISP que simula la red pública. **GRE** crea el túnel lógico punto a punto entre R1 y R2, mientras **IPSec** cifra el tráfico GRE entre las direcciones WAN de ambos peers.

> 💡 **Clave técnica:** La ACL de IPSec no apunta directamente a las LANs, sino al tráfico **GRE** entre las IPs WAN (10.7.25.1 ↔ 10.7.25.5). GRE corresponde al protocolo IP **47**, visible en `show crypto ipsec sa`.

---

## 🗺️ Topología de Red

La topología conserva dos routers como peers VPN, un router ISP al centro y una LAN en cada extremo. El ISP no cifra tráfico, solo brinda alcance entre R1 y R2.

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara | Función |
|:-----------:|:--------:|:-------------:|---------|
| R1 | Ethernet0/0 | 10.7.25.1/30 | WAN hacia ISP |
| ISP | Ethernet0/0 | 10.7.25.2/30 | Enlace hacia R1 |
| ISP | Ethernet0/1 | 10.7.25.6/30 | Enlace hacia R2 |
| R2 | Ethernet0/0 | 10.7.25.5/30 | WAN hacia ISP |
| R1 | Ethernet0/1 | 10.7.25.65/27 | Gateway LAN-A |
| VPC1 | eth0 | 10.7.25.66/27 | Host LAN-A |
| R2 | Ethernet0/1 | 10.7.25.97/27 | Gateway LAN-B |
| VPC2 | eth0 | 10.7.25.98/27 | Host LAN-B |
| **R1** | **Tunnel0** | **10.7.25.133/30** | **Extremo GRE R1** |
| **R2** | **Tunnel0** | **10.7.25.134/30** | **Extremo GRE R2** |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Versión IKE | IKEv1 |
| Cifrado | AES 256 |
| Hash | SHA |
| Autenticación | Pre-Shared Key |
| Clave compartida | VPN12345 |
| Grupo Diffie-Hellman | Grupo 5 |
| Transform-set | TS-IKEV1: esp-aes 256 esp-sha-hmac |
| Crypto map | VPN-MAP |
| Tráfico protegido | GRE entre 10.7.25.1 y 10.7.25.5 |
| Ruta R1 → LAN-B | 10.7.25.96/27 vía Tunnel0 |
| Ruta R2 → LAN-A | 10.7.25.64/27 vía Tunnel0 |

---

## 🚀 Scripts de Configuración

### ISP
```cisco
hostname ISP
interface Ethernet0/0
 description ENLACE_A_R1
 ip address 10.7.25.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description ENLACE_A_R2
 ip address 10.7.25.6 255.255.255.252
 no shutdown
```

### R1 — Configuración Principal
```cisco
hostname R1
interface Ethernet0/0
 ip address 10.7.25.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 10.7.25.65 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.2

crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
crypto isakmp key VPN12345 address 10.7.25.5

crypto ipsec transform-set TS-IKEV1 esp-aes 256 esp-sha-hmac
 mode tunnel

ip access-list extended VPN-TRAFFIC
 permit gre host 10.7.25.1 host 10.7.25.5

crypto map VPN-MAP 10 ipsec-isakmp
 set peer 10.7.25.5
 set transform-set TS-IKEV1
 set pfs group5
 match address VPN-TRAFFIC

interface Ethernet0/0
 crypto map VPN-MAP

interface Tunnel0
 ip address 10.7.25.133 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.7.25.5

ip route 10.7.25.96 255.255.255.224 Tunnel0
```

### R2 — Configuración Principal
```cisco
hostname R2
interface Ethernet0/0
 ip address 10.7.25.5 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 10.7.25.97 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.6

crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
crypto isakmp key VPN12345 address 10.7.25.1

crypto ipsec transform-set TS-IKEV1 esp-aes 256 esp-sha-hmac
 mode tunnel

ip access-list extended VPN-TRAFFIC
 permit gre host 10.7.25.5 host 10.7.25.1

crypto map VPN-MAP 10 ipsec-isakmp
 set peer 10.7.25.1
 set transform-set TS-IKEV1
 set pfs group5
 match address VPN-TRAFFIC

interface Ethernet0/0
 crypto map VPN-MAP

interface Tunnel0
 ip address 10.7.25.134 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.7.25.1

ip route 10.7.25.64 255.255.255.224 Tunnel0
```

### Configuración de VPCs
```bash
# VPC1
ip 10.7.25.66 255.255.255.224 10.7.25.65

# VPC2
ip 10.7.25.98 255.255.255.224 10.7.25.97
```

---

## ✅ Verificación del Túnel

```cisco
show interface tunnel0
show crypto isakmp sa
show crypto ipsec sa
```

| Comando | Estado esperado |
|:-------:|------------------|
| `show interface tunnel0` | up/up — Tunnel protocol/transport GRE/IP |
| `show crypto isakmp sa` | QM_IDLE / ACTIVE |
| `show crypto ipsec sa` | Protocolo 47 (GRE) con encaps/decaps activos |

> 💡 **Nota sobre traceroute:** El mensaje "Destination port unreachable" al final del trace es normal en VPCS — usa paquetes UDP, y al llegar al host final este responde que el puerto no está disponible. No representa una falla de la VPN.

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Tunnel0 en R1 y R2 | ✅ up/up |
| Transporte del túnel | ✅ GRE/IP |
| Ruta hacia LAN remota | ✅ Vía Tunnel0 en ambos routers |
| Matches en ACL GRE | ✅ Confirmados en R1 y R2 |
| IKEv1 SA | ✅ QM_IDLE / ACTIVE |
| IPSec SA | ✅ Protocolo 47, encaps/decaps activos |
| Ping VPC1 → VPC2 | ✅ Exitoso |
| Ping VPC2 → VPC1 | ✅ Exitoso |

---

## 🔍 Explicación Técnica

GRE permite transportar tráfico de una LAN a otra mediante una interfaz **Tunnel0**. IPSec cifra ese tráfico GRE entre las direcciones WAN de R1 y R2. La evidencia clave que confirma el funcionamiento correcto:

- Las ACLs muestran coincidencias de tráfico **GRE**
- **Tunnel0** está en estado up/up con transporte GRE/IP
- Los pings entre VPC1 y VPC2 funcionan en ambas direcciones
- **IKEv1** aparece en QM_IDLE
- **IPSec** muestra encapsulación y decapsulación de paquetes con protocolo 47

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`SaelGerman_2025-0725_Script_GRE_IKEV1.txt`](SaelGerman_2025-0725_Script_GRE_IKEV1.txt) | Scripts de configuración Cisco IOS |
| [`SaelGerman_2025-0725_VPN-IPSec-IKEv1-Site-to-Site-con-tunel-GRE_P2.pdf`](SaelGerman_2025-0725_VPN-IPSec-IKEv1-Site-to-Site-con-tunel-GRE_P2.pdf) | Documentación técnica completa |

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología general de la VPN GRE + IPSec](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%201.%20Topolog%C3%ADa%20general%20de%20la%20VPN%20GRE%20%2B%20IPSec.png)
- 📸 [Figura 2 — Interfaces del ISP](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%202.%20Interfaces%20del%20ISP.png)
- 📸 [Figura 3 — Interfaces de R1](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%203.%20Interfaces%20de%20R1.png)
- 📸 [Figura 4 — Tunnel0 en R1](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%204.%20Tunnel0%20en%20R1.png)
- 📸 [Figura 5 — Crypto en R1](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%205.%20Crypto%20en%20R1.png)
- 📸 [Figura 6 — ACL GRE en R1](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%206.%20ACL%20GRE%20en%20R1.png)
- 📸 [Figura 7 — Ruta de R1 hacia LAN-B](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%207.%20Ruta%20de%20R1%20hacia%20LAN-B.png)
- 📸 [Figura 8 — Interfaces de R2](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%208.%20Interfaces%20de%20R2.png)
- 📸 [Figura 9 — Tunnel0 en R2](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%209.%20Tunnel0%20en%20R2.png)
- 📸 [Figura 10 — Crypto en R2](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2010.%20Crypto%20en%20R2.png)
- 📸 [Figura 11 — ACL GRE en R2](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2011.%20ACL%20GRE%20en%20R2.png)
- 📸 [Figura 12 — Ruta de R2 hacia LAN-A](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2012.%20Ruta%20de%20R2%20hacia%20LAN-A.png)
- 📸 [Figura 13 — Estado de Tunnel0 en R1](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2013.%20Estado%20de%20Tunnel0%20en%20R1.png)
- 📸 [Figura 14 — Estado de Tunnel0 en R2](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2014.%20Estado%20de%20Tunnel0%20en%20R2.png)
- 📸 [Figura 15 — Prueba desde VPC2 hacia VPC1](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2015.%20Prueba%20desde%20VPC2%20hacia%20VPC1.png)
- 📸 [Figura 16 — Prueba desde VPC1 hacia VPC2](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2016.%20Prueba%20desde%20VPC1%20hacia%20VPC2.png)
- 📸 [Figura 17 — ISAKMP en R1](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2017.%20ISAKMP%20en%20R1.png)
- 📸 [Figura 18 — ISAKMP en R2](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2018.%20ISAKMP%20en%20R2.png)
- 📸 [Figura 19 — IPSec SA en R1](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2019.%20IPSec%20SA%20en%20R1.png)
- 📸 [Figura 20 — IPSec SA en R2](VPN-Cisco-IPSec-IKEv1-Site-to-Site-con-tunel-GRE-capturas/Figura%2020.%20IPSec%20SA%20en%20R2.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_VPN-IPSec-IKEv1-Site-to-Site-con-tunel-GRE_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/iEcrhNfrAs4)

---

## 📚 Referencias

1. Cisco Systems. *Configuring a GRE Tunnel over IPSec*. Documentación oficial Cisco IOS.
2. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
