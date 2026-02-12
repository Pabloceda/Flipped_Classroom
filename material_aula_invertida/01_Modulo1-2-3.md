# Investigación Técnica: Despliegue de Servidores Web de Alto Rendimiento con Nginx y Docker

## Metadata de Investigación
- **Fecha**: 10 de febrero de 2026
- **Tema**: Implantación y Administración de Servidor Web con Docker
- **Enfoque**: Arquitectura interna, casos de uso reales, seguridad y alto rendimiento

---

## MÓDULO 1: FUNDAMENTOS DEL SERVICIO WEB

Antes de desplegar un servidor web o configurarlo, necesitamos entender **qué es**, **cómo se comunica con los clientes** y **por qué elegimos uno u otro**. Este módulo cubre los fundamentos: el protocolo HTTP, la evolución de sus versiones, el panorama actual de servidores web, y por qué Nginx se ha convertido en la opción dominante para sitios de alto tráfico.

---

### 1.1 ¿Qué es un Servidor Web?

Un **servidor web** tiene dos componentes:

- **Hardware**: La máquina física o virtual (CPU, RAM, disco, red) que aloja el servicio
- **Software (Daemon)**: El programa que escucha peticiones en los puertos 80 (HTTP) y 443 (HTTPS) y responde con páginas web, archivos o datos

Ejemplos de software servidor: **Nginx**, **Apache HTTP Server**, **IIS** (Microsoft), **LiteSpeed**.

#### Flujo básico: ¿Cómo funciona?

```
  Cliente (Navegador)                     Servidor Web
  ┌──────────────────┐                   ┌──────────────────┐
  │  Escribe URL:    │   1. Petición     │  Nginx/Apache    │
  │  ejemplo.com     │ ──────────────►   │  escucha en :80  │
  │                  │   HTTP GET /      │                  │
  │                  │                   │  2. Busca el     │
  │  4. Muestra la   │   3. Respuesta    │     archivo      │
  │     página       │ ◄──────────────   │     index.html   │
  │                  │   HTTP 200 OK     │                  │
  └──────────────────┘   + HTML          └──────────────────┘
```

**En resumen**: el navegador **pide** (petición HTTP), el servidor **busca** y **responde** (respuesta HTTP). Todo el protocolo HTTP que veremos a continuación define las reglas de esta conversación.

---

### 1.2 El Protocolo HTTP/S

#### Definición Técnica
HTTP (Hypertext Transfer Protocol) es un protocolo de capa de aplicación (Capa 7 OSI) basado en el modelo cliente-servidor que utiliza TCP como transporte.

#### Métodos HTTP Principales
- **GET**: Solicita un recurso (idempotente, cacheable)
- **POST**: Envía datos al servidor (no idempotente)
- **PUT**: Actualiza/crea un recurso (idempotente)
- **DELETE**: Elimina un recurso (idempotente)
- **HEAD**: Solicita headers sin body
- **OPTIONS**: Consulta métodos soportados
- **PATCH**: Modificación parcial

#### Códigos de Estado HTTP

**2xx - Éxito**
- `200 OK`: Petición exitosa
- `201 Created`: Recurso creado
- `204 No Content`: Éxito sin contenido de respuesta

**3xx - Redirección**
- `301 Moved Permanently`: Redirección permanente
- `302 Found`: Redirección temporal
- `304 Not Modified`: Recurso no modificado (caché válida)

**4xx - Error del Cliente**
- `400 Bad Request`: Sintaxis inválida
- `401 Unauthorized`: Autenticación requerida
- `403 Forbidden`: Sin permisos
- `404 Not Found`: Recurso no existe
- `429 Too Many Requests`: Rate limiting aplicado

**5xx - Error del Servidor**
- `500 Internal Server Error`: Error genérico del servidor
- `502 Bad Gateway`: Proxy recibió respuesta inválida del upstream
- `503 Service Unavailable`: Servidor temporalmente no disponible
- `504 Gateway Timeout`: Timeout del upstream

#### Cabeceras HTTP Críticas

**Cabeceras de Petición**:
```http
Host: ejemplo.com
User-Agent: Mozilla/5.0...
Accept: text/html,application/json
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Authorization: Bearer <token>
```

**Cabeceras de Respuesta**:

```http
Content-Type: application/json; charset=utf-8
Content-Length: 1234
Cache-Control: max-age=3600
Set-Cookie: session_id=abc123; Secure; HttpOnly
X-Frame-Options: DENY
```


---

### 1.3 Evolución del Protocolo HTTP

#### HTTP/1.1 (1997 - RFC 2616, actualizado RFC 7230-7235)

**Características**:

- **Keep-Alive (Persistent Connections)**: Reutilización de conexiones TCP
- **Pipelining**: Envío de múltiples peticiones sin esperar respuestas (poco usado por head-of-line blocking)
- **Chunked Transfer Encoding**: Permite transmisión sin conocer longitud total
- **Host Header**: Soporte para virtual hosts

**Limitaciones**:

- Una petición a la vez por conexión (sin multiplexación)
- Headers en texto plano (overhead)
- Head-of-Line Blocking en TCP

**Implementación Nginx**:

```nginx
http {
    keepalive_timeout 65;      # 65 segundos de Keep-Alive
    keepalive_requests 100;    # Máximo 100 peticiones por conexión
}
```


---

#### HTTP/2 (2015 - RFC 7540)

**Características Clave**:

