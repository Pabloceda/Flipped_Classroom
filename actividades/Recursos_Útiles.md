## 📚 Recursos Adicionales
Aquí puedes encontrar recursos adicionales que te pueden ser de utilidad para realizar las actividades:
### Comandos Útiles de Docker

```bash
# Ver logs de un servicio específico
docker-compose logs -f [servicio]

# Reiniciar un servicio
docker-compose restart [servicio]

# Ejecutar comando en contenedor
docker-compose exec [servicio] sh

# Ver recursos utilizados
docker stats

# Limpiar todo
docker-compose down -v
```

### Debugging Común

**Problema: 502 Bad Gateway**
```bash
# Verificar que el backend está corriendo
docker-compose ps

# Ver logs del backend
docker-compose logs backend

# Verificar conectividad
docker-compose exec nginx ping backend1
```

**Problema: Rate limiting no funciona**
```bash
# Verificar que la zona está definida en http {}
# Verificar que limit_req está en location {}
# Ver logs para confirmar 503
docker-compose logs nginx | grep "limiting requests"
```
