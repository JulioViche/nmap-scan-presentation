# Evidencia de Pentesting - Metodología PTES

## Datos del Proyecto

| Campo | Valor |
|-------|-------|
| **Proyecto** | Sistema de Atención QR |
| **Grupo** | Jairo Bonilla, Reishel Tipán, Julio Viche |
| **Objetivo** | 10.40.37.240 |
| **Metodología** | PTES (Penetration Testing Execution Standard) |
| **Herramienta** | Kali Linux |
| **Entorno** | Aplicación propia desplegada con Docker (mismo entorno del repositorio) |
| **Alcance autorizado** | Puertos 5432, 3001-3005, 5173 |

---

## FASE 1: Pre-engagement Interactions

### Actividades realizadas

- [x] Estructura de directorios de auditoría creada (`~/ptes-audit/{pre-engagement,intel,vulns,exploitation,post-exploitation,report}`)
- [x] Alcance documentado en `pre-engagement/scope.txt`
- [x] Conectividad verificada con `ping -c 4 10.40.37.240`

### Hallazgo

`ping` exitoso: 4/4 paquetes recibidos, 0% packet loss, RTT promedio ~1.1 ms, TTL=255. El host está activo y accesible en la misma LAN sin saltos de red intermedios (TTL 255 sugiere origen directo, no NAT/enrutado).

### Checklist Pre-engagement

- [ ] Autorización escrita del propietario
- [ ] Definir ventanas de tiempo para pruebas
- [ ] Establecer canal de comunicación para incidentes
- [ ] Acordar límites (no DoS, no borrado de datos)
- [x] Definir IP de origen (Kali en la LAN)

---

## FASE 2: Intelligence Gathering

### 2.1 Reconocimiento de Red (Activo)

**Comandos ejecutados:**
```bash
nmap -sn 10.40.37.240 -oA intel/host-discovery
nmap -sS -T4 10.40.37.240 -oA intel/ports-quick
```

**Evidencia:** `02-nmap-ports-quick.png`

**Hallazgos:**
- Host activo, escaneo top-1000 TCP completado en 4.78s.
- Puertos del stack autorizado confirmados abiertos: **3001, 3003, 3005, 5432**.
- **3002 y 3004 no aparecen** en el top-1000 (no son puertos "conocidos" de nmap por defecto; se confirman en el escaneo dirigido de 2.2).
- **Puertos fuera del alcance autorizado detectados:** 135 (msrpc), 139 (netbios-ssn), 445 (microsoft-ds), 2968 (enpp), 3000 (ppp), 6000 (X11), 8080 (http-proxy), 10000 (snet-sensor-mgmt). Probablemente servicios del host Windows/hipervisor o de otras VMs/contenedores en la misma LAN. **No fueron escaneados ni interactuados** por estar fuera del `scope.txt` autorizado — se documentan únicamente como observación de superficie de red circundante.

- [ ] Escaneo completo de todos los puertos TCP (`nmap -p- -sS -T4 --min-rate 1000`)
- [ ] Escaneo de puertos UDP comunes (`nmap -sU --top-ports 100`)

### 2.2 Enumeración de Servicios y Versiones

**Comando ejecutado:**
```bash
nmap -sV -sC -p 5432,3001,3002,3003,3004,3005,5173 10.40.37.240 -oA intel/services-detailed
```

**Evidencia:** `03-nmap-services-detailed.png`

**Hallazgos:**
- **3001-3005/tcp**: `Node.js Express framework` sobre HTTP, respuestas JSON sin título — confirmado como los 5 microservicios (admin, appointment, auth, medical, patient).
- **5432/tcp**: `PostgreSQL DB (Spanish)` — versión de idioma del servidor detectada, útil para banner grabbing.
- **5173/tcp**: servidor HTTP no reconocido por firma estándar de nmap; respuesta `GetRequest` devuelve el `index.html` del frontend (Vite build, `<title>frontend</title>`, assets con hash `index-CEAFBvmn.js` / `index-BTEPRGil.css`). Respuesta a `HTTPOptions`/`RTSPRequest` (preflight-like): `204 No Content` con `Access-Control-Allow-Methods: GET,HEAD,PUT,PATCH,POST,DELETE` y `Vary: Origin,Access-Control-Request-Headers` — indica middleware CORS activo en el frontend (Vite dev/preview con CORS habilitado).

