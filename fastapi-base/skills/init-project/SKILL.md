---
name: init-project
description: Inicializa um projeto FastAPI completo com SQLAlchemy async, PostgreSQL, Redis, Docker e estrutura de produção. Use quando o usuário quiser criar um novo projeto do zero.
---

O usuário quer inicializar um novo projeto FastAPI. O argumento fornecido é o nome do projeto: $ARGUMENTS

Crie a estrutura completa do projeto seguindo rigorosamente os padrões abaixo.

## Estrutura a criar

```
{nome-do-projeto}/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── core/
│   │   │   ├── __init__.py
│   │   │   ├── config.py
│   │   │   ├── database.py
│   │   │   ├── security.py
│   │   │   └── exceptions.py
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   └── base.py
│   │   ├── schemas/
│   │   │   ├── __init__.py
│   │   │   └── common.py
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── deps.py
│   │   │   └── v1/
│   │   │       ├── __init__.py
│   │   │       ├── router.py
│   │   │       └── health.py
│   │   ├── services/
│   │   │   └── __init__.py
│   │   └── utils/
│   │       └── __init__.py
│   ├── alembic/
│   │   ├── versions/
│   │   └── env.py
│   ├── tests/
│   │   ├── __init__.py
│   │   └── conftest.py
│   ├── alembic.ini
│   ├── Dockerfile
│   ├── entrypoint.sh
│   └── pyproject.toml
├── .env.example
├── .gitignore
├── docker-compose.yml
├── docker-compose.prod.yml
└── CLAUDE.md
```

## Regras obrigatórias de código

- Aspas simples (`'`) em todo Python
- Docstrings em português
- `comment` em todas as colunas e tabelas SQLAlchemy
- StrEnum (Python 3.12), nunca `(str, enum.Enum)`
- Imports sempre no top-level
- NUNCA blocos decorados (`# ===`, `# ---`)
- PEP8 rigoroso, line-length 79
- Services usam apenas `db.flush()` — NUNCA `db.commit()`
- Relacionamentos async: sempre `selectinload()`, nunca lazy load

## Conteúdo de cada arquivo

### backend/pyproject.toml
```toml
[tool.poetry]
name = "{nome-do-projeto}"
version = "0.1.0"
description = ""
authors = ["TesseratoIntegra"]
readme = "README.md"

[tool.poetry.dependencies]
python = "^3.12"
fastapi = "^0.115"
uvicorn = {extras = ["standard"], version = "^0.32"}
sqlalchemy = {extras = ["asyncio"], version = "^2.0"}
asyncpg = "^0.30"
alembic = "^1.14"
pydantic-settings = "^2.6"
python-jose = {extras = ["cryptography"], version = "^3.3"}
passlib = {extras = ["bcrypt"], version = "^1.7"}
redis = "^5.2"
apscheduler = "^3.10"
slowapi = "^0.1"
httpx = "^0.28"
python-multipart = "^0.0.19"
email-validator = "^2.2"

[tool.poetry.group.dev.dependencies]
pytest = "^8.3"
pytest-asyncio = "^0.24"
pytest-cov = "^6.0"
coverage = "^7.6"
taskipy = "^1.14"
ruff = "^0.8"

[tool.taskipy.tasks]
lint = "ruff check . && ruff check . --diff"
format = "ruff check . --fix && ruff format ."
test = "pytest -s -x -v"
test-cov = "pytest --cov=app --cov-report=term-missing --cov-report=html --cov-report=xml"
run = "uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"
migrate = "alembic upgrade head"
makemigrations = "alembic revision --autogenerate -m"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.coverage.run]
omit = ["tests/*", "alembic/*"]

[tool.coverage.report]
fail_under = 80

[tool.ruff]
target-version = "py312"
line-length = 79

[tool.ruff.lint]
select = ["E", "W", "F", "I", "N", "UP", "B", "SIM", "C4", "DTZ", "T20", "RET", "PTH", "ERA", "PL", "RUF"]
ignore = ["E501", "B008", "PLR0913", "RUF012"]

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["T20", "PLR2004"]
"alembic/*" = ["F401", "E402", "F811"]

[tool.ruff.format]
quote-style = "single"

[tool.ruff.lint.isort]
known-first-party = ["app"]

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### backend/Dockerfile
```dockerfile
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

