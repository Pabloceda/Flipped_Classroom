## MÓDULO 2: DOCKERIZACIÓN DEL SERVICIO

> **Conexión con el Módulo 1**: Sabemos instalar y configurar servidores web (Apache, Nginx). El problema es que esa instalación es **manual, frágil y difícil de reproducir**. Docker resuelve esto empaquetando el servidor y toda su configuración en una imagen portable.

### ¿Qué aprenderás en este módulo?

- Por qué Docker se ha convertido en el estándar del despliegue moderno
- Cómo funcionan las **imágenes**, los **contenedores** y el **Dockerfile**
- Gestión de **puertos**, **volúmenes** y **redes**
- Orquestar múltiples servicios con **Docker Compose**

### Problema que resuelve

Sin Docker, el clásico *"en mi máquina funciona"* es una fuente constante de errores. Con Docker, el entorno de desarrollo, pruebas y producción es **exactamente el mismo**:

```
Sin Docker:          Con Docker:
Dev  ≠ Staging      Dev  = Staging = Producción
Staging ≠ Prod      docker build once → run anywhere
```

---

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

| Capa | 🖥️ Máquina Virtual | 🐳 Contenedor Docker |
|:----:|:------------------:|:--------------------:|
| **App** | App A · App B | App A · App B |
| **Dependencias** | Libs/Deps (×2) | Libs/Deps (×2) |
| **Sistema** | Guest OS (×2) | *(compartido)* |
| **Abstracción** | Hypervisor | Docker Engine |
| **Base** | Host OS | Host OS (Linux) |
| **Físico** | Hardware | Hardware |
| | *~GB por VM · minutos* | *~MB por imagen · segundos* |

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

| # | Capa | Instrucción | Rol |
|:-:|:-----|:------------|:----|
| 4 | 🟢 Arranque | `CMD nginx -g daemon off;` | Punto de entrada del contenedor |
| 3 | 🔵 Config | `COPY nginx.conf /etc/nginx/` | Configuración personalizada |
| 2 | 🟡 Software | `RUN apk add nginx` | Instalación de paquetes |
| 1 | ⬜ Base OS | `FROM alpine:3.19` | Sistema operativo mínimo |

> **Cada capa es de solo lectura.** Al ejecutar un contenedor, Docker añade una capa de escritura temporal encima.

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

**Arquitectura red Bridge**:

| Interfaz | Rol | IP ejemplo |
|:---------|:----|:-----------|
| `docker0` | Switch virtual (bridge) | 172.17.0.1 (gateway) |
| `veth0` | Canal Container 1 ↔ bridge | 172.17.0.2 |
| `veth1` | Canal Container 2 ↔ bridge | 172.17.0.3 |
| `veth2` | Canal Container 3 ↔ bridge | 172.17.0.4 |
| `eth0` | Interfaz física del host | IP externa |

**DNS Integrado**: Los contenedores en la misma red bridge custom se resuelven por nombre:

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

**Arquitectura red Overlay (multi-host)**:

| | 🖥️ Host 1 | 🌐 Túnel VXLAN | 🖥️ Host 2 |
|:-:|:---------:|:--------------:|:---------:|
| **Contenedores** | Container A (10.0.1.2) | ← UDP encapsulado → | Container C (10.0.1.4) |
| **Visibilidad** | Se ven entre sí como si estuvieran en la misma LAN | | |

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
