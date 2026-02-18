# 📋 Cheat Sheet — Referencia Rápida

Comandos más frecuentes agrupados por herramienta para el día a día.

---

## 🐳 Docker

```bash
# Imágenes
docker build -t mi-app .                        # Build imagen
docker build --no-cache -t mi-app .             # Build sin caché
docker pull nginx:alpine                        # Descargar imagen
docker images                                   # Listar imágenes
docker rmi mi-app                               # Eliminar imagen
docker image prune -f                           # Eliminar imágenes sin usar

# Contenedores
docker run -d --name web -p 80:80 nginx:alpine  # Ejecutar en background
docker run -it --rm alpine sh                   # Shell temporal (se elimina al salir)
docker ps                                       # Contenedores activos
docker ps -a                                    # Todos (incluidos parados)
docker stop web && docker start web             # Stop/Start
docker restart web                              # Reiniciar
docker rm -f web                                # Eliminar forzado

# Inspección
docker logs -f web                              # Logs en tiempo real
docker logs --tail 50 web                       # Últimas 50 líneas
docker exec -it web sh                          # Shell interactivo
docker exec web nginx -t                        # Test config nginx
docker exec web nginx -s reload                 # Reload nginx
docker inspect web                              # Info completa JSON
docker stats web                                # CPU/RAM en tiempo real
docker top web                                  # Procesos del contenedor
docker port web                                 # Puertos mapeados

# Volúmenes
docker volume ls                                # Listar volúmenes
docker volume inspect db_data                   # Detalles de un volumen
docker volume prune -f                          # Eliminar volúmenes sin usar

# Redes
docker network ls                               # Listar redes
docker network inspect frontend                 # Detalles de una red
docker network connect frontend web             # Conectar contenedor a red

# Limpieza general
docker system prune -f                          # Eliminar todo lo no usado
docker system prune -af --volumes               # Limpieza total (¡cuidado!)
docker system df                                # Espacio usado por Docker
```

---

## 🐙 Docker Compose

```bash
# Ciclo de vida
docker compose up -d                            # Levantar en background
docker compose up -d --build                    # Levantar reconstruyendo imágenes
docker compose down                             # Parar y eliminar contenedores
docker compose down -v                          # También elimina volúmenes
docker compose restart nginx                    # Reiniciar un servicio
docker compose stop                             # Solo parar (sin eliminar)

# Logs e inspección
docker compose logs -f                          # Todos los logs
docker compose logs -f nginx                    # Logs de un servicio
docker compose ps                               # Estado de servicios
docker compose top                              # Procesos de todos los servicios

# Escalado y ejecución
docker compose up -d --scale app=3             # Escalar a 3 instancias
docker compose exec nginx sh                    # Shell en un servicio
docker compose exec nginx nginx -t              # Test config nginx
docker compose exec nginx nginx -s reload       # Reload nginx

# Build
docker compose build                            # Reconstruir imágenes
docker compose build --no-cache nginx           # Sin caché para un servicio
docker compose pull                             # Actualizar imágenes base
```

---

## ⚙️ Nginx

```bash
# Control del servicio
nginx -t                    # Verificar sintaxis de configuración
nginx -s reload             # Reload sin downtime (graceful)
nginx -s stop               # Stop inmediato
nginx -s quit               # Stop graceful (espera peticiones activas)
nginx -s reopen             # Reabrir ficheros de log (tras logrotate)
nginx -V                    # Versión, módulos compilados y flags

# Dentro de contenedor Docker
docker exec web nginx -t
docker exec web nginx -s reload

# Rutas importantes
# /etc/nginx/nginx.conf          → Config principal
# /etc/nginx/conf.d/             → Virtual hosts
# /etc/nginx/sites-enabled/      → Sites (Debian/Ubuntu)
# /var/log/nginx/access.log      → Log de acceso
# /var/log/nginx/error.log       → Log de errores
# /usr/share/nginx/html/         → Web root por defecto
```

