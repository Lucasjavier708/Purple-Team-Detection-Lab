
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

**Alerta 100402 — Enumeración de usuarios:**

```json
{
  "full_log": "20260928 22:13:03,4ca86714d046,root,172.18.0.2,8,10,QUERY,mysql,'SELECT user FROM mysql.user',0"
}

**Alerta 100403 — Acceso con credenciales válidas (TCFD):**

{
  "full_log": "20260928 22:14:27,4ca86714d046,TCFD,172.18.0.2,9,12,QUERY,,'SELECT 1',0"
}
