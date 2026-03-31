# Lab 01 – Cisco Catalyst SD-WAN Overlay Bring-up

## Addendum: Validación del Overlay en DevNet Sandbox SD-WAN 20.12

> **Contexto:** Este addendum documenta la fase de validación del Lab 01 ejecutada en el sandbox DevNet SD-WAN 20.12, complementando el trabajo de control plane realizado en EVE-NG. El control plane (vManage, vBond, vSmart) fue levantado y validado exitosamente en EVE-NG, pero el onboarding de WAN Edges C8000v quedó bloqueado por limitaciones de la plataforma (ver Lab 01 principal, sección 6.8). El sandbox DevNet proporciona un fabric completo pre-configurado que permite completar las validaciones pendientes del data plane.

---

## Información de la sesión

| Campo | Valor |
|-------|-------|
| Entorno | DevNet Sandbox — Cisco SD-WAN 20.12 Reservable |
| Instancia | Cisco SD-WAN 20.12-20260329T06354711 |
| Fecha | 2026-03-29 al 2026-04-03 |
| Duración | ~5 días |
| Acceso | VPN AnyConnect → red privada del sandbox |
| Versión SD-WAN | 20.12 (Controller Compatibility) / IOS-XE 17.12.01a |
| Organization | DevNet_SD-WAN_Sandbox |

---

## 1. Descubrimiento de la topología

### 1.1 Inventario de dispositivos

El fabric del sandbox contiene 9 dispositivos: 3 controllers y 6 WAN Edges (5 SD-WAN + 1 SD-Routing).

| System-IP | Site-ID | Hostname | Rol | Modelo | Transportes |
|-----------|---------|----------|-----|--------|-------------|
| 10.10.1.1 | 101 | Manager | SD-WAN Manager (vManage) | vManage | Management |
| 10.10.1.3 | 0 | — | SD-WAN Validator (vBond) | vBond | Management |
| 10.10.1.5 | 101 | — | SD-WAN Controller (vSmart) | vSmart | Management |
| 10.10.1.11 | 100 | DC-cEdge01 | DC Hub | C8000V | MPLS + Internet |
| 10.10.1.13 | 1001 | Site1-cEdge01 | Branch Spoke | C8000V | MPLS + Internet |
| 10.10.1.15 | 1002 | Site2-cEdge01 | Branch Spoke | C8000V | MPLS + Internet |
| 10.10.1.17 | 1003 | Site3-cEdge01 | Branch Spoke | C8000V / vEdge | MPLS only |
| 10.10.1.18 | 1003 | Site3-cEdge02 | Branch Spoke | C8000V | Internet only |
| 10.10.1.22 | 1012 | Site11-Edge01 | SD-Routing | C8000V | Default |

> **Nota sobre vBond site-id 0:** El Validator siempre reporta site-id 0 en control connections porque actúa como orquestador entre sites, no pertenece a un site específico. Esto es comportamiento estándar documentado por Cisco.

> **Nota sobre Site3 (site-id 1003):** Este site tiene dos dispositivos separados, cada uno con un solo transporte. 10.10.1.17 solo tiene TLOC MPLS y 10.10.1.18 solo tiene TLOC Internet. Esto simula un escenario donde un site tiene dos Edges con transportes WAN diferentes — análogo a un diseño con dos ISPs donde cada Edge se conecta a un proveedor distinto.

### 1.2 Plan de direccionamiento

#### Controllers (Management network 10.10.20.0/24)

| Dispositivo | IP Management |
|-------------|---------------|
| vManage | 10.10.20.90 |
| vSmart | 10.10.20.70 |
| vBond | 10.10.20.80 |

#### WAN Edges — Transport (VPN 0)

| Dispositivo | MPLS (Gi2) | Internet (Gi4) |
|-------------|------------|----------------|
| DC-cEdge01 | 10.10.23.6/30 | 10.10.23.38/30 |
| Site1-cEdge01 | 10.10.23.10/30 | 10.10.23.42/30 |
| Site2-cEdge01 | 10.10.23.14/30 | 10.10.23.46/30 |
| Site3-cEdge01 | 10.10.23.18/30 | — |
| Site3-cEdge02 | — | 10.10.23.50/30 |

> **Patrón de addressing underlay:** Cada link WAN usa una subnet /30 punto a punto entre el Edge y el router de transporte. MPLS usa el rango 10.10.23.4/30 incrementando de a 4. Internet usa el rango 10.10.23.36/30 con el mismo patrón.

#### WAN Edges — Management (VPN 512)

