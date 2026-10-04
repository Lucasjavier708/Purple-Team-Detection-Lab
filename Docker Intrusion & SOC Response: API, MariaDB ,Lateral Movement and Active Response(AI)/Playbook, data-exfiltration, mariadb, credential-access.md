
---
tags: [playbook, data-exfiltration, mariadb, credential-access]
mitre: T1087, T1078, T1033, T1213
ticket: IRSOC-4
---

# PB-02 — Acceso y Exfiltración de Base de Datos MariaDB

> [!NOTE] Acceso a base de datos vs actividad legítima de administración
> No toda consulta SQL sobre MariaDB es maliciosa.
> La pregunta clave es: **¿el acceso proviene de un host autorizado
> y el usuario tiene razón legítima para ejecutar esa consulta?**
>
> - **Consulta desde IP de administrador conocida** → actividad legítima, monitorear
> - **Consulta desde el contenedor de la API (172.18.0.2)** → acceso no autorizado → P1 inmediato
> - **`SELECT * FROM credentials`** → exfiltración confirmada → P1, escalar inmediatamente
>
> En este escenario el acceso a MariaDB se realizó desde el contenedor
> de la API comprometida (`172.18.0.2`), no desde un cliente administrativo
> legítimo. El primer indicio fue la regla `100402` disparándose con
> origen `172.18.0.2`.

---

## Trigger indicators

- Alerta `100402` — Enumeración de usuarios MariaDB (nivel 8, T1087)
- Alerta `100403` — Acceso con credenciales válidas (nivel 8, T1078)
- Alerta `100404` — Identificación del usuario actual (nivel 8, T1033)
- Alerta `100405` — Enumeración de bases de datos y tablas (nivel 8, T1213)
- Alerta `100406` — Acceso a tabla `credentials` (nivel 12, T1213) 🔴
- Consultas SQL desde host no autorizado (`172.18.0.2` — contenedor API)
- Usuario `TCFD` o `labadmin` ejecutando queries sobre `mysql.user` o `corporate_assets`
- Actividad precedida por alertas de Command Injection (PB-01)

---

## Árbol de investigación

ALERTA: Enumeración de usuarios MariaDB — Regla 100402
 │
 ├─► PASO 1: ¿El acceso proviene de un host autorizado?
 │   Revisar campo: data.mariadb.host
 │         │
 │         ├─ Host administrativo conocido → Posible FP → Verificar con el equipo
 │         └─ 172.18.0.2 (contenedor API) → Acceso no autorizado → PASO 2
 │
 ├─► PASO 2: ¿Qué usuario está ejecutando las consultas?
 │   Revisar campo: data.mariadb.user
 │         │
 │         ├─ Usuario root sin contraseña → Acceso anónimo → P1 inmediato
 │         └─ Usuario TCFD o labadmin → Credenciales comprometidas → PASO 3
 │
 ├─► PASO 3: ¿Hubo progresión hacia bases de datos sensibles?
 │   Wazuh Discover:
 │   agent.name:"Ubunt-Serv-Agent" and rule.id:(100404 or 100405)
 │         │
 │         ├─ Solo enumeración de usuarios → P2, escalar a L2
 │         └─ Acceso a corporate_assets → Datos sensibles en riesgo → PASO 4
 │
 └─► PASO 4: ¿Se accedió a la tabla credentials?
     Wazuh Discover:
     agent.name:"Ubunt-Serv-Agent" and rule.id:100406
 │
 ├─ Sin alerta 100406 → Contener y escalar P2
 └─ Alerta 100406 disparada → Exfiltración confirmada → P1 inmediato 


---

## Paso 1 — Triage L1

### Filtros Wazuh 

Ver todas las alertas de acceso a MariaDB

agent.name:"Ubunt-Serv-Agent" and rule.id:(100402 or 100403 or 100404 or 100405 or 100406)

Verificar si hubo acceso a credenciales (alerta crítica)

agent.name:"Ubunt-Serv-Agent" and rule.id:100406

Ver la cadena completa incluyendo Command Injection previo

agent.name:"Ubunt-Serv-Agent" and rule.id:(100310 or 100402 or 100403 or 100404 or 100405 or 100406)


### Datos a extraer por alerta

| Campo | Descripción |
|---|---|
| `data.mariadb.host` | IP origen de la conexión |
| `data.mariadb.user` | Usuario que ejecutó la consulta |
| `data.mariadb.query` | Query ejecutada |
| `data.mariadb.database` | Base de datos afectada |
| `data.mariadb.event_type` | Tipo de evento (QUERY, CONNECT) |
| `decoder.name` | Debe ser `mariadb-server-audit-fields` |
| `timestamp` | Momento exacto de la consulta |

