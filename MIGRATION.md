# Guía de Migración — Servidor Prittor

**Instrucciones para IA:** Lee este archivo completo antes de ejecutar cualquier paso. Contiene toda la información necesaria para migrar la infraestructura de prittor.com a un nuevo servidor desde cero. Ejecuta los pasos en orden. No omitas ninguno.

---

## 1. Datos del servidor actual

- **IP actual:** (ver DigitalOcean dashboard o `~/.ssh/config` alias `prittor-server`)
- **Usuario:** root
- **Dominio:** prittor.com (DNS en DigitalOcean)
- **SSH alias local:** `prittor-server` (definido en `~/.ssh/config` del usuario `carodev`)

---

## 2. Requisitos del nuevo servidor

- Ubuntu 22.04+ o Debian 12+
- Docker + Docker Compose instalados
- Git instalado
- Mínimo 2GB RAM, 50GB disco
- Puertos 80, 443 abiertos
- SSH key del usuario `carodev` autorizada

---

## 3. GitHub — Clonar todos los repos

Cuenta GitHub: `Caro2870`. Todos los repos son privados. Necesita SSH key configurada en el servidor con acceso a la cuenta.

```bash
# Configurar SSH key para GitHub (copiar desde servidor anterior o generar nueva y agregar a GitHub)
ssh-keygen -t ed25519 -C "server"
# Agregar la key pública a https://github.com/settings/keys

# Clonar todos los repos en /opt/
cd /opt
git clone git@github.com:Caro2870/prittor-infra.git
git clone git@github.com:Caro2870/portafolio-2026.git
git clone git@github.com:Caro2870/finanzas.git
git clone git@github.com:Caro2870/jessica-nails.git
git clone git@github.com:Caro2870/ecommerce-nails.git
git clone git@github.com:Caro2870/llm-visualizer.git
git clone git@github.com:Caro2870/elecciones-peru-2026.git
```

**Nota:** El repo de `atiende` no está en `/opt/` como git clone. Verificar si tiene repo propio o clonar desde `Caro2870/atiende`.

---

## 4. Red Docker compartida

Todos los servicios usan una red Docker externa llamada `prittor-proxy`. Crearla ANTES de levantar cualquier servicio:

```bash
docker network create prittor-proxy
```

---

## 5. Reverse Proxy (Caddy) — LEVANTAR PRIMERO

```bash
cd /opt/prittor-infra
docker compose up -d
```

Esto levanta Caddy en puertos 80/443. Caddy genera certificados SSL automáticamente. El Caddyfile ya está en el repo.

**Verificar:** `docker ps | grep caddy` debe mostrar `prittor-caddy` corriendo.

---

## 6. Levantar cada proyecto (en este orden)

### 6.1 Portafolio (prittor.com)

```bash
cd /opt/portafolio-2026
docker compose up -d --build
```

- Sitio estático (HTML/CSS/JS + nginx)
- Sin base de datos
- Container: `portafolio-app`
- Subdominio: `prittor.com`, `www.prittor.com`

### 6.2 Finanzas

```bash
cd /opt/finanzas
```

**Crear archivo `.env.prod`** con estas variables (obtener valores del servidor anterior):

```env
MYSQL_ROOT_PASSWORD=<ver servidor anterior: /opt/finanzas/.env.prod>
MYSQL_DATABASE=finanzas_caro
MYSQL_USER=finanzas
MYSQL_PASSWORD=<ver servidor anterior>
CORS_ORIGIN=<lista de subdominios permitidos>
JWT_SECRET=<ver servidor anterior>
JWT_EXPIRES_IN=8h
DEFAULT_TENANT_KEY=default
AUTH_USERNAME=<ver servidor anterior>
AUTH_PASSWORD=<ver servidor anterior>
AUTH_PASSWORD_HASH=<ver servidor anterior>
AUTH_TENANT_ID=1
AUTH_TENANT_KEY=default
API_RATE_LIMIT_MAX=800
AUTH_RATE_LIMIT_MAX=20
```

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

