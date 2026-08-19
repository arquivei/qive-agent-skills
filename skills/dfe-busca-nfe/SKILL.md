---
name: dfe-busca-nfe
description: |
  Cookbook para buscar e totalizar NF-e via as tools de DFe do MCP da Qive (search_nfe e
  statistics_nfe) no dia a dia. Use quando o cliente quer buscar notas fiscais, filtrar por CNPJ, chave
  de acesso, papel, status, finalidade ou manifestação, contar/somar notas, ver devoluções, canceladas,
  sincronização com ERP, CC-e, ou paginar resultados. Não é sobre IBS/CBS/reforma tributária (use
  reforma-tributaria-nfe).
compatibility: Requer o MCP da Qive conectado (gateway mcp.qive.com.br, via OAuth), expondo as tools search_nfe e statistics_nfe.
metadata:
  author: Qive
  version: "1.0"
  mcp: qive-gateway
  dominio: dfe-nfe
---

# Busca de NF-e (dia a dia)

## Papel & quando ativar

Esta skill é o cookbook de **buscas e totalizações operacionais rotineiras** de NF-e via as tools do MCP da Qive.
Ela traduz a pergunta do cliente em linguagem natural para a chamada correta de `search_nfe` ou
`statistics_nfe` (com os parâmetros certos) e orienta a leitura/resumo do retorno para o cliente. Ela
**não gera código** e **não é um cliente do MCP** — apenas orienta qual tool chamar, com quais
parâmetros, e como interpretar a resposta.

Ative esta skill para pedidos como: buscar notas por CNPJ, chave de acesso, papel (emitida/recebida/
transportador/autorizada — `authorized` = conta citada na nota via autXML), status, finalidade
(devolução, complementar, ajuste, crédito, débito), manifestação do
destinatário; contar ou somar notas por estado, status, mês, empresa; verificar sincronização com ERP
(`flag_erp`) ou presença de Carta de Correção (`has_cce`); paginar resultados.

**Fronteira com `reforma-tributaria-nfe`:** se a pergunta menciona IBS, CBS, reforma tributária,
apuração dos novos tributos, período de transição (2026+), ou pede para somar/analisar
`total.ibscbs` (bc_ibs_cbs, ibs_uf, ibs_mun, cbs), isso pertence à skill `reforma-tributaria-nfe` —
não a esta. Buscas/contagens/somas "operacionais" sobre `value`, status, papel, CNPJ, CFOP, etc.,
sem esse recorte tributário, pertencem a esta skill.

## Receitas

Tabela resumida — catálogo completo e mais receitas em `references/catalogo-buscas.md`.

| Pergunta do cliente | Chamada |
|---|---|
| "Minhas notas de devolução" | `search_nfe(fin_nfe=[4])` |
| "Notas canceladas do CNPJ X" | `search_nfe(cnpjs=["<14 dígitos>"], status=["canceled"])` |
| "Notas que emiti" | `search_nfe(roles=["emitted"])` |
| "Notas que recebi" | `search_nfe(roles=["received"])` |
| "Notas confirmadas (manifestação)" | `search_nfe(manifestation_type=["210200"])` |
| "Quantas notas por estado de origem?" | `statistics_nfe(operation="count", group_by=["state_origin"])` |
| "Valor total emitido por empresa no mês" | `statistics_nfe(operation="sum", group_by=["cnpj"], emission_date_start="AAAA-MM-01", emission_date_end="AAAA-MM-DD")` |
| "Próxima página" | repetir a chamada anterior com `paginator="<paginator retornado>"` e os MESMOS filtros |

Para receitas por chave de acesso, CFOP, CNPJ emitente, contagem aninhada (mês + status), CC-e, ERP,
e todo o resto do dia a dia, consulte `references/catalogo-buscas.md`.

## Como ler o retorno

**`search_nfe`:** cada item de `nfes[]` é uma nota. Para responder ao cliente, monte um resumo em
texto ou tabela com os campos relevantes à pergunta (ex.: `number`, `emission_date`, `status`,
`value`, `emitter.name`/`receiver.name`). Use `total` para informar quantas notas existem no total
(pode ser maior que as 10 retornadas) e `paginator` para saber se há mais páginas (vazio = acabou).

