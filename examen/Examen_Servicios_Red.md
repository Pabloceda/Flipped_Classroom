# Examen: Servicios de Red - Nginx y Docker

**Módulo**: Servicios de Red e Internet  
**Tema**: Despliegue de Servidores Web con Nginx y Docker  
**Duración**: 90 minutos  
**Puntuación total**: 100 puntos

---

## Instrucciones

- Lee cuidadosamente cada pregunta antes de responder
- En las preguntas tipo test, marca solo UNA opción correcta
- En las preguntas cortas, responde de forma concisa pero completa
- En los casos prácticos, muestra tu razonamiento y código cuando sea necesario
- El examen consta de 3 secciones con diferente peso en la calificación

**Distribución de puntos:**
- Sección A (Tipo Test): 40 puntos
- Sección B (Preguntas Cortas): 30 puntos  
- Sección C (Casos Prácticos): 30 puntos

---

## SECCIÓN A: Preguntas Tipo Test (40 puntos)

*Cada pregunta correcta vale 2 puntos. Respuestas incorrectas no penalizan.*

### A1. ¿Cuál es la principal diferencia arquitectural entre Apache MPM Prefork y Nginx?

a) Apache usa threads, Nginx usa procesos  
b) Apache crea un proceso por conexión, Nginx usa arquitectura event-driven asíncrona  
c) Apache es más rápido que Nginx en concurrencia alta  
d) No hay diferencias significativas

**Respuesta correcta**: b) Apache crea un proceso por conexión, Nginx usa arquitectura event-driven asíncrona

**Justificación**: Apache MPM Prefork crea un proceso completo (pesado) por cada conexión, lo cual consume mucha memoria con alta concurrencia. Nginx usa un modelo event-driven donde pocos workers manejan miles de conexiones simultáneas de forma asíncrona mediante event loops, resultando en menor uso de memoria.

---

### A2. En Docker, ¿qué diferencia hay entre una imagen y un contenedor?

a) Son exactamente lo mismo  
b) La imagen es una plantilla inmutable, el contenedor es una instancia ejecutándose  
c) El contenedor es más grande que la imagen  
d) La imagen solo funciona en Linux

**Respuesta correcta**: b) La imagen es una plantilla inmutable, el contenedor es una instancia ejecutándose

**Justificación**: La relación es similar a clase-objeto en programación orientada a objetos. Una imagen Docker es una plantilla de solo lectura con capas inmutables. Un contenedor es una instancia en ejecución de esa imagen, con una capa de escritura temporal. De una misma imagen puedes crear múltiples contenedores.

---

### A3. ¿Qué hace la directiva `EXPOSE 80` en un Dockerfile?

a) Publica el puerto 80 en el host automáticamente  
b) Solo documenta que el contenedor usa internamente el puerto 80  
c) Abre el puerto 80 en el firewall del sistema  
d) Mapea el puerto 80 al 8080 del host

**Respuesta correcta**: b) Solo documenta que el contenedor usa internamente el puerto 80

**Justificación**: `EXPOSE` es puramente documentación para otros desarrolladores. NO publica ni abre puertos. Para publicar un puerto al host se debe usar `-p 80:80` en `docker run` o la sección `ports:` en docker-compose.yml.

---

### A4. En Nginx, si configuras `location /api/ { proxy_pass http://backend; }`, ¿qué URI recibe el backend cuando llegas a `/api/users`?

a) `/users`  
b) `/api/users`  
c) `/backend/api/users`  
d) Error 404

**Respuesta correcta**: b) `/api/users`

**Justificación**: Sin trailing slash ni URI en `proxy_pass`, Nginx preserva la URI completa original. Si fuera `proxy_pass http://backend/;` (con trailing slash), reemplazaría `/api/` por `/`, enviando solo `/users` al backend.

---

### A5. ¿Qué tipo de volumen Docker es completamente gestionado por Docker?

a) Bind mount  
b) Named volume  
c) tmpfs mount  
d) Host volume

**Respuesta correcta**: b) Named volume

**Justificación**: Los named volumes (creados con `docker volume create` o definidos en docker-compose) son gestionados por Docker en `/var/lib/docker/volumes/`. Los bind mounts mapean directorios específicos del host que el usuario gestiona manualmente.