- [ ] Escaneo agresivo con scripts NSE de vulnerabilidades (`--script=default,vuln`)

### 2.3 Enumeración Web (Frontend y APIs)

**Comando ejecutado (frontend):**
```bash
whatweb http://10.40.37.240:5173 -v
```
**Evidencia:** `whatweb1.png`

**Hallazgo:** SPA moderna — HTML5 + `<script type="module">` (build de Vite). Sin fuga de versión de framework (build de producción minificado). `Cache-Control: no-cache` en el HTML raíz.

**Comandos ejecutados (backends):**
```bash
whatweb http://10.40.37.240:3001 -v > intel/web-admin.txt
whatweb http://10.40.37.240:3002 -v > intel/web-appointment.txt
whatweb http://10.40.37.240:3003 -v > intel/web-auth.txt
whatweb http://10.40.37.240:3004 -v > intel/web-medical.txt
whatweb http://10.40.37.240:3005 -v > intel/web-patient.txt
```
**Evidencia:** `wahtweb2.png`, `whatweb3.png`, `whatweb4.png`

**Hallazgo (repetido en los 5 servicios — admin, appointment, auth, medical, patient):**
- **`X-Powered-By: Express`** expuesto en todas las respuestas HTTP. Severidad: **Informativo/Bajo**. Fuga de información de fingerprinting que facilita a un atacante identificar el framework backend y buscar CVEs específicos.
- **Remediación:** `app.disable('x-powered-by')` en cada microservicio, o middleware `helmet`.
- Todos los servicios comparten el mismo patrón de respuesta (200 OK, JSON, sin título), consistente con una plantilla/boilerplate común entre microservicios — una vulnerabilidad de configuración hallada en uno es candidata a repetirse en los demás.

**Comando ejecutado (headers y métodos permitidos):**
```bash
curl -I -X OPTIONS http://10.40.37.240:5173 2>/dev/null | tee intel/headers-frontend.txt
curl -I -X OPTIONS http://10.40.37.240:3001 2>/dev/null | tee intel/headers-admin.txt
curl -I -X OPTIONS http://10.40.37.240:3003 2>/dev/null | tee intel/headers-auth.txt
```
**Evidencia:** `headershttp.png`

**Hallazgo:**
- Los 3 servicios responden `204 No Content` a `OPTIONS` sin header `Origin`, con `Access-Control-Allow-Methods: GET,HEAD,PUT,PATCH,POST,DELETE` — todos los métodos de escritura permitidos a nivel de preflight genérico.
- **admin (3001) y auth (3003)** exponen `X-Powered-By: Express`; **frontend (5173) no** (sirve estático vía Vite, no Express) — diferencia de stack esperada.
- Prueba complementaria (no incluida literalmente en la guía, ejecutada para verificar políticas CORS por origen):
  ```bash
  curl -I -X OPTIONS -H "Origin: http://evil.com" http://10.40.37.240:5173
  curl -I -X OPTIONS -H "Origin: http://evil.com" -H "Access-Control-Request-Method: GET" http://10.40.37.240:3003
  curl -I -X OPTIONS -H "Origin: http://evil.com" -H "Access-Control-Request-Method: GET" http://10.40.37.240:3005
  ```
  **Evidencia:** `04b-cors-check.png`, `04b-cors-check-backends.png`
  **Hallazgo positivo (buena práctica confirmada):** `auth-service` (3003) y `patient-service` (3005) devuelven **`403 Forbidden`** ante origen no autorizado (`evil.com`) — indica whitelist de CORS activa en ambos servicios, contradiciendo la hipótesis inicial de "CORS permisivo" del modelo de amenazas para estos dos endpoints. El frontend (5173), en cambio, no refleja el `Origin` mandado ni lo rechaza explícitamente en el preflight — pendiente de validar con una petición real (no solo preflight) si aplica.

