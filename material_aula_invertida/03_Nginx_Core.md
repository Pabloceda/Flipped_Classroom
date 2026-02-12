## MÓDULO 3: CONFIGURACIÓN DE NGINX

En el Módulo 2 aprendimos a desplegar Nginx en Docker con `docker run` o `docker-compose`. Pero un servidor web sin configurar solo muestra la página por defecto. En este módulo aprenderemos a **personalizar el comportamiento de Nginx** editando su archivo de configuración.

**¿Dónde está la configuración dentro del contenedor?**

```bash
# Archivo principal de configuración
/etc/nginx/nginx.conf

# Configuraciones adicionales (virtual hosts, sitios)
/etc/nginx/conf.d/*.conf

# Para editar la configuración, montamos archivos del host:
docker run -v ./mi-nginx.conf:/etc/nginx/nginx.conf:ro nginx:alpine
```

> Recuerda: los cambios dentro de un contenedor se pierden al eliminarlo. Por eso montamos la configuración como **volúmenes** (sección 2.5).

---

### 3.1 Anatomía de nginx.conf

#### Estructura General

```nginx
# Contexto Main (global)
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

# Contexto Events
events {
    worker_connections 1024;
    use epoll;
}

# Contexto HTTP
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Contexto Server (Virtual Host)
    server {
        listen 80;
        server_name ejemplo.com;
        
        # Contexto Location
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}

# Contexto Mail (opcional)
mail {
    # Configuración de proxy de mail
}

# Contexto Stream (TCP/UDP proxy)
stream {
    # Configuración de load balancer L4
}
```


#### Contextos Principales

**1. Main**: Configuración global

- Procesos worker
- Usuario de ejecución
- Logs globales

**2. Events**: Configuración del event loop

- Conexiones por worker
- Mecanismo de eventos (epoll/kqueue)

**3. HTTP**: Configuración del servidor web

- MIME types
- Logging
- Gzip
- Upstreams
- Servers

**4. Server**: Virtual host (un sitio/dominio)

- Puerto de escucha
- Nombre del servidor
- Locations

**5. Location**: Configuración de rutas específicas

- Matching de URI
- Proxy pass
- Headers
- Caching

---

### 3.2 Contextos y Jerarquía

#### Herencia de Directivas

```nginx
http {
    # Heredado por todos los servers
    gzip on;
    
    server {
        # Heredado por todos los locations de este server
        client_max_body_size 10M;
        
        location / {
            # Específico de esta location
            # Puede sobrescribir configuración heredada
            client_max_body_size 50M;
        }
    }
}
```

**Reglas de Herencia**:

1. Directivas de contexto externo se heredan en interno
2. Directivas de contexto interno sobrescriben externo
3. Algunas directivas NO son heredables (check docs)

#### Organización Modular

```nginx
# nginx.conf (principal)
http {
    include /etc/nginx/conf.d/*.conf;  # Incluye todos los .conf
}

# /etc/nginx/conf.d/api.conf
server {
    listen 80;
    server_name api.ejemplo.com;
    # ... configuración del API
}

# /etc/nginx/conf.d/web.conf
server {
    listen 80;
    server_name www.ejemplo.com;
    # ... configuración del sitio web
}
```


---

### 3.3 Directiva `server`

#### Definición

La directiva `server` define un **Virtual Host** (servidor virtual): configuración específica para un dominio/IP/puerto.

#### Sintaxis Básica

```nginx
server {
    listen 80;                    # Puerto de escucha
    server_name ejemplo.com;      # Nombre del servidor
    
    root /var/www/ejemplo;        # Directorio raíz
    index index.html;             # Archivo por defecto
    
    location / {
        # Configuración de rutas
    }
}
```


#### Múltiples Server Blocks

```nginx
# Sitio principal
server {
    listen 80;
    server_name ejemplo.com www.ejemplo.com;
    root /var/www/ejemplo;
}

# API
server {
    listen 80;
    server_name api.ejemplo.com;
    
    location / {
        proxy_pass http://localhost:3000;
    }
}

# Admin panel
server {
    listen 80;
    server_name admin.ejemplo.com;
    
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
    
    root /var/www/admin;
}
```


---

