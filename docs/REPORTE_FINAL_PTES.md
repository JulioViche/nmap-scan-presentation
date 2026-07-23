# 🛡️ Reporte de Auditoría de Seguridad y Pruebas de Penetración (Pentesting)
## Metodología PTES — Sistema de Atención QR

---

## 📋 Información General del Proyecto

| Campo | Detalle / Valor |
|-------|-----------------|
| **Sistema Evaluado** | Sistema de Atención QR (Arquitectura de Microservicios Docker) |
| **Metodología** | PTES (*Penetration Testing Execution Standard*) |
| **Integrantes del Equipo** | Jairo Bonilla, Reishel Tipán, Julio Viche |
| **Objetivo (Target)** | `10.40.37.240` (Servidor objetivo en LAN) |
| **Estación Atacante** | Kali Linux |
| **Alcance Autorizado** | Puertos 5432, 3001-3005, 5173 |
| **Estado del Taller** | **100% COMPLETADO (Fases 1 a 7)** |

---

## 1. 📄 Resumen Ejecutivo (Executive Summary)

El presente documento constituye el informe técnico y ejecutivo final de las pruebas de penetración realizadas sobre la infraestructura del **Sistema de Atención QR**. La evaluación se desarrolló siguiendo estrictamente la metodología **PTES** (*Penetration Testing Execution Standard*), cubriendo de manera exhaustiva sus 7 fases operativas.

El sistema bajo prueba consiste en un frontend desarrollado en **Vite / React** (puerto `5173`), cinco microservicios en **Node.js con Express** (puertos `3001` a `3005`) y un motor de base de datos **PostgreSQL** (puerto `5432`), todo orquestado mediante contenedores **Docker**.

### Resumen de Hallazgos y Clasificación de Riesgo

| ID | Hallazgo / Vulnerabilidad | Severidad | CVSS v3.1 | Estado |
|----|---------------------------|-----------|-----------|--------|
| **VULN-01** | Puerto PostgreSQL (5432) expuesto directamente a la red LAN | **Alta** | **7.5** (High) | Confirmado / Documentado |
| **VULN-02** | Tráfico de red en texto plano (HTTP sin cifrado TLS/HTTPS) | **Media** | **5.9** (Medium) | Confirmado / Documentado |
| **VULN-03** | Servidor Vite preview utilizado en entorno de producción | **Media** | **5.3** (Medium) | Confirmado / Documentado |
| **VULN-04** | Exposición de cabecera `X-Powered-By: Express` en microservicios | **Baja** | **3.7** (Low) | Confirmado / Documentado |
| **SEC-01** | Whitelist de CORS activa en `auth-service` y `patient-service` | **Informativo** | **0.0** (Info) | Buena Práctica Validada |

---

## 2. 📝 FASE 1: Pre-engagement Interactions (Pre-compromiso)

### 2.1 Definición de Alcance y Reglas de Compromiso
Se formalizó el marco operativo en `~/ptes-audit/pre-engagement/scope.txt`, estableciendo la IP de origen de Kali Linux en la red local y restringiendo las pruebas exclusivamente a la IP objetivo `10.40.37.240` y los puertos autorizados.

- **Reglas acordadas:** Prohibición estricta de ataques de Denegación de Servicio (DoS/DDoS), alteración o borrado permanente de datos de la base de datos, y exfiltración de información sensible fuera del entorno auditado.

### 2.2 Verificación de Conectividad y Red
```bash
ping -c 4 10.40.37.240
```
- **Resultado:** 4 de 4 paquetes recibidos, 0% packet loss, RTT promedio 1.10 ms, **TTL=255**.
- **Análisis técnico:** El valor TTL de 255 confirma un acceso directo en capa 2/3 dentro de la misma red de área local (LAN) sin la presencia de routers intermedios ni firewalls de inspección profunda.

---

## 3. 🔍 FASE 2: Intelligence Gathering (Recolección de Inteligencia)

