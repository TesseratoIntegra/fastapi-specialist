---
name: add-auth
description: Adiciona módulo completo de autenticação JWT ao projeto — model User, endpoints de login/refresh/logout, password reset e middleware de autenticação. Use quando o usuário quiser implementar autenticação no projeto.
---

O usuário quer adicionar autenticação JWT ao projeto.

## O que criar

1. `backend/app/models/user.py` — model User com roles
2. `backend/app/schemas/auth.py` — schemas de login, tokens, reset
3. `backend/app/services/auth_service.py` — lógica de autenticação
4. `backend/app/api/v1/auth.py` — endpoints de auth
5. Atualizar `backend/app/api/deps.py` com `get_current_user` completo
6. Atualizar `backend/app/models/__init__.py`
7. Atualizar `backend/app/api/v1/router.py`

## Regras

- Apenas dois roles: `admin` e `customer` (StrEnum)
- Tokens: access (30 min) + refresh (7 dias), ambos JWT HS256
- Senhas: bcrypt via passlib
- `get_current_user`: valida JWT, busca usuário, verifica `is_active`
- `require_role(*roles)`: factory dependency que verifica role
- Services: apenas `db.flush()`, nunca `db.commit()`

## Conteúdo a gerar

### models/user.py
```python
import uuid
from datetime import datetime
from enum import StrEnum

from sqlalchemy import Boolean, DateTime, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin, UUIDMixin


class UserRole(StrEnum):
    """Roles disponíveis no sistema."""
    ADMIN = 'admin'
    CUSTOMER = 'customer'


class User(UUIDMixin, TimestampMixin, Base):
    """Usuário do sistema."""

    __tablename__ = 'users'
    __table_args__ = {'comment': 'Usuários do sistema'}

    email: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        nullable=False,
        index=True,
        comment='E-mail do usuário (login)',
    )
    password_hash: Mapped[str] = mapped_column(
        String(255),
        nullable=False,
        comment='Hash bcrypt da senha',
    )
    full_name: Mapped[str] = mapped_column(
        String(255),
        nullable=False,
        comment='Nome completo',
    )
    role: Mapped[UserRole] = mapped_column(
        String(20),
        nullable=False,
        default=UserRole.CUSTOMER,
        comment='Role do usuário no sistema',
    )
    is_active: Mapped[bool] = mapped_column(
        Boolean,
        nullable=False,
        default=True,
        comment='Indica se o usuário está ativo',
    )
    company_id: Mapped[uuid.UUID | None] = mapped_column(
        ForeignKey('companies.id', ondelete='SET NULL'),
        nullable=True,
        index=True,
        comment='Empresa do usuário (null para admins)',
    )
    last_login_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
        comment='Data do último login',
    )
```

### schemas/auth.py
```python
from pydantic import BaseModel, EmailStr


class LoginRequest(BaseModel):
    """Dados para autenticação."""
    email: EmailStr
    password: str


class TokenResponse(BaseModel):
    """Resposta com tokens JWT."""
    access_token: str
    refresh_token: str
    token_type: str = 'bearer'


class RefreshRequest(BaseModel):
    """Dados para renovar o access token."""
    refresh_token: str


class ChangePasswordRequest(BaseModel):
    """Dados para alterar senha."""
    current_password: str
    new_password: str


class PasswordResetRequest(BaseModel):
    """Solicitação de reset de senha."""
    email: EmailStr


class PasswordResetConfirm(BaseModel):
    """Confirmação de reset com token."""
    token: str
    new_password: str
```

