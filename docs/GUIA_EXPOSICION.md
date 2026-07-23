# 🎤 Guía de Exposición y Script de Presentación
## Pruebas de Penetración bajo Metodología PTES — Sistema de Atención QR

---

## 👥 Integrantes y Distribución de Roles Recomendada

| Integrante | Rol en la Auditoría | Diapositivas a Exponer | Enfoque Principal |
|------------|---------------------|------------------------|-------------------|
| **Jairo Bonilla** | *Intelligence Gathering & Recon Specialist* | **Slide 1 – Slide 5** | Introducción al proyecto, metodología PTES, alcance legal/técnico (Fase 1) y escaneos iniciales de red/servicios con Nmap (Fase 2). |
| **Reishel Tipán** | *Web Enumeration & Threat Modeling Analyst* | **Slide 6 – Slide 10** | Fingerprinting web (WhatWeb), análisis de cabeceras HTTP/CORS, Gobuster/wildcard routing, fuzzing con FFUF, PostgreSQL enum y tcpdump. |
| **Julio Viche** | *Vulnerability & Exploitation Lead* | **Slide 11 – Slide 16** | Modelo de amenazas (Fase 3), plan de vulnerabilidades/herramientas (Fase 4), plan de explotación/post-explotación (Fases 5-6), mapa de riesgos y conclusiones (Fase 7). |

---

## 📽️ Guía Diapositiva por Diapositiva

---

### 🔹 Slide 1: Portada
- **Expositor:** Jairo Bonilla
- **Objetivo:** Presentar al equipo, el tema y las variables fundamentales del entorno de prueba.
- **Discurso Sugerido:**
  > *"Buenos días con todos. Presentamos los resultados y avance de la auditoría de seguridad realizada al **Sistema de Atención QR**, una arquitectura orientada a microservicios desplegada en contenedores Docker. La prueba fue realizada bajo la metodología estándar **PTES** (Penetration Testing Execution Standard) utilizando como estación atacante **Kali Linux** hacia la dirección IP del servidor objetivo **10.40.37.240**. El equipo auditor está conformado por Reishel Tipán, Julio Viche y mi persona, Jairo Bonilla."*
- **Conceptos a Explicar si Preguntan:**
  - **PTES:** Estándar que define 7 fases estructuradas para auditar la seguridad sin omitir ningún vector crítico.
  - **IP 10.40.37.240:** IP asignada al servidor de producción/prueba en la red local (LAN).

---

### 🔹 Slide 2: PTES — Overview y Stack Objetivo
- **Expositor:** Jairo Bonilla
- **Objetivo:** Mostrar las 7 fases de PTES y mapear la superficie expuesta.
- **Discurso Sugerido:**
  > *"Nuestra auditoría contempla el ciclo completo de PTES, desde el Pre-engagement hasta el Reporting. En la tabla inferior observamos los 5 microservicios backend desarrollados en Node.js/Express (puertos 3001 a 3005), la base de datos PostgreSQL en el puerto 5432 y el frontend web desarrollado en Vite en el puerto 5173. Todos confirmados abiertos y escuchando en la red."*
- **Conceptos Clave:**
  - **Microservicios auditados:** `admin-service` (3001), `appointment-service` (3002), `auth-service` (3003), `medical-service` (3004), `patient-service` (3005).

---

### 🔹 Slide 3: Fase 1 — Pre-engagement Interactions
- **Expositor:** Jairo Bonilla
- **Objetivo:** Demostrar que el pentest se realiza bajo un marco legal y controlado.
- **Discurso Sugerido:**
  > *"En la Fase 1 establecimos los límites legales y operativos. Estructuramos un espacio de trabajo organizado en Kali en `~/ptes-audit/`, definimos el archivo `scope.txt` para acotar los puertos autorizados (5432, 3001-3005, 5173) y verificamos conectividad mediante un ICMP ping. El resultado mostró 0% de pérdida de paquetes con un TTL de 255 y latencia de ~1.1 ms, confirmando acceso directo en capa 2/3 dentro de la misma LAN sin saltos de enrutamiento o NAT."*
