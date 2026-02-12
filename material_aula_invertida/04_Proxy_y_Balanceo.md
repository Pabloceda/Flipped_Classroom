# Investigación Nginx-Docker (Parte 2)

## MÓDULO 4: PROXY INVERSO Y BALANCEO

En el Módulo 3 aprendimos a configurar Nginx para servir contenido estático. Pero en la práctica, las aplicaciones web modernas tienen **múltiples componentes**: una app en Node.js, otra en PHP, una base de datos, un servicio de caché... ¿Cómo exponemos todo esto al usuario como si fuera un solo servidor? La respuesta es el **proxy inverso**, y Nginx es una de las mejores herramientas para implementarlo.

---

### 4.1 Concepto de Proxy Inverso

#### Definición Técnica

Un **proxy inverso** (reverse proxy) es un servidor que se sitúa entre los clientes (navegadores) y los servidores backend (donde están las aplicaciones). Intercepta todas las peticiones de los clientes y las reenvía al backend adecuado.

**¿Por qué es importante?** Imagina que tienes una tienda online: el frontend está en React (puerto 3000), la API en Node.js (puerto 8080) y las imágenes en otro servidor. Sin proxy inverso, el usuario tendría que conocer tres direcciones diferentes. Con Nginx como proxy inverso, el usuario accede a `ejemplo.com` y Nginx distribuye internamente cada petición al servidor correcto.

#### Proxy Forward vs Proxy Inverso

**Forward Proxy**:

```
Cliente → [Forward Proxy] → Internet → Servidor
        (cliente configura proxy)
```

- Cliente conoce y configura el proxy
- Oculta identidad del cliente
- Usado para filtrado, caché, bypass de restricciones

**Reverse Proxy**:

```
Cliente → Internet → [Reverse Proxy] → Backend
                    (transparente al cliente)
```

- Cliente NO sabe que hay proxy
- Oculta infraestructura backend
- Usado para load balancing, SSL termination, caching


#### Arquitectura con Nginx como Reverse Proxy

```
                    ┌──────────────────────┐
Internet ───────────┤   Nginx (Reverse    │
                    │   Proxy) :80, :443   │
                    └──────────┬───────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
         ┌──────▼───┐   ┌─────▼────┐  ┌──────▼───┐
         │  Web App │   │  API     │  │  Static  │
         │  :3000   │   │  :8080   │  │  :9000   │
         └──────────┘   └──────────┘  └──────────┘
```

**Ventajas**:

1. **Terminación SSL/TLS**: Nginx maneja cifrado, backend en HTTP
2. **Load Balancing**: Distribuye carga entre múltiples backends
3. **Caching**: Cachea respuestas para reducir carga backend
4. **Compresión**: Gzip/Brotli en proxy, backend ahorra CPU
5. **Seguridad**: Oculta versiones de software, IPs internas
6. **Routing**: Diferentes URIs a diferentes backends

---

### 4.2 Arquitectura General

Antes de entrar en directivas, veamos cómo encaja Nginx como proxy inverso en una arquitectura web real. Este diagrama muestra el recorrido completo de una petición:

```
┌─────────────────────────────────────────────────────────────┐
│                         INTERNET                            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                    ┌───────▼────────┐
                    │  Firewall      │
                    │  Port 80, 443  │
                    └───────┬────────┘
                            │
             ┌──────────────▼──────────────┐
             │   Nginx Reverse Proxy       │
             │   - SSL Termination         │
             │   - Load Balancing          │
             │   - Caching                 │
             │   - Compression             │
             │   - Rate Limiting           │
             └──────────────┬──────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   ┌────▼─────┐        ┌───▼────┐         ┌────▼─────┐
   │  Nginx   │        │  Node  │         │  PHP-FPM │
   │  Static  │        │  App   │         │  Backend │
   │  :9000   │        │  :3000 │         │  :9000   │
   └──────────┘        └───┬────┘         └────┬─────┘
                           │                   │
                       ┌───▼───────────────────▼───┐
                       │   PostgreSQL Database     │
                       │   :5432                   │
                       └───────────────────────────┘
```


#### Flujo de Petición

```
1. Cliente HTTPS → Nginx:443
2. Nginx termina SSL
3. Nginx decide routing:
   - /static/* → Nginx Static (9000)
   - /api/* → Node App (3000)
   - /*.php → PHP-FPM (9000)
4. Backend procesa
5. Backend responde a Nginx
6. Nginx cachea (si aplicable)
7. Nginx comprime
8. Nginx envía a cliente
```


