---
name: new-migration
description: Gera uma nova migration Alembic após mudanças nos models SQLAlchemy. Verifica se os models estão importados no env.py e orienta sobre boas práticas de migration. Use após adicionar ou alterar models.
---

O usuário precisa gerar uma migration Alembic. Descrição: $ARGUMENTS

## env.py customizado (obrigatório para SQLAlchemy async)

O Alembic não suporta engines async nativamente. O `env.py` gerado pelo `init-project` já inclui a configuração correta — confirme que ela existe antes de rodar migrations:

```python
# backend/alembic/env.py
import asyncio
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from app.models import Base  # garante que todos os models são importados

target_metadata = Base.metadata


def do_run_migrations(connection: Connection) -> None:
    """Executa as migrations dentro de uma conexão síncrona."""
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    """Cria o engine async e executa as migrations via run_sync."""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix='sqlalchemy.',
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    """Wrapper síncrono que o Alembic chama — entra no loop async."""
    asyncio.run(run_async_migrations())
```

Se o `env.py` estiver com o padrão síncrono padrão do Alembic (sem `async_engine_from_config`), as migrations **não funcionarão** com engines `asyncpg`.

## Passos a executar

1. **Verifique se o model está importado no `alembic/env.py`**
   - Leia `backend/alembic/env.py`
   - Confirme que o novo model está no `from app.models import *` ou tem import explícito
   - Se não estiver, adicione ao `backend/app/models/__init__.py`

2. **Gere a migration**

   O comando `makemigrations` no taskipy é definido como `alembic revision --autogenerate -m`, portanto a mensagem é passada diretamente como argumento posicional:
   ```bash
   docker compose exec backend task makemigrations "add invoices table"
   # equivale a: alembic revision --autogenerate -m "add invoices table"
   ```
   Exemplos de mensagem: `"add invoices table"`, `"add status column to orders"`, `"rename user email field"`

3. **Revise o arquivo gerado** em `alembic/versions/`
   - Verifique se `upgrade()` e `downgrade()` estão corretos
   - Confirme que `downgrade()` é reversível (não apaga dados desnecessariamente)
   - Cheque se há índices, unique constraints e foreign keys

4. **Execute a migration**
   ```bash
   docker compose exec backend task migrate
   ```

5. **Confirme que aplicou**
   ```bash
   docker compose exec backend alembic current
   ```

## Boas práticas de migration

- **Uma migration por mudança lógica** — não agrupe alterações não relacionadas
- **Sempre implemente `downgrade()`** — permite reverter com `alembic downgrade -1`
- **Cuidado com dados existentes** — se renomear coluna, faça em 3 steps (add nova, migrar dados, drop antiga)
- **Índices têm `IF NOT EXISTS`** — use `if_not_exists=True` no Alembic
- **Nunca edite migrations já commitadas** — crie nova migration para corrigir

## Enum changes no PostgreSQL (cuidado especial)

PostgreSQL armazena enums como tipos nativos. Adicionar valor é seguro, renomear/remover não é:

```python
# SEGURO — adicionar novo valor
def upgrade():
    op.execute("ALTER TYPE orderstatus ADD VALUE 'pending'")

def downgrade():
    # NÃO dá pra remover valor de enum no PostgreSQL sem recriar o tipo
    pass

# RENOMEAR valor — requer sequência completa
def upgrade():
    # 1. Adiciona novo valor
    op.execute("ALTER TYPE orderstatus ADD VALUE 'processing'")
    # 2. Migra dados
    op.execute("UPDATE orders SET status = 'processing' WHERE status = 'pending'")
    # 3. Não é possível remover o valor antigo no PostgreSQL 12 ou anterior
    #    No PostgreSQL 14+: ALTER TYPE orderstatus RENAME VALUE 'pending' TO 'processing'
```

## Data migrations (migração de dados junto com schema)

```python
from alembic import op
import sqlalchemy as sa
from sqlalchemy.sql import table, column

def upgrade():
    # 1. Alteração de schema
    op.add_column('orders', sa.Column('new_field', sa.String(50), nullable=True))

    # 2. Migração de dados usando helpers do Alembic (sem ORM — usa Core)
    orders = table('orders',
        column('id', sa.String),
        column('old_field', sa.String),
        column('new_field', sa.String),
    )
    op.execute(
        orders.update().values(new_field=orders.c.old_field)
    )

    # 3. Tornar not null depois de popular
    op.alter_column('orders', 'new_field', nullable=False)

def downgrade():
    op.drop_column('orders', 'new_field')
```

## Problemas comuns

**Migration vazia (`pass` em upgrade/downgrade)**
→ Model não está importado no `env.py`. Verifique `app/models/__init__.py`.

**`Can't locate revision`**
→ Rode `alembic history` e verifique se há conflito de branches.

**`Target database is not up to date`**
→ Rode `task migrate` antes de gerar nova migration.
