# 🧩 Arquitectura y Dimensionamiento

[⬅ Volver al índice](../README.md)

## Visión general del stack

La arquitectura AIO (All-in-One) combina cuatro procesos con perfiles de consumo muy distintos:

- **Wazuh Indexer (OpenSearch)** — el más pesado en RAM y disco, por I/O de indexación.
- **Wazuh Manager** — correlación de eventos y decodificación.
- **Suricata** y **Zeek** — sensibles a CPU durante la captura de paquetes; compiten entre sí y con el Indexer por los mismos núcleos en picos de tráfico.

## Especificación de referencia (producción de pequeña escala)

Para un entorno de hasta ~50-80 agentes y un enlace monitoreado de hasta 1 Gbps con tráfico moderado:

| Recurso | Mínimo | Recomendado |
|---|---|---|
| CPU | 8 vCPU | 12–16 vCPU |
| RAM | 16 GB | 32 GB |
| Disco (SO) | 50 GB SSD | 100 GB SSD |
| Disco (datos/índices) | 200 GB SSD | 500 GB SSD NVMe |
| NIC de monitoreo | 1x dedicada, modo promiscuo | 1x dedicada + SPAN/TAP |
| Kernel | 5.15+ | 6.x |

> **Regla de oro (innegociable):** el heap de la JVM del Indexer nunca debe exceder el **50%** de la RAM física total del servidor, y **nunca debe superar 31 GB** (límite de *compressed oops* de la JVM). Verifícalo en `/etc/wazuh-indexer/jvm.options`.

### Recursos reales usados en este laboratorio

Este es un laboratorio académico, no el entorno de producción de la tabla anterior: corre sobre una única VM de VirtualBox con dos interfaces de red (sin NIC dedicada por SPAN/TAP). El disco raíz partió subdimensionado en 24 GB y se llevó a 100 GB — ver el procedimiento más abajo.

## Topología de Red

![Diagrama de arquitectura](../assets/arquitectura-red.svg)

| Interfaz | IP | Rol |
|---|---|---|
| `enp0s8` | `192.168.1.10/24` — **sin gateway** | Interfaz dedicada de monitoreo (captura pasiva) para Suricata y Zeek |
| `enp0s17` | DHCP (NAT de VirtualBox) | Gestión/SSH — única interfaz con salida real a Internet |

> ⚠️ **`enp0s8` nunca debe tener ruta por defecto ni gateway.** Cuando esto ocurre, `suricata-update` y cualquier salida a Internet fallan de forma confusa (la ruta por defecto termina "colgada" de la interfaz que no tiene salida real). El incidente real que causó esto — y su diagnóstico — está documentado en **[docs/08](08-troubleshooting-y-lecciones.md#4-troubleshooting-de-red-netplan)**.

## Redimensionamiento de Disco (VM subdimensionada)

Un disco virtual **no se expande "mágicamente" solo con crecer el `.vdi`**: hay que propagar el cambio a través de cada capa, desde el host hasta el filesystem. Cada capa es independiente — omitir una deja las capas superiores sin ver el espacio nuevo aunque las inferiores ya lo tengan.

**1. Host (fuera de la VM, con la VM apagada) — solo funciona con discos dinámicos, no de tamaño fijo:**

```bash
VBoxManage modifymedium disk "/ruta/al/disco.vdi" --resize 102400   # tamaño destino en MB
```

**2. Guest (dentro de la VM) — cadena completa, en este orden:**

```bash
# 1) Extiende la partición hasta ocupar el espacio nuevo del disco
sudo growpart /dev/sda 3          # ajusta disco/número de partición según `lsblk`

# 2) Extiende el physical volume de LVM
sudo pvresize /dev/sda3

# 3) Extiende el logical volume
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv

# 4) Extiende el filesystem ext4 encima
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

# Verificación
df -h /
lsblk
```

> 💡 Si alguno de estos cuatro pasos se omite, `df -h` seguirá mostrando el tamaño viejo aunque el paso anterior haya funcionado — cada capa (disco virtual → partición → LVM → filesystem) tiene que "enterarse" del cambio por separado.
