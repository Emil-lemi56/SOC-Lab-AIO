# 🦈 Suricata IDS

[⬅ Volver al índice](README.md)

Suricata es un motor **IDS/IPS** de código abierto: inspecciona el tráfico de red en tiempo real y lo compara contra un conjunto de **reglas (firmas)**. Cuando un paquete o flujo coincide con una firma, genera una alerta en formato JSON (`eve.json`).

En este laboratorio corre en **modo IDS** (solo alerta, no bloquea) sobre la interfaz `enp0s8`, combinando dos fuentes de reglas:

- **Emerging Threats (ET Open)** — ruleset público, se actualiza con `suricata-update`.
- **Reglas custom propias** — 20 firmas estáticas, ver [docs/06](06-reglas-suricata.md).

## 1. Instalación

```bash
# Repositorio oficial del proyecto (versión estable más reciente)
sudo add-apt-repository -y ppa:oisf/suricata-stable
sudo apt update
sudo apt install -y suricata

# Verificación
suricata --build-info | head -5
suricata -V
```

## 2. Actualización de reglas (Emerging Threats)

`suricata-update` descarga y actualiza el ruleset público ET Open. **No toca las reglas custom** — esas son estáticas y se editan a mano.

```bash
sudo suricata-update
sudo suricata-update list-sources
sudo suricata-update enable-source et/open
```

Actualización diaria automática (3:00 a.m.) con recarga en caliente:

```bash
echo "0 3 * * * root suricata-update && suricatasc -c 'reload-rules'" | sudo tee /etc/cron.d/suricata-update
```

## 3. Gestión del servicio (systemd)

| Comando | Qué hace |
|---|---|
| `sudo systemctl enable --now suricata` | Habilita al arranque **y** lo inicia de inmediato |
| `sudo systemctl restart suricata` | Reinicia (necesario tras cambios en `suricata.yaml`) |
| `sudo systemctl status suricata --no-pager` | Estado actual |
| `sudo journalctl -u suricata -f` | Sigue el log del servicio en vivo |

## 4. Validación de configuración

> ⚠️ **Siempre** valida antes de recargar o reiniciar. Un YAML roto tumba el servicio.

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```

Si el error no es claro, valida el YAML puro con Python — da línea y columna exactas (más preciso que el propio Suricata):

```bash
python3 -c "import yaml; yaml.safe_load(open('/etc/suricata/suricata.yaml'))"
```

Metodología completa de depuración de YAML en [docs/08](08-troubleshooting-y-lecciones.md#metodología-de-depuración-de-yaml).

## 5. Interfaz de red / captura de paquetes

Desactivar el *offloading* de la NIC evita que la tarjeta reensamble paquetes antes de que Suricata los vea — un reensamblado incorrecto puede ocultar un ataque.

```bash
ip -br link show

sudo ethtool -K enp0s8 rx off tx off tso off gso off gro off lro off
sudo ethtool -k enp0s8 | grep -E "tcp-segmentation|generic-segmentation|generic-receive|large-receive"
ethtool -l enp0s8   # colas RSS de la NIC (relevante si usas varios threads)
```

Esta configuración **se resetea en cada reinicio o actualización de kernel**. Persístela con una unidad systemd — ver [docs/08](08-troubleshooting-y-lecciones.md#persistencia-del-offloading-de-nic).

## 6. Reglas — ubicación y despliegue

```bash
grep -n "^default-rule-path" /etc/suricata/suricata.yaml   # confirma la ruta REAL de reglas
grep -n "^rule-files:" -A5 /etc/suricata/suricata.yaml     # qué archivos están cargados

sudo nano /var/lib/suricata/rules/local.rules   # editar reglas custom

wc -l /var/lib/suricata/rules/local.rules
grep -c "sid:" /var/lib/suricata/rules/local.rules

