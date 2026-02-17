## BLOQUE 1: FUNDAMENTOS WEB (10 preguntas)

### Pregunta 1
**¿Cuál es la principal diferencia arquitectural entre Apache MPM Prefork y Nginx?**

- A) Apache usa hilos, Nginx usa procesos
- B) Apache usa Event Loop, Nginx usa CGI
- C) Apache usa un proceso por conexión, Nginx usa Event-Driven asíncrono ✅
- D) No hay diferencia, ambos usan la misma arquitectura

**Explicación**: Apache MPM Prefork crea un proceso pesado por cada conexión (bloqueante), mientras que Nginx usa una arquitectura Event-Driven donde un worker puede manejar miles de conexiones simultáneas de forma asíncrona sin bloquear.

---

### Pregunta 2
**¿Qué puerto se usa por defecto para HTTPS?**

- A) 80
- B) 8080
- C) 443 ✅
- D) 22

**Explicación**: El puerto 443 es el estándar para conexiones HTTPS (HTTP sobre TLS/SSL). El puerto 80 es para HTTP sin cifrar.

---

### Pregunta 3
**En el modelo cliente-servidor, ¿qué protocolo de capa de aplicación se usa para transferir páginas web?**

- A) FTP
- B) SMTP
- C) HTTP/HTTPS ✅
- D) SSH

**Explicación**: HTTP (Hypertext Transfer Protocol) y su versión segura HTTPS son los protocolos de capa de aplicación diseñados específicamente para transferir documentos hipermedia (páginas web).

---

### Pregunta 4
**¿Qué código de estado HTTP indica que un recurso se ha movido permanentemente?**

- A) 200 OK
- B) 301 Moved Permanently ✅
- C) 404 Not Found
- D) 500 Internal Server Error

**Explicación**: El código 301 indica una redirección permanente. El navegador y los motores de búsqueda deben actualizar sus enlaces al nuevo URI. Se usa comúnmente para redirecciones HTTP→HTTPS.

---

### Pregunta 5
**¿Cuál es la principal ventaja de Nginx sobre Apache para sitios con alta concurrencia?**

- A) Mejor soporte para PHP
- B) Más módulos disponibles
- C) Menor uso de memoria con miles de conexiones simultáneas ✅
- D) Configuración más simple

**Explicación**: Nginx fue diseñado para el problema C10K (manejar 10,000 conexiones concurrentes). Su arquitectura Event-Driven consume mucha menos memoria que el modelo de procesos/hilos de Apache bajo alta concurrencia.

---

### Pregunta 6
**¿Qué significa que HTTP es un protocolo stateless?**

- A) Que no usa TCP
- B) Que cada petición es independiente y no guarda estado entre peticiones ✅
- C) Que no puede enviar datos
- D) Que solo funciona con GET

**Explicación**: HTTP es stateless porque cada request-response es independiente. El servidor no mantiene información sobre peticiones previas. Para mantener sesiones se usan cookies, tokens JWT, etc.

---

### Pregunta 7
**¿Qué método HTTP se debe usar para enviar datos de un formulario de forma segura (no visible en URL)?**

- A) GET
- B) POST ✅
- C) DELETE
- D) HEAD

**Explicación**: POST envía los datos en el body del request, no en la URL. GET pone los datos en la query string (visible en URL y logs), por lo que no debe usarse para datos sensibles como contraseñas.

---

### Pregunta 8
**¿Qué header HTTP se usa para indicar al navegador que cachee un recurso por 1 año?**

- A) `Cache-Control: no-cache`
- B) `Cache-Control: max-age=31536000` ✅
- C) `Expires: never`
- D) `Pragma: cache`

**Explicación**: `Cache-Control: max-age=31536000` indica que el recurso puede cachearse por 31,536,000 segundos (1 año). Es ideal para assets estáticos con versionado.

---

### Pregunta 9
**¿Qué es un reverse proxy?**

- A) Un servidor que oculta la identidad del cliente
- B) Un servidor que se sitúa entre clientes e Internet
- C) Un servidor que intercepta peticiones y las reenvía a backends ✅
- D) Un firewall de aplicación