### backend/.dockerignore
```
__pycache__/
*.py[cod]
*.egg-info/
.env
.venv/
venv/
htmlcov/
.coverage
.pytest_cache/
.ruff_cache/
tests/
alembic/versions/
*.log
.DS_Store
```

### backend/entrypoint.sh
```bash
#!/bin/bash
set -e

echo "Aguardando PostgreSQL..."
MAX_RETRIES=30
RETRY=0

until python -c "
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine
async def check():
    engine = create_async_engine('$DATABASE_URL')
    async with engine.connect() as conn:
        await conn.execute(__import__('sqlalchemy').text('SELECT 1'))
    await engine.dispose()
asyncio.run(check())
" 2>/dev/null; do
    RETRY=$((RETRY+1))
    if [ $RETRY -ge $MAX_RETRIES ]; then
        echo "PostgreSQL não respondeu após $MAX_RETRIES tentativas. Abortando."
        exit 1
    fi
    echo "Tentativa $RETRY/$MAX_RETRIES — aguardando 2s..."
    sleep 2
done

echo "PostgreSQL pronto. Executando migrations..."
alembic upgrade head

echo "Criando usuário admin padrão (se não existir)..."
python -c "
import asyncio
from sqlalchemy import select
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from app.core.config import settings
from app.core.security import hash_password

async def seed():
    from app.models.user import User, UserRole
    engine = create_async_engine(settings.database_url)
    SessionLocal = async_sessionmaker(engine, expire_on_commit=False)
    async with SessionLocal() as db:
        result = await db.execute(select(User).where(User.role == UserRole.ADMIN))
        if not result.scalar_one_or_none():
            admin = User(
                email=settings.admin_email,
                password_hash=hash_password(settings.admin_password),
                full_name='Administrador',
                role=UserRole.ADMIN,
                is_active=True,
            )
            db.add(admin)
            await db.commit()
            print(f'Admin criado: {settings.admin_email}')
        else:
            print('Admin já existe, pulando seed.')
    await engine.dispose()

asyncio.run(seed())
" || echo "Seed ignorado (model User ainda não existe)"

echo "Iniciando aplicação..."
exec uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### backend/alembic.ini
```ini
[alembic]
script_location = alembic
file_template = %%(year)d_%%(month).2d_%%(day).2d_%%(rev)s_%%(slug)s
timezone = America/Sao_Paulo
truncate_slug_length = 40
prepend_sys_path = .

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

### backend/alembic/env.py
```python
import asyncio
from logging.config import fileConfig

from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config

from alembic import context

from app.core.config import settings
from app.models.base import Base
from app.models import *  # noqa: F401, F403

config = context.config
config.set_main_option('sqlalchemy.url', settings.database_url)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    """Executa migrations em modo offline (gera SQL sem conectar ao banco)."""
    url = config.get_main_option('sqlalchemy.url')
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={'paramstyle': 'named'},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    """Executa as migrations usando a conexão fornecida."""
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    """Cria engine async e executa migrations."""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix='sqlalchemy.',
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    """Ponto de entrada para execução online das migrations."""
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### backend/app/__init__.py
```python
```

### backend/app/main.py
```python
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from slowapi import _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware

from app.core.config import settings
from app.core.database import engine
from app.core.limiter import limiter
from app.api.v1.router import router as v1_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Gerencia o ciclo de vida da aplicação."""
    yield
    await engine.dispose()


app = FastAPI(
    title=settings.project_name,
    version=settings.version,
    lifespan=lifespan,
)

app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
app.add_middleware(SlowAPIMiddleware)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=['*'],
    allow_headers=['*'],
)

app.include_router(v1_router, prefix='/api/v1')


@app.get('/health', tags=['health'])
async def health_check():
    """Verifica se a aplicação está no ar."""
    return {'status': 'ok', 'version': settings.version}
```

### backend/app/core/config.py
```python
import json
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Configurações da aplicação carregadas via variáveis de ambiente."""

    model_config = SettingsConfigDict(env_file='.env', extra='ignore')

    project_name: str = '{Nome do Projeto}'
    version: str = '0.1.0'
    debug: bool = False

    database_url: str
    redis_url: str = 'redis://redis:6379/0'

    secret_key: str
    jwt_algorithm: str = 'HS256'
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7

    cors_origins_raw: str = '["http://localhost:3000"]'

    # Seed do admin inicial
    admin_email: str = 'admin@example.com'
    admin_password: str = 'Admin@1234'

    @property
    def cors_origins(self) -> list[str]:
        """Retorna lista de origens permitidas para CORS."""
        try:
            return json.loads(self.cors_origins_raw)
        except (json.JSONDecodeError, TypeError):
            return [o.strip() for o in self.cors_origins_raw.split(',')]


settings = Settings()
```

### backend/app/core/database.py
```python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

from app.core.config import settings

engine = create_async_engine(
    settings.database_url,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    expire_on_commit=False,
)


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """Dependency que fornece sessão de banco com commit automático."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### backend/app/core/security.py
```python
from datetime import UTC, datetime, timedelta

from jose import JWTError, jwt
from passlib.context import CryptContext

from app.core.config import settings

pwd_context = CryptContext(schemes=['bcrypt'], deprecated='auto')


def hash_password(password: str) -> str:
    """Gera hash bcrypt da senha."""
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verifica se a senha corresponde ao hash."""
    return pwd_context.verify(plain_password, hashed_password)


def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    """Cria JWT de acesso com expiração configurável."""
    to_encode = data.copy()
    expire = datetime.now(UTC) + (
        expires_delta or timedelta(minutes=settings.access_token_expire_minutes)
    )
    to_encode.update({'exp': expire, 'type': 'access'})
    return jwt.encode(to_encode, settings.secret_key, algorithm=settings.jwt_algorithm)


def create_refresh_token(data: dict) -> str:
    """Cria JWT de refresh com expiração longa."""
    to_encode = data.copy()
    expire = datetime.now(UTC) + timedelta(days=settings.refresh_token_expire_days)
    to_encode.update({'exp': expire, 'type': 'refresh'})
    return jwt.encode(to_encode, settings.secret_key, algorithm=settings.jwt_algorithm)


def decode_token(token: str) -> dict | None:
    """Decodifica e valida JWT, retornando None se inválido."""
    try:
        return jwt.decode(token, settings.secret_key, algorithms=[settings.jwt_algorithm])
    except JWTError:
        return None
```

### backend/app/core/exceptions.py
```python
from fastapi import HTTPException, status


class CredentialsException(HTTPException):
    """Credenciais inválidas ou token expirado."""
    def __init__(self):
        super().__init__(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail='Credenciais inválidas',
            headers={'WWW-Authenticate': 'Bearer'},
        )


class ForbiddenException(HTTPException):
    """Acesso negado ao recurso."""
    def __init__(self, detail: str = 'Acesso negado'):
        super().__init__(status_code=status.HTTP_403_FORBIDDEN, detail=detail)


class NotFoundException(HTTPException):
    """Recurso não encontrado."""
    def __init__(self, detail: str = 'Recurso não encontrado'):
        super().__init__(status_code=status.HTTP_404_NOT_FOUND, detail=detail)


class ConflictException(HTTPException):
    """Conflito — recurso já existe."""
    def __init__(self, detail: str = 'Recurso já existe'):
        super().__init__(status_code=status.HTTP_409_CONFLICT, detail=detail)


