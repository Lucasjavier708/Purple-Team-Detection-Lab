

<div align="center">

# 🛡️ Orchestrated Attack Framework

### Multi-Stage Brute Force & SQLi with Real-Time SIEM Detection

**`Wazuh`** `·` **`Kali Linux`** `·` **`Hydra`** `·` **`Burp Suite`**

---

🔴 **Ataque** &nbsp;→&nbsp; 🔵 **Detección** &nbsp;→&nbsp; 🟢 **Respuesta**

</div>

## 📌 Objetivo 

Armar un escenario de ataque controlado contra una app vulnerable a propósito (Bank App) para ver cómo Wazuh detecta, registra y alertá cada movimiento. Me pongo en el lugar del SOC y trabajo el ciclo completo: desde que se genera el evento hasta que lo escalo.

Para eso armé tres scripts: uno de fuerza bruta, uno de SQL injection, y uno que orquesta los dos anteriores simulando un atacante que cambia de técnica si la primera falla. Cada uno genera su propio escenario, con su triage, su playbook y su ticket de escalación.

## Arquitectura

El lab lo armé con dos redes separadas, simulando un esquema real de SOC.

En la red de laboratorio pasan los ataques: Kali Linux ataca directo al Windows Server, donde corre la Bank App (Node.js + SQL Server). El router de esa red tiene el DHCP apagado, así que el mismo Windows Server actúa como servidor DHCP del laboratorio.

Del otro lado está la red de estudio, donde vive Wazuh y llegan las alertas. Configuré port forwarding en el router principal hacia el router de estudio (Telefónica) para que los agentes se comuniquen entre ambas redes sin problema.

En resumen: una red ataca, la otra detecta y centraliza. Entre las dos arman el SOC del lab.



<img width="808" height="608" alt="Caso Diagrama" src="https://github.com/user-attachments/assets/a322f426-e69c-405c-aad4-f4a41cf4d31a" />

<img width="828" height="622" alt="Diagrama sin título" src="https://github.com/user-attachments/assets/78d88b79-ce65-4dcd-ac8c-d17b6c527b0f" />

## Herramientas 

