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