---

### A6. ¿Qué algoritmo de balanceo permite "sticky sessions" (mismo cliente siempre al mismo backend)?

a) round_robin  
b) least_conn  
c) ip_hash  
d) random

**Respuesta correcta**: c) ip_hash

**Justificación**: `ip_hash` calcula un hash de la IP del cliente y lo envía siempre al mismo servidor backend (mientras esté disponible). Es esencial para aplicaciones que mantienen estado de sesión en memoria del servidor.

---

### A7. ¿Qué código de estado HTTP indica redirección permanente?

a) 200 OK  
b) 301 Moved Permanently  
c) 302 Found  
d) 404 Not Found

**Respuesta correcta**: b) 301 Moved Permanently

**Justificación**: 301 indica que el recurso se ha movido permanentemente a otra URL. Los navegadores y motores de búsqueda actualizan sus referencias. Se usa comúnmente para redirecciones HTTP→HTTPS. 302 es redirección temporal.

---

### A8. ¿Por qué NO se debe usar `:latest` en imágenes Docker de producción?

a) Es muy lenta  
b) Puede cambiar sin previo aviso rompiendo la aplicación  
c) No existe como etiqueta  
d) Consume más recursos

**Respuesta correcta**: b) Puede cambiar sin previo aviso rompiendo la aplicación

**Justificación**: `:latest` apunta a la versión más reciente publicada, que puede incluir breaking changes. Viola el principio de reproducibilidad. En producción se debe pinar versión exacta: `nginx:1.25.3-alpine3.19`.

---

### A9. ¿Qué directiva Nginx oculta el número de versión en las respuestas HTTP?

a) `hide_version on;`  
b) `server_tokens off;`  
c) `version_hide true;`  
d) `expose_version false;`

**Respuesta correcta**: b) `server_tokens off;`

**Justificación**: `server_tokens off;` en contexto `http {}` evita que Nginx incluya su versión en el header `Server:`. Es una medida básica de security by obscurity para dificultar ataques dirigidos a vulnerabilidades de versiones específicas.

---

### A10. ¿Qué significa HSTS (HTTP Strict Transport Security)?

a) Sistema de autenticación HTTP  
b) Mecanismo que fuerza al navegador a usar solo HTTPS  
c) Protocolo de cifrado  
d) Header para cachear contenido

**Respuesta correcta**: b) Mecanismo que fuerza al navegador a usar solo HTTPS

**Justificación**: HSTS mediante el header `Strict-Transport-Security` indica al navegador que recuerde usar solo HTTPS con ese dominio. Después de la primera visita, el navegador convierte automáticamente cualquier intento HTTP a HTTPS antes de enviar el request, previniendo downgrade attacks.

---

### A11. ¿Cuál es el propósito de un multi-stage build en Docker?

a) Ejecutar múltiples contenedores  
b) Reducir tamaño de imagen separando build de runtime  
c) Crear backups automáticos  
d) Mejorar velocidad de ejecución

**Respuesta correcta**: b) Reducir tamaño de imagen separando build de runtime

**Justificación**: Multi-stage builds usan múltiples `FROM` en un Dockerfile. Se compila en una imagen pesada con todas las herramientas (gcc, npm, etc.) y solo se copia el artefacto final a una imagen pequeña de runtime, reduciendo drásticamente el tamaño final.

---

### A12. En un upstream Nginx, ¿qué hace el parámetro `backup`?

a) Hace backup de los datos  
b) El servidor solo recibe tráfico si todos los demás fallan  
c) Duplica las peticiones  
d) Cachea las respuestas

**Respuesta correcta**: b) El servidor solo recibe tráfico si todos los demás fallan

**Justificación**: Un servidor marcado como `backup` en el bloque upstream es un failover de último recurso. Solo se activa cuando todos los servidores activos (no-backup) están marcados como down por health checks.

---

### A13. ¿Qué header proxy se usa para pasar la IP real del cliente al backend?

a) `X-Client-IP`  
b) `X-Real-IP`  
c) `X-Remote-Address`  
d) `Client-Origin-IP`

**Respuesta correcta**: b) `X-Real-IP`

