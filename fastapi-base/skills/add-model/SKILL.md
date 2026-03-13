---
name: add-model
description: Cria um novo model SQLAlchemy seguindo todos os padrões do projeto (UUIDMixin, TimestampMixin, StrEnum, comments em colunas, selectinload). Use quando o usuário quiser adicionar uma nova entidade ao banco de dados.
---

O usuário quer criar um novo model SQLAlchemy. O argumento é o nome do model: $ARGUMENTS

## Regras obrigatórias

- Herdar de `UUIDMixin`, `TimestampMixin` e `Base` (nessa ordem)
- **Toda coluna** deve ter `comment=` descrevendo o campo
- Enums: sempre `StrEnum` (Python 3.12), nunca `(str, enum.Enum)`
- Relacionamentos async: usar `relationship()` com `lazy='raise'` ou sem lazy (usar `selectinload()` nas queries)
- Campos monetários: `Numeric(15, 2)`
- Campos de data com timezone: `DateTime(timezone=True)`
- Tabela com `__tablename__` em inglês, plural, snake_case
- `__table_args__` com `comment=` descrevendo a tabela
- Nomes de campos em inglês
- Aspas simples em todo o código
- Docstrings em português
- NUNCA `db.commit()` — services usam `db.flush()`

## Exemplo canônico (model Order)

```python
import uuid
from datetime import datetime
from decimal import Decimal
from enum import StrEnum

from sqlalchemy import DateTime, ForeignKey, Index, Numeric, String, UniqueConstraint
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin, UUIDMixin


class OrderStatus(StrEnum):
    """Status possíveis de um pedido."""
    OPEN = 'open'
    INVOICED = 'invoiced'
    CANCELLED = 'cancelled'


class Order(UUIDMixin, TimestampMixin, Base):
    """Pedido de venda sincronizado do ERP."""

    __tablename__ = 'orders'
    __table_args__ = (
        Index('ix_orders_company_date', 'company_id', 'order_date'),
        UniqueConstraint('company_id', 'order_number', name='uq_orders_company_number'),
        {'comment': 'Pedidos de venda'},
    )

    company_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey('companies.id', ondelete='CASCADE'),
        nullable=False,
        index=True,
        comment='Empresa dona do pedido',
    )
    order_number: Mapped[str] = mapped_column(
        String(20),
        nullable=False,
        comment='Número do pedido',
    )
    order_date: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
        comment='Data de emissão do pedido',
    )
    status: Mapped[OrderStatus] = mapped_column(
        String(20),
        nullable=False,
        default=OrderStatus.OPEN,
        comment='Status atual do pedido',
    )
    total_value: Mapped[Decimal] = mapped_column(
        Numeric(15, 2),
        nullable=False,
        default=Decimal('0'),
        comment='Valor total do pedido',
    )
    raw_data: Mapped[dict | None] = mapped_column(
        JSONB,
        nullable=True,
        comment='Payload bruto do ERP para auditoria',
    )

    company: Mapped['Company'] = relationship(back_populates='orders')
    items: Mapped[list['OrderItem']] = relationship(
        back_populates='order',
        cascade='all, delete-orphan',
    )
```

## Tipos de coluna por caso de uso

| Caso | Tipo SQLAlchemy | Python type hint |
|------|----------------|-----------------|
| Dinheiro | `Numeric(15, 2)` | `Decimal` |
| Data + hora | `DateTime(timezone=True)` | `datetime` |
| Data + hora opcional | `DateTime(timezone=True)` | `datetime \| None` |
| Texto curto | `String(N)` | `str` |
| Texto longo | `Text` | `str` |
| Flag booleana | `Boolean` | `bool` |
| JSON livre | `JSONB` (PostgreSQL) | `dict \| None` |
| UUID FK | `ForeignKey('tabela.id')` | `uuid.UUID` |
| Enum | `String(N)` + `StrEnum` | `NomeEnum` |

## Quando usar `__table_args__` como tupla

Quando há múltiplas constraints/índices além do `comment`:
```python
__table_args__ = (
    Index('ix_nome', 'col1', 'col2'),
    UniqueConstraint('col1', 'col2', name='uq_nome'),
    {'comment': 'Descrição da tabela'},
)
```

Quando há apenas `comment`:
```python
__table_args__ = {'comment': 'Descrição da tabela'}
```

## Passos a executar

1. Leia `backend/app/models/base.py` para confirmar imports corretos
2. Crie `backend/app/models/{nome_em_snake_case}.py` com o model completo
3. Adicione import no `backend/app/models/__init__.py`:
   ```python
   from app.models.{nome_em_snake_case} import {NomeModel}  # noqa: F401
   ```
4. Informe o usuário para rodar dentro do container:
   ```bash
   docker compose exec backend task makemigrations "add {nome} table"
   docker compose exec backend task migrate
   ```

## Perguntas que você deve fazer antes de criar (se não estiverem claras no argumento)

- Quais campos o model precisa?
- Tem relacionamentos com outros models?
- Tem enums? Quais os valores?
- É multi-tenant (precisa de `company_id`)?
- Tem dados vindos de ERP externo? (usar campo `raw_data: JSONB`)
- Tem unicidade composta? (usar `UniqueConstraint` em `__table_args__`)
