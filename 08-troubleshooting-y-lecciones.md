# 🔍 Troubleshooting y Lecciones

[⬅ Volver al índice](README.md)

Esta sección no es una lista de comandos que funcionaron: es el diagnóstico detrás de cada problema real que apareció al construir este SOC. En un curso de Gestión de Incidentes, esa es la parte que realmente importa — la capacidad de leer un log de error y encontrar la causa raíz, no solo copiar comandos.

## 1. Errores de descarga del instalador de Wazuh

**Síntoma:** `bash: wazuh-install.sh: No such file or directory`.

**Causa raíz:** `curl -sO` escrito como `curl -s0` — un solo carácter (`O` mayúscula vs. cero) cambia todo el comportamiento. `-O` guarda el archivo en disco con el nombre remoto; sin él (o con el cero), `curl` solo imprime el contenido en pantalla y no queda ningún archivo.

**Lección:** revisa siempre los flags carácter por carácter cuando un comando "no hace nada" — muchas veces sí hizo algo, solo que no lo que esperabas.

## 2. Ruta inválida del script de instalación

El literal `4.x/wazuh-install.sh` **sí** funciona para los *repositorios* apt/yum (`baseurl=.../4.x/apt/`), pero **no** para la descarga directa del script instalador, que exige el número real de minor version (`4.14`).

**Lección:** no todos los "comodines de versión" de un proveedor aplican igual en todas sus rutas — hay que verificar cada una contra la documentación oficial vigente.

## 3. Verificación de hash con formato inesperado

El archivo `.sha512` que publica Wazuh trae registrada la ruta absoluta del servidor de build (`/codebuild/output/.../wazuh-install-assistant/4.14...`), no el nombre `wazuh-install.sh`. Por eso `sha512sum -c` falla con "No such file" aunque el archivo esté bien.

**Solución:** comparar manualmente `cat archivo.sha512` contra `sha512sum wazuh-install.sh` — si las cadenas hex coinciden, la integridad está validada igual.

## 4. Fallo por disco lleno → rollback automático

El instalador de Wazuh hace **rollback completo** si cualquier componente falla — quitó Manager, Indexer y Filebeat sin dejar nada a medias. Es un concepto de instaladores "transaccionales" que vale la pena reconocer. La causa real no era solo espacio del guest, sino el disco de 24 GB subdimensionado (resuelto en [docs/01](01-arquitectura-y-dimensionamiento.md)).

## 5. UFW bloqueando puertos necesarios

El instalador advierte desde el inicio (`WARNING: The system has UFW enabled...`) sobre los puertos que Wazuh necesita: `443` (dashboard), `1515`/`1516` (registro y comunicación de agentes). Se resuelve con `sudo ufw allow <puerto>/tcp` para cada uno, **antes** de instalar.

## 6. Troubleshooting de red (Netplan)

**Síntoma:** `suricata-update` no podía descargar el ruleset de Emerging Threats — `ping 8.8.8.8` daba *Destination Host Unreachable*.

**Diagnóstico:** `ip route show` reveló que la **ruta por defecto apuntaba a `enp0s8`** (interfaz de Suricata, sin salida real a Internet) en vez de a `enp0s17` (NAT). Causa raíz: **dos archivos de Netplan contradictorios** (`/etc/netplan/00-installer-config.yaml` y `50-cloud-init.yaml`), cada uno con configuración distinta para las mismas interfaces. Netplan los aplica en orden alfabético y el segundo pisa/mezcla claves del primero, generando una ruta por defecto mal colgada de la interfaz equivocada.

**Solución:** consolidar en **un solo archivo**, eliminando el duplicado:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s17:
      dhcp4: true
    enp0s8:
      addresses:
        - "192.168.1.10/24"
