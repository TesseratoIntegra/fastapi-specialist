---
name: devops
description: Configura infraestrutura completa de DevOps — Dockerfile multi-stage otimizado, docker-compose dev/prod, GitHub Actions CI/CD e checklist de segurança para produção. Use quando o usuário precisar preparar o projeto para produção ou melhorar a pipeline de deploy.
---

O usuário quer configurar DevOps para: $ARGUMENTS

Identifique o que está sendo pedido e gere os artefatos correspondentes. Se não for especificado, gere tudo.

---

## 1. Dockerfile multi-stage (dev + prod)

```dockerfile
# backend/Dockerfile

# ---- Stage 1: builder ----
FROM python:3.12-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

ENV POETRY_VERSION=1.8.5 \
    POETRY_NO_INTERACTION=1 \
    POETRY_VIRTUALENVS_CREATE=false \
    POETRY_VENV_IN_PROJECT=0

RUN pip install --no-cache-dir "poetry==$POETRY_VERSION"

WORKDIR /app
COPY pyproject.toml poetry.lock* ./
RUN poetry install --no-root --only=main

# ---- Stage 2: runtime ----
FROM python:3.12-slim AS runtime

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && addgroup --system app \
    && adduser --system --ingroup app app

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY --chown=app:app . .

RUN chmod +x entrypoint.sh

USER app

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

ENTRYPOINT ["./entrypoint.sh"]
```

**Por que multi-stage?**
- Stage `builder`: instala compiladores e dependências de build
- Stage `runtime`: imagem final limpa sem gcc/compiladores (~60% menor)
- Roda como usuário não-root (`app`) — melhor postura de segurança

---

## 2. docker-compose.yml (desenvolvimento)

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: builder
    restart: unless-stopped
    ports:
      - "8000:8000"
    env_file:
      - .env
    environment:
      - DEBUG=true
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./backend:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

volumes:
  postgres_data:
  redis_data:
```

---

## 3. docker-compose.prod.yml (produção)

```yaml
# docker-compose.prod.yml
services:
  db:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backups:/backups
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal

  redis:
    image: redis:7-alpine
    restart: always
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --save 60 1
      --loglevel warning
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal

  backend:
    image: ghcr.io/${GITHUB_REPOSITORY}/backend:${IMAGE_TAG:-latest}
    restart: always
    env_file:
      - .env.prod
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "8000:8000"
    networks:
      - internal
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          memory: 256M

volumes:
  postgres_data:
  redis_data:

networks:
  internal:
    driver: bridge
```

---

## 4. GitHub Actions CI/CD

### .github/workflows/ci.yml (testes + lint)
```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: app_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install Poetry
        run: pip install poetry==1.8.5

      - name: Cache dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pypoetry
          key: ${{ runner.os }}-poetry-${{ hashFiles('backend/poetry.lock') }}

      - name: Install dependencies
        working-directory: backend
        run: poetry install --no-root

      - name: Lint
        working-directory: backend
        run: poetry run task lint

      - name: Test
        working-directory: backend
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/app_test
          REDIS_URL: redis://localhost:6379/0
          SECRET_KEY: test-secret-key-for-ci-only
        run: poetry run task test-cov

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: backend/coverage.xml
        # Gerar coverage.xml: adicione --cov-report=xml ao test-cov em pyproject.toml
```

### .github/workflows/deploy.yml (build + push + deploy)
```yaml
name: Deploy

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/backend

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_KEY }}
          script: |
            set -e
            cd /opt/app

            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

            export IMAGE_TAG=sha-${{ github.sha }}
            export PREVIOUS_TAG=$(docker inspect --format='{{index .RepoTags 0}}' $(docker compose -f docker-compose.prod.yml ps -q backend) 2>/dev/null || echo "")

            # Pull nova imagem
            docker compose -f docker-compose.prod.yml pull backend

            # Deploy
            docker compose -f docker-compose.prod.yml up -d --no-deps backend

            # Health check pós-deploy (aguarda até 60s)
            for i in $(seq 1 12); do
              if curl -sf http://localhost:8000/health > /dev/null; then
                echo "Deploy OK"
                break
              fi
              echo "Aguardando app iniciar... ($i/12)"
              sleep 5
              if [ $i -eq 12 ]; then
                echo "FALHA: app não respondeu após deploy. Iniciando rollback..."
                if [ -n "$PREVIOUS_TAG" ]; then
                  docker compose -f docker-compose.prod.yml up -d --no-deps backend
                fi
                exit 1
              fi
            done

            docker image prune -f
```

---

## 5. Checklist de produção

Execute e responda cada item antes de fazer deploy:

### Segurança
- [ ] `SECRET_KEY` com 32+ caracteres aleatórios (não o valor padrão)
- [ ] `DEBUG=false`
- [ ] Senha do Redis configurada (`REDIS_PASSWORD`)
- [ ] Senha do PostgreSQL forte (não `postgres`)
- [ ] Variáveis de ambiente em `.env.prod` — NUNCA commitado no git
- [ ] `.env*` no `.gitignore`
- [ ] `CORS_ORIGINS_RAW` restrito ao domínio real

### Docker
- [ ] Imagem usando stage `runtime` (não `builder`)
- [ ] Rodando como usuário não-root
- [ ] `HEALTHCHECK` configurado no Dockerfile
- [ ] `restart: always` em todos os serviços prod
- [ ] Rede `internal` para serviços sem exposição externa

### Banco de dados
- [ ] Migrations aplicadas antes do deploy (`alembic upgrade head`)
- [ ] Backup automatizado configurado (pg_dump agendado)
- [ ] Pool de conexões dimensionado corretamente (`pool_size`, `max_overflow`)
- [ ] Índices criados para queries frequentes

### Aplicação
- [ ] `uvicorn` sem `--reload` em produção
- [ ] Health check endpoint respondendo (`GET /health`)
- [ ] Rate limiting ativo
- [ ] Sem secrets hardcoded no código

### CI/CD
- [ ] Testes passando no CI antes do merge
- [ ] Coverage >= 80%
- [ ] Lint passando (sem erros Ruff)
- [ ] Segredos no GitHub Secrets (não no código)
- [ ] Deploy automático apenas na branch `main`

---

## Passos a executar

Baseado no que foi pedido em `$ARGUMENTS`:

1. Se for **dockerfile**: crie/atualize `backend/Dockerfile` com multi-stage
2. Se for **docker-compose**: crie/atualize `docker-compose.yml` e `docker-compose.prod.yml`
3. Se for **ci/cd** ou **github actions**: crie `.github/workflows/ci.yml` e `.github/workflows/deploy.yml`
4. Se for **checklist** ou **produção**: execute e apresente o checklist respondido com base nos arquivos existentes
5. Se não especificado: gere tudo acima

Sempre leia os arquivos existentes antes de sobrescrever — preserve configurações customizadas.
