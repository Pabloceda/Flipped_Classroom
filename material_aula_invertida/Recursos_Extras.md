# 📚 Recursos Extras - Servicios de Red e Internet

Aquí encontrarás recursos seleccionados para ampliar conocimientos, resolver dudas y profundizar en los temas del curso. Están organizados por tipo y módulo.

---

## 🎬 Vídeos Seleccionados

> Todos los vídeos han sido revisados y seleccionados por su calidad técnica y adecuación al nivel del módulo.

### Módulo 1 – Fundamentos Web y Servidores

**🔗 [Arquitectura Cliente-Servidor](https://www.youtube.com/watch?v=lC6JOQLIgp0)**
- **Justificación**: Introducción a la arquitectura cliente-servidor.
- **Duración**: ~10 min | **Nivel**: Introductorio

**🔗 [Protocolo HTTP](https://www.youtube.com/watch?v=l2MihYAj0Iw)**
- **Justificación**: Introducción al protocolo HTTP, con explicaciones visuales.
- **Duración**: 13 min | **Nivel**: Introductorio

---

### Módulo 2 – Docker e Infraestructura

**🔗 [Docker en 100 segundos (Fireship)](https://www.youtube.com/watch?v=Gjnup-PuquQ)**
- **Justificación**: Resumen visual y conciso de qué es Docker, imágenes, contenedores y por qué se usa. Perfecto como introducción rápida antes de leer el módulo completo.
- **Duración**: ~2 min | **Nivel**: Introductorio

**🔗 [Docker - Curso completo (Pabpereza)](https://pabpereza.dev/docs/cursos/docker)**
- **Justificación**: Tutorial práctico en español sobre Docker. Cubre servicios, redes, volúmenes y variables de entorno. Muy alineado con la Actividad 1 del curso.
- **Duración**: ~2 h | **Nivel**: Intermedio

**🔗 [Redes en Docker explicado](https://www.youtube.com/watch?v=bKFMS5C4CG0)**
- **Justificación**: Explica en detalle los drivers de red de Docker (bridge, host, overlay) con ejemplos prácticos. Complementa la sección 2.6 del módulo.
- **Duración**: ~40 min | **Nivel**: Intermedio

---

### Módulo 4 – Proxy Inverso y Balanceo

**🔗 [Servidor Web Compartido con Docker y Nginx Proxy Manager](https://www.youtube.com/watch?v=WCSdh37Z6Wk)**
- **Justificación**: Tutorial práctico en español sobre cómo desplegar un servidor web compartido usando Docker y Nginx Proxy Manager (NPM). Muy útil como complemento visual a la Actividad 1 del curso.
- **Duración**: ~15 min | **Nivel**: Introductorio-Intermedio

**🔗 [Acceso seguro con Nginx Proxy Manager - Reverse Proxy](https://www.youtube.com/watch?v=0ghEc_R6png&t=107s)**
- **Justificación**: Explica cómo configurar acceso seguro a servicios locales y remotos usando Nginx Proxy Manager como reverse proxy. Complementa directamente los conceptos del Módulo 4.
- **Duración**: ~20 min | **Nivel**: Intermedio

**🔗 [NGINX como Reverse Proxy - Explicado](https://www.youtube.com/watch?v=lZVAI3PqgHc)**
- **Justificación**: Demostración práctica de cómo configurar Nginx como proxy inverso, incluyendo cabeceras proxy y upstream. Directamente aplicable a la configuración del curso.
- **Duración**: ~16 min | **Nivel**: Intermedio

**🔗 [Balanceo de carga explicado visualmente](https://www.youtube.com/watch?v=sCR3SAVdyCc)**
- **Justificación**: Explica los algoritmos de balanceo (round robin, least connections, ip hash) con animaciones. Ideal para entender la Actividad 2 antes de implementarla.
- **Duración**: ~10 min | **Nivel**: Introductorio-Intermedio

---

### Módulo 5 – Seguridad y HTTPS

**🔗 [TLS/SSL explicado - Cómo funciona HTTPS](https://www.youtube.com/watch?v=j9QmMEWmcfo)**
- **Justificación**: Explica el handshake TLS, certificados, claves asimétricas y simétricas de forma visual. Esencial para entender la sección 5.1 del módulo.
- **Duración**: ~6 min | **Nivel**: Introductorio

---

## 📄 Apuntes del Curso (Markdown)

Los módulos teóricos del curso están disponibles en esta misma carpeta:

| Módulo | Archivo | Contenido |
|:-------|:--------|:----------|
| Módulo 1 | [01_Fundamentos_Web.md](./Apuntes_Teoría/01_Fundamentos_Web.md) | HTTP, arquitectura web, Apache vs Nginx |
| Módulo 2 | [02_Docker_Infraestructura.md](./Apuntes_Teoría/02_Docker_Infraestructura.md) | Imágenes, contenedores, Dockerfile, redes, volúmenes |
| Módulo 3 | [03_Nginx_Core.md](./Apuntes_Teoría/03_Nginx_Core.md) | Configuración Nginx, virtual hosts, locations |
| Módulo 4 | [04_Proxy_y_Balanceo.md](./Apuntes_Teoría/04_Proxy_y_Balanceo.md) | Proxy inverso, upstreams, algoritmos de balanceo |
| Módulo 5 | [05_Seguridad_Hardening.md](./Apuntes_Teoría/05_Seguridad_Hardening.md) | HTTPS, HSTS, rate limiting, autenticación, headers |
| Módulo 6 | [06_Operaciones_y_Debug.md](./Apuntes_Teoría/06_Operaciones_y_Debug.md) | Comandos de operación, debugging y troubleshooting |
| Módulo 7 | [07_Laboratorio_Practicos.md](./Apuntes_Teoría/07_Laboratorio_Practicos.md) | Laboratorios prácticos guiados |


---

## 🔗 Documentación Oficial

### Docker

| Recurso | URL | Para qué sirve |
|:--------|:----|:---------------|
| **Docker Docs** | [docs.docker.com](https://docs.docker.com) | Referencia completa de Docker |
| **Dockerfile reference** | [docs.docker.com/reference/dockerfile](https://docs.docker.com/reference/dockerfile/) | Todas las instrucciones del Dockerfile |
| **Docker Compose reference** | [docs.docker.com/compose](https://docs.docker.com/compose/compose-file/) | Especificación completa de docker-compose.yml |
| **Docker Hub** | [hub.docker.com](https://hub.docker.com) | Registro de imágenes oficiales |

### Nginx

| Recurso | URL | Para qué sirve |
|:--------|:----|:---------------|
| **Nginx Docs** | [nginx.org/en/docs](https://nginx.org/en/docs/) | Documentación oficial completa |
| **Nginx Beginner's Guide** | [nginx.org/en/docs/beginners_guide.html](https://nginx.org/en/docs/beginners_guide.html) | Guía de inicio oficial |
| **Nginx proxy module** | [nginx.org/en/docs/http/ngx_http_proxy_module.html](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) | Referencia de `proxy_pass` y directivas proxy |
| **Nginx upstream module** | [nginx.org/en/docs/http/ngx_http_upstream_module.html](https://nginx.org/en/docs/http/ngx_http_upstream_module.html) | Referencia de upstreams y balanceo |
| **Nginx limit_req module** | [nginx.org/en/docs/http/ngx_http_limit_req_module.html](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html) | Rate limiting |

### Seguridad y TLS

| Recurso | URL | Para qué sirve |
|:--------|:----|:---------------|
| **Mozilla SSL Config Generator** | [ssl-config.mozilla.org](https://ssl-config.mozilla.org) | Generador de configuración TLS recomendada |
| **Let's Encrypt** | [letsencrypt.org/docs](https://letsencrypt.org/docs/) | Certificados SSL gratuitos |
| **OWASP Secure Headers** | [owasp.org/www-project-secure-headers](https://owasp.org/www-project-secure-headers/) | Referencia de security headers HTTP |
| **HSTS Preload List** | [hstspreload.org](https://hstspreload.org) | Registro para HSTS Preload |
| **SSL Labs Test** | [ssllabs.com/ssltest](https://www.ssllabs.com/ssltest/) | Herramienta para auditar configuración TLS |

### Herramientas de Referencia

| Recurso | URL | Para qué sirve |
|:--------|:----|:---------------|
| **HTTP Status Codes** | [developer.mozilla.org/en-US/docs/Web/HTTP/Status](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) | Referencia de todos los códigos HTTP |
| **HTTP Headers** | [developer.mozilla.org/en-US/docs/Web/HTTP/Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers) | Referencia de cabeceras HTTP |
| **Docker Hub - Nginx** | [hub.docker.com/_/nginx](https://hub.docker.com/_/nginx) | Imagen oficial de Nginx, tags disponibles |
| **Docker Hub - MariaDB** | [hub.docker.com/_/mariadb](https://hub.docker.com/_/mariadb) | Imagen oficial de MariaDB |
| **Docker Hub - WordPress** | [hub.docker.com/_/wordpress](https://hub.docker.com/_/wordpress) | Imagen oficial de WordPress |

---

## 💡 Recomendación de Estudio

**Antes de la clase** (trabajo en casa):
1. Ver vídeos introductorios de cada módulo (~1h total)
2. Leer los apuntes del módulo correspondiente
3. Anotar dudas para resolver en clase

**Durante la clase**:
- Resolver dudas con el profesor
- Realizar las actividades prácticas
- Consultar documentación oficial cuando sea necesario

**Después de la clase**:
- Revisar vídeos de nivel intermedio para profundizar
- Practicar con los casos del examen
