## Solucionario Actividad 2: Balanceo de Carga con Nginx y Docker

### 1. Estructura de archivos

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

### 2. Aplicación Backend (Node.js)

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

### 3. Docker Compose

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

### 4. Configuraciones de Nginx

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

### 5. Script de pruebas

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

### 6. Pasos de despliegue

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

### 7. Verificaciones

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
