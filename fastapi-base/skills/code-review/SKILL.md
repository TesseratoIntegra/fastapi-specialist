---
name: code-review
description: Revisa código Python/FastAPI contra os padrões do projeto — aspas simples, docstrings em português, db.flush() em vez de commit, selectinload obrigatório, StrEnum, sem blocos decorados. Use quando o usuário quiser uma revisão de código.
---

O usuário quer revisar o código: $ARGUMENTS

Leia o(s) arquivo(s) indicado(s) e aplique o checklist abaixo. Para cada problema encontrado, aponte a linha e corrija diretamente.

## Checklist de revisão

### Estilo e formatação
- [ ] Aspas simples (`'`) em todo Python — nunca aspas duplas
- [ ] Line-length máximo 79 caracteres
- [ ] Sem blocos decorados (`# ===`, `# ---`, `# ***`)
- [ ] Imports no top-level (nunca dentro de funções, exceto para evitar circular import)
- [ ] Imports ordenados: stdlib → third-party → local (`app.*`)

### Docstrings e comentários
- [ ] Toda função/classe/método tem docstring em português
- [ ] `comment=` em toda coluna e tabela SQLAlchemy
- [ ] Comentários explicam o "porquê", não o "o quê"

### Models SQLAlchemy
- [ ] Herda de `UUIDMixin`, `TimestampMixin`, `Base`
- [ ] `StrEnum` — nunca `(str, enum.Enum)`
- [ ] Toda coluna monetária usa `Numeric(15, 2)`
- [ ] Toda coluna de data com `DateTime(timezone=True)`
- [ ] `__table_args__` com `comment=` na tabela
- [ ] Nomes de campos em inglês (exceto termos fiscais sem equivalente)

### Services
- [ ] Usa `db.flush()` — NUNCA `db.commit()`
- [ ] Raises `NotFoundException` para recursos não encontrados
- [ ] Não acessa relationships via lazy load (usa `selectinload()` na query)
- [ ] Nenhuma query Oracle/Protheus em tempo real (só via jobs de sync)

### Endpoints
- [ ] `get_current_user` em todos os endpoints protegidos
- [ ] `get_company_scope` em todos os endpoints multi-tenant
- [ ] Response types anotados corretamente
- [ ] Paginação com `ge=1`, `le=100` nos Query params
- [ ] Schemas com `model_config = {'from_attributes': True}`

### Segurança
- [ ] Sem secrets hardcoded — usar `settings.*`
- [ ] Sem SQL injection — usar SQLAlchemy ORM/Core, nunca `text()` com f-string
- [ ] Sem logging de senhas ou tokens

### Async
- [ ] Sem `asyncio.sleep()` desnecessário
- [ ] Sem blocking I/O em contexto async (sem `requests`, `open()` síncrono)
- [ ] `selectinload()` explícito para todo relacionamento acessado

## Formato do relatório

Para cada problema:
```
[ARQUIVO:LINHA] CATEGORIA — Descrição do problema
ANTES: {código problemático}
DEPOIS: {código corrigido}
```

Ao final, aplique todas as correções diretamente nos arquivos.