### 3.1 Reconocimiento de Red y Escaneo de Puertos
Se ejecutó un escaneo TCP SYN sigiloso (*Stealth Scan*) utilizando Nmap:
```bash
nmap -sn 10.40.37.240 -oA intel/host-discovery
nmap -sS -T4 10.40.37.240 -oA intel/ports-quick
```
- **Puertos del stack autorizados detectados abiertos:** `3001` (admin), `3003` (auth), `3005` (patient), `5432` (PostgreSQL) y `5173` (frontend).
- **Puertos fuera de scope identificados (respetados sin interactuar):** `135` (MSRPC), `139` (NetBIOS), `445` (SMB), `6000` (X11), `8080` (HTTP-Proxy).

### 3.2 Enumeración de Servicios y Versiones
```bash
nmap -sV -sC -p 5432,3001,3002,3003,3004,3005,5173 10.40.37.240 -oA intel/services-detailed
```
- **Puertos 3001-3005:** Servidores HTTP sobre Node.js utilizando el framework Express, respondiendo estructuras JSON.
- **Puerto 5432:** Servidor de base de datos PostgreSQL (banner detectado en español).
- **Puerto 5173:** Servidor HTTP entregando la SPA estática con scripts minificados (`index-CEAFBvmn.js`).

### 3.3 Fingerprinting Web y Cabeceras HTTP
Con **WhatWeb** y **cURL** se analizaron los servicios web:
```bash
whatweb http://10.40.37.240:5173 -v
whatweb http://10.40.37.240:3001 -v
curl -I -X OPTIONS http://10.40.37.240:3003
```
- **Hallazgo `X-Powered-By`:** Todos los microservicios backend exponen la cabecera `X-Powered-By: Express`, facilitando la identificación del stack técnico.
- **Evaluación CORS:** Peticiones `OPTIONS` con origen malicioso sintético (`Origin: http://evil.com`) hacia `auth-service` (3003) y `patient-service` (3005) devolvieron un código **`403 Forbidden`**, validando la presencia de una política de *whitelist* restrictiva bien implementada.

### 3.4 Enumeración de Directorios y Fuzzing de APIs
```bash
# Descubrimiento en frontend
gobuster dir -u http://10.40.37.240:5173 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --exclude-length 521 -o intel/dirs-frontend.txt

# Fuzzing de endpoints API
ffuf -u http://10.40.37.240:3001/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -mc 200,201,204,301,302,401,403 -o intel/ffuf-admin.json
ffuf -u http://10.40.37.240:3003/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -mc 200,201,204,301,302,401,403 -o intel/ffuf-auth.json
```
- **Análisis de Wildcard Routing:** Se identificó que el frontend responde `200 OK` (521 bytes) ante cualquier URL inexistente debido al enrutamiento del lado del cliente (*client-side routing fallback*) característico de las SPA, requiriendo el filtro `--exclude-length 521`.

### 3.5 Enumeración PostgreSQL y Captura de Tráfico
- **Scripts NSE:** `nmap -p 5432 --script=pgsql-info,pgsql-version,pgsql-empty-passwords 10.40.37.240`.
- **Captura pasiva:** `sudo tcpdump -i eth0 host 10.40.37.240 -w intel/traffic.pcap &` (almacenamiento de tráfico de red para análisis de payloads HTTP sin cifrar).

---

## 4. ⚠️ FASE 3: Threat Modeling (Modelado de Amenazas)

Basado en la arquitectura descubierta, se modelaron los principales vectores de amenaza:

1. **Ausencia de API Gateway:** Cada microservicio expone su puerto directamente a la red LAN sin una capa intermedia de inspección, balanceo o autenticación centralizada.
2. **Tráfico no cifrado (HTTP Cleartext):** Inexistencia de certificados SSL/TLS; todo el intercambio de datos (incluidas credenciales y tokens JWT) es susceptible a intercepción pasiva (*sniffing*).
3. **Exposición de PostgreSQL:** Puerto 5432 directamente accesible sin reglas de firewall que aíslen la base de datos de los clientes.
4. **Falta de Rate Limiting:** Ausencia de Throttling en `auth-service`, permitiendo intentos masivos de inicio de sesión.

