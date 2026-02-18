## Módulo 7:CASOS PRÁCTICOS, ESCENARIOS REALES Y AUTOMATIZACIÓN

En los módulos anteriores hemos aprendido los conceptos teóricos: proxy inverso, balanceo de carga (Módulo 4), cifrado TLS, autenticación y rate limiting (Módulo 5). Pero la teoría cobra sentido cuando se aplica a **escenarios reales de producción**. Este documento complementario presenta cuatro casos prácticos completos —con arquitectura, configuración y código funcional— que integran todo lo visto en el curso, seguidos de scripts de automatización para el despliegue y la monitorización.

---

### Caso 1: E-commerce con Alta Concurrencia

Este primer caso es el más completo: combina **balanceo de carga**, **proxy inverso**, **SSL/TLS**, **rate limiting** y **Docker Compose multi-servicio**. Representa la arquitectura típica de una tienda online que necesita servir miles de peticiones simultáneas.

#### Arquitectura

```

Internet → Cloudflare (DDoS Protection)
↓
Nginx (Load Balancer)
↓
┌──────────┼──────────┐
↓          ↓          ↓
Nginx1     Nginx2     Nginx3
(Static)   (Static)   (Static)
↓          ↓          ↓
Node.js    Node.js    Node.js
(API)      (API)      (API)
↓
PostgreSQL (Primary)
↓
PostgreSQL (Replica)

```

#### docker-compose.yml

```yaml


services:
  # Load Balancer
  lb:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/lb.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - frontend
    depends_on:
      - web1
      - web2
      - web3

  # Web Servers
  web1:
    image: nginx:alpine
    volumes:
      - ./nginx/web.conf:/etc/nginx/nginx.conf:ro
      - ./static:/usr/share/nginx/html:ro
    networks:
      - frontend
      - backend
    depends_on:
      - api1

  web2:
    image: nginx:alpine
    volumes:
      - ./nginx/web.conf:/etc/nginx/nginx.conf:ro
      - ./static:/usr/share/nginx/html:ro
    networks:
      - frontend
      - backend
    depends_on:
      - api2

  web3:
    image: nginx:alpine
    volumes:
      - ./nginx/web.conf:/etc/nginx/nginx.conf:ro
      - ./static:/usr/share/nginx/html:ro
    networks:
      - frontend
      - backend
    depends_on:
      - api3

  # API Servers
  api1:
    image: node:20-alpine
    working_dir: /app
    volumes:
      - ./app:/app
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - REDIS_HOST=redis
    networks:
      - backend
    command: node server.js

  api2:
    image: node:20-alpine
    working_dir: /app
    volumes:
      - ./app:/app
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - REDIS_HOST=redis
    networks:
      - backend
    command: node server.js

  api3:
    image: node:20-alpine
    working_dir: /app
    volumes:
      - ./app:/app
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - REDIS_HOST=redis
    networks:
      - backend
    command: node server.js

  # Database
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: admin
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - backend
    secrets:
      - db_password

  # Redis Cache
  redis:
    image: redis:7-alpine
    networks:
      - backend
    command: redis-server --appendonly yes

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true

volumes:
  db_data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```


