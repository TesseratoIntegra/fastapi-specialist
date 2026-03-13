---
name: add-mapper
description: Cria um mapper que transforma dados brutos do Protheus (Oracle) em models SQLAlchemy locais. Inclui tratamento de campos CHAR com espaços, datas, decimais e CNPJ. Use após criar uma query com /fastapi-protheus:add-query.
---

O usuário quer criar um mapper Protheus → model local para: $ARGUMENTS

## O que é um mapper

O mapper transforma o `dict` retornado pela query Oracle em instâncias do model SQLAlchemy local, tratando:
- Strings CHAR do Oracle (têm espaços à direita) — sempre `.strip()`
- Datas em string `'YYYYMMDD'` → `datetime.date`
- Decimais Oracle → `Decimal` Python
- CNPJ: remover formatação, manter 14 dígitos
- Campos nulos: `None` ou valor padrão
- Campos de status: mapear código Protheus → StrEnum local

## Regras obrigatórias

- Aspas simples em todo código
- Docstrings em português
- Função principal: `map_{entidade}(row: dict) -> {Model}`
- Função auxiliar: `map_{entidade}s(rows: list[dict]) -> list[{Model}]`
- Campos do model em inglês, referência Protheus no `comment` da coluna
- Nunca lançar exceção para campo faltante — usar `.get()` com default seguro
- Logar warning se campo obrigatório estiver vazio

## Helpers padrão

```python
# backend/app/integrations/protheus/mapper.py
import logging
from datetime import date
from decimal import Decimal, InvalidOperation

logger = logging.getLogger(__name__)


def _str(value) -> str:
    """Converte valor Oracle CHAR para string limpa."""
    if value is None:
        return ''
    return str(value).strip()


def _date(value) -> date | None:
    """Converte string 'YYYYMMDD' do Protheus para date."""
    s = _str(value)
    if not s or s == '00000000' or len(s) < 8:
        return None
    try:
        year = int(s[:4])
        month = int(s[4:6])
        day = int(s[6:8])
        # Datas inválidas do Protheus (ex: 99991231, mês 00, dia 00)
        if year < 1900 or year > 2100 or month == 0 or day == 0:
            return None
        return date(year, month, day)
    except (ValueError, IndexError):
        return None


def _decimal(value, default: str = '0') -> Decimal:
    """Converte valor numérico Oracle para Decimal."""
    try:
        # Oracle pode retornar Decimal nativo — converter via str para segurança
        return Decimal(str(value)) if value is not None else Decimal(default)
    except InvalidOperation:
        return Decimal(default)


def _cnpj(value) -> str:
    """Normaliza CNPJ removendo formatação — deve ter 14 dígitos."""
    digits = ''.join(filter(str.isdigit, _str(value)))
    if digits and len(digits) != 14:
        logger.warning('CNPJ com tamanho inesperado: %d dígitos (valor: %r)', len(digits), value)
    return digits


def _safe_dict(row: dict) -> dict:
    """Serializa row Oracle para JSON-safe dict (remove tipos nativos do oracledb)."""
    result = {}
    for key, val in row.items():
        if val is None:
            result[key] = None
        elif isinstance(val, Decimal):
            result[key] = str(val)
        elif isinstance(val, date):
            result[key] = val.isoformat()
        else:
            result[key] = _str(val) if hasattr(val, 'read') else val
    return result
```

**Por que `_safe_dict`?** O campo `raw_data` (JSONB) precisa de tipos JSON-serializáveis. Rows do Oracle podem conter `oracledb.LOB`, `Decimal`, `datetime.date` — todos precisam ser convertidos antes de persistir no PostgreSQL.

## Exemplo completo (mapper de pedidos SC5/SC6)

```python
from app.integrations.protheus.mapper import _date, _decimal, _str
from app.models.order import Order, OrderStatus


def map_order(row: dict) -> dict:
    """Converte linha SC5 do Protheus em dict para upsert no model Order."""
    status_map = {
        '': OrderStatus.OPEN,
        'L': OrderStatus.INVOICED,
        'E': OrderStatus.CANCELLED,
    }
    raw_status = _str(row.get('c5_nota'))
    status = status_map.get(raw_status, OrderStatus.OPEN)

    return {
        'protheus_order_number': _str(row.get('c5_num')),
        'protheus_branch': _str(row.get('c5_filial')),
        'customer_code': _str(row.get('c5_cliente')),
        'customer_branch': _str(row.get('c5_lojacli')),
        'order_date': _date(row.get('c5_emissao')),
        'delivery_date': _date(row.get('c5_fecent')),  # C5_ENTREG NÃO EXISTE
        'total_value': _decimal(row.get('c5_valbrut')),
        'status': status,
        'raw_data': _safe_dict(row),  # sempre usar _safe_dict, nunca dict(row) direto
    }
```

## Passos a executar

1. Verifique se existe `backend/app/integrations/protheus/mapper.py` — se não, crie com os helpers
2. Adicione as funções `map_{entidade}` e `map_{entidade}s`
3. Documente cada campo mapeado com a coluna Protheus de origem
4. Oriente o usuário a usar o mapper na task de sync (`/fastapi-protheus:add-sync`)