### Evidencia real del laboratorio

**Alerta 100402 — Enumeración de usuarios MariaDB:**

```json
{
  "full_log": "20260928 22:13:03,4ca86714d046,root,172.18.0.2,8,10,QUERY,mysql,'SELECT user FROM mysql.user',0"
}
```

**Alerta 100403 — Acceso con credenciales válidas (TCFD):**

```json
{
  "full_log": "20260928 22:14:27,4ca86714d046,TCFD,172.18.0.2,9,12,QUERY,,'SELECT 1',0"
}
```

**Alerta 100404 — Identificación de usuario actual:**

```json
{
  "full_log": "20260928 22:15:33,4ca86714d046,TCFD,172.18.0.2,10,14,QUERY,,'SELECT CURRENT_USER()',0"
}
```

**Alerta 100405 — Enumeración de infraestructura:**

```json
{
  "full_log": "20260928 22:19:28,4ca86714d046,labadmin,172.18.0.2,12,18,QUERY,,'SHOW DATABASES',0"
}
```

**Alerta 100406 — Exfiltración de credenciales (CRÍTICA):**

```json
{
  "full_log": "20260928 22:26:28,4ca86714d046,labadmin,172.18.0.2,15,30,QUERY,corporate_assets,'SELECT * FROM credentials',0"
}
```
-----

> [!WARNING] Alerta 100406 — Nivel 12
> La ejecución de `SELECT * FROM credentials` sobre `corporate_assets`
> confirma que el atacante accedió a credenciales de múltiples sistemas
> de la infraestructura. Escalar a L2 inmediatamente y activar
> protocolo de rotación de credenciales.

---

## Paso 2 — Análisis L2

### Análisis de logs en el servidor

```bash
# Ver el log de auditoría de MariaDB
sudo tail -n 100 /var/lib/docker/volumes/corp_assetdata/_data/server_audit.log

# Buscar accesos desde el contenedor de la API
grep "172.18.0.2" /var/lib/docker/volumes/corp_assetdata/_data/server_audit.log

# Buscar accesos a la tabla credentials
grep "credentials" /var/lib/docker/volumes/corp_assetdata/_data/server_audit.log

# Verificar usuarios activos en MariaDB
docker exec corp-db-assets mariadb \
    -ulabadmin -p'password123' \
    -e "SELECT user, host, authentication_string FROM mysql.user;"

# Verificar conexiones activas
docker exec corp-db-assets mariadb \
    -ulabadmin -p'password123' \
    -e "SHOW PROCESSLIST;"
```

### Verificar qué datos fueron expuestos

```bash
# Ver contenido de la tabla credentials
docker exec corp-db-assets mariadb \
    -ulabadmin -p'password123' \
    -e "USE corporate_assets; SELECT * FROM credentials;"

# Ver contenido de la tabla endpoints
docker exec corp-db-assets mariadb \
    -ulabadmin -p'password123' \
    -e "USE corporate_assets; SELECT * FROM endpoints;"
```

### Reglas Wazuh involucradas

```xml
<rule id="100402" level="8">
  <field name="mariadb.event_type">^QUERY$</field>
  <field name="mariadb.query">^SELECT user FROM mysql\.user$</field>
  <description>MariaDB - Enumeración de usuarios del sistema</description>
  <mitre><id>T1087</id></mitre>
</rule>

<rule id="100403" level="8">
  <field name="mariadb.event_type">^QUERY$</field>
  <field name="mariadb.query">^SELECT 1$</field>
  <description>MariaDB - Acceso mediante credenciales válidas detectado</description>
  <mitre><id>T1078</id></mitre>
</rule>

<rule id="100404" level="8">
  <field name="mariadb.event_type">^QUERY$</field>
  <field name="mariadb.query">^SELECT CURRENT_USER\(\)$</field>
  <description>MariaDB - Identificación del usuario actual detectada</description>
  <mitre><id>T1033</id></mitre>
</rule>

<rule id="100405" level="8">
  <field name="mariadb.event_type">^QUERY$</field>
  <field name="mariadb.query" type="pcre2">
    ^(SHOW DATABASES|SHOW TABLES|SELECT \* FROM endpoints)$
  </field>
  <description>MariaDB - Enumeración de estructura e infraestructura detectada</description>
  <mitre><id>T1213</id></mitre>
</rule>

<rule id="100406" level="12">
  <field name="mariadb.event_type">^QUERY$</field>
  <field name="mariadb.query" type="pcre2">^SELECT \* FROM credentials$</field>
  <description>MariaDB - Acceso a información de credenciales detectada</description>
  <mitre><id>T1213</id></mitre>
</rule>
```