![Kali Linux](https://img.shields.io/badge/Kali%20Linux-Penetration%20Testing-557C94?style=flat-square&logo=linux&logoColor=white)
![Hydra](https://img.shields.io/badge/Hydra-Brute%20Force-FF6B6B?style=flat-square&logo=python&logoColor=white)
![Burpsuit](https://img.shields.io/badge/Burpsuit-Web%20Proxy-FF9900?style=flat-square&logo=firefox&logoColor=white)
![Nmap](https://img.shields.io/badge/Nmap-Port%20Scanner-4169E1?style=flat-square&logo=network&logoColor=white)


![Windows Server](https://img.shields.io/badge/Windows%20Server-2019-0078D4?style=flat-square&logo=windows&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-Bank%20App-339933?style=flat-square&logo=node.js&logoColor=white)
![SQL Server](https://img.shields.io/badge/SQL%20Server-2019-CC2927?style=flat-square&logo=microsoft-sql-server&logoColor=white)


![Wazuh](https://img.shields.io/badge/Wazuh-4.x%20SIEM-0078D4?style=flat-square&logo=security&logoColor=white)


----

 ## Paso a Paso 



APP

Este es el servidor objetivo. La Bank App corre sin protecciones (vulnerable intencionalmente) para que podamos simular ataques reales: brute force contra el endpoint /api/login y SQL injection en los campos de usuario y contraseña.

<img width="1024" height="768" alt="Captura de pantalla (1)" src="https://github.com/user-attachments/assets/04153a85-29da-4ff3-a77b-666835ba8450" />

<img width="1024" height="768" alt="Captura de pantalla (5)" src="https://github.com/user-attachments/assets/8d4ad243-ec3e-47e6-94d5-fbc6a9ebfe26" />


-------
Nmap

Antes de atacar, necesitamos saber qué puertos están abiertos en el Windows Server y qué servicios corren. Este escaneo es la fase de reconocimiento (footprinting) donde identificamos la superficie de ataque disponible.

<img width="1024" height="768" alt="nmap1" src="https://github.com/user-attachments/assets/656bac1d-1819-4b5d-bd79-1a806503f6ec" />


------

BurpSuit

Configuramos Burpsuit en modo proxy listening en 127.0.0.1:8080, activamos Intercept, y capturamos las peticiones legítimas a la Bank App.Al activar el módulo Intercept, capturamos cada petición POST al endpoint /api/login, permitiéndonos examinar los parámetros de usuario y contraseña, los headers HTTP, y las respuestas del servidor. Este análisis manual es crucial antes de automatizar ataques, ya que identificamos patrones de error, tiempos de respuesta, y validaciones que luego configuraremos en los scripts de Hydra y SQLi.



<img width="1024" height="768" alt="Screenshot_2026-07-21_10_06_11" src="https://github.com/user-attachments/assets/47f00d77-82c5-4a69-9136-e85a1bb110f1" />

<img width="1024" height="768" alt="Screenshot_2026-07-21_10_06_57" src="https://github.com/user-attachments/assets/55021553-bda4-400b-a6fe-3544b3ad55ed" />

<img width="1024" height="768" alt="Screenshot_2026-07-21_10_07_07" src="https://github.com/user-attachments/assets/7e9f346d-24ab-45fe-b4ca-0185e1b9a032" />

<img width="1024" height="768" alt="Screenshot_2026-07-21_10_07_32" src="https://github.com/user-attachments/assets/13d3cb61-3594-47ed-9574-4082aaca2d7b" />



-----


scripts automatizados

En /redlab-attacks se encuentran los tres pilares de la automatización: orchestrator.sh que coordina todo el ataque, attack_brute_force.sh que ejecuta Hydra contra /api/login, y attack_sqli.sh que prueba inyecciones SQL. Los diccionarios y wordlists (midiccionaries.txt) alimentan los ataques, mientras que los archivos de output registran cada intento exitoso y fallido.

<img width="1024" height="768" alt="redlab" src="https://github.com/user-attachments/assets/b9d2a711-bb33-47e2-aecc-54c0ae003d6b" />


**El orchestrator.sh** es el script maestro que coordina todos los ataques. Al ejecutarse, inicia automáticamente el ataque de brute force contra /api/login usando Hydra, espera a que termine, luego lanza el ataque de SQL injection. Cada fase se marca con [+] al comenzar y [✓] al completarse. Al finalizar ambos ataques, genera un resumen de sesión con todos los eventos registrados.

<img width="1024" height="768" alt="Orquestador 1" src="https://github.com/user-attachments/assets/54dc44d1-d99c-4271-b6dd-0f458fcc0c85" />

<img width="1024" height="768" alt="Orquestador 2" src="https://github.com/user-attachments/assets/4b09f48c-878d-4534-a35c-03db4dc980c1" />


**Attack_brute_force***.sh automatiza el ataque que hicimos manualmente en Burpsuit. Hydra toma la estructura POST que vimos (/api/login:username=USER&password=PASS) y la replica miles de veces con diferentes contraseñas del diccionario. El comando hydra -l admin -P midiccionary.txt -s 3000 192.168.3.10 http-post-form "/api/login:username=USER&password=PASS:i:F=Usuario o contraseña inválido" automatiza exactamente lo que interceptamos en Burpsuit: envíos POST al /api/login con parámetro username y password. La cadena i:F=Usuario o contraseña inválido es la respuesta de error que vimos en Burpsuit, permitiendo a Hydra diferenciar intentos fallidos de exitosos. Cuando Hydra encuentra una contraseña válida, Wazuh registra ese acceso exitoso como evento de seguridad.

<img width="1020" height="710" alt="fuezabruta" src="https://github.com/user-attachments/assets/e7e93869-3fc5-4ff0-a6ae-d6a70b311bde" />

<img width="1007" height="592" alt="Captura de pantalla_2026-07-21_11-07-22" src="https://github.com/user-attachments/assets/75bef765-d04c-4e82-b604-1ce547c9abca" />

-----


## Hydra Code- 

hydra -l admin -P midiccionary.txt -s 3000 192.168.3.10 http-post-form "/api/login:username=USER&password=PASS:i:F=Usuario o contraseña inválido"

| Parámetro | Descripción |
|-----------|-------------|
| `hydra` | Herramienta de fuerza bruta para ataques de login |
| `-l admin` | Define usuario específico a atacar (login fijo) |
| `-P midiccionary.txt` | Carga diccionario de contraseñas a probar |
| `-s 3000` | Puerto específico donde corre la Bank App |
| `192.168.3.10` | IP del Windows Server objetivo |
| `http-post-form` | Módulo de ataque vía formulario POST |
| `/api/login` | Endpoint/ruta del servidor a atacar |
| `username=USER&password=PASS` | Estructura POST (USER y PASS se reemplazan en cada intento) |
| `i:F=` | Flag: `i` (case-insensitive) + `F` (fail string) |
| `Usuario o contraseña inválido` | Mensaje de error exacto que detecta intentos fallidos |




**Attack_sqli.sh** automatiza inyecciones SQL contra /api/login. Envía payloads maliciosos como admin' or '1'='1 -- en los campos de usuario y contraseña para bypassear la validación. El servidor intenta ejecutar estas consultas y responde con HTTP 200 si la inyección funciona. Wazuh captura cada intento como evento de ataque SQLi.


0<img width="1024" height="768" alt="Sqli" src="https://github.com/user-attachments/assets/f0e1400a-fe3a-4ef7-a430-3be0a289f059" />


------------------------------------- 

<div align="center">

## 🔴 Ataque — 🔵 Detección — 🟢 Respuesta

</div>

<img width="2554" height="974" alt="Captura de pantalla 2026-07-20 210513" src="https://github.com/user-attachments/assets/ea22a075-1d0d-4de7-a5a3-d6914dcb3958" />


 ## 🔵 **Monitoreo**




<img width="2544" height="907" alt="Captura de pantalla 2026-07-21 172021" src="https://github.com/user-attachments/assets/9c5bd579-6e05-403a-9d2e-854f082f3c38" />



### ORQUESTADOR 

Acá el orchestrator.sh trabaja los dos ataques en cadena: arranca con fuerza bruta contra admin, y si no entra por ahí, pasa solo a probar SQLi. La idea era ver si Wazuh me detectaba el cambio de técnica dentro de la misma sesión, o si me lo tiraba como eventos sueltos sin relacionarlos.

<img width="2557" height="903" alt="Evento Orq" src="https://github.com/user-attachments/assets/bf87d84b-5f03-4d5a-af9a-5b3bd7b379a4" />

<img width="2557" height="902" alt="LOG 1 Orquestador" src="https://github.com/user-attachments/assets/69ef60d6-1085-4c3a-b8cb-984deda89fc8" />

## Triage — Escenario: Orquestador (Brute Force + SQL Injection combinados)

**Fuente:** `login_attempts.log` (Bank App) vía agente Wazuh — `Wazuh-windoServ-PC2-Lab`

| # | Campo | Dato obtenido |
|---|---|---|
| 1 | Fecha y hora del evento (timestamp) | 21/07/2026 05:42:29.858Z |
| 2 | Tipo de alerta (rule.description) | Bank App: Successful login detected |
| 3 | Regla Wazuh (rule.id) | 100405 |
| 4 | Nivel de alerta (rule.level) | 6 |
| 5 | Veces que disparó la regla (rule.firedtimes) | 5 |
| 6 | IP de origen | 192.168.3.163 |
| 7 | Equipo objetivo (agent.name) | Wazuh-windoServ-PC2-Lab |
| 8 | IP del equipo objetivo (agent.ip) | 192.168.3.10 |
| 9 | Usuario afectado | admin' or 1=1 -- |
| 10 | Código HTTP | 200 |

**Prioridad asignada:** Alta → pasa a Validación.

## Validación - Veredicto: Verdadero Positivo — Escalar

El HTTP 200 no es un login real, el campo usuario tiene el payload Admin´ or 1=1 -- , tipico de bypass por SQLi, no una creddencial legitima.Lo que llama la atencion es el timing: apenas terminan los intentos fallidos de fuerza bruta contra "admin", arranca las inyecciones SQLi casi de inmediato, sin pausa.Eso no es casualidad, es un orquestador que al no lograr entrar por fuerza bruta, cambio de tactica automaticamente y ademas no menos importante la regla haya disparado 5 veces tambien suma, el origen 192.168.3.163 no pertencen a nigun segmento valido de la red 

## IOCs 
Reputación externa (AbuseIPDB / VirusTotal): No aplica — 192.168.3.163 es una IP privada (RFC1918), sin visibilidad pública.

  - Contexto interno:
  192.168.3.163 corresponde a la maquina atacante (kali linux) dentro de la topologia de la red
   - contexto activo del objetivo:
  192.168.3.10:3000 es el servidor que corre la Bank APP , activo de criticidad alta dentro del ejercicio, por que simula
  una aplicacion financiera con datos sensibles

  ## Ticket de Escalación
```
┌──────────────────────────────────────────────────────────────┐
│                        🎫 TICKET DE ESCALACIÓN                │
├──────────────────────────────────────────────────────────────┤
│ ID: INC-2026-0721-001                                         │
│ Escenario: Orquestador (Fuerza Bruta + SQL Injection)         │
│                                                                │
│ Severidad: 🔴 Alta                                            │
│ Estado: 🟡 Escalado a L2                                      │
│ Categoría: Intrusión confirmada / Explotación app web         │
├──────────────────────────────────────────────────────────────┤
│ Resumen                                                       │
│ Acceso exitoso (HTTP 200) en /api/login tras fallo de         │
│ fuerza bruta contra "admin". El acceso se logró con un        │
│ payload SQLi, no una credencial real. El cambio de vector     │
│ inmediato confirma orquestación automatizada.                 │
├──────────────────────────────────────────────────────────────┤
│ Datos del Triage                                              │
│ ├── Fecha/hora: 21/07/2026 05:42:29.858Z                      │
│ ├── Regla Wazuh: 100405                                       │
│ ├── Nivel de alerta: 6                                        │
│ ├── Veces disparada: 5                                        │
│ ├── IP origen: 192.168.3.163                                  │
│ ├── Equipo objetivo: Wazuh-windoServ-PC2-Lab                  │
│ ├── IP equipo objetivo: 192.168.3.10                          │
│ ├── Usuario afectado: admin' or 1=1 --                        │
│ └── Código HTTP: 200                                          │
├──────────────────────────────────────────────────────────────┤
│ Validación                          VERDADERO POSITIVO        │
│ El "usuario" es un payload SQLi, no una credencial válida.    │
│ El timing entre fin de fuerza bruta e inicio de SQLi          │
│ descarta evento aislado. Origen no autorizado, sin ventana    │
│ de mantenimiento que lo justifique.                           │
├──────────────────────────────────────────────────────────────┤
│ Enriquecimiento                                               │
│ ├── Reputación externa: No aplica (IP privada / RFC1918)      │
│ ├── Origen: Máquina atacante, Lab Network                     │
│ ├── Activo afectado: 192.168.3.10 — Bank App (crítico)        │
│ └── Pendiente: revisar actividad IP fuera de 05:42–05:46Z     │
├──────────────────────────────────────────────────────────────┤
│ Playbook aplicado                                             │
│ → Playbook de Respuesta — Brute Force / SQL Injection         │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ Analista: SOC L1 — Lucas                                      │
│ Escalado a: SOC L2                                            │
│ Fecha: 21/07/2026                                              │
└──────────────────────────────────────────────────────────────┘
```

📋 **Playbook** [Brute Force / SQL Injection](https://github.com/Lucasjavier708/Purple-Team-Case-Studies--HOMELAB-SOC./blob/main/casos/caso-1/playbook-orchestrated-attack.md)



------
### FUERZA BRUTA 

El attack_brute_force.sh le pegó a /api/login con Hydra, tirando el diccionario de contraseñas contra el usuario admin. Quería ver si Wazuh diferenciaba este patrón (fallos a repetición + éxito con credencial limpia) del de SQLi.

<img width="2552" height="902" alt="Fuerza bruta evento" src="https://github.com/user-attachments/assets/060bb68d-780b-464b-9e66-8e2f666f99ed" />

<img width="2557" height="903" alt="Captura de pantalla 2026-07-21 111627" src="https://github.com/user-attachments/assets/5b3f00e0-279a-4e66-a8e1-598a9b4c9aab" />


## Triage — Escenario: Fuerza Bruta 


| # | Campo | Dato obtenido |
|---|---|---|
| 1 | Fecha y hora del evento (timestamp) | 2026-07-21 05:44:36.967Z|
| 2 | Tipo de alerta (rule.description) | Bank App: Successful login detected |
| 3 | Regla Wazuh (rule.id) | 100405 |
| 4 | Nivel de alerta (rule.level) | 6 |
| 5 | Veces que disparó la regla (rule.firedtimes) | 6 |
| 6 | IP de origen | 192.168.3.163 |
| 7 | Equipo objetivo (agent.name) | Wazuh-windoServ-PC2-Lab |
| 8 | IP del equipo objetivo (agent.ip) | 192.168.3.10 |
| 9 | Usuario afectado | username=admin  |
| 10 | Código HTTP | 200 |

**Prioridad asignada:** Alta → pasa a Validación.

## Validación - Veredicto: Verdadero Positivo — Escalar

Después de varios intentos fallidos (401) contra el usuario "admin", se registró un acceso exitoso (200) — la regla 100405 ya lleva 6 disparos, lo que confirma que no es un evento aislado. Que una fuerza bruta termine en acceso  sobre una cuenta con privilegios es justo el escenario que se quiere encontrar , así que se define como verdadero positivo y se escala para revisión.

## IOCs 
Reputación externa (AbuseIPDB / VirusTotal): No aplica — 192.168.3.163 es una IP privada (RFC1918), sin visibilidad pública.

  - Contexto interno:
  192.168.3.163 corresponde a la maquina atacante (kali linux) dentro de la topologia de la red
   - contexto activo del objetivo:
  192.168.3.10:3000 es el servidor que corre la Bank APP , activo de criticidad alta dentro del ejercicio, por que simula
  una aplicacion financiera con datos sensibles

 ## Ticket de Escalación
```
┌──────────────────────────────────────────────────────────────┐
│                        🎫 TICKET DE ESCALACIÓN                │
├──────────────────────────────────────────────────────────────┤
│ ID: INC-2026-0721-002                                         │
│ Escenario: Fuerza Bruta                                       │
│                                                                │
│ Severidad: 🔴 Alta                                            │
│ Estado: 🟡 Escalado a L2                                      │
│ Categoría: Intrusión confirmada / Compromiso de credencial    │
├──────────────────────────────────────────────────────────────┤
│ Resumen                                                       │
│ Múltiples intentos fallidos (401) contra la cuenta "admin"    │
│ culminaron en un acceso exitoso (200) en /api/login. La       │
│ credencial fue obtenida por fuerza bruta, sin uso de payload  │
│ SQLi — acceso legítimo del sistema pero de origen malicioso.  │
├──────────────────────────────────────────────────────────────┤
│ Datos del Triage                                              │
│ ├── Fecha/hora (full_log): 21/07/2026 05:44:36.967Z           │
│ ├── Regla Wazuh: 100405                                       │
│ ├── Nivel de alerta: 6                                        │
│ ├── Veces disparada: 6                                        │
│ ├── IP origen: 192.168.3.163                                  │
│ ├── Equipo objetivo: Wazuh-windoServ-PC2-Lab                  │
│ ├── IP equipo objetivo: 192.168.3.10                          │
│ ├── Usuario afectado: admin                                    │
│ └── Código HTTP: 200                                          │
├──────────────────────────────────────────────────────────────┤
│ Validación                          VERDADERO POSITIVO        │
│ Intentos fallidos repetidos (401) precedieron el acceso       │
│ exitoso (200) sobre la misma cuenta. 6 disparos de la regla   │
│ descartan evento aislado — patrón consistente con fuerza      │
│ bruta automatizada que logró comprometer la credencial.       │
├──────────────────────────────────────────────────────────────┤
│ Enriquecimiento                                               │
│ ├── Reputación externa: No aplica (IP privada / RFC1918)      │
│ ├── Origen: Máquina atacante (Kali Linux), Lab Network         │
│ ├── Activo afectado: 192.168.3.10 — Bank App (crítico)        │
│ └── Pendiente: confirmar si "admin" es cuenta con privilegios │
│     elevados dentro de la app                                 │
├──────────────────────────────────────────────────────────────┤
│ Playbook aplicado                                             │
│ → Playbook de Respuesta — Brute Force                         │
│                                                               │
├──────────────────────────────────────────────────────────────┤
│ Analista: SOC L1 — Lucas                                      │
│ Escalado a: SOC L2                                            │
│ Fecha: 21/07/2026                                              │
└──────────────────────────────────────────────────────────────┘

```

📋 **Playbook:** [Brute Force](https://github.com/Lucasjavier708/Purple-Team-Case-Studies--HOMELAB-SOC./blob/main/casos/caso-1/Playbook-brute-force.md)

------
### SQL INJECTION 

El attack_sqli.sh probó varias variantes de payload contra /api/login para saltarse el login sin necesitar una credencial real. Lo que buscaba acá era confirmar si Wazuh me detectaba el bypass como acceso exitoso y si quedaba el payload registrado en el log.

<img width="2557" height="897" alt="evento sqli" src="https://github.com/user-attachments/assets/7873b163-dc9e-4d54-bea1-508a56a3b84e" />

<img width="2554" height="906" alt="Captura de pantalla 2026-07-21 121441" src="https://github.com/user-attachments/assets/df1514f2-59ba-4e25-9cb3-4e8bcf876fd8" />

## Triage — Escenario: SQLinjection

| # | Campo | Dato obtenido |
|---|---|---|
| 1 | Fecha y hora del evento (timestamp) | 2026-07-21 05:45:58.890Z|
| 2 | Tipo de alerta (rule.description) | Bank App: Successful login detected |
| 3 | Regla Wazuh (rule.id) | 100405 |
| 4 | Nivel de alerta (rule.level) | 6 |
| 5 | Veces que disparó la regla (rule.firedtimes) | 10 |
| 6 | IP de origen | 192.168.3.163 |
| 7 | Equipo objetivo (agent.name) | Wazuh-windoServ-PC2-Lab |
| 8 | IP del equipo objetivo (agent.ip) | 192.168.3.10 |
| 9 | Usuario afectado | username=admin or 1=1  |
| 10 | Código HTTP | 200 |


## Validación - Veredicto: Verdadero Positivo — Escalar

1=1 significa la forma en que se quiere bypasear un usuario de un formulario en este caso no es un usuario legitimo sino un intento de codigo injection , queda como verdadero positov y se escala 

## IOCs 
1=1 es la forma clásica de bypassear la validación de un formulario  — no es un usuario legítimo, es una condición que siempre se evalúa como verdadera, permitiendo entrar sin conocer una contraseña . Que la regla haya disparado 10 veces confirma que no fue un intento aislado, sino la iteración de variantes del mismo payload hasta encontrar una que funcione. Queda como verdadero positivo y se escala.

  - Contexto interno:
  192.168.3.163 corresponde a la maquina atacante (kali linux) dentro de la topologia de la red
   - contexto activo del objetivo:
  192.168.3.10:3000 es el servidor que corre la Bank APP , activo de criticidad alta dentro del ejercicio, por que simula
  una aplicacion financiera con datos sensibles

 ## Ticket de Escalación

```
┌──────────────────────────────────────────────────────────────┐
│                        🎫 TICKET DE ESCALACIÓN                │
├──────────────────────────────────────────────────────────────┤
│ ID: INC-2026-0721-003                                         │
│ Escenario: SQL Injection                                      │
│                                                                │
│ Severidad: 🔴 Alta                                            │
│ Estado: 🟡 Escalado a L2                                      │
│ Categoría: Intrusión confirmada / Explotación de vulnerabilidad│
├──────────────────────────────────────────────────────────────┤
│ Resumen                                                       │
│ Acceso exitoso (HTTP 200) en /api/login mediante bypass de    │
│ autenticación por SQL injection. El payload admin' or 1=1 --  │
│ manipula la query de validación para que siempre retorne      │
│ verdadero, sin necesidad de credencial real.                  │
├──────────────────────────────────────────────────────────────┤
│ Datos del Triage                                              │
│ ├── Fecha/hora (full_log): 21/07/2026 05:45:58.890Z           │
│ ├── Regla Wazuh: 100405                                       │
│ ├── Nivel de alerta: 6                                        │
│ ├── Veces disparada: 10                                       │
│ ├── IP origen: 192.168.3.163                                  │
│ ├── Equipo objetivo: Wazuh-windoServ-PC2-Lab                  │
│ ├── IP equipo objetivo: 192.168.3.10                          │
│ ├── Usuario afectado: admin' or 1=1 --                        │
│ └── Código HTTP: 200                                          │
├──────────────────────────────────────────────────────────────┤
│ Validación                          VERDADERO POSITIVO        │
│ El payload manipula la lógica de la query SQL para bypasear   │
│ la autenticación sin credencial real. 10 disparos de la       │
│ regla confirman iteración de variantes hasta lograr éxito,    │
│ no un evento accidental.                                      │
├──────────────────────────────────────────────────────────────┤
│ Enriquecimiento                                               │
│ ├── Reputación externa: No aplica (IP privada / RFC1918)      │
│ ├── Origen: Máquina atacante (Kali Linux), Lab Network         │
│ ├── Activo afectado: 192.168.3.10 — Bank App (crítico)        │
│ └── Pendiente: confirmar si la inyección funciona en otros    │
│     endpoints además de /api/login                            │
├──────────────────────────────────────────────────────────────┤
│ Playbook aplicado                                             │
│ → Playbook de Respuesta —  SQL Injection                      │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ Analista: SOC L1 — Lucas                                      │
│ Escalado a: SOC L2                                            │
│ Fecha: 21/07/2026                                              │
└──────────────────────────────────────────────────────────────┘

```

📋 **Playbook:** [SQLinjection](https://github.com/Lucasjavier708/Purple-Team-Case-Studies--HOMELAB-SOC./blob/main/casos/caso-1/Playbook-sql-injection.md)

-------
# Trhean Hunting 

Después de terminar el triage de los tres casos, me puse a revisar toda la ventana horaria completa para chequear si el patrón se sostenía más allá de los tres eventos que ya había escalado.

Filtré por rule.id: 2501 entre 10:10 y 10:15 UTC y en el histograma se ven dos ráfagas bien marcadas (9 intentos en 30 segundos, y después otro pico parecido), en vez de una distribución más pareja. Eso ya de entrada te dice que no es un usuario cagándose a errores con la contraseña, es un script tirando intentos.

Después fui a buscar el payload de SQLi puntual (full_log: "1' or '1'='1") y me tiró 2 resultados en esa ventana — los dos son intentos fallidos (401, regla 2501), previos a que finalmente lograra entrar. Abrí el JSON de uno para ver el detalle completo: timestamp, IP origen (192.168.3.163), el payload tal cual y rule.firedtimes: 20. Aunque sea un intento fallido, ver el payload ahí crudo confirma que venía probando esta misma firma de SQLi antes de conseguir el bypass.


<img width="2554" height="498" alt="Captura de pantalla 2026-07-21 182441" src="https://github.com/user-attachments/assets/05ce1168-87dc-4573-ac02-5d42af226773" />

<img width="2541" height="781" alt="Captura de pantalla 2026-07-21 182524" src="https://github.com/user-attachments/assets/6395df00-8422-446f-8bfb-0ffa40a1d369" />





<img width="2550" height="806" alt="payload sqli" src="https://github.com/user-attachments/assets/56815818-beff-4d70-b2f8-164590c4c9a8" />


<img width="2548" height="907" alt="payload sqli jason" src="https://github.com/user-attachments/assets/d59c41b8-826c-4805-8619-21d1008fa26b" />




---

## 4. CORRELACIÓN DE EVENTOS

Acá encontré algo más interesante: 3 accesos exitosos (200) desde la misma IP 192.168.3.163, todos en 3 minutos y medio. El primero (05:42:29Z) fue con SQLi, el segundo (05:44:36Z) con la credencial admin limpia por fuerza bruta, y el tercero (05:45:58Z) otra vez con SQLi. O sea, no fue un acceso suelto — fueron tres, con dos técnicas distintas, todo en el mismo rango de tiempo. Eso ya descarta casualidad: hubo reconocimiento previo y el tipo volvió a confirmar el acceso después.

---


## 5. Mitigación

Cada escenario tiene su playbook (PB-ORQ, PB-FB, PB-SQLI) con su propia contención y lecciones aprendidas, así que en vez de tirar una lista genérica acá, resumo lo que dice cada uno:

Contención inmediata:

Orquestador: bloqueo de IP, reseteo de la credencial admin, revisar sesiones activas.
Fuerza Bruta: bloqueo de IP + reseteo de la credencial admin.
SQLi: bloqueo de IP + revisar urgente la sanitización de inputs en /api/login.

Remediación:

Fuerza Bruta: no hay lockout de cuenta tras varios intentos fallidos — falta rate limiting (5 intentos cada 5 min) y una regla en Wazuh que lo detecte antes del acceso exitoso.
SQLi: la app no usa consultas parametrizadas — hay que meter prepared statements y reglas de WAF para patrones típicos (or '1'='1, --).
Orquestador: Wazuh no distingue qué vector se usó dentro del mismo rule.id, hay que revisarlo a mano. Falta una regla de correlación que detecte solo el patrón de "fallo repetido + éxito con sintaxis SQL".