| Dispositivo | IP Management (Gi1) |
|-------------|---------------------|
| DC-cEdge01 | 10.10.20.172/24 |
| Site1-cEdge01 | (ver tab I/O del sandbox) |
| Site2-cEdge01 | 10.10.20.175/24 |

#### WAN Edges — Service VPN (VPN 1)

| Dispositivo | Service IP (Gi3) | LAN Subnet |
|-------------|-----------------|------------|
| DC-cEdge01 | 10.10.20.182/24 | 10.10.20.0/24 |
| Site1-cEdge01 | — | 10.10.21.0/24 |
| Site2-cEdge01 | 10.10.22.1/24 | 10.10.22.0/24 |
| Site3-cEdge01 | — | 10.10.24.0/24 |
| Site3-cEdge02 | — | 10.10.25.0/24 |

#### Mapeo de interfaces del DC-cEdge01

| Interfaz | IP | VPN | Función |
|----------|-----|-----|---------|
| GigabitEthernet1 | 10.10.20.172 | 512 | Management |
| GigabitEthernet2 | 10.10.23.6 | 0 | Transport MPLS |
| GigabitEthernet3 | 10.10.20.182 | 1 | Service VPN (LAN) |
| GigabitEthernet4 | 10.10.23.38 | 0 | Transport Internet |
| Loopback65528 | 192.168.1.1 | — | Loopback interno SD-WAN |
| Loopback65529 | 11.1.1.11 | — | Loopback adicional |
| Tunnel2 | 10.10.23.6 | — | Overlay IPsec sobre MPLS (auto-generado) |
| Tunnel4 | 10.10.23.38 | — | Overlay IPsec sobre Internet (auto-generado) |

> **Sobre las interfaces Tunnel:** IOS-XE crea automáticamente una interfaz Tunnel por cada TLOC cuando se habilita `tunnel-interface` en las interfaces de transport. Tunnel2 hereda la IP de GigabitEthernet2 (MPLS) y Tunnel4 la de GigabitEthernet4 (Internet). No se configuran manualmente.

---

## 2. Validación del control plane

### 2.1 Control connections desde vManage

```
Manager# show control connections
```

**Resultado:** Todas las control connections UP. vManage tiene conexiones DTLS activas hacia:
- 1 vSmart (10.10.1.5) — color default
- 1 vBond (10.10.1.3) — color default, más 3 instancias adicionales de vBond (0.0.0.0)
- 5 WAN Edges — cada uno por su color de transporte

**Uptime:** ~1 día 22 horas al momento de la validación, sin transitions.

### 2.2 Control connections desde Site1-cEdge01 (WAN Edge)

```
Site1-cEdge01# show sdwan control connections
```

| Peer | System-IP | Color local | Puerto | Estado |
|------|-----------|-------------|--------|--------|
| vSmart | 10.10.1.5 | mpls | 12446 | up |
| vSmart | 10.10.1.5 | public-internet | 12446 | up |
| vBond | 0.0.0.0 | mpls | 12346 | up |
| vBond | 0.0.0.0 | public-internet | 12346 | up |
| vManage | 10.10.1.1 | public-internet | 12446 | up |

**Análisis:** El Edge mantiene **dos conexiones al vSmart** (una por cada TLOC/transport). Esto es por diseño: si un transporte cae, el Edge mantiene control connectivity por el otro. El vManage solo necesita una conexión (para NETCONF/gestión). El vBond mantiene conexiones por ambos transportes para orchestration.

> **Diferencia clave con el Lab 01 en EVE-NG:** En EVE-NG solo logramos validar las control connections entre los controllers (Manager ↔ vSmart ↔ vBond). Aquí podemos ver la conexión completa incluyendo los WAN Edges, que es exactamente lo que quedó bloqueado. **Hipótesis #2 validada.**

### 2.3 Local properties del Edge

```
Site1-cEdge01# show sdwan control local-properties
```

| Propiedad | Valor | Significado |
|-----------|-------|-------------|
| personality | vedge | Rol de WAN Edge |
| organization-name | DevNet_SD-WAN_Sandbox | Consistente con todos los componentes |
| root-ca-chain-status | Installed | Root CA enterprise presente |
| certificate-status | Installed | Certificado de dispositivo firmado e instalado |
| certificate-validity | Valid (hasta 2034) | Certificado vigente |
| token | Invalid | Esperado — el token OTP se invalida después del primer uso |
| dns-name | 10.10.20.80 | IP del vBond (primer punto de contacto) |
| system-ip | 10.10.1.13 | Identity del Edge en el overlay |
| chassis-num | C8K-8E7F818D-3379-3DD6-3556-19C322680AA5 | UUID del C8000V |
| serial-num | 9AED55A6 | Serial generado por vManage |
| WAN interfaces | 2 (GigabitEthernet2 mpls, GigabitEthernet4 public-internet) | Dual transport |