**`statistics_nfe`:**
- Sem `group_by`: além de `total` (contagem geral do filtro), a resposta traz um nó `aggregation` que
  reflete a operação — `{name:"Contagem", type:"count", value:<n>}` para `operation=count`, ou
  `{name:"Valor Total", type:"sum", field:"Value", value:<soma>}` para `operation=sum`. Use o `value`
  desse nó como o número a reportar.
- Com `group_by`: percorra `aggregation.buckets[]`. Cada `bucket.value` é o rótulo do agrupamento
  (ex.: UF, mês, status, CNPJ) e `bucket.hits` é sempre a CONTAGEM de NF-e no bucket — mesmo quando
  `operation=sum`. Nesse caso, o valor somado fica em `bucket.aggregation.value` (a sub-agregação, com
  `field:"Value"`), não em `bucket.hits`.
- `group_by` com mais de uma dimensão gera aninhamento: o 1º item da lista é o nível externo; cada
  `bucket` tem uma `aggregation` interna com os buckets do próximo nível — percorra recursivamente
  (`bucket.aggregation.buckets[]`) até chegar à última dimensão.
- Para apresentar ao cliente, converta os buckets numa tabela simples (rótulo → valor), ordenando pelo
  que fizer mais sentido para a pergunta (maior `hits` primeiro, ordem cronológica para `month`, etc.).

## Armadilhas

- **Janela de datas em `search_nfe`:** por padrão, `search_nfe` usa os últimos 90 dias, MAS aceita
  `emission_date_start`/`emission_date_end` (mesma semântica do `statistics_nfe` — `AAAA-MM-DD`,
  inclusive) para mirar ou ampliar o período. Use esses parâmetros diretamente em `search_nfe` quando o
  cliente pedir um recorte de data específico; não é mais necessário recorrer a `statistics_nfe` só por
  causa da data.
- **Paginação por cursor exige os MESMOS filtros:** ao usar `paginator` numa próxima chamada, repita
  exatamente os filtros da chamada original. Mudar um filtro no meio da paginação invalida o cursor e
  pode gerar resultados inconsistentes.
- **`value` é texto decimal, não número:** o campo `value` de cada nota vem como string (ex.: "1234.56").
  Converta antes de somar/comparar no lado do cliente.
- **`operation=sum` soma só o campo `Value` da nota:** não existe forma de somar `total.ibscbs.cbs`,
  `ibs_uf`, `ibs_mun` ou `bc_ibs_cbs` via `statistics_nfe` hoje — é uma limitação conhecida das tools do
  MCP da Qive, e pertence ao escopo da skill `reforma-tributaria-nfe`.
- **CNPJ e chave de acesso são só dígitos:** sem pontuação, barra ou traço, tanto em `cnpjs`,
  `cnpjs_emitter` quanto em `access_keys`.
- **Permissão de acesso a NF-e (both tools):** toda busca passa por verificação de permissão. Três
  respostas a saber interpretar: (1) **erro de permissão** ("Você não tem permissão para consultar notas
  fiscais…") = a conta não tem acesso liberado — repasse a orientação de procurar o administrador da
  conta, não é falha técnica; (2) **erro de papel** ("Você não tem acesso aos seguintes papéis: …") =
  algum valor pedido em `roles` não foi concedido — refaça sem ele ou omita `roles`; (3) **retorno
  vazio** (`total: 0`) pode ser ausência de papéis concedidos, não necessariamente conta sem notas.
  Detalhes em `references/filtros-e-enums.md` §1.7.
- **Casos que exigem varrer tudo (custoso):** filtros que não existem hoje — CC-e (`has_cce`),
  sincronização com ERP (`flag_erp`), NCM, faixa de valor, ou agrupamento por NCM/CFOP — só podem ser
  respondidos paginando `search_nfe` por completo e filtrando no cliente. São limitações conhecidas das
  tools do MCP da Qive. Avise o cliente do custo (várias chamadas) quando o volume esperado for grande.

## Ponteiros

- `references/filtros-e-enums.md` — referência completa de parâmetros, enums e schemas de resposta
  das duas tools.
- `references/catalogo-buscas.md` — catálogo extenso de perguntas do dia a dia → chamada da tool,
  com a leitura do retorno para cada uma.