**Explicación**: Un reverse proxy (como Nginx) se sitúa delante de los servidores backend, intercepta peticiones de clientes y las distribuye. Se usa para load balancing, SSL termination, caching, etc.

---

### Pregunta 10
**¿Cuál es el tamaño máximo por defecto de uploads en Nginx sin configuración adicional?**

- A) Sin límite
- B) 1MB ✅
- C) 10MB
- D) 100MB

**Explicación**: Por defecto, `client_max_body_size` es 1MB en Nginx. Para subir archivos más grandes hay que aumentar este valor explícitamente en la configuración.

---

## BLOQUE 2: DOCKER E INFRAESTRUCTURA (12 preguntas)

### Pregunta 11
**¿Cuál es la principal diferencia entre una imagen Docker y un contenedor?**

- A) No hay diferencia, son lo mismo
- B) La imagen es una plantilla inmutable, el contenedor es una instancia en ejecución ✅
- C) El contenedor es más grande que la imagen
- D) La imagen solo funciona en Linux

**Explicación**: Una imagen Docker es una plantilla de solo lectura (como una clase en POO), mientras que un contenedor es una instancia ejecutándose de esa imagen (como un objeto). Puedes crear múltiples contenedores de una misma imagen.

---

### Pregunta 12
**¿Por qué NO se debe usar la etiqueta `:latest` en producción?**

- A) Porque es muy lenta
- B) Porque puede cambiar sin previo aviso y romper la aplicación ✅
- C) Porque no existe
- D) Porque consume más recursos

**Explicación**: `:latest` apunta a la versión más reciente publicada, que puede cambiar en cualquier momento. Esto viola el principio de reproducibilidad. En producción siempre se debe pinar una versión específica (ej: `nginx:1.25.3-alpine`).

---

### Pregunta 13
**¿Qué hace la directiva `EXPOSE 80` en un Dockerfile?**

- A) Publica el puerto 80 en el host
- B) Documenta que el contenedor usa el puerto 80 internamente ✅
- C) Abre el puerto 80 en el firewall
- D) Mapea el puerto 80 al 8080 del host

**Explicación**: `EXPOSE` es solo documentación, NO publica ni abre puertos. Para publicar un puerto hay que usar `-p` en `docker run` o `ports:` en docker-compose.

---

### Pregunta 14
**¿Qué tipo de volumen Docker es gestionado completamente por Docker?**

- A) Bind mount
- B) Named volume ✅
- C) tmpfs mount
- D) Host volume

**Explicación**: Los named volumes son gestionados por Docker (se almacenan en `/var/lib/docker/volumes/`). Los bind mounts mapean directorios específicos del host, que el usuario gestiona manualmente.

---

### Pregunta 15
**¿Cuál es la ventaja de usar Alpine Linux como imagen base?**

- A) Mejor soporte de software
- B) Tamaño reducido (~5MB) y menor superficie de ataque ✅
- C) Más rápido en ejecución
- D) Compatible con Windows

**Explicación**: Alpine usa musl libc en lugar de glibc y solo incluye lo mínimo necesario. Esto resulta en imágenes mucho más pequeñas (nginx:alpine ~40MB vs nginx:latest ~190MB) con menos vulnerabilidades potenciales.

---

### Pregunta 16
**¿Qué network driver de Docker permite comunicación entre contenedores en diferentes hosts físicos?**

- A) bridge
- B) host
- C) overlay ✅
- D) none

**Explicación**: El driver `overlay` crea redes multi-host usando túneles VXLAN. Se usa en clusters Docker Swarm o Kubernetes. `bridge` solo funciona en un mismo host.

---

### Pregunta 17
**En Docker Compose, ¿qué hace `depends_on`?**

- A) Espera a que el servicio esté "healthy" antes de iniciar
- B) Define el orden de inicio de contenedores ✅
- C) Crea una dependencia de red
- D) Comparte volúmenes entre servicios

**Explicación**: `depends_on` controla el orden de inicio, pero NO espera a que el servicio esté listo (solo que el contenedor haya arrancado). Para esperar a que esté "healthy" se debe usar `condition: service_healthy`.

