# Lab 01 – Cisco Catalyst SD-WAN Overlay Bring-up

## Información general

| Campo | Valor |
|-------|-------|
| Lab | 01 |
| Tecnología | Cisco Catalyst SD-WAN |
| Nivel | Intermedio |
| Entorno | EVE-NG |
| Fecha | 2026-03-14 |
| Estado | En progreso |

---

## 1. Objetivo del laboratorio

Construir un entorno funcional de Cisco Catalyst SD-WAN y validar el proceso completo de incorporación de dispositivos WAN Edge al fabric SD-WAN.

El laboratorio busca demostrar:

- Formación del plano de control entre SD-WAN Manager, Validator y Controller
- Onboarding de routers WAN Edge al fabric
- Establecimiento de túneles de overlay (IPsec + BFD)
- Conectividad end-to-end entre sedes a través del fabric SD-WAN

---

## 2. Contexto técnico

Cisco Catalyst SD-WAN se basa en una arquitectura centralizada que separa el plano de gestión, el plano de control y el plano de datos. Esta separación permite operar el fabric de forma centralizada mientras el data plane se distribuye entre los WAN Edge routers.

### Componentes del sistema

| Componente | Nombre legacy | Función |
|------------|--------------|---------|
| SD-WAN Manager | vManage | Gestión centralizada, GUI, APIs, templates, políticas |
| SD-WAN Validator | vBond | Orquestación y autenticación inicial. Primer punto de contacto de los WAN Edge |
| SD-WAN Controller | vSmart | Plano de control. Distribuye rutas vía OMP y aplica políticas |
| WAN Edge | cEdge / vEdge | Routers de sucursal. Forman los túneles IPsec del data plane |

!!! note "Nomenclatura"
    Cisco migró oficialmente a los nombres "SD-WAN Manager", "SD-WAN Validator" y "SD-WAN Controller" a partir de la versión 20.x. Los nombres legacy (vManage, vBond, vSmart) siguen siendo de uso común en la industria y aparecen en la CLI. Este documento usa la nomenclatura oficial y referencia los nombres legacy cuando es necesario para claridad.

### Flujo de autenticación del overlay bringup

El proceso de bringup del overlay sigue esta secuencia:

1. El WAN Edge arranca y contacta al **SD-WAN Validator** (vBond) en VPN 0
2. El Validator autentica al WAN Edge mediante certificados y valida el `organization-name`
3. El Validator informa al WAN Edge las direcciones de **SD-WAN Manager** y **SD-WAN Controller**
4. El WAN Edge establece túneles **DTLS/TLS** hacia el Manager y el Controller
5. El Controller establece sesiones **OMP** con el WAN Edge y distribuye rutas del service VPN
6. Los WAN Edge forman túneles **IPsec** entre sí (data plane) con monitoreo **BFD**

Este flujo es el núcleo de lo que este laboratorio valida.

### Fuentes oficiales

- [Cisco SD-WAN Overlay Network Bring-Up](https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/sdwan-xe-gs-book/cisco-sd-wan-overlay-network-bringup.html)
- [Cisco SD-WAN Systems and Interfaces](https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/system-interface/systems-interfaces-book.html)

---

## 3. Arquitectura del laboratorio

### Topología

![SD-WAN Overlay Topology](./diagrams/sdwan-lab01-overlay-topology.png)

### Componentes

| Nodo | Rol | Descripción |
|------|-----|-------------|
| SD-WAN Manager | Gestión | Plataforma centralizada de administración |
| SD-WAN Validator | Orchestrator | Punto de autenticación inicial para WAN Edges |
| SD-WAN Controller | Control plane | Distribuye rutas OMP y aplica políticas |
| Edge-1 | Branch Site 1 | Router WAN Edge de sucursal 1 |
| Edge-2 | Branch Site 2 | Router WAN Edge de sucursal 2 |

### Segmentación VPN

| VPN | Uso | Descripción |
|-----|-----|-------------|
| VPN 512 | Management plane | Red de gestión de los control components |
| VPN 0 | Transport underlay | Red de transporte para formar el overlay |
| VPN 1 | Service VPN | Tráfico de usuarios (LAN de cada site) |

!!! info "Sobre VPN 1"
    Ambos sites utilizan el mismo VPN ID (VPN 1) para el service VPN, pero con subnets diferentes (192.168.10.0/24 y 192.168.20.0/24). Esto es el comportamiento estándar de segmentación SD-WAN: el VPN ID identifica el segmento lógico, no la subnet específica.

---

## 4. Plan de direccionamiento

### 4.1 System IP y Site ID

| Nodo | System-IP | Site-ID | Rol |
|------|-----------|---------|-----|
| SD-WAN Manager | 1.1.1.1 | 1 | Management |
| SD-WAN Validator | 1.1.1.2 | 1 | Orchestrator |
| SD-WAN Controller | 1.1.1.3 | 1 | Control Plane |
| Edge-1 | 1.1.1.11 | 100 | Branch Site 1 |
| Edge-2 | 1.1.1.12 | 200 | Branch Site 2 |

### 4.2 Management Network (VPN 512)

| Dispositivo | Interfaz | Dirección IP | Gateway |
|-------------|----------|--------------|---------|
| SD-WAN Manager | eth1 | 10.30.30.45/24 | 10.30.30.1 |
| SD-WAN Validator | eth1 | 10.30.30.46/24 | 10.30.30.1 |
| SD-WAN Controller | eth1 | 10.30.30.47/24 | 10.30.30.1 |

### 4.3 Transport Network (VPN 0)

