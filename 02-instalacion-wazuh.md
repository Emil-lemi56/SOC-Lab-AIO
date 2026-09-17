# ⚙️ Instalación de Wazuh (AIO)

[⬅ Volver al índice](README.md)

Instala Wazuh Manager + Indexer + Dashboard en modo single-node sobre el mismo host.

## 1. Preparación del sistema base

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl apt-transport-https gnupg lsb-release unzip jq

# Ajuste de kernel requerido por el Indexer (OpenSearch)
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# Deshabilitar swap (evita latencia de indexación)
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Sincronización horaria — crítica para correlacionar eventos entre Suricata, Zeek y Wazuh
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

> ⚠️ Si se omite persistir `vm.max_map_count` en `/etc/sysctl.conf`, un reinicio del servidor deja al Indexer en *crashloop*. Verifícalo tras cada mantenimiento programado: `sysctl vm.max_map_count`.

## 2. Abrir puertos en el firewall

El propio instalador advierte sobre esto si UFW está activo. Ábrelos **antes** de instalar:

```bash
sudo ufw allow 443/tcp    # Dashboard
sudo ufw allow 1515/tcp   # Registro de agentes
sudo ufw allow 1516/tcp   # Comunicación de agentes
```

## 3. Instalación asistida (AIO)

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh.sha512

# Verificación de integridad
sha512sum -c wazuh-install.sh.sha512
```

> 🔧 **Usa siempre el número real de versión menor** (`4.14`, no `4.x`). El comodín `4.x` solo es válido para las URLs de los *repositorios* apt/yum — no para la descarga directa del script instalador, que exige el minor version exacto.

> 🔧 **Si `sha512sum -c` falla con "No such file or directory"** aunque el script se haya descargado bien: el archivo `.sha512` que publica Wazuh a veces registra la ruta absoluta del servidor de build en vez del nombre `wazuh-install.sh`. Compara los hashes manualmente:
> ```bash
> cat wazuh-install.sh.sha512
> sha512sum wazuh-install.sh
> ```
> Si ambas cadenas hex coinciden, el archivo es íntegro.

Ejecuta el instalador:

```bash
chmod +x wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

Al finalizar, el instalador muestra las credenciales generadas. Extráelas de inmediato:

```bash
sudo tar -xvf wazuh-install-files.tar
cat wazuh-install-files/wazuh-passwords.txt
```

⚠️ Guarda `wazuh-install-files.tar` en un vault (HashiCorp Vault, Bitwarden, KeePassXC cifrado) y **elimínalo del disco del servidor** en cuanto extraigas las credenciales.

## 4. Verificación de servicios

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager
curl -k -u admin:<ADMIN_PASSWORD> https://localhost:9200/_cluster/health?pretty
```

Salida esperada: `"status": "green"` o `"yellow"` (yellow es normal en single-node, por ausencia de réplicas).

## 5. Retención de índices (ISM)

Por defecto, el Indexer **no purga índices automáticamente**. En un nodo único con disco finito, esto llena el disco en semanas.

```bash
curl -k -u admin:<ADMIN_PASSWORD> -X PUT "https://localhost:9200/_plugins/_ism/policies/wazuh-retention" \
  -H 'Content-Type: application/json' -d '{
  "policy": {
    "description": "Retención 90 días para índices wazuh-alerts",
    "default_state": "hot",
    "states": [
      {"name": "hot", "actions": [], "transitions": [{"state_name": "delete", "conditions": {"min_index_age": "90d"}}]},
      {"name": "delete", "actions": [{"delete": {}}], "transitions": []}
    ],
    "ism_template": {"index_patterns": ["wazuh-alerts-*"], "priority": 100}
  }
}'
```

Aplícala **antes** de empezar a ingerir datos de producción, no después.

## Errores reales durante esta instalación (resumen)

| Síntoma | Causa raíz |
|---|---|
| `bash: wazuh-install.sh: No such file or directory` | `curl -sO` escrito como `curl -s0` (un carácter cambia todo: sin `-O` no se guarda archivo, solo se imprime en pantalla) |
| Instalación abortada, componentes desinstalados solos | El instalador de Wazuh hace **rollback transaccional completo** si un componente falla — no deja nada a medias. Causa real: disco de 24 GB subdimensionado |
| `sha512sum -c` falla con archivo íntegro | Formato inesperado del `.sha512` publicado (ver punto 3 arriba) |

Diagnóstico completo en **[docs/08 — Troubleshooting y Lecciones](08-troubleshooting-y-lecciones.md)**.

## Riesgos de esta configuración y remediación

| Riesgo | Impacto | Remediación |
|---|---|---|
| Certificados autofirmados del instalador AIO usados indefinidamente | MITM interno, falta de confianza en integraciones externas | Sustituir por CA interna o certificados firmados por PKI corporativa dentro de los primeros 30 días |
| Ausencia de política ISM al momento de instalación | Disco lleno → Indexer entra en modo read-only, caída total de ingesta | Aplicar la política ISM (paso 5) antes de iniciar ingesta de producción |
| Credenciales generadas en texto plano | Compromiso total del SOC si el archivo no se resguarda | Rotar credenciales de inmediato; almacenar en vault; eliminar `wazuh-passwords.txt` del servidor |
| Nodo único aloja Manager + Indexer + Dashboard | Cualquier fallo de hardware detiene la totalidad del SOC | Documentar RTO/RPO en el plan de respuesta; snapshots automatizados hacia almacenamiento externo |
| `vm.max_map_count` no persistente | Reinicio deja al Indexer en *crashloop* | Verificar persistencia tras cada mantenimiento programado |