### 3.4 Directivas `listen` y `server_name`

#### Directiva `listen`

**Sintaxis**:

```nginx
listen [address][:port] [parameters];
```

**Ejemplos**:

```nginx
# Puerto 80 en todas las IPs
listen 80;

# Puerto específico en IP específica
listen 192.168.1.10:8080;

# IPv6
listen [::]:80;

# SSL/TLS
listen 443 ssl;
listen 443 ssl http2;          # Con HTTP/2
listen 443 quic reuseport;     # HTTP/3

# Default server
listen 80 default_server;

# Solo localhost
listen 127.0.0.1:80;
```

**Parámetros**:

- `default_server`: Este server procesa peticiones sin match de server_name
- `ssl`: Habilita SSL/TLS
- `http2`: Habilita HTTP/2
- `quic`: Habilita QUIC/HTTP3
- `reuseport`: Balanceo de carga entre workers (kernel)


#### Directiva `server_name`

**Sintaxis**:

```nginx
server_name nombre1 [nombre2 ...];
```

**Tipos de Matching**:

```nginx
# 1. Exacto
server_name ejemplo.com;

# 2. Wildcard al inicio
server_name *.ejemplo.com;

# 3. Wildcard al final
server_name ejemplo.*;

# 4. Expresión regular (con ~)
server_name ~^(?<subdomain>.+)\.ejemplo\.com$;

# 5. Múltiples nombres
server_name ejemplo.com www.ejemplo.com ejemplo.net;
```

**Orden de Precedencia** (Nginx busca en este orden):

1. Nombre exacto
2. Wildcard al inicio más largo (`*.ejemplo.com`)
3. Wildcard al final más largo (`ejemplo.*`)
4. Primera regex que haga match (orden en config)
5. Default server

#### Ejemplo: Cómo Nginx Decide qué Server Mostrar

```nginx
# Server 1
server {
    listen 80;
    server_name ejemplo.com;
    return 200 "Server 1: ejemplo.com\n";
}

# Server 2
server {
    listen 80;
    server_name *.ejemplo.com;
    return 200 "Server 2: *.ejemplo.com\n";
}

# Server 3
server {
    listen 80 default_server;
    return 200 "Server 3: default\n";
}
```

**Peticiones**:

```bash
curl -H "Host: ejemplo.com" http://localhost
# → Server 1 (match exacto)

curl -H "Host: api.ejemplo.com" http://localhost
# → Server 2 (wildcard)

curl -H "Host: noexiste.com" http://localhost
# → Server 3 (default)
```


---

### 3.5 Bloques `location`

#### Definición

La directiva `location` define cómo procesar peticiones basándose en el URI solicitado. Es el corazón del enrutamiento en Nginx.

#### Sintaxis

```nginx
location [modifier] uri {
    # configuración
}
```


#### Tipos de Matching (Modificadores)

**1. Sin modificador (Prefix match)**:

```nginx
location /api/ {
    # Matches: /api/, /api/users, /api/users/123
}
```

**2. `=` (Exact match)**:

```nginx
location = /login {
    # SOLO /login exacto, NO /login/ ni /login?query
}
```

**3. `~` (Regex case-sensitive)**:

```nginx
location ~ /case/ {
    # Coincidencia sensible a mayúsculas/minúsculas
}
```

**4. `~*` (Regex case-insensitive)**:

```nginx
location ~* \.(jpg|png|gif)$ {
    # Matches: .JPG, .jpg, .PNG, etc.
}
```

**5. `^~` (Prefix match, stop regex search)**:

```nginx
location ^~ /images/ {
    # Si coincide, NO busca en locations con regex
}
```


#### Orden de Precedencia (CRUCIAL)

Nginx evalúa locations en este orden:

```
1. = (exact)               ← Mayor prioridad
2. ^~ (prefix, no regex)
3. ~ y ~* (regex)          ← Orden de aparición en config
4. Prefix más largo
5. / (catch-all)           ← Menor prioridad
```


#### Ejemplo Completo de Precedencia