---

### Pregunta 18
**¿Qué diferencia hay entre `CMD` y `ENTRYPOINT` en un Dockerfile?**

- A) No hay diferencia
- B) CMD se puede sobrescribir fácilmente, ENTRYPOINT es más fijo ✅
- C) ENTRYPOINT solo funciona con Alpine
- D) CMD es obligatorio, ENTRYPOINT opcional

**Explicación**: `CMD` define argumentos por defecto que se sobrescriben fácilmente con `docker run imagen comando`. `ENTRYPOINT` define el ejecutable principal; para cambiarlo hay que usar `--entrypoint`.

---

### Pregunta 19
**¿Qué hace la instrucción `USER nginx` en un Dockerfile?**

- A) Crea el usuario nginx
- B) Cambia el usuario que ejecuta los comandos siguientes y el contenedor ✅
- C) Elimina permisos de root
- D) Configura autenticación HTTP

**Explicación**: `USER` cambia el contexto de usuario para las instrucciones subsiguientes en el Dockerfile y para el proceso que se ejecutará en el contenedor. Es una best practice de seguridad evitar ejecutar como root.

---

### Pregunta 20
**¿Cuál es el propósito de un multi-stage build en Docker?**

- A) Ejecutar múltiples contenedores
- B) Reducir el tamaño de la imagen final separando build de runtime ✅
- C) Crear backups automáticos
- D) Mejorar la velocidad de ejecución

**Explicación**: Multi-stage builds permiten usar una imagen pesada para compilar (con todas las herramientas de build) y luego copiar solo el artefacto final a una imagen pequeña de runtime, reduciendo drásticamente el tamaño.

---

### Pregunta 21
**¿Qué comando se usa para ver los logs de un contenedor llamado "web"?**

- A) `docker ps web`
- B) `docker logs web` ✅
- C) `docker inspect web`
- D) `docker exec web logs`

**Explicación**: `docker logs [container]` muestra los logs del contenedor (stdout/stderr). Añadir `-f` hace follow en tiempo real.

---

### Pregunta 22
**¿Qué tipo de montaje Docker NO persiste datos cuando el contenedor se elimina?**

- A) Named volume
- B) Bind mount
- C) tmpfs mount ✅
- D) Todos persisten datos

**Explicación**: `tmpfs` monta en RAM y se pierde cuando el contenedor se detiene. Se usa para datos temporales sensibles que no deben escribirse en disco.

---

## BLOQUE 3: CONFIGURACIÓN NGINX (10 preguntas)

### Pregunta 23
**¿En qué contexto de configuración de Nginx se define un upstream?**

- A) server {}
- B) location {}
- C) http {} ✅
- D) events {}

**Explicación**: Los bloques `upstream` se definen en el contexto `http {}`, fuera de server blocks, para poder ser referenciados por múltiples server blocks.

---

### Pregunta 24
**Si configuras `proxy_pass http://backend;` (sin trailing slash) en `location /api/`, ¿qué recibe el backend cuando llegas a `/api/users`?**

- A) `/users`
- B) `/api/users` ✅
- C) `/users/api`
- D) Error 404

**Explicación**: Sin URI en `proxy_pass` (sin trailing slash), se preserva la URI completa original incluyendo el path del location. CON trailing slash (`http://backend/`) se reemplazaría `/api/` con `/`.

---

### Pregunta 25
**¿Qué directiva Nginx se usa para ocultar el número de versión en la respuesta HTTP?**

- A) `hide_version on;`
- B) `server_tokens off;` ✅
- C) `version_hide true;`
- D) `expose_version false;`

**Explicación**: `server_tokens off;` en el contexto `http {}` evita que Nginx incluya su versión en el header `Server:`. Security by obscurity básica.

---

### Pregunta 26
**¿Para qué se usa la directiva `try_files $uri $uri/ =404;`?**

- A) Para hacer backup de archivos
- B) Para intentar servir un archivo, luego directorio, sino retornar 404 ✅
- C) Para cachear archivos
- D) Para comprimir archivos