> **Sobre el token Invalid:** Esto NO indica un error. El token es One-Time Password — se usa durante el onboarding inicial para autenticar el Edge con el vBond. Una vez usado, se marca como Invalid. En producción con PnP/Smart Account, este token se genera automáticamente.

> **Comparación con el bloqueo de EVE-NG:** En el Lab 01 de EVE-NG, el `show sdwan control local-properties` de los C8000v mostraba `system-ip: 0.0.0.0` y `organization-name: (vacío)` porque la configuración SD-WAN no persistía (Problema 8). Aquí el sandbox resolvió esto provisionando los Edges vía templates desde vManage, no por CLI interactiva.

### 2.4 Connection history — secuencia de bringup

Del `show control connections-history` del Manager se puede reconstruir la secuencia de arranque del sandbox:

1. **10:37:25 UTC** — Primeros intentos de conexión al vBond con `DCONFAIL` (DTLS Connection Failure). Normal durante el boot: los Edges intentan conectar antes de que los controllers estén listos.
2. **10:41:58 UTC** — Un peer `unknown` intenta `challenge_ack` pero falla con `VM_TMO` (vManage timeout). Es un Edge registrándose por primera vez.
3. **10:48:29 UTC** — Todas las conexiones previas se cierran con `DISTLOC` (TLOC Disabled). Los controllers se reiniciaron o reconfiguraron.
4. **10:48:31+ UTC** — Todos los Edges reconectan y las conexiones se estabilizan. A partir de aquí, uptime continuo sin interrupciones.

> **Lección:** Los errores `DCONFAIL` y `DISTLOC` durante el boot son normales y no indican problemas. Es útil saber leer el connection-history para distinguir errores transitorios de problemas reales.

---

## 3. Validación de OMP — Hipótesis #3

### 3.1 OMP routes — tabla completa desde Site1-cEdge01

```
Site1-cEdge01# show sdwan omp routes vpn 1
```

| Prefijo | FROM PEER | Status | TLOC (next-hop) | Color | Análisis |
|---------|-----------|--------|-----------------|-------|----------|
| 0.0.0.0/0 | vSmart (10.10.1.5) | C,I,R | 10.10.1.11 (DC) | mpls + internet | Default route originada por el DC |
| 10.10.20.0/24 | vSmart (10.10.1.5) | C,I,R | 10.10.1.11 (DC) | mpls + internet | Red de service/mgmt del DC |
| 10.10.21.0/24 | local (0.0.0.0) | C,Red,R | 10.10.1.13 (self) | mpls + internet | **LAN local** — redistributed a OMP |
| 10.10.22.0/24 | vSmart (10.10.1.5) | C,I,R | 10.10.1.15 (Site2) | mpls + internet | LAN de Site2 |
| 10.10.24.0/24 | vSmart (10.10.1.5) | C,I,R | 10.10.1.17 (Site3) | mpls only | LAN de Site3 (Edge MPLS) |
| 10.10.25.0/24 | vSmart (10.10.1.5) | C,I,R | 10.10.1.18 (Site3) | internet only | LAN de Site3 (Edge Internet) |

**Interpretación de los flags de status:**
- **C** (Chosen) — seleccionada como mejor ruta
- **I** (Installed) — instalada en la RIB/FIB
- **R** (Resolved) — TLOC resuelto, camino válido
- **Red** (Redistributed) — ruta local redistribuida a OMP (connected → OMP)

**Hipótesis #3 validada:** Las sesiones OMP están operativas y las rutas del service VPN se propagan correctamente entre todos los sites vía el vSmart (route reflector).

### 3.2 OMP routes desde DC-cEdge01 (Hub)

```
DC-cEdge01# show sdwan omp routes vpn 1
```

