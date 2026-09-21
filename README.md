# 🕷️ Trawl - Motor Scraping Adaptativo Autohospedado

[![GitHub](https://img.shields.io/badge/GitHub-germondai%2Ftrawl-blue?logo=github)](https://github.com/germondai/trawl)
[![Docker](https://img.shields.io/badge/Docker-ghcr.io%2Fgermondai%2Ftrawl-blue?logo=docker)](https://github.com/germondai/trawl/pkgs/container/trawl)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## 📋 Descripción general

**Trawl** es un motor de scraping web self-hosted que resuelve desafíos de protección (Cloudflare, captchas) de forma nativa sin APIs externas, permitiendo raspado de contenido web con total privacidad y cero cuotas/tokens. Es compatible **drop-in replacement** con el stack *arr (Prowlarr, Jackett, Sonarr, Radarr) como reemplazo directo de FlareSolverr.

- **Resolución nativa Cloudflare**: Challenges resueltos en 4-15 segundos
- **Captchas automáticos**: Turnstile, reCAPTCHA v2/v3, hCaptcha, GeeTest, Imperva (experimental)
- **Caching agresivo Redis**: Sub-500ms en requests repetidos por dominio
- **Estrategia adaptativa 5 tiers**: fetch → CF cached → CF fresh → Browser context → Custom headers
- **FlareSolverr compatible**: API compatible en `http://trawl:8191`
- **Dos builds hardware**: `:latest` (AVX2 moderno) y `:baseline` (kernel 4.4+, Synology compatible)
- **Multiarch**: amd64, arm64 (Raspberry Pi, Synology, x86 server)
- **Bun runtime**: Alto rendimiento, boot 15-30s (pool browsers warmup)

## ✨ Características principales

- 🛡️ **Bypass nativo Cloudflare** — Resuelve challenges en 4-15 seg sin APIs externas, cero cuotas
- 🤖 **Captchas automáticos** — Turnstile, reCAPTCHA v2/v3, hCaptcha, GeeTest, Imperva (experimental) integrados
- ⚡ **Caching agresivo Redis** — Sub-500ms repeats, persistencia por dominio, session pooling
- 🧠 **Estrategia adaptativa 5 tiers** — Fetch simple → Cloudflare cached → Cloudflare fresh → Browser context → Custom headers
- 🔄 **FlareSolverr compatible** — Drop-in replacement Prowlarr, Jackett, Sonarr, Radarr (*arr stack)
- 📡 **API REST simple** — v1 (metadata enriquecida: tier usado, timings, sessionCached, cookies) + endpoint `/scrape`
- 🏷️ **Dos builds hardware** — `:latest` (AVX2 moderno) + `:baseline` (kernel 4.4+, Synology compatible)
- 🌐 **Browser pool + warmup** — Boot 15-30s, subsecuentes rápidos, browsers pooled reutilizables
- 📦 **Multiarch Docker** — amd64, arm64 (Raspberry Pi, Synology, x86 server compatible)
- 📊 **Logging + métricas debug** — Verbose logs, timing data, tier utilizado visible
- 🐳 **Docker Compose simple** — Scraper + Redis included, `.env` config, un comando up
- 📄 **MIT open source** — Código abierto, activamente mantenido, comunidad GitHub

## 📋 Requisitos del sistema

- **Docker** + **Docker Compose v2+**
- **2 GB - 8 GB RAM** mínimo (browser pool + Redis)
- **10 GB espacio disco** (imagen + cache + logs)
- **Puerto 8191** (configurable, API scraping)
- **Redis integrado** (para caching sesiones, incluido en compose)
- **CPU moderno (AVX2)** para tag `:latest` — O **baseline kernel 4.4+** para tag `:baseline` (dos builds disponibles)
- **Conexión red estable** (HTTP/HTTPS outbound)
- **Opcional**: Prowlarr/Jackett/Sonarr/Radarr (para integración *arr)
- **Boot time**: Primera ejecución 15-30 seg (browser pool warms), ejecuciones subsecuentes rápidas (cached)

## 🐳 Instalación

### Opción 1: Git clone + docker compose (recomendado)

```bash
git clone https://github.com/germondai/trawl.git
cd trawl
cp .env.example .env
docker compose up -d
# Verificar health check
curl http://localhost:8191/health
```

### Opción 2: Docker Compose manual (control manual)

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  trawl:
    image: ghcr.io/germondai/trawl:latest
    container_name: trawl
    restart: unless-stopped
    ports:
      - "8191:8191"
    environment:
      - TRAWL_PORT=8191
      # Redis connection (local container por defecto)
      - REDIS_URL=redis://redis:6379
      # Logging level: debug, info, warn, error
      - LOG_LEVEL=info
    depends_on:
      - redis
    volumes:
      - trawl_cache:/tmp/trawl

  redis:
    image: redis:7-alpine
    container_name: trawl_redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  trawl_cache:
  redis_data:
EOF

docker compose up -d
```

### Opción 3: Hardware antiguo (Synology, kernel 4.4+)

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  trawl:
    image: ghcr.io/germondai/trawl:baseline  # :baseline instead of :latest
    container_name: trawl
    restart: unless-stopped
    ports:
      - "8191:8191"
    environment:
      - TRAWL_PORT=8191
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=info
    depends_on:
      - redis
    volumes:
      - trawl_cache:/tmp/trawl

  redis:
    image: redis:7-alpine
    container_name: trawl_redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  trawl_cache:
  redis_data:
EOF

docker compose up -d
```

### Acceder (API disponible)

- **Health check**: `http://localhost:8191/health`
- **API v1**: `http://localhost:8191/v1`
- **Scrape simple**: `http://localhost:8191/scrape`

### Primer test (scraping simple)

```bash
curl -X POST http://localhost:8191/v1 \
  -H 'Content-Type: application/json' \
  -d '{ "cmd":"request.get", "url":"https://nowsecure.nl", "maxTimeout":60000 }'
# Respuesta incluye: tier usado, html, cookies, timings, sessionCached
```

## ⚙️ Configuración

1. **Variables de entorno principales** (en `.env` o `docker-compose.yml`):
   - `TRAWL_PORT=8191` — Puerto API
   - `REDIS_URL=redis://redis:6379` — Conexión Redis
   - `LOG_LEVEL=info` — Nivel logging (debug, info, warn, error)
   - `BROWSER_POOL_SIZE=5` — Tamaño pool browsers
   - `SESSION_CACHE_TTL=3600` — TTL cache sesiones (segundos)

2. **Docker secrets soportados** — Para credenciales sensibles

3. **Health checks integrados** — Verificación automática estado contenedor

4. **Volúmenes persistentes** — `trawl_cache` (browser temp) + `redis_data` (cache sessions)

5. **Red Docker interna** — Comunicación trawl ↔ redis aislada

## 🚀 Primeros pasos

1. **Verificar health (container running)**
   ```bash
   curl http://localhost:8191/health
   # Respuesta: {"status":"ok"} si OK
   ```

2. **Test scraping simple (sin protección)**
   ```bash
   curl -X POST http://localhost:8191/v1 \
     -H 'Content-Type: application/json' \
     -d '{ "cmd":"request.get", "url":"https://example.com", "maxTimeout":60000 }'
   ```

3. **Test con Cloudflare (captcha automático)**
   ```bash
   curl -X POST http://localhost:8191/v1 \
     -H 'Content-Type: application/json' \
     -d '{ "cmd":"request.get", "url":"https://nowsecure.nl", "maxTimeout":60000 }'
   # Primera ejecución: bypasa CF (4-15seg)
   # Segunda ejecución: resultado cached (<500ms)
   ```

4. **Simpler scrape endpoint (si Prowlarr/Jackett espera)**
   ```bash
   curl -X POST http://localhost:8191/scrape \
     -H 'Content-Type: application/json' \
     -d '{ "url":"https://example.com", "maxTimeout":60000 }'
   ```

5. **Integrar con Prowlarr (reemplazo FlareSolverr)**
   - Prowlarr → Settings → Clients
   - Busca "FlareSolverr" (Trawl es compatible)
   - URL: `http://trawl:8191` (si en Docker Compose) o `http://localhost:8191` (docker bridge)
   - Guarda → Prowlarr ahora usa Trawl para bypasear Cloudflare

6. **Integrar con Jackett**
   - Jackett → Indexer Proxy Settings
   - Proxy: "Flaresolverr"
   - Host: `http://trawl:8191`
   - Trawl resuelve CF antes retornar resultados

7. **Ver logs detallados**
   ```bash
   docker logs -f trawl
   # Mostrará tier usado, timings, browsers activity
   ```

8. **Configuración avanzada (.env)**
   ```bash
   cat > .env << 'EOF'
   TRAWL_PORT=8191
   REDIS_URL=redis://redis:6379
   LOG_LEVEL=debug
   BROWSER_POOL_SIZE=5
   SESSION_CACHE_TTL=3600
   EOF
   ```

9. **Monitorear consumo (browser pool)**
   ```bash
   docker stats trawl trawl_redis
   # Trawl: típicamente 300-500MB RAM (browsers) + Redis
   ```

## 💡 Casos de uso

- 🎬 **\*arr stack (Prowlarr, Jackett, Sonarr, Radarr)**: Bypass Cloudflare nativo, reemplaza FlareSolverr, cero cuotas
- 🕸️ **Web scraping automatizado**: Sitios con Cloudflare, datos públicos, automatización inteligente
- 📊 **Data collection**: APIs bloqueadas por CF, scraping escalable, caching eficiente
- 🔄 **Reverse proxy scraping**: Integrable en cualquier stack web, API REST simple
- 🧪 **Testing web apps**: Simula interacción usuario, resuelve captchas automático, QA automation

## 🔒 Acceso remoto seguro

### HTTPS con Caddy (exponer scraper remoto seguro)

```bash
# Caddyfile
trawl.tudominio.com {
    reverse_proxy localhost:8191
    # Opcional: proteger con basic auth
    basicauth / {
        usuario contraseña_fuerte_hash
    }
}
```

- Acceso: `https://trawl.tudominio.com` con HTTPS automático + auth
- **IMPORTANTE**: Trawl NO debe exponerse sin autenticación
- Usar Caddy `basic_auth` o firewall network-level
- Limitar acceso solo a *arr stack

## 🛠️ Gestión y mantenimiento

### Ver logs detallados
```bash
docker logs -f trawl
# Muestra tier usado, timings, bypass strategies
```

### Limpiar cache Redis (si needed)
```bash
# ⚠️ Limpia TODO cache. Siguientes requests serán lentos.
docker exec trawl_redis redis-cli FLUSHALL
```

### Limpieza selectiva por dominio
```bash
docker exec trawl_redis redis-cli KEYS "example.com*" | xargs docker exec trawl_redis redis-cli DEL
```

### Reiniciar containers
```bash
docker compose restart
# O solo Trawl
docker compose restart trawl
```

### Actualizar a versión más reciente
```bash
docker compose pull
docker compose up -d
# O en repo clonado
cd trawl
git pull
docker compose pull
docker compose up -d
```

### Monitorear consumo (browsers + Redis)
```bash
docker stats trawl trawl_redis
# Trawl: 300-500MB RAM (pools), Redis: 50-200MB (cache)
```

### Backup Redis cache (importante)
```bash
docker exec trawl_redis redis-cli BGSAVE
docker cp trawl_redis:/data/dump.rdb ./redis-backup-$(date +%Y%m%d).rdb
```

## 📝 Licencia

MIT License — Código abierto, activamente mantenido. Ver [LICENSE](LICENSE) para detalles.

---

> 📖 **Basado en el post**: [Cómo instalar Trawl en Docker - Motor scraping adaptativo que bypasa Cloudflare autohospedado](https://genbyte.blogspot.com/2026/07/como-instalar-trawl-en-docker-motor.html)