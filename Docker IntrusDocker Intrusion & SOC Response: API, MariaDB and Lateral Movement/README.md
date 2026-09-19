
<div align="center">

# 🛡️ Docker Intrusion & SOC Response: API, MariaDB and Lateral Movement

**`Wazuh`** `·` **`Kali Linux`** `·` **`Docker`** `·` **`Mittre At&ck`**

---

🔴 **Ataque** &nbsp;→&nbsp; 🔵 **Detección** &nbsp;→&nbsp; 🟢 **Respuesta**

</div>



## 📌 Objetivo

El objetivo de este caso es analizar y responder ante una intrusión de múltiples etapas dentro de una infraestructura basada en Docker, desde la perspectiva de Ataque-Detecion-Respuesta

El escenario parte de una API vulnerable desplegada sobre Docker en un servidor Ubuntu, utilizada como punto de acceso inicial. A partir de este acceso, la actividad progresa hacia la exploración del entorno Docker y el acceso a un contenedor MariaDB, donde se encuentra información relacionada con activos, usuarios y credenciales.

La obtención de credenciales permite continuar la cadena de ataque mediante técnicas de Credential Access, recuperación de credenciales y un posterior intento de Lateral Movement hacia otros sistemas de la infraestructura.

Durante el desarrollo del incidente, el SOC utiliza Wazuh para monitorear la actividad, generar alertas mediante reglas personalizadas, utilizar playbooks para responder ante incidentes, investigar los eventos y relacionar las distintas acciones con técnicas de MITRE ATT&CK.

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