#### nginx/lb.conf (Load Balancer)

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';
    
    access_log /var/log/nginx/access.log main buffer=32k flush=5s;
    
    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss 
               application/rss+xml font/truetype font/opentype 
               application/vnd.ms-fontobject image/svg+xml;
    
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=50r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
    
    # Upstream Web Servers
    upstream web_backend {
        least_conn;
        
        server web1:80 max_fails=3 fail_timeout=30s;
        server web2:80 max_fails=3 fail_timeout=30s;
        server web3:80 max_fails=3 fail_timeout=30s;
        
        keepalive 32;
    }
    
    # HTTP → HTTPS Redirect
    server {
        listen 80;
        server_name example.com www.example.com;
        
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
        
        location / {
            return 301 https://$host$request_uri;
        }
    }
    
    # HTTPS Server
    server {
        listen 443 ssl http2;
        server_name example.com www.example.com;
        
        # SSL Configuration
        ssl_certificate /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        
        # Security Headers
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        
        # Hide Version
        server_tokens off;
        
        # Rate Limiting
        limit_req zone=general burst=20 delay=10;
        
        # Proxy to Web Servers
        location / {
            proxy_pass http://web_backend;
            
            # Headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Request-ID $request_id;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
            
            # Buffering
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
            
            # Retry Logic
            proxy_next_upstream error timeout http_502 http_503 http_504;
            proxy_next_upstream_tries 3;
            proxy_next_upstream_timeout 10s;
        }
        
        # API Rate Limiting
        location /api/ {
            limit_req zone=api burst=100 delay=50;
            
            proxy_pass http://web_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        # Login Rate Limiting
        location /login {
            limit_req zone=login burst=3;
            
            proxy_pass http://web_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        # Health Check
        location /health {
            access_log off;
            return 200 "OK\n";
        }
    }
}
```


---

### Caso 2: Microservicios con API Gateway

El caso anterior usaba una arquitectura monolítica (un único backend Node.js replicado). En la práctica, muchas aplicaciones modernas se dividen en **microservicios independientes**, cada uno con su propia base de datos. Nginx actúa como **API Gateway**, enrutando cada petición al servicio correspondiente según la URL.

#### Arquitectura

```
Internet → Nginx (API Gateway)
              ↓
   ┌──────────┼──────────┐
   ↓          ↓          ↓
Users API  Products  Orders
Service    Service   Service
   ↓          ↓          ↓
MongoDB   PostgreSQL  MySQL
```


#### nginx/gateway.conf

```nginx
user nginx;
worker_processes auto;

events {
    worker_connections 2048;
}

http {
    include /etc/nginx/mime.types;
    default_type application/json;
    
    # JSON Logging
    log_format json_combined escape=json
        '{'
            '"time":"$time_iso8601",'
            '"remote_addr":"$remote_addr",'
            '"request_method":"$request_method",'
            '"request_uri":"$request_uri",'
            '"status":$status,'
            '"body_bytes_sent":$body_bytes_sent,'
            '"request_time":$request_time,'
            '"upstream_response_time":"$upstream_response_time",'
            '"upstream_addr":"$upstream_addr",'
            '"http_user_agent":"$http_user_agent",'
            '"http_x_forwarded_for":"$http_x_forwarded_for"'
        '}';
    
    access_log /var/log/nginx/access.json json_combined;
    
    # Rate Limiting per Service
    limit_req_zone $binary_remote_addr zone=users_api:10m rate=100r/s;
    limit_req_zone $binary_remote_addr zone=products_api:10m rate=200r/s;
    limit_req_zone $binary_remote_addr zone=orders_api:10m rate=50r/s;
    
    # Upstreams
    upstream users_service {
        least_conn;
        server users1:3001 max_fails=3 fail_timeout=30s;
        server users2:3001 max_fails=3 fail_timeout=30s;
        keepalive 16;
    }
    
    upstream products_service {
        least_conn;
        server products1:3002 max_fails=3 fail_timeout=30s;
        server products2:3002 max_fails=3 fail_timeout=30s;
        keepalive 16;
    }
    
    upstream orders_service {
        least_conn;
        server orders1:3003 max_fails=3 fail_timeout=30s;
        server orders2:3003 max_fails=3 fail_timeout=30s;
        keepalive 16;
    }
    
    # API Gateway
    server {
        listen 80;
        server_name api.example.com;
        
        # CORS Headers
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, X-Requested-With' always;
        
        # Preflight requests
        if ($request_method = 'OPTIONS') {
            return 204;
        }
        
        # Users Service
        location /api/users {
            limit_req zone=users_api burst=200 delay=100;
            
            rewrite ^/api/users(.*)$ $1 break;
            proxy_pass http://users_service;
            
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Request-ID $request_id;
            
            proxy_connect_timeout 3s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
        }
        
        # Products Service
        location /api/products {
            limit_req zone=products_api burst=400 delay=200;
            
            rewrite ^/api/products(.*)$ $1 break;
            proxy_pass http://products_service;
            
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Request-ID $request_id;
            
            # Caching for GET requests
            proxy_cache products_cache;
            proxy_cache_valid 200 5m;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_background_update on;
            proxy_cache_lock on;
        }
        
        # Orders Service
        location /api/orders {
            limit_req zone=orders_api burst=100 delay=50;
            
            rewrite ^/api/orders(.*)$ $1 break;
            proxy_pass http://orders_service;
            
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Request-ID $request_id;
        }
        
        # Health Check
        location /health {
            access_log off;
            return 200 '{"status":"healthy","timestamp":"$time_iso8601"}';
            default_type application/json;
        }
        
        # Catch-all
        location / {
            return 404 '{"error":"Endpoint not found","path":"$request_uri"}';
            default_type application/json;
        }
    }
    
    # Cache Zone
    proxy_cache_path /var/cache/nginx/products levels=1:2 keys_zone=products_cache:10m max_size=100m inactive=60m use_temp_path=off;
}
```


---

### Caso 3: CDN Interno con Caching Agresivo

En los casos anteriores, Nginx actuaba como proxy y gateway. Pero otra de sus funciones más potentes es el **caching de contenido estático**: imágenes, CSS, JavaScript y fuentes. En este caso, Nginx funciona como un **CDN interno** que reduce drásticamente la carga en los servidores de origen.

#### nginx/cdn.conf

```nginx
user nginx;
worker_processes auto;

events {
    worker_connections 4096;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging
    access_log /var/log/nginx/access.log combined;
    
    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # Cache Zones
    proxy_cache_path /var/cache/nginx/images 
                     levels=1:2 
                     keys_zone=images:100m 
                     max_size=10g 
                     inactive=30d 
                     use_temp_path=off;
    
    proxy_cache_path /var/cache/nginx/static 
                     levels=1:2 
                     keys_zone=static:50m 
                     max_size=5g 
                     inactive=7d 
                     use_temp_path=off;
    
    # Upstream Origin
    upstream origin {
        server origin1.internal:80;
        server origin2.internal:80 backup;
    }
    
    # CDN Server
    server {
        listen 80;
        server_name cdn.example.com;
        
        # Images Cache
        location ~* \.(jpg|jpeg|png|gif|webp|ico|svg)$ {
            proxy_pass http://origin;
            
            # Cache Configuration
            proxy_cache images;
            proxy_cache_valid 200 30d;
            proxy_cache_valid 404 10m;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_background_update on;
            proxy_cache_lock on;
            proxy_cache_lock_timeout 5s;
            
            # Cache Headers
            add_header X-Cache-Status $upstream_cache_status;
            add_header Cache-Control "public, max-age=2592000, immutable";
            
            # Headers to Origin
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            
            # Ignore Cache Control from Origin
            proxy_ignore_headers Cache-Control Expires;
            
            expires 30d;
            access_log off;
        }
        
        # CSS/JS Cache
        location ~* \.(css|js)$ {
            proxy_pass http://origin;
            
            proxy_cache static;
            proxy_cache_valid 200 7d;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            
            add_header X-Cache-Status $upstream_cache_status;
            add_header Cache-Control "public, max-age=604800";
            
            gzip on;
            gzip_types text/css application/javascript;
            gzip_comp_level 9;
            
            expires 7d;
        }
        
        # Fonts Cache
        location ~* \.(woff|woff2|ttf|eot)$ {
            proxy_pass http://origin;
            
            proxy_cache static;
            proxy_cache_valid 200 365d;
            
            add_header Cache-Control "public, max-age=31536000, immutable";
            add_header Access-Control-Allow-Origin "*";
            
            expires 365d;
            access_log off;
        }
        
        # Cache Purge Endpoint (solo desde red interna)
        location /purge {
            allow 10.0.0.0/8;
            deny all;
            
            proxy_cache_purge images $scheme$proxy_host$request_uri;
        }
    }
}
```


---

### Caso 4: WebSocket Proxy con Sticky Sessions

Los tres casos anteriores usan HTTP estándar (petición-respuesta). Sin embargo, aplicaciones como chats en tiempo real, dashboards o juegos online necesitan **conexiones persistentes bidireccionales**: WebSockets. Este caso muestra cómo configurar Nginx para hacer proxy de conexiones WebSocket con sticky sessions (afinidad de sesión).

#### nginx/websocket.conf

```nginx
user nginx;
worker_processes auto;

events {
    worker_connections 2048;
}

http {
    # Mapping for WebSocket upgrade
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }
    
    # Upstream with IP Hash for sticky sessions
    upstream websocket_backend {
        ip_hash;
        
        server ws1:8080 max_fails=3 fail_timeout=30s;
        server ws2:8080 max_fails=3 fail_timeout=30s;
        server ws3:8080 max_fails=3 fail_timeout=30s;
    }
    
    server {
        listen 443 ssl http2;
        server_name ws.example.com;
        
        ssl_certificate /etc/nginx/certs/cert.pem;
        ssl_certificate_key /etc/nginx/certs/key.pem;
        
        # WebSocket Location
        location /ws {
            proxy_pass http://websocket_backend;
            
            # WebSocket Headers
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            
            # Standard Proxy Headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts (largo para WebSocket)
            proxy_connect_timeout 7d;
            proxy_send_timeout 7d;
            proxy_read_timeout 7d;
            
            # No buffering para WebSocket
            proxy_buffering off;
        }
        
        # Regular HTTP
        location / {
            proxy_pass http://websocket_backend;
            
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```


---

## Scripts de Automatización

Una vez que la infraestructura está configurada, el siguiente paso es **automatizar las tareas repetitivas**: despliegue, monitorización y renovación de certificados SSL. Los siguientes scripts resuelven las operaciones más frecuentes del día a día.

### deploy.sh - Despliegue Automatizado

```bash
#!/bin/bash
set -e

PROJECT_NAME="nginx-web"
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)

echo "🚀 Starting deployment of $PROJECT_NAME..."

# 1. Backup actual
echo "📦 Creating backup..."
docker-compose exec nginx tar czf /tmp/nginx-backup-$DATE.tar.gz /etc/nginx /var/www/html
docker cp $(docker-compose ps -q nginx):/tmp/nginx-backup-$DATE.tar.gz $BACKUP_DIR/

# 2. Test nueva configuración
echo "🔍 Testing new configuration..."
docker run --rm \
  -v $(pwd)/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/nginx/conf.d:/etc/nginx/conf.d:ro \
  nginx:alpine nginx -t

if [ $? -ne 0 ]; then
  echo "❌ Configuration test failed. Aborting."
  exit 1
fi

# 3. Pull nuevas imágenes
echo "📥 Pulling new images..."
docker-compose pull

# 4. Build custom images
echo "🔨 Building custom images..."
docker-compose build

# 5. Deploy con zero-downtime
echo "🔄 Deploying with zero-downtime..."
docker-compose up -d --no-deps --build --force-recreate

# 6. Wait for health check
echo "⏳ Waiting for health check..."
sleep 5

for i in {1..30}; do
  if curl -f http://localhost/health > /dev/null 2>&1; then
    echo "✅ Health check passed"
    break
  fi
  
  if [ $i -eq 30 ]; then
    echo "❌ Health check failed. Rolling back..."
    docker-compose down
    docker-compose up -d
    exit 1
  fi
  
  sleep 1
done

# 7. Cleanup old images
echo "🧹 Cleaning up old images..."
docker image prune -f

echo "✅ Deployment completed successfully!"
echo "📊 Running containers:"
docker-compose ps
```


---

### monitor.sh - Monitorización

El script de despliegue asegura que el servicio arranca correctamente, pero ¿cómo sabemos si **sigue funcionando bien** después de horas o días? Este script genera un dashboard rápido del estado actual:

```bash
#!/bin/bash

# Colores
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo "📊 Nginx Monitoring Dashboard"
echo "=============================="

# 1. Container Status
echo -e "\n${GREEN}Container Status:${NC}"
docker-compose ps

# 2. CPU/Memory Usage
echo -e "\n${GREEN}Resource Usage:${NC}"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

# 3. Nginx Connections
echo -e "\n${GREEN}Nginx Connections:${NC}"
docker exec nginx sh -c 'echo "GET /nginx_status HTTP/1.0\r\n\r\n" | nc localhost 80' 2>/dev/null || echo "Stub status not configured"

# 4. Recent Errors
echo -e "\n${YELLOW}Recent Errors (last 10):${NC}"
docker-compose logs --tail=10 nginx | grep -i error || echo "No recent errors"

# 5. Request Rate
echo -e "\n${GREEN}Request Rate (last minute):${NC}"
docker exec nginx sh -c 'tail -1000 /var/log/nginx/access.log | awk "{print \$4}" | cut -d: -f1-3 | sort | uniq -c | tail -5'

# 6. Top IPs
echo -e "\n${GREEN}Top 10 IPs:${NC}"
docker exec nginx sh -c 'tail -10000 /var/log/nginx/access.log | awk "{print \$1}" | sort | uniq -c | sort -rn | head -10'

# 7. Status Codes
echo -e "\n${GREEN}Status Code Distribution:${NC}"
docker exec nginx sh -c 'tail -10000 /var/log/nginx/access.log | awk "{print \$9}" | sort | uniq -c | sort -rn'

# 8. Slow Requests (>1s)
echo -e "\n${YELLOW}Slow Requests (>1s):${NC}"
docker exec nginx sh -c 'tail -1000 /var/log/nginx/access.log | awk "{if (\$NF>1) print \$0}" | tail -5' || echo "No slow requests"
```


---

### ssl-renew.sh - Renovación SSL Automática

Los certificados de Let's Encrypt caducan cada 90 días. Automatizar su renovación evita caídas por certificado expirado:

```bash
#!/bin/bash
set -e

DOMAIN="example.com"
EMAIL="admin@example.com"

echo "🔐 Renewing SSL certificate for $DOMAIN..."

# 1. Renovar certificado
docker-compose run --rm certbot renew --nginx --quiet

# 2. Verificar si se renovó
if [ $? -eq 0 ]; then
  echo "✅ Certificate renewed successfully"
  
  # 3. Reload Nginx
  docker-compose exec nginx nginx -s reload
  
  # 4. Test SSL
  echo "🔍 Testing SSL configuration..."
  echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null | openssl x509 -noout -dates
  
  echo "✅ SSL renewal completed!"
else
  echo "❌ Certificate renewal failed"
  exit 1
fi
```


---

## Testing y Benchmarking

Por último, antes de dar por buena una configuración conviene **medir su rendimiento real** bajo carga. Este script utiliza Apache Bench y wrk para simular tráfico concurrente:

### test-performance.sh

```bash
#!/bin/bash

TARGET="http://localhost"
CONCURRENT=100
REQUESTS=10000

echo "⚡ Performance Testing"
echo "====================="

# 1. Apache Bench
echo -e "\n📊 Apache Bench Results:"
ab -n $REQUESTS -c $CONCURRENT -g bench.tsv $TARGET/

# 2. wrk (si está instalado)
if command -v wrk &> /dev/null; then
  echo -e "\n📊 wrk Results:"
  wrk -t12 -c400 -d30s $TARGET/
fi

# 3. Nginx Metrics
echo -e "\n📈 Nginx Metrics:"
curl -s http://localhost/nginx_status

# 4. Generar reporte
echo -e "\n📄 Generating report..."
cat bench.tsv | awk -F'\t' '{sum+=$5; count++} END {print "Average response time: " sum/count " ms"}'
```


---

## Referencias

- [Zero-Downtime Deploys with Docker & Nginx](https://dev.to/ramer2b58cbe46bc8/the-ultimate-checklist-for-zero-downtime-deploys-with-docker-nginx-20i5)
- [Nginx Docker Compose Tutorial](https://geshan.com.np/blog/2024/03/nginx-docker-compose/)
- [Nginx Instance Manager - Docker Compose](https://docs.nginx.com/nginx-instance-manager/deploy/docker/deploy-nginx-instance-manager-docker-compose/)
- [Monitor Nginx with FluentBit](https://openobserve.ai/blog/how-to-monitor-nginx-with-fluentbit-and-o2/)
- [Bash Script to Analyze Nginx Logs](https://sysadmins.co.za/bash-script-to-parse-and-analyze-nginx-access-logs/)
