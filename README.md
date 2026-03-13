# fastapi-specialist

Monorepo com dois plugins Claude Code para inicialização e desenvolvimento de projetos FastAPI seguindo padrões de produção — com SQLAlchemy async, PostgreSQL, Redis, Docker multi-stage, GitHub Actions CI/CD e integração com ERP Protheus (TOTVS) via Oracle DB.

## Plugins

| Plugin | Foco |
|--------|------|
| `fastapi-base` | Estrutura completa do projeto, models, endpoints, auth, testes e DevOps |
| `fastapi-protheus` | Integração com Protheus via Oracle DB — queries, mappers e sync jobs |

---

## Instalação

### Via marketplace (futuro)

```bash
/plugin marketplace add TesseratoIntegra/fastapi-specialist
/plugin install fastapi-base
/plugin install fastapi-protheus
```

### Local (uso imediato)

```bash
git clone https://github.com/TesseratoIntegra/fastapi-specialist.git ~/fastapi-specialist

# Abra o Claude Code com os plugins carregados
claude --plugin-dir ~/fastapi-specialist/fastapi-base
claude --plugin-dir ~/fastapi-specialist/fastapi-protheus

# Ou adicione ambos de uma vez
claude \
  --plugin-dir ~/fastapi-specialist/fastapi-base \
  --plugin-dir ~/fastapi-specialist/fastapi-protheus
```

---

## fastapi-base — Skills

### `/fastapi-base:init-project`

Gera um projeto FastAPI completo do zero com toda a estrutura de produção.

**Uso:**
```
/fastapi-base:init-project nome-do-projeto
```

**O que cria:**
- `pyproject.toml` com Poetry (FastAPI, SQLAlchemy, asyncpg, Alembic, Redis, JWT, Ruff, pytest)
- `Taskfile` com comandos `run`, `test`, `test-cov`, `lint`, `format`, `migrate`, `makemigrations`
- `backend/app/main.py` com lifespan, CORS e routers
- `backend/app/core/config.py` com Settings via pydantic-settings
- `backend/app/database.py` com engine async e `get_db`
- `backend/app/models/base.py` com `TimestampMixin` (`created_at`, `updated_at` com `onupdate=func.now()`)
- `backend/app/models/__init__.py` com import de Base (obrigatório para Alembic autogenerate)
- `backend/app/core/security.py` com bcrypt + JWT HS256
- `backend/app/core/exceptions.py` com handlers HTTP padronizados
- `backend/alembic/env.py` customizado com suporte async (`asyncio.run` + `run_sync`)
- `Dockerfile` multi-stage (builder + runtime, usuário não-root)
- `.dockerignore`
- `docker-compose.yml` (dev) com PostgreSQL 16 + Redis 7 + backend
- `docker-compose.prod.yml` (produção) sem volumes de código
- `entrypoint.sh` com wait-for-db → alembic upgrade head → seed admin → uvicorn
- `.env.example` com todas as variáveis necessárias
- `.gitignore` completo
- `tests/conftest.py` com fixtures async (banco de teste isolado)

**Stack gerada:**
```
FastAPI 0.115    SQLAlchemy 2.0 async    asyncpg
Alembic          Redis (aioredis)        JWT HS256 (python-jose)
bcrypt/passlib   pydantic-settings       email-validator
Ruff             pytest-asyncio          httpx (test client)
APScheduler      slowapi (rate limit)
```

---

### `/fastapi-base:add-model`

Cria um model SQLAlchemy ORM seguindo todos os padrões do projeto.

**Uso:**
```
/fastapi-base:add-model Invoice
/fastapi-base:add-model Order com campos: company_id, total_value, status
```

**O que cria:**
- `backend/app/models/{entidade}.py` com:
  - `TimestampMixin` (herda `created_at`, `updated_at`)
  - `StrEnum` para status (Python 3.12, nunca `(str, enum.Enum)`)
  - `comment` em todas as colunas e na tabela
  - `UniqueConstraint` + `Index` em `__table_args__` quando necessário
  - Suporte a JSONB para `raw_data`
  - ForeignKey com `ondelete` explícito

**Padrões aplicados:**
- `Mapped[datetime | None]` — nunca `Mapped[DateTime | None]`
- `mapped_column()` com `nullable=True/False` explícito
- Nomes de campos em inglês, referência Protheus no `comment`
- StrEnum com valores em inglês (`OPEN`, `ISSUED`, `CANCELLED`)

---

### `/fastapi-base:add-schema`

Cria schemas Pydantic para request e response de uma entidade.

**Uso:**
```
/fastapi-base:add-schema Invoice
/fastapi-base:add-schema Order
```

