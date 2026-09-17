# 🦉 Zeek (Network Security Monitor)

[⬅ Volver al índice](../README.md)

Zeek complementa a Suricata: mientras Suricata detecta por firmas, Zeek registra **metadatos estructurados de cada conexión y protocolo** (`conn.log`, `dns.log`, `http.log`, `ssl.log`, `files.log`, `notice.log`), habilitando threat hunting y correlación posterior.

## 1. Instalación (Ubuntu Server 24.04 LTS)

```bash
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | \
  sudo tee /etc/apt/sources.list.d/security:zeek.list

curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | \
  gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null

sudo apt update
sudo apt install -y zeek-8.0    # fija la línea LTS 8.0.x, en vez del paquete genérico "zeek"
```

> 🔧 La ruta del repositorio debe coincidir **exactamente** con tu versión de Ubuntu (`xUbuntu_24.04`, no `xUbuntu_22.04`). Usar el repo de una versión distinta es la causa más común de que `apt` no encuentre el paquete o instale una build incompatible con las librerías del sistema.

```bash
export PATH=/opt/zeek/bin:$PATH
echo 'export PATH=/opt/zeek/bin:$PATH' | sudo tee -a /etc/profile.d/zeek.sh
zeek --version
```

## 2. Configuración (modo standalone)

Un nodo único usa modo `standalone`, no cluster. Edita `/opt/zeek/etc/node.cfg`:

```ini
[zeek]
type=standalone
host=localhost
interface=af_packet::enp0s8
```

Ajusta `/opt/zeek/etc/networks.cfg` con los rangos internos monitoreados — evita que Zeek clasifique erróneamente tráfico interno como externo en `conn.log`, lo cual contamina cualquier baseline de threat hunting posterior:

```
192.168.1.0/24
```

## 3. Salida en JSON

Por defecto Zeek escribe TSV. Para simplificar la ingesta posterior (Filebeat/wazuh-agent), fuerza salida JSON:

```bash
echo '@load policy/tuning/json-logs.zeek' | sudo tee -a /opt/zeek/share/zeek/site/local.zeek
sudo zeekctl deploy
sudo zeekctl status
```

## 4. Rotación de logs y symlinks

Zeek rota logs a archivos con timestamp **cada hora** por defecto (vía `zeekctl cron`), lo que puede romper el seguimiento de `current/*.log` como symlinks estáticos si no se gestiona bien.

```bash
sudo zeekctl cron enable   # asegura que zeekctl cron esté activo
```

> 🔧 Si `zeekctl cron` no está activo, los symlinks de `current/` dejan de apuntar al log activo tras la primera rotación, y cualquier `<localfile>` de Wazuh que los siga deja de recibir eventos **sin ningún error visible**. Configura una alerta de "silencio de log" (ausencia de eventos nuevos por más de 15 minutos es anómala).

## Estado en este laboratorio

🟡 **Pendiente.** En este proyecto, Zeek fue instalado y configurado pero el despliegue (`zeekctl deploy`) y la generación de `notice.log` no llegaron a confirmarse antes del cierre de esta fase. Ver [docs/08 — Pendientes reales del proyecto](08-troubleshooting-y-lecciones.md#pendientes-reales-del-proyecto).

## Riesgos de esta configuración y remediación

| Riesgo | Impacto | Remediación |
|---|---|---|
| Zeek en modo standalone sin `networks.cfg` correcto | Clasificación errónea local/remoto contamina `conn.log`, afecta el baseline de threat hunting | Revisión trimestral de rangos CIDR internos, especialmente tras cambios de segmentación |
| Suricata y Zeek compiten por los mismos núcleos de CPU que el Indexer en picos de tráfico | Pérdida de paquetes (*drops*) → ceguera parcial de detección en momentos críticos | Aislar núcleos con `taskset`/cgroups dedicados a captura; monitorear `capture.kernel_drops` como KPI operativo |
| Rotación horaria rompiendo symlinks de `current/` | Ventana ciega de horas sin nuevos eventos de `notice.log` | Validar `zeekctl cron enable`; alertar sobre silencio de log |
