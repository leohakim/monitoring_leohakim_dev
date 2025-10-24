# Guía de Monitoreo - Elecciones 2025

## 📊 Stack de Monitoreo

Sistema completo de observabilidad para monitorear **Production** y **Standby** con métricas detalladas, logs centralizados y dashboards para análisis post-evento.

### Componentes

| Componente | Propósito | Puerto |
|------------|-----------|--------|
| **Prometheus** | Recolección y almacenamiento de métricas | 9090 |
| **Grafana** | Visualización y dashboards | 3000 |
| **Loki** | Agregación de logs | 3100 |
| **AlertManager** | Gestión de alertas | 9093 |
| **cAdvisor** | Métricas de contenedores | 8080 |
| **Node Exporter** | Métricas del SO | 9100 |
| **PostgreSQL Exporter** | Métricas de BD | 9187 |
| **Redis Exporter** | Métricas de Redis | 9121 |
| **Promtail** | Recolector de logs | 9080 |

---

## 🎯 Métricas Recolectadas

### **Velocidad/Performance**
- ✅ Latencia HTTP (p50, p95, p99)
- ✅ Throughput (requests/segundo)
- ✅ Tiempo de respuesta de BD
- ✅ Cache hit ratio de Redis
- ✅ Tiempo de procesamiento Celery

### **Resiliencia**
- ✅ Tasa de errores (4xx, 5xx)
- ✅ Reintentos de requests
- ✅ Reintentos de tareas Celery
- ✅ Conexiones fallidas
- ✅ Health checks fallidos

### **Disponibilidad**
- ✅ Uptime por servicio
- ✅ SLA del sistema
- ✅ Tiempo de recuperación
- ✅ Disponibilidad de endpoints

### **Recursos**
- ✅ CPU (por servicio y total)
- ✅ RAM (por servicio y total)
- ✅ Disco I/O y espacio
- ✅ Red (bandwidth)
- ✅ Conexiones activas

### **Aplicación**
- ✅ Usuarios concurrentes
- ✅ Sesiones activas
- ✅ Queries a la API
- ✅ Tareas Celery (pending/active/completed/failed)
- ✅ Cache hits/misses

---

## 🚀 Paso 1: Despliegue del Servidor de Monitoreo

### Opción A: Servidor Dedicado (Recomendado)

**Requisitos mínimos:**
- 2 CPU cores
- 4GB RAM
- 50GB disco (para retención de 90 días)
- Debian 12 / Ubuntu 22.04

```bash
# En el servidor de monitoreo
cd /opt/apps
git clone <REPO> elecciones-monitoring
cd elecciones-monitoring

# Configurar variables de entorno
nano .env
```

**Contenido de `.env`:**
```bash
GRAFANA_ADMIN_PASSWORD=TU_PASSWORD_SEGURO
```

```bash
# Iniciar servicios de monitoreo
docker compose -f docker-compose.monitoring.yml up -d

# Verificar que todo está corriendo
docker compose -f docker-compose.monitoring.yml ps
```

### Opción B: En el Mismo Servidor de Production

Si no tienes un servidor dedicado, puedes desplegar en production (requiere más recursos):

```bash
cd /opt/server/elecciones-nacionales-octubre-2025
docker compose -f docker-compose.production.yml -f docker-compose.monitoring.yml up -d
```

---

## 🔧 Paso 2: Configurar IPs de los Servidores

### 2.1. Editar Prometheus

```bash
nano compose/monitoring/prometheus/prometheus.yml
```

**Buscar y reemplazar:**
- `IP_PRODUCTION` → IP real del servidor production (ej: `192.168.1.10`)
- `IP_STANDBY` → IP real del servidor standby (ej: `51.38.233.100`)

**Ejemplo:**
```yaml
- targets: ['192.168.1.10:8080']  # cAdvisor production
- targets: ['51.38.233.100:8080']  # cAdvisor standby
```

### 2.2. Recargar configuración de Prometheus