**Justificación**: `proxy_set_header X-Real-IP $remote_addr;` es el estándar para pasar la IP original del cliente. Sin esto, el backend solo vería la IP del proxy. También existe `X-Forwarded-For` que mantiene toda la cadena de proxies.

---

### A14. ¿En qué contexto de Nginx se define un bloque `upstream`?

a) `server {}`  
b) `location {}`  
c) `http {}`  
d) `events {}`

**Respuesta correcta**: c) `http {}`

**Justificación**: Los bloques `upstream` se definen en el contexto `http {}`, fuera de server blocks, para poder ser referenciados por múltiples server blocks diferentes.

---

### A15. ¿Qué TLS/SSL versión NO debería usarse por vulnerabilidades conocidas?

a) TLSv1.3  
b) TLSv1.2  
c) TLSv1.0  
d) Todas son seguras

**Respuesta correcta**: c) TLSv1.0

**Justificación**: TLSv1.0 y TLSv1.1 están deprecados por vulnerabilidades como POODLE y BEAST. Solo TLSv1.2 y TLSv1.3 deben usarse en producción: `ssl_protocols TLSv1.2 TLSv1.3;`

---

### A16. ¿Qué network driver de Docker permite comunicación entre contenedores en diferentes hosts físicos?

a) bridge  
b) host  
c) overlay  
d) none

**Respuesta correcta**: c) overlay

**Justificación**: El driver `overlay` crea redes multi-host mediante túneles VXLAN encapsulados en UDP. Se usa en clusters Docker Swarm o Kubernetes. El driver `bridge` solo funciona en un mismo host.

---

### A17. ¿Para qué se usa `try_files $uri $uri/ =404;` en Nginx?

a) Para hacer backup de archivos  
b) Para intentar servir archivo, luego directorio, sino 404  
c) Para cachear archivos automáticamente  
d) Para comprimir archivos antes de servir

**Respuesta correcta**: b) Para intentar servir archivo, luego directorio, sino 404

**Justificación**: `try_files` intenta servir opciones en orden: primero busca el archivo exacto, luego intenta como directorio (añadiendo `/`), y si nada coincide retorna 404. Es común en SPAs para manejar routing client-side.

---

### A18. ¿Qué hace `limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;`?

a) Limita tamaño de requests a 10MB  
b) Crea zona de rate limiting: 5 peticiones por minuto por IP  
c) Restringe acceso a 5 IPs simultáneas  
d) Cachea por 5 minutos

**Respuesta correcta**: b) Crea zona de rate limiting: 5 peticiones por minuto por IP

**Justificación**: Define una zona de memoria compartida (10MB) para rate limiting basado en IP cliente, permitiendo máximo 5 requests por minuto. Se aplica luego con `limit_req zone=login;` en location blocks específicos.

---

### A19. En Docker Compose, ¿qué hace `depends_on`?

a) Espera a que el servicio esté completamente "healthy"  
b) Define solo el orden de inicio de contenedores  
c) Crea dependencias de red automáticamente  
d) Comparte volúmenes entre servicios

**Respuesta correcta**: b) Define solo el orden de inicio de contenedores

**Justificación**: `depends_on` controla el orden de inicio pero NO espera a que el servicio esté listo (solo que el contenedor arranque). Para esperar a "healthy" se debe usar `depends_on: service: condition: service_healthy`.

---

### A20. ¿Cuál es el valor recomendado mínimo para `max-age` en HSTS para producción?

a) 300 segundos (5 minutos)  
b) 3600 segundos (1 hora)  
c) 86400 segundos (1 día)  
d) 31536000 segundos (1 año)

**Respuesta correcta**: d) 31536000 segundos (1 año)

**Justificación**: Para poder incluirse en la HSTS Preload List de navegadores (hardcoded, protección desde primera visita) se requiere `max-age >= 31536000` (1 año). Esto garantiza protección robusta contra downgrade attacks.

---

## SECCIÓN B: Preguntas Cortas (30 puntos)

*Responde de forma concisa pero completa. Cada pregunta vale 3 puntos.*

### B1. Explica qué es un reverse proxy y menciona 3 ventajas de usar Nginx como reverse proxy en una arquitectura web.

**Respuesta esperada**:

Un reverse proxy es un servidor que se sitúa entre los clientes (navegadores) y los servidores backend, interceptando peticiones y distribuyéndolas internamente. Es transparente para el cliente (no sabe que existe).

**Ventajas de Nginx como reverse proxy:**
1. **SSL Termination**: Nginx maneja el cifrado TLS, los backends trabajan en HTTP (ahorro de CPU)
2. **Load Balancing**: Distribuye carga entre múltiples backends con diferentes algoritmos
3. **Caching**: Cachea respuestas para reducir carga en backends y mejorar latencia
4. **Compresión**: Gzip/Brotli centralizado, backends ahorran CPU
5. **Seguridad**: Oculta IPs y versiones de software interno, WAF, rate limiting

*(Mínimo 3 ventajas bien explicadas para puntaje completo)*

---

### B2. Describe la diferencia entre `CMD` y `ENTRYPOINT` en un Dockerfile. Proporciona un ejemplo de cuándo usar cada uno.

**Respuesta esperada**:

**CMD**: Define el comando/argumentos por defecto que se ejecutan cuando se inicia el contenedor. Se sobrescribe fácilmente con `docker run imagen otro_comando`.

**ENTRYPOINT**: Define el ejecutable principal del contenedor. Es más fijo, requiere `--entrypoint` para cambiarlo. Los argumentos de `CMD` o de `docker run` se pasan como parámetros al ENTRYPOINT.

**Ejemplo de uso:**

```dockerfile
# Usar CMD cuando el contenedor puede ejecutar diferentes comandos
CMD ["nginx", "-g", "daemon off;"]
# docker run imagen nginx -t  → Sobrescribe fácilmente

# Usar ENTRYPOINT cuando hay un ejecutable fijo
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
# docker run imagen -t  → Ejecuta "nginx -t" (test config)
```

**Cuándo usar cada uno:**
- CMD: Contenedores multi-propósito (ej: Ubuntu con bash por defecto)
- ENTRYPOINT: Contenedores tipo herramienta (ej: contenedor que siempre ejecuta un script)

---

### B3. ¿Qué diferencia hay entre un bind mount y un named volume en Docker? ¿Cuándo usarías cada uno?

**Respuesta esperada**:

**Bind Mount:**
- Mapea un directorio/archivo específico del host a un path del contenedor
- Path absoluto del host: `-v /home/user/app:/app`
- El usuario gestiona manualmente el contenido

**Named Volume:**
- Gestionado completamente por Docker en `/var/lib/docker/volumes/`
- Se crea con nombre lógico: `docker volume create db_data`
- Docker maneja ubicación, permisos, backup/restore

**Cuándo usar:**

**Bind Mount:**
- Desarrollo local (cambios en código se reflejan inmediatamente)
- Configuraciones específicas del host
- Logs que se quieren en ubicación específica

**Named Volume:**
- Datos de producción (bases de datos)
- Persistencia entre recreaciones de contenedores
- Portabilidad (funciona igual en Windows/Mac/Linux)
- Backups gestionados por Docker

---

### B4. Explica qué es el "problema C10K" y cómo la arquitectura de Nginx lo resuelve mejor que Apache MPM Prefork.

**Respuesta esperada**:

**Problema C10K (10,000 connections):**
Desafío de manejar 10,000 conexiones concurrentes simultáneas en un servidor. Los servidores tradicionales con modelo proceso-por-conexión o thread-por-conexión colapsan por:
- Uso excesivo de memoria (cada proceso/thread consume MB)
- Overhead de context switching entre miles de procesos
- Límites del sistema operativo

**Solución de Nginx:**

Arquitectura **Event-Driven asíncrona**:
- Pocos worker processes (generalmente 1 por CPU core)
- Cada worker usa un **event loop** que maneja miles de conexiones
- Operaciones I/O asíncronas (non-blocking)
- Una conexión solo consume memoria cuando hay actividad

**Apache MPM Prefork:**
- 1 proceso completo por conexión (bloqueante)
- Con 10,000 conexiones = 10,000 procesos
- Consumo de memoria insostenible

**Resultado**: Nginx puede manejar 10,000+ conexiones con ~10-20 MB de memoria; Apache necesitaría GBs.

