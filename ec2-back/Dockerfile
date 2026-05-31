# dcokerfile para prod

# instaladno dependencias

FROM node:20-alpine AS deps
WORKDIR /app

# Ferramentas nativas exigidas por alguns pacotes (bcrypt, etc.)
RUN apk add --no-cache python3 make g++

COPY package*.json ./
RUN npm ci


# build
FROM node:20-alpine AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Gera o Prisma Client (output: ../generated/prisma)
RUN npx prisma generate

# Compila o TypeScript → dist/
RUN npm run build

# runtime
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

# Copia apenas o necessário para rodar
COPY --from=deps    /app/node_modules       ./node_modules
COPY --from=builder /app/dist               ./dist
COPY --from=builder /app/generated          ./generated
COPY --from=builder /app/prisma             ./prisma
COPY --from=builder /app/package.json       ./package.json

EXPOSE 3333

# Roda as migrations pendentes e sobe o servidor
CMD ["sh", "-c", "npx prisma migrate deploy && node dist/main"]