1. **Multiplexación Binaria**:
    - Múltiples streams en una única conexión TCP
    - Frame-based protocol (binario, no texto)
    - Elimina head-of-line blocking a nivel de aplicación
2. **Compresión de Headers (HPACK)**:
    - Reduce overhead de headers repetitivos
    - Tabla de indexación dinámica
3. **Server Push**:
    - Servidor envía recursos proactivamente
    - Reduce latencia en recursos críticos
4. **Priorización de Streams**:
    - Dependencias y pesos para optimizar carga

**Limitaciones**:

- Sigue usando TCP: head-of-line blocking a nivel de transporte
- Si se pierde un paquete TCP, todos los streams se bloquean

**Configuración Nginx HTTP/2**:

```nginx
server {
    listen 443 ssl http2;
    ssl_certificate /etc/ssl/certs/cert.pem;
    ssl_certificate_key /etc/ssl/private/key.pem;
    
    # Server push
    http2_push /css/styles.css;
    http2_push /js/app.js;
}
```


---

#### HTTP/3 (2022 - RFC 9114) + QUIC

**Definición**: HTTP/3 es el mapeo de semántica HTTP sobre el protocolo de transporte QUIC (RFC 9000), que usa UDP en lugar de TCP.

**QUIC (Quick UDP Internet Connections)**:

- **Capa de transporte sobre UDP** (puerto 443)
- **TLS 1.3 integrado nativamente** en el transporte
- **Multiplexación sin head-of-line blocking**: pérdida de paquete afecta solo a un stream
- **Connection migration**: cambio de IP/red sin reiniciar conexión (identificadores de conexión)

**Ventajas HTTP/3**:

1. **Handshake más rápido**:
    - HTTP/2 sobre TCP+TLS: 2-3 RTT (Round Trip Time)
    - HTTP/3 sobre QUIC: 1 RTT (0-RTT para conexiones repetidas)
2. **Eliminación de head-of-line blocking**:
    - Pérdida de paquetes no bloquea otros streams
3. **Mejor rendimiento en redes móviles**:
    - Connection migration al cambiar de WiFi a 4G/5G

**Stack Comparison**:

```
HTTP/2 Stack:          HTTP/3 Stack:
┌──────────────┐      ┌──────────────┐
│    HTTP/2    │      │    HTTP/3    │
├──────────────┤      ├──────────────┤
│  TLS 1.2/1.3 │      │     QUIC     │ ← TLS 1.3 integrado
├──────────────┤      │  (includes   │
│     TCP      │      │  encryption) │
├──────────────┤      ├──────────────┤
│      IP      │      │     UDP      │
└──────────────┘      ├──────────────┤
                      │      IP      │
                      └──────────────┘
```

**Configuración Nginx HTTP/3 (Nginx 1.25+)**:

```nginx
server {
    listen 443 quic reuseport;          # QUIC/HTTP3
    listen 443 ssl http2;               # Fallback HTTP/2
    
    ssl_protocols TLSv1.3;              # TLS 1.3 obligatorio
    ssl_certificate /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;
    
    # Anunciar HTTP/3 vía header Alt-Svc
    add_header Alt-Svc 'h3=":443"; ma=86400';
    
    quic_retry on;                      # Protección anti-DDoS
    ssl_early_data on;                  # 0-RTT resumption
}
```

**Adopción 2026**:

- Chrome, Firefox, Edge, Safari soportan HTTP/3
- ~40% del tráfico web global usa HTTP/3 (Cloudflare, Google, Facebook)

---

### 1.4 Mercado de Servidores Web (2025-2026)

#### Datos de Netcraft (Mayo 2025)

| Servidor | Mayo 2025 | % Market Share | Cambio |
| :-- | :-- | :-- | :-- |
| **Nginx** | 259,608,513 | **21.15%** | +0.37pp |
| **Apache** | 187,533,263 | **15.28%** | -0.15pp |
| **Cloudflare** | 197,436,000 | ~16% | +0.04pp |
| **LiteSpeed** | 57,000,000 | ~4.6% | Creciendo |

#### Sitios de Alto Tráfico (Top 1M)

| Servidor | Mayo 2025 | % Share |
| :-- | :-- | :-- |
| **Nginx** | 5,516,359 | 40.95% |
| **Apache** | 3,069,468 | 22.79% |

**Tendencias**:

- Nginx domina en sitios de **alto tráfico** (>1M visitas/mes)
- Apache mantiene presencia en hosting compartido y aplicaciones legacy
- Cloudflare Server (fork de Nginx) gana terreno en CDN
- LiteSpeed crece en hosting web por soporte nativo de HTTP/3 y Quic

**Por qué Nginx domina el alto tráfico**:

1. Arquitectura event-driven (mayor concurrencia)
2. Menor footprint de memoria (~2.5MB vs ~10MB Apache)
3. Proxy reverso más eficiente
4. Mejor rendimiento sirviendo contenido estático

---

### 1.5 Arquitectura Nginx: Modelo Event-Driven

#### Definición Técnica

Nginx utiliza una **arquitectura asíncrona, no-bloqueante, basada en eventos** (event-driven) en lugar del modelo tradicional de un proceso/thread por conexión.

#### Comparación: Apache vs Nginx

**Apache (MPM Prefork/Worker)**:

```
┌─────────────────────────────────────┐
│      Apache Master Process          │
└────────────┬────────────────────────┘
             │
    ┌────────┴────────┐
    │  Worker         │  Worker         │  Worker
    │  Process/Thread │  Process/Thread │  Process/Thread
    │  ↓              │  ↓              │  ↓
    │  Connection 1   │  Connection 2   │  Connection 3
    └─────────────────┴─────────────────┴─────────────────┘
```