```

`enp0s8` queda sin gateway ni rutas — así nunca vuelve a competir por ser la ruta por defecto.

**Lección clave:** usa `sudo netplan try` (no `apply` directo) — revierte automáticamente si te quedas sin conexión en 120 segundos. Es la forma segura de aplicar cambios de red remotos sin arriesgarte a perder el acceso SSH.

## 7. Persistencia del offloading de NIC

`ethtool -K enp0s8 rx off tx off tso off gso off gro off lro off` deshabilita el *offloading* de la NIC — necesario para que Suricata/Zeek vean paquetes sin reensamblar incorrectamente —, pero **se resetea en cada reinicio o actualización de kernel**. Se persiste con una unidad systemd `oneshot`:

```ini
# /etc/systemd/system/suricata-nic-offload.service
[Unit]
Description=Deshabilita el offloading de la NIC de monitoreo antes de Suricata
Before=suricata.service
BindsTo=sys-subsystem-net-devices-enp0s8.device
After=sys-subsystem-net-devices-enp0s8.device

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/ethtool -K enp0s8 rx off tx off tso off gso off gro off lro off

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now suricata-nic-offload.service
```

## Metodología de depuración de YAML

`suricata.yaml` se rompió tres veces distintas al editar `af-packet`/`eve-log`, cada vez con el mismo patrón de síntoma engañoso: **YAML reporta el error varias líneas después del problema real**, porque el parser solo "revienta" cuando ya no puede reconciliar la estructura.

Metodología aplicada, en orden de precisión creciente:

1. `sudo suricata -T -c suricata.yaml` → da un número de línea aproximado.
2. `python3 -c "import yaml; yaml.safe_load(open('...'))"` → da **línea y columna exactas**, mucho más preciso que el propio Suricata.
3. Con la columna exacta, se localizó que `threads: auto` tenía indentación distinta a `interface: default` en la línea anterior — no eran hermanas dentro del mismo mapping.

**Lección:** cuando un YAML falla, no confíes en el número de línea del primer error — usa una herramienta de parseo genérica (Python/`yaml`) para la ubicación exacta, y revisa la indentación relativa entre claves hermanas, no solo la clave que el error señala.

## Despliegue de las 20 reglas custom

Tres tipos de error completamente distintos, todos con síntomas parecidos entre sí, pero causas totalmente diferentes:

**a) Ruta equivocada.** `default-rule-path` real era `/var/lib/suricata/rules`, no `/etc/suricata/rules` como asume la documentación genérica. Cada instalación puede tener su propia ruta; siempre verificar con `grep` en el `.yaml` en vez de asumir.

**b) Errores de sintaxis en el material original** (probablemente de extracción/OCR del documento fuente) — corregidos ya en [docs/06](06-reglas-suricata.md#-notas-de-corrección).

**c) Corrupción al pegar en `nano` por SSH.** Las líneas largas de las reglas se partieron en varias líneas físicas del archivo al pegarlas (por falta de *bracketed paste* en el cliente SSH), causando `Signature missing required value "sid"` — cada regla partida a la mitad.

**Solución:** transferir el archivo completo por `scp` desde el equipo local en vez de copiar/pegar texto largo en un editor de terminal — elimina el riesgo de raíz:

```bash
scp local_rules_completas.rules usuario@servidor:/tmp/
sudo mv /tmp/local_rules_completas.rules /var/lib/suricata/rules/local.rules
```

**d) Keywords obsoletos/inválidos en Suricata 8.0.6** (detectados solo tras tener el archivo íntegro) — ver el detalle completo en [docs/06](06-reglas-suricata.md#-notas-de-corrección): `icmp.seq` como flag booleano, `tls.subject`/`tls.issuerdn` renombrados, `dns.type` inexistente (reemplazado por `dns.rrtype`), `tls.version` sin comillas, y un `;` sin escapar dentro de un `pcre`.

**Lección general:** cuando una herramienta reporta un error, no asumas que es "el mismo problema de antes" solo porque el mensaje se parece — cada capa (transporte del archivo, sintaxis original del contenido, compatibilidad de versión del motor) es independiente y necesita su propio diagnóstico.

## Pipeline de ingesta Wazuh — "Too many fields for JSON decoder"

**Síntoma:** Suricata alertaba correctamente (visible en `eve.json`), pero nada llegaba al índice de Wazuh.

**Causa:** con `payload: yes` y `metadata: yes` en el bloque `alert` de `eve-log`, cada evento JSON trae demasiados campos anidados — el decodificador JSON genérico de Wazuh tiene un límite de campos por evento y descarta el evento completo si lo supera, sin generar una alerta de error visible en el evento mismo, solo en el log interno de Wazuh.

**Solución:** `payload: no` / `metadata: no` (config completa en [docs/05](05-integracion-suricata-zeek-wazuh.md#3-configuración-correcta-de-eve-log)).

## Pendientes reales del proyecto

- **Zeek nunca llegó a desplegarse por completo** — `zeekctl status`/`deploy` quedaron pendientes de confirmar; `/opt/zeek/logs/current/notice.log` no existía al momento de la prueba.
- **Socket de control de Suricata** (`/var/run/suricata/suricata-command.socket`) con error de permisos (`Permission denied` al crear el directorio) — bloquea `suricatasc -c "reload-rules"` sin reinicio completo del servicio.
- **La regla de correlación Wazuh `100150`** (Suricata + Zeek en la misma ventana de 120s) no se ha probado aún de forma efectiva, porque requiere que ambos motores disparen sobre el mismo tráfico — un simple `nmap -sS` no necesariamente genera evento en Zeek.
