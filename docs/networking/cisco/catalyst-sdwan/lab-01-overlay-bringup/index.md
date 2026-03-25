# Lab 01 – Cisco Catalyst SD-WAN Overlay Bring-up

## Información general

| Campo | Valor |
|-------|-------|
| Lab | 01 |
| Tecnología | Cisco Catalyst SD-WAN |
| Versión | 20.18.1 |
| Nivel | Intermedio |
| Entorno | EVE-NG |
| Fecha inicio | 2026-03-14 |
| Estado | En progreso — Control plane operativo, WAN Edge onboarding bloqueado (ver sección 6.8) |

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

> **Nomenclatura:** Cisco migró oficialmente a los nombres "SD-WAN Manager", "SD-WAN Validator" y "SD-WAN Controller" a partir de la versión 20.x. Los nombres legacy (vManage, vBond, vSmart) siguen siendo de uso común en la industria y aparecen en la CLI. Este documento usa la nomenclatura oficial y referencia los nombres legacy cuando es necesario para claridad.

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

### Recursos asignados en EVE-NG

| Nodo | vCPU | RAM | Disco |
|------|------|-----|-------|
| SD-WAN Manager | 2 | 32 GB | 100 GB |
| SD-WAN Validator | 2 | 4 GB | 10 GB |
| SD-WAN Controller | 2 | 4 GB | 10 GB |
| Edge-1 | 2 | 4 GB | 8 GB |
| Edge-2 | 2 | 4 GB | 8 GB |

Host EVE-NG: 8 vCPUs, 62 GB RAM, 490 GB disco.

### Segmentación VPN

| VPN | Uso | Descripción |
|-----|-----|-------------|
| VPN 512 | Management plane | Red de gestión de los control components |
| VPN 0 | Transport underlay | Red de transporte para formar el overlay |
| VPN 1 | Service VPN | Tráfico de usuarios (LAN de cada site) |

> **Sobre VPN 1:** Ambos sites utilizan el mismo VPN ID (VPN 1) para el service VPN, pero con subnets diferentes (192.168.10.0/24 y 192.168.20.0/24). Esto es el comportamiento estándar de segmentación SD-WAN: el VPN ID identifica el segmento lógico, no la subnet específica.

### Mapeo de interfaces EVE-NG

Este mapeo fue verificado con `brctl show` en el host EVE-NG y es fundamental para evitar errores de conectividad.

| Bridge EVE-NG | Red | Nodo | Interfaz QEMU (índice) | Interfaz SD-WAN | VPN |
|---------------|-----|------|----------------------|-----------------|-----|
| vnet1_2 | Transport | vManage | vunl1_1_0 | eth0 | 0 |
| vnet1_2 | Transport | vBond | vunl1_2_0 | eth0 | 0 |
| vnet1_2 | Transport | vSmart | vunl1_3_0 | eth0 | 0 |
| pnet0 | Management | vManage | vunl1_1_1 | eth1 | 512 |
| pnet0 | Management | vBond | vunl1_2_1 | ge0/0 | 512 |
| pnet0 | Management | vSmart | vunl1_3_1 | eth1 | 512 |

> **Nota crítica:** Los nombres de interfaz dentro del nodo SD-WAN no siempre coinciden con el orden de cableado en EVE-NG. La única forma confiable de verificar el mapeo es con `brctl show` en el host. El índice `_0` corresponde al primer cable conectado y `_1` al segundo.

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

| Dispositivo | Interfaz | Dirección IP |
|-------------|----------|--------------|
| SD-WAN Manager | eth1 | 10.30.30.45/24 |
| SD-WAN Validator | ge0/0 | 10.30.30.46/24 |
| SD-WAN Controller | eth1 | 10.30.30.47/24 |

### 4.3 Transport Network (VPN 0)

| Dispositivo | Interfaz | Dirección IP |
|-------------|----------|--------------|
| SD-WAN Manager | eth0 | 10.10.10.1/24 |
| SD-WAN Validator | eth0 | 10.10.10.2/24 |
| SD-WAN Controller | eth0 | 10.10.10.3/24 |
| Edge-1 | Gi1 | 10.10.10.11/24 |
| Edge-2 | Gi1 | 10.10.10.12/24 |

> **Sobre el gateway 10.10.10.254:** En este lab no existe un gateway real en la red de transporte. El transport cloud de EVE-NG es un bridge L2 que conecta todos los nodos directamente en la misma subnet. No se configuró default route en VPN 0 para los control components.