- **1 thread/proceso = 1 conexión**
- Consume ~3-5MB RAM por worker
- Context switching costoso con miles de conexiones

**Nginx (Event-Driven)**:

```
┌──────────────────────────────────────────────┐
│         Master Process                       │
│  (gestión, configuración, señales)           │
└────────────┬─────────────────────────────────┘
             │
    ┌────────┴──────────┬───────────────┐
    │  Worker 1         │  Worker 2     │  Worker N
    │  (Event Loop)     │  (Event Loop) │  (Event Loop)
    │  ↓                │               │
    │  ┌─────────────┐  │               │
    │  │ Connections │  │               │
    │  │ [1..1000+]  │  │               │
    │  └─────────────┘  │               │
    └───────────────────┴───────────────┴─────────┘
```

- **1 worker = miles de conexiones**
- Worker single-threaded, pinned a CPU
- Event loop con epoll/kqueue (Linux/BSD)


#### Funcionamiento del Event Loop

**Analogía**: Imagina un **recepcionista de hotel**:

- **Modelo Apache** (1 recepcionista por cliente): Cada huésped tiene su propio recepcionista asignado. Si un recepcionista espera a que alguien rellene un formulario, no puede atender a nadie más. Con 1.000 huéspedes, necesitas 1.000 recepcionistas → costoso.

- **Modelo Nginx** (1 recepcionista para todos): Un solo recepcionista atiende a muchos huéspedes. Mientras uno rellena el formulario, atiende a otro. Cuando el primero termina, vuelve a él. Con 1.000 huéspedes, 1 recepcionista eficiente es suficiente → rápido y económico.

Esto es exactamente lo que hace el **event loop** de Nginx: un worker atiende miles de conexiones alternando entre ellas según quién tenga datos listos para procesar.

**Ciclo del Event Loop (simplificado)**:

```
┌─────────────────────────────────────────────────┐
│  while (servidor encendido):                    │
│                                                 │
│    1. Esperar eventos (¿alguien tiene datos?)   │
│                          ↓                      │
│    2. Para cada conexión con datos listos:       │
│       - Si datos del cliente → leer petición    │
│       - Si respuesta del backend → reenviar     │
│       - Si listo para escribir → enviar HTML    │
│                          ↓                      │
│    3. Volver a 1 (nunca se bloquea)             │
└─────────────────────────────────────────────────┘
```

**Flujo de una petición HTTP**:

```
Leer petición → Procesar → Consultar backend (si hay) → 
Enviar respuesta → Mantener conexión o cerrar
```


#### Ventajas del Modelo Event-Driven

1. **Alta Concurrencia (C10k problem solved)**:
    - 10,000+ conexiones simultáneas con < 100MB RAM
    - Apache requiere GB de RAM para lo mismo
2. **Bajo consumo de CPU**:
    - Sin context switching entre threads
    - Worker pinned a CPU core
3. **Predecibilidad**:
    - Consumo de recursos lineal con el número de workers (no con conexiones)
4. **Eficiencia en I/O**:
    - No bloquea en operaciones lentas (disco, upstream)

#### Configuración Nginx Workers

```nginx
# Nginx.conf
worker_processes auto;  # 1 worker por CPU core
worker_cpu_affinity auto;  # Pin workers a CPU cores específicos

events {
    worker_connections 4096;  # Máx conexiones por worker
    use epoll;                # Event mechanism (Linux)
    multi_accept on;          # Acepta múltiples conexiones a la vez
}
```

**Cálculo de conexiones máximas**:

```
Max Connections = worker_processes × worker_connections
```

Ejemplo: 4 cores × 4096 conexiones = 16,384 conexiones simultáneas

---

### 1.6 Ventajas de Nginx

#### 1. Rendimiento (Performance)

**Benchmarks típicos**:

- **Contenido estático**: 50,000-100,000 req/s en hardware moderno
- **Proxy reverso**: 10,000-30,000 req/s dependiendo de backend
- **Latencia**: sub-milisegundo para recursos estáticos en caché

**Comparación con Apache**:

- Nginx: 2x-4x más rápido sirviendo estáticos
- Nginx: 3x-5x mayor throughput en proxy reverso


#### 2. Bajo Consumo de RAM (Memory Footprint)

**Footprint típico**:

- Master process: ~2-3 MB
- Worker process: ~2-5 MB base + (conexiones × ~1KB)
- Total típico: 50-200 MB para 10,000 conexiones

**Apache equivalente**: 500-1500 MB para la misma carga

#### 3. Escalabilidad Horizontal y Vertical

**Vertical**: Añadir más cores = añadir más workers (lineal)
**Horizontal**: Múltiples instancias detrás de load balancer

#### 4. Versatilidad

- Servidor web estático
- Proxy reverso (HTTP, WebSocket, gRPC)
- Load balancer (L7)
- Proxy de caché
- Terminación SSL/TLS


#### 5. Configuración Declarativa

Archivos de configuración legibles y modularizables (no requiere recompilación).

---

## MÓDULO 2: DOCKERIZACIÓN DEL SERVICIO

### 2.1 ¿Por qué Docker en Servidores Web?

#### Definición Técnica: Docker

Docker es una plataforma de **containerización** que utiliza tecnologías de aislamiento del kernel de Linux (namespaces, cgroups, chroot) para ejecutar aplicaciones en entornos aislados llamados contenedores.

