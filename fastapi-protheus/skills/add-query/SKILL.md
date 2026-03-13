---
name: add-query
description: Cria uma nova query Oracle para uma tabela do Protheus (SA1, SC5, SC6, SF2, SD2, SE1, etc.). Segue o padrão de conexão direta async com oracledb sem pool. Use quando precisar buscar dados de uma nova tabela do ERP.
---

O usuário quer criar uma query Oracle para: $ARGUMENTS

## Regras obrigatórias

- Usar `oracledb.connect_async()` — NUNCA `create_pool_async()` (trava no Oracle Cloud)
- Conexão direta, abrir e fechar por operação
- DSN montado como `{host}:{port}/{service_name}` — não usar TNS
- NUNCA fazer query ao Oracle em endpoints de real-time — apenas em jobs de sync agendados
- Prefixar tabelas com schema: `{settings.oracle_schema}.SA1010`
- Filtrar registros deletados: `WHERE D_E_L_E_T_ = ' '`
- Aspas simples, docstrings em português
- Retornar lista de dicts com chaves em snake_case

## Estrutura padrão de conexão

```python
# backend/app/integrations/protheus/connection.py
import oracledb
from app.core.config import settings


async def get_oracle_connection():
    """Abre conexão direta async com Oracle (sem pool)."""
    return await oracledb.connect_async(
        user=settings.oracle_user,
        password=settings.oracle_password,
        dsn=settings.oracle_dsn,
    )
```

## Tabelas mais comuns do Protheus

| Tabela | Conteúdo | Filtros importantes |
|--------|----------|---------------------|
| SA1010 | Clientes | `A1_MSBLQL <> '1'`, `D_E_L_E_T_ = ' '` |
| SC5010 | Cabeçalho Pedidos | `C5_FILIAL`, `C5_NUM`, `D_E_L_E_T_ = ' '` |
| SC6010 | Itens Pedidos | `C6_FILIAL`, `C6_NUM`, `D_E_L_E_T_ = ' '` |
| SF2010 | NF Saída (cabeçalho) | `F2_FILIAL`, `F2_DOC`, `D_E_L_E_T_ = ' '` |
| SD2010 | NF Saída (itens) | `D2_FILIAL`, `D2_DOC`, `D_E_L_E_T_ = ' '` |
| SE1010 | Títulos a Receber | `E1_FILIAL`, `E1_NUM`, `D_E_L_E_T_ = ' '` |

## Atenção SC5010

`C5_ENTREG` NÃO EXISTE nessa tabela. Para data de entrega, usar `C5_FECENT`.

## Exemplo completo (query de clientes SA1)

```python
# backend/app/integrations/protheus/queries.py
from app.core.config import settings
from app.integrations.protheus.connection import get_oracle_connection


async def fetch_customers(limit: int = 1000, offset: int = 0) -> list[dict]:
    """Busca clientes ativos da tabela SA1 do Protheus."""
    schema = settings.oracle_schema
    sql = f"""
        SELECT
            A1_COD,
            A1_LOJA,
            A1_NOME,
            A1_NREDUZ,
            A1_CGC,
            A1_END,
            A1_MUN,
            A1_EST,
            A1_CEP,
            A1_TEL,
            A1_EMAIL
        FROM {schema}.SA1010
        WHERE D_E_L_E_T_ = ' '
          AND A1_MSBLQL <> '1'
        ORDER BY A1_COD, A1_LOJA
        OFFSET :offset ROWS FETCH NEXT :limit ROWS ONLY
    """
    conn = await get_oracle_connection()
    try:
        # oracledb: cursor() é SÍNCRONO, execute() e fetchall() são ASSÍNCRONOS
        cursor = conn.cursor()
        await cursor.execute(sql, {'offset': offset, 'limit': limit})
        columns = [col[0].lower() for col in cursor.description]
        rows = await cursor.fetchall()
        return [dict(zip(columns, row)) for row in rows]
    finally:
        await conn.close()
```

**Atenção — API do oracledb async**:
- `conn.cursor()` → **síncrono** (sem await)
- `cursor.execute()` → **assíncrono** (com await)
- `cursor.fetchall()` → **assíncrono** (com await)
- Parâmetros nomeados: passar como `dict` (`{'offset': 0, 'limit': 1000}`), não como kwargs

## Passos a executar

1. Verifique se existe `backend/app/integrations/protheus/connection.py` — se não, crie
2. Verifique se existe `backend/app/integrations/protheus/queries.py` — se não, crie
3. Adicione a nova função de query ao arquivo `queries.py`
4. Documente os campos retornados com comentário mapeando nome Protheus → snake_case
5. Informe ao usuário que a query deve ser usada apenas em tasks de sync (`/fastapi-protheus:add-sync`)

## Settings necessárias (confirme que existem em config.py)

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
