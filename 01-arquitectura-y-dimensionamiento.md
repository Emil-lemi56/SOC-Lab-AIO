# 🧩 Arquitectura y Dimensionamiento

[⬅ Volver al índice](README.md)

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

![Diagrama de arquitectura](https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1&dark=auto#R%3Cmxfile%3E%3Cdiagram%20name%3D%22P%C3%A1gina-1%22%20id%3D%220RJ-V3WzvnKvHmTl5VYD%22%3E7VpLc%2BI4EP4te6Aqc4DyG3MkD2ZStZmkhuzOzF62hCWMJsbyyiJAfv22bPmJA84Ew9TWXsBqtVpSf92tVts982q5%2BchRtLhjmAQ9Q8ObnnndMwxdMzX4k5StogxdRfE5xYpWEKb0hWRDFXVFMYkrjIKxQNCoSvRYGBJPVGiIc7auss1ZUJ01Qj7ZIUw9FOxSv1IsFinVNYYF%2FROh%2FiKbWXdGac8SZcxqJ%2FECYbYukcybnnnFGRPp03JzRQKpvUwv6bjJK735wjgJRZsB3p%2BP1pfJZXS7%2BNt99O8evgWa2zdTKQTvaKEQq0gxW3GP7JGV8YltpjwpdqqajIsF81mIgpuCesnZKsRErlCDVsHzO2MREHUg%2FiBCbJVhoJVgQFqIZaB6yYaKb6Xn71LUwFat642SnDS2WSMUfPut3CiNks1iWNLKxinzQ9wnYo8ejBxccAvClgRkwDhOAiToc1XPSJmnn%2FMVCMKDAvENgO7Br8Cl0LpU23pBBZlGKAF3DX5c1fCcBsEVCxhPxppY8wiB%2FV3GgrMnUuqxHVsfklxPzyhYqfluQ0F4CEpLOwgXZFNa266uFiWXcpT%2FrAv30zOfUlKGqpkFmZHWkXbtI7qL8b%2B7JHqwzuouzn8F0QLF72UQGxH1Vvw5n%2FMVeA%2BAi1G8yCNIS6SddyKthj4wCsjkzm%2BPqt7fd2ven65LjarZS76Mnzcho%2BuIO9OISZymiKsRV3PdpojbM5xASJjoMzz6IsEv0mJ9mPXAXkudDfyfx4%2F9609XD60HfCSxoCzsyeA8gd%2Fp9NNrY7s5Bk53Dlidn7KIuHMP6D5HmMIkpb4hQQ7RmuzB8Vwym7e0h6%2FoZbUALmugw3a0i%2FHt%2FYfWWN%2BhEDJf3pr%2FFjSxeQP%2FNYSXGUMcS2syx5Zlth6akWa8Tjk4NFFJHzaWxJeLMQTb%2FpKFVDBOWHvlBGRLQszksfJMBj9i5RPAqgXMj%2BV1iMDPX4Q8HdFBcmsveYgxqnqIZdc8xLC7OliNIx6sznkPVqN8smZdh3KlIj1qc5y2PEF1rdkOTpQsHfO2eGZMh%2B0xbYvNKz56GmyczrMQ5LnYbDp1DNOybPyWLMRtHUr1kTHQHXegD%2FQRMBvypOrLTVEZUz8iQdZo21rakiVR%2BYGzJY29FVPC7ufzgCF51NxPJt2mLEb95upWA3JnGcvoiK7r%2Fjr3nKN47nvvJe8Cxu3acecIj7Dd5LiIWLppNDnuFyK9IUt9EEZdpeyjWkLSN7pyAN06ogdk1ekzuUD5on%2BgeNPSBc5bhNE694G5hw23%2BTo1nyNba7xO4eFopmlN%2FtFwtkxXnHpIoFPcU26vp%2BroKi4Yp7xo108tQ%2BvMa49ZcdXP%2BobiLV77SoGueqfQj3OneK%2FrJ0PHnMs8LGeIZLUtLkmuFe1GZs2CzNrLqwP8pr2f3x7Z%2B%2FjhIV1xy9lOXFLU9xj5kepLNnGx1RwQR0OsDYdNAdE1ZqbjtAyI%2B8obxwyGn6d3KhjKd8CSPQTDm8RxMAiY33FgtGvpjHOywNgqb4TtiaodNJlJFeeQhaRmMIqEAuqH0PSIfJ8HBKlAOPSCsepYUowbguar1hJHKKws2PlnJV%2BHX3rpxGMZh%2FzZBSgRtKRlfx8SmdqchaI%2FR0sabFNWGI%2BWckNKypT4TBba%2Frit96SCxpyiIH2MURj3Y8LpvCQ7TkK6lKxr0abckS5Y9oSML0FI0feMQCr8g0qQWHH57cJePg9Fr7GslfHJTkslIloAhw3hfVCdR0N%2FdyTj0QI2k3akyb0mzaCv0BsnCkdclLoogBWqifKdJj2Cg6w5iM8mSgwhdYLk24rSLGvGcXVduSzYyuyJgjgpMzW3vnKjCl9inYmMXL3KRrMFYeIxjuR7hr5YUO8pJLFaBYXLCs20UOctIbaXr2R4FT5M4yhA26wnoKE0rN%2FoMoKMAIVKmbKMIGqacnKTS0NPrPLE%2FirCSEgpFzeP8HsfkTC1xefeldkbXyOZ9qTvbMql59Rn3hHGzMNVCbP2IsXqKooZrTL%2FXzSKTYLVD1lCSor5gCWL8zATzuLoiPjUq%2FhaDZ5hZ%2Fi0ern5i%2BIz9sQKhL1AQErewCQ4ceIHKD4WMLpVRcY5HTJvqcWX7jshHsuv4qR%2BQQ8x9arAHTy4DykpS4garw8lzdl7FPNTNwpzWM34Ldca2G%2FI%2Bc1aJaoYny8hvWDuZP07onb8c1dU198kvKXY9bPW0fAJyHlMZff2dzwsHb07LKFZfP6Zshdf0Zo3%2FwI%3D%3C%2Fdiagram%3E%3C%2Fmxfile%3E)

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