class BadRequestException(HTTPException):
    """Requisição inválida."""
    def __init__(self, detail: str = 'Requisição inválida'):
        super().__init__(status_code=status.HTTP_400_BAD_REQUEST, detail=detail)
```

### backend/app/core/__init__.py, backend/app/schemas/__init__.py, backend/app/api/__init__.py, backend/app/api/v1/__init__.py, backend/app/services/__init__.py, backend/app/utils/__init__.py
Todos vazios.

### backend/app/models/__init__.py
```python
# Importar todos os models aqui para o Alembic detectar automaticamente
from app.models.base import Base  # noqa: F401
```
**IMPORTANTE**: sempre que um novo model for criado, ele deve ser importado neste arquivo — caso contrário o Alembic não gera a migration correta.

### backend/app/core/limiter.py
```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
```

### backend/app/models/base.py
```python
import uuid
from datetime import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    """Classe base para todos os models SQLAlchemy."""
    pass


class UUIDMixin:
    """Mixin que adiciona id UUID como chave primária."""
    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True,
        default=uuid.uuid4,
        comment='Identificador único UUID',
    )


class TimestampMixin:
    """Mixin que adiciona timestamps de criação e atualização."""
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        comment='Data de criação do registro',
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        comment='Data da última atualização',
    )
```

### backend/app/schemas/common.py
```python
import math
from typing import Generic, TypeVar

from pydantic import BaseModel

T = TypeVar('T')


class MessageResponse(BaseModel):
    """Resposta genérica com mensagem."""
    message: str


class PaginationParams(BaseModel):
    """Parâmetros de paginação."""
    page: int = 1
    per_page: int = 20

    @property
    def offset(self) -> int:
        """Calcula o offset para query SQL."""
        return (self.page - 1) * self.per_page


class PaginatedResponse(BaseModel, Generic[T]):
    """Resposta paginada genérica."""
    items: list[T]
    total: int
    page: int
    per_page: int
    pages: int

    @classmethod
    def build(cls, items: list[T], total: int, page: int, per_page: int):
        """Constrói resposta paginada calculando total de páginas."""
        return cls(
            items=items,
            total=total,
            page=page,
            per_page=per_page,
            pages=math.ceil(total / per_page) if total > 0 else 0,
        )
```

### backend/app/api/deps.py
```python
from fastapi import Depends
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import get_db
from app.core.exceptions import CredentialsException, ForbiddenException
from app.core.security import decode_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl='/api/v1/auth/login')


async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
):
    """Valida JWT e retorna usuário autenticado."""
    from app.models.user import User

    payload = decode_token(token)
    if not payload or payload.get('type') != 'access':
        raise CredentialsException()

    user_id = payload.get('sub')
    if not user_id:
        raise CredentialsException()

    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()

    if not user or not user.is_active:
        raise CredentialsException()

    return user


def require_role(*roles):
    """Factory de dependency que exige um dos roles informados."""
    async def _check(current_user=Depends(get_current_user)):
        if current_user.role not in roles:
            raise ForbiddenException()
        return current_user
    return _check
```

### backend/app/api/v1/router.py
```python
from fastapi import APIRouter

from app.api.v1.health import router as health_router

router = APIRouter()
router.include_router(health_router)
```

### backend/app/api/v1/health.py
```python
from fastapi import APIRouter

router = APIRouter(prefix='/health', tags=['Health'])


@router.get('')
async def health():
    """Verifica saúde da aplicação."""
    return {'status': 'ok'}
```

### backend/tests/conftest.py
```python
import pytest
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.core.config import settings
from app.core.database import get_db
from app.core.limiter import limiter
from app.main import app
from app.models.base import Base

# Banco de testes separado — nunca usar o banco de desenvolvimento
TEST_DATABASE_URL = settings.database_url.replace(
    '/' + settings.database_url.rsplit('/', 1)[-1],
    '/' + settings.database_url.rsplit('/', 1)[-1] + '_test',
)