#### Principios Clave

**1. Inmutabilidad**:

- Imagen inmutable (read-only layers)
- Deployment reproducible
- "Build once, run anywhere"

**2. Portabilidad**:

- Misma imagen en dev, staging, production
- Independiente del host OS (siempre que tenga Docker Engine)

**3. Aislamiento**:

- Namespaces: PID, network, mount, IPC, UTS, user
- Recursos limitados por cgroups
- Filesystem separado (overlay2/devicemapper)


#### Ventajas en Servidores Web

**Desarrollo**:

- Entorno consistente entre desarrolladores
- Setup rápido (`docker-compose up`)

**Operaciones**:

- Despliegue rápido (segundos vs minutos)
- Rollback trivial (cambiar tag de imagen)
- Escalado horizontal simple (docker scale)

**Seguridad**:

- Aislamiento de procesos
- Capability dropping (principio de mínimo privilegio)
- Imágenes escaneables (Trivy, Clair)

#### Contenedores vs Máquinas Virtuales

```
Máquina Virtual:                    Contenedor Docker:
┌────────────────────────┐          ┌────────────────────────┐
│   App A    │   App B   │          │   App A    │   App B   │
├────────────┼───────────┤          ├────────────┼───────────┤
│  Libs/Deps │ Libs/Deps │          │  Libs/Deps │ Libs/Deps │
├────────────┼───────────┤          ├────────────┴───────────┤
│  Guest OS  │ Guest OS  │          │   Docker Engine        │
├────────────┴───────────┤          ├────────────────────────┤
│      Hypervisor        │          │      Host OS (Linux)   │
├────────────────────────┤          ├────────────────────────┤
│      Host OS           │          │      Hardware          │
├────────────────────────┤          └────────────────────────┘
│      Hardware          │
└────────────────────────┘
  ~GB por VM, minutos            ~MB por contenedor, segundos
```

| Característica | Máquina Virtual | Contenedor |
|:--------------|:----------------|:-----------|
| **Arranque** | Minutos | Segundos |
| **Tamaño** | GBs (OS completo) | MBs (solo app + deps) |
| **Aislamiento** | Total (hardware virtual) | Procesos (namespaces) |
| **Rendimiento** | Overhead del hypervisor | Casi nativo |
| **Portabilidad** | Imagen pesada | Imagen ligera, compartible |
| **Uso típico** | Múltiples OS, aislamiento fuerte | Microservicios, CI/CD, escalado |

---

### 2.2 Imágenes Docker

#### ¿Qué es una Imagen Docker?

Una **imagen Docker** es una plantilla de solo lectura que contiene todo lo necesario para ejecutar una aplicación: código, runtime, librerías, variables de entorno y archivos de configuración. Es como una "instantánea" del sistema de archivos.

```
Imagen Docker (capas de solo lectura):
┌─────────────────────────────────┐
│  Capa 4: CMD nginx -g daemon   │  ← Instrucción de arranque
├─────────────────────────────────┤
│  Capa 3: COPY nginx.conf       │  ← Configuración
├─────────────────────────────────┤
│  Capa 2: RUN apk add nginx     │  ← Software instalado
├─────────────────────────────────┤
│  Capa 1: Alpine Linux 3.19     │  ← Sistema operativo base
└─────────────────────────────────┘
```

**Conceptos clave**:
- **Imagen ≠ Contenedor**: La imagen es la plantilla (clase); el contenedor es una instancia en ejecución (objeto)
- **Capas (Layers)**: Cada instrucción del Dockerfile crea una capa. Las capas se reutilizan entre imágenes, ahorrando espacio
- **Inmutable**: Una vez construida, la imagen no cambia. Los cambios se hacen en una capa de escritura temporal del contenedor
- **Registro (Registry)**: Las imágenes se almacenan en registros como **Docker Hub**, GitHub Container Registry o registros privados


#### Descargar y Gestionar Imágenes

```bash
# Descargar una imagen del registro (Docker Hub por defecto)
docker pull nginx:alpine

# Listar imágenes locales
docker images

# Ejemplo de salida:
# REPOSITORY    TAG            IMAGE ID       SIZE
# nginx         alpine         a2c3e5d8f1b2   42MB
# nginx         latest         b7d4e9f2a3c1   190MB
# postgres      16-alpine      c8e5f1d3b2a4   85MB

# Inspeccionar detalles de una imagen
docker inspect nginx:alpine

# Eliminar imagen
docker rmi nginx:alpine
```


#### Etiquetas (Tags): Versiones de una Imagen

Cada imagen tiene múltiples **etiquetas** (tags) que representan variantes o versiones. La nomenclatura sigue el formato:

```
repositorio:etiqueta
    │            │
    │            └── Variante o versión (ej: alpine, 1.25, latest)
    └── Nombre de la imagen (ej: nginx, httpd, postgres)
```

**Ejemplo real con nginx en Docker Hub**:

```bash
nginx:latest            # Última versión, base Debian (⚠️ cambia sin aviso)
nginx:1.25              # Versión mayor pinada
nginx:1.25.3            # Versión exacta pinada (más seguro)
nginx:alpine            # Última versión, base Alpine
nginx:1.25-alpine       # Versión + Alpine
nginx:1.25.3-alpine3.19 # Versión exacta + Alpine exacto (máximo control)
```

> Si no se indica una etiqueta, Docker asume **`:latest`** por defecto:
> `docker pull nginx` equivale a `docker pull nginx:latest`


#### ⚠️ El Problema de `:latest`

