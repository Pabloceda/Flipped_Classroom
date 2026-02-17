# Actividades Prácticas

## ACTIVIDAD 1: Stack WordPress con Nginx, MariaDB y Políticas de Seguridad

### 📝 Enunciado

Debes desplegar un entorno completo de WordPress utilizando Docker Compose, aplicando las técnicas y políticas de seguridad vistas en los módulos. El stack debe incluir:

**Componentes:**
- **Nginx**: como reverse proxy y servidor web
- **WordPress**: aplicación PHP
- **MariaDB**: base de datos
- **PHP-FPM**: procesador PHP

**Requisitos técnicos:**

1. **Dockerización (Módulo 2)**
   - Utilizar imágenes oficiales con versiones específicas (NO usar `:latest`)
   - Configurar redes Docker personalizadas
   - Implementar volúmenes persistentes para datos y configuraciones

2. **Proxy Inverso (Módulo 4)**
   - Nginx debe actuar como reverse proxy hacia WordPress
   - Configurar correctamente las cabeceras proxy (`X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`)
   - Nginx debe servir archivos estáticos directamente (imágenes, CSS, JS)

3. **Seguridad y Hardening (Módulo 5)**
   - Implementar HTTPS con certificados autofirmados
   - Configurar redirección HTTP → HTTPS
   - Añadir header HSTS
   - Ocultar versión de Nginx (`server_tokens off`)
   - Proteger `/wp-admin` con autenticación HTTP básica
   - Implementar rate limiting en login (5 intentos por minuto)
   - Restringir acceso a `wp-config.php` y archivos `.htaccess`

**Estructura esperada:**

```
wordpress-stack/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   ├── conf.d/
│   │   └── wordpress.conf
│   └── certs/
│       ├── selfsigned.crt
│       └── selfsigned.key
├── .htpasswd
└── README.md
```

**Detalles adicionales:**
- El sitio debe ser accesible en `https://localhost`
- La base de datos debe usar un volumen nombrado
- WordPress debe persistir uploads y plugins
- Los logs de Nginx deben estar accesibles desde el host

---

### ✅ Solución

#### 1. Estructura de archivos

**docker-compose.yml**

```yaml
version: '3.8'

services:
  # Base de datos MariaDB
  db:
    image: mariadb:11.2-jammy
    container_name: wordpress_db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword123
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppassword123
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - backend
    # Healthcheck para verificar disponibilidad
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 30s
      timeout: 10s
      retries: 3

  # WordPress con PHP-FPM
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
      WORDPRESS_DB_PASSWORD: wppassword123
      WORDPRESS_TABLE_PREFIX: wp_
    volumes:
      - wordpress_data:/var/www/html
    networks:
      - backend
    # Sin exponer puerto al host, solo interno

  # Nginx como Reverse Proxy
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
      # Configuraciones
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      # Certificados SSL
      - ./nginx/certs:/etc/nginx/certs:ro
      # Autenticación
      - ./.htpasswd:/etc/nginx/.htpasswd:ro
      # WordPress files (solo lectura para servir estáticos)
      - wordpress_data:/var/www/html:ro
      # Logs persistentes
      - ./nginx/logs:/var/log/nginx
    networks:
      - backend
      - frontend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Red interna sin acceso a internet

volumes:
  db_data:
    driver: local
  wordpress_data:
    driver: local
```

#### 2. Configuración de Nginx