### 4.4 Service VPN (VPN 1)

| Site | Dispositivo | Interfaz | Red |
|------|-------------|----------|-----|
| Site 1 | Edge-1 | Gi2 | 192.168.10.0/24 |
| Site 2 | Edge-2 | Gi2 | 192.168.20.0/24 |

---

## 5. Hipótesis

Si todos los control components (SD-WAN Manager, Validator y Controller) comparten el mismo `organization-name` y tienen reachability entre sí en VPN 0, entonces:

1. Los control connections DTLS/TLS se establecerán entre todos los componentes de forma automática
2. Los WAN Edge routers completarán el proceso de autenticación vía el Validator
3. Los WAN Edge establecerán sesiones OMP con el Controller
4. Se formarán túneles IPsec BFD entre los WAN Edge sin necesidad de políticas manuales
5. Las rutas del service VPN (VPN 1) se propagarán vía OMP, permitiendo conectividad entre 192.168.10.0/24 y 192.168.20.0/24

---

## 6. Procedimiento

### 6.1 Secuencia de despliegue

De acuerdo con la documentación oficial de Cisco, el overlay bringup sigue esta secuencia:

1. Iniciar SD-WAN Manager
2. Iniciar SD-WAN Validator
3. Iniciar SD-WAN Controller
4. Onboard WAN Edge routers

Este orden permite que los control components se autentiquen entre sí antes de que los WAN Edge intenten unirse al fabric.

### 6.2 Verificación de recursos del host

Antes de encender cualquier nodo, se verificaron los recursos del host EVE-NG:

```
root@eve-ng:~# free -h
               total        used        free      shared  buff/cache   available
Mem:            62Gi       1.4Gi        58Gi       5.0Mi       2.5Gi        60Gi
Swap:          8.0Gi          0B       8.0Gi

root@eve-ng:~# nproc
8

root@eve-ng:~# df -h /
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  490G   35G  436G   8% /
```

Recursos suficientes para los 5 nodos (48 GB RAM estimados + overhead de EVE-NG).

### 6.3 SD-WAN Manager bootstrap

El primer componente encendido fue el SD-WAN Manager. Durante el primer boot, seleccionó el storage device `vdb` (500 GB) y formateó el filesystem.

#### Configuración aplicada

```
config
system
 host-name             vmanage
 system-ip             1.1.1.1
 site-id               1
 organization-name     Lab_Stiven_SDWAN
 vbond 10.10.10.2
!
vpn 0
 interface eth0
  ip address 10.10.10.1/24
  tunnel-interface
  no shutdown
 !
!
vpn 512
 interface eth1
  ip address 10.30.30.45/24
  no shutdown
 !
!
commit
```

#### Validación

```
vmanage# show system status
Controller Compatibility:
Version: 20.18.1
System state:            GREEN. All daemons up
System FIPS state:       Enabled
Personality:             vmanage
CPU allocation:          4 total
Memory usage:            32105540K total
```

```
vmanage# ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.181 ms
--- 4 packets transmitted, 4 received, 0% packet loss
```

La interfaz eth0 estaba operativa (ping a sí mismo OK). El ping al gateway 10.10.10.254 falló porque no existe un gateway real en el lab (ver sección 4.3).

### 6.4 SD-WAN Validator bootstrap

#### Identificación del mapeo de interfaces

La imagen de vBond tiene los nombres de interfaz `eth0` y `ge0/0`. El mapeo contra EVE-NG fue verificado con `brctl show`:

- `eth0` (índice _0) = bridge vnet1_2 = **Transport (VPN 0)**
- `ge0/0` (índice _1) = bridge pnet0 = **Management (VPN 512)**

> **Problema encontrado:** La imagen de vBond trae `eth0` asignada a VPN 0 por defecto. Al intentar configurar `eth0` en VPN 512 y `ge0/0` en VPN 0, el commit fallaba con el error `values are not unique: eth0`. Fue necesario primero remover `eth0` de VPN 0 antes de reasignarla. Ver sección 9 para más detalle.

#### Configuración final aplicada

