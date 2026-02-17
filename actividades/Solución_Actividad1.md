## Solucionario Actividad 1: Despliegue de WordPress con Docker

### 1. Estructura de archivos

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

### 2. Configuración de Nginx

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

### 3. Generar certificados SSL autofirmados

```bash
# Crear directorio
mkdir -p nginx/certs

# Generar certificado autofirmado válido por 365 días
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/selfsigned.key \
  -out nginx/certs/selfsigned.crt \
  -subj "/C=ES/ST=Madrid/L=Madrid/O=ILERNA/CN=localhost"
```

### 4. Crear archivo de autenticación HTTP

```bash
# Instalar htpasswd (si no está instalado)
# Ubuntu/Debian: sudo apt-get install apache2-utils
# Windows: usar Docker

# Crear usuario 'admin'
htpasswd -c .htpasswd admin
# Te pedirá la contraseña (por ejemplo: admin123)
```

### 5. Estructura de directorios

```bash
# Crear directorios necesarios
mkdir -p nginx/conf.d nginx/certs nginx/logs
```

### 6. Desplegar el stack

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

### 7. Verificaciones de seguridad

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
