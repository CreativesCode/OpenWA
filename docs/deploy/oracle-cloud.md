# OpenWA — Despliegue completo en Oracle Cloud Free Tier

Guía paso a paso (de verdad, paso a paso) para desplegar OpenWA en una VPS gratuita de Oracle Cloud, con dominio propio y HTTPS automático.

**Tiempo total estimado**: 60-90 minutos (la mayor parte es esperar verificación de cuenta de Oracle y propagación DNS).

**Resultado final**: `https://open-wa.whizzlyplus.com/` corriendo con dashboard, API y SSL válido.

---

## Índice

1. [Por qué Oracle Cloud Free Tier](#1-por-qué-oracle-cloud-free-tier)
2. [Crear cuenta de Oracle Cloud](#2-crear-cuenta-de-oracle-cloud)
3. [Crear la VPS (Compute Instance)](#3-crear-la-vps-compute-instance)
4. [Configurar la red — Security List](#4-configurar-la-red--security-list)
5. [Conectar por SSH](#5-conectar-por-ssh)
6. [Abrir puertos en el firewall del sistema](#6-abrir-puertos-en-el-firewall-del-sistema)
7. [Instalar Docker](#7-instalar-docker)
8. [Configurar el DNS de tu dominio](#8-configurar-el-dns-de-tu-dominio)
9. [Clonar OpenWA desde GitHub](#9-clonar-openwa-desde-github)
10. [Configurar el archivo `.env`](#10-configurar-el-archivo-env)
11. [Levantar OpenWA con Docker](#11-levantar-openwa-con-docker)
12. [Verificar que todo funciona](#12-verificar-que-todo-funciona)
13. [Crear primera sesión de WhatsApp](#13-crear-primera-sesión-de-whatsapp)
14. [Mantenimiento](#14-mantenimiento)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. Por qué Oracle Cloud Free Tier

Oracle Cloud ofrece un **"Always Free"** tier (gratis para siempre, no de prueba) con:

- **4 vCPU ARM Ampere A1** + **24 GB RAM** + **200 GB disco** — más que suficiente para muchas sesiones de WhatsApp
- IP pública IPv4
- 10 TB de tráfico saliente al mes
- 100% gratis sin caducidad si te mantienes dentro del tier

Comparado con shared hosting cPanel, esto es **infinitamente más estable y flexible**.

### Letra pequeña

- Requiere tarjeta de crédito para verificación (no cobran nada, solo validan)
- La verificación puede tardar de 1 hora a 3 días
- Una vez creas la cuenta, eliges una "Home Region" que **no se puede cambiar** después
- Si Oracle detecta abuso (minar, spam, etc.) suspenden la cuenta

---

## 2. Crear cuenta de Oracle Cloud

### 2.1 Registro

1. Abre https://www.oracle.com/cloud/free/ en el navegador
2. Click en **"Start for free"** (arriba a la derecha)
3. Rellena el formulario:
   - **Country/Territory**: tu país real (afecta a qué centro de datos te asignan)
   - **Email**: usa uno que revises (te llegan códigos de verificación)
4. Verifica el email (te llega un código)

### 2.2 Datos personales y tarjeta

1. Completa nombre + dirección
2. **Home Region**: elige la más cercana a donde estarás operando WhatsApp. Opciones recomendadas:
   - 🇪🇸 / 🇪🇺 Europa → **Frankfurt** o **Madrid**
   - 🇺🇸 / LATAM → **Phoenix**, **Ashburn** o **São Paulo**
   - 🇲🇽 / 🇨🇴 / 🇵🇪 / 🇨🇱 → **São Paulo** o **Phoenix**

   > ⚠️ Esto NO se puede cambiar después. Elige bien.

3. Verifica el teléfono (te llega SMS con código)
4. **Tarjeta de crédito**: introduce una válida. Oracle hace un cargo de verificación de ~$1 USD que devuelven en 24-48 h. No cobran nada más mientras estés en el tier gratuito.
5. Acepta términos y click en **"Start my free trial"**

### 2.3 Esperar aprobación

Oracle revisa la cuenta. Puede tardar:
- ⚡ Inmediato si todo cuadra
- ⏰ 1-3 días si la verificación requiere intervención manual

Recibes email cuando esté lista (asunto típico: "Welcome to Oracle Cloud Infrastructure").

---

## 3. Crear la VPS (Compute Instance)

### 3.1 Entrar a la consola

1. Loguéate en https://cloud.oracle.com/
2. Te aparece el **Dashboard** principal
3. En el menú hamburguesa (☰ arriba izquierda) → **Compute** → **Instances**

### 3.2 Crear instancia

1. Click azul **"Create instance"**
2. **Name**: `openwa-prod` (o como quieras)
3. **Compartment**: déjalo en el default (`yourname (root)`)

### 3.3 Image and shape

Click **"Edit"** en la sección "Image and shape".

**Image:**
- Click **"Change image"**
- Selecciona **"Canonical Ubuntu"**
- Versión: **22.04** (la más estable para Docker en ARM)
- Imagen: la que diga **"aarch64"** al final (es la versión ARM)
- Click **"Select image"**

**Shape:**
- Click **"Change shape"**
- En la tabla, busca **"Ampere"** (sección "Specialty and previous generation" o similar)
- Selecciona **"VM.Standard.A1.Flex"**
- Configura:
  - **OCPU**: `4` (el máximo del free tier)
  - **Memory**: `24` GB (el máximo del free tier)
- Click **"Select shape"**

> 💡 Si te dice "Out of capacity for this shape" → es porque Oracle no tiene ARM disponible en ese momento en tu región. Tienes 3 opciones:
> 1. **Reintentar** en unas horas (suele liberarse)
> 2. **Cambiar a otra Availability Domain** (AD-1, AD-2, AD-3 — botón arriba de la página)
> 3. **Usar el shape AMD** `VM.Standard.E2.1.Micro` (1 OCPU + 1 GB RAM, también free) — pero será MUY justo para Chromium. **No recomendado** para OpenWA.

### 3.4 Networking

Déjalo en automático:
- **Virtual cloud network**: "Create new virtual cloud network"
- **Subnet**: "Create new public subnet"
- ✅ **Assign a public IPv4 address**: marcado

### 3.5 SSH keys

**Tienes 2 opciones**:

#### Opción A — Generar nueva clave (recomendado si no tienes una)

1. Selecciona **"Generate a key pair for me"**
2. Click **"Save private key"** → te descarga `ssh-key-XXXX.key` (¡guárdala bien!)
3. Click **"Save public key"** → te descarga `ssh-key-XXXX.key.pub`

#### Opción B — Subir clave que ya tienes

Si ya tienes una en tu Windows (`C:\Users\TU_USER\.ssh\id_rsa.pub`):

1. Selecciona **"Upload public key files (.pub)"** o **"Paste public keys"**
2. Sube o pega el contenido

### 3.6 Boot volume

- Deja **"Specify a custom boot volume size"** marcado
- **Boot volume size**: `50` GB (free tier permite hasta 200 GB total entre todas tus instancias)
- VPU: **10** (default, suficiente)

### 3.7 Crear

Click azul **"Create"** abajo.

Espera 30-60 segundos. La instancia pasa de **"Provisioning"** (amarillo) → **"Running"** (verde).

### 3.8 Anotar datos importantes

Cuando esté en **Running**, en la página de la instancia anota:

- **Public IP Address**: algo como `132.226.45.123`. **CRÍTICO** — la necesitarás para todo.
- **Username**: `ubuntu` (default para Ubuntu)

---

## 4. Configurar la red — Security List

Oracle bloquea TODOS los puertos por defecto excepto SSH (22). Hay que abrir 80 y 443 manualmente.

### 4.1 Encontrar la Security List

1. En la página de tu instancia, scroll a la sección **"Instance details"**
2. En el panel izquierdo busca **"Virtual cloud network"** → click en el nombre de tu VCN
3. En la página del VCN, panel izquierdo → **"Security Lists"**
4. Click en **"Default Security List for vcn-XXXX"**

### 4.2 Añadir reglas Ingress

Click en **"Add Ingress Rules"**.

**Regla 1 — HTTP (puerto 80):**
- Stateless: **No**
- Source Type: **CIDR**
- Source CIDR: `0.0.0.0/0`
- IP Protocol: **TCP**
- Source Port Range: (vacío)
- Destination Port Range: `80`
- Description: `HTTP for Let's Encrypt and redirect`

Click **"+ Another Ingress Rule"** para añadir la segunda:

**Regla 2 — HTTPS (puerto 443):**
- Stateless: **No**
- Source Type: **CIDR**
- Source CIDR: `0.0.0.0/0`
- IP Protocol: **TCP**
- Source Port Range: (vacío)
- Destination Port Range: `443`
- Description: `HTTPS for OpenWA`

Click azul **"Add Ingress Rules"**.

Ahora la Security List acepta tráfico en 22, 80 y 443.

---

## 5. Conectar por SSH

### 5.1 Desde Windows (PowerShell, recomendado)

Abre PowerShell (Windows + R → `powershell`):

```powershell
# Sustituye estos 2 valores:
$KEY = "C:\Users\TU_USUARIO\Downloads\ssh-key-XXXX.key"
$IP = "132.226.45.123"   # la IP pública de tu VPS

# Bloquear permisos de la clave (Oracle exige esto)
icacls $KEY /inheritance:r /grant:r "${env:USERNAME}:R"

# Conectar
ssh -i $KEY ubuntu@$IP
```

Te preguntará algo como `Are you sure you want to continue connecting (yes/no/[fingerprint])?` → escribe `yes` + Enter.

Si todo va bien, ves:
```
ubuntu@openwa-prod:~$
```

🎉 ¡Estás dentro de tu VPS!

### 5.2 Si te da "Permission denied (publickey)"

- Asegúrate que el archivo `.key` (clave privada) sea el correcto (no el `.pub`)
- Verifica la IP pública
- Verifica el username (es `ubuntu` para Ubuntu, `opc` para Oracle Linux)
- Verifica que ejecutaste el comando `icacls` para bloquear permisos

---

## 6. Abrir puertos en el firewall del sistema

⚠️ **MUY IMPORTANTE — paso que casi todo el mundo olvida.** Oracle Ubuntu trae `iptables` configurado para bloquear TODO excepto SSH, aunque la Security List esté abierta. Hay que abrirlos también a nivel OS.

Pega estos comandos:

```bash
# Abrir HTTP (80) y HTTPS (443) en iptables
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT

# Hacer las reglas persistentes (sobreviven a reboot)
sudo netfilter-persistent save

# Verificar
sudo iptables -L INPUT -n --line-numbers | head -15
```

Debes ver entradas con `ACCEPT ... dpt:80` y `dpt:443`.

---

## 7. Instalar Docker

Pega los siguientes comandos (uno a uno o todos juntos):

```bash
# Actualizar el sistema
sudo apt update && sudo apt upgrade -y

# Dependencias previas
sudo apt install -y ca-certificates curl gnupg git nano

# Añadir repo oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker Engine + Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Habilitar Docker al arranque
sudo systemctl enable --now docker

# Añadir tu usuario al grupo docker (evita escribir sudo siempre)
sudo usermod -aG docker $USER

# Verificar
docker --version
docker compose version
```

Salida esperada:
```
Docker version 26.x.x, build ...
Docker Compose version v2.x.x
```

**Cierra y reabre la sesión SSH** para que tu usuario tenga acceso al grupo docker sin sudo:

```bash
exit
# Y vuelves a hacer ssh -i ...
```

Verifica al volver:
```bash
docker run hello-world
```

Debe imprimir un mensaje de "Hello from Docker!".

---

## 8. Configurar el DNS de tu dominio

Necesitas que `open-wa.whizzlyplus.com` apunte a la IP pública del VPS Oracle.

### 8.1 Ir al panel del dominio

Probablemente sea **Cloudflare**, **Banahosting cPanel** o el panel de tu registrador. Busca la sección de **DNS** / **Zone Editor**.

### 8.2 Crear o editar el A record

| Campo | Valor |
|---|---|
| **Type** | A |
| **Name** | `open-wa` (sin el `.whizzlyplus.com`) |
| **Value / Points to** | la IP pública de tu Oracle VPS (ej `132.226.45.123`) |
| **TTL** | 300 (o "Auto") |
| **Proxy status** (Cloudflare) | ⚠️ **DNS only** (nube gris). NO uses el proxy naranja, rompe Let's Encrypt. |

Guarda.

### 8.3 Verificar propagación (desde tu PC)

En PowerShell:
```powershell
nslookup open-wa.whizzlyplus.com 8.8.8.8
```

Debe devolver la IP del Oracle VPS. Si devuelve otra cosa (o nada), espera 5-15 min y reintenta.

> ⚠️ **No avances hasta que el DNS resuelva correctamente.** Let's Encrypt fallará si el dominio no apunta al servidor cuando intente emitir el certificado.

---

## 9. Clonar OpenWA desde GitHub

De vuelta en SSH del VPS:

```bash
# Ir al directorio /opt (estándar para apps)
cd /opt

# Clonar (sudo porque /opt es del root)
sudo git clone https://github.com/rmyndharis/OpenWA.git
sudo chown -R $USER:$USER OpenWA
cd OpenWA

# Confirmar que estás en main
git status
```

---

## 10. Configurar el archivo `.env`

### 10.1 Generar API_MASTER_KEY

```bash
# Generar clave aleatoria fuerte
API_KEY=$(openssl rand -hex 32)
echo "Tu API_MASTER_KEY (guárdala en lugar seguro):"
echo $API_KEY
```

**COPIA ESA CLAVE** a un gestor de contraseñas. La necesitarás para autenticarte con la API.

### 10.2 Crear el `.env` desde cero

```bash
nano /opt/OpenWA/.env
```

Pega EXACTAMENTE esto (reemplaza los valores marcados con `⚠️`):

```env
# =============================================================================
# OpenWA - Producción Oracle Cloud
# Despliegue: https://open-wa.whizzlyplus.com/
# =============================================================================

# --- CORE ---
NODE_ENV=production
API_PORT=2785
LOG_LEVEL=info

# --- DOMINIO ---
DOMAIN=open-wa.whizzlyplus.com
DASHBOARD_PORT=2886

BASE_URL=https://open-wa.whizzlyplus.com
DASHBOARD_URL=https://open-wa.whizzlyplus.com

CORS_ORIGINS=https://open-wa.whizzlyplus.com

# --- SSL/TLS (Traefik + Let's Encrypt) ---
# ⚠️ Pon TU email real (Let's Encrypt avisa cuando caducan certs)
TRAEFIK_ACME_EMAIL=admin@whizzlyplus.com
TRAEFIK_ACME_STORAGE=/letsencrypt/acme.json

# --- COMPONENTES ---
DASHBOARD_ENABLED=true
PROXY_ENABLED=true

# --- ENGINE WhatsApp ---
ENGINE_TYPE=whatsapp-web.js
SESSION_DATA_PATH=./data/sessions
PUPPETEER_HEADLESS=true
PUPPETEER_ARGS=--no-sandbox,--disable-setuid-sandbox,--disable-dev-shm-usage,--disable-gpu

# --- BASE DE DATOS (SQLite, simple) ---
DATABASE_TYPE=sqlite
POSTGRES_BUILTIN=false
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=openwa
DATABASE_USERNAME=openwa
DATABASE_PASSWORD=ignored-with-sqlite
DATABASE_SYNCHRONIZE=false
DATABASE_LOGGING=false

# --- REDIS (deshabilitado al inicio) ---
REDIS_ENABLED=false
REDIS_BUILTIN=false
REDIS_HOST=localhost
REDIS_PORT=6379

# --- STORAGE (local) ---
STORAGE_TYPE=local
MINIO_BUILTIN=false
STORAGE_LOCAL_PATH=./data/media

# --- WEBHOOK ---
WEBHOOK_TIMEOUT=10000
WEBHOOK_MAX_RETRIES=3
WEBHOOK_RETRY_DELAY=5000

# --- RATE LIMITING ---
RATE_LIMIT_TTL=60
RATE_LIMIT_MAX=100

# --- PLUGINS ---
PLUGINS_ENABLED=true
PLUGINS_DIR=./data/plugins

# --- SECURITY ⚠️ REEMPLAZAR ---
# Pegar aquí la clave que generaste con openssl rand -hex 32
API_MASTER_KEY=PEGAR_AQUI_TU_CLAVE_GENERADA

# --- SWAGGER ---
ENABLE_SWAGGER=true
```

Guarda: `Ctrl+O`, `Enter`, `Ctrl+X`.

### 10.3 Verificar que el .env quedó bien

```bash
cd /opt/OpenWA

# Comprobar que existe API_MASTER_KEY y NO es el placeholder
grep API_MASTER_KEY .env

# Comprobar que el dominio está correcto
grep DOMAIN .env
```

> ⚠️ Si dejaste `PEGAR_AQUI_TU_CLAVE_GENERADA`, vuelve a editar y pega la clave real.

---

## 11. Levantar OpenWA con Docker

```bash
cd /opt/OpenWA

# Construir y levantar todos los contenedores en background
# (la primera vez tarda ~5-10 min porque construye la imagen ARM)
docker compose -f docker-compose.yml -f docker-compose.prod.yml --profile full up -d --build

# Ver los logs en vivo (Ctrl+C para salir, no detiene los contenedores)
docker compose -f docker-compose.yml -f docker-compose.prod.yml logs -f
```

Verás logs de varios servicios:
- `openwa-traefik`: arranca, valida certificados
- `openwa-api`: NestJS arrancando módulos, al final `🚀 OpenWA is running on...`
- `openwa-dashboard`: nginx sirviendo el frontend React

Cuando veas en los logs de Traefik algo como:
```
... ACME: register account ...
... obtained certificate for "open-wa.whizzlyplus.com"
```
**Let's Encrypt funcionó. SSL listo.**

Ctrl+C para salir de los logs (los contenedores siguen corriendo).

### Verificar el estado

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps
```

Todos deben estar en `Up` o `Up (healthy)`.

---

## 12. Verificar que todo funciona

### 12.1 Desde el VPS

```bash
# La API responde localmente
curl -sI http://localhost:2785/api/health

# La API responde vía Traefik (HTTPS)
curl -sI https://open-wa.whizzlyplus.com/api/health
```

Ambos deben dar `HTTP/1.1 200 OK` (o `HTTP/2 200` el segundo).

### 12.2 Desde tu PC (navegador)

Abre en el navegador:

1. **Dashboard**: https://open-wa.whizzlyplus.com/
   - Debe cargar la UI de OpenWA
   - SSL verde sin warnings

2. **Swagger API docs**: https://open-wa.whizzlyplus.com/api/docs
   - Debe cargar Swagger UI listando todos los endpoints

3. **Health check**: https://open-wa.whizzlyplus.com/api/health
   - Debe mostrar JSON tipo `{"status":"ok","timestamp":"..."}`

Si los 3 funcionan: 🎉 **OpenWA desplegado correctamente**.

---

## 13. Crear primera sesión de WhatsApp

### 13.1 Desde el dashboard (más fácil)

1. Abre https://open-wa.whizzlyplus.com/
2. En el panel izquierdo busca **"API Keys"** → crea una si no existe (usa tu `API_MASTER_KEY` para autenticarte la primera vez).
3. Ve a **"Sessions"** → **"Create Session"**
4. Dale un nombre (ej: `bot-ventas`) → Create
5. Click en **"Start"**
6. Aparece un QR
7. En tu teléfono: WhatsApp → Configuración → Dispositivos vinculados → Vincular dispositivo → escanea el QR
8. ¡Listo! La sesión queda activa permanentemente.

### 13.2 Desde la API (alternativa)

```bash
# Crear sesión
curl -X POST https://open-wa.whizzlyplus.com/api/sessions \
  -H "Content-Type: application/json" \
  -H "X-API-Key: TU_API_MASTER_KEY" \
  -d '{"name": "bot-ventas"}'

# Iniciar sesión (te devuelve QR como base64 o URL)
curl -X POST https://open-wa.whizzlyplus.com/api/sessions/bot-ventas/start \
  -H "X-API-Key: TU_API_MASTER_KEY"

# Obtener QR
curl https://open-wa.whizzlyplus.com/api/sessions/bot-ventas/qr \
  -H "X-API-Key: TU_API_MASTER_KEY"
```

### 13.3 Enviar primer mensaje de prueba

Después de escanear el QR:

```bash
curl -X POST https://open-wa.whizzlyplus.com/api/sessions/bot-ventas/messages/send-text \
  -H "Content-Type: application/json" \
  -H "X-API-Key: TU_API_MASTER_KEY" \
  -d '{
    "chatId": "521XXXXXXXXXX@c.us",
    "text": "¡Hola desde OpenWA!"
  }'
```

> El `chatId` formato es: `[código país sin +][número]@c.us`. Ej México 521555..., España 34655..., Colombia 573...

---

## 14. Mantenimiento

### Ver logs

```bash
cd /opt/OpenWA

# Todos los servicios, en vivo
docker compose -f docker-compose.yml -f docker-compose.prod.yml logs -f

# Solo la API
docker compose -f docker-compose.yml -f docker-compose.prod.yml logs -f openwa-api

# Últimas 100 líneas
docker compose -f docker-compose.yml -f docker-compose.prod.yml logs --tail=100 openwa-api
```

### Reiniciar

```bash
cd /opt/OpenWA
docker compose -f docker-compose.yml -f docker-compose.prod.yml restart
```

### Detener todo

```bash
cd /opt/OpenWA
docker compose -f docker-compose.yml -f docker-compose.prod.yml down
```

### Levantar otra vez

```bash
cd /opt/OpenWA
docker compose -f docker-compose.yml -f docker-compose.prod.yml --profile full up -d
```

### Actualizar a la última versión

```bash
cd /opt/OpenWA

# Bajar última versión del código
git pull

# Reconstruir y relanzar
docker compose -f docker-compose.yml -f docker-compose.prod.yml --profile full up -d --build

# Limpiar imágenes viejas (libera disco)
docker image prune -f
```

### Backup de las sesiones de WhatsApp

Las sesiones viven dentro del volumen Docker `openwa-data`. Para backup:

```bash
# Crear directorio para backups
sudo mkdir -p /backup
sudo chown $USER:$USER /backup

# Backup completo (data + db)
docker run --rm \
  -v openwa-data:/data \
  -v /backup:/backup \
  alpine tar czf /backup/openwa-$(date +%Y%m%d-%H%M).tar.gz -C /data .

ls -lah /backup
```

Programa esto en cron para backups diarios:
```bash
crontab -e
# Añade al final:
0 3 * * * docker run --rm -v openwa-data:/data -v /backup:/backup alpine tar czf /backup/openwa-$(date +\%Y\%m\%d).tar.gz -C /data .
```

### Monitorear recursos

```bash
# Uso de CPU y RAM por contenedor
docker stats

# Uso de disco
df -h
docker system df
```

---

## 15. Troubleshooting

### El navegador da "ERR_CERT_AUTHORITY_INVALID" o cert inválido

**Causa**: Let's Encrypt aún no emitió el certificado o falló.

**Fix**:
```bash
# Ver logs de Traefik
docker compose -f docker-compose.yml -f docker-compose.prod.yml logs traefik | grep -i acme

# Si hay error "unable to obtain certificate":
# 1. Verificar que DNS apunta al server:
dig +short open-wa.whizzlyplus.com

# 2. Verificar que puerto 80 está accesible desde fuera:
curl -v http://open-wa.whizzlyplus.com/

# 3. Si DNS y puerto OK, forzar reintento:
docker compose -f docker-compose.yml -f docker-compose.prod.yml restart traefik
```

### Dashboard carga pero API da 404

**Causa**: el routing de Traefik está mal o el orden de prioridad.

**Fix**: revisar `traefik/dynamic.prod.yml` — `/api/` debe tener priority alta (100) y `/` priority baja (1).

### `docker compose up` falla con "no matching manifest for linux/arm64"

**Causa**: una imagen no tiene build ARM.

**Fix**: el `Dockerfile` y todas las imágenes usadas (postgres, redis, traefik, alpine) tienen variantes ARM oficiales. Si ves esto, verifica que estás en una shape Ampere y que apuntas a las imágenes correctas.

### Puerto 443 no responde desde fuera (timeout)

**Causa**: o no abriste la Security List, o no abriste iptables, o ambos.

**Fix**:
```bash
# Verificar iptables
sudo iptables -L INPUT -n | grep -E '80|443'

# Si no aparecen, repetir paso 6

# Verificar que algo escucha en 443
sudo ss -tlnp | grep 443
```

Si nada escucha en 443, revisa `docker compose ps` — Traefik debe estar Up.

### Chromium se cierra solo / "Browser disconnected"

**Causa**: memoria insuficiente. Cada sesión usa ~500 MB-1 GB.

**Fix**:
```bash
# Ver uso de RAM
free -h
docker stats --no-stream

# Si vas justo de RAM, reduce sesiones simultáneas o sube a 24 GB (free tier permite)
```

### WhatsApp dice "Phone not connected" después de escanear

**Causa**: tu teléfono o el server perdieron conectividad temporalmente.

**Fix**:
```bash
# Reiniciar la sesión
curl -X POST https://open-wa.whizzlyplus.com/api/sessions/SESSION_NAME/stop \
  -H "X-API-Key: TU_API_MASTER_KEY"

curl -X POST https://open-wa.whizzlyplus.com/api/sessions/SESSION_NAME/start \
  -H "X-API-Key: TU_API_MASTER_KEY"
```

Si persiste, el teléfono debe estar online y la app de WhatsApp abierta al menos cada ~14 días.

### Quiero deshacer todo y empezar de nuevo

```bash
cd /opt/OpenWA

# Detener y borrar contenedores + volúmenes (¡borra todas las sesiones!)
docker compose -f docker-compose.yml -f docker-compose.prod.yml down -v

# Borrar imágenes
docker compose -f docker-compose.yml -f docker-compose.prod.yml down --rmi all

# Empezar limpio
docker compose -f docker-compose.yml -f docker-compose.prod.yml --profile full up -d --build
```

---

## Resumen ultra-corto (para cuando ya hayas hecho esto 1 vez)

```bash
# 1. Crear VPS Ampere A1 4 vCPU/24 GB en Oracle, Ubuntu 22.04 ARM
# 2. Abrir Security List 80 + 443
# 3. SSH

sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save

# 4. Instalar Docker (script de Docker)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER && exit  # re-login

# 5. DNS A record → IP del VPS

# 6. Clonar y configurar
cd /opt
sudo git clone https://github.com/rmyndharis/OpenWA.git
sudo chown -R $USER:$USER OpenWA
cd OpenWA
cp docs/deploy/.env.example .env   # o copia el .env de esta guía
nano .env   # editar DOMAIN, TRAEFIK_ACME_EMAIL, API_MASTER_KEY

# 7. Levantar
docker compose -f docker-compose.yml -f docker-compose.prod.yml --profile full up -d --build

# 8. Probar
curl -sI https://TU_DOMINIO/api/health
```

---

## Costes esperados

| Concepto | Coste mensual |
|---|---|
| VPS Oracle Ampere 4 vCPU + 24 GB RAM | **$0** (Always Free Tier) |
| Tráfico saliente (hasta 10 TB/mes) | **$0** |
| Disco 50 GB block storage | **$0** (hasta 200 GB free) |
| IP pública IPv4 | **$0** |
| SSL Let's Encrypt | **$0** |
| Dominio (whizzlyplus.com) | Lo que ya pagas |
| **Total** | **$0/mes** |

Comparado con shared cPanel + IONOS VPS + Hetzner — Oracle gana en relación calidad/precio si estás dispuesto a invertir 1 hora en el setup inicial.
