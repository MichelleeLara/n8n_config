# AWS Setup for Docker + Nginx + Certbot (n8n)
# ------------------------------------------------------------
# 1) Ejecuta ESTE comando en tu máquina local para entrar al servidor:
ssh -i YOUR_KEY.pem ec2-user@YOUR_PUBLIC_IP

# ------------------------------------------------------------
# 2) Ya dentro de la EC2, pega TODO lo que sigue (de aquí hacia abajo).
#    Puedes copiar desde esta línea y pegarlo tal cual en la terminal remota.

# ----- Variables -----
DOMAIN="your-domain-name"

# ----- Actualizar e instalar Docker -----
sudo yum update -y
sudo yum install -y docker

# ----- Iniciar y habilitar Docker -----
sudo systemctl start docker
sudo systemctl enable docker

# (Opcional) Agregar ec2-user al grupo docker (seguiremos usando 'sudo', no necesitas reconectar)
sudo usermod -aG docker ec2-user

# ----- Instalar Docker Compose -----
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose version || /usr/local/bin/docker-compose version || true

# ----- Levantar n8n en Docker -----
sudo docker run -d --restart unless-stopped -it \
  --name n8n \
  -p 5678:5678 \
  -e N8N_HOST="$DOMAIN" \
  -e WEBHOOK_TUNNEL_URL="https://$DOMAIN/" \
  -e WEBHOOK_URL="https://$DOMAIN/" \
  -v ~/.n8n:/root/.n8n \
  n8nio/n8n

# ----- Instalar y habilitar Nginx -----
sudo dnf install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# ----- Configurar Nginx como reverse proxy para n8n -----
sudo tee /etc/nginx/conf.d/n8n.conf > /dev/null <<EOF
server {
    listen 80;
    server_name $DOMAIN;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;

        # WebSocket
        proxy_set_header Connection "Upgrade";
        proxy_set_header Upgrade \$http_upgrade;

        # Encabezados adicionales
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

# ----- Probar y recargar Nginx -----
sudo nginx -t
sudo systemctl restart nginx

# ----- Instalar Certbot e instalar certificado SSL -----
sudo dnf install -y certbot python3-certbot-nginx
sudo certbot --nginx -d "$DOMAIN"

# ----- Recargar Nginx tras Certbot -----
sudo systemctl restart nginx

# ----- Info útil -----
# Logs de n8n:  sudo docker logs -f n8n
# Reiniciar n8n: sudo docker restart n8n
# Estado Nginx:   systemctl status nginx