test_engine = create_async_engine(TEST_DATABASE_URL)
TestSessionLocal = async_sessionmaker(test_engine, expire_on_commit=False)


@pytest_asyncio.fixture(autouse=True)
async def setup_database():
    """Cria e destrói as tabelas para cada teste."""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest_asyncio.fixture
async def db_session():
    """Sessão de banco isolada com rollback ao fim do teste."""
    async with TestSessionLocal() as session:
        yield session
        await session.rollback()


@pytest_asyncio.fixture
async def client(db_session: AsyncSession):
    """Cliente HTTP assíncrono com banco de teste injetado."""
    limiter.enabled = False

    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url='http://test',
    ) as ac:
        yield ac
    app.dependency_overrides.clear()
```

**Nota**: `asyncio_mode = "auto"` em `pyproject.toml` elimina a necessidade de `@pytest.mark.asyncio` em cada teste e torna o `event_loop` fixture desnecessário (removido).

### .env.example
```
# Banco de dados
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB={nome_do_projeto}
DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/{nome_do_projeto}

# Redis
REDIS_PASSWORD=redis_secret
REDIS_URL=redis://:redis_secret@redis:6379/0

# Segurança — gerar com: python -c "import secrets; print(secrets.token_hex(32))"
SECRET_KEY=troque-esta-chave-por-um-valor-seguro-em-producao

# Aplicação
DEBUG=false
CORS_ORIGINS_RAW=["http://localhost:3000"]

# Admin seed — alterar antes de ir para produção
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=Admin@1234
```

### .gitignore
```
__pycache__/
*.py[cod]
*.egg-info/
.env
.env.prod
.venv/
venv/
htmlcov/
.coverage
.coverage.*
.pytest_cache/
.ruff_cache/
dist/
build/
*.log
.DS_Store
site/
```

### docker-compose.yml
```yaml
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
    restart: unless-stopped
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./backend:/app

volumes:
  postgres_data:
  redis_data:
```

### docker-compose.prod.yml
```yaml
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
    command: redis-server --requirepass ${REDIS_PASSWORD} --save 60 1 --loglevel warning
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
    ports:
      - "8000:8000"
    env_file:
      - .env.prod
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - internal
    deploy:
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

### CLAUDE.md
```markdown
# {Nome do Projeto}

## Stack

- **Backend**: FastAPI 0.115 + Python 3.12
- **ORM**: SQLAlchemy 2.0 async + Alembic
- **DB**: PostgreSQL 16 + Redis 7
- **Jobs**: APScheduler
- **Infra**: Docker + Docker Compose
- **Deps**: Poetry
- **Lint**: Ruff (aspas simples, line-length 79, py312)
- **Testes**: pytest + pytest-asyncio

## Comandos

```bash
docker compose up -d
docker compose exec backend bash
task run
task test
task lint
task format
task migrate
task makemigrations
```

## Padrões de Código

- Aspas simples (`'`) em todo Python
- Docstrings em português
- `comment` em todas as colunas/tabelas SQLAlchemy
- StrEnum (Python 3.12), nunca `(str, enum.Enum)`
- Imports sempre no top-level
- NUNCA blocos decorados (`# ===`, `# ---`)

## Padrão de Session DB (CRÍTICO)

Services usam apenas `db.flush()`. Commit feito pelo `get_db` ao encerrar o request.
NUNCA usar `db.commit()` em services.

## Async e Lazy Loading

Em contexto async, NUNCA acessar relationships via lazy load — usa `greenlet_spawn`.
Sempre usar `selectinload()` explícito.
```

## Após criar todos os arquivos

1. Informe o usuário sobre todos os arquivos criados
2. Oriente a rodar:
```bash
cd {nome-do-projeto}
cp .env.example .env
# Editar .env com os valores corretos
docker compose up -d
```
3. Sugira os próximos passos: `/fastapi-base:add-model`, `/fastapi-base:add-endpoint`, `/fastapi-base:add-auth`