---

## Paso 3 — Contención

### Paso 3 — Contención

### Paso 3 — Contención

#### Si solo hay enumeración de usuarios (100402):
- [ ] Bloquear IP atacante en firewall
- [ ] Revocar acceso anónimo a MariaDB
- [ ] **P2** — Escalar a L2 con ticket

#### Si hay acceso con credenciales válidas (100403/100404):
- [ ] Bloquear IP atacante inmediatamente
- [ ] Revocar credenciales comprometidas (TCFD, labadmin)
- [ ] Aislar el contenedor de MariaDB de la red Docker
- [ ] **P1** — Escalar a L2 con ticket urgente

#### Si se confirmó acceso a tabla credentials (100406):
- [ ] Aislar contenedor MariaDB inmediatamente
- [ ] Rotar TODAS las credenciales expuestas
- [ ] Preservar logs para análisis forense
- [ ] Notificar al equipo de seguridad
- [ ] **P1** — Activar PB-03 si hay evidencia de movimiento lateral

----

```bash
# Revocar acceso anónimo a MariaDB
docker exec corp-db-assets mariadb \
    -ulabadmin -p'password123' \
    -e "DELETE FROM mysql.user WHERE User=''; FLUSH PRIVILEGES;"

# Revocar usuario TCFD
docker exec corp-db-assets mariadb \
    -ulabadmin -p'password123' \
    -e "DROP USER 'TCFD'@'%'; FLUSH PRIVILEGES;"

# Aislar el contenedor MariaDB de la red Docker
docker network disconnect <network_name> corp-db-assets

# Detener el contenedor MariaDB
docker stop corp-db-assets
```

---

## Criterios de escalada

| Condición | Prioridad | Acción |
|---|---|---|
| Solo alerta 100402, host autorizado | P3 | Monitorear, verificar con equipo |
| Alerta 100402 desde 172.18.0.2 | P2 | Escalar a L2, ticket urgente |
| Alertas 100403/100404/100405 | P2 | Escalar a L2, aislar contenedor |
| Alerta 100406 disparada | P1 | Escalar inmediatamente, rotar credenciales, activar PB-03 |

---

## Cobertura MITRE ATT&CK

| Técnica | ID | Táctica | Descripción |
|---|---|---|---|
| Account Discovery | T1087 | Discovery | `SELECT user FROM mysql.user` |
| Valid Accounts | T1078 | Initial Access / Persistence | Acceso con TCFD y labadmin |
| System Owner/User Discovery | T1033 | Discovery | `SELECT CURRENT_USER()` |
| Data from Information Repositories | T1213 | Collection | `SELECT * FROM credentials` |

---

## Recomendaciones de remediación

```sql
-- Eliminar acceso anónimo
DELETE FROM mysql.user WHERE User='';
FLUSH PRIVILEGES;

-- Crear usuario con permisos mínimos para la API
CREATE USER 'api_readonly'@'172.18.0.2' IDENTIFIED BY 'password_seguro';
GRANT SELECT ON corporate_assets.endpoints TO 'api_readonly'@'172.18.0.2';
FLUSH PRIVILEGES;

-- Encriptar tabla de credenciales
ALTER TABLE credentials MODIFY password VARBINARY(255);
```

**Acciones adicionales:**
- Eliminar cuentas sin contraseña (`root` sin auth, usuario anónimo)
- Restringir acceso a MariaDB solo desde IPs autorizadas
- Encriptar datos sensibles en la tabla `credentials`
- Habilitar TLS en la conexión API → MariaDB
- Implementar rotación automática de credenciales

---

## Herramientas

| Herramienta | Propósito |
|---|---|
| Wazuh Dashboard — Discover | Análisis de alertas y correlación |
| Wazuh Threat Hunting | Revisión de actividad por agente |
| MariaDB Audit Plugin | Fuente de logs de auditoría |
| Docker CLI | Inspección y contención de contenedores |
| John the Ripper | Cracking offline de hashes |
| Jira — IRSOC-4 | Ticket de referencia |

→ Playbook anterior: **PB-01 — Command Injection sobre API vulnerable**
→ Playbook siguiente: **PB-03 — Fuerza Bruta RDP con Active Response**
