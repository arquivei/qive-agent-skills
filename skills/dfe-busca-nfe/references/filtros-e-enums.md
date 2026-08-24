# Referência — filtros e enums das tools de DFe (MCP da Qive)

Referência fiel das tools de DFe do MCP da Qive (`search_nfe` e `statistics_nfe`). **Nenhum campo ou
enum listado aqui pode ser inventado ou estendido** — se um filtro não está nesta tabela, ele não existe
na tool.

## 1. `search_nfe`

Busca NF-e da conta (**por padrão** os últimos 90 dias; use `emission_date_start`/`emission_date_end`
para definir o período), retornando **10 documentos por página**, com paginação por cursor.

### 1.1 Filtros (todos opcionais)

| Parâmetro | Tipo | Valores / observação |
|---|---|---|
| `cnpjs` | `[]string` | CNPJs das empresas da conta (apenas dígitos, sem pontuação) |
| `access_keys` | `[]string` | Chaves de acesso da NF-e (44 dígitos) |
| `roles` | `[]string` | enum: `received`, `emitted`, `transporter`, `authorized` — ver tabela 1.5. Filtrado pela permissão do usuário (ver §1.7) |
| `fin_nfe` | `[]int` | finalidade da nota (1 a 6) — ver tabela 1.2 |
| `event_type` | `[]string` | tipos de evento da NF-e |
| `cfop` | `[]string` | CFOPs (código fiscal de operação) |
| `manifestation_type` | `[]string` | enum de manifestação do destinatário — ver tabela 1.3 |
| `status` | `[]string` | enum de status da nota — ver tabela 1.4 |
| `cnpjs_emitter` | `[]string` | CNPJ do emitente (apenas dígitos) |
| `emission_date_start` | string | `AAAA-MM-DD`, inclusive. Só a inicial informada ⇒ dessa data em diante. Ambas omitidas ⇒ últimos 90 dias (default) |
| `emission_date_end` | string | `AAAA-MM-DD`, inclusive. Só a final informada ⇒ tudo até essa data. Ambas informadas ⇒ período fechado, com `emission_date_end >= emission_date_start` |
| `paginator` | `string` | cursor de paginação; omitir na 1ª página; usar o `paginator` retornado na resposta anterior, mantendo os MESMOS filtros |

### 1.2 Enum `fin_nfe` (finalidade da nota)

| Valor | Significado |
|---|---|
| `1` | Normal |
| `2` | Complementar |
| `3` | Ajuste |
| `4` | Devolução |
| `5` | Crédito |
| `6` | Débito |

### 1.3 Enum `manifestation_type` (manifestação do destinatário)

| Valor | Significado |
|---|---|
| `210200` | Confirmada |
| `210210` | Ciente |
| `210220` | Desconhecida |
| `210240` | Não realizada |

### 1.4 Enum `status` (status da nota)

| Valor | Significado |
|---|---|
| `authorized` | Autorizada |
| `canceled` | Cancelada |
| `not_authorized` | Não autorizada |
| `not_protocoled` | Não protocolada |
| `unused` | Inutilizada |
| `waiting` | Aguardando |

### 1.5 Enum `roles` (papel da conta na nota)

| Valor | Significado |
|---|---|
| `received` | Notas recebidas |
| `emitted` | Notas emitidas |
| `transporter` | Notas em que a conta é transportadora |
| `authorized` | Notas em que a conta é citada (autXML — empresa autorizada a acessar o XML) |

Comportamento do filtro `roles` (ambas as tools): a tool restringe a busca **aos papéis que o usuário
tem permissão** de buscar. Se `roles` for **omitido**, considera automaticamente todos os papéis
concedidos; se for **informado**, cada papel pedido precisa estar entre os concedidos — senão a tool
rejeita a chamada listando os papéis sem acesso. Ver §1.7.

### 1.6 Resposta de `search_nfe`

```
Response {
  nfes[]      // até 10 por página
  total       // int, total geral (todas as páginas do filtro aplicado)
  paginator   // cursor da próxima página; string vazia quando não há mais páginas
}
```

**Campos de cada `NFe`:**

| Campo | Tipo | Observação |
|---|---|---|
| `access_key` | string | chave de acesso (44 dígitos) |
| `cfops[]` | []string | CFOPs presentes na nota |
| `emission_date` | string | ISO 8601 |
| `emitter` | objeto | `{cnpj, fantasy_name, ie, name, uf}` |
| `flag_erp` | bool | `true` se a nota está integrada/sincronizada com o ERP |
| `has_cce` | bool | `true` se a nota tem Carta de Correção Eletrônica |
| `number` | — | número da nota |
| `status` | string | ver enum 1.4 |
| `receiver` | objeto | `{cnpj, fantasy_name, ie, name, uf}` |
| `origin` | string | ex.: `sefaz`, `upload` |
| `value` | string | **texto decimal**, não numérico — converter antes de somar/comparar |
| `total.ibscbs` | objeto | `{bc_ibs_cbs, ibs_uf, ibs_mun, cbs}` (floats em R$; tributos da reforma tributária; zerados quando a nota não apura IBS/CBS — fora do escopo desta skill, ver `reforma-tributaria-nfe`) |
| `products[]` | []objeto | ver tabela abaixo |