```
config
system
 host-name             vbond
 system-ip             1.1.1.2
 site-id               1
 organization-name     Lab_Stiven_SDWAN
 vbond 10.10.10.2 local
!
vpn 0
 interface eth0
  ip address 10.10.10.2/24
  tunnel-interface
   encapsulation ipsec
   allow-service sshd
   allow-service netconf
  !
  no shutdown
 !
!
vpn 512
 interface ge0/0
  ip address 10.30.30.46/24
  no shutdown
 !
!
commit
```

> **Nota:** El keyword `local` en `vbond 10.10.10.2 local` es obligatorio en el Validator — le indica al dispositivo que él mismo es el vBond. Los servicios `sshd` y `netconf` en el tunnel-interface son necesarios para que vManage pueda gestionar el dispositivo vía NETCONF (puerto 830).

#### Validación

```
vbond# ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.793 ms
--- 3 packets transmitted, 3 received, 0% packet loss
```

### 6.5 SD-WAN Controller bootstrap

#### Mapeo de interfaces

- `eth0` (índice _0) = bridge vnet1_2 = **Transport (VPN 0)**
- `eth1` (índice _1) = bridge pnet0 = **Management (VPN 512)**

#### Configuración aplicada

```
config
system
 host-name             vsmart
 system-ip             1.1.1.3
 site-id               1
 organization-name     Lab_Stiven_SDWAN
 vbond 10.10.10.2
!
vpn 0
 interface eth0
  ip address 10.10.10.3/24
  tunnel-interface
   allow-service sshd
   allow-service netconf
  !
  no shutdown
 !
!
vpn 512
 interface eth1
  ip address 10.30.30.47/24
  no shutdown
 !
!
commit
```

#### Validación

```
vsmart# ping 10.10.10.1
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=3.11 ms
--- 3 packets transmitted, 3 received, 0% packet loss

vsmart# ping 10.10.10.2
64 bytes from 10.10.10.2: icmp_seq=1 ttl=64 time=1.03 ms
--- 2 packets transmitted, 2 received, 0% packet loss
```

Conectividad full mesh entre los tres control components confirmada.

### 6.6 Instalación de certificados

Los control connections DTLS/TLS no se forman sin certificados. El output de `show control local-properties` en vManage confirmó `certificate-status: Not-Installed`.

#### 6.6.1 Generación del Root CA

Se generó un Root CA enterprise auto-firmado desde una VM Linux:

```bash
openssl genrsa -out ROOTCA.key 4096

openssl req -x509 -new -nodes -key ROOTCA.key -sha256 -days 3650 \
  -subj "/C=BO/ST=SantaCruz/L=SantaCruz/O=Lab_Stiven_SDWAN/CN=SD-WAN-Lab-RootCA" \
  -out ROOTCA.pem
```

#### 6.6.2 Configuración en vManage GUI