---

### 4.3 Directiva `proxy_pass`

Ya hemos visto la arquitectura general. Ahora, ¿cómo le decimos a Nginx que **reenvíe** una petición a un backend? Para eso se usa `proxy_pass`, la directiva más importante del proxy inverso.

#### Sintaxis Básica

```nginx
location /api/ {
    proxy_pass http://backend_server;
}
```


#### Comportamiento con/sin Trailing Slash

**Sin URI en proxy_pass** (preserva URI completa):

```nginx
location /api/ {
    proxy_pass http://localhost:3000;
}
# Petición: /api/users
# Backend recibe: /api/users
```

**Con URI en proxy_pass** (reemplaza matched part):

```nginx
location /api/ {
    proxy_pass http://localhost:3000/;  # ← trailing slash
}
# Petición: /api/users
# Backend recibe: /users (se quita /api/)
```

> **⚠️ Error común**: Olvidar o poner el trailing slash cambia completamente el comportamiento. Es una de las fuentes de errores más frecuentes al configurar proxy_pass. Ante la duda, prueba siempre con `curl` qué recibe tu backend.

**Ejemplo con path diferente**:

```nginx
location /api/ {
    proxy_pass http://localhost:3000/v1/;
}
# Petición: /api/users
# Backend recibe: /v1/users
```


#### Proxy a Upstream Block

```nginx
upstream backend_api {
    server 192.168.1.10:3000;
    server 192.168.1.11:3000;
}

server {
    location /api/ {
        proxy_pass http://backend_api;
    }
}
```


#### Proxy con Variables

```nginx
location ~ ^/proxy/(.+)$ {
    proxy_pass http://backend/$1;
}
# Petición: /proxy/users/123
# Backend: http://backend/users/123
```


---

### 4.4 Cabeceras Proxy (Headers)

Cuando Nginx actúa como proxy, el backend pierde información importante del cliente original (su IP, el protocolo, el dominio). Las cabeceras proxy permiten **transmitir esa información** al backend.

Cuando Nginx hace proxy, el backend ve la petición como proveniente de Nginx (no del cliente original).

**Sin headers proxy**:

```
Cliente (IP: 203.0.113.45)
    ↓
Nginx (IP: 192.168.1.5)
    ↓
Backend ve: $remote_addr = 192.168.1.5  ← IP de Nginx, NO del cliente
```


#### Directiva `proxy_set_header`

**Sintaxis**:

```nginx
proxy_set_header Header-Name Value;
```


#### Headers Esenciales

```nginx
location / {
    proxy_pass http://backend;
    
    # 1. Host original
    proxy_set_header Host $host;
    # O: proxy_set_header Host $http_host;  (incluye puerto)
    
    # 2. IP real del cliente
    proxy_set_header X-Real-IP $remote_addr;
    
    # 3. Cadena de proxies (para múltiples proxies)
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    # 4. Protocolo original (http o https)
    proxy_set_header X-Forwarded-Proto $scheme;
    
    # 5. Puerto original
    proxy_set_header X-Forwarded-Port $server_port;
    
    # 6. Hostname del proxy
    proxy_set_header X-Forwarded-Host $host;
}
```


#### Explicación de Headers

**1. `Host`**:

- Backend necesita saber el hostname original para virtual hosting
- Sin esto, backend podría responder con host incorrecto

**El "Baile de Cabeceras" (Visualización)**:
> Imagina que `ejemplo.com` es una etiqueta en la camisa del cliente.
> 1. **Cliente** llega a Nginx con la etiqueta `Host: ejemplo.com`.
> 2. **Nginx** se gira para hablar con el Backend (localhost:3000). Si no le pasamos la etiqueta, Nginx se presentará como "localhost".
> 3. **Backend** genera un link. Como cree que se llama "localhost", genera `http://localhost:3000/perfil`.
> 4. **Cliente** recibe ese link roto y el sitio falla.
>
> **Solución**: `proxy_set_header Host $host;` le pega la etiqueta original al backend para que sepa quién es realmente.

**2. `X-Real-IP`**:

- IP del cliente final
- Útil para logging, geolocalización, rate limiting

**3. `X-Forwarded-For`** (XFF):

- Lista de IPs en cadena de proxies
- Formato: `client, proxy1, proxy2`
- `$proxy_add_x_forwarded_for` añade \$remote_addr a lista existente

