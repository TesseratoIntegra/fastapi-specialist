---
name: add-schema
description: Cria schemas Pydantic para request e response de uma entidade existente. Segue os padrões do projeto com from_attributes, campos em inglês, PaginatedResponse e separação entre schemas de leitura e escrita. Use quando precisar criar apenas os schemas sem o endpoint ou service.
---

O usuário quer criar schemas Pydantic para: $ARGUMENTS

## Regras obrigatórias

- `model_config = {'from_attributes': True}` em todos os response schemas
- Campos em inglês, espelhando exatamente o model SQLAlchemy
- Separar schemas de **request** (create/update) dos de **response**
- Usar `PaginatedResponse[T]` de `app.schemas.common` para listagens
- Validações no schema de request (min_length, max_length, ge, le, pattern)
- Aspas simples em todo o código
- Docstrings em português
- NUNCA importar o model SQLAlchemy dentro do schema — usar tipos primitivos ou re-declarar enums

## Estrutura padrão

```python
# backend/app/schemas/{entidade}.py
from datetime import datetime
from decimal import Decimal
from uuid import UUID

from pydantic import BaseModel, EmailStr, Field

from app.models.{entidade} import {Enum}  # importar apenas enums


# Schemas de REQUEST (entrada de dados)

class {Entidade}CreateRequest(BaseModel):
    """Dados para criação de {entidade}."""
    name: str = Field(..., min_length=1, max_length=255)
    email: EmailStr
    value: Decimal = Field(..., ge=0)


class {Entidade}UpdateRequest(BaseModel):
    """Dados para atualização parcial de {entidade}."""
    name: str | None = Field(None, min_length=1, max_length=255)
    value: Decimal | None = Field(None, ge=0)


# Schemas de RESPONSE (saída de dados)

class {Entidade}Response(BaseModel):
    """Resposta com dados de {entidade}."""
    model_config = {'from_attributes': True}

    id: UUID
    name: str
    email: str
    value: Decimal
    status: {Enum}
    created_at: datetime
    updated_at: datetime


class {Entidade}ListResponse(BaseModel):
    """Resposta resumida para listagem de {entidade}s."""
    model_config = {'from_attributes': True}

    id: UUID
    name: str
    status: {Enum}
    created_at: datetime
```

## Quando usar cada schema

| Schema | Quando usar |
|--------|-------------|
| `{Entidade}CreateRequest` | `POST /entidades` — corpo da requisição |
| `{Entidade}UpdateRequest` | `PUT/PATCH /entidades/{id}` — campos opcionais |
| `{Entidade}Response` | `GET /entidades/{id}` — detalhe completo |
| `{Entidade}ListResponse` | `GET /entidades` — item dentro de `PaginatedResponse` |

## Validações comuns com Field

```python
from pydantic import BaseModel, Field, field_validator

class ExemploRequest(BaseModel):
    """Exemplo com validações."""
    name: str = Field(..., min_length=1, max_length=255)
    code: str = Field(..., pattern=r'^[A-Z0-9]+$')
    value: Decimal = Field(..., ge=Decimal('0'), le=Decimal('999999.99'))
    page: int = Field(1, ge=1)
    per_page: int = Field(20, ge=1, le=100)
    email: EmailStr  # requer email-validator instalado
    cnpj: str = Field(..., min_length=14, max_length=14)

    @field_validator('cnpj')
    @classmethod
    def validate_cnpj(cls, v: str) -> str:
        """Valida que o CNPJ contém apenas dígitos."""
        if not v.isdigit():
            raise ValueError('CNPJ deve conter apenas dígitos')
        return v
```

## Passos a executar

1. Leia o model existente em `backend/app/models/{entidade}.py`
2. Crie `backend/app/schemas/{entidade}.py` com os schemas completos
3. Se o model tem enums, importe-os do model (não redeclare)
4. Se já existe um schema parcial, leia-o antes de editar
5. Informe ao usuário que pode usar `/fastapi-base:add-endpoint {entidade}` para criar o endpoint que usa esses schemas
