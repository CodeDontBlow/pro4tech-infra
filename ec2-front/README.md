# ec2-front

Instância responsável pelo frontend web (Next.js) e mobile web (Expo), com proxy reverso Nginx e HTTPS via Let's Encrypt.

## Estrutura do repo `pro4tech-infra/ec2-front`

```
ec2-front/
├── userdata-front.sh            # script colado no User Data ao criar a EC2
├── docker-compose.yml
├── .env.example
├── nginx/
│   ├── nginx.conf               # proxy reverso (HTTP inicial)
│   └── nginx.https.conf         # proxy reverso (HTTPS com websocket)
├── pro4tech-frontend/
│   └── Dockerfile
└── pro4tech-mobile/
    ├── Dockerfile
    └── nginx.conf               # nginx interno do container mobile
```

Na EC2, após o User Data rodar, a estrutura fica assim:

```
/home/ubuntu/ec2-front/
├── docker-compose.yml
├── .env                         # criado automaticamente pelo User Data
├── nginx/
├── pro4tech-frontend/           # clonado automaticamente
└── pro4tech-mobile/             # clonado automaticamente
```

---

## Pré-requisitos

Antes de criar a EC2:

- [ ] Security Group com portas **22**, **80** e **443** abertas para `0.0.0.0/0`
- [ ] Elastic IP criado e pronto para associar
- [ ] Domínios DDNS configurados no no-ip apontando para o Elastic IP:
  - `web.orbita4tech.hopto.org`
  - `orbita4tech.hopto.org`
- [ ] IP privado da `ec2-back` em mãos (para preencher o `userdata-front.sh`)

---

## Criando a EC2

### 1. Editar o `userdata-front.sh`

Antes de criar a instância, abra o `userdata-front.sh` e preencha as duas variáveis no topo:

```bash
CERTBOT_EMAIL="seu-email@exemplo.com"   # ← seu e-mail real
BACKEND_PRIVATE_IP="10.0.0.X"           # ← IP privado da ec2-back
```

### 2. Criar a instância na AWS

No painel EC2 → Launch Instance:

- AMI: **Ubuntu 22.04 LTS**
- Instance type: `t2.micro` (ou maior se disponível)
- Security Group: portas 22, 80, 443 abertas
- Em **Advanced details → User data**: cole o conteúdo do `userdata-front.sh`

Associe o Elastic IP à instância após criá-la.

### 3. Acompanhar o progresso

O User Data roda na primeira inicialização e leva alguns minutos (Next.js e Expo compilam dentro do Docker). Para acompanhar:

```bash
ssh ubuntu@<ELASTIC-IP>
tail -f /var/log/userdata-front.log
```

Quando aparecer `✅ UserData-front concluído!`, está pronto.

Verifique os containers:

```bash
cd ~/ec2-front
docker compose ps
```

Todos devem estar com status `Up`.

---

## Após o User Data — configurar HTTPS

O User Data sobe tudo em HTTP. Os passos abaixo você faz **uma única vez**, após confirmar que os domínios DDNS já estão propagados.

### 1. Testar HTTP

No browser:
- `http://web.orbita4tech.hopto.org` → deve abrir o Next.js
- `http://orbita4tech.hopto.org` → deve abrir o Expo web

Se não abrir, o DDNS ainda não propagou. Aguarde alguns minutos e tente novamente.

### 2. Emitir os certificados TLS

```bash
cd ~/ec2-front
docker compose run --rm certbot
```

Saída esperada ao final:
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/web.orbita4tech.hopto.org/fullchain.pem
```

Se falhar com `Connection refused` ou `Timeout`, o DDNS ainda não propagou. Aguarde e tente novamente.

### 3. Atualizar o Nginx para HTTPS

Copie a configuração HTTPS para substituir a HTTP:

```bash
cp ~/ec2-front/nginx/nginx.https.conf ~/ec2-front/nginx/nginx.conf
```

Se preferir editar manualmente, o conteúdo de `nginx.https.conf` já é o bloco final.

### 4. Recarregar o Nginx

```bash
docker compose exec nginx-proxy nginx -s reload
```

Teste no browser:
- `https://web.orbita4tech.hopto.org` → cadeado verde, Next.js
- `https://orbita4tech.hopto.org` → cadeado verde, Expo web
- `http://web.orbita4tech.hopto.org` → deve redirecionar para HTTPS automaticamente

---

## Após reinicialização da sessão AWS Academy

A cada nova sessão do AWS Academy as credenciais mudam, mas a EC2 continua rodando e os containers sobem automaticamente com `restart: unless-stopped`.

Ao reconectar, apenas verifique:

```bash
cd ~/ec2-front
docker compose ps
```

Se algum container estiver parado:

```bash
docker compose up -d
```

---

## Atualizar o código (redeploy)

Quando houver mudanças nos repos:

```bash
cd ~/ec2-front
git -C pro4tech-frontend pull
git -C pro4tech-mobile pull
docker compose build frontend mobile
docker compose up -d
```

---

## Renovação dos certificados

Os certificados expiram em **90 dias**. Configure renovação automática:

```bash
crontab -e
```

Adicione:

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
# Logs em tempo real de todos os containers
docker compose logs -f

# Logs de um container específico
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