**Ejemplo de XFF**:

```
Cliente (203.0.113.45)
    ↓
Proxy 1 (añade: X-Forwarded-For: 203.0.113.45)
    ↓
Proxy 2 (añade: X-Forwarded-For: 203.0.113.45, proxy1_ip)
    ↓
Backend ve: X-Forwarded-For: 203.0.113.45, 10.0.0.1
```

**4. `X-Forwarded-Proto`**:

- `http` o `https`
- Backend sabe si conexión original fue segura

**5. `X-Forwarded-Port`**:

- Puerto original (80, 443, custom)


#### Template Completo de Proxy Headers

```nginx
# Configuración reusable
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Port $server_port;
proxy_set_header X-Forwarded-Host $host;

# Headers adicionales útiles
proxy_set_header X-Request-ID $request_id;  # Request tracking
proxy_set_header X-Nginx-Proxy true;        # Identifica proxy

# Remover headers sensibles
proxy_set_header Authorization "";  # Si no se necesita en backend
```


#### Caso de Uso: Backend PHP (FastCGI)

> **Nota**: PHP no usa `proxy_pass` (HTTP), sino `fastcgi_pass` (protocolo FastCGI). Es importante no confundirlos.

```nginx
location ~ \.php$ {
    fastcgi_pass php-fpm:9000;    # FastCGI, NO proxy_pass HTTP
    
    # PHP necesita saber el script real
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
    
    # IP real para logs de PHP
    fastcgi_param HTTP_X_REAL_IP $remote_addr;
}
```


---

### 4.5 Upstreams (Balanceo de Carga)

Hasta ahora hemos redirigido peticiones a **un solo backend**. Pero, ¿qué pasa si ese servidor se cae o no puede con todo el tráfico? La solución es tener **múltiples servidores backend** y distribuir la carga entre ellos. En Nginx, esto se define con bloques `upstream`.

Un **upstream** es un grupo de servidores backend que Nginx usa para distribuir tráfico (load balancing).

#### Sintaxis Básica

```
┌───────────────┐     ┌───────────────────┐
│               │     │  backend1 :3000  │
│   Nginx       │───►│                   │
│   (proxy)     │     └───────────────────┘
│               │     ┌───────────────────┐
│  upstream     │───►│  backend2 :3000  │
│  "backend"    │     │                   │
│               │     └───────────────────┘
│               │     ┌───────────────────┐
│               │───►│  backend3 :3000  │
└───────────────┘     └───────────────────┘
```

```nginx
upstream backend_name {
    server backend1.example.com:3000;
    server backend2.example.com:3000;
    server 192.168.1.10:8080;
}

server {
    location / {
        proxy_pass http://backend_name;
    }
}
```


#### Parámetros de Server

```nginx
upstream backend {
    # Peso (por defecto: weight=1)
    server backend1.com weight=3;  # Recibe 3x más tráfico
    server backend2.com weight=1;
    
    # Max connections
    server backend3.com max_conns=100;
    
    # Max fails antes de marcar como down
    server backend4.com max_fails=3 fail_timeout=30s;
    
    # Backup (solo se usa si todos los demás fallan)
    server backend5.com backup;
    
    # Down (administrativamente deshabilitado)
    server backend6.com down;
}
```


---

### 4.6 Algoritmos de Balanceo

Ya definimos un grupo de backends con `upstream`. Ahora, ¿cómo decide Nginx **a cuál enviar** cada petición? Para eso existen los algoritmos de balanceo.

#### 1. Round Robin (Default)

Distribuye peticiones secuencialmente.

```nginx
upstream backend {
    server backend1.com;
    server backend2.com;
    server backend3.com;
}
# Petición 1 → backend1
# Petición 2 → backend2
# Petición 3 → backend3
# Petición 4 → backend1 (repite)
```

**Con pesos**:

```nginx
upstream backend {
    server backend1.com weight=3;
    server backend2.com weight=1;
}
# 3 de cada 4 peticiones → backend1
# 1 de cada 4 peticiones → backend2
```


#### 2. Least Connections (least_conn)

Envía peticiones al servidor con menos conexiones activas.

```nginx
upstream backend {
    least_conn;
    
    server backend1.com;
    server backend2.com;
    server backend3.com;
}
```

**Uso ideal**: Peticiones de duración variable (algunos requests lentos, otros rápidos)