- Backend: Node.js API (container `finanzas-backend-prod`)
- Frontend: Vite/React (container `finanzas-frontend-prod`)
- DB: MySQL 8.4 (container `finanzas-db-prod`)
- Multi-tenant: subdominio determina el tenant
- Subdominios: `finanzas.prittor.com`, `sam.prittor.com`, `caro.prittor.com`, `jessica.prittor.com`, `demo.prittor.com`

**Restaurar base de datos:**

```bash
# Copiar backup del servidor anterior
scp servidor-anterior:/opt/backups/finanzas_LATEST.sql.gz /opt/backups/

# Restaurar
gunzip -c /opt/backups/finanzas_LATEST.sql.gz | docker exec -i finanzas-db-prod mysql -uroot -p<ROOT_PASSWORD> finanzas_caro
```

### 6.3 Jessica Nails (Landing)

```bash
cd /opt/jessica-nails
docker compose up -d --build
```

- Sitio estático (HTML/CSS + nginx)
- Sin base de datos
- Container: `jessica-nails-app`
- Subdominios: `shop.prittor.com`, `tienda.prittor.com`
- Cache: HTML tiene `no-cache`, CSS usa `?v=N` query string

### 6.4 Ecommerce Nails (Laravel)

```bash
cd /opt/ecommerce-nails
```

**Crear archivo `.env`** con estas variables (obtener valores del servidor anterior):

```env
MYSQL_ROOT_PASSWORD=<ver servidor anterior: /opt/ecommerce-nails/.env>
NAILS_DB_DATABASE=nails_db
NAILS_DB_USERNAME=<ver servidor anterior>
NAILS_DB_PASSWORD=<ver servidor anterior>
NAILS_APP_NAME=Jessica Nails Studio
APP_URL=https://nails.prittor.com
APP_KEY=<ver servidor anterior>
NAILS_ADMIN_EMAIL=<ver servidor anterior>
NAILS_ADMIN_PASSWORD=<ver servidor anterior>
```

```bash
docker compose up -d --build
```

- App: Laravel + PHP-FPM (container `ecommerce-nails-app`)
- Web: nginx (container `ecommerce-nails-web`)
- DB: MySQL 8.4 (container `ecommerce-nails-db`)
- Subdominio: `nails.prittor.com`

**Restaurar base de datos:**

```bash
gunzip -c /opt/backups/ecommerce_nails_LATEST.sql.gz | docker exec -i ecommerce-nails-db mysql -uroot -p<ROOT_PASSWORD> nails_db
```

**Restaurar storage (uploads):**

```bash
scp -r servidor-anterior:/opt/ecommerce-nails/storage/ /opt/ecommerce-nails/storage/
scp -r servidor-anterior:/opt/ecommerce-nails/public/ /opt/ecommerce-nails/public/
```

### 6.5 LLM Visualizer

```bash
cd /opt/llm-visualizer
docker compose up -d --build
```

- Next.js 14 standalone (container `llm-visualizer-app-1`)
- Sin base de datos
- i18n: español/inglés con auto-detección
- Subdominio: `llm.prittor.com`

### 6.6 Atiende

```bash
cd /opt/atiende  # o donde esté clonado
docker compose up -d --build
```

- Next.js 14 (container `atiende-atiende-1`)
- Sin base de datos (landing estática con demo)
- Subdominio: `atiende.prittor.com`

### 6.7 Elecciones Peru 2026

```bash
cd /opt/elecciones-peru-2026
docker compose up -d --build
```

- Next.js (container `elecciones-app`)
- Sin base de datos
- Subdominio: `elecciones.prittor.com`

---

## 7. Backups automáticos

Crear directorio y script:

```bash
mkdir -p /opt/backups
```

Crear `/opt/backups/backup-dbs.sh`:

```bash
#!/bin/bash
BACKUP_DIR=/opt/backups
DATE=$(date +%Y%m%d_%H%M%S)
KEEP_DAYS=7

# Finanzas DB
docker exec finanzas-db-prod mysqldump -uroot -p<MYSQL_ROOT_PASSWORD> --single-transaction finanzas_caro | gzip > $BACKUP_DIR/finanzas_$DATE.sql.gz

# Ecommerce Nails DB
docker exec ecommerce-nails-db mysqldump -uroot -p<MYSQL_ROOT_PASSWORD> --single-transaction nails_db | gzip > $BACKUP_DIR/ecommerce_nails_$DATE.sql.gz

find $BACKUP_DIR -name '*.sql.gz' -mtime +$KEEP_DAYS -delete
echo "[$DATE] Backup completed"
```