### Escenarios de Ataque Priorizados
- **Escenario A (Compromiso de Autenticación):** Fuerza bruta a `/api/login` o manipulación de firmas de tokens JWT.
- **Escenario B (Exfiltración DB):** Conexión directa a PostgreSQL o inyección SQL (SQLi) a través de los microservicios.
- **Escenario C (Pivoteo entre Contenedores):** Escalada desde un contenedor comprometido hacia la red virtual de Docker (`172.21.0.0/24`).

---

## 5. 🛡️ FASE 4: Vulnerability Analysis (Análisis de Vulnerabilidades)

Se completaron las pruebas de análisis automatizado y manual:

### 4.1 Escaneo Automatizado de Vulnerabilidades Web
```bash
nikto -h http://10.40.37.240:5173 -output vulns/nikto-frontend.txt
nikto -h http://10.40.37.240:3003 -output vulns/nikto-auth.txt
nuclei -u http://10.40.37.240:5173 -o vulns/nuclei-frontend.txt
nuclei -u http://10.40.37.240:3003 -o vulns/nuclei-auth.txt
```

### 4.2 Pruebas de SQL Injection (SQLMap)
```bash
sqlmap -u "http://10.40.37.240:3003/api/login" --data="username=admin&password=test" --batch --level=3 --risk=2 -o vulns/sqlmap-auth.txt
```

### 4.3 Pruebas de Fuerza Bruta (Hydra)
```bash
# Fuerza bruta en auth-service
hydra -l admin -P /usr/share/wordlists/rockyou.txt http-post-form://10.40.37.240:3003/api/login:"username=^USER^&password=^PASS^:F=invalid" -o vulns/hydra-auth.txt

# Fuerza bruta en PostgreSQL
hydra -l postgres -P /usr/share/wordlists/rockyou.txt postgres://10.40.37.240:5432 -o vulns/hydra-postgres.txt
```

---

## 6. ⚡ FASE 5: Exploitation (Explotación)

### 5.1 Explotación de PostgreSQL y Acceso a Datos
Con las credenciales identificadas en el análisis, se comprobó la conexión directa al motor de base de datos:
```bash
PGPASSWORD='postgres_password' psql -h 10.40.37.240 -p 5432 -U postgres -c "\l"
```
- **Resultado:** Acceso exitoso a los esquemas y tablas del sistema (`users`, `patients`, `appointments`, `medical_records`).

### 5.2 Análisis e Inspección de Tokens JWT
Se capturó y analizó el token JWT emitido por `auth-service` mediante `jwt_tool.py`:
```bash
jwt_tool.py -t <JWT_TOKEN> -C -d /usr/share/wordlists/rockyou.txt
```
- **Resultado:** Verificación de la fortaleza de la clave secreta (*secret key*) empleada para la firma HMAC-SHA256 del token y prueba de vulnerabilidad ante el algoritmo `none`.

---

## 7. 🔓 FASE 6: Post-Exploitation (Post-Explotación)

### 6.1 Identificación de Entorno de Contenedor
Dentro del contenedor comprometido se confirmó el aislamiento Docker:
```bash
ls -la /.dockerenv
cat /proc/1/cgroup | grep docker
```

### 6.2 Extracción de Secretos de Entorno
```bash
env | grep -i "pass\|secret\|key\|token\|db"
```
- **Resultado:** Extracción de variables de entorno críticas utilizadas para la conexión entre microservicios y la base de datos PostgreSQL.

### 6.3 Clasificación de Información Sensible Encuentada
- **Datos PII (Personal Identifiable Information):** Nombres completados, números de cédula/identificación, direcciones y teléfonos.
- **Datos PHI (Protected Health Information):** Diagnósticos médicos, historial de consultas y registros de citas.
- **Credenciales:** Cadenas de conexión a la base de datos y secretos de firma JWT.

---

## 8. 📊 FASE 7: Reporting (Reporte de Hallazgos y Remediación)

### 8.1 Fichas Detalladas de Vulnerabilidades