#### 3. IP Hash (ip_hash)

**¿Qué son las sticky sessions?** En aplicaciones con estado (como un carrito de la compra guardado en memoria del servidor), necesitamos que un mismo usuario siempre llegue al **mismo backend**. Si el usuario añade un producto en backend1 y la siguiente petición va a backend2, su carrito estará vacío.

`ip_hash` resuelve esto enviando siempre al mismo backend según la IP del cliente:

```nginx
upstream backend {
    ip_hash;
    
    server backend1.com;
    server backend2.com;
    server backend3.com;
}
```

**Cálculo**: `hash(client_ip) % num_servers`

**Uso ideal**: Aplicaciones con sesiones no compartidas

**Limitaciones**:

- No funciona bien con IPv6 (subredes grandes)
- Problemas con clientes detrás de NAT (misma IP)


#### 4. Generic Hash (hash)

Hash customizable por cualquier variable.

```nginx
# Hash por URI
upstream backend {
    hash $request_uri consistent;
    server backend1.com;
    server backend2.com;
}

# Hash por cookie
upstream backend {
    hash $cookie_sessionid consistent;
    server backend1.com;
    server backend2.com;
}

# Hash por header custom
upstream backend {
    hash $http_x_user_id consistent;
    server backend1.com;
    server backend2.com;
}
```

**`consistent`**: Usa consistent hashing (minimiza redistribución cuando cambia número de backends)

#### 5. Least Time (least_time) - Solo NGINX Plus

> **Nota**: Este algoritmo solo está disponible en NGINX Plus (versión comercial de pago). No se puede usar en la versión open source gratuita.

Envía a servidor con menor tiempo de respuesta promedio.

```nginx
upstream backend {
    least_time header;  # O: last_byte
    
    server backend1.com;
    server backend2.com;
}
```


#### Comparación de Algoritmos

| Algoritmo | Sticky Sessions | Balance | Uso Ideal |
| :-- | :-- | :-- | :-- |
| Round Robin | ❌ No | Equilibrado | Backends homogéneos |
| Weighted RR | ❌ No | Ponderado | Backends de diferente capacidad |
| Least Conn | ❌ No | Dinámico | Requests de duración variable |
| IP Hash | ✅ Sí | Estático | Apps con sesión local |
| Generic Hash | ✅ Sí (configurable) | Flexible | Casos custom |
| Least Time | ❌ No | Inteligente | Performance crítico |


---

### 4.7 Health Checks

¿Qué ocurre si uno de los backends del grupo `upstream` se cae? Sin health checks, Nginx seguiría enviando peticiones a un servidor caído, y los usuarios verían errores. Los health checks permiten a Nginx **detectar automáticamente** qué backends están funcionando y cuáles no.

#### Passive Health Checks (Open Source Nginx)

Nginx marca servidor como unhealthy basándose en respuestas reales.

```nginx
upstream backend {
    server backend1.com max_fails=3 fail_timeout=30s;
    server backend2.com max_fails=3 fail_timeout=30s;
}

# max_fails: Fallos consecutivos antes de marcar como down
# fail_timeout: Tiempo que permanece marcado como down
```

**Comportamiento**:

1. Si backend falla 3 veces consecutivas → marcado como unhealthy
2. No recibe tráfico durante 30 segundos
3. Después de 30s, se reintenta (1 request de prueba)
4. Si tiene éxito → vuelve a pool

#### Directiva `proxy_next_upstream`

Define qué se considera "fallo":

```nginx
location / {
    proxy_pass http://backend;
    
    # Reintentar en otro backend si:
    proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    
    # Límites
    proxy_next_upstream_tries 3;      # Máx 3 intentos
    proxy_next_upstream_timeout 30s;  # Timeout total
}
```

**Opciones de `proxy_next_upstream`**:

- `error`: Error de conexión
- `timeout`: Timeout de conexión/lectura
- `invalid_header`: Cabecera inválida
- `http_500`, `http_502`, `http_503`, `http_504`: Códigos de error específicos
- `off`: No reintentar


#### Active Health Checks (Solo NGINX Plus)

> **Nota**: Los health checks activos solo están disponibles en NGINX Plus (de pago). En la versión gratuita, solo disponemos de los passive health checks descritos arriba, que para la mayoría de escenarios de administración de servidores web son suficientes.

NGINX Plus envía requests de prueba periódicos al backend, sin necesidad de que un cliente real falle primero:

```nginx
upstream backend {
    zone backend 64k;
    server backend1.com;
    server backend2.com;
}

server {
    location / {
        proxy_pass http://backend;
        health_check interval=5s fails=3 passes=2 uri=/health;
    }
}
```

---

### 4.8 Buffering y Timeouts

Cuando Nginx actúa como proxy, hay dos conexiones separadas: una con el **cliente** y otra con el **backend**. Los buffers y timeouts controlan cómo se gestiona la transferencia de datos entre ambas, evitando que clientes lentos bloqueen a los backends.

Sin buffering, Nginx mantendría conexión a backend mientras envía respuesta lenta a cliente, bloqueando worker del backend.

**Con buffering**:

```
Backend → Nginx (rápido, buffered)
          ↓
          Cliente (lento, desde buffer de Nginx)
```

Backend se libera rápidamente, Nginx maneja la escritura lenta.

```
Sin buffering:                     Con buffering:
Backend ─── lento ───► Cliente    Backend ── rápido ─► Nginx ─ lento ─► Cliente
(bloqueado hasta que               (se libera                (buffer)
 el cliente reciba todo)             inmediatamente)
```

#### Directivas de Buffering

```nginx
location / {
    proxy_pass http://backend;
    
    # Buffer de respuesta
    proxy_buffering on;                  # Default: on
    proxy_buffer_size 4k;                # Buffer para headers
    proxy_buffers 8 4k;                  # Número y tamaño de buffers
    proxy_busy_buffers_size 8k;          # Límite de buffers ocupados
    
    # Buffer de request body
    client_body_buffer_size 128k;        # Buffer para POST data
    client_max_body_size 10M;            # Tamaño máximo de upload
    
    # Buffering en disco si excede RAM
    proxy_temp_path /var/cache/nginx/proxy_temp;
    proxy_max_temp_file_size 1024M;
}
```


#### Timeouts

```nginx
location / {
    proxy_pass http://backend;
    
    # Timeout de conexión al backend
    proxy_connect_timeout 60s;       # Default: 60s
    
    # Timeout de lectura desde backend
    proxy_read_timeout 60s;          # Default: 60s
    # Se resetea con cada lectura exitosa
    
    # Timeout de escritura a backend
    proxy_send_timeout 60s;          # Default: 60s
    
    # Timeout de cliente
    client_header_timeout 60s;       # Lectura de headers del cliente
    client_body_timeout 60s;         # Lectura de body del cliente
    send_timeout 60s;                # Envío al cliente
}
```


#### Configuración para APIs Lentas

```nginx
# API que toma tiempo procesando
location /api/reports {
    proxy_pass http://backend_reports;
    
    proxy_connect_timeout 5s;       # Conexión rápida
    proxy_read_timeout 300s;        # Lectura lenta (5 minutos)
    proxy_send_timeout 30s;         # Escritura normal
    
    proxy_buffering on;             # Buffer la respuesta
    proxy_buffers 16 32k;           # Más buffers para respuestas grandes
}
```


#### Configuración para File Uploads

```nginx
location /upload {
    proxy_pass http://backend;
    
    client_max_body_size 100M;      # Permitir uploads grandes
    client_body_timeout 300s;       # Timeout largo para uploads
    
    proxy_request_buffering off;    # Stream upload directamente
    # Evita buffering completo en Nginx antes de proxy
}
```


#### Desactivar Buffering (Streaming/WebSocket)

En algunos casos (WebSocket, Server-Sent Events), necesitamos que Nginx **no almacene** la respuesta y la reenvíe en tiempo real:

```nginx
location /stream {
    proxy_pass http://backend;
    
    proxy_buffering off;            # No almacenar respuesta
    proxy_cache off;                # No cachear
    proxy_read_timeout 24h;         # Conexión de larga duración
    
    # Necesario para WebSocket
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```


---

### 4.9 Caso de Uso: Nginx + Apache/Node

#### Escenario: Nginx Sirve Estáticos, Apache Dinámicos

**Ventajas**:

- Nginx: excelente para estáticos (alto throughput)
- Apache: buen soporte PHP con mod_php

**Arquitectura**:

```
Cliente → Nginx:80
            ├─ /static/* → Nginx sirve directamente
            └─ /*.php → Proxy a Apache:8080
```

**Configuración Nginx**:

