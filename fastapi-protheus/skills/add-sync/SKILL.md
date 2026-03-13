---
name: add-sync
description: Cria uma task APScheduler de sincronização periódica Protheus → PostgreSQL local. Inclui controle de concorrência, logging, tratamento de erros e upsert eficiente. Use quando precisar sincronizar dados do Oracle para o banco local.
---

O usuário quer criar uma task de sync para: $ARGUMENTS

## Arquitetura de sync

```
APScheduler (agendado)
  → query Oracle (oracledb.connect_async)
  → mapper (Protheus dict → model dict)
  → upsert PostgreSQL (INSERT ... ON CONFLICT DO UPDATE)
  → log resultado
```

**NUNCA** fazer query Oracle em endpoints de real-time. Todo acesso ao Protheus é via jobs de sync agendados.

## Regras obrigatórias

- Controle de concorrência: flag `_is_running` por task para evitar execução paralela
  - **Atenção**: flag de módulo só funciona com **1 worker uvicorn**. Com múltiplos workers, use Redis como lock (`SET nx ex`) ou APScheduler com `coalesce=True, max_instances=1`
- Processar em batches (500–1000 registros por vez) para evitar OOM
- Usar `INSERT ... ON CONFLICT DO UPDATE` (upsert) — nunca delete + insert
- `raw_data` DEVE usar `_safe_dict(row)` do mapper — nunca `dict(row)` direto (Oracle retorna tipos não-JSON-serializáveis)
- Logar início, fim, total processado e erros
- Capturar exceções sem derrubar o scheduler
- Fechar conexão Oracle no `finally`
- Aspas simples, docstrings em português
- O scheduler cria sua própria sessão e faz `db.commit()` diretamente — não usa `get_db` (que é dependency FastAPI)

## Estrutura do scheduler

```python
# backend/app/tasks/scheduler.py
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler(timezone='America/Sao_Paulo')


def start_scheduler():
    """Registra e inicia o scheduler com as tasks agendadas."""
    from app.tasks.sync_customers import sync_customers
    from app.tasks.sync_orders import sync_orders

    scheduler.add_job(sync_customers, 'interval', hours=1, id='sync_customers')
    scheduler.add_job(sync_orders, 'interval', minutes=30, id='sync_orders')
    scheduler.start()


def stop_scheduler():
    """Para o scheduler graciosamente."""
    if scheduler.running:
        scheduler.shutdown(wait=False)
```

## Exemplo completo (sync de pedidos)

```python
# backend/app/tasks/sync_orders.py
import logging
from datetime import UTC, datetime

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import AsyncSessionLocal
from app.integrations.protheus.queries import fetch_orders
from app.integrations.protheus.mapper import map_order

logger = logging.getLogger(__name__)

_is_running = False
BATCH_SIZE = 500


async def sync_orders():
    """Sincroniza pedidos do Protheus para o banco local em batches."""
    global _is_running
    if _is_running:
        logger.warning('sync_orders: já em execução, pulando...')
        return

    _is_running = True
    started_at = datetime.now(UTC)
    total_processed = 0

    try:
        logger.info('sync_orders: iniciando sincronização')
        offset = 0

        while True:
            rows = await fetch_orders(limit=BATCH_SIZE, offset=offset)
            if not rows:
                break

            async with AsyncSessionLocal() as db:
                await _upsert_orders(db, rows)
                await db.commit()

            total_processed += len(rows)
            offset += BATCH_SIZE
            logger.info(f'sync_orders: {total_processed} pedidos processados')

            if len(rows) < BATCH_SIZE:
                break

        elapsed = (datetime.now(UTC) - started_at).total_seconds()
        logger.info(
            f'sync_orders: concluído — {total_processed} pedidos em {elapsed:.1f}s'
        )

    except Exception:
        logger.exception('sync_orders: erro durante sincronização')
    finally:
        _is_running = False


async def _upsert_orders(db: AsyncSession, rows: list[dict]):
    """Faz upsert em batch dos pedidos no banco local."""
    import json
    for row in rows:
        mapped = map_order(row)
        # raw_data precisa ser string JSON para o parâmetro do text()
        if mapped.get('raw_data') is not None:
            mapped['raw_data'] = json.dumps(mapped['raw_data'])

        await db.execute(
            text('''
                INSERT INTO orders (
                    protheus_order_number, protheus_branch, order_date,
                    delivery_date, total_value, status,
                    raw_data, created_at, updated_at
                ) VALUES (
                    :protheus_order_number, :protheus_branch, :order_date,
                    :delivery_date, :total_value, :status,
                    CAST(:raw_data AS jsonb), NOW(), NOW()
                )
                ON CONFLICT (protheus_order_number, protheus_branch)
                DO UPDATE SET
                    delivery_date = EXCLUDED.delivery_date,
                    total_value = EXCLUDED.total_value,
                    status = EXCLUDED.status,
                    raw_data = EXCLUDED.raw_data,
                    updated_at = NOW()
            '''),
            mapped,
        )
```

## Registrar no lifespan (main.py)

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    from app.tasks.scheduler import start_scheduler, stop_scheduler
    if settings.oracle_host:
        start_scheduler()
    yield
    stop_scheduler()
    await engine.dispose()
```

## Passos a executar

1. Verifique se existem a query (`/fastapi-protheus:add-query`) e o mapper (`/fastapi-protheus:add-mapper`)
2. Crie `backend/app/tasks/sync_{entidade}.py` seguindo o padrão acima
3. Registre a task no `scheduler.py`
4. Registre o scheduler no `lifespan` do `main.py` se ainda não estiver
5. Confirme que existe unique constraint na tabela para o `ON CONFLICT` funcionar
6. Adicione endpoint admin para trigger manual: `POST /admin/sync/{entidade}`
