# Índice del Curso

## Estructura del Curso

| Archivo | Módulo | Temática Principal |
|---------|--------|-------------------|
| `01_Fundamentos_Web.md` | Módulo 1 | Fundamentos HTTP/3, C10k, Event-Driven |
| `02_Docker_Infraestructura.md` | Módulo 2 | Docker, Imágenes, Redes, Compose |
| `03_Nginx_Core.md` | Módulo 3 | Configuración Nginx, Server Blocks, Location |
| `04_Proxy_y_Balanceo.md` | Módulo 4 | Reverse Proxy, Load Balancing (L7), Upstreams |
| `05_Seguridad_Hardening.md` | Módulo 5 | HTTPS/TLS, Rate Limiting, WAF, Hardening |
| `06_Operaciones_y_Debug.md` | Módulo 6 | Troubleshooting, Herramientas, Golden Signals |
| `07_Laboratorio_Practicos.md` | Laboratorio | Casos de uso reales y Scripts |

---

## Detalle de Contenidos

### MÓDULO 1: Fundamentos del Servicio Web (`01_Fundamentos_Web.md`)
- Definición hardware vs software
- Protocolo HTTP/HTTPS con códigos de estado
- Evolución HTTP/1.1 → HTTP/2 → HTTP/3 (QUIC/UDP)
- Market Share: Nginx vs Apache vs LiteSpeed
- Arquitectura Event-Driven (Nginx) vs Thread-Per-Connection (Apache)
- Resolución del problema C10K

### MÓDULO 2: Dockerización del Servicio (`02_Docker_Infraestructura.md`)
- Inmutabilidad, portabilidad, aislamiento
- Imágenes Alpine vs Debian (seguridad y tamaño)
- Ciclo de vida del contenedor
- Networking: Bridge, Host, Overlay
- Gestión de puertos (`EXPOSE` vs `-p`)
- Volúmenes y persistencia
- Docker Compose y `docker-compose.yml` production-ready

### MÓDULO 3: Configuración de Nginx (`03_Nginx_Core.md`)
- Anatomía de `nginx.conf` y jerarquía de contextos
- Directivas `server` y `listen`
- `server_name` (matching exacto, wildcard, regex)
- `root` vs `alias`
- Algoritmo de selección de `location` (precedencia)
- `try_files` (Front Controller Pattern para SPAs)
- Páginas de error personalizadas y Logs (JSON)

### MÓDULO 4: Proxy Inverso y Balanceo de Carga (`04_Proxy_y_Balanceo.md`)
- Concepto de Reverse Proxy vs Forward Proxy
- `proxy_pass` y normalización de URIs
- Cabeceras proxy (`X-Forwarded-For`, `X-Real-IP`, `Header Dance`)
- Upstreams y algoritmos de balanceo (Round Robin, Least Conn, IP Hash)
- Health Checks activos y pasivos

### MÓDULO 5: Seguridad y Hardening (`05_Seguridad_Hardening.md`)
- Handshake TLS 1.2/1.3
- Configuración SSL/TLS y redirección HTTPS
- HSTS (Strict-Transport-Security)
- Autenticación básica
- Ocultación de tokens de versión
- Rate Limiting y WAF (ModSecurity)
- CORS y Preflight Requests

### MÓDULO 6: Operaciones y Debugging (`06_Operaciones_y_Debug.md`)
- Golden Signals (Latencia, Tráfico, Errores, Saturación)
- Herramientas de diagnóstico (`curl`, `ab`, `wrk`, `openssl`)
- Troubleshooting de errores comunes (502, 504, loops de redirección)
- Checklist de paso a producción

### LABORATORIO PRÁCTICO (`07_Laboratorio_Practico.md`)
- **Caso 1**: E-commerce con Alta Concurrencia
- **Caso 2**: Microservicios con API Gateway
- **Caso 3**: CDN Interno
- **Caso 4**: WebSocket Proxy
- Scripts de automatización

---
