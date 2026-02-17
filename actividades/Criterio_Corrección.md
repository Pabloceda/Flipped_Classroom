## Crierios de correción para las actividades

### Criterios de Corrección Actividad 1

| Criterio | Puntos | Descripción |
|:---------|:------:|:------------|
| **Dockerización correcta** | 20 | - Imágenes con versiones específicas (no `:latest`)<br>- Volúmenes persistentes configurados<br>- Redes personalizadas (frontend/backend)<br>- Healthchecks implementados |
| **Proxy inverso funcional** | 15 | - Nginx redirecciona correctamente a WordPress<br>- Headers proxy configurados (`X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`)<br>- Nginx sirve estáticos sin pasar por PHP |
| **HTTPS y certificados** | 15 | - Certificados SSL generados correctamente<br>- HTTPS funcional en puerto 443<br>- Redirección HTTP → HTTPS |
| **Headers de seguridad** | 10 | - HSTS configurado<br>- `server_tokens off`<br>- Headers X-Frame-Options, X-Content-Type-Options |
| **Autenticación HTTP** | 10 | - `/wp-admin` protegido con autenticación básica<br>- Archivo `.htpasswd` correctamente generado |
| **Rate Limiting** | 15 | - Zona de rate limit configurada<br>- Límite aplicado en `/wp-login.php`<br>- Funciona correctamente (verificable con curl) |
| **Protección de archivos** | 10 | - `wp-config.php` bloqueado<br>- Archivos ocultos (`.ht*`) bloqueados<br>- `xmlrpc.php` bloqueado |
| **Documentación** | 5 | - README.md con instrucciones de despliegue<br>- Comandos de verificación incluidos |

**Puntuación total: 100 puntos**

**Niveles:**
- **90-100**: Excelente - Todas las políticas implementadas correctamente
- **75-89**: Notable - Funcional con algunas mejoras menores
- **60-74**: Aprobado - Funciona pero faltan políticas importantes
- **< 60**: Suspenso - No cumple requisitos mínimos

---

### Criterios de Corrección Actividad 2

| Criterio | Puntos | Descripción |
|:---------|:------:|:------------|
| **Aplicación Backend funcional** | 15 | - Aplicación Node.js responde correctamente<br>- Endpoint `/health` implementado<br>- Muestra ID del servidor en respuesta<br>- Dockerfile correcto |
| **Upstream configurado** | 15 | - Grupo upstream con 3+ servidores<br>- Servidor backup configurado<br>- Pesos (weight) correctamente asignados |
| **Algoritmos de balanceo** | 25 | - **Round Robin** con pesos funcional (10pts)<br>- **Least Connections** funcional (8pts)<br>- **IP Hash** funcional (7pts) |
| **Health Checks** | 15 | - `max_fails` y `fail_timeout` configurados<br>- `proxy_next_upstream` implementado<br>- Failover funciona correctamente |
| **Logs personalizados** | 10 | - Formato de log muestra backend usado<br>- Muestra tiempos de respuesta<br>- Logs accesibles desde host |
| **Pruebas y verificación** | 15 | - Script de pruebas funcional<br>- Demuestra distribución de carga<br>- Demuestra failover<br>- Verifica servidor backup |
| **Documentación** | 5 | - README con instrucciones claras<br>- Explicación de cada algoritmo<br>- Comandos de verificación |

**Puntuación total: 100 puntos**

**Comparativa de algoritmos esperada:**

| Algoritmo | Escenario ideal | Sticky Sessions |
|:----------|:----------------|:----------------|
| Round Robin | Backends homogéneos, requests similares | ❌ No |
| Weighted RR | Backends con diferente capacidad | ❌ No |
| Least Conn | Requests de duración variable | ❌ No |
| IP Hash | Aplicaciones con sesiones locales | ✅ Sí |

---