| Prefijo | FROM PEER | Status | TLOC | Análisis |
|---------|-----------|--------|------|----------|
| **0.0.0.0/0** | local (0.0.0.0) | **C,Red,R** | 10.10.1.11 (self) | **El DC origina la default** |
| 10.10.20.0/24 | local (0.0.0.0) | C,Red,R | 10.10.1.11 (self) | Connected del DC redistribuida |
| 10.10.21.0/24 | vSmart | C,I,R | 10.10.1.13 | LAN de Site1 |
| 10.10.22.0/24 | vSmart | C,I,R | 10.10.1.15 | LAN de Site2 |
| 10.10.24.0/24 | vSmart | C,I,R | 10.10.1.17 | LAN de Site3 MPLS |
| 10.10.25.0/24 | vSmart | C,I,R | 10.10.1.18 | LAN de Site3 Internet |

**Descubrimiento clave — origen de la default route:**

La ruta `0.0.0.0/0` en el DC tiene `FROM PEER: 0.0.0.0` y status `Red` (Redistributed). Esto confirma que es una ruta **local** del DC que se redistribuye a OMP. La routing table del VRF 1 del DC muestra:

```
S*    0.0.0.0/0 [1/0] via 10.10.20.254
```

La cadena completa es:
1. Ruta estática `0.0.0.0/0 → 10.10.20.254` configurada en el VRF 1 del DC (via template)
2. El template tiene `redistribute static` en OMP para VPN 1
3. OMP del DC anuncia la ruta al vSmart
4. vSmart la propaga a todos los spokes
5. Los spokes instalan `0.0.0.0/0` con next-hop TLOC del DC

> **Aplicación en producción:** En un diseño hub-and-spoke real, el Hub DC necesitaría una ruta default en el service VPN apuntando al firewall existente o al gateway corporativo. Esta ruta se redistribuye a OMP y todos los spokes la reciben automáticamente del vSmart, garantizando que todo el tráfico inter-site y hacia Internet pase por el DC donde están los servicios de seguridad centralizados.

### 3.3 Routing table del VRF 1 — DC-cEdge01

```
DC-cEdge01# show ip route vrf 1
```

```
S*    0.0.0.0/0 [1/0] via 10.10.20.254          ← ruta estática (configurada por template)
C     10.10.20.0/24 is directly connected, Gi3    ← LAN local del DC
m     10.10.21.0/24 [251/0] via 10.10.1.13        ← Site1 via OMP
m     10.10.22.0/24 [251/0] via 10.10.1.15        ← Site2 via OMP
m     10.10.24.0/24 [251/0] via 10.10.1.17        ← Site3-MPLS via OMP
m     10.10.25.0/24 [251/0] via 10.10.1.18        ← Site3-Internet via OMP
```

**Sobre las rutas OMP (código `m`):** La distancia administrativa de OMP es **251** — más alta que OSPF (110), EIGRP (90), BGP (20/200) o estática (1). Esto es intencional: si existe una ruta aprendida por un protocolo de routing local (underlay), esa tiene prioridad sobre OMP (overlay). Esto previene loops cuando un Edge tiene conectividad tanto en el underlay como en el overlay.

El next-hop de las rutas OMP es `Sdwan-system-intf` — una interfaz virtual de sistema. La resolución real del next-hop (a qué túnel IPsec enviar) ocurre en el data plane basándose en el TLOC seleccionado.

---

## 4. Validación del data plane (IPsec + BFD) — Hipótesis #4

### 4.1 BFD sessions desde Site1-cEdge01

```
Site1-cEdge01# show sdwan bfd sessions
```

Site1-cEdge01 tiene **12 sesiones BFD**, todas UP con 0 transitions (sin caídas desde el bringup):

| Destino | Site-ID | LOCAL→REMOTE | Source IP | Dest IP | Estado | Uptime |
|---------|---------|--------------|-----------|---------|--------|--------|
| DC | 100 | mpls→mpls | .23.10 | .23.6 | up | 1d22h |
| DC | 100 | mpls→internet | .23.10 | .23.38 | up | 1d22h |
| DC | 100 | internet→mpls | .23.42 | .23.6 | up | 1d22h |
| DC | 100 | internet→internet | .23.42 | .23.38 | up | 1d22h |
| Site2 | 1002 | mpls→mpls | .23.10 | .23.14 | up | 1d22h |
| Site2 | 1002 | mpls→internet | .23.10 | .23.46 | up | 1d22h |
| Site2 | 1002 | internet→mpls | .23.42 | .23.14 | up | 1d22h |
| Site2 | 1002 | internet→internet | .23.42 | .23.46 | up | 1d22h |
| Site3 | 1003 | mpls→mpls | .23.10 | .23.18 | up | 1d22h |
| Site3 | 1003 | mpls→internet | .23.10 | .23.50 | up | 1d22h |
| Site3 | 1003 | internet→mpls | .23.42 | .23.18 | up | 1d22h |
| Site3 | 1003 | internet→internet | .23.42 | .23.50 | up | 1d22h |