**O que cria:**
- `backend/app/schemas/{entidade}.py` com:
  - `{Entidade}CreateRequest` — campos obrigatórios com validações
  - `{Entidade}UpdateRequest` — campos opcionais (PATCH)
  - `{Entidade}Response` — resposta completa com `model_config = {'from_attributes': True}`
  - `{Entidade}ListResponse` — versão resumida para uso em `PaginatedResponse[T]`

**Padrões:**
- `Field(..., min_length=1, max_length=255)` para strings
- `Field(..., ge=0)` para decimais/inteiros
- `field_validator` para CNPJ e outros campos customizados
- `PaginatedResponse[T]` de `app.schemas.common` para listagens

---

### `/fastapi-base:add-endpoint`

Cria router, service e schemas para um novo recurso REST.

**Uso:**
```
/fastapi-base:add-endpoint orders
/fastapi-base:add-endpoint invoices com filtros: status, date_from, date_to
```

**O que cria:**
- `backend/app/api/v1/{entidade}.py` — router FastAPI com endpoints CRUD
- `backend/app/services/{entidade}_service.py` — lógica de negócio
- Schemas (se não existirem)

**Endpoints gerados:**
```
GET    /api/v1/{entidades}          # Listagem paginada
GET    /api/v1/{entidades}/{id}     # Detalhe
POST   /api/v1/{entidades}          # Criação
PUT    /api/v1/{entidades}/{id}     # Atualização
DELETE /api/v1/{entidades}/{id}     # Remoção (soft delete quando aplicável)
```

**Padrões críticos:**
- `get_company_scope` aplicado em todos os endpoints de dados de negócio
- `selectinload()` explícito para relacionamentos (nunca lazy load em async)
- `db.flush()` em services — commit feito pelo `get_db` ao encerrar o request
- Query de `count` separada da query de listagem (correta para paginação)
- Filtragem por `company_ids` via `IN` clause

---

### `/fastapi-base:add-auth`

Adiciona autenticação JWT completa ao projeto.

**Uso:**
```
/fastapi-base:add-auth
```

**O que cria/configura:**
- `backend/app/models/user.py` — model User com `UserRole` StrEnum, `company_id` FK, `last_login`, `is_active`
- `backend/app/schemas/auth.py` — `LoginRequest`, `TokenResponse`, `ChangePasswordRequest`
- `backend/app/api/v1/auth.py` — endpoints `/login`, `/refresh`, `/me`, `/change-password`
- `backend/app/api/deps.py` — `get_current_user`, `get_company_scope`, `require_admin`
- Configurações JWT em `settings`: `SECRET_KEY`, `ACCESS_TOKEN_EXPIRE_MINUTES=30`, `REFRESH_TOKEN_EXPIRE_DAYS=7`

**Roles:**
- `admin` — acesso global, `company_scope = None`
- `customer` — filtrado por `company_id` do JWT, `company_scope = [UUID]`

**Segurança:**
- Tokens HS256 com payload `sub`, `role`, `company_id`, `exp`
- Logout stateless (cliente descarta o token — sem blacklist por padrão)
- `change-password` exige senha atual antes de alterar

---

### `/fastapi-base:add-test`

Cria testes pytest profissionais para um endpoint ou service.

**Uso:**
```
/fastapi-base:add-test orders
/fastapi-base:add-test auth
```

**O que cria:**
- `tests/test_{entidade}.py` com:
  - Fixtures isoladas por teste (banco limpo)
  - Helpers `admin_token()` e `auth_header()`
  - Testes de autenticação (401 sem token, 403 sem permissão)
  - Testes de isolamento multi-tenant (customer A não vê dados de customer B)
  - Testes de CRUD completo
  - Testes de validação de campos

**Configuração:**
- `asyncio_mode = "auto"` — sem `@pytest.mark.asyncio` necessário
- `pytest-asyncio` com `AsyncClient` do httpx
- Banco de testes isolado: `portal_cliente_test`
- Sem mock de banco — testes de integração reais

---

### `/fastapi-base:new-migration`

Gera e orienta criação de migrations Alembic.

**Uso:**
```
/fastapi-base:new-migration add_invoices_table
/fastapi-base:new-migration add_status_column_to_orders
```

**O que faz:**
- Verifica se o model está importado em `models/__init__.py`
- Orienta o comando `task makemigrations "mensagem"`
- Documenta o `env.py` async customizado (obrigatório para SQLAlchemy async):

```python
# alembic/env.py — padrão gerado pelo init-project
async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix='sqlalchemy.',
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()

def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())
```

- Avisa sobre mudanças de enum no PostgreSQL (requer `ALTER TYPE` manual)
- Mostra padrão de data migration com Core helpers