- **Conceptos Clave:**
  - **TTL 255:** *Time To Live* máximo inicial en paquetes IP; confirma conexión directa en red local sin routers intermedios pasando por Saltos (hops).

---

### 🔹 Slide 4: Escaneo de Puertos — Nmap
- **Expositor:** Jairo Bonilla
- **Objetivo:** Justificar las herramientas de escaneo inicial y los hallazgos de red.
- **Discurso Sugerido:**
  > *"Para la recolección activa ejecutamos `nmap -sS -T4`, un escaneo TCP SYN 'sigiloso' y rápido hacia los puertos TOP-1000. Confirmamos abiertos los puertos autorizados 3001, 3003, 3005 y 5432. Los puertos 3002 y 3004 no aparecieron en el TOP-1000 por ser puertos no estándar, pero fueron verificados más adelante. Además, detectamos servicios adicionales del sistema operativo base (como SMB en 445 y X11 en 6000), los cuales fueron catalogados fuera del alcance y respetados sin interactuar con ellos."*
- **Conceptos Clave:**
  - **`-sS` (SYN Stealth Scan):** Envía paquetes SYN sin completar el handshake TCP de 3 vías (no envía el ACK final), reduciendo el registro en logs tradicionales.
  - **`-T4`:** Plantilla de tiempo agresiva optimizada para redes LAN de alta velocidad.

---

### 🔹 Slide 5: Fingerprinting de Servicios — Nmap -sV -sC
- **Expositor:** Jairo Bonilla
- **Objetivo:** Identificar las versiones y tecnologías de los puertos abiertos.
- **Discurso Sugerido:**
  > *"Profundizamos el reconocimiento con `-sV -sC` sobre los puertos específicos. Identificamos que los puertos 3001 a 3005 corren sobre Node.js con el framework Express respondiendo en formato JSON. En el puerto 5432 se identificó PostgreSQL en su versión de lenguaje español. En el puerto 5173 del frontend, Nmap no reconoció la firma por tratarse de un bundle de Vite, pero la respuesta capturó los scripts minificados `index-CEAFBvmn.js`."*
- **Conceptos Clave:**
  - **`-sV` (Version Detection):** Interroga a los puertos abiertos enviando probes para determinar la aplicación y versión exacta.
  - **`-sC` (Default Scripts):** Ejecuta scripts del Nmap Scripting Engine (NSE) por defecto para extraer títulos, banners y configuraciones basicas.

---

### 🔹 Slide 6: WhatWeb — Fingerprinting de Tecnologías Web
- **Expositor:** Reishel Tipán
- **Objetivo:** Analizar la tecnología frontend y reportar el primer hallazgo de severidad baja.
- **Discurso Sugerido:**
  > *"Con la herramienta WhatWeb analizamos las aplicaciones web. El frontend se identificó como una SPA (Single Page Application) moderna sobre HTML5 minificado con Vite, sin fuga de versiones. Sin embargo, en los 5 microservicios backend detectamos el encabezado `X-Powered-By: Express`. Esto constituye un hallazgo de exposición de información (fingerprinting), ya que revela el framework backend exacto a un atacante. La remediación recomendada es aplicar `app.disable('x-powered-by')` o usar la librería `helmet`."*
- **Conceptos Clave:**
  - **X-Powered-By:** Cabecera HTTP que envía Express por defecto. Permite a los atacantes buscar vulnerabilidades (CVEs) asociadas a versiones específicas de Express/Node.

---