Desde la GUI de vManage (https://10.30.30.45):

1. **Administration > Settings > Organization Name** — verificar `Lab_Stiven_SDWAN`
2. **Administration > Settings > Validator** — configurar `10.10.10.2` port `12346`
3. **Administration > Settings > Trust and Privacy > Certificate Settings** — seleccionar Enterprise, importar el Root CA (ROOTCA.pem)

#### 6.6.3 Instalación del Root CA en vBond y vSmart

Antes de instalar certificados firmados, cada dispositivo necesita el Root CA para poder validar la cadena de certificados:

```
vshell
echo '[contenido del ROOTCA.pem]' > /home/admin/root-ca.pem
exit

request root-cert-chain install /home/admin/root-ca.pem
```

Este paso se ejecutó en vBond y vSmart. Sin el Root CA instalado, la instalación de certificados falla con el error: `root-ca-chain unable to validate the certificate`.

#### 6.6.4 Flujo de certificados por dispositivo

Para cada control component (vManage, vBond, vSmart):

1. Generar CSR desde vManage GUI: **Configuration > Certificates > Control Components > ... > Generate CSR**
2. Firmar el CSR con el Root CA:
   ```bash
   openssl x509 -req -in [device].csr -CA ROOTCA.pem -CAkey ROOTCA.key \
     -CAcreateserial -out [device].pem -days 3650 -sha256
   ```
3. Subir el certificado firmado: **Install Certificate** (link azul en la parte superior de la tabla)
4. Sincronizar con vBond: **Send to Validator**

#### 6.6.5 Registro de vBond y vSmart en vManage

Los dispositivos vBond y vSmart se registraron en vManage usando el workflow:

**Configuration > Devices > Control Components > + Add control component**

- Para cada dispositivo se usó la IP de transporte (VPN 0) como management IP
- vManage se conecta por NETCONF (puerto 830) para registrar el dispositivo
- El wizard permite configurar System, VPN 0 y VPN 512 del dispositivo remoto

> **Prerequisito:** Antes de agregar dispositivos, se deben completar los "Common control component settings" (NTP, AAA, Security, Controller).

### 6.7 Validación del control plane

Después de instalar certificados y enviar al Validator, los control connections se formaron:

```
vmanage# show control connections
PEER    PEER PEER            SITE       DOMAIN PEER
TYPE    PROT SYSTEM IP       ID         ID     PRIVATE IP          PORT  STATE UPTIME
----------------------------------------------------------------------------------------------
vsmart  dtls 1.1.1.3         1          1      10.10.10.3          12346 up    0:00:01:45
vbond   dtls 1.1.1.2         0          0      10.10.10.2          12346 up    0:00:02:59
```

**Hipótesis #1 validada:** Los control connections DTLS se establecieron automáticamente entre los control components.

### 6.8 WAN Edge onboarding (bloqueado — múltiples problemas con imágenes C8000v)

#### Imagen 1: catc8k-17.18.01a — No bootea

La primera imagen desplegada para los WAN Edges fue `catc8k-17.18.01a`. Los nodos no booteaban — la consola se quedaba en blanco y el proceso QEMU consumía 99% CPU con solo 65 MB de RAM (indicando que la VM nunca arrancaba el OS). El problema era doble:

1. El template de EVE-NG asignaba **qemu-4.1.0** a los nodos catc8k
2. La imagen 17.18 no era compatible con QEMU — se quedaba en `Booting from Hard Disk...` incluso con qemu-8.2.1

#### Imagen 2: catc8k-17.15.04c — Bootea pero config no persiste

Se descargó `c8000v-universalk9_16G_serial.17.15.04c.qcow2` (variante Serial, 16G, no-EFI, QCOW2 nativo para KVM). Se actualizaron los templates de QEMU en EVE-NG a versión 8.2.1 y se recrearon los nodos.

La imagen booteó correctamente en modo autónomo. Se activó controller mode con `controller-mode enable`. Verificación exitosa:

```
Router operating mode: Controller-Managed
Personality:             vEdge
Device role:             cEdge-SDWAN
Controller Compatibility: 20.15
```

Sin embargo, la configuración SD-WAN aplicada vía `config-transaction` no persistía — `show sdwan running-config system` solo mostraba `admin-tech-on-failure` después del commit.

#### Imagen 3: catc8k-17.12.04a — Mismo problema de config

Se descargó `c8000v-universalk9_16G_serial.17.12.04a.qcow2` (Dublin, versión MD estable). Controller Compatibility: 20.12.

El mismo problema: la configuración SD-WAN no persiste después del commit. Se intentaron múltiples sintaxis de CLI (`sdwan system`, `system` directo, `vpn 0`) sin éxito.

#### Intento de bootstrap file

Se detectó que el Edge reportaba `Bootstrap file doesn't exist` al ejecutar `request platform software sdwan config reset`. Se intentó crear el bootstrap file (`ciscosdwan_cloud_init.cfg`) en bootflash por múltiples vías:

1. **Desde CLI**: `tclsh` bloqueado en controller mode, `vshell` no disponible, `request platform software system shell` requiere consent-token
2. **Montando el disco QCOW2 desde EVE-NG**: Se usó `qemu-nbd` para montar la partición bootflash y escribir el archivo directamente. El archivo se creó correctamente pero el Edge no lo leyó al bootear.

#### Intento de registro en vManage via WAN Edge List

Se intentó registrar los Edges en vManage subiendo un CSV con los chassis-numbers:

| Edge | Chassis Number |
|------|---------------|
| Edge-1 | C8K-dcc98548-d577-453e-9095-c0f908c417cc |
| Edge-2 | C8K-356a0162-aa0d-4e5b-9945-0f36ad04e002 |

El upload falló con el error: **"Virtual Platform is not allowed in CSV Upload"**. vManage 20.18.1 no permite registrar plataformas virtuales (C8000V) vía CSV — requiere Smart Account o PnP Connect, que no están disponibles en un lab offline.

El workflow "Quick Connect" con la opción "Skip for now" tampoco descubrió los Edges porque no están en el inventario de vManage.

#### Análisis del bloqueo

La causa raíz del problema de onboarding de WAN Edges C8000v en un lab offline es la combinación de:

1. **Las imágenes C8000v en controller mode no aceptan configuración SD-WAN por CLI interactiva** — están diseñadas para recibir configuración desde vManage vía templates o desde un bootstrap file generado por PnP
2. **vManage 20.18.1 no permite registrar C8000V virtuales via CSV** — requiere Smart Account o PnP Connect
3. **Sin registro en vManage, no se pueden generar certificados ni empujar configuración a los Edges**

> **Estado actual:** Onboarding de WAN Edges bloqueado. El control plane está 100% operativo. Para desbloquear se necesita: (a) imágenes vEdge cloud (Viptela nativas) que aceptan configuración por CLI directa, (b) acceso a Smart Account para registrar los C8000v, o (c) uso de la API REST de vManage para agregar los dispositivos al inventario programáticamente.

### 6.9 Alternativas identificadas para desbloquear

| Alternativa | Descripción | Viabilidad |
|-------------|-------------|------------|
| Imágenes vEdge cloud | Dispositivos Viptela nativos, CLI directa como vBond/vSmart | Requiere obtener imagen `viptela-edge-*.qcow2` |
| Smart Account | Registrar C8000v via Cisco Smart Account + PnP | Requiere cuenta Cisco con Smart Licensing activo |
| API REST de vManage | Agregar dispositivos al inventario via `POST /dataservice/system/device` | Viable si la API lo permite sin Smart Account |
| Versiones anteriores de vManage | Usar vManage 20.9.x o anterior que puede permitir CSV upload de plataformas virtuales | Requiere redesplegar los control components |

---

## 7. Validaciones técnicas

### Control plane (ejecutados)

| Comando | Qué valida | Resultado |
|---------|-----------|-----------|
| `show control connections` | Túneles DTLS/TLS entre componentes | UP — vBond y vSmart conectados |
| `show control local-properties` | Certificados y organization-name | Certificado instalado, org-name correcto |

### Pendientes (requieren WAN Edges)

| Comando | Qué valida |
|---------|-----------|
| `show omp peers` | Sesiones OMP entre WAN Edges y Controller |
| `show omp routes` | Rutas del service VPN aprendidas vía OMP |
| `show bfd sessions` | Túneles BFD entre WAN Edges |
| `show tunnel statistics` | Estado y estadísticas de los túneles IPsec |
| `ping vrf 1 192.168.20.1 source 192.168.10.1` | Conectividad entre sites a través del overlay |

---

## 8. Resultados obtenidos (parciales)

El control plane del fabric SD-WAN se levantó correctamente:

- Los tres control components (Manager, Validator, Controller) establecieron conexiones DTLS
- Los certificados enterprise se generaron, firmaron e instalaron exitosamente
- La organización `Lab_Stiven_SDWAN` es consistente en todos los dispositivos
- El Validator (vBond) tiene la lista de certificados sincronizada

Pendiente: onboarding de WAN Edges y validación del data plane.

---

## 9. Problemas encontrados

### Problema 1: Interfaces de vBond invertidas respecto al cableado EVE-NG

**Síntoma:** Después de configurar `ge0/0` como transport (VPN 0) y `eth0` como management (VPN 512), el ping entre vBond y vManage fallaba con `Destination Host Unreachable`.

**Diagnóstico:** Se ejecutó `brctl show` en el host EVE-NG y se descubrió que:
- `eth0` (índice `_0`) estaba en el bridge `vnet1_2` (Transport)
- `ge0/0` (índice `_1`) estaba en el bridge `pnet0` (Management)

La configuración asignaba las interfaces al VPN opuesto al cableado real.

**Causa raíz:** En EVE-NG, el índice de interfaz (`_0`, `_1`) depende del orden de conexión de los cables, no del nombre de la interfaz dentro del nodo. El nombre que el sistema operativo del nodo asigna a cada interfaz puede no coincidir con el orden esperado.

**Solución:** Se invirtió la configuración para que coincidiera con el cableado real: `eth0` en VPN 0 (transport) y `ge0/0` en VPN 512 (management).

**Validación:** `ping 10.10.10.1` desde vBond respondió exitosamente.

### Problema 2: Commit fallido por interfaz duplicada en vBond

**Síntoma:** Al intentar configurar `eth0` en VPN 512, el commit falló con: `values are not unique: eth0 — 'vpn 0 interface eth0 if-name' / 'vpn 512 interface eth0 if-name'`.

**Causa raíz:** La imagen de vBond traía `eth0` pre-asignada a VPN 0 por defecto. SD-WAN no permite asignar la misma interfaz a dos VPNs simultáneamente.

**Solución:** Primero se removió `eth0` de VPN 0 (`no interface eth0`), y luego se configuraron ambas interfaces en sus VPNs correctos.

### Problema 3: vManage no podía agregar vBond/vSmart via GUI (puerto 830)

**Síntoma:** Al intentar agregar vBond como control component, vManage mostraba: `Failed to add device — Unable to connect to admin@10.10.10.2:830`.

**Causa raíz:** Los servicios SSH y NETCONF estaban deshabilitados en el tunnel-interface de vBond y vSmart (`no allow-service sshd`, `no allow-service netconf`). NETCONF corre sobre SSH en el puerto 830, y ambos servicios deben estar permitidos en el tunnel-interface para que vManage pueda gestionar los dispositivos remotamente.

**Solución:** Se habilitaron ambos servicios en el tunnel-interface de VPN 0 en vBond y vSmart:
```
tunnel-interface
 allow-service sshd
 allow-service netconf
```

### Problema 4: Certificado no se instalaba — Root CA no presente en vBond/vSmart

**Síntoma:** Al intentar instalar el certificado firmado en vBond, el error fue: `root-ca-chain unable to validate the certificate... Aborting!`

**Causa raíz:** vBond y vSmart no tenían el Root CA enterprise instalado. Sin la cadena de confianza, no podían validar los certificados firmados por ese CA.

**Solución:** Se instaló el Root CA manualmente en cada dispositivo:
```
vshell
echo '[contenido ROOTCA.pem]' > /home/admin/root-ca.pem
exit
request root-cert-chain install /home/admin/root-ca.pem
```

### Problema 5: Configuración de vBond se perdió después de reinicio

**Síntoma:** Después de varios días sin usar el lab, el `show running-config system` de vBond no mostraba `system-ip`, `site-id`, ni `organization-name`.

**Causa raíz:** La configuración bootstrap no se persistió correctamente, posiblemente porque el nodo se reinició antes de que la configuración se guardara en NVRAM.

**Solución:** Se reconfiguró el bootstrap completo de vBond y se verificó el commit.

### Problema 6: Imagen C8000v 17.18.01a no booteaba en EVE-NG

**Síntoma:** Los nodos Edge se encendían pero la consola quedaba vacía. El proceso QEMU consumía 99% CPU con solo 65 MB de RAM. Al testear manualmente, la consola mostraba `Booting from Hard Disk...` y no avanzaba.

**Diagnóstico:** Se probó con múltiples versiones de QEMU (4.1.0, 8.2.1) y con diferentes tipos de disco (virtio, ide). La imagen tenía particiones válidas (`qemu-img check` limpio, `virt-filesystems` mostraba particiones IOS-XE correctas) pero no booteaba.

**Causa raíz:** La imagen `catc8k-17.18.01a` no era compatible con el entorno QEMU/KVM de EVE-NG. Posiblemente era una imagen para una plataforma de virtualización diferente (ESXi) o tenía un bootloader incompatible.

**Solución:** Se descargó una imagen diferente desde el portal de Cisco: `c8000v-universalk9_16G_serial.17.15.04c.qcow2` (variante Serial, 16G, no-EFI, QCOW2 nativo para KVM). Esta imagen booteó correctamente.

### Problema 7: EVE-NG asignaba versión incorrecta de QEMU a los Edges

**Síntoma:** Incluso con la nueva imagen, los Edges no booteaban en EVE-NG. Manualmente con qemu-8.2.1 sí funcionaban.

**Diagnóstico:** `ps aux | grep "D 4" | grep -o "qemu-[0-9.]*"` mostraba que EVE-NG usaba `qemu-4.1.0` para los Edges.

**Causa raíz:** El template `catc8k.yml` de EVE-NG tenía `qemu_version: 4.1.0` hardcodeado. Cambiar el template no era suficiente para los nodos ya existentes — los nodos guardaban la configuración de QEMU de cuando fueron creados.

**Solución:** Se actualizó el template y se **eliminaron y recrearon** los nodos Edge en la topología:
```bash
sed -i 's/qemu_version: 4.1.0/qemu_version: 8.2.1/' /opt/unetlab/html/templates/intel/catc8k.yml
```
Después se eliminaron los nodos Edge de la topología de EVE-NG y se recrearon para que tomaran el nuevo template.

### Problema 8: Configuración SD-WAN no persiste en WAN Edge C8000v (17.15 y 17.12)

**Síntoma:** Después de aplicar la configuración SD-WAN via `config-transaction` y hacer `commit`, el `show sdwan running-config` no mostraba system-ip, organization-name ni interfaces configuradas. El `show sdwan control local-properties` mostraba system-ip 0.0.0.0 y organization-name vacío. El commit no reportaba errores.

**Intentos realizados:**
- Sintaxis `sdwan system` dentro de `config-transaction`
- Sintaxis `system` directa (estilo Viptela) dentro de `config-transaction`
- Sintaxis mixta (IOS-XE para interfaces + sdwan para system)
- Tres imágenes diferentes: 17.18.01a, 17.15.04c, 17.12.04a

**Causa raíz:** Las imágenes C8000v en controller mode requieren un bootstrap file previo (`ciscosdwan_cloud_init.cfg`) o provisión desde vManage para inicializar el datastore de SD-WAN. Sin esta inicialización, el proceso confd/vdaemon no procesa la configuración aplicada por CLI.

**Estado:** Sin resolver. El problema es arquitectónico — no es de versión ni de sintaxis.

### Problema 9: Bootstrap file inyectado manualmente no es leído por el Edge

**Síntoma:** Se creó el archivo `ciscosdwan_cloud_init.cfg` directamente en la partición bootflash del disco QCOW2 usando `qemu-nbd` desde el host EVE-NG. Al bootear, el Edge no leyó el archivo y `show sdwan running-config system` seguía vacío.

**Diagnóstico:** El archivo existía en bootflash (confirmado montando el disco), pero el proceso de boot no lo procesaba. El comando `request platform software sdwan bootstrap-config save` generaba el archivo pero tampoco se aplicaba al reiniciar.

**Causa raíz (probable):** El bootstrap file debe tener un formato específico y/o debe ser generado por vManage o PnP Connect. Un archivo creado manualmente puede no tener las cabeceras o el formato que el proceso de inicialización de SD-WAN espera.

**Estado:** Sin resolver.

### Problema 10: vManage no permite registrar plataformas virtuales via CSV

**Síntoma:** Al subir el CSV con los chassis-numbers de los C8000V, vManage mostró: `Virtual Platform is not allowed in CSV Upload`.

**Causa raíz:** vManage 20.18.1 tiene una restricción que impide agregar plataformas virtuales (C8000V) al inventario via CSV upload. El registro de dispositivos virtuales requiere Smart Account + PnP Connect o sincronización con Cisco cloud.

**Impacto:** Sin registro en el inventario de vManage, no se pueden generar certificados para los Edges ni empujar configuración vía templates. Esto bloquea completamente el onboarding en un lab offline.

**Estado:** Sin resolver. Se requiere Smart Account, imágenes vEdge nativas, o uso de la API REST de vManage.

---

## 10. Lecciones aprendidas

1. **`brctl show` es el comando definitivo para mapear interfaces en EVE-NG.** No confiar en los nombres de interfaz dentro del nodo — el índice QEMU (`_0`, `_1`) determina a qué bridge está conectada cada interfaz, independientemente del nombre que el OS del nodo le asigne.

2. **Los certificados son un prerequisito duro para los control connections.** Sin certificados instalados y un Root CA distribuido, los túneles DTLS/TLS no se forman. No hay workaround.

3. **NETCONF y SSH deben estar habilitados en el tunnel-interface** para que vManage pueda gestionar remotamente a vBond y vSmart. Esto no es obvio en la configuración por defecto, donde ambos servicios vienen deshabilitados.

4. **El Root CA debe instalarse manualmente en vBond y vSmart antes de instalar certificados.** vManage instala el Root CA automáticamente en sí mismo, pero los demás dispositivos requieren instalación manual vía CLI (`request root-cert-chain install`).

5. **vManage consume la mayoría de los recursos del host.** Con 32 GB de RAM asignados, vManage usa ~31 GB y 4+ cores de CPU. El primer boot puede tardar 15+ minutos hasta que la GUI esté disponible. Los otros componentes (vBond, vSmart) son ligeros (~1 GB RAM cada uno).

6. **El keyword `local` en `vbond [ip] local` es obligatorio en el Validator.** Sin este keyword, el dispositivo no asume el rol de vBond correctamente.

7. **Los "Common control component settings" son un prerequisito en vManage 20.x.** Antes de poder agregar control components via GUI, estos settings deben estar configurados (aunque sea con valores por defecto).

8. **Las imágenes C8000v para EVE-NG deben ser variante Serial, QCOW2, no-EFI.** Al descargar desde el portal de Cisco, seleccionar `c8000v-universalk9_XG_serial.VERSION.qcow2`. Las variantes VGA, EFI, ISO y OVA no son compatibles con EVE-NG o requieren configuración adicional.

9. **Los templates de EVE-NG guardan la versión de QEMU.** Si se actualiza la versión de QEMU en el template YAML, los nodos existentes no se actualizan — deben eliminarse y recrearse para tomar la nueva configuración.

10. **La compatibilidad de versiones entre controladores y Edges es crítica.** No asumir que cualquier versión de IOS-XE funciona con cualquier versión de vManage. El campo `Controller Compatibility` en el `show sdwan system status` del Edge debe coincidir con la versión major del controlador. Verificar la matriz de compatibilidad de Cisco antes de desplegar.

11. **Los C8000v en controller mode no aceptan configuración SD-WAN por CLI interactiva de la misma forma que los dispositivos Viptela.** Los cEdges requieren un bootstrap file o provisión desde vManage. Esto es una diferencia fundamental con vBond/vSmart/vEdge donde la CLI funciona directamente.

12. **vManage 20.18.1 no permite registrar plataformas virtuales (C8000V) via CSV upload.** En labs offline sin Smart Account, esto bloquea el onboarding. Las alternativas son: imágenes vEdge cloud, API REST de vManage, o versiones anteriores de vManage que no tengan esta restricción.

13. **El GRUB de las imágenes C8000v ofrece dos opciones de boot:** `packages.conf` (boot modular, producción) y `GOLDEN IMAGE` (imagen monolítica de recovery). Siempre bootear con `packages.conf` excepto en escenarios de recovery.

14. **El comando `controller-mode enable` en C8000v borra NVRAM y toda la configuración.** Es un proceso destructivo que reinicia el router. Debe ejecutarse una sola vez y no se puede revertir sin reinstalar la imagen.

---

## 11. Mejores prácticas

- Siempre verificar el mapeo de interfaces con `brctl show` antes de configurar cualquier nodo en EVE-NG
- Habilitar `allow-service sshd` y `allow-service netconf` en el tunnel-interface de todos los control components desde el inicio
- Instalar el Root CA en todos los dispositivos antes de intentar instalar certificados firmados
- Validar conectividad underlay (VPN 0) entre cada par de control components antes de iniciar el proceso de certificados
- Mantener consistencia estricta en `organization-name` entre todos los componentes
- Seguir siempre la secuencia de despliegue: Manager > Validator > Controller > Edges
- Monitorear recursos del host EVE-NG durante el boot de vManage (`free -h`, `top`)
- Para imágenes C8000v en EVE-NG: usar variante Serial + QCOW2 + no-EFI, y verificar que el template YAML apunte a una versión de QEMU 8.x o superior
- Verificar la matriz de compatibilidad de versiones de Cisco antes de combinar diferentes versiones de controladores y Edges
- Los WAN Edges C8000v requieren mínimo 8 GB de RAM para bootear correctamente en controller mode

---

## 12. Próximos pasos

| Prioridad | Paso | Descripción |
|-----------|------|-------------|
| 1 | Resolver onboarding de Edges | Obtener imágenes vEdge cloud, usar API REST de vManage, o configurar Smart Account |
| 2 | Onboarding Edge-1 y Edge-2 | Bootstrap, certificados y registro en el fabric |
| 3 | Validación OMP | Verificar sesiones OMP y propagación de rutas |
| 4 | Validación data plane | Verificar túneles IPsec/BFD entre Edges |
| 5 | Ping end-to-end | Conectividad 192.168.10.0/24 a 192.168.20.0/24 |
| 6 | Documentación final | Completar resultados, actualizar hipótesis validadas |

## 13. Próximos laboratorios

| Lab | Tema |
|-----|------|
| 02 | Templates y automatización de configuración en Cisco Catalyst SD-WAN |
| 03 | Políticas de control (route policy, TLOC) |
| 04 | App-Aware Routing |