La etiqueta `latest` **no significa "estable"**, significa "la última versión publicada". Esto puede romper proyectos en producción:

```bash
# Hoy: nginx:latest → versión 1.25.3
docker pull nginx:latest

# Mañana: nginx:latest → versión 1.27.0 (breaking changes!)
docker pull nginx:latest
# → Tu aplicación puede fallar por incompatibilidades
```

**Regla de oro**: Siempre pinar la versión en producción.

```dockerfile
# ❌ MAL — No sabemos qué versión se instalará
FROM nginx:latest

# ⚠️ MEJORABLE — Pinamos la versión mayor pero no la menor
FROM nginx:1.25-alpine

# ✅ BIEN — Versión exacta, reproducible
FROM nginx:1.25.3-alpine3.19
```

| Etiqueta | Estabilidad | Ejemplo | Cuándo usar |
|:---------|:----------:|:--------|:------------|
| `latest` | ❌ Baja | `nginx:latest` | Nunca en producción, solo pruebas rápidas |
| `major` | ⚠️ Media | `nginx:1` | Desarrollo, si aceptas parches automáticos |
| `major.minor` | ✅ Alta | `nginx:1.25-alpine` | Desarrollo y staging |
| `major.minor.patch` | ✅✅ Máxima | `nginx:1.25.3-alpine3.19` | **Producción** |


#### Imágenes de Servidores Web

En el Módulo 1 vimos que **Nginx** y **Apache** son los servidores web más usados. Ambos tienen imágenes oficiales en Docker Hub listas para usar:

| Imagen | Tamaño | Base | Uso principal |
|:-------|:------:|:-----|:--------------|
| `nginx:alpine` | ~42 MB | Alpine Linux | Proxy inverso, estáticos, alto rendimiento |
| `nginx:latest` | ~190 MB | Debian | Compatibilidad completa |
| `httpd:alpine` | ~55 MB | Alpine Linux | Apache HTTP Server |
| `httpd:latest` | ~175 MB | Debian | Apache con módulos completos |

```bash
# Probar el servidor web en un solo comando
docker run -d --name mi_web -p 8080:80 nginx:alpine

# Acceder al servidor
curl http://localhost:8080
# → Welcome to nginx!

# Limpiar
docker stop mi_web && docker rm mi_web
```

> Con un solo comando hemos descargado un servidor web, lo hemos ejecutado y está sirviendo páginas. Esto es el poder de Docker.


#### nginx:alpine en Detalle

**nginx:alpine** es una imagen oficial basada en Alpine Linux (distribución minimalista).

**Comparación de variantes nginx**:

```bash
# Tamaños
nginx:latest       → ~190 MB (basada en Debian)
nginx:alpine       → ~42 MB  (basada en Alpine)
nginx:alpine-slim  → ~11 MB  (solo Nginx, sin extras)
```

**Ventajas de Alpine**:

1. **Superficie de ataque reducida**: Menos paquetes = menos vulnerabilidades
2. **Menor tamaño**: Transferencias más rápidas, menos uso de disco
3. **musl libc** en lugar de glibc (más ligero)
4. **apk**: Gestor de paquetes minimalista

**Desventajas**:

- Incompatibilidades con software que asume glibc
- Debugging más difícil (menos herramientas incluidas)


#### Dockerfile de nginx:alpine (simplificado)

```dockerfile
FROM alpine:3.19

# Instalar Nginx
RUN apk add --no-cache nginx

# Configuración
COPY nginx.conf /etc/nginx/nginx.conf

# Exponer puerto (solo documentación)
EXPOSE 80

# Usuario no-root (seguridad)
RUN adduser -D -u 1000 nginx && \
    chown -R nginx:nginx /var/lib/nginx

USER nginx

# Comando por defecto
CMD ["nginx", "-g", "daemon off;"]
```


#### Mejores Prácticas de Seguridad (2026)

**1. Imágenes Mínimas**:

```dockerfile
FROM nginx:alpine  # Preferir alpine sobre debian
```

**2. Usuario no-root**:

```dockerfile
USER nginx  # Nunca ejecutar como root
```

**3. Dropped Capabilities**:

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx:alpine
```

**4. Read-only Filesystem**:

```bash
docker run --read-only --tmpfs /tmp --tmpfs /var/cache/nginx nginx:alpine
```

**5. Escaneo de Vulnerabilidades**:

```bash
trivy image nginx:alpine
```

**6. Pin de Versiones**:

```dockerfile
FROM nginx:1.25.3-alpine3.19  # NO usar :latest
```

**7. Multi-stage Builds** (para builds custom):

```dockerfile
# Stage 1: Build
FROM alpine:3.19 AS builder
RUN apk add nginx-module-njs

# Stage 2: Runtime
FROM nginx:alpine
COPY --from=builder /usr/lib/nginx/modules /usr/lib/nginx/modules
```



### 2.3 Ciclo de Vida del Contenedor

#### Estados del Contenedor

```
Created → Running → Paused → Stopped → Deleted
```


#### Comandos Fundamentales

**1. `docker create`**: Crea contenedor sin iniciarlo

```bash
docker create --name web nginx:alpine
# Crea filesystem, configura namespaces, pero no inicia proceso
```

**2. `docker start`**: Inicia contenedor creado

```bash
docker start web
# Ejecuta CMD definido en Dockerfile
```

**3. `docker run`**: create + start en un comando

```bash
docker run -d --name web -p 80:80 nginx:alpine
# -d: detached mode
# --name: nombre del contenedor
# -p: port mapping
```

**4. `docker stop`**: Para contenedor (SIGTERM → SIGKILL después de 10s)

```bash
docker stop web
```

**5. `docker rm`**: Elimina contenedor

```bash
docker rm web
# Requiere que esté stopped o -f para forzar
```

**6. `docker exec`**: Ejecuta comando en contenedor running

```bash
docker exec -it web sh  # Shell interactiva
docker exec web nginx -t  # Test configuración
```


#### Logs y Debugging

```bash
# Ver logs
docker logs web
docker logs -f web  # Follow

