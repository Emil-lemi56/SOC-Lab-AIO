# 🛡️ Tutorial: Cómo Implementar un SOC (Wazuh + Suricata + Zeek)
### Laboratorio SOC AIO · Gestión de Incidentes · PUCMM

![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%204.14-1A73E8)
![Suricata](https://img.shields.io/badge/IDS-Suricata%208.0-orange)
![Zeek](https://img.shields.io/badge/NSM-Zeek%208.0%20LTS-000000)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%20Server%2024.04-E95420)
![Curso](https://img.shields.io/badge/Curso-Gesti%C3%B3n%20de%20Incidentes-6f42c1)
![Estado](https://img.shields.io/badge/Estado-En%20progreso-yellow)

> Tutorial completo, paso a paso, para desplegar un SOC All-in-One (Wazuh + Suricata + Zeek) sobre un único nodo Linux. No es solo una lista de comandos: cada fase documenta también los errores reales que aparecieron al construirlo y **por qué** ocurrieron — porque esa es la habilidad que realmente se evalúa en Gestión de Incidentes: leer un log de error y llegar a la causa raíz, no solo copiar comandos de un manual.

---

## 🎯 Objetivo del Proyecto

Diseñar e implementar un **Centro de Operaciones de Seguridad (SOC) All-in-One** capaz de:

- **Detectar** intrusiones y tráfico anómalo en tiempo real mediante un IDS de firmas (**Suricata**).
- **Registrar y analizar** el tráfico de red a nivel de protocolo mediante un motor NSM (**Zeek**).
- **Centralizar, correlacionar y visualizar** todos los eventos en un SIEM único (**Wazuh**), incluyendo el auto-monitoreo del propio servidor SOC.

Todo sobre un solo nodo Ubuntu Server, lo cual obliga a tomar decisiones reales de dimensionamiento, segmentación de red y tolerancia a fallos — no un entorno de juguete.

## 🧠 Filosofía de este tutorial

La mayoría de las guías de laboratorio muestran el comando final que funcionó y nada más. Este repositorio hace lo contrario a propósito: la sección de **[Troubleshooting y Lecciones](docs/08-troubleshooting-y-lecciones.md)** documenta los errores reales encontrados durante la implementación —desde un `curl -sO` mal escrito hasta un YAML que rompía el servicio por una indentación— junto con el método de diagnóstico usado para resolverlos. Los comandos de código obsoleto o roto que aparecían en los materiales originales del curso **no se reproducen aquí**; cada fragmento de código en este repositorio fue verificado contra la documentación oficial vigente de cada herramienta.

## 🧩 Arquitectura General

![Diagrama de arquitectura](assets/arquitectura-red.svg)

El servidor usa **dos interfaces de red con roles estrictamente separados**:

| Interfaz | Rol | Dirección IP | Notas |
|---|---|---|---|
| `enp0s8` | Captura pasiva (Suricata + Zeek) | `192.168.1.10/24` | **Sin gateway** — nunca debe tener salida a Internet ni ruta por defecto |
| `enp0s17` | Gestión / SSH / actualizaciones | DHCP (NAT) | Única interfaz con salida real a Internet |

Detalle completo de la topología y el dimensionamiento en **[docs/01](01-arquitectura-y-dimensionamiento.md)**.

## 📐 Parámetros del Entorno

| Parámetro | Valor |
|---|---|
| Sistema operativo | Ubuntu Server 24.04 LTS (VirtualBox) |
| SIEM | Wazuh 4.14.x (Manager + Indexer + Dashboard, AIO) |
| IDS | Suricata 8.0.x (modo IDS, solo alerta) |
| NSM | Zeek 8.0 (línea LTS) |
| Fuente de reglas pública | Emerging Threats — ET Open |
| Reglas custom | 20 reglas Suricata propias (SID 9000001–9000020) |
| Log de eventos Suricata | `/var/log/suricata/eve.json` |
| Logs Zeek | `/opt/zeek/logs/current/*.log` |

## 📚 Tabla de Contenidos

| # | Guía | Contenido |
|---|---|---|
| 01 | [Arquitectura y Dimensionamiento](docs/01-arquitectura-y-dimensionamiento.md) | Topología de red, sizing de referencia, redimensionamiento de disco |
| 02 | [Instalación de Wazuh (AIO)](docs/02-instalacion-wazuh.md) | Preparación del SO, instalador asistido, retención de índices, riesgos |
| 03 | [Suricata IDS](docs/03-suricata-ids.md) | Instalación, servicio, reglas, socket de control, monitoreo en vivo |
| 04 | [Zeek NSM](docs/04-zeek-nsm.md) | Instalación, configuración standalone, rotación de logs |
| 05 | [Integración Suricata + Zeek → Wazuh](docs/05-integracion-suricata-zeek-wazuh.md) | Agente local, decoders, reglas de correlación |
| 06 | [Reglas Custom de Suricata](docs/06-reglas-suricata.md) | Las 20 firmas, mapeo MITRE ATT&CK y notas de corrección |
| 07 | [Scripts de Zeek](docs/07-scripts-zeek.md) | Scripts NSM disponibles y pendientes |
| 08 | [Troubleshooting y Lecciones](docs/08-troubleshooting-y-lecciones.md) | Diagnóstico real de cada incidente de la implementación |

## 🚦 Orden Recomendado de Implementación

- [ ] 1. Definir arquitectura y redimensionar disco → [docs/01](docs/01-arquitectura-y-dimensionamiento.md)
- [ ] 2. Instalar Wazuh AIO → [docs/02](docs/02-instalacion-wazuh.md)
- [ ] 3. Instalar y configurar Suricata → [docs/03](docs/03-suricata-ids.md)
- [ ] 4. Instalar y configurar Zeek → [docs/04](docs/04-zeek-nsm.md)
- [ ] 5. Integrar ambos sensores con Wazuh → [docs/05](docs/05-integracion-suricata-zeek-wazuh.md)
- [ ] 6. Desplegar el set de reglas custom → [docs/06](docs/06-reglas-suricata.md)
- [ ] 7. Desplegar los scripts de Zeek → [docs/07](docs/07-scripts-zeek.md)
- [ ] 8. Generar tráfico de prueba y validar alertas de extremo a extremo

## ✅ Estado del Proyecto

| Componente | Estado |
|---|---|
| Wazuh AIO (Manager + Indexer + Dashboard) | ✅ Operativo |
| Suricata + 20 reglas custom | ✅ Operativo |
| Ingesta Suricata → Wazuh | ✅ Operativo |
| Zeek | 🟡 Pendiente de despliegue completo |
| Correlación Suricata + Zeek (regla 100150) | 🟡 Pendiente de validar |
| Socket de control de Suricata (`suricatasc`) | 🟡 Error de permisos por resolver |

Detalle en [docs/08](docs/08-troubleshooting-y-lecciones.md#pendientes-reales-del-proyecto).

## 🗺️ Roadmap — Próximos Pasos

Para expandir el SOC más allá de la capa de detección de red (NIDS/NSM):

- **MISP** — threat intelligence real, para alimentar los scripts de intel de Zeek (`intel-dns`, `intel-ssh`, `intel-tor`).
- **TheHive + Cortex** — gestión de casos e investigaciones, cerrando el ciclo detección → respuesta.
- **Shuffle / n8n** — SOAR, automatización de respuesta.
- **Velociraptor / osquery** — forense y threat hunting de endpoint.
- **OpenVAS / Greenbone** — gestión de vulnerabilidades.
- **T-Pot / Cowrie** — honeypots, tráfico de ataque real de alta confianza.
- **YARA** — firmas de malware, integrable con Wazuh (FIM) y Suricata (file extraction).

## ✍️ Autor

*Emil — Matrícula 10166165*
Proyecto del curso **Gestión de Incidentes**, PUCMM.