```bash
# Recargar sin reiniciar (hot reload)
curl -X POST http://localhost:9090/-/reload

# O reiniciar el contenedor
docker compose -f docker-compose.monitoring.yml restart prometheus
```

---

## 📡 Paso 3: Desplegar Exporters en Production

### 3.1. En el servidor de Production

```bash
cd /opt/server/elecciones-nacionales-octubre-2025

# Configurar IP del servidor de monitoreo
nano docker-compose.exporters.yml
```

**Buscar y cambiar:**
```yaml
LOKI_URL: "http://IP_MONITORING_SERVER:3100"
```

**Ejemplo:**
```yaml
LOKI_URL: "http://192.168.1.5:3100"  # IP del servidor de monitoreo
```

### 3.2. Configurar variable de entorno para PostgreSQL

```bash
nano .envs/.production/.postgres
```

**Agregar (si no existe):**
```bash
POSTGRES_PASSWORD=tu_password_actual
```

### 3.3. Iniciar exporters

```bash
# Iniciar production + exporters
docker compose -f docker-compose.production.yml -f docker-compose.exporters.yml up -d

# Verificar que los exporters están corriendo
docker compose -f docker-compose.production.yml -f docker-compose.exporters.yml ps | grep exporter
```

### 3.4. Abrir puertos en firewall

```bash
# Permitir acceso desde el servidor de monitoreo
ufw allow from IP_MONITORING_SERVER to any port 8080  # cAdvisor
ufw allow from IP_MONITORING_SERVER to any port 9100  # Node Exporter
ufw allow from IP_MONITORING_SERVER to any port 9187  # PostgreSQL Exporter
ufw allow from IP_MONITORING_SERVER to any port 9121  # Redis Exporter
```

---

## 📡 Paso 4: Desplegar Exporters en Standby

### 4.1. En el servidor de Standby

```bash
cd /opt/server/elecciones-nacionales-octubre-2025

# Configurar IP del servidor de monitoreo
nano docker-compose.exporters.yml
```

**Cambiar:**
```yaml
LOKI_URL: "http://IP_MONITORING_SERVER:3100"
```

### 4.2. Configurar variable de entorno para PostgreSQL

```bash
nano .envs/.standby/.postgres
```

**Verificar que existe:**
```bash
POSTGRES_PASSWORD=tu_password_standby
```

### 4.3. Iniciar exporters

```bash
# Iniciar standby + exporters
docker compose -f docker-compose.standby.yml -f docker-compose.exporters.yml up -d

# Verificar
docker compose -f docker-compose.standby.yml -f docker-compose.exporters.yml ps | grep exporter
```

### 4.4. Abrir puertos en firewall

```bash
ufw allow from IP_MONITORING_SERVER to any port 8080
ufw allow from IP_MONITORING_SERVER to any port 9100
ufw allow from IP_MONITORING_SERVER to any port 9187
ufw allow from IP_MONITORING_SERVER to any port 9121
```

---

## ✅ Paso 5: Verificar Conectividad

### 5.1. Desde el servidor de monitoreo

```bash
# Verificar que Prometheus puede alcanzar los exporters

# Production
curl http://IP_PRODUCTION:8080/metrics  # cAdvisor
curl http://IP_PRODUCTION:9100/metrics  # Node Exporter
curl http://IP_PRODUCTION:9187/metrics  # PostgreSQL Exporter
curl http://IP_PRODUCTION:9121/metrics  # Redis Exporter

# Standby
curl http://IP_STANDBY:8080/metrics
curl http://IP_STANDBY:9100/metrics
curl http://IP_STANDBY:9187/metrics
curl http://IP_STANDBY:9121/metrics
```

### 5.2. Verificar targets en Prometheus

```bash
# Abrir en navegador
http://IP_MONITORING_SERVER:9090/targets

# Todos los targets deben estar en estado "UP" (verde)
```

---

## 📊 Paso 6: Acceder a Grafana

### 6.1. Abrir Grafana

```
URL: http://IP_MONITORING_SERVER:3000
Usuario: admin
Password: (el que configuraste en GRAFANA_ADMIN_PASSWORD)
```

