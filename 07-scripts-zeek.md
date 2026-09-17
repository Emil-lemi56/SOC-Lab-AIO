# 🧵 Scripts de Zeek

[⬅ Volver al índice](../README.md)

El plan original contempla 20 scripts/firmas para Zeek, a cargar desde `$ZEEKPATH/site/` mediante `@load`. A la fecha de este tutorial, **2 de esos 20 scripts están completos y listos para usar**; el resto sigue pendiente de redacción y validación (ver [Estado y pendientes](#estado-y-pendientes) abajo) — se documenta así, con honestidad, en vez de presentar contenido no probado como terminado.

## 1. Detección de Exfiltración ICMP

```zeek
# detect-icmp-exfil.zeek
@load base/frameworks/notice

module ICMP;

export {
    redef enum Notice::Type += {
        ICMP_DataExfil,
        ICMP_AsymPayload,
        ICMP_UnpairedEchoReply
    };

    const EXFIL_BYTES_THRESHOLD = 100000 &redef;
    const EXFIL_TIME_WINDOW = 60sec &redef;
}

global icmp_bytes: table[addr] of count = table();

event icmp_echo_request(c: connection, code: count, id: count, seq: count, payload: string)
{
    local src = c$id$orig_h;
    local bytes = |payload|;
    icmp_bytes[src] = (icmp_bytes[src] ? icmp_bytes[src] + bytes : bytes);

    if (icmp_bytes[src] > EXFIL_BYTES_THRESHOLD) {
        NOTICE([$note=ICMP_DataExfil,
                $src=src,
                $msg=fmt("Host %s sent %d bytes of ICMP traffic in %s", src, icmp_bytes[src], EXFIL_TIME_WINDOW)]);
        delete icmp_bytes[src];
    }
}

event icmp_echo_reply(c: connection, code: count, id: count, seq: count, payload: string)
{
    local src = c$id$orig_h;
    local dst = c$id$resp_h;
    local bytes = |payload|;

    if (bytes > 100) {
        NOTICE([$note=ICMP_AsymPayload,
                $src=src,
                $dst=dst,
                $msg=fmt("Asymmetric ICMP payload detected: %d bytes", bytes)]);
    }
}

event zeek_init()
{
    schedule EXFIL_TIME_WINDOW { delete_icmp_state() };
}

event delete_icmp_state()
{
    delete icmp_bytes;
    schedule EXFIL_TIME_WINDOW { delete_icmp_state() };
}
```

**Descripción:** detecta exfiltración ICMP mediante tres técnicas: (1) volumen de tráfico ICMP por host, (2) asimetría de payload entre echo-request y echo-reply, (3) respuestas echo-reply sin solicitud previa. **MITRE:** T1048.

## 2. Detección de Túneles ICMP (Pingback C2)

```zeek
# detect-icmp-tunnel.zeek
@load base/frameworks/notice

module ICMPTunnel;

export {
    redef enum Notice::Type += {
        ICMP_Tunnel_Detected
    };

    const TUNNEL_PATTERN = /[A-Za-z0-9+\/]{40,}/ &redef;
}

event icmp_echo_request(c: connection, code: count, id: count, seq: count, payload: string)
{
    if (|payload| > 200 && TUNNEL_PATTERN in payload) {
        NOTICE([$note=ICMP_Tunnel_Detected,
                $src=c$id$orig_h,
                $dst=c$id$resp_h,
                $msg=fmt("Potential ICMP tunnel detected (payload size: %d bytes)", |payload|)]);
    }
}
```

**Descripción:** detecta túneles ICMP mediante la búsqueda de payloads largos (>200 bytes) con patrones base64, característico de herramientas como Pingback C2. **MITRE:** T1572.

## Estado y pendientes

| # | Script | Estado |
|---|---|---|
| 1 | Detección de Exfiltración ICMP | ✅ Completo |
| 2 | Detección de Túneles ICMP (Pingback C2) | ✅ Completo |
| 3 | Detección de Flujo ICMP Anómalo | 🟡 Incompleto en el material fuente — solo el encabezado (`@load base/frameworks/notice`) llegó a redactarse |
| 4–20 | (DNS, C2, exfiltración, fuerza bruta, exploits conocidos, anomalías de protocolo, etc. — según el alcance planeado en la introducción del proyecto) | ⬜ No redactados/validados aún |

> Al completar el script 3 y los siguientes, sigue el mismo patrón que los scripts 1 y 2: `module` propio, `export` con el/los `Notice::Type` nuevos, un evento de Zeek como disparador, y `NOTICE(...)` con `$note`, `$src`/`$dst` y `$msg`. Cada script nuevo debe probarse contra un PCAP real o tráfico generado en laboratorio antes de darlo por válido — igual que se hizo con las reglas de Suricata en [docs/06](06-reglas-suricata.md).

Zeek en general —despliegue incluido— quedó pendiente de confirmar en este proyecto; ver [docs/08 — Pendientes reales](08-troubleshooting-y-lecciones.md#pendientes-reales-del-proyecto).
