# ec2-front

Instância responsável pelo frontend web (Next.js) e mobile web (Expo), com proxy reverso Nginx e HTTPS via Let's Encrypt.

## Estrutura

```
ec2-front/
├── docker-compose.yml
├── .env.example
├── nginx/
│   └── nginx.conf               # proxy reverso (HTTP — Certbot adiciona HTTPS depois)
├── pro4tech-frontend/
│   └── Dockerfile
└── pro4tech-mobile/
    ├── Dockerfile
    └── nginx.conf               # nginx interno do container mobile
```

Na EC2, os repos ficam assim:
```
~/ec2-front/
├── docker-compose.yml
├── .env
├── nginx/
├── pro4tech-frontend/           # git clone do repo
└── pro4tech-mobile/             # git clone do repo
```

---

## Pré-requisitos

Antes de começar, garanta que:

- [ ] EC2 Ubuntu 22.04 criada com Elastic IP associado
- [ ] Security Group com portas **22**, **80** e **443** abertas para `0.0.0.0/0`
- [ ] Domínios DDNS configurados no no-ip apontando para o Elastic IP:
  - `web.orbita4tech.hopto.org`
  - `orbita4tech.hopto.org`
- [ ] EC2 do backend já criada (você precisará do IP privado dela)

---

## Primeira execução

### 1. Instalar Docker na EC2

Conecte via SSH e rode:

```bash
sudo apt-get update -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
newgrp docker
```

Verifique:
```bash
docker --version
docker compose version
```

### 2. Clonar os repositórios

```bash
mkdir ~/ec2-front && cd ~/ec2-front

# Infra (docker-compose, Dockerfiles, nginx.conf)
git clone https://github.com/CodeDontBlow/pro4tech-infra.git tmp-infra
cp -r tmp-infra/ec2-front/. .
rm -rf tmp-infra

# Código-fonte
git clone https://github.com/CodeDontBlow/pro4tech-frontend.git
git clone https://github.com/CodeDontBlow/pro4tech-mobile.git
```

A estrutura final deve ser a mostrada acima.

### 3. Copiar os Dockerfiles para dentro dos repos

```bash
cp pro4tech-frontend/Dockerfile pro4tech-frontend/Dockerfile   # já está no lugar certo
cp pro4tech-mobile/Dockerfile   pro4tech-mobile/Dockerfile     # idem
cp pro4tech-mobile/nginx.conf   pro4tech-mobile/nginx.conf     # idem
```

> Os Dockerfiles já estão clonados dentro de cada pasta de repo. Nenhuma ação adicional necessária.

### 4. Configurar variáveis de ambiente

```bash
cp .env.example .env
nano .env
```

Preencha:

```bash
NEXT_PUBLIC_API_URL=https://web.orbita4tech.hopto.org/api
EXPO_PUBLIC_API_URL_WEB=https://web.orbita4tech.hopto.org/api
EXPO_PUBLIC_API_URL_ANDROID=http://<IP-PRIVADO-DA-EC2-BACK>:3333
CERTBOT_EMAIL=seu-email@exemplo.com
```

> O IP privado da ec2-back aparece no painel da AWS em **EC2 → Instances → Private IPv4 address**.

### 5. Subir os containers (HTTP primeiro)

```bash
cd ~/ec2-front
docker compose up -d frontend mobile nginx-proxy
```

Aguarde o build — na primeira vez leva alguns minutos (Next.js e Expo compilam dentro do Docker).

Verifique se tudo subiu:
```bash
docker compose ps
```

Todos devem estar com status `Up`. Teste no browser:
- `http://web.orbita4tech.hopto.org` → deve abrir o Next.js
- `http://orbita4tech.hopto.org` → deve abrir o Expo web

### 6. Emitir certificados TLS (Let's Encrypt)

Com o Nginx respondendo na porta 80, rode o Certbot:

```bash
docker compose run --rm certbot
```

