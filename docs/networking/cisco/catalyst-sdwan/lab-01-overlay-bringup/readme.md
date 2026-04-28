# Lab 01 – Cisco Catalyst SD-WAN Overlay Bring-up
 
Full overlay bring-up of Cisco Catalyst SD-WAN 20.18.1 on EVE-NG.
 
**Control plane**: fully operational — SD-WAN Manager, Validator, and Controller with enterprise CA certificates, DTLS connections established.
 
**Data plane**: WAN Edge onboarding blocked due to C8000v bootstrap limitations in offline lab environments (no Smart Account / PnP Connect available). Multiple images tested (17.18, 17.15, 17.12) with root cause analysis documented.
 
**10 troubleshooting scenarios** documented with symptoms, diagnosis, root cause, and resolution — including interface mapping issues in EVE-NG, certificate chain failures, NETCONF connectivity, QEMU version conflicts, and C8000v controller-mode configuration persistence.
 
## Lab methodology
 
Every lab follows a structured engineering approach: objective → hypothesis → architecture → procedure → validations → results → problems found → lessons learned → best practices.
 
## Documentation
 
Full lab documentation: [index.md](./index.md)
 
## Stack
 
Cisco Catalyst SD-WAN 20.18.1 · EVE-NG · C8000v · Enterprise CA (OpenSSL) · DTLS/TLS · OMP · IPsec/BFD
