## MÓDULO 6: CIERRE, TROUBLESHOOTING Y RECURSOS

A lo largo de los módulos anteriores hemos construido una infraestructura web completa: desde los fundamentos del protocolo HTTP y la arquitectura de Nginx (Módulo 1), pasando por la dockerización del servicio (Módulo 2), la configuración de contenido estático y virtual hosts (Módulo 3), el proxy inverso y balanceo de carga (Módulo 4), hasta la seguridad con HTTPS, rate limiting y escaneo de imágenes (Módulo 5). Este último módulo cierra la investigación con las habilidades que todo administrador necesita en el **día a día**: diagnosticar problemas, conocer las herramientas clave, tener una referencia rápida de comandos y saber por dónde continuar aprendiendo.

---

### 6.1 Troubleshooting: Diagnóstico de Problemas Comunes

El primer reto al que se enfrenta un administrador no es configurar, sino **resolver problemas cuando algo deja de funcionar**. A continuación se recogen los errores más frecuentes con Nginx en Docker y cómo diagnosticarlos paso a paso.

**1. Nginx no arranca**

```bash
# Verificar sintaxis de configuración
nginx -t

# Ver logs de error detallados
tail -f /var/log/nginx/error.log

# Verificar puerto ocupado
netstat -tlnp | grep :80
# O con ss (moderno)
ss -tlnp | grep :80

# Verificar permisos
ls -la /var/log/nginx/
ls -la /etc/nginx/nginx.conf
```

**2. Error 502 Bad Gateway**

```bash
# Causas comunes:
- Backend caído → docker-compose ps
- Timeout corto → aumentar proxy_read_timeout
- SELinux bloqueando → setenforce 0 (temporal)

# Verificar conectividad al backend
docker exec nginx curl -v http://backend:3000/health

# Ver logs del backend
docker-compose logs --tail=50 backend
```

**3. Error 504 Gateway Timeout**

```nginx
# Aumentar timeouts en nginx.conf
location /api/ {
    proxy_connect_timeout 10s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;  # ← Aumentar si backend es lento
}
```

**4. Rendimiento degradado**

```bash
# Monitorizar en tiempo real
docker stats

# Ver conexiones activas
docker exec nginx sh -c 'echo "GET /nginx_status HTTP/1.0\r\n" | nc localhost 80'

# Top IPs consumidoras
docker exec nginx awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Identificar requests lentos (>1s)
docker exec nginx awk '{if ($NF > 1.0) print $0}' /var/log/nginx/access.log | tail -20
```

**5. Problemas SSL/TLS**

```bash
# Test handshake SSL
openssl s_client -connect ejemplo.com:443 -servername ejemplo.com

# Verificar certificado válido
echo | openssl s_client -servername ejemplo.com -connect ejemplo.com:443 2>/dev/null | openssl x509 -noout -dates

# Test con SSL Labs
# https://www.ssllabs.com/ssltest/analyze.html?d=ejemplo.com
```

**6. Rate Limiting no funciona**

```bash
# Verificar zona configurada
grep "limit_req_zone" /etc/nginx/nginx.conf

# Test manual
for i in {1..20}; do curl -w "%{http_code}\n" http://localhost/api; sleep 0.1; done
# Debe mostrar 503 o 429 después de burst

# Ver logs de rate limit
docker exec nginx grep "limiting requests" /var/log/nginx/error.log
```

**7. Problemas específicos de Docker**

```bash
# Contenedor en restart loop
docker logs nginx --tail=100

# Verificar healthcheck
docker inspect nginx | jq '..State.Health'

# Red no alcanzable entre contenedores
docker network inspect nombre_red
docker exec nginx ping -c 3 backend

# Volúmenes no montados
docker inspect nginx | jq '..Mounts'
```


---

### 6.2 Herramientas de Diagnóstico

Diagnosticar un problema es más fácil si se conocen las **herramientas adecuadas**. Esta tabla resume las más utilizadas en el ecosistema Nginx + Docker:

#### Toolkit del Administrador

| Herramienta | Uso | Ejemplo |
| :-- | :-- | :-- |
| **nginx -t** | Validar sintaxis | `docker exec nginx nginx -t` |
| **curl** | Test endpoints | `curl -I -H "X-API-Key: test" https://api.com` |
| **ab** | Benchmark | `ab -n 1000 -c 100 http://localhost/` |
| **wrk** | Load testing avanzado | `wrk -t12 -c400 -d30s http://localhost/` |
| **htop** | Monitorizar procesos | `htop` (dentro del contenedor) |
| **netstat / ss** | Ver puertos abiertos | `netstat -tlnp \| grep nginx` |
| **strace** | Trace syscalls | `strace -p $(pgrep nginx)` |
| **tcpdump** | Capturar tráfico | `tcpdump -i any port 80 -w capture.pcap` |
| **jq** | Parse JSON logs | `cat access.json \| jq '.status'` |
| **docker stats** | Uso recursos | `docker stats --no-stream` |
| **Trivy** | Scan vulnerabilidades | `trivy image nginx:alpine` |

#### Monitorización Continua

**Script de monitoreo básico**:

```bash
# monitor.sh - ejecutar cada minuto con cron
*/1 * * * * /opt/scripts/monitor.sh >> /var/log/monitor.log 2>&1
```

**Métricas clave a vigilar:**

- **Latencia** (Latency): Tiempo que tarda en servir una petición.
- **Tráfico** (Traffic): Demanda del sistema (stats de requests/seg).
- **Errores** (Errors): Tasa de fallos (códigos 5xx).
- **Saturación** (Saturation): Cuan "lleno" está el servicio (uso CPU/RAM).

---

### 6.3 Checklist de Seguridad Pre-Producción

Antes de desplegar en producción, es fundamental verificar que se han aplicado todas las medidas de seguridad que vimos en el Módulo 5. Este checklist resume los puntos críticos en un formato práctico para revisar antes de cada despliegue:

**TLS/SSL**:

- [ ] TLS 1.2 y 1.3 solamente (desactivar TLS 1.0/1.1)
- [ ] Ciphers fuertes (AES-GCM, ChaCha20-Poly1305)
- [ ] Certificados de CA confiable (Let's Encrypt, DigiCert)
- [ ] Renovación automática de certificados

**Headers de Seguridad**:

- [ ] HSTS habilitado (`max-age=31536000; includeSubDomains`)
- [ ] `server_tokens off` (ocultar versión)
- [ ] `X-Frame-Options: DENY` (anti-clickjacking)
- [ ] `X-Content-Type-Options: nosniff`
- [ ] `X-XSS-Protection: 1; mode=block`
- [ ] CSP (Content Security Policy) configurado

**Control de Acceso**:

- [ ] Rate limiting en login/API (`limit_req`)
- [ ] IP whitelisting para admin panels
- [ ] HTTP Basic Auth + HTTPS (o mejor: OAuth2/JWT)
- [ ] Geo-blocking si aplica

**Docker Security**:

- [ ] Usuario no-root en contenedor
- [ ] Capabilities dropped (`--cap-drop=ALL`)
- [ ] Read-only filesystem donde sea posible
- [ ] Escaneo de vulnerabilidades (Trivy)
- [ ] Pin de versiones (no `:latest`)

**Configuración**:

- [ ] Directorio de archivos ocultos bloqueado (`location ~ /\.`)
- [ ] Tamaño máximo de upload configurado (`client_max_body_size`)
- [ ] Timeouts configurados (prevenir slow loris)
- [ ] Logs monitorizados y rotados

---

### 6.4 Comandos de Referencia Rápida

Para el día a día, resulta muy útil tener un **cheat sheet** con los comandos más frecuentes. A continuación se recogen agrupados por herramienta:

**Docker**:

```bash
docker build -t my-nginx .                                                  # Build imagen
docker run -d --name web -p 80:80 -v ./html:/usr/share/nginx/html nginx:alpine  # Run
docker logs -f web                                                          # Logs
docker exec -it web sh                                                      # Shell
docker exec web nginx -t                                                    # Test config
docker exec web nginx -s reload                                             # Reload
docker stop web && docker start web                                         # Stop/Start
docker stats web                                                            # Stats
```

**Docker Compose**:

```bash
docker-compose up -d              # Levantar servicios
docker-compose logs -f nginx      # Ver logs
docker-compose restart nginx      # Reiniciar servicio
docker-compose up -d --scale app=3  # Escalar
docker-compose down               # Detener y eliminar
docker-compose build               # Reconstruir imágenes
```

**Nginx**:

```bash
nginx -t            # Test sintaxis configuración
nginx -s reload     # Reload sin downtime
nginx -s stop       # Stop inmediato
nginx -s quit       # Stop graceful
nginx -s reopen     # Reopen log files
nginx -V            # Mostrar config compilada y módulos
```

**Debugging**:

```bash
tail -f /var/log/nginx/access.log                          # Peticiones en tiempo real
tail -f /var/log/nginx/error.log                           # Errores
curl -I https://ejemplo.com                                # Test endpoint
curl -H "X-API-Key: test" https://ejemplo.com/api          # Test con headers
for i in {1..20}; do curl http://ejemplo.com; done          # Test rate limiting
openssl s_client -connect ejemplo.com:443 -servername ejemplo.com  # Check SSL
```

---

### 6.5 Roadmap de Aprendizaje

Con los conocimientos adquiridos en estos 6 módulos, el alumno tiene una **base sólida** en despliegue y administración de servidores web con Nginx y Docker. A continuación se sugieren los siguientes pasos para profundizar:

#### Nivel 1: Optimización y Producción (1-2 meses)

**Caching Avanzado**

- FastCGI cache para PHP (WordPress, Drupal)
- Proxy cache con purging (CDN interno)
- Microcaching (1-5s TTL para contenido semi-dinámico)
- Cache warming automatizado

**HTTP/3 y QUIC**

- Compilar Nginx con soporte QUIC
- Configurar UDP 443 para HTTP/3
- Medir mejoras de latencia (especialmente móvil)
- Fallback a HTTP/2 transparente

**Observabilidad**

- Nginx Amplify / Datadog / New Relic integración
- Prometheus + Grafana dashboards
- OpenTelemetry tracing (request IDs)
- Alerting con PagerDuty / Slack


#### Nivel 2: Orquestación y Escalado (2-4 meses)

**Kubernetes + Ingress**

- Nginx Ingress Controller en K8s
- Cert-manager para SSL automático
- Horizontal Pod Autoscaling (HPA)
- Service Mesh (Istio / Linkerd) vs Ingress

**CI/CD Avanzado**

- Blue-Green deployments con Docker Swarm / K8s
- Canary releases (1% tráfico → 10% → 100%)
- A/B testing con split traffic por header/cookie
- Automated rollback en caso de error rate spike

**Multi-Region y Alta Disponibilidad**

- GeoDNS con Route 53 / Cloudflare
- Active-Active entre datacenters
- Database replication (PostgreSQL streaming, MySQL Group Replication)
- Disaster Recovery playbooks


#### Nivel 3: Seguridad Avanzada (1-2 meses)

**WAF (Web Application Firewall)**

- ModSecurity con OWASP Core Rule Set
- Bloqueo de SQL injection, XSS, RCE
- Rate limiting inteligente (por endpoint, por usuario)
- Bot detection y mitigation

**Hardening Avanzado**

- Fail2Ban para bloqueo automático de IPs
- Security headers: CSP (Content Security Policy), Permissions-Policy
- Subresource Integrity (SRI) para CDN
- Regular security audits con Nessus / OpenVAS

**Compliance**

- GDPR: logs anonimizados, data retention policies
- PCI-DSS: segmentación de red, logs de auditoría
- SOC 2: access controls, encryption at rest


#### Nivel 4: Arquitecturas Cloud-Native (3-6 meses)

**Serverless Backends**

- Nginx → AWS Lambda / Google Cloud Functions
- API Gateway vs Nginx como BFF (Backend for Frontend)
- Event-driven architectures (SQS, Kafka)

**Service Mesh**

- Istio / Linkerd para microservicios
- mTLS automático entre servicios
- Circuit breakers y retry policies
- Observability out-of-the-box

**Edge Computing**

- Nginx en edge locations (Cloudflare Workers, Lambda@Edge)
- CDN programable con lógica custom
- Reduce latencia global (<50ms P95)

---

### 6.6 Recursos y Comunidad

Para seguir aprendiendo y resolver dudas, estos son los recursos de referencia más importantes:

#### Documentación Oficial

**Nginx**

- https://nginx.org/en/docs/ - Documentación completa
- https://docs.nginx.com/ - Nginx Plus (features comerciales)
- https://github.com/nginx/nginx - Código fuente

**Docker**

- https://docs.docker.com/ - Docker Engine y CLI
- https://docs.docker.com/compose/ - Docker Compose
- https://hub.docker.com/_/nginx - Imagen oficial Nginx

**Seguridad**

- https://owasp.org/ - Top 10 vulnerabilidades web
- https://ssl-config.mozilla.org/ - Generador config SSL
- https://hstspreload.org/ - HSTS Preload list
- https://letsencrypt.org/ - Certificados SSL gratuitos


#### Cursos y Certificaciones Recomendados

- **Nginx Fundamentals** (Udemy, A Cloud Guru)
- **Docker Certified Associate** (DCA)
- **Certified Kubernetes Administrator** (CKA)
- **AWS Solutions Architect** (si se trabaja con AWS)


#### Comunidad y Soporte

**Foros y Slack**

- r/nginx (Reddit)
- Nginx mailing list (nginx@nginx.org)
- Docker Community Slack
- Stack Overflow (\#nginx, \#docker)

**Blogs Técnicos**

- https://blog.nginx.org/ - Blog oficial
- https://www.nginx.com/blog/ - Nginx Inc.
- https://sysadvent.blogspot.com/ - Artículos DevOps


#### Herramientas de Terceros

**Monitorización**

- Prometheus + Grafana (open source)
- Datadog / New Relic (comercial)
- Elastic Stack (ELK) para logs

**Testing**

- Locust (load testing en Python)
- K6 (load testing moderno)
- Postman / Insomnia (API testing)

**Seguridad**

- Trivy (container scanning)
- OWASP ZAP (penetration testing)
- Snyk (dependency scanning)

---

### Reflexión Final

> **"La arquitectura web moderna no se trata de dominar una herramienta, sino de orquestar un ecosistema."**

A lo largo de esta investigación hemos cubierto:

- Fundamentos HTTP/HTTPS y evolución del protocolo
- Arquitectura event-driven de Nginx
- Dockerización y orquestación con Compose
- Configuración avanzada (proxy inverso, balanceo de carga)
- Seguridad production-grade (TLS, HSTS, rate limiting, escaneo de imágenes)
- Troubleshooting y herramientas de diagnóstico

El valor de un profesional en administración de servidores web está en:

1. **Entender el flujo completo**: Cliente → CDN → Load Balancer → App → Database
2. **Tomar decisiones informadas**: ¿Cuándo cachear? ¿Cuándo escalar horizontal vs vertical?
3. **Automatizar**: Infraestructura como código (Terraform, Ansible)
4. **Medir**: "No puedes mejorar lo que no mides" (Peter Drucker)
5. **Aprender continuamente**: La tecnología evoluciona rápido; la curiosidad debe mantenerse

**El próximo paso**: aplicar este conocimiento en un proyecto real. La teoría sin práctica es estéril; la práctica sin teoría es ciega.