---

### B5. ¿Qué son las "sticky sessions" en balanceo de carga? ¿Qué problema resuelven y qué riesgo introducen?

**Respuesta esperada**:

**Sticky Sessions (Session Affinity):**
Mecanismo que garantiza que un mismo cliente siempre se conecte al mismo servidor backend durante toda su sesión.

**Problema que resuelven:**
Aplicaciones que mantienen estado de sesión en memoria del servidor (ej: carrito de compra, sesiones PHP sin redis). Sin sticky sessions:
1. Usuario añade producto → va a backend1 (guarda en memoria)
2. Siguiente petición → va a backend2 (carrito vacío)

**Implementación en Nginx:**
`ip_hash` - hash de IP del cliente determina el backend

**Riesgos/Desventajas:**
1. **Distribución desigual**: Clientes detrás de mismo NAT van todos al mismo backend
2. **Loss de sesión si backend cae**: Si backend1 se cae, usuarios pierden sesión
3. **Escalado complicado**: No se puede quitar backends sin perder sesiones
4. **No funciona con CDN/proxies**: IP del proxy, no del cliente real

**Solución mejor**: Sesiones compartidas (Redis, base de datos) elimina necesidad de sticky sessions.

---

### B6. Explica qué es HSTS (HTTP Strict Transport Security) y qué ataque previene específicamente.

**Respuesta esperada**:

**HSTS (HTTP Strict Transport Security):**
Mecanismo de seguridad que fuerza a navegadores a usar solo HTTPS con un dominio durante un periodo determinado.

**Header:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**Funcionamiento:**
1. Primera visita HTTPS → servidor envía header HSTS
2. Navegador guarda: "usar solo HTTPS con este dominio por 1 año"
3. Visitas posteriores → navegador intercepta antes de enviar request HTTP y lo convierte automáticamente a HTTPS

**Ataque que previene:**

