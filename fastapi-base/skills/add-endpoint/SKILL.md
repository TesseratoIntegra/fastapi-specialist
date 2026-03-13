---
name: add-endpoint
description: Cria um endpoint FastAPI completo com service, schema e router. Inclui paginação, filtros, isolamento por company_scope e tratamento de erros. Use quando o usuário quiser expor uma entidade via API REST.
---

O usuário quer criar um endpoint REST completo. O argumento é o recurso/entidade: $ARGUMENTS

## O que criar

Para cada recurso, criar:
1. `backend/app/schemas/{recurso}.py` — schemas Pydantic de request/response
2. `backend/app/services/{recurso}_service.py` — lógica de negócio
3. `backend/app/api/v1/{recurso}.py` — router FastAPI
4. Registrar o router em `backend/app/api/v1/router.py`

## Regras obrigatórias

### Schemas
- `model_config = {'from_attributes': True}` em todos os response schemas
- Campos em inglês, espelhando o model
- Usar `PaginatedResponse[T]` para listagens
- Separar schemas de request (create/update) dos de response

### Services
- Recebem `db: AsyncSession` como primeiro argumento
- Usam `db.flush()` — NUNCA `db.commit()`
- Raises `NotFoundException` quando recurso não existe
- Raises `ConflictException` para duplicatas
- Queries com `selectinload()` quando há relacionamentos

### Endpoints
- Todos os endpoints de dados de negócio precisam de `get_current_user`
- Endpoints multi-tenant precisam de `company_scope = Depends(get_company_scope)`
- Paginação: `page: int = Query(1, ge=1)`, `per_page: int = Query(20, ge=1, le=100)`
- Response types anotados corretamente
- Tags em português para o Swagger

## Exemplo canônico

### schemas/order.py
```python
from datetime import datetime
from decimal import Decimal
from uuid import UUID

from pydantic import BaseModel

from app.models.order import OrderStatus
from app.schemas.common import PaginatedResponse


class OrderItemResponse(BaseModel):
    model_config = {'from_attributes': True}

    id: UUID
    product_code: str
    description: str
    quantity: Decimal
    unit_price: Decimal


class OrderListResponse(BaseModel):
    model_config = {'from_attributes': True}

    id: UUID
    order_number: str
    status: OrderStatus
    total_value: Decimal
    order_date: datetime
    created_at: datetime


class OrderDetailResponse(OrderListResponse):
    items: list[OrderItemResponse]
```

### services/order_service.py
```python
from uuid import UUID

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.core.exceptions import NotFoundException
from app.models.order import Order


async def list_orders(
    db: AsyncSession,
    company_ids: list[UUID] | None,
    page: int = 1,
    per_page: int = 20,
) -> tuple[list[Order], int]:
    """Lista pedidos com paginação e filtro por empresa."""
    base_query = select(Order)
    if company_ids is not None:
        base_query = base_query.where(Order.company_id.in_(company_ids))

    # count separado para não perder filtros
    total = await db.scalar(
        select(func.count(Order.id)).where(
            *([Order.company_id.in_(company_ids)] if company_ids is not None else [])
        )
    )
    result = await db.execute(
        base_query.offset((page - 1) * per_page).limit(per_page)
    )
    return list(result.scalars()), total or 0


async def get_order(db: AsyncSession, order_id: UUID) -> Order:
    """Busca pedido por ID com itens carregados."""
    result = await db.execute(
        select(Order)
        .options(selectinload(Order.items))
        .where(Order.id == order_id)
    )
    order = result.scalar_one_or_none()
    if not order:
        raise NotFoundException('Pedido não encontrado')
    return order


async def create_order(db: AsyncSession, company_id: UUID, data: dict) -> Order:
    """Cria novo pedido para a empresa informada."""
    from app.core.exceptions import ConflictException
    from app.models.order import Order

    existing = await db.scalar(
        select(Order).where(
            Order.company_id == company_id,
            Order.order_number == data['order_number'],
        )
    )
    if existing:
        raise ConflictException('Pedido já existe para esta empresa')

    order = Order(company_id=company_id, **data)
    db.add(order)
    await db.flush()
    return order
```

### api/v1/orders.py
```python
from uuid import UUID

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_company_scope, get_current_user
from app.core.database import get_db
from app.schemas.common import PaginatedResponse
from app.schemas.order import OrderDetailResponse, OrderListResponse
from app.services import order_service

router = APIRouter(prefix='/orders', tags=['Pedidos'])


@router.get('', response_model=PaginatedResponse[OrderListResponse])
async def list_orders(
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
    company_scope=Depends(get_company_scope),
    _=Depends(get_current_user),
):
    """Lista pedidos com paginação."""
    orders, total = await order_service.list_orders(db, company_scope, page, per_page)
    return PaginatedResponse.build(
        items=[OrderListResponse.model_validate(o) for o in orders],
        total=total,
        page=page,
        per_page=per_page,
    )


@router.get('/{order_id}', response_model=OrderDetailResponse)
async def get_order(
    order_id: UUID,
    db: AsyncSession = Depends(get_db),
    _=Depends(get_current_user),
):
    """Busca pedido por ID com itens."""
    order = await order_service.get_order(db, order_id)
    return OrderDetailResponse.model_validate(order)
```

## `get_company_scope` — isolamento multi-tenant

Esta dependency deve estar em `backend/app/api/deps.py`. Se ainda não existir, crie:

```python
from app.models.user import UserRole

async def get_company_scope(
    current_user=Depends(get_current_user),
) -> list[uuid.UUID] | None:
    """Retorna IDs das empresas acessíveis ao usuário.

    Admin recebe None (acesso global). Customer recebe lista com seu company_id.
    """
    if current_user.role == UserRole.ADMIN:
        return None  # sem filtro — acesso total
    if current_user.company_id is None:
        raise ForbiddenException('Usuário sem empresa associada')
    return [current_user.company_id]
```

## Passos a executar

1. Leia o model existente antes de criar schemas
2. Verifique se `get_company_scope` existe em `api/deps.py` — se não, adicione
3. Crie os 3 arquivos (schema, service, router)
4. Registre o router em `api/v1/router.py`:
   ```python
   from app.api.v1.{recurso} import router as {recurso}_router
   router.include_router({recurso}_router)
   ```
5. Se for multi-tenant: use `get_company_scope` no endpoint e filtre no service
6. Se for apenas admin: use `require_role(UserRole.ADMIN)` como dependency
