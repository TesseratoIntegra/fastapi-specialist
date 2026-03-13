---
name: add-table
description: Documenta e configura integração com uma nova tabela do Protheus (TOTVS). Identifica campos relevantes, convenções de nomenclatura do ERP e cria o mapeamento completo. Use quando precisar integrar uma nova tabela do Protheus que ainda não está mapeada.
---

O usuário quer documentar/integrar a tabela Protheus: $ARGUMENTS

## Tabelas já mapeadas (referência)

| Sigla | Tabela Oracle | Conteúdo | Campos-chave |
|-------|---------------|----------|--------------|
| SA1 | SA1010 | Clientes | A1_COD, A1_LOJA, A1_CGC |
| SC5 | SC5010 | Cabeçalho Pedidos | C5_FILIAL, C5_NUM, C5_CLIENTE |
| SC6 | SC6010 | Itens Pedidos | C6_FILIAL, C6_NUM, C6_ITEM |
| SF2 | SF2010 | NF Saída (cabeçalho) | F2_FILIAL, F2_DOC, F2_SERIE |
| SD2 | SD2010 | NF Saída (itens) | D2_FILIAL, D2_DOC, D2_ITEM |
| SE1 | SE1010 | Títulos a Receber | E1_FILIAL, E1_NUM, E1_PREFIXO, E1_PARCELA |

## Convenções do Protheus

- Todas as tabelas têm sufixo `010` (filial 01, empresa 0)
- `D_E_L_E_T_` = `' '` → registro ativo (sempre filtrar)
- Campos CHAR: sempre `.strip()` ao ler
- Datas: formato `'YYYYMMDD'` como string
- Decimais: tipo NUMBER do Oracle
- Campos de 2 letras = código de filial (`XX_FILIAL`)
- Soft delete: `D_E_L_E_T_ = '*'`

## O que produzir

1. **Documentação da tabela** — listar campos relevantes com tipo, tamanho e significado
2. **Model SQLAlchemy** — mapeamento com nomes em inglês e `comment` referenciando campo Oracle
3. **Query Oracle** — SELECT com filtros obrigatórios
4. **Mapper** — transformação dos dados
5. **Orientação de sync** — frequência recomendada e campos de controle de atualização

## Template de documentação

```markdown
## Tabela {SIGLA}010 — {Nome}

### Campos mapeados

| Campo Oracle | Tipo | Tamanho | Campo local | Descrição |
|-------------|------|---------|-------------|-----------|
| XX_FILIAL   | CHAR | 2       | branch      | Filial do registro |
| XX_COD      | CHAR | 6       | code        | Código |
| XX_NOME     | CHAR | 40      | name        | Nome/descrição |
| XX_EMISSAO  | CHAR | 8       | issue_date  | Data de emissão (YYYYMMDD) |
| XX_VALOR    | NUM  | 15,2    | value       | Valor |

### Filtros obrigatórios
- `D_E_L_E_T_ = ' '` — exclui registros deletados
- `{filtros específicos da tabela}`

### Chave única para upsert
`({campos que identificam unicamente um registro})`

### Frequência de sync recomendada
- {Justificativa baseada na volatilidade dos dados}
```

## Passos a executar

1. Se o usuário não souber os campos, solicite acesso ao dicionário de dados do Protheus ou pergunte quais informações ele precisa extrair
2. Gere a documentação da tabela
3. Crie o model SQLAlchemy correspondente
4. Oriente a usar `/fastapi-protheus:add-query` para a query e `/fastapi-protheus:add-mapper` para o mapper
5. Oriente a usar `/fastapi-protheus:add-sync` para criar a task de sincronização
