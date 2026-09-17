# 🔗 Integración: Suricata + Zeek → Wazuh

[⬅ Volver al índice](../README.md)

## Arquitectura de ingesta

Wazuh **no requiere Filebeat** para ingerir los logs de Suricata y Zeek. En esta arquitectura, los eventos se leen directamente mediante el propio **wazuh-agent instalado en localhost**, que audita el servidor SOC contra sí mismo (auto-monitoreo). Esto permite que Wazuh decodifique y correlacione los eventos antes de indexarlos, sin un salto intermedio.

## 1. Instalar el agente local (auto-monitoreo)

```bash
sudo apt-get install -y gnupg apt-transport-https

curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
  gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee -a /etc/apt/sources.list.d/wazuh.list
sudo apt-get update

WAZUH_MANAGER="127.0.0.1" sudo apt-get install -y wazuh-agent

sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

> 🔧 La ruta `.../4.x/apt/` **sí es válida** para el repositorio APT — a diferencia del script de instalación del AIO (que exige el minor version exacto, ver [docs/02](02-instalacion-wazuh.md)), el repositorio de paquetes sí acepta el comodín `4.x`. Es la única excepción documentada a esa regla.

## 2. Configurar la lectura de logs (`ossec.conf`)

Edita `/var/ossec/etc/ossec.conf` del agente local:

```xml
<ossec_config>
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>

  <localfile>
    <log_format>json</log_format>
    <location>/opt/zeek/logs/current/conn.log</location>
  </localfile>

  <localfile>
    <log_format>json</log_format>
    <location>/opt/zeek/logs/current/dns.log</location>
  </localfile>

  <localfile>
    <log_format>json</log_format>
    <location>/opt/zeek/logs/current/http.log</location>
  </localfile>

  <localfile>
    <log_format>json</log_format>
    <location>/opt/zeek/logs/current/notice.log</location>
  </localfile>

  <localfile>
    <log_format>json</log_format>
    <location>/opt/zeek/logs/current/ssl.log</location>
  </localfile>
</ossec_config>
```

```bash
sudo systemctl restart wazuh-agent
```

## 3. Configuración correcta de `eve-log`

En `/etc/suricata/suricata.yaml`, la sección de salida de alertas debe mantenerse **liviana**:

```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - alert:
            payload: no
            metadata: no
        - http
        - dns
        - tls
        - flow
        - ssh
  - stats:
      totals: yes
      threads: no
```

> 🔧 **Por qué `payload: no` / `metadata: no`:** con esos campos activados, cada evento de alerta trae demasiados campos JSON anidados, y el decodificador JSON genérico de Wazuh tiene un límite de campos por evento — **descarta el evento completo si lo supera, sin generar ningún error visible en el evento mismo**, solo en el log interno de Wazuh. El síntoma es engañoso: Suricata alerta correctamente (se ve en `eve.json`), pero nada llega al índice. Para un laboratorio, el volumen de detalle adicional no vale la pérdida silenciosa de eventos. Detalle completo del diagnóstico en [docs/08](08-troubleshooting-y-lecciones.md#pipeline-de-ingesta-wazuh--too-many-fields-for-json-decoder).

## 4. Verificar el decoder de Suricata en Wazuh

```bash
sudo grep -r "suricata" /var/ossec/ruleset/decoders/ | head
sudo grep -r "suricata" /var/ossec/ruleset/rules/0370-suricata_rules.xml | head
```

Valida que la versión del decoder corresponda a la versión de Suricata instalada; actualízalo tras cada upgrade de Suricata.

## 5. Reglas de correlación custom (Zeek)

Zeek no tiene un decoder nativo completo en Wazuh — se recomienda mapear campos clave vía decoder JSON genérico y reglas custom en `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="zeek,network,">
  <rule id="100100" level="0">
    <decoded_as>json</decoded_as>
    <field name="location">^/opt/zeek/logs</field>
    <description>Evento base de Zeek</description>
  </rule>

  <rule id="100101" level="7">
    <if_sid>100100</if_sid>
    <field name="location">notice.log</field>
    <description>Zeek Notice: posible actividad anómala detectada</description>
    <group>zeek_notice,</group>
  </rule>

  <rule id="100102" level="5">
    <if_sid>100100</if_sid>
    <field name="location">ssl.log</field>
    <field name="validation_status">^(?!ok).+</field>
    <description>Zeek SSL: certificado con validación fallida</description>
    <group>zeek_ssl,pci_dss_10.6.1,</group>
  </rule>
</group>
```

## 6. Verificación del flujo de ingesta

```bash
sudo tail -f /var/ossec/logs/ossec.log | grep -i "suricata\|zeek"
curl -k -u admin:<ADMIN_PASSWORD> "https://localhost:9200/wazuh-alerts-*/_search?q=rule.groups:suricata&pretty"
```

## Riesgos de esta configuración y remediación

| Riesgo | Impacto | Remediación |
|---|---|---|
| El propio servidor SOC actúa como agente de sí mismo | Si el `wazuh-agent` local cae, el SOC deja de recibir sus propios logs sin alertar por un canal externo | Configurar un segundo canal de heartbeat externo que consulte `ossec.log` |
| Rotación horaria de Zeek rompiendo symlinks `current/` | Filebeat/`localfile` deja de leer; ventana ciega de horas | Validar `zeekctl cron enable`; alertar sobre silencio de log (ver [docs/04](04-zeek-nsm.md)) |
| `eve.json` con demasiados campos anidados | Eventos descartados silenciosamente por el decodificador de Wazuh | `payload: no` / `metadata: no` (ver punto 3) |
| Decodificador de Wazuh desactualizado respecto a Suricata | Eventos llegan como logs planos sin parsear (`rule.id: 0` genérico) | Validar `0370-suricata_rules.xml` contra la versión instalada tras cada upgrade |