**Explicación**: `try_files` intenta servir opciones en orden: primero el archivo exacto, luego como directorio, y si nada coincide retorna 404. Es común en SPAs para redirigir URIs no existentes a index.html.

---

### Pregunta 27
**¿Qué hace `add_header X-Frame-Options "SAMEORIGIN";`?**

- A) Permite incrustar el sitio en cualquier iframe
- B) Previene clickjacking permitiendo iframe solo del mismo origen ✅
- C) Oculta la versión de Nginx
- D) Habilita CORS

**Explicación**: `X-Frame-Options: SAMEORIGIN` permite que la página sea embebida en iframes solo si provienen del mismo dominio, previniendo ataques de clickjacking.

---

### Pregunta 28
**Si defines `location ~ \.php$ { }`, ¿qué tipo de matching estás usando?**

- A) Exacto
- B) Prefix
- C) Regex case-sensitive ✅
- D) Regex case-insensitive

**Explicación**: El operador `~` indica regex case-sensitive. `~*` sería case-insensitive, `=` matching exacto, y sin símbolo es prefix matching.

---

### Pregunta 29
**¿Cuál es el orden de prioridad de los location blocks en Nginx?**

- A) El orden en que aparecen en el archivo
- B) Alfabético
- C) Exacto > Regex > Prefix (longest match) ✅
- D) Aleatorio

**Explicación**: El orden de prioridad es: 1) `=` (exacto), 2) `^~` (prefix preferente), 3) `~` o `~*` (regex en orden de aparición), 4) prefix (longest match first).

---

### Pregunta 30
**¿Qué módulo de Nginx se usa INCORRECTAMENTE para proxy HTTP?**

- A) ngx_http_proxy_module
- B) ngx_http_fastcgi_module ✅
- C) ngx_http_upstream_module
- D) Todos son correctos

**Explicación**: `fastcgi_module` es para FastCGI (PHP-FPM), NO para proxy HTTP. Para HTTP se usa `proxy_pass`, para FastCGI se usa `fastcgi_pass`.

---

### Pregunta 31
**¿Qué variable Nginx contiene la IP del cliente original?**

- A) `$server_addr`
- B) `$remote_addr` ✅
- C) `$client_ip`
- D) `$origin`

**Explicación**: `$remote_addr` contiene la IP del cliente conectado directamente. Si hay proxies intermedios, la IP real puede estar en headers como `X-Real-IP` o `X-Forwarded-For`.

---

### Pregunta 32
**¿Qué hace `return 301 https://$server_name$request_uri;`?**

- A) Retorna código 301 como texto
- B) Redirecciona permanentemente a HTTPS preservando URI ✅
- C) Retorna error 301
- D) Cachea por 301 segundos

**Explicación**: `return 301 [URL]` hace una redirección permanente. `$server_name` es el dominio y `$request_uri` incluye el path y query string, así que `/about?id=5` se conserva en la redirección.

---

## BLOQUE 4: PROXY Y BALANCEO (10 preguntas)

### Pregunta 33
**¿Qué algoritmo de balanceo en Nginx permite "sticky sessions" (mismo cliente al mismo backend)?**

- A) round_robin
- B) least_conn
- C) ip_hash ✅
- D) random

**Explicación**: `ip_hash` calcula un hash de la IP del cliente y siempre lo envía al mismo backend (mientras esté disponible). Útil para aplicaciones con sesiones locales no compartidas.

---

### Pregunta 34
**En un upstream, ¿qué hace el parámetro `backup`?**

- A) Hace backup de los datos
- B) El servidor solo recibe tráfico si todos los demás fallan ✅
- C) Duplica las peticiones al servidor
- D) Cachea las respuestas

**Explicación**: Un servidor marcado como `backup` solo se usa cuando todos los servidores activos (no-backup) están down. Es el ultimo recurso.

---

### Pregunta 35
**¿Qué header proxy se usa para pasar la IP real del cliente al backend?**

- A) `X-Client-IP`
- B) `X-Real-IP` ✅
- C) `X-Remote-Address`
- D) `Client-IP`

**Explicación**: `proxy_set_header X-Real-IP $remote_addr;` es el estándar de facto para pasar la IP original. También existe `X-Forwarded-For` que mantiene una lista de todas las IPs en la cadena de proxies.