### services/auth_service.py
```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.exceptions import BadRequestException, CredentialsException
from app.core.security import (
    create_access_token,
    create_refresh_token,
    decode_token,
    hash_password,
    verify_password,
)
from app.models.user import User
from app.schemas.auth import TokenResponse


async def authenticate_user(db: AsyncSession, email: str, password: str) -> User:
    """Valida credenciais e retorna o usuário autenticado."""
    result = await db.execute(select(User).where(User.email == email))
    user = result.scalar_one_or_none()

    if not user or not verify_password(password, user.password_hash):
        raise CredentialsException()
    if not user.is_active:
        raise BadRequestException('Usuário inativo')

    return user


def create_tokens_for_user(user: User) -> TokenResponse:
    """Gera par de tokens JWT para o usuário."""
    payload = {'sub': str(user.id), 'role': user.role}
    return TokenResponse(
        access_token=create_access_token(payload),
        refresh_token=create_refresh_token(payload),
    )


async def refresh_access_token(db: AsyncSession, refresh_token: str) -> TokenResponse:
    """Renova o access token a partir de um refresh token válido."""
    payload = decode_token(refresh_token)
    if not payload or payload.get('type') != 'refresh':
        raise CredentialsException()

    result = await db.execute(
        select(User).where(User.id == payload.get('sub'))
    )
    user = result.scalar_one_or_none()
    if not user or not user.is_active:
        raise CredentialsException()

    return create_tokens_for_user(user)


async def change_password(
    db: AsyncSession,
    user: User,
    current_password: str,
    new_password: str,
) -> None:
    """Altera senha do usuário após verificar a senha atual."""
    if not verify_password(current_password, user.password_hash):
        raise BadRequestException('Senha atual incorreta')
    if len(new_password) < 8:
        raise BadRequestException('Nova senha deve ter no mínimo 8 caracteres')
    user.password_hash = hash_password(new_password)
    await db.flush()


async def update_last_login(db: AsyncSession, user: User) -> None:
    """Atualiza a data do último login do usuário."""
    from datetime import UTC, datetime
    user.last_login_at = datetime.now(UTC)
    await db.flush()
```

### api/v1/auth.py
```python
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_current_user
from app.core.database import get_db
from app.schemas.auth import (
    LoginRequest,
    RefreshRequest,
    TokenResponse,
)
from app.services import auth_service

router = APIRouter(prefix='/auth', tags=['Autenticação'])


@router.post('/login', response_model=TokenResponse)
async def login(data: LoginRequest, db: AsyncSession = Depends(get_db)):
    """Autentica usuário e retorna tokens JWT."""
    user = await auth_service.authenticate_user(db, data.email, data.password)
    await auth_service.update_last_login(db, user)
    return auth_service.create_tokens_for_user(user)


@router.post('/refresh', response_model=TokenResponse)
async def refresh(data: RefreshRequest, db: AsyncSession = Depends(get_db)):
    """Renova o access token usando o refresh token."""
    return await auth_service.refresh_access_token(db, data.refresh_token)


@router.get('/me')
async def me(current_user=Depends(get_current_user)):
    """Retorna dados do usuário autenticado."""
    return {
        'id': str(current_user.id),
        'email': current_user.email,
        'full_name': current_user.full_name,
        'role': current_user.role,
    }


@router.post('/change-password')
async def change_password(
    data: ChangePasswordRequest,
    db: AsyncSession = Depends(get_db),
    current_user=Depends(get_current_user),
):
    """Altera a senha do usuário autenticado."""
    await auth_service.change_password(
        db, current_user, data.current_password, data.new_password
    )
    return {'message': 'Senha alterada com sucesso'}
```

**Nota sobre logout**: JWT é stateless — não há invalidação server-side por padrão. Para logout seguro em produção, implemente uma blacklist de tokens no Redis com TTL igual ao tempo restante do access_token. O `/refresh` com refresh_token já expirado naturalmente force o relogin.

## Após criar os arquivos

1. Importar `User` em `app/models/__init__.py`:
   ```python
   from app.models.user import User  # noqa: F401
   ```
2. Atualizar `api/v1/router.py` incluindo o router de auth:
   ```python
   from app.api.v1.auth import router as auth_router
   router.include_router(auth_router)
   ```
3. Adicionar `ChangePasswordRequest` nos imports do router de auth
4. Confirmar que `email-validator` está em `pyproject.toml` (necessário para `EmailStr`)
5. Gerar migration dentro do container:
   ```bash
   docker compose exec backend task makemigrations "add users table"
   docker compose exec backend task migrate
   ```
6. Informar ao usuário que pode usar `/fastapi-base:add-endpoint users` para criar CRUD de usuários
