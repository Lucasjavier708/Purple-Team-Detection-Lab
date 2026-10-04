
---
tags: [playbook, command-injection, web-app, api]
mitre: T1059.004, T1016, T1018
ticket: IRSOC-3
---

# PB-01 — Command Injection sobre API vulnerable

> [!NOTE] Command Injection vs actividad legítima
> No toda request con caracteres especiales sobre un endpoint de API es un ataque real.
> La pregunta clave es: **¿el parámetro devolvió output de un comando del sistema?**
>
> - **Request con `;` pero respuesta de error** → posible scanner, monitorear
> - **Request con `; whoami` y respuesta `root`** → explotación confirmada → P1 inmediato
>
> En este escenario el endpoint `/check-host` ejecuta el valor del parámetro
> `hostname` directamente en el sistema sin sanitización (`shell=True` sin validación).
> El primer indicio fue la regla `100310` (nivel 10) disparándose sobre `Ubunt-Serv-Agent`.

---

## Trigger indicators

- Alerta `100310` — Command Injection detectado (nivel 10, T1059.004)
- Alerta `100311` — Network Discovery detectado (nivel 8, T1016)
- Alerta `100312` — Host Discovery detectado (nivel 8, T1018)
- Parámetro `hostname` con caracteres de inyección (`;`, `&&`, `||`, `` ` ``)
- Output de comandos del sistema en la respuesta de la API (`root`, rutas, IPs internas)
- Actividad de reconocimiento de red interna desde el servidor Ubuntu

---

## Árbol de investigación

<br>

## Flujo de decisión — Command Injection (Regla 100310)

```text
ALERTA: Command Injection detectado — Regla 100310
│
├─► PASO 1: ¿Es actividad legítima o explotación real?
│
│   Wazuh Discover:
│   agent.name:"Ubunt-Serv-Agent" AND rule.id:100310
│
│   ├─ Request con `;` pero sin output relevante
│   │  └─► Posible scanner → Monitorear
│   │
│   └─ Output de comando del sistema confirmado
│      └─► Explotación real → PASO 2
│
├─► PASO 2: ¿Hubo actividad posterior al Command Injection?
│
│   Wazuh Discover:
│   agent.name:"Ubunt-Serv-Agent" AND rule.id:(100311 OR 100312)
│
│   ├─ Sin alertas posteriores
│   │  └─► Intento aislado → P2 → Monitorear
│   │
│   └─ Alertas 100311 / 100312 disparadas
│      └─► Reconocimiento activo → PASO 3
│
├─► PASO 3: ¿Se identificó algún servicio interno?
│
│   Revisar:
│   data.parameters.hostname
│   en las alertas 100311 y 100312
│
│   ├─ Solo reconocimiento de red
│   │  └─► Escalar a L2 → Ticket P2
│   │
│   └─ Host interno identificado (172.18.0.3)
│      └─► Posible pivoteo → PASO 4
│
└─► PASO 4: ¿Hay evidencia de acceso a servicios internos?
    
    Wazuh Discover:
    agent.name:"Ubunt-Serv-Agent"
    AND rule.id:(100402 OR 100403 OR 100404 OR 100405 OR 100406)

    ├─ Sin acceso a servicios internos
    │  └─► Contener y escalar
    │
    └─ Acceso a MariaDB confirmado
       └─► P1 → Escalar inmediatamente → PB-02
```

---

## Paso 1 — Triage L1

### Filtros Wazuh

Alerta inicial de Command Injection

agent.name:"Ubunt-Serv-Agent" and rule.id:100310

Cadena completa Fase 1

agent.name:"Ubunt-Serv-Agent" and rule.id:(100310 or 100311 or 100312)

Verificar actividad posterior hacia MariaDB

agent.name:"Ubunt-Serv-Agent" and rule.id:(100402 or 100403 or 100404 or 100405 or 100406)


### Datos a extraer por alerta

| Campo | Descripción |
|---|---|
| `data.parameters.hostname` | Comando inyectado |
| `data.endpoint` | Endpoint afectado |
| `data.result` | Output del sistema |
| `agent.ip` | IP del servidor comprometido |
| `rule.mitre.id` | Técnica MITRE asociada |
| `timestamp` | Momento exacto de la ejecución |

### Evidencia real del laboratorio

**Alerta 100310 — full_log:**
```json
{
  "timestamp": "2026-09-28T22:06:57.445182",
  "endpoint": "/check-host",
  "parameters": {"hostname": "8.8.8.8; whoami"},
  "result": "root",
  "status": "success",
  "source": "infrastructure-status-api"
}
```

**Alerta 100311 — full_log:**
```json
{
  "timestamp": "2026-09-28T22:08:05.616892",
  "endpoint": "/check-host",
  "parameters": {"hostname": "127.0.0.1; cat /proc/net/fib_trie"},
  "result": "PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data...",
  "status": "success",
  "source": "infrastructure-status-api"
}
```

**Alerta 100312 — full_log:**
```json
{
  "timestamp": "2026-09-28T22:08:53.222722",
  "endpoint": "/check-host",
  "parameters": {"hostname": "127.0.0.1; ping -c 1 172.18.0.3"},
  "result": "PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data...",
  "status": "success",
  "source": "infrastructure-status-api"
}
```

> [!NOTE] Observación técnica
> En los tres casos `data.result` muestra el output del ping base (`127.0.0.1`)
> y no el output del comando inyectado. Esto ocurre porque la API ejecuta
> el ping al hostname base y después el comando inyectado, pero el log
> registra solo el resultado del ping. El comando inyectado se ejecutó igual
> — Wazuh lo detecta a través del `full_log` completo y el campo
> `data.parameters.hostname`.

---

## Paso 2 — Análisis L2

### Análisis de logs en el servidor

```bash
# Revisar el log de la API
sudo tail -n 100 /home/ub-serv/api-infrastructure-status/logs/api.log

# Buscar requests con caracteres de inyección
grep -E "(;|&&|\|\|)" /home/ub-serv/api-infrastructure-status/logs/api.log

# Buscar comandos de reconocimiento ejecutados
grep -E "(whoami|id|ifconfig|ip addr|ping|nc|nmap|cat /proc)" \
    /home/ub-serv/api-infrastructure-status/logs/api.log

# Ver actividad del contenedor de la API
docker logs infrastructure-status-api --tail 100

# Verificar conexiones hacia la red interna Docker
sudo netstat -tnp | grep 172.18.0
```

### Verificar alcance del acceso

```bash
# Confirmar qué comandos se ejecutaron desde el contenedor
docker exec infrastructure-status-api ps aux

# Verificar conexiones activas desde el contenedor
docker exec infrastructure-status-api netstat -tn

# Comprobar si hay acceso a MariaDB desde el contenedor de la API
docker exec infrastructure-status-api mariadb \
    -h 172.18.0.3 --skip-ssl -e "SELECT 1;"

# Revisar el log de auditoría de MariaDB
sudo tail -n 50 /var/lib/docker/volumes/corp_assetdata/_data/server_audit.log
```

> [!WARNING] Si `SELECT 1` devuelve resultado desde el contenedor de la API
> el atacante puede pivotar hacia MariaDB desde el acceso obtenido.
> Activar inmediatamente **PB-02 — Acceso y Exfiltración de Base de Datos**.

### Reglas Wazuh involucradas

```xml
<rule id="100310" level="10">
  <decoded_as>json</decoded_as>
  <field name="parameters.hostname" type="pcre2">
    (?i)^(?!.*(?:ip addr|ip route|/proc/net/route|/proc/net/fib_trie|
    ping -c 1 172\.18\.0\.[0-9]+))(?:(?:.*)(?:;|&&|\|\||`|\$\()).*$
  </field>
  <description>Command Injection - Ejecución de comandos detectada</description>
  <mitre><id>T1059.004</id></mitre>
</rule>

<rule id="100311" level="8">
  <decoded_as>json</decoded_as>
  <field name="parameters.hostname" type="pcre2">
    ip addr|ip route|/proc/net/route|/proc/net/fib_trie
  </field>
  <description>Infrastructure API - Network Discovery detectado</description>
  <mitre><id>T1016</id></mitre>
</rule>

<rule id="100312" level="8">
  <decoded_as>json</decoded_as>
  <field name="parameters.hostname" type="pcre2">
    (?i)ping -c 1 172\.18\.0\.[0-9]+
  </field>
  <description>Infrastructure API - Host Discovery detectado</description>
  <mitre><id>T1018</id></mitre>
</rule>
```

> [!NOTE] Relación entre reglas
> Las reglas `100311` y `100312` son indicadores de actividad posterior
> al Command Injection confirmado por `100310`. De forma aislada pueden
> corresponder a actividad legítima de administración. En combinación
> con `100310` forman parte de la misma cadena de reconocimiento
> post-acceso y deben tratarse como un único incidente.

---

## Paso 3 — Contención


### Paso 3 — Contención

#### Si solo hay Command Injection sin actividad posterior:
- [ ] Bloquear IP atacante en el firewall de red
- [ ] Detener el contenedor de la API temporalmente
- [ ] Revisar y sanitizar el parámetro `hostname` en el código fuente
- [ ] **P2** — Escalar a L2 con ticket

#### Si hay reconocimiento de red (100311/100312):
- [ ] Bloquear IP atacante inmediatamente
- [ ] Aislar el contenedor de la API de la red interna Docker
- [ ] Revisar reglas de red entre contenedores
- [ ] **P1** — Escalar a L2 con ticket urgente

#### Si hay evidencia de acceso a servicios internos:
- [ ] Detener el contenedor de la API
- [ ] Aislar el contenedor de MariaDB
- [ ] Preservar los logs para análisis forense
- [ ] **P1** — Activar PB-02 inmediatamente


<br>


```bash
# Detener el contenedor de la API
docker stop infrastructure-status-api

# Aislar el contenedor de la red interna Docker
docker network disconnect <network_name> infrastructure-status-api

# Bloquear IP atacante en iptables
iptables -A INPUT -s 192.168.3.163 -j DROP
iptables -A OUTPUT -d 192.168.3.163 -j DROP
```

---

## Criterios de escalada

| Condición | Prioridad | Acción |
|---|---|---|
| Solo alerta 100310, sin actividad posterior | P3 | Monitorear, ticket informativo |
| Alertas 100310 + 100311 + 100312 | P2 | Escalar a L2, ticket urgente |
| Evidencia de acceso a MariaDB | P1 | Escalar inmediatamente, activar PB-02 |

---

## Cobertura MITRE ATT&CK

| Técnica | ID | Táctica | Descripción |
|---|---|---|---|
| Unix Shell | T1059.004 | Execution | Ejecución de comandos mediante inyección en parámetro `hostname` |
| System Network Configuration Discovery | T1016 | Discovery | Reconocimiento de red interna mediante `/proc/net/fib_trie` |
| Remote System Discovery | T1018 | Discovery | Identificación de hosts activos en la red interna Docker |

---

## Recomendaciones de remediación

```python
# Código vulnerable
cmd = f"ping -c 1 {hostname}"
subprocess.run(cmd, shell=True)

# Código corregido
if not re.match(r'^[a-zA-Z0-9.-]+$', hostname):
    raise HTTPException(status_code=400, detail="Hostname inválido")

subprocess.run(['ping', '-c', '1', hostname])
```

**Acciones adicionales:**
- Implementar WAF (ModSecurity + Nginx) delante de la API
- Agregar validación de input en todos los endpoints
- Revisar y restringir reglas de red entre contenedores Docker
- Habilitar logging detallado de todas las requests de la API

---

## Herramientas

| Herramienta | Propósito |
|---|---|
| Wazuh Dashboard — Discover | Análisis de alertas y correlación de eventos |
| Wazuh Threat Hunting | Revisión de actividad completa por agente |
| Docker CLI | Inspección y contención de contenedores |
| Jira — IRSOC-3 | Ticket de referencia |

→ Playbook relacionado: **PB-02 — Acceso y Exfiltración de Base de Datos MariaDB**