**nginx/nginx.conf**

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Seguridad: Ocultar versión de Nginx
    server_tokens off;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css text/xml text/javascript 
               application/xml application/json application/javascript;

    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=wp_login:10m rate=5r/m;
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;

    # Incluir configuraciones de sitios
    include /etc/nginx/conf.d/*.conf;
}
```

**nginx/conf.d/wordpress.conf**

```nginx
# Redirección HTTP a HTTPS
server {
    listen 80;
    server_name localhost;

    # Redirigir todo el tráfico a HTTPS
    return 301 https://$server_name$request_uri;
}

# Servidor HTTPS
server {
    listen 443 ssl http2;
    server_name localhost;

    root /var/www/html;
    index index.php index.html;

    # === CONFIGURACIÓN SSL ===
    ssl_certificate /etc/nginx/certs/selfsigned.crt;
    ssl_certificate_key /etc/nginx/certs/selfsigned.key;
    
    # Protocolos y ciphers seguros
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    
    # SSL Session
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # === HEADERS DE SEGURIDAD ===
    # HSTS: forzar HTTPS por 1 año
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # Prevenir clickjacking
    add_header X-Frame-Options "SAMEORIGIN" always;
    
    # Prevenir MIME sniffing
    add_header X-Content-Type-Options "nosniff" always;
    
    # XSS Protection
    add_header X-XSS-Protection "1; mode=block" always;

    # === LOGGING ===
    access_log /var/log/nginx/wordpress_access.log;
    error_log /var/log/nginx/wordpress_error.log;

    # === ARCHIVOS ESTÁTICOS ===
    # Nginx sirve directamente (sin pasar por PHP)
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
        try_files $uri =404;
    }

    # === PROTECCIÓN DE ARCHIVOS SENSIBLES ===
    # Bloquear acceso a wp-config.php
    location ~* wp-config.php {
        deny all;
    }
    
    # Bloquear archivos .htaccess, .htpasswd
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
    
    # Bloquear acceso a readme.html y license.txt
    location ~* ^/(readme|license)\.(html|txt)$ {
        deny all;
    }

    # === PROTECCIÓN ADMIN CON AUTENTICACIÓN ===
    location /wp-admin/ {
        # Requiere usuario y contraseña
        auth_basic "WordPress Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
        
        # Rate limiting estricto
        limit_req zone=general burst=5 nodelay;
        
        # Proxy a PHP-FPM
        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass wordpress:9000;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            
            # Headers proxy
            fastcgi_param HTTP_X_REAL_IP $remote_addr;
            fastcgi_param HTTP_X_FORWARDED_FOR $proxy_add_x_forwarded_for;
            fastcgi_param HTTP_X_FORWARDED_PROTO $scheme;
        }
    }

    # === PROTECCIÓN LOGIN (Rate Limiting) ===
    location = /wp-login.php {
        # Máximo 5 intentos por minuto
        limit_req zone=wp_login burst=2 nodelay;
        
        include fastcgi_params;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    # === PROCESAMIENTO PHP GENERAL ===
    location ~ \.php$ {
        try_files $uri =404;
        
        include fastcgi_params;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        
        # Headers para que WordPress sepa el esquema original
        fastcgi_param HTTPS on;
        fastcgi_param HTTP_X_FORWARDED_PROTO https;
    }

    # === LOCATION PRINCIPAL ===
    location / {
        # Rate limiting general
        limit_req zone=general burst=10 nodelay;
        
        try_files $uri $uri/ /index.php?$args;
    }

    # Bloquear xmlrpc.php (común en ataques de fuerza bruta)
    location = /xmlrpc.php {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

#### 3. Generar certificados SSL autofirmados

```bash
# Crear directorio
mkdir -p nginx/certs

# Generar certificado autofirmado válido por 365 días
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/selfsigned.key \
  -out nginx/certs/selfsigned.crt \
  -subj "/C=ES/ST=Madrid/L=Madrid/O=ILERNA/CN=localhost"
```

#### 4. Crear archivo de autenticación HTTP

```bash
# Instalar htpasswd (si no está instalado)
# Ubuntu/Debian: sudo apt-get install apache2-utils
# Windows: usar Docker

# Crear usuario 'admin'
htpasswd -c .htpasswd admin
# Te pedirá la contraseña (por ejemplo: admin123)
```

#### 5. Estructura de directorios

```bash
# Crear directorios necesarios
mkdir -p nginx/conf.d nginx/certs nginx/logs
```

#### 6. Desplegar el stack

```bash
# Levantar los servicios
docker-compose up -d

# Verificar servicios activos
docker-compose ps

# Ver logs
docker-compose logs -f nginx

# Acceder a WordPress
# https://localhost
```

#### 7. Verificaciones de seguridad

```bash
# 1. Verificar redirección HTTP → HTTPS
curl -I http://localhost
# Debe retornar: HTTP/1.1 301 Moved Permanently
# Location: https://localhost/

# 2. Verificar header HSTS
curl -Ik https://localhost
# Debe incluir: Strict-Transport-Security: max-age=31536000

# 3. Verificar que oculta versión
curl -I http://localhost
# Server: nginx (sin versión)

# 4. Probar rate limiting en login
for i in {1..10}; do 
  curl -Ik https://localhost/wp-login.php
done
# Después del 5º intento debe retornar: HTTP/1.1 503 Service Temporarily Unavailable

# 5. Verificar autenticación en /wp-admin
curl -Ik https://localhost/wp-admin/
# Debe retornar: HTTP/1.1 401 Unauthorized
# Con header: WWW-Authenticate: Basic realm="WordPress Admin Area"

# 6. Verificar bloqueo de wp-config.php
curl -Ik https://localhost/wp-config.php
# Debe retornar: HTTP/1.1 403 Forbidden
```

---

### 🎯 Criterios de Corrección

| Criterio | Puntos | Descripción |
|:---------|:------:|:------------|
| **Dockerización correcta** | 20 | - Imágenes con versiones específicas (no `:latest`)<br>- Volúmenes persistentes configurados<br>- Redes personalizadas (frontend/backend)<br>- Healthchecks implementados |
| **Proxy inverso funcional** | 15 | - Nginx redirecciona correctamente a WordPress<br>- Headers proxy configurados (`X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`)<br>- Nginx sirve estáticos sin pasar por PHP |
| **HTTPS y certificados** | 15 | - Certificados SSL generados correctamente<br>- HTTPS funcional en puerto 443<br>- Redirección HTTP → HTTPS |
| **Headers de seguridad** | 10 | - HSTS configurado<br>- `server_tokens off`<br>- Headers X-Frame-Options, X-Content-Type-Options |
| **Autenticación HTTP** | 10 | - `/wp-admin` protegido con autenticación básica<br>- Archivo `.htpasswd` correctamente generado |
| **Rate Limiting** | 15 | - Zona de rate limit configurada<br>- Límite aplicado en `/wp-login.php`<br>- Funciona correctamente (verificable con curl) |
| **Protección de archivos** | 10 | - `wp-config.php` bloqueado<br>- Archivos ocultos (`.ht*`) bloqueados<br>- `xmlrpc.php` bloqueado |
| **Documentación** | 5 | - README.md con instrucciones de despliegue<br>- Comandos de verificación incluidos |

**Puntuación total: 100 puntos**

**Niveles:**
- **90-100**: Excelente - Todas las políticas implementadas correctamente
- **75-89**: Notable - Funcional con algunas mejoras menores
- **60-74**: Aprobado - Funciona pero faltan políticas importantes
- **< 60**: Suspenso - No cumple requisitos mínimos

---

## ACTIVIDAD 2: Sistema de Balanceo de Carga con Alta Disponibilidad

### 📝 Enunciado

Debes implementar un sistema de balanceo de carga utilizando Nginx con múltiples instancias de backend, aplicando diferentes algoritmos de balanceo y técnicas de alta disponibilidad vistas en el Módulo 4.

**Componentes:**
- **Nginx**: balanceador de carga (load balancer)
- **3 instancias de backend**: aplicación Node.js simple
- **1 servidor de backup**: se activa solo si todos los demás fallan

**Requisitos técnicos:**

1. **Configuración de Upstream (Módulo 4.5)**
   - Definir un grupo upstream con 3 servidores activos
   - Un servidor backup
   - Configurar pesos diferentes (weight) para simular capacidades diferentes

2. **Algoritmos de Balanceo (Módulo 4.6)**
   - Implementar y documentar 3 configuraciones diferentes:
     - **Configuración A**: Round Robin con pesos
     - **Configuración B**: Least Connections
     - **Configuración C**: IP Hash (sticky sessions)

3. **Health Checks (Módulo 4.7)**
   - Configurar passive health checks (`max_fails`, `fail_timeout`)
   - Implementar `proxy_next_upstream` para reintentos automáticos
   - Crear endpoint `/health` en backends para verificación

4. **Monitorización y Logs**
   - Logs personalizados que muestren qué backend procesó cada petición
   - Formato de log que incluya: IP cliente, backend usado, tiempo de respuesta

**Aplicación Backend (Node.js):**
Cada instancia debe:
- Responder con su nombre/ID de contenedor
- Tener un endpoint `/health` que retorne status 200
- Simular carga variable (opcional: añadir delay aleatorio)

**Pruebas requeridas:**
1. Demostrar que el balanceo funciona (peticiones se distribuyen)
2. Simular caída de un backend (detenerlo) y verificar failover automático
3. Verificar que el servidor backup solo se usa cuando todos fallan
4. Medir distribución de carga según pesos configurados

---

### ✅ Solución

#### 1. Estructura de archivos

```
load-balancer/
├── docker-compose.yml
├── nginx/
│   ├── nginx-roundrobin.conf
│   ├── nginx-leastconn.conf
│   └── nginx-iphash.conf
├── backend/
│   ├── app.js
│   ├── package.json
│   └── Dockerfile
├── test-load.sh
└── README.md
```

#### 2. Aplicación Backend (Node.js)

**backend/package.json**

```json
{
  "name": "backend-demo",
  "version": "1.0.0",
  "main": "app.js",
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

**backend/app.js**

```javascript
const express = require('express');
const os = require('os');

const app = express();
const PORT = process.env.PORT || 3000;
const SERVER_ID = process.env.SERVER_ID || os.hostname();

// Contador de peticiones
let requestCount = 0;

// Middleware de logging
app.use((req, res, next) => {
  requestCount++;
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url} - Request #${requestCount}`);
  next();
});

// Endpoint principal
app.get('/', (req, res) => {
  const response = {
    server: SERVER_ID,
    hostname: os.hostname(),
    timestamp: new Date().toISOString(),
    requestNumber: requestCount,
    uptime: process.uptime(),
    message: `Hello from ${SERVER_ID}!`
  };
  
  res.json(response);
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    server: SERVER_ID,
    uptime: process.uptime()
  });
});

// Endpoint con carga simulada (delay aleatorio)
app.get('/slow', (req, res) => {
  const delay = Math.floor(Math.random() * 3000); // 0-3 segundos
  
  setTimeout(() => {
    res.json({
      server: SERVER_ID,
      delay: delay,
      message: `Processed after ${delay}ms`
    });
  }, delay);
});

// Endpoint para simular error
app.get('/fail', (req, res) => {
  res.status(500).json({
    error: 'Simulated failure',
    server: SERVER_ID
  });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server ${SERVER_ID} listening on port ${PORT}`);
});
```

**backend/Dockerfile**

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY app.js ./

EXPOSE 3000

CMD ["node", "app.js"]
```

#### 3. Docker Compose

**docker-compose.yml**

```yaml
version: '3.8'

services:
  # === BACKEND SERVERS ===
  backend1:
    build: ./backend
    container_name: backend_1
    environment:
      - SERVER_ID=Backend-1
      - PORT=3000
    networks:
      - backend_net
    restart: unless-stopped

  backend2:
    build: ./backend
    container_name: backend_2
    environment:
      - SERVER_ID=Backend-2
      - PORT=3000
    networks:
      - backend_net
    restart: unless-stopped

  backend3:
    build: ./backend
    container_name: backend_3
    environment:
      - SERVER_ID=Backend-3
      - PORT=3000
    networks:
      - backend_net
    restart: unless-stopped

  # Servidor de backup (solo se activa si todos fallan)
  backend_backup:
    build: ./backend
    container_name: backend_backup
    environment:
      - SERVER_ID=Backend-BACKUP
      - PORT=3000
    networks:
      - backend_net
    restart: unless-stopped

  # === NGINX LOAD BALANCER ===
  nginx:
    image: nginx:1.25-alpine3.19
    container_name: load_balancer
    ports:
      - "80:80"
    volumes:
      # Cambiar este archivo para probar diferentes algoritmos
      - ./nginx/nginx-roundrobin.conf:/etc/nginx/nginx.conf:ro
      # Para probar otros:
      # - ./nginx/nginx-leastconn.conf:/etc/nginx/nginx.conf:ro
      # - ./nginx/nginx-iphash.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/logs:/var/log/nginx
    depends_on:
      - backend1
      - backend2
      - backend3
      - backend_backup
    networks:
      - backend_net
    restart: unless-stopped

networks:
  backend_net:
    driver: bridge
```

#### 4. Configuraciones de Nginx

**nginx/nginx-roundrobin.conf** (Round Robin con pesos)

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    # Logging personalizado para ver qué backend responde
    log_format balancer '$remote_addr - [$time_local] "$request" '
                        '$status $body_bytes_sent '
                        'upstream: $upstream_addr '
                        'response_time: $upstream_response_time '
                        'request_time: $request_time';

    access_log /var/log/nginx/access.log balancer;

    # === UPSTREAM: ROUND ROBIN CON PESOS ===
    upstream backend_servers {
        # Round Robin (algoritmo por defecto)
        # Backend 1 recibe el doble de tráfico (simulando mayor capacidad)
        server backend1:3000 weight=2 max_fails=3 fail_timeout=30s;
        server backend2:3000 weight=1 max_fails=3 fail_timeout=30s;
        server backend3:3000 weight=1 max_fails=3 fail_timeout=30s;
        
        # Servidor de backup (solo se usa si todos los demás fallan)
        server backend_backup:3000 backup;
    }

    server {
        listen 80;
        server_name localhost;

        # Ocultar versión de Nginx
        server_tokens off;

        location / {
            proxy_pass http://backend_servers;
            
            # Headers proxy
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
            
            # Reintentar en caso de error
            proxy_next_upstream error timeout http_500 http_502 http_503;
            proxy_next_upstream_tries 3;
            proxy_next_upstream_timeout 10s;
        }

        # Health check endpoint (agregado por Nginx)
        location /nginx-status {
            stub_status on;
            access_log off;
            allow 127.0.0.1;
            deny all;
        }
    }
}
```

**nginx/nginx-leastconn.conf** (Least Connections)

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    log_format balancer '$remote_addr - [$time_local] "$request" '
                        '$status $body_bytes_sent '
                        'upstream: $upstream_addr '
                        'response_time: $upstream_response_time';

    access_log /var/log/nginx/access.log balancer;

    # === UPSTREAM: LEAST CONNECTIONS ===
    upstream backend_servers {
        # Envía al servidor con menos conexiones activas
        least_conn;
        
        server backend1:3000 max_fails=3 fail_timeout=30s;
        server backend2:3000 max_fails=3 fail_timeout=30s;
        server backend3:3000 max_fails=3 fail_timeout=30s;
        server backend_backup:3000 backup;
    }

    server {
        listen 80;
        server_name localhost;
        server_tokens off;

        location / {
            proxy_pass http://backend_servers;
            
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
            
            proxy_next_upstream error timeout http_500 http_502 http_503;
            proxy_next_upstream_tries 3;
        }
    }
}
```

**nginx/nginx-iphash.conf** (IP Hash - Sticky Sessions)

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    log_format balancer '$remote_addr - [$time_local] "$request" '
                        '$status upstream: $upstream_addr '
                        'response_time: $upstream_response_time';

    access_log /var/log/nginx/access.log balancer;

    # === UPSTREAM: IP HASH (STICKY SESSIONS) ===
    upstream backend_servers {
        # Mismo cliente siempre va al mismo servidor
        ip_hash;
        
        server backend1:3000 max_fails=3 fail_timeout=30s;
        server backend2:3000 max_fails=3 fail_timeout=30s;
        server backend3:3000 max_fails=3 fail_timeout=30s;
        server backend_backup:3000 backup;
    }

    server {
        listen 80;
        server_name localhost;
        server_tokens off;

        location / {
            proxy_pass http://backend_servers;
            
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_connect_timeout 5s;
            proxy_read_timeout 10s;
            
            proxy_next_upstream error timeout http_500 http_502 http_503;
        }
    }
}
```

#### 5. Script de pruebas

**test-load.sh**

```bash
#!/bin/bash

echo "=== Test de Balanceo de Carga ==="
echo ""

# Función para hacer peticiones
test_requests() {
    echo "Enviando 20 peticiones..."
    for i in {1..20}; do
        response=$(curl -s http://localhost/ | jq -r '.server')
        echo "Request $i: $response"
        sleep 0.2
    done
}

# Función para ver distribución
show_distribution() {
    echo ""
    echo "=== Distribución de peticiones ==="
    curl -s http://localhost/ -w "\n" | jq -r '.server' | sort | uniq -c
}

# Menú
echo "Elige una opción:"
echo "1. Test básico (20 peticiones)"
echo "2. Test con análisis de distribución"
echo "3. Test de failover (detener backend1)"
echo "4. Ver logs de Nginx"
read -p "Opción: " option

case $option in
    1)
        test_requests
        ;;
    2)
        echo "Enviando 100 peticiones..."
        for i in {1..100}; do
            curl -s http://localhost/ >> /tmp/load_test_results.txt
        done
        echo ""
        echo "Distribución:"
        cat /tmp/load_test_results.txt | jq -r '.server' | sort | uniq -c
        rm /tmp/load_test_results.txt
        ;;
    3)
        echo "Deteniendo backend1..."
        docker stop backend_1
        echo "Enviando peticiones (debería usar backend2 y backend3)..."
        test_requests
        echo ""
        read -p "¿Re-activar backend1? (y/n): " restart
        if [ "$restart" = "y" ]; then
            docker start backend_1
            echo "Backend1 reactivado"
        fi
        ;;
    4)
        echo "=== Últimas 20 líneas del log ==="
        tail -n 20 nginx/logs/access.log
        ;;
    *)
        echo "Opción no válida"
        ;;
esac
```

#### 6. Pasos de despliegue

```bash
# 1. Construir e iniciar servicios
docker-compose up -d --build

# 2. Verificar que todos los contenedores están corriendo
docker-compose ps

# 3. Probar acceso básico
curl http://localhost/

# 4. Hacer el script ejecutable
chmod +x test-load.sh

# 5. Ejecutar pruebas
./test-load.sh

# 6. Ver logs en tiempo real
docker-compose logs -f nginx

# 7. Para probar otro algoritmo:
# - Editar docker-compose.yml y cambiar el volumen de nginx
# - Reiniciar: docker-compose restart nginx
```

#### 7. Verificaciones

**Test 1: Distribución Round Robin con pesos**

```bash
# Enviar 100 peticiones
for i in {1..100}; do 
  curl -s http://localhost/ | jq -r '.server'
done | sort | uniq -c

# Resultado esperado (aproximado):
# 50 Backend-1  (weight=2, recibe ~50%)
# 25 Backend-2  (weight=1, recibe ~25%)
# 25 Backend-3  (weight=1, recibe ~25%)
```

**Test 2: Failover automático**

```bash
# Detener un backend
docker stop backend_1

# Hacer peticiones
for i in {1..10}; do 
  curl -s http://localhost/ | jq -r '.server'
done

# Resultado: Solo veremos Backend-2 y Backend-3
# Backend-1 no aparece
```

**Test 3: Servidor de backup**

```bash
# Detener todos los backends principales
docker stop backend_1 backend_2 backend_3

# Hacer peticiones
curl -s http://localhost/ | jq -r '.server'

# Resultado: Backend-BACKUP
# El backup solo se activa cuando todos los demás fallan
```

**Test 4: Health checks pasivos**

```bash
# Ver logs de Nginx para verificar reintentos
docker-compose logs nginx | grep "upstream"

# Debe mostrar reintentos automáticos cuando un backend falla
```

---

### 🎯 Criterios de Corrección

| Criterio | Puntos | Descripción |
|:---------|:------:|:------------|
| **Aplicación Backend funcional** | 15 | - Aplicación Node.js responde correctamente<br>- Endpoint `/health` implementado<br>- Muestra ID del servidor en respuesta<br>- Dockerfile correcto |
| **Upstream configurado** | 15 | - Grupo upstream con 3+ servidores<br>- Servidor backup configurado<br>- Pesos (weight) correctamente asignados |
| **Algoritmos de balanceo** | 25 | - **Round Robin** con pesos funcional (10pts)<br>- **Least Connections** funcional (8pts)<br>- **IP Hash** funcional (7pts) |
| **Health Checks** | 15 | - `max_fails` y `fail_timeout` configurados<br>- `proxy_next_upstream` implementado<br>- Failover funciona correctamente |
| **Logs personalizados** | 10 | - Formato de log muestra backend usado<br>- Muestra tiempos de respuesta<br>- Logs accesibles desde host |
| **Pruebas y verificación** | 15 | - Script de pruebas funcional<br>- Demuestra distribución de carga<br>- Demuestra failover<br>- Verifica servidor backup |
| **Documentación** | 5 | - README con instrucciones claras<br>- Explicación de cada algoritmo<br>- Comandos de verificación |

**Puntuación total: 100 puntos**

**Comparativa de algoritmos esperada:**

| Algoritmo | Escenario ideal | Sticky Sessions |
|:----------|:----------------|:----------------|
| Round Robin | Backends homogéneos, requests similares | ❌ No |
| Weighted RR | Backends con diferente capacidad | ❌ No |
| Least Conn | Requests de duración variable | ❌ No |
| IP Hash | Aplicaciones con sesiones locales | ✅ Sí |

---

## 📚 Recursos Adicionales
Aquí puedes encontrar recursos adicionales que te pueden ser de utilidad para realizar las actividades:
### Comandos Útiles de Docker

```bash
# Ver logs de un servicio específico
docker-compose logs -f [servicio]

# Reiniciar un servicio
docker-compose restart [servicio]

# Ejecutar comando en contenedor
docker-compose exec [servicio] sh

# Ver recursos utilizados
docker stats

# Limpiar todo
docker-compose down -v
```

### Debugging Común

**Problema: 502 Bad Gateway**
```bash
# Verificar que el backend está corriendo
docker-compose ps

# Ver logs del backend
docker-compose logs backend

# Verificar conectividad
docker-compose exec nginx ping backend1
```

**Problema: Rate limiting no funciona**
```bash
# Verificar que la zona está definida en http {}
# Verificar que limit_req está en location {}
# Ver logs para confirmar 503
docker-compose logs nginx | grep "limiting requests"
```