---

### `/fastapi-base:devops`

Configura toda a infraestrutura Docker e pipeline CI/CD.

**Uso:**
```
/fastapi-base:devops
```

**O que cria:**
- `Dockerfile` multi-stage:
  - Stage `builder`: instala deps com Poetry, exporta `requirements.txt`
  - Stage `runtime`: imagem enxuta com usuário não-root (`appuser`)
- `docker-compose.yml` (desenvolvimento) com hot-reload
- `docker-compose.prod.yml` (produção) com image registry
- `.github/workflows/ci.yml`:
  - Lint com Ruff
  - Testes com PostgreSQL service + coverage
  - Upload de cobertura para Codecov (`coverage.xml`)
- `.github/workflows/deploy.yml`:
  - Build e push para GHCR
  - Deploy via SSH com health check e rollback automático

**Health check + rollback:**
```yaml
for i in $(seq 1 12); do
  if curl -sf http://localhost:8000/health > /dev/null; then
    echo "Deploy OK"; break
  fi
  # após 12 tentativas (60s): rollback para imagem anterior
done
```

---

## fastapi-protheus — Skills

> Todas as skills de Protheus pressupõem que `fastapi-base` já foi configurado.
> **Nunca** fazer query Oracle em real-time — apenas em jobs de sync agendados.

---

### `/fastapi-protheus:add-query`

Cria query Oracle async para uma tabela do Protheus.

**Uso:**
```
/fastapi-protheus:add-query SA1
/fastapi-protheus:add-query SC5 pedidos dos últimos 90 dias
```

**O que cria/atualiza:**
- `backend/app/integrations/protheus/connection.py` (se não existir)
- `backend/app/integrations/protheus/queries.py` com nova função `fetch_{entidade}()`

**Padrão de conexão obrigatório:**
```python
# CORRETO — conexão direta, sem pool
conn = await oracledb.connect_async(
    user=settings.oracle_user,
    password=settings.oracle_password,
    dsn=settings.oracle_dsn,  # "host:port/service_name"
)

# API async do oracledb
cursor = conn.cursor()              # SÍNCRONO — sem await
await cursor.execute(sql, params)   # ASSÍNCRONO
rows = await cursor.fetchall()      # ASSÍNCRONO
```

**Tabelas suportadas:**

| Tabela | Conteúdo | Filtros obrigatórios |
|--------|----------|----------------------|
| SA1010 | Clientes | `D_E_L_E_T_ = ' '`, `A1_MSBLQL <> '1'` |
| SC5010 | Pedidos (cabeçalho) | `D_E_L_E_T_ = ' '` |
| SC6010 | Pedidos (itens) | `D_E_L_E_T_ = ' '` |
| SF2010 | NF Saída (cabeçalho) | `D_E_L_E_T_ = ' '` |
| SD2010 | NF Saída (itens) | `D_E_L_E_T_ = ' '` |
| SE1010 | Títulos a Receber | `D_E_L_E_T_ = ' '` |

> **Atenção SC5010**: `C5_ENTREG` não existe — usar `C5_FECENT` para data de entrega.

**Settings necessárias em `config.py`:**
```python
oracle_user: str
oracle_password: str
oracle_host: str
oracle_port: int = 1521
oracle_service_name: str
oracle_schema: str

@property
def oracle_dsn(self) -> str:
    return f'{self.oracle_host}:{self.oracle_port}/{self.oracle_service_name}'
```

---

### `/fastapi-protheus:add-mapper`

Cria mapper que transforma dados Oracle → model SQLAlchemy local.

**Uso:**
```
/fastapi-protheus:add-mapper orders
/fastapi-protheus:add-mapper customers
```

**O que cria/atualiza:**
- `backend/app/integrations/protheus/mapper.py` com helpers e funções de mapeamento

**Helpers incluídos:**

```python
_str(value)          # CHAR Oracle → string limpa (strip)
_date(value)         # 'YYYYMMDD' → date (trata datas inválidas: 00000000, 99991231)
_decimal(value)      # Oracle NUMBER → Decimal Python
_cnpj(value)         # CNPJ → 14 dígitos sem formatação
_safe_dict(row)      # row Oracle → JSON-safe dict (para campo JSONB raw_data)
```

> `_safe_dict()` é obrigatório para `raw_data` — Oracle retorna `LOB`, `Decimal`, `datetime.date`
> que não são JSON-serializáveis nativamente.

---

### `/fastapi-protheus:add-sync`

Cria task APScheduler de sincronização periódica Protheus → PostgreSQL.

**Uso:**
```
/fastapi-protheus:add-sync orders
/fastapi-protheus:add-sync customers intervalo: 1h
```