```nginx
server {
    listen 80;
    server_name ejemplo.com;
    
    # 1. Exact match (máxima prioridad)
    location = /exact {
        return 200 "1. Exact match\n";
    }
    
    # 2. Prefix match con stop regex
    location ^~ /images/ {
        return 200 "2. Prefix with stop regex\n";
    }
    
    # 3. Regex case-sensitive
    location ~ /case/ {
        return 200 "3. Regex case-sensitive\n";
    }
    
    # 4. Regex case-insensitive
    location ~* /images/.*\.png$ {
        return 200 "4. Regex case-insensitive\n";
    }
    
    # 5. Prefix match largo
    location /images/special/ {
        return 200 "5. Prefix long\n";
    }
    
    # 6. Prefix match corto
    location /images/ {
        return 200 "6. Prefix short\n";
    }
    
    # 7. Catch-all (menor prioridad)
    location / {
        return 200 "7. Catch-all\n";
    }
}
```

**Pruebas**:

```bash
curl ejemplo.com/exact
# → 1. Exact match

curl ejemplo.com/images/logo.png
# → 2. Prefix with stop regex (^~ impide búsqueda regex)

curl ejemplo.com/case/test
# → 3. Regex case-sensitive

curl ejemplo.com/other/test
# → 7. Catch-all
```


#### Casos de Uso Prácticos

**Archivos Estáticos**:

```nginx
# Imágenes con cache largo
location ~* \.(jpg|jpeg|png|gif|ico|webp)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    access_log off;
}

# CSS/JS con cache medio
location ~* \.(css|js)$ {
    expires 30d;
    add_header Cache-Control "public";
}
```

**API Proxy**:

```nginx
location /api/ {
    proxy_pass http://backend_api;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

**Bloquear Acceso**:

```nginx
# Bloquear archivos ocultos
location ~ /\.\ {
    deny all;
    return 404;
}

# Bloquear .git, .env, etc.
location ~* \.(git|env|htaccess)$ {
    deny all;
}
```

**Reescritura de URLs**:

```nginx
location /old-path/ {
    rewrite ^/old-path/(.*)$ /new-path/$1 permanent;
}
```


---

### 3.6 Directivas `root` vs `alias`

Ahora que sabemos cómo Nginx **enruta las peticiones** usando bloques `location` (sección 3.5), necesitamos indicarle **dónde buscar los archivos** en el sistema de ficheros. Para eso se usan `root` y `alias`.

#### `root`: Define directorio base

```nginx
location /images/ {
    root /var/www;
}
```

**Comportamiento**:

```
Petición: /images/foto.jpg
Archivo servido: /var/www/images/foto.jpg
                        ↑ URI completa se AÑADE a root
```


#### `alias`: Reemplaza la parte matched del URI

```nginx
location /images/ {
    alias /var/www/static/;  # Nota el trailing slash
}
```

**Comportamiento**:

```
Petición: /images/foto.jpg
Archivo servido: /var/www/static/foto.jpg
                         ↑ /images/ se REEMPLAZA por alias
```


#### Comparación Práctica

```nginx
# Ejemplo con root
location /assets/ {
    root /var/www;
}
# /assets/logo.png → /var/www/assets/logo.png

# Ejemplo con alias
location /assets/ {
    alias /var/www/static/;
}
# /assets/logo.png → /var/www/static/logo.png
```


#### Errores Comunes

**Error 1**: Olvidar trailing slash en alias

```nginx
location /images/ {
    alias /var/www/static;  # ← Falta el /
}
# /images/foto.jpg → /var/www/staticfoto.jpg (INCORRECTO)
```

**Correcto**:

```nginx
location /images/ {
    alias /var/www/static/;  # ← Con trailing slash
}
```

❌ **Error 2**: Usar alias con regex location sin captura

```nginx
location ~* \.(jpg|png)$ {
    alias /var/www/images/;  # No funciona bien
}
```

**Regla general**:

- Usa `root` cuando la estructura de directorios refleja la URI
- Usa `alias` cuando quieres mapear a una ubicación diferente

---

### 3.7 Directiva `index`

#### Definición

Ya sabemos cómo Nginx enruta (`location`) y dónde busca archivos (`root`/`alias`). Pero, **¿qué archivo sirve cuando el usuario pide un directorio?** Por ejemplo, si alguien visita `http://ejemplo.com/blog/`, ¿qué archivo carga?

