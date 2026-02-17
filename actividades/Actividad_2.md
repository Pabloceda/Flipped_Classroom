## ACTIVIDAD 2: Sistema de Balanceo de Carga con Alta Disponibilidad

### 📝 Enunciado

Debes implementar un sistema de balanceo de carga utilizando Nginx con múltiples instancias de backend, aplicando diferentes algoritmos de balanceo y técnicas de alta disponibilidad vistas en el Módulo 4.

**Componentes:**
- **Nginx**: balanceador de carga (load balancer)
- **3 instancias de backend**: aplicación Node.js simple
- **1 servidor de backup**: se activa solo si todos los demás fallan

**Requisitos técnicos:**

1. **Configuración de Upstream (Módulo 4.5)**
   - Definir un grupo upstream con 3 servidores activos
   - Un servidor backup
   - Configurar pesos diferentes (weight) para simular capacidades diferentes

2. **Algoritmos de Balanceo (Módulo 4.6)**
   - Implementar y documentar 3 configuraciones diferentes:
     - **Configuración A**: Round Robin con pesos
     - **Configuración B**: Least Connections
     - **Configuración C**: IP Hash (sticky sessions)

3. **Health Checks (Módulo 4.7)**
   - Configurar passive health checks (`max_fails`, `fail_timeout`)
   - Implementar `proxy_next_upstream` para reintentos automáticos
   - Crear endpoint `/health` en backends para verificación

4. **Monitorización y Logs**
   - Logs personalizados que muestren qué backend procesó cada petición
   - Formato de log que incluya: IP cliente, backend usado, tiempo de respuesta

**Aplicación Backend (Node.js):**
Cada instancia debe:
- Responder con su nombre/ID de contenedor
- Tener un endpoint `/health` que retorne status 200
- Simular carga variable (opcional: añadir delay aleatorio)

**Pruebas requeridas:**
1. Demostrar que el balanceo funciona (peticiones se distribuyen)
2. Simular caída de un backend (detenerlo) y verificar failover automático
3. Verificar que el servidor backup solo se usa cuando todos fallan
4. Medir distribución de carga según pesos configurados

---