**Análisis:** La topología actual es **full-mesh**. Site1 tiene túneles IPsec directos hacia todos los demás Edges, no solo hacia el DC. Esto confirma que **no hay políticas centralizadas activas** — el estado default de SD-WAN es full-mesh.

Con dual transport (2 TLOCs) en Site1 y los peers:
- DC tiene 2 TLOCs → 2×2 = 4 sesiones
- Site2 tiene 2 TLOCs → 2×2 = 4 sesiones
- Site3 tiene 2 TLOCs (1 por cada Edge) → 2×2 = 4 sesiones
- **Total: 12 sesiones BFD** — exactamente lo que vemos

Todas las sesiones usan encapsulación IPsec con BFD multiplier 7 y TX interval 1000ms (1 segundo). Esto significa que un túnel se declara down después de 7 segundos sin respuesta BFD.

**Hipótesis #4 validada:** Los túneles IPsec con BFD se formaron automáticamente entre todos los WAN Edges sin necesidad de configuración manual de políticas.

---

## 5. Tabla completa de TLOCs

### 5.1 TLOCs del fabric

```
DC-cEdge01# show sdwan omp tlocs
```

| System-IP | Site-ID | Color | Public IP | Private IP | Status | Encap |
|-----------|---------|-------|-----------|------------|--------|-------|
| 10.10.1.11 | 100 | mpls | 10.10.23.6 | 10.10.23.6 | C,Red,R | aes256 |
| 10.10.1.11 | 100 | public-internet | 10.10.23.38 | 10.10.23.38 | C,Red,R | aes256 |
| 10.10.1.13 | 1001 | mpls | 10.10.23.10 | 10.10.23.10 | C,I,R | aes256 |
| 10.10.1.13 | 1001 | public-internet | 10.10.23.42 | 10.10.23.42 | C,I,R | aes256 |
| 10.10.1.15 | 1002 | mpls | 10.10.23.14 | 10.10.23.14 | C,I,R | aes256 |
| 10.10.1.15 | 1002 | public-internet | 10.10.23.46 | 10.10.23.46 | C,I,R | aes256 |
| 10.10.1.17 | 1003 | mpls | 10.10.23.18 | 10.10.23.18 | C,I,R | aes256 |
| 10.10.1.18 | 1003 | public-internet | 10.10.23.50 | 10.10.23.50 | C,I,R | aes256 |

**Observaciones:**
- **public-ip = private-ip en todos los TLOCs:** No hay NAT en ningún transporte del lab. En un deployment de producción con ISPs reales, los Edges estarán probablemente detrás de NAT y las IPs diferirán entre public y private.
- **TLOCs locales (DC) muestran `C,Red,R`** (Redistributed): el dispositivo genera sus propios TLOCs hacia OMP. Los TLOCs remotos muestran `C,I,R` (Installed): fueron recibidos del vSmart.
- **Todos usan `encap-encrypt: aes256` y `encap-auth: sha1-hmac`:** Perfil de seguridad IPsec del overlay.
- **`bfd-status: up`** en todos los TLOCs: cada transporte está operativo y monitoreado.

---

## 6. Estado del fabric — resumen

El sandbox está en un estado que corresponde a un **Day 1 completo** con el data plane totalmente operativo:

### Lo que está configurado

- **Control plane:** Enterprise CA certificates, tres controllers operativos, todas las control connections DTLS estables
- **Templates:** Device templates aplicados a todos los Edges (`vManaged: true`)
- **Dual transport:** MPLS + Internet en DC, Site1, Site2. Single transport por Edge en Site3
- **Service VPN 1:** Subnets 10.10.2x.0/24 por site, default route originada por el DC
- **Full-mesh IPsec:** Túneles entre todos los Edges con BFD monitoring
- **WAN emulator:** ~200ms latencia y ~0.05% loss en el path Internet del DC
- **SD-Routing device:** Site11-Edge01 en site 1012, onboarded como dispositivo autónomo

### Lo que NO está configurado (pendiente para labs siguientes)

- Políticas centralizadas (Hub-and-Spoke, Custom Control)
- Route leaking entre VPNs (Global ↔ Service, Inter-Service)
- Configuration Groups UX 2.0
- Tags y Global Site Topology
- RBAC con Resource Groups
- License management

---

## 7. Validación de hipótesis — resultado final