**Campos de cada `Product`:**

`item_number`, `code`, `gtin`, `description`, `ncm` (8 dígitos), `cfop`, `commercial_unit`,
`commercial_quantity`, `commercial_unit_value`, `value`, `taxable_gtin`, `taxable_unit[]`,
`taxable_quantity`, `taxable_unit_value`, `composes_total` (bool), `order_item_number`.

### 1.7 Permissão de acesso (vale para `search_nfe` e `statistics_nfe`)

Toda busca passa por uma verificação de permissão antes de retornar dados. Reflexos que a skill
precisa interpretar:

- **Sem permissão de buscar NF-e:** a tool responde com **erro** de permissão — mensagem "Você não
  tem permissão para consultar notas fiscais. Entre em contato com o administrador da sua conta para
  liberar o acesso." Não é falha técnica nem ausência de notas: o acesso não foi liberado para a conta.
  Repasse a orientação ao cliente; não tente contornar com outros filtros.
- **Papel pedido sem permissão:** se `roles` incluir um papel não concedido, a tool responde com erro
  "Você não tem acesso aos seguintes papéis: …". Refaça sem os papéis bloqueados (ou omita `roles` para
  usar só os concedidos).
- **Nenhum papel concedido:** a busca retorna **vazia** (`nfes: []`, `total: 0`) em vez de erro. Antes de
  afirmar "você não tem notas", considere que pode ser ausência de papéis liberados — não confunda com
  conta sem movimento.

## 2. `statistics_nfe`

Estatísticas/totalizações de NF-e: `count` (contar) ou `sum` (somar), com agrupamentos opcionais.

### 2.1 Filtros

Aceita **todos os filtros de `search_nfe` da seção 1.1, exceto `paginator`**, mais:

| Parâmetro | Tipo | Valores / observação |
|---|---|---|
| `emission_date_start` | string | `AAAA-MM-DD`, inclusive. Só a inicial informada ⇒ dessa data em diante. Ambas omitidas ⇒ últimos 90 dias |
| `emission_date_end` | string | `AAAA-MM-DD`, inclusive. Só a final informada ⇒ tudo até essa data. Ambas informadas ⇒ período fechado, com `emission_date_end >= emission_date_start` |
| `operation` | string | enum: `count`, `sum`. Default: `count` |
| `group_by` | []string | enum de dimensões — ver tabela 2.2. **A ordem da lista define o aninhamento das agregações (1º item = nível mais externo)** |

### 2.2 Enum `group_by` (dimensões de agrupamento)

| Valor | Significado |
|---|---|
| `role` | papel da conta na nota (`received`/`emitted`/`transporter`/`authorized`) |
| `month` | mês de emissão |
| `status` | status da nota (enum 1.4) |
| `nfe_type` | tipo da NF-e |
| `fin_nfe` | finalidade da nota (enum 1.2) |
| `state_origin` | UF de origem |
| `state_destination` | UF de destino |
| `origin` | origem do documento (ex.: `sefaz`, `upload`) |
| `cnpj` | CNPJ da conta (a que o filtro se refere) |
| `cnpj_emitter` | CNPJ do emitente |
| `cfop` | CFOP presente na nota (campo `CFOPs`, nível da nota — não do produto) |

**Atenção ao agrupar por `cfop`:** o campo é multivalorado por nota (uma NF-e pode ter itens com
CFOPs distintos). A agregação conta a nota em CADA CFOP presente nela, não uma única vez — a soma dos
`hits` de todos os buckets de CFOP pode, portanto, **superar** o `total` de notas. Isso é o
comportamento esperado, não um erro de contagem.

Não há dimensão `ncm` — é uma limitação conhecida das tools do MCP da Qive.

### 2.3 `operation=sum` — atenção

`sum` soma **apenas o campo `Value`** (valor da nota). Não existe hoje forma de somar
`total.ibscbs.cbs`, `ibs_uf`, `ibs_mun` ou `bc_ibs_cbs` via `statistics_nfe` — é uma limitação conhecida
das tools do MCP da Qive, e está fora do escopo desta skill (busca "dia a dia"). Se o cliente pedir
soma de IBS/CBS, encaminhar para a skill `reforma-tributaria-nfe`.

### 2.4 Resposta de `statistics_nfe`

```
Response {
  total          // int
  aggregation    // presente só quando group_by foi informado
}

Aggregation {
  name
  type     // ex.: "term", "monthlyhistogram", "count", "sum"
  field
  value
  buckets[]
}

Bucket {
  hits         // int — contagem (ou base da soma) daquele bucket
  value        // string — rótulo do bucket (ex.: nome do estado, mês, status)
  aggregation  // sub-agregação aninhada, presente quando group_by tem mais de um nível
}
```