### 🔹 Slide 7: Headers HTTP & Análisis CORS
- **Expositor:** Reishel Tipán
- **Objetivo:** Explicar el comportamiento de CORS y la verificación de seguridad.
- **Discurso Sugerido:**
  > *"Analizamos los métodos HTTP permitidos enviando peticiones preflight `OPTIONS`. Los servicios responden `204 No Content` anunciando soporte para métodos como GET, POST, PUT, DELETE. Inicialmente se teorizó un riesgo de CORS permisivo; sin embargo, al inyectar un origen malicioso sintético (`Origin: http://evil.com`), servicios críticos como `auth-service` (3003) y `patient-service` (3005) respondieron con `403 Forbidden`. Esto confirma que existe una lista blanca de orígenes (whitelist) configurada correctamente."*
- **Conceptos Clave:**
  - **CORS (Cross-Origin Resource Sharing):** Mecanismo de seguridad de los navegadores que limita peticiones entre diferentes dominios.
  - **Preflight (OPTIONS):** Solicitud previa que hace el navegador para consultar al servidor qué métodos y orígenes están autorizados antes de enviar datos reales.
  - **403 Forbidden:** Respuesta que valida la efectividad del bloqueo hacia orígenes no autorizados.

---

### 🔹 Slide 8: Gobuster — Descubrimiento de Rutas y Wildcard Response
- **Expositor:** Reishel Tipán
- **Objetivo:** Explicar una peculiaridad técnica de las SPA al hacer fuerza bruta de directorios.
- **Discurso Sugerido:**
  > *"Al ejecutar Gobuster para descubrir archivos o rutas en el puerto 5173, la herramienta se detuvo por detectar una respuesta 'Wildcard'. Esto ocurre porque el servidor devuelve siempre `200 OK` con un tamaño de 521 bytes para cualquier URL inexistente, enviando el `index.html` para que el enrutador de React/Vite en el cliente maneje la ruta. Aclaramos que **no es una vulnerabilidad**, sino el comportamiento estándar de una SPA, el cual se soluciona técnicamente agregando el parámetro `--exclude-length 521` a Gobuster."*
- **Conceptos Clave:**
  - **Wildcard Response / Catch-all:** Configuración donde cualquier ruta no encontrada redirige a una página por defecto (generalmente `index.html`).

---

### 🔹 Slide 9: FFUF — API Endpoint Fuzzing
- **Expositor:** Reishel Tipán
- **Objetivo:** Demostrar la enumeración de endpoints en servicios REST.
- **Discurso Sugerido:**
  > *"Para mapear la estructura interna de los microservicios `admin-service` y `auth-service`, utilizamos FFUF (Fuzz Faster Fool). Fuzzear consiste en enviar miles de palabras clave reemplazando la palabra `FUZZ` en la URL objetivo y filtrando respuestas válidas por código de estado (200, 201, 401, 403). Como se observa en la evidencia, esto nos permite descubrir endpoints ocultos o no documentados en la API."*
- **Conceptos Clave:**
  - **Fuzzing:** Técnica de pruebas automatizadas que envía entradas aleatorias o de diccionarios para descubrir rutas, parámetros o fallos de software.

---

### 🔹 Slide 10: PostgreSQL Enum & Captura de Tráfico (tcpdump)
- **Expositor:** Reishel Tipán
- **Objetivo:** Explicar la enumeración de base de datos y la recolección de tráfico de red.
- **Discurso Sugerido:**
  > *"Finalizando la Fase 2, utilizamos scripts especializados de Nmap (`pgsql-info`, `pgsql-version`) contra el puerto 5432 para verificar el servicio de base de datos. En paralelo, iniciamos una captura de tráfico persistente con `tcpdump` guardada en `intel/traffic.pcap`. Esto nos permite registrar en texto plano todo el tráfico HTTP sin cifrar que fluye en la red local para su posterior análisis."*
- **Conceptos Clave:**
  - **tcpdump / .pcap:** Herramienta de captura de paquetes en línea de comandos. El archivo `.pcap` generado puede abrirse en Wireshark para analizar credenciales en texto plano.

---