---

## 🔍 Debugging y Testing

```bash
# Logs en tiempo real
tail -f /var/log/nginx/access.log               # Peticiones
tail -f /var/log/nginx/error.log                # Errores
tail -n 100 /var/log/nginx/error.log            # Últimas 100 líneas

# curl — Testing de endpoints
curl -I https://ejemplo.com                     # Solo headers (HEAD)
curl -v https://ejemplo.com                     # Verbose (headers + body)
curl -L https://ejemplo.com                     # Seguir redirecciones
curl -k https://ejemplo.com                     # Ignorar certificado inválido
curl -H "Host: otro.com" http://localhost       # Test virtual host
curl -H "X-API-Key: test" https://ejemplo.com/api  # Con header personalizado
curl -u usuario:password https://ejemplo.com    # Basic Auth
curl -w "%{http_code}" -o /dev/null -s https://ejemplo.com  # Solo código HTTP
curl -X POST -d '{"key":"val"}' -H "Content-Type: application/json" https://ejemplo.com/api

# Test de rate limiting
for i in {1..20}; do curl -s -o /dev/null -w "%{http_code}\n" http://ejemplo.com; done

# SSL/TLS
openssl s_client -connect ejemplo.com:443 -servername ejemplo.com  # Info SSL
openssl x509 -in cert.pem -noout -dates        # Fechas de validez del certificado
openssl x509 -in cert.pem -noout -subject      # Sujeto del certificado
# Generar certificado autofirmado
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout key.pem -out cert.pem \
  -subj "/CN=localhost"

# Red y puertos
ss -tlnp | grep nginx                           # Puertos en escucha (Linux)
netstat -tlnp | grep :80                        # Alternativa clásica
nmap -p 80,443 ejemplo.com                      # Escaneo de puertos
ping -c 4 ejemplo.com                           # Conectividad básica
traceroute ejemplo.com                          # Ruta de red
dig ejemplo.com                                 # Resolución DNS
nslookup ejemplo.com                            # DNS alternativo
```

---

## 🔐 Seguridad — Comandos Útiles

```bash
# Generar contraseña para Basic Auth (htpasswd)
htpasswd -c /etc/nginx/.htpasswd usuario        # Crear archivo nuevo
htpasswd /etc/nginx/.htpasswd otro_usuario      # Añadir usuario

# Verificar headers de seguridad
curl -I https://ejemplo.com | grep -E "Strict|X-Frame|X-Content|CSP"

# Ver certificado SSL en detalle
echo | openssl s_client -connect ejemplo.com:443 2>/dev/null | openssl x509 -noout -text

# Comprobar que HSTS está activo
curl -sI https://ejemplo.com | grep Strict-Transport-Security
```

---

## 🧹 Limpieza y Mantenimiento

```bash
# Docker — liberar espacio
docker system df                                # Ver uso de disco
docker system prune -f                          # Eliminar recursos sin usar
docker image prune -af                          # Eliminar todas las imágenes sin contenedor
docker volume prune -f                          # Eliminar volúmenes huérfanos
docker container prune -f                       # Eliminar contenedores parados

# Logs — rotar manualmente
docker exec web truncate -s 0 /var/log/nginx/access.log  # Vaciar log de acceso
```

---

## ⚡ Atajos Frecuentes

| Tarea | Comando |
|-------|---------|
| Ver qué ocupa más espacio | `docker system df` |
| Entrar en un contenedor | `docker exec -it <nombre> sh` |
| Ver logs de un servicio | `docker compose logs -f <servicio>` |
| Recargar nginx sin downtime | `docker exec <nombre> nginx -s reload` |
| Verificar config antes de recargar | `docker exec <nombre> nginx -t` |
| Comprobar puertos en uso | `ss -tlnp` |
| Test rápido de endpoint | `curl -I https://ejemplo.com` |
| Limpiar todo Docker | `docker system prune -af --volumes` |