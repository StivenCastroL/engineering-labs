# Lab 01 – Cisco Catalyst SD-WAN Overlay Bring-up

## Información general

| Campo | Valor |
|------|------|
| Lab | 01 |
| Tecnología | Cisco Catalyst SD-WAN |
| Nivel | Intermedio |
| Entorno | EVE-NG |
| Fecha | YYYY-MM-DD |

---

# 1. Objetivo del laboratorio

Construir un entorno funcional de Cisco Catalyst SD-WAN y validar el proceso completo de incorporación de dispositivos WAN Edge al fabric SD-WAN.

El laboratorio busca demostrar:

- formación del plano de control
- onboarding de routers WAN Edge
- establecimiento de túneles de overlay
- conectividad entre sedes a través del fabric SD-WAN

---

# 2. Contexto técnico

Cisco Catalyst SD-WAN se basa en una arquitectura centralizada que separa:

- plano de gestión
- plano de control
- plano de datos

Los componentes principales del sistema son:

| Componente | Función |
|------------|---------|
| SD-WAN Manager | Gestión centralizada |
| SD-WAN Validator | Orquestación y autenticación |
| SD-WAN Controller | Plano de control |
| WAN Edge | Routers de sucursal |

Los WAN Edge establecen túneles seguros sobre una red de transporte (underlay) para formar el **overlay SD-WAN**.

Fuentes oficiales:

- https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/sdwan-xe-gs-book/cisco-sd-wan-overlay-network-bringup.html
- https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/system-interface/systems-interfaces-book.html

---

# 3. Arquitectura del laboratorio

## Topología del laboratorio

![SD-WAN Overlay Topology](../../../../../diagrams/networking/cisco/catalyst-sdwan/lab-01/sdwan-lab01-overlay-topology.png)

La siguiente topología representa el entorno desplegado en EVE-NG para validar el funcionamiento básico del fabric Cisco Catalyst SD-WAN.

## Componentes

| Nodo | Rol |
|-----|-----|
| vManage | Gestión |
| vBond | Orchestrator |
| vSmart | Control plane |
| Edge-1 | Sucursal 1 |
| Edge-2 | Sucursal 2 |

## Redes

| Red | Uso |
|----|----|
| VPN 512 | Gestión |
| VPN 0 | Transporte |
| VPN 1 | LAN Site 1 |
| VPN 1 | LAN Site 2 |

---

# 4. Topology and Addressing Plan

## 4.1 System IP y Site ID

| Nodo | System-IP | Site-ID | Rol |
|-----|-----------|--------|------|
| vManage | 1.1.1.1 | 1 | Management |
| vBond | 1.1.1.2 | 1 | Orchestrator |
| vSmart | 1.1.1.3 | 1 | Control Plane |
| Edge-1 | 1.1.1.11 | 100 | Branch Site 1 |
| Edge-2 | 1.1.1.12 | 200 | Branch Site 2 |

---

## 4.2 Management Network (VPN 512)

| Dispositivo | Interfaz | Dirección IP | Gateway |
|-------------|-----------|-------------|---------|
| vManage | eth1 | 10.30.30.45 | 10.30.30.1 |
| vBond | eth1 | 10.30.30.46 | 10.30.30.1 |
| vSmart | eth1 | 10.30.30.47 | 10.30.30.1 |

Red de gestión:

10.30.30.0/24

Uso: red de administración de los control components.

---

## 4.3 Transport Network (VPN 0)

| Dispositivo | Interfaz | Dirección IP |
|-------------|-----------|-------------|
| vManage | eth0 | 10.10.10.1 |
| vBond | eth0 | 10.10.10.2 |
| vSmart | eth0 | 10.10.10.3 |
| Edge-1 | Gi1 | 10.10.10.11 |
| Edge-2 | Gi1 | 10.10.10.12 |

Gateway del transporte:

10.10.10.254

Red de transporte:

10.10.10.0/24

Uso: underlay transport utilizado para formar el overlay SD-WAN.

---

## 4.4 Service VPN (LAN Sites)

### Site 1

| Dispositivo | Interfaz | Red |
|-------------|-----------|------|
| Edge-1 | Gi2 | 192.168.10.0/24 |

### Site 2

| Dispositivo | Interfaz | Red |
|-------------|-----------|------|
| Edge-2 | Gi2 | 192.168.20.0/24 |

---

## 4.5 VPN Segmentación

| VPN | Uso |
|----|----|
| VPN 512 | Management plane |
| VPN 0 | Transport underlay |
| VPN 1 | Service VPN (LAN usuarios) |

---

# 5. Procedimiento

## 5.1 Deployment Sequence

De acuerdo con la documentación oficial de Cisco, el proceso de inicialización del overlay SD-WAN sigue una secuencia específica de arranque de los control components.

Fuente:

https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/sdwan-xe-gs-book/cisco-sd-wan-overlay-network-bringup.html

Orden recomendado de despliegue:

1. Start SD-WAN Manager (vManage)
2. Start SD-WAN Validator (vBond)
3. Start SD-WAN Controller (vSmart)
4. Onboard WAN Edge routers

Este orden permite que los componentes de control se autentiquen entre sí antes de que los routers WAN Edge intenten unirse al fabric SD-WAN.

---

## 5.2 vManage Bootstrap Configuration

El primer componente desplegado en el laboratorio es el **Cisco SD-WAN Manager (vManage)**.

Según la documentación oficial de Cisco, todo dispositivo SD-WAN requiere al menos tres bloques de configuración inicial:

- system
- vpn 0
- vpn 512

Fuente:

https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/system-interface/systems-interfaces-book.html

### Parámetros del sistema

| Parámetro | Valor |
|----------|------|
| Hostname | vmanage |
| System IP | 1.1.1.1 |
| Site ID | 1 |
| Organization Name | Lab_Stiven_SDWAN |
| vBond Address | 10.10.10.2 |

### Validaciones iniciales

Verificar estado del sistema:


show system status


Verificar conectividad hacia la red de transporte:


ping 10.10.10.254


Verificar conectividad hacia la red de gestión:


ping 10.30.30.1


Estas validaciones confirman que el dispositivo tiene conectividad en el underlay antes de continuar con el despliegue del resto de control components.

---

# 6. Validaciones técnicas

Las validaciones incluyen:

- estado de control connections
- estado de túneles overlay
- tabla de rutas del fabric
- pruebas de conectividad entre sedes

---

# 7. Resultados obtenidos

El sistema SD-WAN se levantó correctamente y se verificó conectividad entre:

- 192.168.10.0/24
- 192.168.20.0/24

a través del overlay SD-WAN.

---

# 8. Problemas encontrados

(Se documentarán aquí los errores encontrados durante el laboratorio.)

---

# 9. Lecciones aprendidas

Este laboratorio permitió comprender:

- el proceso de onboarding de routers SD-WAN
- la diferencia entre underlay y overlay
- el rol de cada componente del sistema

---

# 10. Mejores prácticas

- Validar siempre conectividad underlay antes de iniciar onboarding.
- Mantener consistencia en `organization-name`, `site-id` y `system-ip`.
- Verificar reachability entre control components antes de incorporar edges.

---

# 11. Próximos experimentos

Lab 02 – Templates y automatización de configuración en Cisco Catalyst SD-WAN.