### 6.2. Verificar datasources

1. Ir a **Configuration** → **Data Sources**
2. Verificar que **Prometheus** y **Loki** están conectados (verde)

### 6.3. Importar dashboards adicionales

Grafana tiene dashboards pre-hechos excelentes:

**Dashboards recomendados:**

1. **Node Exporter Full** (ID: 1860)
   - Métricas completas del sistema operativo
   
2. **Docker Container & Host Metrics** (ID: 179)
   - Métricas de contenedores Docker
   
3. **PostgreSQL Database** (ID: 9628)
   - Métricas detalladas de PostgreSQL
   
4. **Redis Dashboard** (ID: 11835)
   - Métricas de Redis

**Para importar:**
1. Ir a **Dashboards** → **Import**
2. Ingresar el ID del dashboard
3. Seleccionar datasource **Prometheus**
4. Click en **Import**

---

## 📈 Paso 7: Crear Dashboard Personalizado para Stakeholders

### 7.1. Crear nuevo dashboard

1. **Dashboards** → **New Dashboard** → **Add visualization**
2. Seleccionar datasource **Prometheus**

### 7.2. Paneles recomendados para el reporte

#### Panel 1: Uptime Total
```promql
avg(up{job=~".*production.*|.*standby.*"}) * 100
```
**Tipo:** Stat
**Título:** Disponibilidad Total (%)

#### Panel 2: Requests por Segundo
```promql
sum(rate(traefik_service_requests_total[5m])) by (server)
```
**Tipo:** Time series
**Título:** Throughput (req/s)

#### Panel 3: Latencia p95
```promql
histogram_quantile(0.95, rate(traefik_service_request_duration_seconds_bucket[5m]))
```
**Tipo:** Time series
**Título:** Latencia p95 (segundos)

#### Panel 4: Tasa de Errores
```promql
sum(rate(traefik_service_requests_total{code=~"5.."}[5m])) / sum(rate(traefik_service_requests_total[5m])) * 100
```
**Tipo:** Time series
**Título:** Tasa de Errores 5xx (%)

#### Panel 5: CPU Usage por Servidor
```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
**Tipo:** Time series
**Título:** CPU Usage (%)

#### Panel 6: Memory Usage por Servidor
```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```
**Tipo:** Time series
**Título:** Memory Usage (%)

#### Panel 7: Conexiones a PostgreSQL
```promql
pg_stat_activity_count
```
**Tipo:** Time series
**Título:** Conexiones Activas a BD

#### Panel 8: Tareas Celery
```promql
sum(celery_tasks_total) by (state, server)
```
**Tipo:** Time series
**Título:** Tareas Celery por Estado

---

## 📝 Paso 8: Configurar Anotaciones para Eventos

Las anotaciones te permiten marcar eventos importantes en los dashboards.

### 8.1. Crear anotación manual

1. En cualquier dashboard, hacer click en un punto del gráfico
2. **Add annotation**
3. Agregar descripción (ej: "Inicio de votación", "Cierre de mesas", "Pico de tráfico")
4. **Save**

### 8.2. Anotaciones automáticas desde Prometheus

Editar dashboard → **Settings** → **Annotations** → **Add annotation query**

```promql
ALERTS{alertstate="firing"}
```

Esto mostrará automáticamente cuando se disparan alertas.

---

## 📊 Paso 9: Generar Reportes Post-Evento

### 9.1. Exportar dashboard como PDF

1. Abrir el dashboard que quieres exportar
2. Click en **Share** (icono de compartir)
3. **Export** → **Save as PDF**
4. Configurar:
   - Time range: Seleccionar el período del evento
   - Layout: Portrait o Landscape
5. **Save as PDF**

### 9.2. Exportar datos crudos

```bash
# Desde el servidor de monitoreo

# Exportar snapshot de Prometheus
docker exec monitoring_prometheus promtool tsdb dump /prometheus > metrics_snapshot.json

# Comprimir
gzip metrics_snapshot.json