---

### Pregunta 36
**¿Qué parámetro de upstream define cuántos fallos consecutivos antes de marcar servidor como down?**

- A) `failures=3`
- B) `max_fails=3` ✅
- C) `fail_count=3`
- D) `retries=3`

**Explicación**: `max_fails` define el número de intentos fallidos antes de marcar el servidor como unhealthy por el tiempo especificado en `fail_timeout`.

---

### Pregunta 37
**¿Qué hace `proxy_next_upstream error timeout http_502;`?**

- A) Configura timeouts del proxy
- B) Define cuándo reintentar la petición en otro servidor del upstream ✅
- C) Oculta errores 502
- D) Cachea errores

**Explicación**: `proxy_next_upstream` define en qué condiciones Nginx debe reintentar la petición en otro backend del grupo upstream (error de conexión, timeout, código 502, etc.).

---

### Pregunta 38
**En un upstream con `weight=3` y `weight=1`, ¿qué proporción de tráfico recibe el primer servidor?**

- A) 30%
- B) 50%
- C) 75% ✅
- D) 100%

**Explicación**: El peso total es 3+1=4. El servidor con weight=3 recibe 3/4 = 75% del tráfico. Weighted round robin distribuye según estos pesos.

---

### Pregunta 39
**¿Cuál es la ventaja del algoritmo `least_conn` sobre round robin?**

- A) Más rápido
- B) Mejor para requests de duración variable ✅
- C) Sticky sessions
- D) Más simple de configurar

**Explicación**: `least_conn` envía cada nueva petición al servidor con menos conexiones activas. Es ideal cuando los requests tienen tiempos de procesamiento muy variables, evitando sobrecargar un servidor.

---

### Pregunta 40
**¿Qué header debe configurarse para que el backend sepa el hostname original?**

- A) `X-Hostname`
- B) `Host` ✅
- C) `X-Domain`
- D) `Server-Name`

**Explicación**: `proxy_set_header Host $host;` pasa el hostname original al backend. Sin esto, el backend vería el hostname interno (ej: "backend1:3000") y podría generar URLs incorrectas.

---

### Pregunta 41
**¿Qué directiva activa el algoritmo least connections en un upstream?**

- A) `algorithm least_conn;`
- B) `least_conn;` ✅
- C) `balancer least_connections;`
- D) `method leastconn;`

**Explicación**: Simplemente se añade `least_conn;` dentro del bloque `upstream {}`. Sin ninguna directiva, el default es round robin.

---

### Pregunta 42
**¿Por qué es útil `proxy_set_header X-Forwarded-Proto $scheme;`?**

- A) Para ocultar el protocolo al backend
- B) Para que el backend sepa si la conexión original fue HTTP o HTTPS ✅
- C) Para habilitar HTTP/2
- D) Para comprimir las respuestas

**Explicación**: Cuando Nginx termina SSL y se conecta al backend por HTTP, el backend necesita saber si el cliente original usó HTTPS. Esto es crucial para generar URLs correctas y políticas de cookies seguras.

---

## BLOQUE 5: SEGURIDAD Y HARDENING (10 preguntas)

### Pregunta 43
**¿Qué significa HSTS (HTTP Strict Transport Security)?**

- A) Sistema de autenticación HTTP
- B) Mecanismo que fuerza al navegador a usar solo HTTPS con ese dominio ✅
- C) Protocolo de cifrado
- D) Header para cachear contenido seguro

**Explicación**: HSTS le dice al navegador "recuerda usar HTTPS con este dominio por X tiempo". Después de la primera visita, el navegador convierte automáticamente cualquier intento HTTP a HTTPS antes de enviar el request.

---

### Pregunta 44
**¿Cuál es el valor recomendado mínimo para `max-age` en HSTS para producción?**

- A) 300 (5 minutos)
- B) 3600 (1 hora)
- C) 86400 (1 día)
- D) 31536000 (1 año) ✅