```bash
chmod +x /opt/backups/backup-dbs.sh
```

**IMPORTANTE:** Reemplazar `<MYSQL_ROOT_PASSWORD>` con las contraseñas reales de cada DB.

---

## 8. Server Cleanup automático

Crear `/opt/server-cleanup.sh`:

```bash
#!/bin/bash
LOG="/var/log/server-cleanup.log"
D=$(date "+%Y-%m-%d %H:%M")
echo "[$D] === Cleanup started ===" >> $LOG
echo "[$D] Containers: $(docker container prune -f 2>&1 | tail -1)" >> $LOG
echo "[$D] Images: $(docker image prune -f 2>&1 | tail -1)" >> $LOG
echo "[$D] Volumes: $(docker volume prune -f 2>&1 | tail -1)" >> $LOG
echo "[$D] Networks: $(docker network prune -f 2>&1 | tail -1)" >> $LOG
docker builder prune -f --filter "until=168h" > /dev/null 2>&1
apt-get clean -y > /dev/null 2>&1
journalctl --vacuum-time=3d > /dev/null 2>&1
find /tmp -type f -mtime +7 -delete 2>/dev/null
find /var/log -name "*.gz" -mtime +7 -delete 2>/dev/null
find /var/log -name "*.old" -mtime +7 -delete 2>/dev/null
cd /opt/backups 2>/dev/null && {
  for prefix in finanzas ecommerce_nails; do
    ls -1t ${prefix}_*.sql.gz 2>/dev/null | tail -n +3 | xargs rm -f 2>/dev/null
  done
}
cat /dev/null > /var/log/btmp 2>/dev/null
DISK=$(df -h / | tail -1 | awk "{print \$3 \" / \" \$2 \" (\" \$5 \")\"}")
echo "[$D] Disk: $DISK" >> $LOG
echo "[$D] === Done ===" >> $LOG
```

```bash
chmod +x /opt/server-cleanup.sh
```

---

## 9. Cron jobs

```bash
crontab -e
```

Agregar:

```cron
0 3 * * * /opt/backups/backup-dbs.sh >> /opt/backups/backup.log 2>&1
0 4 */3 * * /opt/server-cleanup.sh
```

---

## 10. DNS — Apuntar dominio al nuevo servidor

En DigitalOcean (o donde esté el DNS de `prittor.com`), actualizar los registros A:

| Registro | Tipo | Valor |
|----------|------|-------|
| `@` | A | `<NUEVA_IP>` |
| `www` | A | `<NUEVA_IP>` |
| `finanzas` | A | `<NUEVA_IP>` |
| `sam` | A | `<NUEVA_IP>` |
| `caro` | A | `<NUEVA_IP>` |
| `jessica` | A | `<NUEVA_IP>` |
| `demo` | A | `<NUEVA_IP>` |
| `shop` | A | `<NUEVA_IP>` |
| `tienda` | A | `<NUEVA_IP>` |
| `nails` | A | `<NUEVA_IP>` |
| `atiende` | A | `<NUEVA_IP>` |
| `llm` | A | `<NUEVA_IP>` |
| `elecciones` | A | `<NUEVA_IP>` |

**Tip:** Puedes usar un wildcard `*.prittor.com → <NUEVA_IP>` para cubrir todos los subdominios.

---

## 11. Actualizar SSH config local

En la máquina del usuario (`carodev`), actualizar `~/.ssh/config`:

```
Host prittor-server
  HostName <NUEVA_IP>
  User root
  IdentityFile ~/.ssh/id_ed25519_personal
```

---

## 12. Verificación post-migración

Ejecutar estos checks después de migrar:

```bash
# Todos los containers corriendo
docker ps --format "table {{.Names}}\t{{.Status}}"

# Test cada subdominio
curl -sI https://prittor.com | head -1
curl -sI https://finanzas.prittor.com | head -1
curl -sI https://shop.prittor.com | head -1
curl -sI https://nails.prittor.com | head -1
curl -sI https://atiende.prittor.com | head -1
curl -sI https://llm.prittor.com | head -1
curl -sI https://elecciones.prittor.com | head -1

# Verificar SSL
echo | openssl s_client -connect prittor.com:443 2>/dev/null | grep "subject="

# Verificar cron
crontab -l

# Verificar backups
/opt/backups/backup-dbs.sh && ls -la /opt/backups/
```

