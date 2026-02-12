# Guía Didáctica: Despliegue de Servidores Web con Nginx y Docker

## 1. Metodología de la Sesión
Esta unidad está diseñada bajo el modelo **Flipped Classroom** (Aula Invertida).
- **En Casa (Previo)**: Los alumnos deben estudiar los Módulos 1, 2 y 3 por su cuenta.
- **En Clase**: Se dedica el tiempo a resolver dudas, realizar el laboratorio práctico (Módulo 7) y debatir sobre arquitectura.
- **Rol del Docente**: Facilitador y "Senior Engineer" que guía en la resolución de problemas, no mero transmisor de información.

## 2. Cronograma Estimado (Sesión de 4 horas)

| Tiempo | Actividad | Descripción |
| :--- | :--- | :--- |
| **00:00 - 00:30** | **Q&A Conceptual** | Resolución de dudas teóricas sobre Event-Driven, C10k y capas de Docker. Uso de diagrama en pizarra. |
| **00:30 - 01:30** | **Live Coding Demo** | El profesor levanta un stack básico (Nginx + PHP/Node) explicando `docker-compose.yml` línea a línea. |
| **01:30 - 02:30** | **Laboratorio Parte 1** | Alumnos realizan "Caso Práctico 1: E-commerce" (Balanceo + SSL). |
| **02:30 - 03:30** | **Laboratorio Parte 2** | Alumnos realizan "Caso Práctico 2: API Gateway" y pruebas de carga con `ab` o `wrk`. |
| **03:30 - 04:00** | **Retro & Debug** | Puesta en común de errores encontrados (`502 Bad Gateway`, permisos). Cierre. |

## 3. Conceptos Clave (Must-Know)
El alumno **DEBE** entender estos conceptos para aprobar:

1.  **Modelo de Procesos**: Por qué Nginx (Event-Driven) escala mejor que Apache (Process-Per-Connection) con alta concurrencia.
2.  **Inmutabilidad**: Por qué no se editan archivos *dentro* de un contenedor en ejecución (se usan volúmenes o se reconstruye la imagen).
3.  **Reverse Proxy**: La diferencia entre servir contenido estático y pasarle la petición a un backend (PHP-FPM, Node.js).
4.  **Networking Docker**: Cómo funciona la resolución de nombres DNS interna entre contenedores en una red bridge custom.
5.  **Puertos**: La diferencia entre `EXPOSE` (documentación) y `-p` (publicación real).

## 4. Errores Típicos del Alumnado ("Trampas")

### ❌ 1. "No me conecta" (Binding a Localhost)
- **Problema**: Configuran su app (Node/Python) para escuchar en `127.0.0.1` dentro del contenedor.
- **Por qué falla**: `localhost` dentro del contenedor NO es el `localhost` del host.
- **Solución**: La app debe escuchar en `0.0.0.0`.

### ❌ 2. El Trailing Slash en `proxy_pass`
- **Problema**: Confusión entre `proxy_pass http://backend;` y `proxy_pass http://backend/;`.
- **Consecuencia**: Rutas mal reescritas (ej: `/api/users` se convierte en `//users`).
- **Solución**: Explicar la regla de normalización de URIs de Nginx.

### ❌ 3. Persistencia de Datos
- **Problema**: Pierden la base de datos al borrar el contenedor.
- **Solución**: Insistir en el uso de Volúmenes Nombrados para `/var/lib/mysql` o `/var/lib/postgresql/data`.

### ❌ 4. HTTP/3 sobre TCP
- **Problema**: Mapean el puerto 443 solo como TCP en Docker Compose para HTTP/3.
- **Solución**: HTTP/3 usa **UDP**. Recordar `443:443/udp`.

### ❌ 5. "El cambio en nginx.conf no se ve"
- **Problema**: Editan el archivo en el host pero olvidaron montarlo como volumen, o no reiniciaron/recargaron el contenedor.
- **Solución**: `docker-compose restart nginx` o `docker exec nginx nginx -s reload`.
