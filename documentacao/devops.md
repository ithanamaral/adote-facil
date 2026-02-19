# Melhorias de DevOps - Adote Fácil

Este documento detalha as otimizações implementadas no pipeline de CI/CD e na estrutura de containers do projeto, visando performance, segurança e manutenibilidade.

## 1. Otimização do Dockerfile (Backend)

**Melhoria:** Implementação de *Multi-stage Build*.
**Objetivo:** Reduzir o tamanho da imagem final e remover código fonte e dependências de desenvolvimento do ambiente de produção.

### Implementação (`backend/Dockerfile`)

```dockerfile
# Estágio 1: Build
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./

# Instala todas as dependências (incluindo devDependencies) com npm ci para consistência
RUN npm ci

COPY . .

# Gera o cliente Prisma e compila o projeto
RUN npm run generate
RUN npm run build

# Estágio 2: Produção
FROM node:20-alpine AS runner

RUN apk add --no-cache openssl

WORKDIR /app

# Copia apenas os artefatos necessários do estágio de build
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/prisma ./prisma

# Instala apenas dependências de produção
RUN npm ci --only=production

# Gera o cliente Prisma para o ambiente de produção
RUN npm run generate

EXPOSE 8080

CMD ["npm", "start"]
```

## 2. Melhorias no Pipeline CI/CD (GitHub Actions)

**Melhoria:** Uso de `npm ci` e Cache.
**Objetivo:** Acelerar o tempo de execução do pipeline e garantir instalações determinísticas.

*   **`npm ci`**: Ao contrário do `npm install`, o `npm ci` deleta a pasta `node_modules` existente e instala as dependências exatamente como especificadas no `package-lock.json`. É mais rápido e seguro para CI.
*   **Cache**: O cache do npm evita baixar as mesmas bibliotecas repetidamente entre as execuções.

### Trecho do Workflow (`.github/workflows/experimento-ci-cd.yml`)

```yaml
- name: Configurar Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
    cache-dependency-path: backend/package-lock.json

- name: Instalar dependências do backend
  run: |
    cd backend
    npm ci  # Instalação limpa e rápida
```

## 3. Arquivo .dockerignore

**Melhoria:** Configuração correta do `.dockerignore`.
**Objetivo:** Evitar que arquivos locais que são desnecessários e grandes (como `node_modules`, `.git`, `.env`) sejam copiados para dentro da imagem Docker, o que deixaria o build lento e a imagem insegura.