| Dispositivo | Interfaz | Dirección IP | Gateway |
|-------------|----------|--------------|---------|
| SD-WAN Manager | eth0 | 10.10.10.1/24 | 10.10.10.254 |
| SD-WAN Validator | eth0 | 10.10.10.2/24 | 10.10.10.254 |
| SD-WAN Controller | eth0 | 10.10.10.3/24 | 10.10.10.254 |
| Edge-1 | Gi1 | 10.10.10.11/24 | 10.10.10.254 |
| Edge-2 | Gi1 | 10.10.10.12/24 | 10.10.10.254 |

### 4.4 Service VPN (VPN 1)

| Site | Dispositivo | Interfaz | Red |
|------|-------------|----------|-----|
| Site 1 | Edge-1 | Gi2 | 192.168.10.0/24 |
| Site 2 | Edge-2 | Gi2 | 192.168.20.0/24 |

---

## 5. Hipótesis

> Si todos los control components (SD-WAN Manager, Validator y Controller) comparten el mismo `organization-name` y tienen reachability entre sí en VPN 0, entonces:
>
> 1. Los control connections DTLS/TLS se establecerán entre todos los componentes de forma automática
> 2. Los WAN Edge routers completarán el proceso de autenticación vía el Validator
> 3. Los WAN Edge establecerán sesiones OMP con el Controller
> 4. Se formarán túneles IPsec BFD entre los WAN Edge sin necesidad de políticas manuales
> 5. Las rutas del service VPN (VPN 1) se propagarán vía OMP, permitiendo conectividad entre 192.168.10.0/24 y 192.168.20.0/24

---

## 6. Procedimiento

### 6.1 Secuencia de despliegue

De acuerdo con la documentación oficial de Cisco, el overlay bringup sigue esta secuencia:

1. Iniciar SD-WAN Manager
2. Iniciar SD-WAN Validator
3. Iniciar SD-WAN Controller
4. Onboard WAN Edge routers

Este orden permite que los control components se autentiquen entre sí antes de que los WAN Edge intenten unirse al fabric.

> **Fuente:** [Cisco SD-WAN Overlay Network Bring-Up](https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/sdwan-xe-gs-book/cisco-sd-wan-overlay-network-bringup.html)

### 6.2 SD-WAN Manager bootstrap

El primer componente desplegado es el SD-WAN Manager. Todo dispositivo SD-WAN requiere al menos tres bloques de configuración inicial: `system`, `vpn 0` y `vpn 512`.

#### Parámetros del sistema

| Parámetro | Valor |
|-----------|-------|
| Hostname | vmanage |
| System IP | 1.1.1.1 |
| Site ID | 1 |
| Organization Name | Lab_Stiven_SDWAN |
| vBond Address | 10.10.10.2 |

#### Configuración

```
! TODO: Pegar configuración real del lab
```

#### Validaciones del Manager

```
show system status
```

```
ping 10.10.10.254
```

```
ping 10.30.30.1
```

Estas validaciones confirman que el Manager tiene conectividad en el underlay antes de continuar con el despliegue.

### 6.3 SD-WAN Validator bootstrap

<!-- TODO: Documentar bootstrap del Validator -->

### 6.4 SD-WAN Controller bootstrap

<!-- TODO: Documentar bootstrap del Controller -->

### 6.5 Validación del control plane

<!-- TODO: Documentar verificación de control connections entre los tres componentes -->

### 6.6 WAN Edge onboarding — Edge-1

<!-- TODO: Documentar onboarding de Edge-1 -->

### 6.7 WAN Edge onboarding — Edge-2

<!-- TODO: Documentar onboarding de Edge-2 -->

---

## 7. Validaciones técnicas

Las siguientes validaciones deben ejecutarse para confirmar que el overlay está operativo:

### Control plane

| Comando | Qué valida |
|---------|-----------|
| `show control connections` | Túneles DTLS/TLS entre todos los componentes |
| `show control local-properties` | Certificados, serial number, organization-name |
| `show omp peers` | Sesiones OMP entre WAN Edges y Controller |
| `show omp routes` | Rutas del service VPN aprendidas vía OMP |

### Data plane

| Comando | Qué valida |
|---------|-----------|
| `show bfd sessions` | Túneles BFD entre WAN Edges |
| `show tunnel statistics` | Estado y estadísticas de los túneles IPsec |

### Conectividad end-to-end

| Comando | Qué valida |
|---------|-----------|
| `ping vrf 1 192.168.20.1 source 192.168.10.1` | Conectividad entre sites a través del overlay |

### Outputs

<!-- TODO: Pegar outputs reales de cada comando durante la ejecución del lab -->

---

## 8. Resultados obtenidos

<!-- TODO: Documentar qué ocurrió realmente durante el lab -->

---

## 9. Problemas encontrados

<!-- TODO: Para cada problema documentar:
- Descripción del problema
- Síntoma observado
- Causa raíz
- Solución aplicada
- Comando o output que confirmó la resolución
-->

---

## 10. Lecciones aprendidas

<!-- TODO: Documentar lecciones específicas derivadas de ESTE lab, no conceptos genéricos -->

---

## 11. Mejores prácticas

- Validar conectividad underlay (VPN 0) en cada componente antes de iniciar el onboarding
- Mantener consistencia estricta en `organization-name` entre todos los componentes — una discrepancia impide la formación de control connections
- Verificar reachability entre control components antes de incorporar WAN Edges
- Seguir siempre la secuencia de despliegue: Manager → Validator → Controller → Edges

---

## 12. Próximos experimentos

| Lab | Tema |
|-----|------|
| 02 | Templates y automatización de configuración en Cisco Catalyst SD-WAN |
| 03 | Políticas de control (route policy, TLOC) |
