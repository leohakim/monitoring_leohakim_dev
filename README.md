# Monitoring Stack - leohakim.dev

> Stack de observabilidad completo para monitoreo de infraestructura y aplicaciones en producción.

[![Stack](https://img.shields.io/badge/Stack-Prometheus%20%7C%20Grafana%20%7C%20Loki-orange)](https://monitoring.leohakim.dev)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Deployment](https://img.shields.io/badge/Deployment-Docker%20Compose-2496ED?logo=docker)](docker-compose.monitoring.yml)

## 🎯 Propósito

Este repositorio contiene la configuración completa de un stack de monitoreo diseñado para proporcionar **observabilidad total** sobre los proyectos que lidero. Permite:

- **Monitoreo en tiempo real** de métricas de infraestructura y aplicaciones
- **Análisis post-evento** con retención de 90 días
- **Generación de reportes** para stakeholders con exportación a PDF
- **Alertas proactivas** para respuesta rápida ante incidentes
- **Centralización de logs** para troubleshooting eficiente

## 🏗️ Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│         monitoring.leohakim.dev (2 cores / 4GB)             │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Prometheus  │◄─┤   Grafana    │◄─┤     Loki     │     │
│  │  (Métricas)  │  │(Dashboards)  │  │   (Logs)     │     │
│  │              │  │              │  │              │     │
│  │ Retención:   │  │ • Alertas    │  │ • Agregación │     │
│  │ 90 días      │  │ • Export PDF │  │ • Búsqueda   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         ▲                                     ▲             │
└─────────┼─────────────────────────────────────┼─────────────┘
          │ Pull (cada 15s)                     │ Push
          │                                     │
┌─────────┴─────────────────────────────────────┴─────────────┐
│              Servidores Monitoreados                         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ • cAdvisor (métricas de contenedores)                │  │
│  │ • Node Exporter (métricas del sistema)               │  │
│  │ • PostgreSQL Exporter (métricas de BD)               │  │
│  │ • Redis Exporter (métricas de cache)                 │  │
│  │ • Promtail (recolección de logs)                     │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## 📊 Stack Tecnológico

| Componente | Versión | Propósito | Puerto |
|------------|---------|-----------|--------|
| **Prometheus** | 2.48.0 | Time-series database para métricas | 9090 |
| **Grafana** | 10.2.2 | Visualización y dashboards | 3000 |
| **Loki** | 2.9.3 | Agregación y búsqueda de logs | 3100 |
| **AlertManager** | 0.26.0 | Gestión y routing de alertas | 9093 |
| **Promtail** | 2.9.3 | Recolector de logs | 9080 |

### Exporters (Desplegados en servidores monitoreados)

| Exporter | Versión | Métricas | Puerto |
|----------|---------|----------|--------|
| **cAdvisor** | 0.47.2 | Contenedores Docker | 8080 |
| **Node Exporter** | 1.7.0 | Sistema operativo | 9100 |
| **PostgreSQL Exporter** | 0.15.0 | Base de datos | 9187 |
| **Redis Exporter** | 1.55.0 | Cache Redis | 9121 |

## 🚀 Quick Start

### Prerrequisitos

- Docker Engine 24.0+
- Docker Compose 2.20+
- 2 CPU cores y 4GB RAM mínimo
- 50GB de disco (300GB recomendado para 90 días de retención)

### Despliegue del Stack de Monitoreo

```bash
# Clonar el repositorio
git clone https://github.com/leohakim/monitoring_leohakim_dev.git
cd monitoring_leohakim_dev

# Configurar variables de entorno
cat > .env << EOF
GRAFANA_ADMIN_PASSWORD=tu_password_seguro_aqui
EOF

# Iniciar el stack
docker compose -f docker-compose.monitoring.yml up -d

# Verificar que todos los servicios están corriendo
docker compose -f docker-compose.monitoring.yml ps
```

### Configurar Servidores Monitoreados

En cada servidor que quieras monitorear:

```bash
# Copiar el archivo de exporters al servidor
scp docker-compose.exporters.yml user@servidor:/opt/apps/

# En el servidor monitoreado
cd /opt/apps
export POSTGRES_PASSWORD="tu_password"
export LOKI_URL="http://monitoring.leohakim.dev:3100"

# Iniciar exporters junto con tu aplicación
docker compose -f docker-compose.yml -f docker-compose.exporters.yml up -d
```

### Configurar IPs en Prometheus

Editar `compose/monitoring/prometheus/prometheus.yml` y reemplazar:
- `IP_PRODUCTION` con la IP de tu servidor de producción
- `IP_STANDBY` con la IP de tu servidor de standby (si aplica)

```bash
# Reiniciar Prometheus para aplicar cambios
docker compose -f docker-compose.monitoring.yml restart prometheus
```

### Acceso a Grafana

```
URL: https://monitoring.leohakim.dev
Usuario: admin
Password: (configurado en .env)
```

## 📈 Métricas Recolectadas

### Performance
- **Latencia HTTP** (p50, p95, p99)
- **Throughput** (requests/segundo)
- **Tiempo de respuesta** de base de datos
- **Cache hit ratio** de Redis
- **Tiempo de procesamiento** de tareas asíncronas

### Resiliencia
- **Tasa de errores** (4xx, 5xx)
- **Reintentos** de requests y tareas
- **Conexiones fallidas** a servicios
- **Health checks** fallidos

### Disponibilidad
- **Uptime** por servicio
- **SLA** del sistema
- **MTTR** (Mean Time To Recovery)
- **Disponibilidad** de endpoints críticos

### Recursos
- **CPU y RAM** (por servicio y total)
- **Disco** I/O y espacio disponible
- **Ancho de banda** de red
- **Conexiones activas** por servicio

### Aplicación
- **Usuarios concurrentes**
- **Sesiones activas**
- **Queries a la API**
- **Estado de tareas** asíncronas
- **Métricas de cache**

## 📊 Dashboards

### Dashboards Incluidos

1. **Overview Dashboard** - Vista general del estado de todos los servidores
2. **Node Exporter Full** (ID: 1860) - Métricas completas del sistema operativo
3. **Docker & Host Metrics** (ID: 179) - Contenedores y recursos
4. **PostgreSQL Database** (ID: 9628) - Performance de base de datos
5. **Redis Dashboard** (ID: 11835) - Métricas de cache

### Importar Dashboards de la Comunidad

```
Grafana UI → Dashboards → Import → Ingresar ID → Load
```

Dashboards recomendados:
- **1860** - Node Exporter Full
- **179** - Docker & Host Metrics
- **9628** - PostgreSQL Database
- **11835** - Redis Dashboard
- **11462** - Traefik 2

## 🔔 Alertas

El sistema incluye 20+ reglas de alertas pre-configuradas:

### Alertas Críticas
- CPU >90% por más de 5 minutos
- RAM >95% por más de 5 minutos
- Disco >90%
- Servicio caído (down)
- Base de datos inaccesible

### Alertas Warning
- CPU >80% por más de 10 minutos
- RAM >85% por más de 10 minutos
- Disco >80%
- Alta latencia (>1s p95)
- Tasa de errores >5%

Las alertas se envían a través de AlertManager con soporte para:
- Email
- Slack
- PagerDuty
- Webhooks personalizados

## 📝 Generación de Reportes

### Exportar Dashboard a PDF

```
Grafana → Dashboard → Share → Export → Save as PDF
```

### Queries PromQL para Reportes

```promql
# SLA del sistema (últimos 30 días)
(sum(up) / count(up)) * 100

# Throughput promedio
rate(http_requests_total[5m])

# Latencia p95
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Tasa de errores
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

### KPIs para Stakeholders

| KPI | Descripción | Objetivo |
|-----|-------------|----------|
| **SLA** | Disponibilidad del sistema | >99.9% |
| **Throughput Pico** | Máximo req/s soportado | Documentar |
| **Latencia p95** | 95% de requests < X ms | <500ms |
| **Tasa de Errores** | % de requests fallidos | <0.1% |
| **MTTR** | Tiempo promedio de recuperación | <5 min |

## 🔒 Seguridad

### Medidas Implementadas

- ✅ Autenticación requerida en Grafana
- ✅ Firewall configurado (solo servidor de monitoreo accede a exporters)
- ✅ Prometheus y Loki no expuestos públicamente
- ✅ HTTPS con certificados Let's Encrypt (vía Traefik)
- ✅ Credenciales en variables de entorno

### Recomendaciones

- Cambiar password por defecto de Grafana
- Habilitar 2FA en Grafana para usuarios críticos
- Rotar credenciales de exporters periódicamente
- Usar VPN para acceso a Prometheus/AlertManager
- Configurar firewall para permitir solo IPs conocidas

## 📁 Estructura del Proyecto

```
.
├── README.md                           # Este archivo
├── GUIA-MONITOREO.md                   # Guía detallada de implementación
├── RESUMEN-MONITOREO.md                # Resumen ejecutivo
├── docker-compose.monitoring.yml       # Stack de monitoreo
├── docker-compose.exporters.yml        # Exporters para servidores
└── compose/
    └── monitoring/
        ├── prometheus/
        │   ├── prometheus.yml          # Configuración de scraping
        │   ├── alerts.yml              # Reglas de alertas
        │   └── alertmanager.yml        # Configuración de notificaciones
        ├── grafana/
        │   ├── provisioning/           # Datasources y dashboards
        │   └── dashboards/             # Dashboards personalizados
        └── loki/
            ├── loki-config.yml         # Configuración de Loki
            ├── promtail-config.yml     # Recolector local
            └── promtail-remote-config.yml  # Recolector remoto
```

## 🛠️ Configuración Avanzada

### Ajustar Retención de Datos

Editar `docker-compose.monitoring.yml`:

```yaml
prometheus:
  command:
    - '--storage.tsdb.retention.time=90d'  # Cambiar según necesidad
```

### Agregar Nuevos Targets

Editar `compose/monitoring/prometheus/prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'mi-nuevo-servidor'
    static_configs:
      - targets: ['nuevo-servidor.leohakim.dev:9100']
        labels:
          environment: 'production'
          server: 'nuevo-servidor'
```

### Configurar Notificaciones por Email

Editar `compose/monitoring/prometheus/alertmanager.yml`:

```yaml
receivers:
  - name: 'email'
    email_configs:
      - to: 'alerts@leohakim.dev'
        from: 'monitoring@leohakim.dev'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'monitoring@leohakim.dev'
        auth_password: 'tu_password'
```

## 📚 Documentación

- **[GUIA-MONITOREO.md](GUIA-MONITOREO.md)** - Guía paso a paso de implementación
- **[RESUMEN-MONITOREO.md](RESUMEN-MONITOREO.md)** - Resumen ejecutivo y decisiones técnicas
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/)

## 🐛 Troubleshooting

### Prometheus no puede alcanzar targets

```bash
# Verificar conectividad
telnet target-server.leohakim.dev 9100

# Verificar firewall
sudo ufw status
sudo ufw allow from <IP_MONITORING_SERVER> to any port 9100
```

### Grafana no muestra datos

```bash
# Verificar que Prometheus está recibiendo métricas
curl http://localhost:9090/api/v1/targets

# Verificar datasource en Grafana
Grafana → Configuration → Data Sources → Prometheus → Test
```

### Loki no recibe logs

```bash
# Verificar configuración de Promtail
docker logs monitoring_promtail

# Verificar que Loki está escuchando
curl http://localhost:3100/ready
```

### Targets aparecen como "DOWN"

```bash
# Verificar que los exporters están corriendo
docker ps | grep exporter

# Verificar logs de Prometheus
docker logs monitoring_prometheus

# Verificar firewall en servidor monitoreado
sudo ufw allow from <IP_MONITORING_SERVER> to any port 9100
sudo ufw allow from <IP_MONITORING_SERVER> to any port 8080
sudo ufw allow from <IP_MONITORING_SERVER> to any port 9187
sudo ufw allow from <IP_MONITORING_SERVER> to any port 9121
```

## 💰 Costos y Recursos

### Servidor de Monitoreo
- **CPU**: 2 cores
- **RAM**: 4GB
- **Disco**: 50-300GB (según retención)
- **Costo estimado**: $10-20/mes (VPS)

### Overhead por Servidor Monitoreado
- **CPU**: ~5%
- **RAM**: ~220MB
- **Impacto**: Mínimo

### Almacenamiento Estimado
- **Métricas**: ~1-2GB por día
- **Logs**: ~500MB-1GB por día
- **Total 90 días**: ~135-270GB

## 🤝 Contribuciones

Este es un proyecto personal, pero si encuentras algún issue o tienes sugerencias, siéntete libre de abrir un issue o pull request.

## 📄 Licencia

MIT License - Ver [LICENSE](LICENSE) para más detalles.

## 📧 Contacto

**Leo Hakim**  
SRE & Infrastructure Lead  
Email: leohakim@gmail.com  
Website: [leohakim.dev](https://leohakim.dev)  
Monitoring: [monitoring.leohakim.dev](https://monitoring.leohakim.dev)

---

**Última actualización**: Octubre 2024  
**Versión**: 1.0  
**Stack**: Prometheus 2.48 + Grafana 10.2 + Loki 2.9  
**Retención**: 90 días  
**Overhead**: ~5% CPU, ~220MB RAM por servidor
