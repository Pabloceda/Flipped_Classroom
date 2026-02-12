# Índice del Material de Aula Invertida

## Estructura del Curso

| Archivo | Contenido | Secciones |
|---------|-----------|-----------|
| `01_Modulo1-2-3.md` | Módulos 1, 2 y 3 | Fundamentos Web, Docker, Configuración Nginx |
| `02_Modulo4.md` | Módulo 4 | Proxy Inverso y Balanceo de Carga |
| `03_Modulo5.md` | Módulo 5 | Seguridad y HTTPS |
| `05_Casos_Practicos.md` | Casos Prácticos | Escenarios reales y automatización |
| `06_Modulo6_Final.md` | Módulo 6 | Cierre, Troubleshooting y Recursos |

---

## MÓDULO 1: Fundamentos del Servicio Web (7 slides)

- Definición hardware vs software
- Protocolo HTTP/HTTPS con códigos de estado
- Evolución HTTP/1.1 → HTTP/2 → HTTP/3 (datos de adopción 2025: ~31-40%)
- Market Share actual: Nginx 18.89%, Apache 17.24% (datos Netcraft enero 2025)
- Arquitectura Event-Driven vs Thread-Per-Connection
- Resolución del problema C10K con números reales

## MÓDULO 2: Dockerización del Servicio (8 slides)

- Inmutabilidad, portabilidad, aislamiento
- Imágenes Alpine: 5.59 MB base + 40-42 MB final (nginx:alpine vs nginx:latest 150-200 MB)
- Ciclo de vida completo (create → start → stop → rm)
- 3 tipos de networking: Bridge (defecto), Host (performance), Overlay (multi-host)
- EXPOSE vs -p (documentación vs mapeo real)
- Volúmenes y persistencia de datos
- Docker Compose con ejemplo production-ready
- Estructura de carpetas optimizada (.dockerignore, conf.d, html, logs, certs)

## MÓDULO 3: Configuración de Nginx (10 slides)

- Anatomía completa nginx.conf
- Jerarquía de contextos (http > server > location)
- Directivas server y listen
- server\_name con regex y comodines
- root vs alias con ejemplos prácticos
- Directiva index y prioridad de archivos
- Bloques location: = (exacto), ^~ (prefix), ~ (regex), ~\* (insensible caso)
- Orden de evaluación y precedencia
- Páginas de error personalizadas
- Logs en formato combined y JSON
- nginx -s reload (0% downtime)

## MÓDULO 4: Proxy Inverso y Balanceo de Carga (9 slides)

- Forward Proxy vs Reverse Proxy
- proxy\_pass con reemplazo de rutas
- Cabeceras proxy: X-Forwarded-For, X-Real-IP, X-Forwarded-Proto
- Upstreams: Grupo de servidores backend
- Algoritmos: Round Robin, Least Connections, IP Hash
- Health checks (fail\_timeout, max\_fails)
- Diagrama arquitectura completo
- Buffering y timeouts críticos
- Caso de uso: Estáticos + PHP-FPM + Node.js simultáneamente

## MÓDULO 5: Seguridad y HTTPS (9 slides)

- Certificados, claves públicas/privadas
- Handshake TLS 1.2/1.3 detallado
- Configuración SSL en puerto 443
- Redirección HTTP → HTTPS (301)
- HSTS (Strict-Transport-Security) con max-age, includeSubDomains, preload
- Autenticación básica (.htpasswd)
- Ocultación de versión (server\_tokens off)
- Control de acceso por IP (ACL, CIDR)
- Rate Limiting: limit\_req\_zone, burst, nodelay (protección DDoS básica)
- Escaneo de vulnerabilidades en imágenes Docker (Trivy, Docker Scout)

---

## CASOS PRÁCTICOS: Escenarios Reales y Automatización

- **Caso 1**: E-commerce con Alta Concurrencia (balanceo + SSL + rate limiting)
- **Caso 2**: Microservicios con API Gateway (enrutamiento por servicio)
- **Caso 3**: CDN Interno con Caching Agresivo (imágenes, CSS/JS, fuentes)
- **Caso 4**: WebSocket Proxy con Sticky Sessions (conexiones persistentes)
- Scripts de automatización: deploy.sh, monitor.sh, ssl-renew.sh
- Testing y benchmarking con Apache Bench y wrk

---

## MÓDULO 6: Cierre, Troubleshooting y Recursos (6 slides)

- Troubleshooting: diagnóstico de problemas comunes (502, 504, SSL, rate limiting, Docker)
- Herramientas de diagnóstico: curl, ab, wrk, htop, netstat, strace, tcpdump, Trivy
- Checklist de seguridad pre-producción (TLS, headers, control acceso, Docker)
- Comandos de referencia rápida (Docker, Compose, Nginx, debugging)
- Roadmap de aprendizaje: Caching → HTTP/3 → Kubernetes → WAF → Cloud-Native
- Recursos y comunidad: documentación oficial, cursos, certificaciones, herramientas