**Explicación**: Para estar en la HSTS Preload List de Google (hardcoded en navegadores) se requiere `max-age >= 31536000` (1 año). Esto garantiza protección a largo plazo contra downgrade attacks.

---

### Pregunta 45
**¿Qué hace `limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;`?**

- A) Limita el tamaño de requests a 10MB
- B) Crea zona de rate limiting: 5 peticiones por minuto por IP ✅
- C) Restringe access a 5 IPs
- D) Cachea por 5 minutos

**Explicación**: Define una zona de memoria compartida (10MB) para rate limiting basado en IP del cliente, permitiendo máximo 5 requests por minuto. Se aplica con `limit_req zone=login;` en los location blocks.

---

### Pregunta 46
**¿Qué diferencia hay entre `auth_basic` y autenticación mediante OAuth2/JWT?**

- A) No hay diferencia
- B) auth_basic es HTTP nativo pero inseguro sin HTTPS; OAuth2/JWT son más robustos ✅
- C) auth_basic es más seguro
- D) OAuth2 es más simple

**Explicación**: `auth_basic` envía credenciales en Base64 (reversible) en cada request. Es simple pero requiere HTTPS y no permite logout. OAuth2/JWT son protocolos más complejos con tokens, refresh, scopes, etc.

---

### Pregunta 47
**¿Qué TLS/SSL protocolo NO debería usarse en 2024+ por vulnerabilidades conocidas?**

- A) TLSv1.3
- B) TLSv1.2
- C) TLSv1.0 ✅
- D) Todos son seguros

**Explicación**: TLSv1.0 y TLSv1.1 están deprecados (POODLE, BEAST attacks). Solo TLSv1.2 y TLSv1.3 deben usarse. Configurar con: `ssl_protocols TLSv1.2 TLSv1.3;`

---

### Pregunta 48
**¿Qué hace `deny all;` después de reglas `allow`?**

- A) Bloquea solo IPs no permitidas explícitamente ✅
- B) Bloquea todas las IPs incluyendo las permitidas
- C) No hace nada
- D) Reinicia el servidor

**Explicación**: Las reglas `allow`/`deny` se evalúan en orden. Después de `allow IP1; allow IP2;`, un `deny all;` bloquea todo lo demás (whitelist). La primera coincidencia detiene la evaluación.

---

### Pregunta 49
**¿Para qué se usa el parámetro `burst` en rate limiting?**

- A) Para aumentar la velocidad
- B) Para permitir ráfagas temporales por encima del rate límite ✅
- C) Para bloquear bursts de tráfico
- D) Para comprimir datos

**Explicación**: `limit_req zone=login burst=5;` permite que hasta 5 requests excedan momentáneamente el rate límite antes de rechazar con 503. Útil para tráfico legítimo con picos.

---

### Pregunta 50
**¿Qué header de seguridad previene que navegadores interprete incorrectamente el MIME type?**

- A) `X-Frame-Options`
- B) `X-Content-Type-Options: nosniff` ✅
- C) `Content-Security-Policy`
- D) `Strict-Transport-Security`

**Explicación**: `X-Content-Type-Options: nosniff` evita que el navegador "adivine" el MIME type (MIME sniffing) y solo confíe en el Content-Type del servidor. Previene ciertos ataques XSS.

---

### Pregunta 51
**¿Qué hace `ssl_prefer_server_ciphers on;`?**

- A) Desactiva cifrado
- B) El servidor elige el cipher de la lista, no el cliente ✅
- C) Solo acepta ciphers del cliente
- D) Usa cifrado aleatorio

**Explicación**: Con `ssl_prefer_server_ciphers on;`, si hay coincidencias múltiples entre ciphers soportados por cliente y servidor, se usa el orden de preferencia del servidor (generalmente el más seguro).

---

### Pregunta 52
**¿Para qué se usan certificados SSL autofirmados?**

- A) Para producción en Internet
- B) Para desarrollo y testing local ✅
- C) Para máxima seguridad
- D) Son obligatorios

**Explicación**: Los certificados autofirmados NO son confiados por navegadores (generan warnings). Se usan solo en desarrollo/testing local. Para producción se debe usar Let's Encrypt (gratis) o CAs comerciales.

---