```nginx
upstream apache_backend {
    server 127.0.0.1:8080;
}

server {
    listen 80;
    server_name ejemplo.com;
    
    root /var/www/html;
    
    # Servir estáticos directamente
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
    
    # Proxy PHP a Apache
    location ~ \.php$ {
        proxy_pass http://apache_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Servir HTML directamente
    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Configuración Apache** (`/etc/apache2/ports.conf`):

```apache
# Escuchar solo en localhost:8080
Listen 127.0.0.1:8080
```

**docker-compose.yml**:

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./html:/var/www/html
    networks:
      - webnet
  
  apache:
    image: php:8.2-apache
    expose:
      - "8080"
    volumes:
      - ./html:/var/www/html
    networks:
      - webnet
    environment:
      APACHE_RUN_PORT: 8080

networks:
  webnet:
```


---

#### Escenario: Nginx + Node.js

**Arquitectura**:

```
Cliente → Nginx:443 (HTTPS)
            ├─ / → Sirve React/Vue SPA
            └─ /api/* → Proxy a Node:3000
```

**Configuración**:

```nginx
upstream node_backend {
    server node1:3000;
    server node2:3000;
    server node3:3000;
}

server {
    listen 443 ssl http2;
    server_name app.ejemplo.com;
    
    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    
    root /var/www/dist;  # Build de React/Vue
    
    # SPA routing (History Mode)
    location / {
        try_files $uri $uri/ /index.html;
        
        # Cache para assets con hash
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
    
    # API proxy
    location /api/ {
        proxy_pass http://node_backend;
        
        # Headers estándar
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support (si aplica)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

**Node.js Backend**:

```javascript
const express = require('express');
const app = express();

// Trust proxy (para X-Forwarded-* headers)
app.set('trust proxy', true);

app.get('/api/users', (req, res) => {
  // req.ip ahora refleja X-Forwarded-For
  console.log('Client IP:', req.ip);
  res.json({ users: [] });
});

app.listen(3000, () => {
  console.log('API listening on :3000');
});
```


---

### 4.10 Caso Real: WordPress + MariaDB + Nginx (Docker Compose)

Este caso integra todo lo aprendido en los módulos 2 y 4: Docker Compose, proxy inverso, balanceo y persistencia de datos.

#### Arquitectura

```
                         ┌─────────────────┐
  Internet :443/:80  ───►│  Nginx (Proxy)  │
                         │  + SSL/TLS      │
                         └────────┬────────┘
                                  │ proxy_pass :80
                         ┌────────▼────────┐
                         │   WordPress     │
                         │   (PHP-FPM)     │
                         │   :80           │
                         └────────┬────────┘
                                  │ TCP :3306
                         ┌────────▼────────┐
                         │   MariaDB       │
                         │   :3306         │
                         └─────────────────┘

Volúmenes persistentes:
  wordpress_data → /var/www/html
  db_data        → /var/lib/mysql
  ssl_certs      → /etc/nginx/ssl
```


#### docker-compose.yml Completo

```yaml
version: '3.9'

services:
  # ===== BASE DE DATOS =====
  db:
    image: mariadb:11-jammy
    container_name: wp_mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-rootsecret}
      MYSQL_DATABASE: ${DB_NAME:-wordpress}
      MYSQL_USER: ${DB_USER:-wpuser}
      MYSQL_PASSWORD: ${DB_PASSWORD:-wpsecret}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - backend
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ===== WORDPRESS =====
  wordpress:
    image: wordpress:6-php8.2-fpm-alpine
    container_name: wp_app
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${DB_NAME:-wordpress}
      WORDPRESS_DB_USER: ${DB_USER:-wpuser}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD:-wpsecret}
      WORDPRESS_TABLE_PREFIX: wp_
    volumes:
      - wordpress_data:/var/www/html
    networks:
      - frontend
      - backend

  # ===== NGINX (PROXY INVERSO) =====
  nginx:
    image: nginx:1.25-alpine
    container_name: wp_nginx
    restart: unless-stopped
    depends_on:
      - wordpress
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - wordpress_data:/var/www/html:ro  # Acceso a estáticos de WP
    networks:
      - frontend

# ===== REDES =====
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true   # Sin acceso a Internet (seguridad)

# ===== VOLÚMENES PERSISTENTES =====
volumes:
  db_data:
  wordpress_data:
```

**Puntos clave de diseño**:
- **Red `backend` con `internal: true`**: MariaDB NO tiene acceso a Internet
- **WordPress en ambas redes**: Habla con Nginx (frontend) y MariaDB (backend)
- **Nginx solo en frontend**: No puede alcanzar la base de datos directamente
- **Volúmenes nombrados (`db_data`, `wordpress_data`)**: Datos persistentes entre reinicios
- **Health check en MariaDB**: WordPress no arranca hasta que la BD esté sana (`condition: service_healthy`)


#### Configuración Nginx para WordPress

```nginx
# nginx/conf.d/wordpress.conf

upstream wordpress_fpm {
    server wordpress:9000;   # PHP-FPM escucha en puerto 9000
}

# Redirección HTTP → HTTPS
server {
    listen 80;
    server_name ejemplo.com www.ejemplo.com;
    return 301 https://$host$request_uri;
}

# Servidor HTTPS principal
server {
    listen 443 ssl http2;
    server_name ejemplo.com www.ejemplo.com;

    # === SSL ===
    ssl_certificate     /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # === Headers de seguridad ===
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000" always;

    # === Root de WordPress ===
    root /var/www/html;
    index index.php index.html;

    # === Archivos estáticos (servidos por Nginx, no PHP) ===
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff2?|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # === PHP → WordPress via FastCGI ===
    location ~ \.php$ {
        fastcgi_pass wordpress_fpm;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # Cabeceras para WordPress
        fastcgi_param HTTPS on;
        fastcgi_param HTTP_X_FORWARDED_PROTO $scheme;

        # Timeouts para admin con operaciones largas
        fastcgi_read_timeout 300s;
    }

    # === Permalinks de WordPress ===
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # === Seguridad: bloquear acceso a archivos sensibles ===
    location ~ /\.(ht|git|env) {
        deny all;
    }

    location = /wp-config.php {
        deny all;
    }

    location ~* /wp-content/uploads/.*\.php$ {
        deny all;
    }

    # === Limitar tamaño de subida ===
    client_max_body_size 64M;
}
```

**Aspectos importante**:
- **`upstream wordpress_fpm`**: Usa FastCGI (puerto 9000), NO proxy_pass HTTP
- **`try_files` para permalinks**: WordPress necesita que todas las URLs pasen por `index.php`
- **Archivos sensibles bloqueados**: `.htaccess`, `.git`, `.env`, `wp-config.php`
- **Uploads PHP denegados**: Previene ejecución de scripts subidos maliciosamente


#### Certificado SSL Autofirmado (Desarrollo)

```bash
# Crear directorio
mkdir -p ssl

# Generar certificado autofirmado (válido 365 días)
openssl req -x509 -nodes \
    -days 365 \
    -newkey rsa:2048 \
    -keyout ssl/key.pem \
    -out ssl/cert.pem \
    -subj "/C=ES/ST=Barcelona/L=Barcelona/O=MiOrganizacion/CN=ejemplo.com"
```


#### Archivo `.env` (Variables de entorno)

```env
# .env (NO subir a Git)
DB_ROOT_PASSWORD=mi_root_password_seguro
DB_NAME=wordpress
DB_USER=wpuser
DB_PASSWORD=mi_wp_password_seguro
```


#### Despliegue Paso a Paso

```bash
# 1. Crear estructura de carpetas
mkdir -p nginx/conf.d ssl

# 2. Copiar configuración Nginx (wordpress.conf)
# 3. Generar certificado SSL
# 4. Crear archivo .env con contraseñas

# 5. Levantar toda la infraestructura
docker compose up -d

# 6. Verificar estado de los contenedores
docker compose ps

# 7. Ver logs en tiempo real
docker compose logs -f

# 8. Acceder a WordPress
# → https://ejemplo.com (o https://localhost si es local)

# 9. Detener sin perder datos
docker compose down

# 10. Detener Y eliminar datos (¡CUIDADO!)
docker compose down -v
```


---

## Referencias

- [Nginx Load Balancer - Admin Guide](https://docs.nginx.com/nginx/admin-guide/load-balancer/)
- [Nginx HTTP Upstream Module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [Nginx HTTP Health Check](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-health-check/)
- [Nginx Rate Limiting Module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- [Nginx Logging Guide](https://edgedelta.com/company/knowledge-center/nginx-logging-guide)
- [Rate Limiting con Nginx - Protección contra fuerza bruta](https://www.cybrosys.com/blog/how-nginx-rate-limiting-shields-your-server-from-brute-force-attacks)