sudo suricatasc -c "ruleset-stats"
sudo suricatasc -c "reload-rules"    # recargar SIN reiniciar el proceso
```

> 🔧 Nunca asumas la ruta de `default-rule-path` — cada instalación puede diferir de lo que dice la documentación genérica (en este entorno es `/var/lib/suricata/rules`, no `/etc/suricata/rules`). Verifícala siempre con `grep`.

## 7. Socket de control (`suricatasc`)

```bash
grep -A3 "^unix-command:" /etc/suricata/suricata.yaml
ls -la /var/run/suricata/suricata-command.socket

sudo suricatasc -c "iface-stat enp0s8"   # paquetes procesados / drops
sudo suricatasc -c "iface-list"
sudo suricatasc -c "version"
sudo suricatasc -c "ruleset-stats"
```

## 8. Monitoreo de alertas en vivo

```bash
# Solo alertas
sudo tail -f /var/log/suricata/eve.json | grep --line-buffered '"event_type":"alert"'

# Formateado con jq
sudo tail -f /var/log/suricata/eve.json | jq 'select(.event_type=="alert")'

# Filtrar por SID específico
sudo tail -f /var/log/suricata/eve.json | grep --line-buffered '"signature_id":9000018'

# Estadísticas del motor
sudo tail -f /var/log/suricata/stats.log
```

## 9. Generar tráfico de prueba

Así confirmas que las reglas custom realmente disparan (confirma primero que `tail -f eve.json` ya está corriendo en otra terminal, para no perderte la alerta):

| Prueba | Comando | SID | Regla disparada | MITRE ATT&CK |
|---|---|---|---|---|
| Port scan | `sudo nmap -sS -p 1-1000 <IP_DEL_SERVIDOR>` | 9000018 | Port Scan Detected | T1046 |
| Fuerza bruta SSH | `for i in {1..6}; do ssh usuario_invalido@<IP_DEL_SERVIDOR> -o ConnectTimeout=2; done` | 9000010 | SSH Brute Force | T1110.001 |
| Exfiltración ICMP | `ping -s 1200 -c 3 <IP_DEL_SERVIDOR>` | 9000001 | ICMP Data Exfiltration | T1048 |

## 10. Diagnóstico general

```bash
ps aux | grep suricata
grep -i "^User=\|^Group=" /usr/lib/systemd/system/suricata.service
ls -ld /var/run/suricata
grep -n "interface:" /etc/suricata/suricata.yaml
grep -n "af-packet" /etc/suricata/suricata.yaml
```

> 💡 Tras un `suricata-update`, el resumen final indica cuántas reglas quedaron activas: *"Enabled X rules. Disabled Y rules."*

## 11. Flujo típico de cambio de configuración

- [ ] 1. Editar `suricata.yaml` o el archivo de reglas correspondiente
- [ ] 2. Validar SIEMPRE antes de aplicar → `sudo suricata -T -c /etc/suricata/suricata.yaml`
- [ ] 3a. Si solo cambiaron **reglas** → recarga en caliente: `sudo suricatasc -c "reload-rules"`
- [ ] 3b. Si cambió el **`suricata.yaml`** (af-packet, outputs, etc.) → reinicio completo: `sudo systemctl restart suricata`
- [ ] 4. Confirmar que quedó sano → `sudo systemctl status suricata --no-pager` y `sudo suricatasc -c "iface-stat enp0s8"`

## 💡 Buenas prácticas

- Nunca reinicies ni recargues sin validar primero (`-T -c`).
- Mantén las reglas custom en un archivo separado del ruleset de ET Open, para no perderlas en una actualización.
- Revisa `iface-stat` periódicamente: los *drops* de paquetes indican que la interfaz no está dando abasto.
- El offloading de la NIC debe quedar desactivado mientras Suricata monitorea esa interfaz.
- Configura la salida `eve-log` según [docs/05](05-integracion-suricata-zeek-wazuh.md#configuración-correcta-de-eve-log) — un exceso de campos anidados hace que Wazuh descarte eventos silenciosamente.