**SSL Stripping / Downgrade Attack:**
1. Usuario escribe `example.com` (sin https://)
2. Primera petición va por HTTP
3. Atacante MITM intercepta
4. Atacante mantiene conexión HTTP con víctima, HTTPS con servidor
5. Víctima cree estar en HTTP normal, atacante ve todo en claro

**Con HSTS:**
- Después de primera visita, navegador NUNCA permite HTTP
- Si certificado SSL es inválido, no permite continuar (previene MITM)
- Con HSTS Preload List, protección desde primera visita

---

### B7. Describe los 3 algoritmos principales de balanceo de carga en Nginx y da un caso de uso ideal para cada uno.

**Respuesta esperada**:

**1. Round Robin (default):**
- **Funcionamiento**: Distribuye peticiones secuencialmente, rotando entre backends
- **Variante con pesos**: `weight=3` recibe triple de tráfico
- **Caso de uso**: Backends homogéneos con requests de duración similar
- **Ventaja**: Simple, distribución equitativa
- **Desventaja**: No considera carga real de cada backend

**2. Least Connections (`least_conn;`):**
- **Funcionamiento**: Envía nueva petición al backend con menos conexiones activas
- **Caso de uso**: Requests de duración muy variable (algunos rápidos, otros lentos)
- **Ventaja**: Balance dinámico según carga real
- **Example**: API con endpoints mixtos (GET /status rápido, POST /report lento)

**3. IP Hash (`ip_hash;`):**
- **Funcionamiento**: Hash de IP cliente determina backend (mismo cliente → mismo servidor)
- **Caso de uso**: Aplicaciones con sesiones en memoria local (no compartida)
- **Ventaja**: Sticky sessions sin configuración adicional
- **Desventaja**: Distribución desigual si muchos clientes detrás de NAT

**Elección:**
- Backends iguales + requests simples → Round Robin
- Requests complejos/variables → Least Conn
- Sesiones locales → IP Hash (o mejor: migrar a sesiones compartidas)

---

### B8. ¿Qué es rate limiting en Nginx y cómo protege contra ataques de fuerza bruta?

**Respuesta esperada**:

**Rate Limiting:**
Mecanismo que limita el número de peticiones que un cliente puede hacer en un periodo de tiempo.

**Configuración ejemplo:**
```nginx
# Definir zona (en http {})
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

# Aplicar en location
location /wp-login.php {
    limit_req zone=login burst=2 nodelay;
}
```

**Protección contra fuerza bruta:**

**Sin rate limiting:**
- Atacante puede intentar 1000s de contraseñas/segundo
- Login: admin/pass1, admin/pass2, admin/pass3...

**Con rate limiting (5 req/min):**
- Solo 5 intentos por minuto permitidos
- Intento 6+ → 503 Service Unavailable
- Para probar 1000 contraseñas: 200 minutos (3+ horas)

**Parámetros clave:**
- `rate=5r/m`: Tasa sostenida (5 por minuto)
- `burst=2`: Permite 2 peticiones extras rápidas (tolerancia)
- `nodelay`: No demora las peticiones dentro del burst

**Protección adicional:**
- DDoS de capa 7 (application layer)
- Scraping abusivo
- Consumo excesivo de APIs

---

### B9. Explica qué headers de seguridad HTTP deberían configurarse en Nginx y qué ataque previene cada uno.

**Respuesta esperada**:

**1. X-Frame-Options:**
```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```
- **Previene**: Clickjacking (embedding malicioso en iframes)
- **Valores**: DENY (no iframes), SAMEORIGIN (solo mismo dominio)

**2. X-Content-Type-Options:**
```nginx
add_header X-Content-Type-Options "nosniff" always;
```
- **Previene**: MIME sniffing attacks
- **Efecto**: Navegador respeta Content-Type del servidor, no lo adivina

**3. X-XSS-Protection:**
```nginx
add_header X-XSS-Protection "1; mode=block" always;
```
- **Previene**: Cross-Site Scripting (XSS) reflejado (en navegadores antiguos)
- **Nota**: Deprecado en navegadores modernos (usar CSP)

**4. Strict-Transport-Security (HSTS):**
```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```
- **Previene**: SSL stripping, downgrade attacks
- **Efecto**: Fuerza HTTPS por 1 año

**5. Content-Security-Policy:**
```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self' https://trusted.com" always;
```
- **Previene**: XSS, injection de scripts maliciosos
- **Efecto**: Define fuentes permitidas de recursos

**6. Referrer-Policy:**
```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```
- **Previene**: Leakage de información sensible en referrer

---

### B10. ¿Qué es un health check pasivo en Nginx y cómo difiere de un health check activo?

**Respuesta esperada**:

**Health Check Pasivo (Nginx Open Source):**
- Nginx observa las respuestas de tráfico REAL de usuarios
- Si backend falla N veces (`max_fails`) → marcado como unhealthy
- Permanece down durante `fail_timeout`
- Después se reintenta con 1 petición de prueba

**Configuración:**
```nginx
upstream backend {
    server backend1:3000 max_fails=3 fail_timeout=30s;
    server backend2:3000 max_fails=3 fail_timeout=30s;
}
```

**Problema**: Primera petición real a backend caído falla (mala UX)

**Health Check Activo (Solo NGINX Plus):**
- Nginx envía peticiones de prueba periódicas (`health_check interval=5s;`)
- NO depende de tráfico real
- Detecta fallos ANTES de que usuarios se vean afectados

**Configuración (NGINX Plus):**
```nginx
location / {
    proxy_pass http://backend;
    health_check interval=5s uri=/health;
}
```

**Comparación:**

| Aspecto | Pasivo | Activo |
|---------|--------|--------|
| Disponibilidad | Open Source | Solo NGINX Plus (pago) |
| Detección | Después de fallo real | Antes de afectar usuarios |
| Overhead | Bajo (usa tráfico existente) | Mayor (peticiones extra) |
| Precisión | Reactivo | Proactivo |

---

## SECCIÓN C: Casos Prácticos (30 puntos)

*Analiza los siguientes escenarios y proporciona soluciones técnicas detalladas.*

### C1. Caso: Redirección HTTP a HTTPS (6 puntos)

Tienes un sitio web `example.com` que funciona en HTTP. Necesitas:
1. Forzar HTTPS en todas las conexiones
2. Implementar HSTS con política de 1 año
3. Asegurar que los subdominios también usen HTTPS

**Proporciona la configuración completa de Nginx que implementa estos requisitos.**

**Solución esperada:**

```nginx
# Server block HTTP - Redirección a HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    
    # Redirección permanente a HTTPS
    return 301 https://$server_name$request_uri;
}

# Server block HTTPS
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    
    # Certificados SSL
    ssl_certificate /etc/nginx/certs/example.com.crt;
    ssl_certificate_key /etc/nginx/certs/example.com.key;
    
    # Protocolos seguros
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    
    # HSTS con 1 año + subdominios
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # Headers de seguridad adicionales
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    
    root /var/www/html;
    index index.html;
}
```

**Criterios de corrección:**
- Redirección 301 correcta (2 pts)
- HSTS con max-age=31536000 (2 pts)
- includeSubDomains en HSTS (1 pt)
- Configuración SSL válida (1 pt)

---

### C2. Caso: Docker Compose con Persistencia (8 puntos)

Diseña un `docker-compose.yml` para un blog WordPress que cumpla:

**Requisitos:**
1. WordPress (PHP-FPM)
2. MariaDB como base de datos
3. Nginx como reverse proxy
4. Los datos de la BD deben persistir cuando se elimine el contenedor
5. WordPress debe poder subir archivos (persistir uploads)
6. Usar redes separadas: frontend (Nginx-WordPress) y backend (WordPress-DB)

**Proporciona el archivo docker-compose.yml completo.**

**Solución esperada:**

```yaml
version: '3.8'

services:
  # Base de datos
  db:
    image: mariadb:11.2-jammy
    container_name: wordpress_db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql  # Persistencia BD
    networks:
      - backend
    healthcheck:
      test: ["CMD", "health check.sh", "--connect"]
      interval: 30s
      timeout: 10s
      retries: 3

  # WordPress
  wordpress:
    image: wordpress:6.4-php8.2-fpm-alpine
    container_name: wordpress_app
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
    volumes:
      - wordpress_data:/var/www/html  # Persistencia WordPress + uploads
    networks:
      - frontend
      - backend

  # Nginx
  nginx:
    image: nginx:1.25-alpine3.19
    container_name: wordpress_nginx
    restart: unless-stopped
    depends_on:
      - wordpress
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - wordpress_data:/var/www/html:ro  # Acceso a archivos para servir estáticos
    networks:
      - frontend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Sin acceso a internet

volumes:
  db_data:
    driver: local
  wordpress_data:
    driver: local
```

**Criterios de corrección:**
- Named volumes para BD y WordPress (2 pts)
- Redes separadas frontend/backend (2 pts)
- Dependencias y healthcheck correcto (2 pts)
- Configuración válida de servicios (2 pts)

---

### C3. Caso: Balanceo de Carga con Failover (8 puntos)

Tienes una API REST con 3 servidores backend:
- `api1.internal:3000` (servidor potente, doble capacidad)
- `api2.internal:3000` (servidor estándar)
- `api3.internal:3000` (servidor estándar)
- `api-backup.internal:3000` (solo usar si todos fallan)

**Requisitos:**
1. Distribuir carga proporcionalmente a capacidad (api1 doble tráfico)
2. Si un servidor falla 3 veces, marcarlo como down por 30 segundos
3. Usar servidor backup solo cuando todos fallen
4. Reintentar peticiones fallidas en otro servidor automáticamente

**Proporciona la configuración Nginx del upstream y location.**

**Solución esperada:**

```nginx
http {
    # Upstream con weighted round robin y health checks
    upstream api_backend {
        # Servidor potente: peso 2 (recibe doble tráfico)
        server api1.internal:3000 weight=2 max_fails=3 fail_timeout=30s;
        
        # Servidores estándar: peso 1
        server api2.internal:3000 weight=1 max_fails=3 fail_timeout=30s;
        server api3.internal:3000 weight=1 max_fails=3 fail_timeout=30s;
        
        # Servidor de backup (solo si todos los demás fallan)
        server api-backup.internal:3000 backup;
    }

    server {
        listen 80;
        server_name api.example.com;

        location /api/ {
            proxy_pass http://api_backend;
            
            # Headers proxy
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
            
            # Reintentos automáticos
            proxy_next_upstream error timeout http_500 http_502 http_503;
            proxy_next_upstream_tries 3;
            proxy_next_upstream_timeout 10s;
        }
    }
}
```

**Criterios de corrección:**
- Pesos configurados correctamente (2 pts)
- max_fails y fail_timeout (2 pts)
- Servidor backup implementado (2 pts)
- proxy_next_upstream correcto (2 pts)

---

### C4. Caso: Protección de API con Rate Limiting (8 puntos)

Tienes una API pública en `/api/` que sufre ataques de fuerza bruta en `/api/login` y scraping excesivo en `/api/users`.

**Requisitos:**
1. Limitar `/api/login` a 3 intentos por minuto por IP
2. Limitar `/api/users` a 10 peticiones por segundo por IP
3. El resto de endpoints: 100 peticiones por minuto
4. Mostrar mensaje personalizado cuando se exceda límite

**Proporciona la configuración completa de Nginx.**

**Solución esperada:**

```nginx
http {
    # Zonas de rate limiting
    limit_req_zone $binary_remote_addr zone=api_login:10m rate=3r/m;
    limit_req_zone $binary_remote_addr zone=api_users:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api_general:10m rate=100r/m;
    
    # Status code personalizado para rate limit
    limit_req_status 429;

    server {
        listen 80;
        server_name api.example.com;

        # Login: 3 peticiones por minuto
        location = /api/login {
            limit_req zone=api_login burst=1 nodelay;
            limit_req_log_level warn;
            
            # Mensaje personalizado al exceder límite
            error_page 429 = @rate_limit_error;
            
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Listado usuarios: 10 peticiones por segundo
        location /api/users {
            limit_req zone=api_users burst=20 nodelay;
            
            error_page 429 = @rate_limit_error;
            
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Resto de API: 100 peticiones por minuto
        location /api/ {
            limit_req zone=api_general burst=10 nodelay;
            
            error_page 429 = @rate_limit_error;
            
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Mensaje personalizado para rate limit
        location @rate_limit_error {
            default_type application/json;
            return 429 '{"error": "Too many requests. Please try again later.", "retry_after": 60}';
        }
    }
}
```

**Criterios de corrección:**
- Tres zonas de rate limiting configuradas (3 pts)
- Límites aplicados correctamente a locations (2 pts)
- Burst configurado apropiadamente (1 pt)
- Mensaje personalizado implementado (2 pts)

---

## Criterios Generales de Corrección

### Sección A (Tipo Test)
- **40 puntos total**: 20 preguntas × 2 puntos
- Respuesta correcta = 2 puntos
- Respuesta incorrecta = 0 puntos (no penaliza)

### Sección B (Preguntas Cortas)
- **30 puntos total**: 10 preguntas × 3 puntos
- 3 puntos: Respuesta completa, correcta y bien explicada
- 2 puntos: Respuesta correcta pero incompleta o falta claridad
- 1 punto: Respuesta parcialmente correcta con errores menores
- 0 puntos: Respuesta incorrecta o no responde

### Sección C (Casos Prácticos)
- **30 puntos total**: 4 casos prácticos
- Se evalúa por criterios específicos indicados en cada caso
- Sintaxis correcta de configuración
- Implementación completa de requisitos
- Buenas prácticas aplicadas

### Escala de Calificación Final

| Puntos | Nota | Calificación |
|:------:|:----:|:-------------|
| 90-100 | 9-10 | Sobresaliente |
| 75-89 | 7-8.9 | Notable |
| 60-74 | 6-6.9 | Aprobado |
| 50-59 | 5-5.9 | Suficiente |
| < 50 | < 5 | Suspenso |

---

## Material Permitido

- No se permite consulta de apuntes ni Internet
- Se permite una hoja A4 de fórmulas/comandos básicos (manuscrita)

## Recomendaciones para el Examen

1. Lee todas las preguntas antes de empezar
2. Gestiona bien el tiempo (90 minutos = ~2.5 min por pregunta tipo test, ~4.5 min por corta, ~7.5 min por práctica)
3. En casos prácticos, muestra tu razonamiento aunque no recuerdes sintaxis exacta
4. Revisa tus respuestas si te sobra tiempo
5. Las explicaciones en preguntas cortas valen más que memorización

¡Buena suerte!