# Descargar a tu máquina local
scp user@monitoring_server:/path/metrics_snapshot.json.gz ./
```

### 9.3. Queries útiles para el reporte

#### Disponibilidad Total (SLA)
```promql
avg_over_time(up{job=~".*production.*"}[24h]) * 100
```

#### Throughput Promedio
```promql
avg_over_time(rate(traefik_service_requests_total[5m])[24h])
```

#### Throughput Máximo
```promql
max_over_time(rate(traefik_service_requests_total[5m])[24h])
```

#### Latencia Promedio
```promql
avg_over_time(histogram_quantile(0.95, rate(traefik_service_request_duration_seconds_bucket[5m]))[24h])
```

#### Total de Requests
```promql
sum(increase(traefik_service_requests_total[24h]))
```

#### Total de Errores
```promql
sum(increase(traefik_service_requests_total{code=~"5.."}[24h]))
```

#### CPU Promedio
```promql
avg_over_time((100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))[24h])
```

#### CPU Máximo
```promql
max_over_time((100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))[24h])
```

#### Memoria Promedio
```promql
avg_over_time(((1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100)[24h])
```

---

## 📈 Paso 10: Análisis y Storytelling para Stakeholders

### 10.1. Estructura del Reporte

**1. Executive Summary**
- Disponibilidad total (SLA)
- Throughput total
- Usuarios concurrentes pico
- Incidentes críticos (si hubo)

**2. Performance**
- Gráfico de requests/segundo durante el evento
- Latencia promedio y picos
- Comparación production vs standby (si se usó)

**3. Resiliencia**
- Tasa de errores
- Tiempo de respuesta ante incidentes
- Efectividad de alertas

**4. Recursos**
- Uso de CPU/RAM/Disco
- Identificación de cuellos de botella
- Margen de capacidad restante

**5. Lecciones Aprendidas**
- Qué funcionó bien
- Qué se puede mejorar
- Recomendaciones para futuros eventos

### 10.2. Métricas Clave para Stakeholders

| Métrica | Descripción | Query Prometheus |
|---------|-------------|------------------|
| **SLA** | % de disponibilidad | `avg_over_time(up[24h]) * 100` |
| **Throughput Pico** | Máximo req/s | `max_over_time(rate(requests_total[5m])[24h])` |
| **Usuarios Concurrentes** | Máximo simultáneo | `max_over_time(active_sessions[24h])` |
| **Latencia p95** | 95% de requests < X ms | `histogram_quantile(0.95, ...)` |
| **Tasa de Errores** | % de requests fallidos | `sum(errors) / sum(total) * 100` |
| **Tiempo de Recuperación** | MTTR de incidentes | Manual desde anotaciones |

### 10.3. Visualizaciones Recomendadas

1. **Línea de tiempo del evento**
   - Requests/segundo con anotaciones de eventos clave
   
2. **Heatmap de latencia**
   - Mostrar distribución de latencias a lo largo del tiempo
   
3. **Comparación Production vs Standby**
   - Si se activó standby, mostrar transición
   
4. **Uso de recursos**
   - Gráficos de área apilada de CPU/RAM
   
5. **Top errores**
   - Tabla con los errores más frecuentes

---

## 🔔 Configuración de Alertas (Opcional)

### Alertas ya configuradas

El sistema incluye alertas para:
- ✅ Servicios caídos
- ✅ Alto uso de CPU (>80%)
- ✅ Alto uso de memoria (>85%)
- ✅ Alto uso de disco (>80%)
- ✅ PostgreSQL caído
- ✅ Redis caído
- ✅ Queries lentas
- ✅ Alta tasa de errores

### Configurar notificaciones

#### Opción 1: Email (requiere SMTP)

```bash
nano compose/monitoring/prometheus/alertmanager.yml
```

```yaml
receivers:
  - name: 'critical'
    email_configs:
      - to: 'tu-email@ejemplo.com'
        from: 'alertas@elecciones.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'tu-email@gmail.com'
        auth_password: 'tu-app-password'
```

#### Opción 2: Webhook (Slack, Discord, etc.)

```yaml
receivers:
  - name: 'critical'
    webhook_configs:
      - url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        send_resolved: true
```

#### Opción 3: Telegram

```yaml
receivers:
  - name: 'critical'
    telegram_configs:
      - bot_token: 'YOUR_BOT_TOKEN'
        chat_id: YOUR_CHAT_ID
```

---

## 🛠️ Mantenimiento y Troubleshooting

### Ver logs de servicios de monitoreo

```bash
# Prometheus
docker compose -f docker-compose.monitoring.yml logs -f prometheus

# Grafana
docker compose -f docker-compose.monitoring.yml logs -f grafana

# Loki
docker compose -f docker-compose.monitoring.yml logs -f loki
```

### Verificar espacio en disco

```bash
# Ver uso de volúmenes
docker system df -v

# Limpiar datos antiguos (cuidado!)
# Esto eliminará métricas más allá de la retención configurada
docker exec monitoring_prometheus promtool tsdb cleanup /prometheus
```

### Backup de dashboards de Grafana

```bash
# Exportar todos los dashboards
docker exec monitoring_grafana grafana-cli admin export-dashboard > dashboards_backup.json

# O copiar el volumen completo
docker run --rm -v monitoring_grafana_data:/data -v $(pwd):/backup alpine tar czf /backup/grafana_backup.tar.gz /data
```

### Restaurar desde backup

```bash
# Restaurar volumen de Grafana
docker run --rm -v monitoring_grafana_data:/data -v $(pwd):/backup alpine tar xzf /backup/grafana_backup.tar.gz -C /
```

### Problema: Targets en estado "DOWN"

```bash
# 1. Verificar conectividad de red
ping IP_TARGET

# 2. Verificar que el exporter está corriendo
curl http://IP_TARGET:PORT/metrics

# 3. Verificar firewall
ufw status

# 4. Ver logs del exporter
docker logs monitoring_node_exporter
```

### Problema: Grafana no muestra datos

```bash
# 1. Verificar datasource
curl http://localhost:9090/api/v1/query?query=up

# 2. Verificar que Prometheus tiene datos
docker exec monitoring_prometheus promtool tsdb list /prometheus

# 3. Reiniciar Grafana
docker compose -f docker-compose.monitoring.yml restart grafana
```

---

## 📚 Recursos Adicionales

### Documentación Oficial
- [Prometheus](https://prometheus.io/docs/)
- [Grafana](https://grafana.com/docs/)
- [Loki](https://grafana.com/docs/loki/)

### Dashboards de la Comunidad
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)

### PromQL (Lenguaje de queries)
- [PromQL Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [PromQL Examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)

---

## 📞 Checklist de Despliegue

- [ ] Servidor de monitoreo desplegado
- [ ] IPs configuradas en prometheus.yml
- [ ] Exporters desplegados en production
- [ ] Exporters desplegados en standby
- [ ] Puertos de firewall abiertos
- [ ] Todos los targets en estado "UP"
- [ ] Grafana accesible
- [ ] Datasources conectados
- [ ] Dashboards importados
- [ ] Dashboard personalizado creado
- [ ] Alertas configuradas (opcional)
- [ ] Notificaciones configuradas (opcional)
- [ ] Backup de configuración realizado

---

## 🎯 Resumen de Puertos

| Servicio | Puerto | Acceso |
|----------|--------|--------|
| Prometheus | 9090 | Interno |
| Grafana | 3000 | Público (con auth) |
| Loki | 3100 | Interno |
| AlertManager | 9093 | Interno |
| cAdvisor | 8080 | Interno (desde monitoring) |
| Node Exporter | 9100 | Interno (desde monitoring) |
| PostgreSQL Exporter | 9187 | Interno (desde monitoring) |
| Redis Exporter | 9121 | Interno (desde monitoring) |

---

**Última actualización**: Octubre 2025  
**Versión**: 1.0  
**Retención de datos**: 90 días  
**Contacto**: leohakim@gmail.com
