# Estágio 1: Build
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./

# Instala todas as dependências (incluindo devDependencies)
RUN npm ci

COPY . .

# Gera o cliente Prisma e compila o projeto
RUN npm run generate
RUN npm run build

# Estágio 2: Produção
FROM node:20-alpine AS runner

RUN apk add --no-cache openssl

WORKDIR /app

# Copia apenas os arquivos necessários do estágio de build
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/prisma ./prisma

# Instala apenas dependências de produção
RUN npm ci --only=production

# Gera o cliente Prisma para o ambiente de produção
RUN npm run generate

EXPOSE 8080

CMD ["npm", "start"]
