OpenWA — Guía de despliegue para administrador de servidor
1. Requisitos del servidor
Hardware mínimo
Recurso	Mínimo	Recomendado	Para múltiples sesiones
CPU	1 vCPU	2 vCPU	1 vCPU por sesión
RAM	2 GB	4 GB	~1.5 GB por sesión activa
Disco	20 GB SSD	40 GB SSD	+3 GB por sesión
Red	IPv4 pública	IPv4 pública	—
Sistema operativo
Cualquiera de estos funciona:

✅ Ubuntu 22.04 / 24.04 LTS (recomendado — más simple)
✅ Debian 11 / 12
✅ AlmaLinux / Rocky / RHEL 8/9
✅ CentOS Stream 9
Acceso necesario
SSH como root (o sudo)
Capacidad de abrir puertos 80/443 en firewall
Resolución DNS funcionando
Red — IPs
⚠️ Crítico: WhatsApp Web a veces banea rangos de IP de datacenters conocidos por spam.

✅ Buenos: Hetzner, OVH, AWS Lightsail, IPs residenciales
⚠️ Cuidado: DigitalOcean, Linode, Vultr (a veces bloqueados)
❌ Evitar: rangos VPN públicos
2. Camino A — Despliegue con Docker (RECOMENDADO)
Es como el proyecto está pensado. 10 minutos de inicio a fin.

A.1 — Instalar Docker
Ubuntu/Debian:


apt update
apt install -y ca-certificates curl gnupg git
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
docker --version  # verificar
AlmaLinux/RHEL:


dnf install -y dnf-plugins-core git
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
A.2 — Clonar OpenWA y configurar

cd /opt
git clone https://github.com/rmyndharis/OpenWA.git
cd OpenWA
cp .env.example .env
nano .env
Configuración mínima del .env:


NODE_ENV=production
API_PORT=2785
DOMAIN=whatsapp.tudominio.com
DASHBOARD_ENABLED=true
PROXY_ENABLED=true                 # usa Traefik auto-SSL incluido en compose
TRAEFIK_ACME_EMAIL=tu@email.com    # para Let's Encrypt
DATABASE_TYPE=sqlite
REDIS_ENABLED=false
API_MASTER_KEY=GENERA_UNA_CLAVE_LARGA   # usar: openssl rand -hex 32
A.3 — Apuntar el DNS
En el panel del dominio:

Tipo: A
Nombre: whatsapp
Valor: la IP pública del servidor
TTL: 300
Esperar 1-15 min para que propague.

A.4 — Levantar todo

docker compose --profile full up -d
docker compose logs -f
Listo. En ~2 minutos están los contenedores arriba:

API: https://whatsapp.tudominio.com/api/docs (Swagger)
Dashboard: https://whatsapp.tudominio.com/
SSL auto-generado por Traefik
3. Camino B — Despliegue manual con npm (sin Docker)
Para hosts que NO permiten Docker (cPanel compartido, restricted shared, etc.).

B.1 — Instalar dependencias del sistema
Ubuntu/Debian:


apt update
apt install -y curl git build-essential python3 \
  chromium-browser \
  libnss3 libatk1.0-0 libatk-bridge2.0-0 libcups2 \
  libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 \
  libxrandr2 libgbm1 libpango-1.0-0 libasound2 libxss1 \
  fonts-liberation libappindicator3-1

# Node.js 22 LTS
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
AlmaLinux/RHEL:


dnf install -y curl git gcc-c++ make python3 \
  chromium \
  nss atk at-spi2-atk at-spi2-core cups-libs \
  libdrm libxkbcommon libXcomposite libXdamage \
  libXrandr libgbm pango alsa-lib libXScrnSaver \
  liberation-fonts

# Node.js 22 LTS
curl -fsSL https://rpm.nodesource.com/setup_22.x | bash -
dnf install -y nodejs
B.2 — Crear usuario dedicado (buenas prácticas)

useradd -m -s /bin/bash openwa
su - openwa
B.3 — Clonar y compilar

cd ~
git clone https://github.com/rmyndharis/OpenWA.git
cd OpenWA
cp .env.example .env
nano .env
Configuración mínima .env:


NODE_ENV=production
PORT=2785
DOMAIN=whatsapp.tudominio.com
DASHBOARD_ENABLED=false              # se sirve por nginx
PROXY_ENABLED=false                  # nginx fuera del proceso
DATABASE_TYPE=sqlite
REDIS_ENABLED=false
PUPPETEER_HEADLESS=true
PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium       # o /usr/bin/chromium-browser
PUPPETEER_ARGS=--no-sandbox,--disable-setuid-sandbox,--disable-dev-shm-usage,--disable-gpu
API_MASTER_KEY=GENERA_UNA_CLAVE_LARGA
ENABLE_SWAGGER=true
Instalar y compilar:


npm install --include=dev    # incluye nest CLI para build
npm run build
B.4 — Crear servicio systemd (como root)

exit   # salir del usuario openwa
nano /etc/systemd/system/openwa.service
Contenido:


[Unit]
Description=OpenWA WhatsApp API Gateway
After=network.target

[Service]
Type=simple
User=openwa
WorkingDirectory=/home/openwa/OpenWA
EnvironmentFile=/home/openwa/OpenWA/.env
ExecStart=/usr/bin/node dist/main.js
Restart=always
RestartSec=10
StandardOutput=append:/var/log/openwa/app.log
StandardError=append:/var/log/openwa/error.log

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=read-only
ReadWritePaths=/home/openwa/OpenWA

[Install]
WantedBy=multi-user.target
Activar:


mkdir -p /var/log/openwa
chown openwa:openwa /var/log/openwa
systemctl daemon-reload
systemctl enable --now openwa
systemctl status openwa     # ver si arrancó
curl http://localhost:2785/api/health   # debe dar JSON
B.5 — Nginx como reverse proxy + SSL

apt install -y nginx certbot python3-certbot-nginx   # Ubuntu/Debian
# o: dnf install -y nginx certbot python3-certbot-nginx  # RHEL family

nano /etc/nginx/sites-available/openwa
Contenido:


server {
    listen 80;
    server_name whatsapp.tudominio.com;

    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:2785;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
Activar y SSL:


ln -s /etc/nginx/sites-available/openwa /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx

# SSL automático (después de que el DNS apunte aquí)
certbot --nginx -d whatsapp.tudominio.com --redirect --agree-tos -m tu@email.com
Listo. https://whatsapp.tudominio.com/api/docs debe cargar Swagger.

4. Verificación final

# Estado del servicio
systemctl status openwa
journalctl -u openwa -n 50

# API responde
curl https://whatsapp.tudominio.com/api/health

# Probar autenticación
curl -X POST https://whatsapp.tudominio.com/api/sessions \
  -H "Content-Type: application/json" \
  -H "X-API-Key: TU_API_MASTER_KEY" \
  -d '{"name": "test"}'
5. Troubleshooting
Síntoma	Causa probable	Fix
Failed to launch browser	Faltan libs de Chromium	Reinstalar paquetes del paso B.1
EACCES en /var/log	Permisos	chown -R openwa:openwa /var/log/openwa
502 Bad Gateway en nginx	Node no responde	systemctl restart openwa && curl localhost:2785/api/health
Chrome consume mucha RAM	Normal con Chromium	Cada sesión usa ~500-800 MB. Asegurar RAM suficiente.
WhatsApp no acepta QR	IP del server baneada	Probar con otra IP / mover a otro datacenter
6. Mantenimiento

# Logs
journalctl -u openwa -f
tail -f /var/log/openwa/app.log

# Reiniciar
systemctl restart openwa

# Actualizar OpenWA
cd /home/openwa/OpenWA
sudo -u openwa git pull
sudo -u openwa npm install --include=dev
sudo -u openwa npm run build
systemctl restart openwa
Resumen para tu colega: si admin un servidor Linux propio (no shared cPanel), lo más rápido es Camino A con Docker — 10 minutos y está. Si por alguna razón no quiere/puede Docker, Camino B con systemd + nginx es estándar y robusto, ~30 min.

Cualquiera de los dos funciona infinitamente mejor que el shared cPanel donde nos estancamos por la integración rota de LiteSpeed↔Node Selector.