**O que cria/atualiza:**
- `backend/app/tasks/sync_{entidade}.py` com lógica de sync em batches
- `backend/app/tasks/scheduler.py` com registro da nova task
- Orienta atualização do `lifespan` em `main.py`

**Padrões da task:**

```python
_is_running = False      # flag de concorrência por módulo
BATCH_SIZE = 500         # processar em lotes para evitar OOM

async def sync_orders():
    global _is_running
    if _is_running:
        return  # evita execução paralela
    _is_running = True
    try:
        # loop de batches com offset
        # AsyncSessionLocal() — cria própria sessão, faz db.commit() diretamente
        # (diferente de services que usam get_db)
    finally:
        _is_running = False
```

> **Atenção**: flag de módulo funciona apenas com **1 worker uvicorn**.
> Com múltiplos workers: usar Redis lock (`SET nx ex`) ou `APScheduler(coalesce=True, max_instances=1)`.

**Upsert JSONB:**
```python
# raw_data deve ser serializado antes de passar como parâmetro
mapped['raw_data'] = json.dumps(mapped['raw_data'])

# Na query SQL
CAST(:raw_data AS jsonb)
```

---

## Padrões Técnicos Aplicados em Todas as Skills

### Código Python
- Aspas simples `'` em todo o código
- Docstrings em português
- `comment` em todas as colunas e tabelas SQLAlchemy
- StrEnum (Python 3.12) — nunca `(str, enum.Enum)`
- Imports no top-level — nunca dentro de funções
- Nomes de campos, classes, enums e tabelas em **inglês**
- NUNCA blocos de comentário decorados (`# ===`, `# ---`)

### Session e Transações
- Services: `db.flush()` — nunca `db.commit()`
- Commit feito pelo `get_db` ao encerrar o request (via `try/finally`)
- Tasks de sync: `AsyncSessionLocal()` com `db.commit()` direto (fora do contexto HTTP)

### Async e ORM
- `selectinload()` obrigatório para qualquer relationship
- Nunca lazy load em contexto async (`greenlet_spawn` error)
- `async_engine_from_config` + `NullPool` no Alembic env.py

### Segurança e Isolamento
- Isolamento multi-tenant via `company_scope` (obrigatório em endpoints de negócio)
- Admin: `company_scope = None` (acesso global)
- Customer: `company_scope = [company_id]` (filtrado por empresa)
- Docker: usuário não-root (`appuser`)
- Secrets via variáveis de ambiente (nunca hardcoded)

---

## Estrutura do Repositório

```
fastapi-specialist/
├── fastapi-base/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       ├── init-project/
│       │   └── SKILL.md
│       ├── add-model/
│       │   └── SKILL.md
│       ├── add-schema/
│       │   └── SKILL.md
│       ├── add-endpoint/
│       │   └── SKILL.md
│       ├── add-auth/
│       │   └── SKILL.md
│       ├── add-test/
│       │   └── SKILL.md
│       ├── new-migration/
│       │   └── SKILL.md
│       └── devops/
│           └── SKILL.md
├── fastapi-protheus/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       ├── add-query/
│       │   └── SKILL.md
│       ├── add-mapper/
│       │   └── SKILL.md
│       └── add-sync/
│           └── SKILL.md
└── README.md
```

---

## Projeto de Referência

As skills deste repositório foram extraídas e generalizadas a partir do projeto **Portal do Cliente** — plataforma B2B que integra FastAPI com ERP Protheus (TOTVS) via Oracle DB, com sincronização de pedidos, notas fiscais e títulos a receber.

Padrões validados em produção:
- `oracledb.connect_async()` sem pool (pool trava no Oracle Cloud)
- `run_async_migrations()` no Alembic env.py (SQLAlchemy async não funciona com env.py padrão)
- `_safe_dict()` para serializar rows Oracle em JSONB
- `company_scope` como filtro obrigatório em todos os endpoints de negócio

---

## Contribuindo

```bash
git clone https://github.com/TesseratoIntegra/fastapi-specialist.git
cd fastapi-specialist

# Edite a skill desejada
# Cada skill é um único arquivo SKILL.md em skills/{nome}/SKILL.md

# Teste localmente
claude --plugin-dir ./fastapi-base
# No Claude: /fastapi-base:add-model MinhaEntidade
```

Para adicionar uma nova skill:
1. Crie o diretório `skills/{nome-da-skill}/`
2. Crie o `SKILL.md` com frontmatter `name` e `description`
3. Use `$ARGUMENTS` como placeholder para os argumentos do usuário
4. Documente os passos em português

---

## Autor

[TesseratoIntegra](https://github.com/TesseratoIntegra)