---

## 13. Resumen de secretos a copiar del servidor anterior

Estos archivos contienen credenciales que NO están en los repos:

| Archivo | Proyecto |
|---------|----------|
| `/opt/finanzas/.env.prod` | Finanzas (DB, JWT, auth) |
| `/opt/ecommerce-nails/.env` | Ecommerce Nails (DB, APP_KEY) |
| `/opt/backups/backup-dbs.sh` | Contraseñas de MySQL para dumps |

**Copiar todo de golpe:**

```bash
# Desde el nuevo servidor:
scp servidor-anterior:/opt/finanzas/.env.prod /opt/finanzas/.env.prod
scp servidor-anterior:/opt/ecommerce-nails/.env /opt/ecommerce-nails/.env
scp servidor-anterior:/opt/backups/backup-dbs.sh /opt/backups/backup-dbs.sh
```

---

## 14. Orden completo de ejecución (copiar y pegar)

```bash
# 1. Red
docker network create prittor-proxy

# 2. Clonar repos
cd /opt
git clone git@github.com:Caro2870/prittor-infra.git
git clone git@github.com:Caro2870/portafolio-2026.git
git clone git@github.com:Caro2870/finanzas.git
git clone git@github.com:Caro2870/jessica-nails.git
git clone git@github.com:Caro2870/ecommerce-nails.git
git clone git@github.com:Caro2870/llm-visualizer.git
git clone git@github.com:Caro2870/elecciones-peru-2026.git

# 3. Copiar secretos del servidor anterior
scp servidor-anterior:/opt/finanzas/.env.prod /opt/finanzas/.env.prod
scp servidor-anterior:/opt/ecommerce-nails/.env /opt/ecommerce-nails/.env
scp servidor-anterior:/opt/backups/backup-dbs.sh /opt/backups/backup-dbs.sh

# 4. Copiar backups de DBs
mkdir -p /opt/backups
scp servidor-anterior:/opt/backups/finanzas_*.sql.gz /opt/backups/
scp servidor-anterior:/opt/backups/ecommerce_nails_*.sql.gz /opt/backups/

# 5. Copiar ecommerce storage
scp -r servidor-anterior:/opt/ecommerce-nails/storage/ /opt/ecommerce-nails/storage/
scp -r servidor-anterior:/opt/ecommerce-nails/public/ /opt/ecommerce-nails/public/

# 6. Levantar Caddy primero
cd /opt/prittor-infra && docker compose up -d

# 7. Levantar servicios
cd /opt/portafolio-2026 && docker compose up -d --build
cd /opt/finanzas && docker compose -f docker-compose.prod.yml up -d --build
cd /opt/jessica-nails && docker compose up -d --build
cd /opt/ecommerce-nails && docker compose up -d --build
cd /opt/llm-visualizer && docker compose up -d --build
cd /opt/elecciones-peru-2026 && docker compose up -d --build

# 8. Restaurar DBs
gunzip -c /opt/backups/finanzas_*.sql.gz | docker exec -i finanzas-db-prod mysql -uroot -p<PASSWORD> finanzas_caro
gunzip -c /opt/backups/ecommerce_nails_*.sql.gz | docker exec -i ecommerce-nails-db mysql -uroot -p<PASSWORD> nails_db

# 9. Cleanup + Backup scripts
cp /opt/server-cleanup.sh /opt/server-cleanup.sh 2>/dev/null
chmod +x /opt/backups/backup-dbs.sh /opt/server-cleanup.sh

# 10. Cron
(crontab -l 2>/dev/null; echo '0 3 * * * /opt/backups/backup-dbs.sh >> /opt/backups/backup.log 2>&1'; echo '0 4 */3 * * /opt/server-cleanup.sh') | crontab -

# 11. Actualizar DNS a nueva IP
# 12. Actualizar ~/.ssh/config local con nueva IP
```