# Inspeccionar contenedor
docker inspect web

# Stats en tiempo real
docker stats web

# Listar procesos
docker top web
```


---
### 2.4 Gestión de Puertos

#### EXPOSE vs -p (Puerto Mapping)

**EXPOSE en Dockerfile**:

```dockerfile
EXPOSE 80
```

- **Documentación**: Indica qué puerto usa el contenedor
- **No publica** el puerto (no accesible desde host)
- Útil para `docker run -P` (publish all exposed)

**-p en docker run (Port Mapping)**:

```bash
docker run -p 8080:80 nginx:alpine
#          ↑    ↑
#       host  container
```

- **Publica** el puerto: accesible desde host
- Crea regla iptables: `host:8080 → container:80`


#### Formatos de Port Mapping

```bash
# Básico
-p 80:80                # host:80 → container:80

# Host específico
-p 127.0.0.1:80:80      # Solo localhost

# Puerto aleatorio en host
-p 80                   # Random high port → container:80

# Múltiples puertos
-p 80:80 -p 443:443

# Rango de puertos
-p 8000-8010:8000-8010

# UDP
-p 53:53/udp
```


#### Ejemplo Completo

```bash
# Sin publish: contenedor aislado
docker run --name web1 nginx:alpine
# Puerto 80 del contenedor NO accesible desde host

# Con publish: accesible desde host
docker run --name web2 -p 8080:80 nginx:alpine
curl http://localhost:8080  # Funciona

# Verificar port mapping
docker port web2
# 80/tcp -> 0.0.0.0:8080
```


#### IPTables (bajo el capó)

Cuando haces `-p 8080:80`, Docker crea reglas iptables:

```bash
iptables -t nat -L -n

# PREROUTING chain
DNAT tcp dpt:8080 to:172.17.0.2:80

# POSTROUTING chain
MASQUERADE tcp dpt:80
```


---
### 2.5 Volúmenes y Persistencia

#### Tipos de Montajes

**1. Bind Mounts**: Mapea directorio host → contenedor

```bash
docker run -v /host/path:/container/path nginx:alpine
```

**2. Named Volumes**: Docker gestiona ubicación

```bash
docker volume create nginx_data
docker run -v nginx_data:/usr/share/nginx/html nginx:alpine
```

**3. tmpfs**: Montaje en RAM (no persistente)

```bash
docker run --tmpfs /tmp nginx:alpine
```


#### Ejemplo: Servir Contenido Custom

**Bind mount**:

```bash
# Crear contenido en host
mkdir -p ~/web/html
echo "<h1>Hola Docker</h1>" > ~/web/html/index.html

# Montar en contenedor
docker run -d \
  --name web \
  -p 80:80 \
  -v ~/web/html:/usr/share/nginx/html:ro \  # :ro = read-only
  nginx:alpine

curl http://localhost
# <h1>Hola Docker</h1>
```

**Ventajas de volumes vs bind mounts**:

- Volumes: gestionados por Docker, backup/restore más fácil
- Volumes: funcionan en Windows/Mac (abstracción de filesystem)
- Bind mounts: desarrollo local (editas archivo, se refleja inmediatamente)


#### Persistencia de Logs

```bash
docker run -d \
  --name web \
  -v ~/web/logs:/var/log/nginx \
  nginx:alpine

# Los logs de Nginx se guardan en host
tail -f ~/web/logs/access.log
```


---
### 2.6 Networking en Docker

#### Tipos de Redes (Network Drivers)

**1. Bridge (Default)**:

- Red virtual privada en el host
- Contenedores se comunican entre sí
- NAT para acceso externo

```bash
docker network create my_bridge
docker run --network my_bridge nginx:alpine
```

**Arquitectura**:

```
Host
  ├─ docker0 (bridge interface)
  │    ├─ veth0 → Container 1 (172.17.0.2)
  │    ├─ veth1 → Container 2 (172.17.0.3)
  │    └─ veth2 → Container 3 (172.17.0.4)
  └─ eth0 (external)
```

**DNS Integrado**:

```bash
# Contenedores en misma red custom bridge se resuelven por nombre
docker run --network my_bridge --name web nginx:alpine
docker run --network my_bridge --name app alpine ping web  # Funciona
```

**2. Host**:

- Contenedor usa stack de red del host directamente
- No hay NAT ni aislamiento de red
- Máximo rendimiento (sin overhead de red virtual)

```bash
docker run --network host nginx:alpine
# Nginx escucha en puerto 80 del HOST directamente
```

**Uso**: Aplicaciones de alto rendimiento, monitorización de red

**3. Overlay**:

- Multi-host networking (Docker Swarm/Kubernetes)
- Túnel VXLAN entre hosts
- Service discovery integrado

```bash
docker network create --driver overlay --attachable my_overlay
```

**Arquitectura**:

```
Host 1                        Host 2
  ├─ Container A                ├─ Container C
  │  (10.0.1.2)                 │  (10.0.1.4)
  └─ VXLAN ←─────────────────→ └─ VXLAN
       (encapsulado en UDP)