O Certbot vai:
1. Fazer um desafio HTTP no `/.well-known/acme-challenge/` nos dois domínios
2. Confirmar que os domínios apontam para este servidor
3. Salvar os certificados no volume `certbot-certs`

Saída esperada ao final:
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/web.orbita4tech.hopto.org/fullchain.pem
```

Se falhar com `Connection refused` ou `Timeout`, o DDNS ainda não propagou. Aguarde alguns minutos e tente novamente.

### 7. Atualizar o Nginx para HTTPS

Abra o arquivo do proxy:

```bash
nano ~/ec2-front/nginx/nginx.conf
```

Substitua o conteúdo pelo seguinte (adiciona blocos SSL e redireciona HTTP → HTTPS):

```nginx
# ── Redireciona todo HTTP para HTTPS ────────────────────────
server {
    listen 80;
    server_name web.orbita4tech.hopto.org orbita4tech.hopto.org;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# ── Frontend Web (Next.js) — HTTPS ──────────────────────────
server {
    listen 443 ssl;
    server_name web.orbita4tech.hopto.org;

    ssl_certificate     /etc/letsencrypt/live/web.orbita4tech.hopto.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/web.orbita4tech.hopto.org/privkey.pem;

    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;

    location / {
        proxy_pass         http://frontend:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade    $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_cache_bypass $http_upgrade;
    }
}

# ── Mobile / Expo Web — HTTPS ────────────────────────────────
server {
    listen 443 ssl;
    server_name orbita4tech.hopto.org;

    ssl_certificate     /etc/letsencrypt/live/web.orbita4tech.hopto.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/web.orbita4tech.hopto.org/privkey.pem;

    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;

    location / {
        proxy_pass         http://mobile:80;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade    $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_cache_bypass $http_upgrade;
    }
}
```

Salve e recarregue o Nginx:

```bash
docker compose exec nginx-proxy nginx -s reload
```

Teste no browser:
- `https://web.orbita4tech.hopto.org` → cadeado verde, Next.js
- `https://orbita4tech.hopto.org` → cadeado verde, Expo web
- `http://web.orbita4tech.hopto.org` → deve redirecionar para HTTPS

---

## Após reinicialização da sessão AWS Academy

A cada nova sessão do AWS Academy as credenciais mudam, mas a instância EC2 continua rodando. Os containers sobem automaticamente com `restart: unless-stopped`.

Verifique o status ao reconectar:

```bash
docker compose -f ~/ec2-front/docker-compose.yml ps
```

Se algum container estiver parado:

```bash
cd ~/ec2-front
docker compose up -d
```

---

## Atualizar o código (redeploy)

Quando houver mudanças nos repos:

```bash
cd ~/ec2-front

# Puxa as atualizações
git -C pro4tech-frontend pull
git -C pro4tech-mobile pull

# Rebuilda apenas os containers alterados e reinicia
docker compose build frontend mobile
docker compose up -d
```

---

## Renovação dos certificados

Os certificados do Let's Encrypt expiram em **90 dias**. Configure renovação automática:

```bash
crontab -e
```

Adicione a linha:

```
0 3 * * * cd /home/ubuntu/ec2-front && docker compose run --rm certbot renew --quiet && docker compose exec nginx-proxy nginx -s reload
```

Para renovar manualmente:

```bash
cd ~/ec2-front
docker compose run --rm certbot renew
docker compose exec nginx-proxy nginx -s reload
```

---

## Comandos úteis

```bash
# Ver logs em tempo real de todos os containers
docker compose logs -f

# Ver logs de um container específico
docker compose logs -f frontend
docker compose logs -f mobile
docker compose logs -f nginx-proxy

# Reiniciar um container sem rebuild
docker compose restart nginx-proxy

# Parar tudo
docker compose down

# Parar e remover volumes (apaga certificados — cuidado)
docker compose down -v
```