| # | Hipótesis | EVE-NG | Sandbox | Resultado |
|---|-----------|--------|---------|-----------|
| 1 | Control connections DTLS/TLS entre controllers | ✅ Validada | ✅ Re-confirmada (con Edges incluidos) | **Validada** |
| 2 | WAN Edges autenticados via Validator | ❌ Bloqueada | ✅ `show sdwan control local-properties` confirma cert Installed, token usado | **Validada** |
| 3 | Sesiones OMP entre Edges y Controller | ❌ Bloqueada | ✅ Rutas OMP propagadas correctamente entre todos los sites | **Validada** |
| 4 | Túneles IPsec/BFD entre Edges | ❌ Bloqueada | ✅ 12 sesiones BFD UP, full-mesh, 0 transitions | **Validada** |
| 5 | Conectividad E2E entre sites | ❌ Bloqueada | ✅ Tabla OMP completa, rutas instaladas en VRF 1 | **Validada** (pendiente ping explícito) |

---

## 8. Lecciones aprendidas

### De la exploración del sandbox

1. **SD-WAN forma full-mesh por defecto.** Sin políticas centralizadas, todos los Edges crean túneles IPsec directos hacia todos los demás. Las políticas Hub-and-Spoke no son el estado inicial — son una decisión de diseño que se implementa posteriormente.

2. **La default route del Hub es una ruta estática redistribuida a OMP.** El DC tiene una ruta estática `0.0.0.0/0` en el VRF del service VPN que se redistribuye a OMP vía el template. Los spokes la reciben automáticamente del vSmart. Este patrón es el estándar para topologías hub-and-spoke donde se requiere que todo el tráfico inter-site pase por un punto centralizado de seguridad.

3. **Cada TLOC genera una control connection independiente al vSmart.** Un Edge con dual transport (MPLS + Internet) mantiene dos control connections al vSmart — una por cada transporte. Si un transporte cae, el control plane sobrevive por el otro.

4. **Las interfaces Tunnel son auto-generadas por IOS-XE.** No se configuran manualmente. Se crean cuando se habilita `tunnel-interface` en una interfaz de transport, y heredan la IP de la interfaz física.

5. **Las rutas OMP tienen AD 251 en IOS-XE.** Más alta que cualquier protocolo de routing estándar. Esto previene loops en escenarios de dual-homing underlay/overlay y es una decisión de diseño deliberada.

6. **El token OTP se marca como Invalid después del primer uso.** Esto es comportamiento esperado, no un error. El token se usa una sola vez durante el onboarding y luego se invalida.

7. **Los errores DCONFAIL y DISTLOC en el connection-history durante el boot son normales.** Indican que los Edges intentaron conectar antes de que los controllers estuvieran listos, o que hubo un reinicio de controllers. No indican problemas persistentes.

8. **Un site puede tener múltiples Edges con transportes diferentes.** Site3 (site-id 1003) tiene dos C8000V separados, uno solo MPLS y otro solo Internet. Cada uno anuncia su propio TLOC y subnet de service VPN. El site-id los agrupa lógicamente.

### Comparación EVE-NG vs. Sandbox DevNet

| Aspecto | EVE-NG (Lab 01 original) | DevNet Sandbox |
|---------|--------------------------|----------------|
| Control plane bringup | Manual (bootstrap, certificados, troubleshooting) | Pre-configurado |
| WAN Edge onboarding | Bloqueado (C8000v + vManage 20.18.1) | Funcional (templates + provisioning automático) |
| Conocimiento adquirido | Profundo en certificados, interfaces, troubleshooting | Profundo en OMP, data plane, arquitectura del fabric |
| Valor de ingeniería | Alto — transferible a producción | Alto — entender el estado operativo del fabric |

> **Conclusión:** Los dos entornos son complementarios, no redundantes. EVE-NG enseñó el "cómo se construye" (Day 0). El sandbox enseña el "cómo funciona una vez construido" (Day 1). En producción se necesitan ambos conocimientos.

---

## 9. Próximos pasos en el sandbox

| Sesión | Tema | Módulos de la guía | Relevancia |
|--------|------|---------------------|------------|
| 2 | Templates + Config Groups UX 2.0 | Módulo 6 | Modelo de configuración escalable para múltiples device types |
| 3 | Políticas Hub-and-Spoke + Route Leaking | Módulos 1, 2, 4, 5 | Segmentación por VPN, topología controlada, inter-VPN routing |
| 4 | Licensing + RBAC + Escenarios avanzados | Módulos 7, 8 | Gestión operativa, modelo de licenciamiento |