**Comando ejecutado (descubrimiento de directorios, frontend):**
```bash
gobuster dir -u http://10.40.37.240:5173 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o intel/dirs-frontend.txt
```
**Evidencia:** `gobuster.png`

**Hallazgo:** gobuster abortó por detección de **wildcard response** — el servidor devuelve `200 OK` (521 bytes) incluso para una ruta aleatoria inexistente (UUID generado por la herramienta). Comportamiento esperado de una **SPA con fallback de client-side routing**: cualquier ruta no reconocida por el servidor devuelve el mismo `index.html` para que el router del frontend la resuelva. No es una vulnerabilidad, pero invalida el escaneo de directorios en modo default.
**Ajuste técnico aplicado (no en la guía original):** relanzar con `--exclude-length 521` para excluir el wildcard y obtener resultados útiles.

- [ ] Gobuster contra admin (3001) y auth (3003)
- [ ] Fuzzing de endpoints con ffuf (admin y auth)

### 2.4 Enumeración de PostgreSQL

- [ ] Verificar versión de PostgreSQL con `nmap --script=banner`
- [ ] Scripts NSE (`pgsql-brute`, `pgsql-empty-passwords`, `pgsql-info`, `pgsql-version`)

### 2.5 Captura de Tráfico

- [ ] `tcpdump` hacia el objetivo

---

## FASE 3: Threat Modeling

- [ ] Documentar modelo de amenazas (`intel/threat-model.md`) actualizado con hallazgos reales de Fase 2

**Notas preliminares basadas en Fase 2 (a formalizar):**
- CORS parece correctamente restringido en auth-service y patient-service (contradice supuesto inicial) — revisar admin/appointment/medical.
- `X-Powered-By: Express` expuesto en los 5 microservicios — fingerprinting trivial.
- Frontend es SPA con fallback de rutas — enumeración de directorios no es efectiva sin ajuste de herramienta.
- Puertos ajenos al stack (SMB, X11, proxy, etc.) presentes en la LAN pero fuera de alcance.

---

## FASE 4: Vulnerability Analysis

- [ ] Nikto (frontend, admin, auth)
- [ ] Nuclei (los 6 servicios)
- [ ] Scripts NSE de vulnerabilidades PostgreSQL
- [ ] SQLMap contra endpoints con parámetros
- [ ] Fuerza bruta con Hydra (auth-service, PostgreSQL)
- [ ] Verificación de documentación Swagger/OpenAPI expuesta
- [ ] Prueba de endpoints sin autenticación
- [ ] Verificación de exposición del daemon Docker (2375/2376) y registro (5000)

---

## FASE 5: Exploitation

- [ ] Explotación de PostgreSQL (si hay credenciales válidas)
- [ ] Explotación Web/API (SQLi, endpoints sin auth)
- [ ] Validación de fuerza bruta exitosa
- [ ] Análisis de tokens JWT (weak secret, alg none)
- [ ] Búsqueda de exploits en Metasploit

---

## FASE 6: Post-Exploitation

- [ ] Escalada de privilegios en contenedor comprometido
- [ ] Movimiento lateral entre contenedores
- [ ] Exfiltración controlada / clasificación de datos sensibles
- [ ] Cubrir huellas (solo si autorizado)

---

## FASE 7: Reporting

- [ ] Resumen ejecutivo
- [ ] Compilación de evidencias en `report/evidence/`
- [ ] Fichas de hallazgo por vulnerabilidad (severidad, CVSS, evidencia, impacto, remediación)

---

## Checklist Final

- [ ] Autorización documentada
- [x] Pre-engagement completado (parcial — falta autorización formal)
- [x] Intelligence gathering — en progreso (2.1, 2.2, 2.3 avanzados; 2.4 y 2.5 pendientes)
- [ ] Threat modeling documentado
- [ ] Vulnerability analysis (automático + manual)
- [ ] Exploitation controlada con evidencia
- [ ] Post-exploitation (impacto demostrado)
- [ ] Reporte entregado con recomendaciones
- [ ] Limpieza de herramientas y persistencia