La directiva `index` define los archivos que Nginx intentará servir cuando se solicita un directorio (sin archivo específico en la URI).

#### Sintaxis

```nginx
index file1 [file2 file3 ...];
```


#### Prioridad de Búsqueda

Nginx busca los archivos en el orden especificado y sirve el primero que encuentre.

```nginx
location / {
    root /var/www/html;
    index index.php index.html index.htm default.html;
}
```

**Comportamiento**:

```
Petición: http://ejemplo.com/blog/
           ↓
Búsqueda:
1. /var/www/html/blog/index.php   ← Busca primero
2. /var/www/html/blog/index.html  ← Si no existe #1
3. /var/www/html/blog/index.htm   ← Si no existe #2
4. /var/www/html/blog/default.html← Si no existe #3
5. Si nada existe → 403 Forbidden o autoindex
```


#### Ejemplo con PHP

```nginx
server {
    listen 80;
    server_name ejemplo.com;
    root /var/www/html;
    
    # Prioriza PHP sobre HTML
    index index.php index.html;
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
    }
}
```


#### Autoindex (Listado de Directorios)

```nginx
location /files/ {
    root /var/www;
    autoindex on;              # Habilita listado de directorios
    autoindex_exact_size off;  # Tamaño legible (KB, MB)
    autoindex_localtime on;    # Timestamps en hora local
}
```

**Resultado** (si no hay index.html):

```
Index of /files/
../
documento.pdf                 15-Jan-2026 14:30     2.5M
imagen.jpg                    12-Jan-2026 09:15     856K
script.sh                     10-Jan-2026 18:45     4.2K
```


---

### 3.8 La Directiva "Mágica": `try_files`

#### El Problema de las SPAs (Single Page Applications)
En el desarrollo moderno (React, Vue, Angular, Laravel), muchas veces tenemos una sola página de entrada (`index.html` o `index.php`) que maneja todas las rutas del navegador.

Si un usuario entra a `ejemplo.com/usuarios/123`, el archivo `/usuarios/123` **no existe físicamente** en el disco. Sin configuración, Nginx devolvería un **404 Not Found**.

#### Solución: Front Controller Pattern
La directiva `try_files` permite definir una **cadena de intentos** de búsqueda antes de devolver un error.

**Sintaxis**:
```nginx
try_files file1 file2 ... final_fallback;
```

**Configuración Estándar para SPAs**:
```nginx
location / {
    # 1. Busca el archivo exacto ($uri)
    # 2. Busca el directorio ($uri/)
    # 3. Si no existen, reenvía todo al index.html
    try_files $uri $uri/ /index.html;
}
```

**Explicación Visual**:
> "Nginx, busca si existe el archivo `foto.jpg`. Si no, busca si existe la carpeta `foto/`. Y si tampoco, no des un 404, pásale la 'patata caliente' al `index.html` para que React se encargue."

**Para PHP (Laravel/Symfony)**:
```nginx
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

---

### 3.9 Páginas de Error Personalizadas

#### Directiva `error_page`

**Sintaxis**:

```nginx
error_page code [code...] [=response_code] uri;
```


#### Ejemplos Básicos

```nginx
server {
    listen 80;
    server_name ejemplo.com;
    root /var/www/html;
    
    # Error 404 personalizado
    error_page 404 /custom_404.html;
    location = /custom_404.html {
        internal;  # Solo accesible vía error_page
    }
    
    # Error 500, 502, 503, 504 a misma página
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        internal;
    }
}
```


#### Cambiar Código de Respuesta

```nginx
# Mantener código original
error_page 404 /404.html;

# Cambiar código (ej: 404 → 200)
error_page 404 =200 /404.html;