### 🔹 Slide 11: Threat Modeling — Modelado de Amenazas (Fase 3)
- **Expositor:** Julio Viche
- **Objetivo:** Explicar los vectores de ataque consolidados y los escenarios de riesgo.
- **Discurso Sugerido:**
  > *"En la Fase 3 estructuramos el modelo de amenazas. Identificamos 5 vectores de riesgo clave: 1) Ausencia de un API Gateway centralizado; 2) Tráfico en texto plano HTTP sin cifrado TLS; 3) PostgreSQL accesible directamente en la red local; 4) Uso del servidor de desarrollo Vite preview en producción; y 5) Ausencia de limitación de tasa (rate limiting). Definimos 3 escenarios prioritarios: el compromiso del servicio de autenticación, la exfiltración de la base de datos y el movimiento lateral entre contenedores Docker."*
- **Conceptos Clave:**
  - **API Gateway:** Punto único de entrada que centraliza autenticación, rate limiting y SSL/TLS.
  - **Movimiento Lateral:** Técnica mediante la cual un atacante compromete un contenedor y pivotea hacia otros contenedores dentro de la misma red virtual de Docker (`172.21.0.0/24`).

---

### 🔹 Slide 12: Análisis de Vulnerabilidades (Fase 4 - Plan)
- **Expositor:** Julio Viche
- **Objetivo:** Detallar el arsenal de herramientas y las pruebas de vulnerabilidades planificadas.
- **Discurso Sugerido:**
  > *"Para la Fase 4 planificamos un escaneo de vulnerabilidades automatizado y manual. Ejecutaremos **Nikto** y **Nuclei** contra los servicios web para detectar desconfiguraciones; **SQLMap** contra endpoints con parámetros para buscar inyecciones SQL; y **Hydra** para pruebas de fuerza bruta de credenciales en `auth-service` y en PostgreSQL. También validaremos si el daemon de Docker (`2375/2376`) está expuesto hacia la red."*
- **Conceptos Clave:**
  - **Nuclei:** Escáner moderno basado en plantillas YAML para detectar vulnerabilidades conocidas rápidamente.
  - **SQLMap:** Herramienta automatizada para detectar y explotar vulnerabilidades de SQL Injection.
  - **Hydra:** Herramienta de ataque por fuerza bruta de contraseñas altamente paralela.

---

### 🔹 Slide 13: Explotación y Post-Explotación (Fases 5 y 6 - Plan)
- **Expositor:** Julio Viche
- **Objetivo:** Mostrar cómo se demuestra el impacto real de los hallazgos.
- **Discurso Sugerido:**
  > *"En las Fases 5 y 6 demostraremos el impacto real. En explotación, intentaremos el acceso directo a PostgreSQL para verificar el acceso a tablas con información sensible; analizaremos los tokens JWT utilizando `jwt_tool.py` probando firmas débiles o algoritmo `none`; y utilizaremos módulos de Metasploit si aplica. En post-explotación, verificaremos si estamos dentro de un contenedor (`/.dockerenv`), extrayendo secretos de las variables de entorno (`env`) y clasificando datos sensibles como PII (datos de pacientes) y PHI (historiales médicos)."*
- **Conceptos Clave:**
  - **JWT (JSON Web Token):** Token de sesión usado en los microservicios. Se audita que no use claves débiles o el fallo `alg: none`.
  - **PII / PHI:** *Personally Identifiable Information* y *Protected Health Information*. Datos críticos regulados legalmente.

---

### 🔹 Slide 14: Mapa de Riesgo y Resumen de Hallazgos
- **Expositor:** Julio Viche
- **Objetivo:** Sintetizar el estado actual de la auditoría y la severidad de los hallazgos.
- **Discurso Sugerido:**
  > *"Sintetizando la superficie de ataque, confirmamos 5 hallazgos principales en la Fase 2: El puerto de PostgreSQL accesible en LAN (Riesgo Alto); comunicación HTTP sin cifrar y Vite preview en producción (Riesgo Medio); exposición del encabezado Express (Riesgo Bajo); y la validación positiva de whitelist en CORS. Actualmente la auditoría presenta un avance del 80% en Pre-engagement y 70% en Intelligence Gathering."*
