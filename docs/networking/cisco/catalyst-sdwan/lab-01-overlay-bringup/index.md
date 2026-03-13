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

---

# 3. Arquitectura del laboratorio

## Componentes

| Nodo | Rol |
|-----|-----|
| Manager | Gestión |
| Validator | Orquestación |
| Controller | Plano de control |
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

# 4. Hipótesis

Si:

- existe conectividad IP en el underlay
- los control components están correctamente configurados
- los WAN Edge son autorizados

entonces:

los routers WAN Edge se incorporarán correctamente al fabric SD-WAN y se establecerá conectividad entre las redes LAN de cada sitio.

---

# 5. Procedimiento

## 5.1 Despliegue del entorno en EVE-NG

Se despliegan los siguientes nodos:

- Manager
- Validator
- Controller
- Edge-1
- Edge-2

---

## 5.2 Configuración inicial de control components

Configuración de:

- system-ip
- site-id
- org-name
- VPN 512
- VPN 0

---

## 5.3 Verificación de reachability

Validar conectividad IP entre todos los control components.

---

## 5.4 Incorporación de WAN Edge

Los dispositivos WAN Edge se registran en el sistema y son autorizados.

---

## 5.5 Aplicación de configuración

Se aplica configuración básica de servicio para permitir conectividad entre LANs.

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
- Mantener consistencia en `org-name`, `site-id` y `system-ip`.
- Verificar reachability entre control components antes de incorporar edges.

---

# 11. Próximos experimentos

**Lab 02**

Templates y automatización de configuración en Cisco Catalyst SD-WAN.