```

**4. None**:

- Sin red (aislamiento completo)
- Útil para contenedores que solo procesan datos locales

```bash
docker run --network none alpine
```

**5. Macvlan**:

- Contenedor con su propia MAC address
- Aparece como dispositivo físico en la red
- Útil para legacy apps que esperan estar en red física


#### Comparación de Network Drivers

| Driver | Scope | Isolation | Performance | Use Case |
| :-- | :-- | :-- | :-- | :-- |
| Bridge | Single host | Alta | Media (NAT) | Apps locales, desarrollo |
| Host | Single host | Nula | Máxima | Servidores alto perf |
| Overlay | Multi-host | Alta | Media (túnel) | Clusters, microservicios |
| Macvlan | Single host | Media | Alta | Legacy apps |
| None | Single host | Total | N/A | Procesamiento aislado |


---

### 2.7 Anatomía del Dockerfile

#### ¿Qué es un Dockerfile?

Un **Dockerfile** es un archivo de texto con instrucciones secuenciales que Docker usa para **construir una imagen**. Cada instrucción crea una **capa** (layer) en el sistema de archivos de la imagen.

```
Dockerfile → docker build → Imagen → docker run → Contenedor
```


#### Instrucciones Esenciales

**1. `FROM`** — Imagen base (obligatoria, siempre primera)

```dockerfile
FROM nginx:1.25-alpine3.19    # Imagen base con versión pinada
FROM node:20-alpine AS builder  # Con alias para multi-stage
FROM scratch                    # Imagen vacía (binarios estáticos)
```

**2. `COPY`** — Copiar archivos del host al contenedor

```dockerfile
COPY nginx.conf /etc/nginx/nginx.conf
COPY ./html/ /usr/share/nginx/html/
COPY --chown=nginx:nginx . /app   # Copiar con permisos
```

**3. `ADD`** — Similar a COPY pero con extras (descompresión, URLs)

```dockerfile
ADD app.tar.gz /opt/             # Descomprime automáticamente
ADD https://example.com/config /etc/  # Descarga desde URL
```

> **Buena práctica**: Usar `COPY` siempre que sea posible. `ADD` solo cuando necesites descomprimir o descargar.

**4. `RUN`** — Ejecutar comandos durante la construcción

```dockerfile
# Múltiples comandos en una sola capa (mejor práctica)
RUN apk update && \
    apk add --no-cache curl openssl && \
    rm -rf /var/cache/apk/*

# Cada RUN crea una capa separada (evitar cuando sea posible)
RUN apk update
RUN apk add curl        # ← Dos capas innecesarias
```

**5. `EXPOSE`** — Documenta el puerto que usa la aplicación

```dockerfile
EXPOSE 80
EXPOSE 443
```

> **Importante**: `EXPOSE` NO publica el puerto. Es **documentación**. Para publicar se usa `-p 80:80` en `docker run`.

**6. `CMD`** — Comando por defecto al iniciar el contenedor

```dockerfile
CMD ["nginx", "-g", "daemon off;"]  # Forma exec (recomendada)
CMD nginx -g "daemon off;"          # Forma shell
```

**7. `ENTRYPOINT`** — Comando fijo (no sobreescribible fácilmente)

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]   # Argumentos por defecto

# docker run mi-imagen           → nginx -g "daemon off;"
# docker run mi-imagen -t        → nginx -t (solo test config)
```

| | `CMD` | `ENTRYPOINT` |
|:--|:------|:-------------|
| **Propósito** | Comando por defecto | Ejecutable principal |
| **Sobreescribir** | Fácil (`docker run img comando`) | Difícil (requiere `--entrypoint`) |
| **Uso típico** | Argumentos variables | Ejecutable fijo |

**8. `WORKDIR`** — Directorio de trabajo

```dockerfile
WORKDIR /app    # Crea el directorio si no existe
COPY . .        # Copia al /app
RUN npm install # Se ejecuta en /app
```

**9. `ENV`** — Variables de entorno (disponibles en build Y runtime)

```dockerfile
ENV NGINX_VERSION=1.25.3 \
    NGINX_PORT=80

# Uso en runtime: docker run -e NGINX_PORT=8080
```

**10. `ARG`** — Variables solo de build (NO disponibles en runtime)

```dockerfile
ARG NGINX_VERSION=1.25.3
FROM nginx:${NGINX_VERSION}-alpine
```

**11. `HEALTHCHECK`** — Verificación de salud del contenedor

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost/ || exit 1
```


#### Ejemplo Completo: Dockerfile para Nginx Personalizado

```dockerfile
# Dockerfile para servidor Nginx con configuración custom
FROM nginx:1.25-alpine3.19

# Metadata
LABEL maintainer="alumno@ejemplo.com"
LABEL description="Nginx con configuración personalizada para producción"
LABEL version="1.0"

# Instalar herramientas necesarias
RUN apk add --no-cache curl openssl

# Crear usuario no-root
RUN addgroup -S webgroup && adduser -S webuser -G webgroup

# Copiar configuración personalizada
COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/ /etc/nginx/conf.d/

# Copiar archivos web
COPY --chown=webuser:webgroup html/ /usr/share/nginx/html/

# Puerto documentado
EXPOSE 80 443

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost/ || exit 1

# Comando de inicio
CMD ["nginx", "-g", "daemon off;"]
```


#### Multi-Stage Build (Optimización Avanzada)

Un **multi-stage build** permite usar múltiples `FROM` para separar el entorno de compilación del de ejecución, reduciendo el tamaño final de la imagen.

```dockerfile
# ===== Stage 1: Build de la aplicación =====
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# ===== Stage 2: Imagen de producción (solo Nginx + build) =====
FROM nginx:1.25-alpine3.19

# Copiar SOLO el resultado del build (no el código fuente ni node_modules)
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# Resultado: Imagen final ~50MB en vez de ~500MB+
```

**Ventajas**:
- Imagen final NO contiene herramientas de compilación
- Menor superficie de ataque
- Despliegue más rápido (menos bytes a transferir)


#### El archivo `.dockerignore`

Similar a `.gitignore`, excluye archivos del contexto de build:

```dockerignore
# .dockerignore
node_modules
.git
.gitignore
*.md
docker-compose*.yml
.env
logs/
```

**Sin `.dockerignore`**: Docker envía TODO el directorio al daemon (lento, innecesario, riesgo de seguridad).


#### Construir y Ejecutar

```bash
# Construir imagen
docker build -t mi-nginx:1.0 .

# Construir con argumentos
docker build --build-arg NGINX_VERSION=1.25 -t mi-nginx .

# Ver capas de la imagen
docker history mi-nginx:1.0

# Ejecutar contenedor
docker run -d --name web -p 80:80 mi-nginx:1.0
```


#### Buenas Prácticas de Dockerfile

1. **PIN de versiones**: `FROM nginx:1.25-alpine3.19` (nunca `:latest`)
2. **Minimizar capas**: Unir `RUN` con `&&`
3. **Ordenar por frecuencia de cambio**: `COPY package.json` antes de `COPY . .`
4. **Limpiar caché**: `rm -rf /var/cache/apk/*` en la misma capa que `apk add`
5. **Usar `.dockerignore`**: Excluir todo lo innecesario
6. **Usuario no-root**: `USER nginx` al final
7. **HEALTHCHECK**: Siempre definir en producción


### 2.8 Docker Compose

#### Definición

Docker Compose es una herramienta para definir y ejecutar aplicaciones Docker multi-contenedor usando un archivo YAML declarativo.

#### Ventajas sobre Comandos Sueltos

```bash
# Sin Compose (manual)
docker network create webnet
docker volume create web_data
docker run -d --name web --network webnet -p 80:80 -v web_data:/data nginx
docker run -d --name app --network webnet myapp
docker run -d --name db --network webnet postgres

# Con Compose (declarativo)
docker-compose up -d
```


#### Estructura docker-compose.yml

```yaml
version: '3.9'  # Versión del formato

services:
  web:
    image: nginx:alpine
    container_name: web
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./logs:/var/log/nginx
    networks:
      - webnet
    restart: unless-stopped  # Reinicio automático
    
  app:
    build: ./app  # Construye desde Dockerfile
    depends_on:
      - db
    networks:
      - webnet
    environment:
      DB_HOST: db
      DB_PORT: 5432
  
  db:
    image: postgres:16-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - webnet
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

networks:
  webnet:
    driver: bridge

volumes:
  db_data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```


#### Comandos Compose

```bash
# Levantar servicios
docker-compose up -d

# Ver logs
docker-compose logs -f web

# Escalar servicio
docker-compose up -d --scale app=3

# Parar servicios
docker-compose stop

# Parar y eliminar contenedores
docker-compose down

# Rebuild de imágenes
docker-compose build
docker-compose up -d --build

# Ver estado
docker-compose ps
```



### 2.9 Estructura del Proyecto
#### Estructura de Carpetas Recomendada

```
proyecto-nginx-docker/
│
├── docker-compose.yml        # Orquestación
│
├── nginx/
│   ├── Dockerfile            # Build custom si necesario
│   ├── nginx.conf            # Configuración principal
│   └── conf.d/               # Configuraciones de sitios
│       ├── default.conf
│       ├── api.conf
│       └── ssl.conf
│
├── html/                     # Contenido estático
│   ├── index.html
│   ├── assets/
│   └── static/
│
├── logs/                     # Logs persistentes
│   ├── access.log
│   └── error.log
│
├── certs/                    # Certificados SSL
│   ├── cert.pem
│   └── key.pem
│
└── secrets/                  # Secretos (no subir a Git)
    └── .gitkeep
```


#### Ejemplo docker-compose.yml Completo

```yaml
version: '3.9'

services:
  nginx:
    image: nginx:1.25.3-alpine3.19  # Pin version
    container_name: nginx_web
    
    ports:
      - "80:80"
      - "443:443"
    
    volumes:
      # Configuración
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      
      # Contenido
      - ./html:/usr/share/nginx/html:ro
      
      # SSL
      - ./certs:/etc/nginx/certs:ro
      
      # Logs
      - ./logs:/var/log/nginx
      
      # Cache (tmpfs para rendimiento)
      - type: tmpfs
        target: /var/cache/nginx
        tmpfs:
          size: 100M
    
    networks:
      - frontend
      - backend
    
    restart: unless-stopped
    
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Seguridad
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      - CHOWN
      - SETUID
      - SETGID
    
    security_opt:
      - no-new-privileges:true
    
    # Recursos
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 128M

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Sin acceso externo
```


---

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

### 3.8 Páginas de Error Personalizadas

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

### 3.9 Logs en Nginx

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

### 3.10 Recarga en Caliente (Graceful Reload)

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