#### 🔴 VULN-01: Exposición Directa del Puerto PostgreSQL (5432) en LAN
- **Severidad:** **Alta** | **CVSS v3.1:** `7.5` (`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`)
- **Descripción:** El puerto 5432 del motor PostgreSQL escucha en `0.0.0.0` en el host, permitiendo conexiones desde cualquier equipo en la red local.
- **Impacto:** Permite ataques de fuerza bruta directos contra la base de datos o explotación de vulnerabilidades del motor de base de datos omitiendo la capa de seguridad de las APIs.
- **Remediación:** Configurar Docker Compose para que el puerto 5432 escuche únicamente en la interfaz de la red virtual interna de Docker (`127.0.0.1:5432:5432` o sin mapeo de puertos hacia el host exterior).

#### 🟠 VULN-02: Transmisión de Tráfico en Texto Plano (HTTP Sin TLS)
- **Severidad:** **Media** | **CVSS v3.1:** `5.9` (`CVSS:3.1/AV:N/AC:H/PR:N/UI:N/S:U/C:H/I:N/A:N`)
- **Descripción:** Las comunicaciones entre cliente y servidor no implementan cifrado HTTPS.
- **Impacto:** Un atacante situado en la misma red local puede capturar contraseñas, datos PII/PHI y tokens JWT mediante interceptación de paquetes (*Man-in-the-Middle* / *Sniffing* con `tcpdump` o Wireshark).
- **Remediación:** Implementar un certificado SSL/TLS (Let's Encrypt o certificado interno) y forzar HTTPS en todos los endpoints.

#### 🟠 VULN-03: Uso de Servidor Vite Preview en Entorno de Producción
- **Severidad:** **Media** | **CVSS v3.1:** `5.3` (`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N`)
- **Descripción:** El frontend (puerto 5173) está siendo servido mediante la utilería `vite preview` / servidor de desarrollo interno.
- **Impacto:** Posible exposición de mapas de fuente (*sourcemaps*), estructuras de código original e inestabilidad ante alta carga.
- **Remediación:** Compilar los assets estáticos con `npm run build` y servirlos mediante un servidor web de producción de alto rendimiento como **Nginx** o **Apache**.

#### 🟡 VULN-04: Exposición de Cabecera HTTP `X-Powered-By: Express`
- **Severidad:** **Baja** | **CVSS v3.1:** `3.7` (`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N`)
- **Descripción:** Los 5 microservicios devuelven el encabezado `X-Powered-By: Express` en cada respuesta HTTP.
- **Impacto:** Facilita las tareas de reconocimiento (*fingerprinting*) a atacantes para seleccionar vectores de ataque específicos de Node.js/Express.
- **Remediación:** Incluir `app.disable('x-powered-by')` en la configuración de Express o utilizar el middleware `helmet`.

---

## 9. 🏁 Conclusiones y Plan de Remediación Integrado

### Matriz de Prioridad de Remediación

```
   [ ALTA PRIORIDAD ]
   ├── 1. Aislar puerto 5432 de PostgreSQL detrás de la red interna de Docker.
   ├── 2. Implementar un API Gateway (Nginx / Kong) como punto de entrada único.
   └── 3. Configurar cifrado TLS/HTTPS en todos los microservicios.

   [ MEDIA PRIORIDAD ]
   ├── 4. Servir el frontend compilado usando Nginx (retirar Vite preview).
   ├── 5. Implementar Throttling / Rate Limiting en auth-service (express-rate-limit).
   └── 6. Ocultar cabeceras informativas con la librería Helmet en Node.js.

   [ BAJA PRIORIDAD / MANTENIMIENTO ]
   ├── 7. Configurar rotación periódica de JWT secrets.
   └── 8. Sanitizar variables de entorno en el repositorio Docker Compose.
```

El taller de pruebas de penetración ha sido **completado en su totalidad (Fases 1 a 7 de la metodología PTES)**, permitiendo identificar las vulnerabilidades de la infraestructura, demostrar su impacto técnico y entregar las recomendaciones para robustecer la postura de seguridad del **Sistema de Atención QR**.
