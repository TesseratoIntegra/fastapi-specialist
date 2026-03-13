---
name: add-test
description: Cria testes pytest assíncronos para um endpoint ou service existente. Segue o padrão do projeto com fixtures, factories e isolamento de banco. Use quando o usuário quiser adicionar cobertura de testes.
---

O usuário quer criar testes para: $ARGUMENTS

## Regras obrigatórias

- Sempre pytest + pytest-asyncio
- Banco de testes separado (`{nome_db}_test`)
- Fixtures em `conftest.py`, testes em `tests/test_{recurso}.py`
- Cada teste é independente (tabelas recriadas via `setup_database`)
- Usar `AsyncClient` com `ASGITransport` — nunca `TestClient` síncrono
- Usar factory functions (`make_user()`, `make_order()`, etc.) para criar dados
- Testar: happy path, not found (404), unauthorized (401), forbidden (403), conflict (409)
- Aspas simples, docstrings em português
- NUNCA `db.commit()` nas fixtures — usar `db.flush()` + `await db_session.flush()`

## Estrutura padrão

Com `asyncio_mode = "auto"` no `pyproject.toml`, **não é necessário** `@pytest.mark.asyncio` em cada teste.

```python
import uuid

from httpx import AsyncClient

from app.core.security import create_access_token


def admin_token() -> str:
    """Gera token JWT de admin para uso nos testes."""
    return create_access_token({'sub': str(uuid.uuid4()), 'role': 'admin'})


def auth_header(token: str) -> dict:
    """Retorna header Authorization com Bearer token."""
    return {'Authorization': f'Bearer {token}'}


async def test_list_{recurso}_success(client: AsyncClient, db_session):
    """Deve retornar lista paginada de {recurso}s."""
    # arrange — criar dados no banco de teste
    item = {nome_model}(...)  # preencher campos obrigatórios
    db_session.add(item)
    await db_session.flush()

    token = admin_token()

    # act
    response = await client.get(
        '/api/v1/{recurso}s',
        headers=auth_header(token),
    )

    # assert
    assert response.status_code == 200
    data = response.json()
    assert data['total'] == 1
    assert data['items'][0]['id'] == str(item.id)


async def test_get_{recurso}_not_found(client: AsyncClient):
    """Deve retornar 404 para {recurso} inexistente."""
    response = await client.get(
        f'/api/v1/{recurso}s/{uuid.uuid4()}',
        headers=auth_header(admin_token()),
    )
    assert response.status_code == 404


async def test_{recurso}_unauthorized(client: AsyncClient):
    """Deve retornar 401 sem token."""
    response = await client.get('/api/v1/{recurso}s')
    assert response.status_code == 401


async def test_{recurso}_customer_isolation(client: AsyncClient, db_session):
    """Customer deve ver apenas {recurso}s da sua empresa."""
    import uuid as _uuid
    from app.core.security import create_access_token
    from app.models.user import UserRole

    company_id = _uuid.uuid4()
    other_company_id = _uuid.uuid4()

    # {recurso} da empresa do customer
    own_item = {nome_model}(company_id=company_id, ...)
    # {recurso} de outra empresa
    other_item = {nome_model}(company_id=other_company_id, ...)
    db_session.add_all([own_item, other_item])
    await db_session.flush()

    # token de customer com company_id específico
    token = create_access_token({
        'sub': str(_uuid.uuid4()),
        'role': UserRole.CUSTOMER,
        'company_id': str(company_id),
    })

    response = await client.get(
        '/api/v1/{recurso}s',
        headers=auth_header(token),
    )
    assert response.status_code == 200
    data = response.json()
    assert data['total'] == 1
    assert data['items'][0]['id'] == str(own_item.id)
```

## Passos a executar

1. Leia o arquivo do endpoint (`api/v1/{recurso}.py`) e do service (`services/{recurso}_service.py`)
2. Leia `tests/conftest.py` para ver fixtures disponíveis
3. Crie `tests/test_{recurso}.py` com testes completos
4. Se precisar de novas factory functions, adicione ao `conftest.py`
5. Informe ao usuário como rodar: `docker compose exec backend pytest tests/test_{recurso}.py -v`