# Usar código de la página servida
error_page 404 = /404.php;
```


#### Redireccionar a URL Externa

```nginx
error_page 404 =301 https://ejemplo.com/not-found;
```


#### Ejemplo Completo con JSON

```nginx
server {
    listen 80;
    server_name api.ejemplo.com;
    
    # Error 404 con JSON
    error_page 404 = @error404;
    location @error404 {
        default_type application/json;
        return 404 '{"error": "Resource not found", "code": 404}';
    }
    
    # Error 500+ con JSON
    error_page 500 502 503 504 = @error5xx;
    location @error5xx {
        default_type application/json;
        return 500 '{"error": "Internal server error", "code": 500}';
    }
}
```


#### Páginas de Error con Diseño

```html
<!-- /var/www/html/custom_404.html -->
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>404 - Página No Encontrada</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding-top: 100px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        h1 { font-size: 120px; margin: 0; }
        p { font-size: 24px; }
        a { color: #fff; text-decoration: underline; }
    </style>
</head>
<body>
    <h1>404</h1>
    <p>Página no encontrada</p>
    <a href="/">Volver al inicio</a>
</body>
</html>
```


---

### 3.10 Logs en Nginx

#### Tipos de Logs

**1. Access Log**: Registra todas las peticiones
**2. Error Log**: Registra errores y warnings

#### Directiva `access_log`

**Sintaxis**:

```nginx
access_log path [format [buffer=size] [gzip[=level]] [flush=time]];
```

**Ejemplos**:

```nginx
# Log estándar
access_log /var/log/nginx/access.log;

# Con formato custom
access_log /var/log/nginx/access.log custom_format;

# Con buffer (mejor rendimiento)
access_log /var/log/nginx/access.log buffer=32k;

# Con compresión
access_log /var/log/nginx/access.log gzip=5 flush=1m;

# Desactivar log
access_log off;
```


#### Directiva `error_log`

**Sintaxis**:

```nginx
error_log path [level];
```

**Niveles** (menor a mayor severidad):

- `debug`: Todo (muy verboso, solo con --with-debug)
- `info`: Información general
- `notice`: Eventos significativos
- `warn`: Warnings (default)
- `error`: Errores de procesamiento
- `crit`: Problemas críticos
- `alert`: Acción inmediata requerida
- `emerg`: Sistema inutilizable

**Ejemplos**:

```nginx
# Error log con nivel warn (default)
error_log /var/log/nginx/error.log warn;

# Error log con nivel debug (desarrollo)
error_log /var/log/nginx/debug.log debug;

# Múltiples error logs
error_log /var/log/nginx/error.log warn;
error_log /var/log/nginx/critical.log crit;
```


#### Formatos de Log

**Formato Combined (default)**:

```nginx
log_format combined '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
```

**Ejemplo de línea**:

```
192.168.1.10 - - [10/Feb/2026:20:30:45 +0100] "GET /index.html HTTP/1.1" 200 1234 "-" "Mozilla/5.0..."
```

**Formato JSON (recomendado para parsing)**:

```nginx
log_format json escape=json
    '{'
        '"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"request_method":"$request_method",'
        '"request_uri":"$request_uri",'
        '"status":$status,'
        '"body_bytes_sent":$body_bytes_sent,'
        '"http_referer":"$http_referer",'
        '"http_user_agent":"$http_user_agent",'
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time"'
    '}';

access_log /var/log/nginx/access.json json;
```

**Ejemplo de línea JSON**:

```json
{"time":"2026-02-10T20:30:45+01:00","remote_addr":"192.168.1.10","request_method":"GET","request_uri":"/api/users","status":200,"body_bytes_sent":2048,"http_referer":"-","http_user_agent":"curl/7.68.0","request_time":0.123,"upstream_response_time":"0.089"}
```


#### Variables Útiles en Logs

| Variable | Descripción |
| :-- | :-- |
| `$remote_addr` | IP del cliente |
| `$remote_user` | Usuario HTTP auth |
| `$time_local` | Timestamp |
| `$request` | Request line completa |
| `$status` | Código de estado HTTP |
| `$body_bytes_sent` | Bytes enviados (sin headers) |
| `$http_referer` | Referer header |
| `$http_user_agent` | User-Agent |
| `$request_time` | Tiempo de procesamiento (s) |
| `$upstream_response_time` | Tiempo de respuesta del backend |
| `$ssl_protocol` | Protocolo SSL/TLS usado |
| `$ssl_cipher` | Cipher usado |

#### Logs Condicionales

```nginx
# Solo loggear errores (4xx, 5xx)
map $status $loggable {
    ~^ 0;  # 2xx, 3xx → no log[^1]
    default 1; # 4xx, 5xx → log
}

access_log /var/log/nginx/errors_only.log combined if=$loggable;
```

```nginx
# No loggear bots
map $http_user_agent $log_ua {
    ~*(bot|crawler|spider) 0;
    default 1;
}

access_log /var/log/nginx/access.log combined if=$log_ua;
```


#### Rotación de Logs

**Usando logrotate** (Linux):

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily                    # Rotar diariamente
    missingok                # No error si log no existe
    rotate 14                # Mantener 14 días
    compress                 # Comprimir logs antiguos
    delaycompress            # Comprimir en próximo ciclo
    notifempty               # No rotar si está vacío
    create 0640 nginx nginx  # Permisos del nuevo archivo
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```


---

### 3.11 Recarga en Caliente (Graceful Reload)

#### Señales de Proceso

Nginx soporta señales POSIX para control sin downtime:


| Señal | Comando | Acción |
| :-- | :-- | :-- |
| `TERM, INT` | `nginx -s stop` | Stop rápido |
| `QUIT` | `nginx -s quit` | Graceful shutdown |
| `HUP` | `nginx -s reload` | Recarga configuración |
| `USR1` | `nginx -s reopen` | Reabrir logs |
| `USR2` | - | Upgrade binario sin downtime |

#### Recarga de Configuración (HUP)

**Proceso**:

1. Master lee nueva configuración
2. Verifica sintaxis
3. Si OK: inicia nuevos workers
4. Envía señal QUIT a workers antiguos
5. Workers antiguos terminan requests actuales y mueren
6. Nuevos workers aceptan nuevas conexiones

**Comandos**:

```bash
# Método 1: Con nginx binario
nginx -s reload

# Método 2: Con señal directa
kill -HUP $(cat /var/run/nginx.pid)

# Método 3: Con systemctl (systemd)
systemctl reload nginx
```


#### Test de Configuración

**Siempre testear ANTES de reload**:

```bash
# Test de sintaxis
nginx -t

# Output esperado:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful
```

**Script de reload seguro**:

```bash
#!/bin/bash
# safe-reload.sh

echo "Testing Nginx configuration..."
nginx -t

if [ $? -eq 0 ]; then
    echo "Configuration OK, reloading..."
    nginx -s reload
    echo "Reload completed successfully"
else
    echo "Configuration ERROR, aborting reload"
    exit 1
fi
```


#### Reabrir Logs (USR1)

Útil después de rotación de logs:

```bash
nginx -s reopen

# O con señal
kill -USR1 $(cat /var/run/nginx.pid)
```


#### Upgrade de Binario Sin Downtime (USR2)

**Proceso avanzado**:

```bash
# 1. Reemplazar binario
cp /usr/sbin/nginx /usr/sbin/nginx.old
cp /tmp/nginx-new /usr/sbin/nginx

# 2. Enviar USR2 al master antiguo
kill -USR2 $(cat /var/run/nginx.pid)
# Arranca nuevo master con workers nuevos

# 3. Graceful shutdown de workers antiguos
kill -QUIT $(cat /var/run/nginx.pid.oldbin)

# 4. Verificar
ps aux | grep nginx
```


---

## Referencias

- [Docker Bridge and Overlay Networks](https://uberconf.com/blog/arun_gupta/2015/12/docker_bridge_and_overlay_network_with_compose_variable_substitution)
- [Nginx Basics Workshops - GitHub](https://github.com/nginxinc/nginx-basics-workshops)
- [Despliegue de aplicación web con Nginx y Docker](https://josersosa.github.io/personalweb/posts/2021-05-11-despliegue-de-una-aplicacin-web-con-nginx-y-docker/)
- [Configurar Laravel, Nginx y MySQL con Docker Compose](https://www.digitalocean.com/community/tutorials/como-configurar-laravel-nginx-y-mysql-con-docker-compose-es)
- [Introducción Docker + Nginx](https://producthackers.com/es/blog/introduccion-docker-nginx)
- [Nginx Docker Tutorial - DataCamp](https://www.datacamp.com/es/tutorial/nginx-docker)
- [Docker Training](https://www.docker.com/trainings/)
