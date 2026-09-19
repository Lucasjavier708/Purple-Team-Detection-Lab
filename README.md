<img width="1509" height="680" alt="Diseño sin título" src="https://github.com/user-attachments/assets/1ca0e2cf-b544-4551-8596-dab6ae628bd4" />






#   Casos Prácticos de Ataque, Detección e Investigación
Este proyecto está orientado al desarrollo y documentación de casos prácticos que simulan escenarios reales de ataque y defensa dentro de mi HomeLab.

Cada caso de estudio comienza con la ejecución de un escenario de ataque (Red Team) y continúa con su detección, análisis, investigación y respuesta desde la perspectiva de un Centro de Operaciones de Seguridad (SOC). Todos los casos son documentados desde su planificación hasta su conclusión.

## Objetivo
Demostrar conocimientos prácticos en ciberseguridad mediante la implementación de una infraestructura de laboratorio, la utilización de herramientas , la aplicación de metodologías y procedimientos empleados en entornos profesionales.

# 📋 Casos de Estudio

Los siguientes escenarios forman parte del laboratorio y serán ampliados progresivamente con nuevos casos de ataque y defensa.

| Caso | Escenario | Principales técnicas | Estado |
|------|-----------|----------------------|--------|
| **Caso 01** | [Orchestrated Attack Framework: Multi-Stage Brute Force & SQLi with Real-Time SIEM Detection](./casos/caso-1) | SQL Injection · Brute Force · SIEM Detection | 🟢 Completado |
| **Caso 02** |  [Docker Intrusion & SOC Response: API, MariaDB and Lateral Movement](./Docker%20Intrusion%20%26%20SOC%20Response:%20API,%20MariaDB%20and%20Lateral%20Movement)  |  Docker · API · MariaDB · Lateral Movement · SOC Detectcion  | 🟡 En desarrollo |
| |  | |  |



# Arquitectura

Vista general de la infraestructura **HomeLab/SOC**, donde se desarrollan los escenarios de seguridad dentro de la red de laboratorio.

<p align="center">
  <img width="818" alt="Infraestructura general" src="https://github.com/user-attachments/assets/a944ba4b-7184-45be-88c7-63ba51881e05">
</p>

---

Vista de la infraestructura **HomeLab/SOC** con los agentes **Wazuh** desplegados en cada equipo, permitiendo el monitoreo centralizado de todos los dispositivos.

<p align="center">
  <img width="838" alt="Infraestructura con Wazuh" src="https://github.com/user-attachments/assets/fc7cd4ac-7fbf-4375-9b5a-54202f1100a4">
</p>









# Stack Stenico
 
### 🖥️ Sistemas Operativos
![Windows Server](https://img.shields.io/badge/Windows_Server-2019-0078D4?style=flat-square&logo=windows)
![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?style=flat-square&logo=ubuntu)
![Kali Linux](https://img.shields.io/badge/Kali_Linux-2023-557C94?style=flat-square&logo=kalilinux)
![Windows 11](https://img.shields.io/badge/Windows-11-0078D4?style=flat-square&logo=windows)

### 🛡️ SIEM & Monitoreo
![Wazuh](https://img.shields.io/badge/Wazuh-Manager-005EB8?style=flat-square&logo=wazuh)
![Sysmon](https://img.shields.io/badge/Sysmon-Monitoring-00A4EF?style=flat-square)

### 🌐 Redes
![TCP/IP](https://img.shields.io/badge/TCP%2FIP-Networking-FF6B6B?style=flat-square)
![DNS](https://img.shields.io/badge/DNS-Resolution-4285F4?style=flat-square)
![DHCP](https://img.shields.io/badge/DHCP-Protocol-00AA00?style=flat-square)
![VPN](https://img.shields.io/badge/VPN-Secure-FF00FF?style=flat-square)
![Firewall](https://img.shields.io/badge/Firewall-Protection-FFA500?style=flat-square)
![IDS/IPS](https://img.shields.io/badge/IDS%2FIPS-Detection-DC143C?style=flat-square)

### 🔍 Análisis de Seguridad 
![Nmap](https://img.shields.io/badge/Nmap-Scanning-0D5B05?style=flat-square)
![VirusTotal](https://img.shields.io/badge/VirusTotal-Analysis-390EFF?style=flat-square)

### ⚔️ Pentesting
![Metasploit](https://img.shields.io/badge/Metasploit-Framework-2D2D2D?style=flat-square&logo=metasploit)
![Hydra](https://img.shields.io/badge/Hydra-Brute_Force-FF6B6B?style=flat-square)
![Burp Suite](https://img.shields.io/badge/Burp_Suite-Testing-FF6633?style=flat-square)
![John](https://img.shields.io/badge/John%20the%20Ripper-Credential%20Cracking-FFD700?style=flat-square&logoColor=black) 
![Hashcat](https://img.shields.io/badge/Hashcat-Credential%20Cracking-FFD700?style=flat-square&logoColor=black)

### 💻 Desarrollo
![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python)
![Bash](https://img.shields.io/badge/Bash-5.0-4EAA25?style=flat-square&logo=gnubash)
![PowerShell](https://img.shields.io/badge/PowerShell-7.0-5391FE?style=flat-square&logo=powershell)
![Node.js](https://img.shields.io/badge/Node.js-18.x-339933?style=flat-square&logo=node.js)


### ☁️ Infraestructura
![Docker](https://img.shields.io/badge/Docker-Latest-2496ED?style=flat-square&logo=docker)
![Kubernetes](https://img.shields.io/badge/Kubernetes-K8s-326CE5?style=flat-square&logo=kubernetes)
![VirtualBox](https://img.shields.io/badge/VirtualBox-Hypervisor-183A61?style=flat-square&logo=virtualbox)

### 📚 Frameworks & Buenas Prácticas
![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-Framework-EE0000?style=flat-square)
![ISO 27001](https://img.shields.io/badge/ISO_27001-Compliance-0066CC?style=flat-square)





