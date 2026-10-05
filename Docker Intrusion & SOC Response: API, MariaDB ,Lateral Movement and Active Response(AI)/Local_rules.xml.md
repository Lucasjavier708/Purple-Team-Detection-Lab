
# Wazuh — Local Rules

## Infrastructure API

```xml
<group name="infrastructure_api,">

  <rule id="100311" level="8">
    <decoded_as>json</decoded_as>
    <field name="parameters.hostname" type="pcre2">ip addr|ip route|/proc/net/route|/proc/net/fib_trie</field>
    <description>Infrastructure API - Network Discovery detectado</description>
    <mitre>
      <id>T1016</id>
    </mitre>
    <group>api_security,network_discovery,</group>
  </rule>

  <rule id="100310" level="10">
    <decoded_as>json</decoded_as>
    <field name="parameters.hostname" type="pcre2">(?i)^(?!.*(?:ip addr|ip route|/proc/net/route|/proc/net/fib_trie|ping -c 1 172\.18\.0\.[0-9]+))(?:(?:.*)(?:;|&&|\|\||`|\$\()).*$</field>
    <description>Command Injection - Ejecución de comandos detectada</description>
    <mitre>
      <id>T1059.004</id>
    </mitre>
    <group>api_security,command_injection,</group>
  </rule>

  <rule id="100312" level="8">
    <decoded_as>json</decoded_as>
    <field name="parameters.hostname" type="pcre2">(?i)ping -c 1 172\.18\.0\.[0-9]+</field>
    <description>Infrastructure API - Host Discovery detectado</description>
    <mitre>
      <id>T1018</id>
    </mitre>
    <group>api_security,host_discovery,</group>
  </rule>

</group>
```

### MariaDB — Reglas específicas

```xml
<group name="mariadb_specific,">

  <rule id="100402" level="8">
    <if_sid>100301</if_sid>
    <field name="mariadb.event_type">^QUERY$</field>
    <field name="mariadb.query">^SELECT user FROM mysql\.user$</field>
    <description>MariaDB - Enumeración de usuarios del sistema</description>
    <group>mariadb,sql_query,account_discovery,</group>
    <mitre>
      <id>T1087</id>
    </mitre>
  </rule>

  <rule id="100403" level="8">
    <field name="mariadb.event_type">^QUERY$</field>
    <field name="mariadb.query">^SELECT 1$</field>
    <description>MariaDB - Acceso mediante credenciales válidas detectado</description>
    <group>mariadb,sql_query,query_test,</group>
    <mitre>
      <id>T1078</id>
    </mitre>
  </rule>

  <rule id="100404" level="8">
    <field name="mariadb.event_type">^QUERY$</field>
    <field name="mariadb.query">^SELECT CURRENT_USER\(\)$</field>
    <description>MariaDB - Identificación del usuario actual detectada</description>
    <group>mariadb,sql_query,user_discovery,</group>
    <mitre>
      <id>T1033</id>
    </mitre>
  </rule>

  <rule id="100405" level="8">
    <field name="mariadb.event_type">^QUERY$</field>
    <field name="mariadb.query" type="pcre2">^(SHOW DATABASES|SHOW TABLES|SELECT \* FROM endpoints)$</field>
    <description>MariaDB - Enumeración de estructura e información de infraestructura detectada</description>
    <group>mariadb,sql_query,enumeration,</group>
    <mitre>
      <id>T1213</id>
    </mitre>
  </rule>

  <rule id="100406" level="12">
    <field name="mariadb.event_type">^QUERY$</field>
    <field name="mariadb.query" type="pcre2">^SELECT \* FROM credentials$</field>
    <description>MariaDB - Acceso a información de credenciales detectado</description>
    <group>mariadb,sql_query,credential_access,exfiltration,</group>
    <mitre>
      <id>T1213</id>
    </mitre>
  </rule>

</group>
```

## Windows RDP — Fase 3

```xml
<group name="RDP_Rules,">

  <rule id="100503" level="12">
    <if_sid>60204</if_sid>
    <field name="win.eventdata.targetUserName">^Administrador$</field>
    <description>Windows RDP - Fuerza bruta detectada - Bloqueo de IP</description>
    <mitre>
      <id>T1110</id>
      <id>T1110.001</id>
    </mitre>
    <group>windows,rdp,brute_force,authentication_failed,</group>
  </rule>

</group>
```