- **Conceptos Clave:**
  - **Niveles de Riesgo:** Clasificación basada en el impacto de negocio y la facilidad de explotación.

---

### 🔹 Slide 15: Recomendaciones de Remediación (Fase 7)
- **Expositor:** Julio Viche
- **Objetivo:** Aportar valor defensivo y recomendaciones de seguridad de acuerdo al estándar.
- **Discurso Sugerido:**
  > *"Como parte del reporte en la Fase 7, planteamos remediaciones por prioridad. **Alta prioridad:** Aislar PostgreSQL detrás de un firewall interno, implementar un API Gateway como Nginx o Kong y habilitar TLS/HTTPS. **Media prioridad:** Migrar el frontend a un servidor web de producción (como Nginx) y ocultar la cabecera Express con `helmet`. Asimismo, remarcamos los principios éticos: mantener estricto control de datos PII/PHI sin exfiltrarlos y realizar la limpieza total post-prueba."*
- **Conceptos Clave:**
  - **Helmet:** Middleware de Node.js que configura automáticamente cabeceras HTTP de seguridad (HSTS, X-Content-Type-Options, etc.).

---

### 🔹 Slide 16: Conclusiones y Próximos Pasos
- **Expositor:** Julio Viche
- **Objetivo:** Cerrar la presentación con métricas claras de lo realizado y los pasos futuros.
- **Discurso Sugerido:**
  > *"En conclusión, hemos documentado las 7 fases de la metodología PTES, auditado 7 servicios/puertos con 12 evidencias capturadas y 5 hallazgos confirmados. Los siguientes pasos inmediatos consisten en completar la fuzzificación de rutas en las APIs, relanzar el escaneo de vulnerabilidades automatizado con Nikto/Nuclei y redactar el informe ejecutivo final con la matriz CVSS v3.1. Quedamos atentos a sus preguntas. Muchas gracias."*
- **Conceptos Clave:**
  - **CVSS v3.1:** *Common Vulnerability Scoring System*, estándar internacional para calcular numéricamente (0.0 a 10.0) la severidad de una vulnerabilidad.

---

## ❓ Preguntas Probables del Jurado y Respuestas Clave

1. **¿Por qué utilizaron la metodología PTES y no OWASP Top 10 o NIST?**
   - *Respuesta:* *"OWASP se enfoca principalmente en vulnerabilidades de aplicaciones web a nivel capa 7. PTES es una metodología integral de pentesting que cubre la infraestructura de red, sistemas, contenedores Docker y servicios backend en 7 fases estructuradas desde el compromiso previo hasta el reporte final."*

2. **¿Por qué la cabecera `X-Powered-By: Express` es un riesgo si solo dice el nombre del framework?**
   - *Respuesta:* *"Porque reduce el trabajo de reconocimiento de un atacante (fingerprinting). Al saber que es Express en Node.js, el atacante no pierde tiempo probando exploits de PHP o ASP.NET y busca directamente vulnerabilidades específicas de paquetes NPM o del motor Node."*

3. **¿Qué significa que Gobuster haya dado 'Wildcard Response' en la SPA y cómo lo solucionan?**
   - *Respuesta:* *"Significa que el servidor devuelve `200 OK` para cualquier URL inexistente porque la SPA redirige todas las rutas hacia `index.html`. No es un fallo de seguridad, sino una característica del enrutamiento en cliente. Se soluciona indicando a Gobuster que ignore respuestas con el tamaño exacto de esa página por defecto (`--exclude-length 521`)."*

4. **¿Por qué es peligroso tener el puerto 5432 de PostgreSQL abierto hacia la red local?**
   - *Respuesta:* *"Porque cualquier equipo o atacante en la misma LAN puede intentar conexiones directas, ataques de fuerza bruta de credenciales o explotar posibles vulnerabilidades del motor de base de datos sin pasar por los controles de autenticación de la API."*
