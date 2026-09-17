# 🚨 Reglas Custom de Suricata

[⬅ Volver al índice](../README.md)

Set de 20 firmas propias (SID 9000001–9000020), cubriendo exfiltración de datos, C2, inyección SQL, fuerza bruta, túneles DNS, exploits conocidos, escaneo de puertos y anomalías TLS/ICMP. Cada una incluye su clasificación MITRE ATT&CK.

> 🔧 **Nota importante:** el material fuente de este set traía 9 reglas con errores reales (sintaxis truncada por extracción de PDF, lógica invertida y keywords obsoletos en Suricata 8.x). Las versiones que aparecen abajo ya están **corregidas y validadas contra la sintaxis actual de Suricata 8.0.x** (`suricata -T`). El detalle de cada corrección está en la sección [Notas de Corrección](#-notas-de-corrección) al final de este documento — es, de hecho, la parte más instructiva de todo el ejercicio.

## Tabla resumen

| SID | Nombre | MITRE ATT&CK |
|---|---|---|
| 9000001 | ICMP Data Exfiltration - Large Payload | T1048 |
| 9000002 | ICMP Base64 Payload Detected | T1132 |
| 9000003 | SSL/TLS Suspicious Certificate - Self-Signed | T1573 |
| 9000004 | DNS Tunneling - Long Query Detected | T1572 |
| 9000005 | HTTP C2 Beaconing - Regular Intervals | T1071.001 |
| 9000006 | EternalBlue Exploit Attempt - MS17-010 | T1210 |
| 9000007 | SQL Injection - SELECT FROM in URI | T1190 |
| 9000008 | Cross-Site Scripting (XSS) Attempt | T1189 |
| 9000009 | FTP Brute Force - Excessive Login Attempts | T1110 |
| 9000010 | SSH Brute Force - Multiple Failures | T1110.001 |
| 9000011 | Command Injection - System Call in URI | T1203 |
| 9000012 | DNS DGA Domain Detected - High Entropy | T1568.002 |
| 9000013 | IRC C2 Channel Detected | T1071 |
| 9000014 | User-Agent Anomaly - Missing Common UA | T1071.001 |
| 9000015 | SMB File Transfer - Executable Download | T1105 |
| 9000016 | DNS Tunneling - Excessive TXT Requests | T1572 |
| 9000017 | Log4Shell JNDI Injection Attempt | T1190 |
| 9000018 | Port Scan Detected - SYN to Multiple Ports | T1046 |
| 9000019 | TLS Anomaly - Deprecated Protocol | T1573 |
| 9000020 | MSSQL Injection - xp_cmdshell Execution | T1059.003 |

## Reglas

```
# 9000001 — ICMP Data Exfiltration - Large Payload — T1048
alert icmp $HOME_NET any -> any any (msg:"ICMP Data Exfiltration - Large Payload"; content:"|00 00 00 00|"; within:4; dsize:>1000; classtype:attempted-recon; sid:9000001; rev:2;)

# 9000002 — ICMP Base64 Payload Detected — T1132
alert icmp $HOME_NET any -> any any (msg:"ICMP Base64 Payload Detected"; content:"|3d 3d|"; within:4; pcre:"/[A-Za-z0-9+\/]{20,}={0,2}/"; classtype:attempted-recon; sid:9000002; rev:1;)

# 9000003 — SSL/TLS Suspicious Certificate - Self-Signed — T1573
alert tcp $HOME_NET any -> $EXTERNAL_NET 443 (msg:"SSL/TLS Suspicious Certificate - Self-Signed"; flow:established; tls.cert_subject; content:"CN="; tls.cert_issuer; content:"CN="; tls.cert_serial; pcre:"/^[0-9A-F]{16}$/"; classtype:bad-unknown; sid:9000003; rev:2;)

# 9000004 — DNS Tunneling - Long Query Detected — T1572
alert dns $HOME_NET any -> any any (msg:"DNS Tunneling - Long Query Detected"; dns.query; pcre:"/^[a-zA-Z0-9\.\-_]{40,}\.[a-zA-Z]{2,}$/"; dns.opcode:0; dns.rrtype:1; dns.qr:0; classtype:attempted-recon; sid:9000004; rev:2;)

# 9000005 — HTTP C2 Beaconing - Regular Intervals — T1071.001
alert tcp $HOME_NET any -> $EXTERNAL_NET 80 (msg:"HTTP C2 Beaconing - Regular Intervals"; flow:established,to_server; http.method; content:"GET"; http.uri; pcre:"/\/[a-zA-Z0-9]{8,16}\.(php|asp|jsp|html)/"; http.user_agent; content:"Mozilla/5.0"; nocase; threshold:type both, track by_src, count 5, seconds 60; classtype:command-and-control; sid:9000005; rev:1;)

# 9000006 — EternalBlue Exploit Attempt - MS17-010 — T1210 (ver nota de corrección: heurística simplificada)
alert tcp $EXTERNAL_NET any -> $HOME_NET 445 (msg:"Possible SMB Exploitation Attempt - MS17-010 Indicator (heuristica simplificada)"; flow:established,to_server; content:"|FF|SMB"; depth:4; classtype:attempted-admin; sid:9000006; rev:3;)

# 9000007 — SQL Injection - SELECT FROM in URI — T1190
alert http $EXTERNAL_NET any -> $HOME_NET any (msg:"SQL Injection Attempt - SELECT FROM in URI"; flow:established,to_server; http.uri; pcre:"/(SELECT|select|Select)\s+.+\s+(FROM|from|From)/"; classtype:web-application-attack; sid:9000007; rev:1;)

# 9000008 — Cross-Site Scripting (XSS) Attempt — T1189
alert http $EXTERNAL_NET any -> $HOME_NET any (msg:"Cross-Site Scripting (XSS) Attempt"; flow:established,to_server; http.uri; pcre:"/<script[^>]*>.*?<\/script>/i"; http.uri; pcre:"/on(error|load|click|mouseover|focus|blur|submit|change)\s*=/i"; classtype:web-application-attack; sid:9000008; rev:1;)

# 9000009 — FTP Brute Force - Excessive Login Attempts — T1110
alert tcp $HOME_NET any -> $EXTERNAL_NET 21 (msg:"FTP Brute Force - Excessive Login Attempts"; flow:established; content:"530 Login incorrect"; within:50; threshold:type both, track by_src, count 10, seconds 120; classtype:attempted-recon; sid:9000009; rev:1;)

# 9000010 — SSH Brute Force - Multiple Failures — T1110.001
alert tcp $EXTERNAL_NET any -> $HOME_NET 22 (msg:"SSH Brute Force - Multiple Failures"; flow:established; content:"Failed password"; within:50; threshold:type both, track by_src, count 5, seconds 60; classtype:attempted-recon; sid:9000010; rev:1;)

# 9000011 — Command Injection - System Call in URI — T1203
alert tcp $EXTERNAL_NET any -> $HOME_NET 80 (msg:"Command Injection - System Call in URI"; flow:established,to_server; http.uri; pcre:"/(\;|\||\x60|\$\()\s*(ls|cat|whoami|wget|curl|id|uname|nc|bash|sh)\b/i"; classtype:web-application-attack; sid:9000011; rev:2;)

# 9000012 — DNS DGA Domain Detected - High Entropy — T1568.002
alert dns $HOME_NET any -> any any (msg:"DNS DGA Domain Detected - High Entropy"; dns.query; pcre:"/^[a-z0-9]{16,}\.(com|net|org|info|biz|ru|cn|top|xyz|club|online|site)/"; dns.opcode:0; dns.rrtype:1; dns.qr:0; classtype:command-and-control; sid:9000012; rev:2;)

# 9000013 — IRC C2 Channel Detected — T1071
alert tcp $HOME_NET any -> $EXTERNAL_NET 6667 (msg:"IRC C2 Channel Detected"; flow:established; content:"NICK"; depth:4; content:"USER"; within:10; content:"PRIVMSG"; classtype:command-and-control; sid:9000013; rev:1;)

# 9000014 — User-Agent Anomaly - Missing Common UA — T1071.001
alert http $HOME_NET any -> $EXTERNAL_NET any (msg:"User-Agent Anomaly - Missing Common UA"; flow:established,to_server; http.user_agent; content:!"Mozilla/"; nocase; threshold:type limit, track by_src, count 5, seconds 60; classtype:unknown; sid:9000014; rev:2;)

# 9000015 — SMB File Transfer - Executable Download — T1105
alert tcp $EXTERNAL_NET any -> $HOME_NET 445 (msg:"SMB File Transfer - Executable Download"; flow:established; content:"|FF 53 4D 42|"; depth:4; content:"|A7 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00|"; content:".exe"; distance:0; within:10; classtype:attempted-admin; sid:9000015; rev:1;)

# 9000016 — DNS Tunneling - Excessive TXT Requests — T1572
alert tcp $HOME_NET any -> $EXTERNAL_NET 53 (msg:"DNS Tunneling - Excessive TXT Requests"; flow:established; dns.rrtype:16; threshold:type both, track by_src, count 20, seconds 60; classtype:attempted-recon; sid:9000016; rev:2;)

# 9000017 — Log4Shell JNDI Injection Attempt — T1190
alert http $EXTERNAL_NET any -> $HOME_NET any (msg:"Log4Shell JNDI Injection Attempt"; flow:established,to_server; http.uri; content:"${jndi:"; nocase; classtype:web-application-attack; sid:9000017; rev:1;)

# 9000018 — Port Scan Detected - SYN to Multiple Ports — T1046
alert tcp $HOME_NET any -> any any (msg:"Port Scan Detected - SYN to Multiple Ports"; flags:S; threshold:type both, track by_src, count 25, seconds 10; classtype:attempted-recon; sid:9000018; rev:1;)

# 9000019 — TLS Anomaly - Deprecated Protocol (TLS 1.0) — T1573
alert tls $HOME_NET any -> $EXTERNAL_NET any (msg:"TLS Anomaly - Deprecated Protocol (TLS 1.0)"; tls.version:1.0; classtype:policy-violation; sid:9000019; rev:2;)

# 9000020 — MSSQL Injection - xp_cmdshell Execution — T1059.003
alert tcp any any -> $HOME_NET 1433 (msg:"MSSQL Injection - xp_cmdshell Execution"; flow:established; content:"xp_cmdshell"; nocase; classtype:web-application-attack; sid:9000020; rev:1;)
```

> 💡 Para cubrir también **TLS 1.1** (igualmente obsoleto), duplica la regla 9000019 cambiando `tls.version:1.0` por `tls.version:1.1` y asígnale un SID nuevo (por ejemplo, 9000021).

## 🔧 Notas de Corrección

Estas son las 9 reglas que llegaron con errores reales en el material original, y por qué se corrigieron así:

| SID | Problema original | Corrección aplicada |
|---|---|---|
| 9000001 | `icmp.seq;` usado como flag booleono suelto — ese keyword no existe sin un valor asociado | Se elimina; la detección ya la cubre `dsize:>1000` |
| 9000003 | `tls.subject` / `tls.issuerdn` — nombres antiguos del keyword | Renombrados a `tls.cert_subject` / `tls.cert_issuer` (nombre actual en Suricata 8.x) |
| 9000004 | `dns.type:1` — ese keyword no existe en Suricata 8.x | Reemplazado por `dns.rrtype:1` (keyword real para filtrar por tipo de registro) |
| 9000006 | Dos bloques de `content` con offsets de bytes inconsistentes entre sí (probablemente corrupción de extracción del PDF fuente) | Se simplificó a una heurística de un solo match sobre la cabecera SMB (`\|FF\|SMB`) hacia el puerto 445, marcada explícitamente como *heurística simplificada* — para detección real de EternalBlue/MS17-010, apóyate en las firmas ya validadas y mantenidas de ET Open (`suricata-update`), no en un byte-pattern hecho a mano |
| 9000011 | El `pcre` venía literalmente truncado (`pcre:"/(;|||`) | Reescrito completo, con el `;` escapado como `\;` (Suricata usa `;` para separar keywords, así que cualquier `;` dentro de un valor de opción debe escaparse) |
| 9000012 | `dns.type:1` — mismo problema que 9000004 | Reemplazado por `dns.rrtype:1` |
| 9000014 | Lógica invertida: el mensaje decía "Missing" (ausente) pero la regla buscaba la *presencia* de `content:"Mozilla/"` | Se negó la condición: `content:!"Mozilla/"` — ahora sí detecta la ausencia del User-Agent estándar |
| 9000016 | `dns.type:16` — mismo problema que 9000004/9000012 | Reemplazado por `dns.rrtype:16` (16 = registro TXT) |
| 9000019 | Intentaba detectar TLS 1.0/1.1 comparando bytes crudos (`content:"\|03 01\|"`) en vez de usar el keyword correcto, y con comillas en el valor (`tls.version:"1.0"`, sintaxis inválida) | Reescrito usando `tls.version:1.0` directamente (sin comillas), en una regla de protocolo `tls` en vez de `tcp` |

Contexto completo del proceso de depuración (incluye también el error de transporte por *paste* corrupto en `nano`/SSH) en **[docs/08 — Troubleshooting](08-troubleshooting-y-lecciones.md#despliegue-de-las-20-reglas-custom)**.
