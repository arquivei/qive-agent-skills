# Catálogo de buscas — dia a dia de NF-e

Cada linha traduz uma pergunta típica de cliente em uma chamada de tool. Os parâmetros usados aqui
existem todos em `references/filtros-e-enums.md` — nada foi inventado. Substitua os valores entre
`<>` pelos dados reais do cliente.

Convenção: CNPJ e chave de acesso sempre **apenas dígitos**, sem pontuação/barra/traço.

## Busca por documento e papel

### "Minhas notas de devolução"

```
search_nfe(fin_nfe=[4])
```
**Como ler:** cada item de `nfes[]` é uma nota com finalidade Devolução. Confira `total` para saber
quantas existem no total (pode ser mais que as 10 retornadas) e use `paginator` para seguir.

### "Notas canceladas do CNPJ X"

```
search_nfe(cnpjs=["<14 dígitos>"], status=["canceled"])
```
**Como ler:** liste `access_key`, `number`, `emission_date` e `value` de cada nota cancelada. Se
`total` > 10, pagine para trazer todas.

### "Notas que eu emiti" / "Notas que eu recebi"

```
search_nfe(roles=["emitted"])
search_nfe(roles=["received"])
```
**Como ler:** confirme o papel olhando `emitter`/`receiver` de cada nota — deve bater com a conta do
cliente no papel pedido.

### "Notas em que sou transportador"

```
search_nfe(roles=["transporter"])
```
**Como ler:** liste as notas; o CNPJ do cliente aparece como transportador, não necessariamente como
emitente/destinatário.

### "Notas com a chave de acesso X" (ou lista de chaves)

```
search_nfe(access_keys=["<44 dígitos>", "<44 dígitos>"])
```
**Como ler:** retorno traz no máximo os documentos correspondentes às chaves informadas; confira
`status` e `emission_date` de cada um.

### "Notas emitidas pelo CNPJ X" (quando X é o emitente, não a conta)

```
search_nfe(cnpjs_emitter=["<14 dígitos>"])
```
**Como ler:** útil quando o cliente quer ver notas recebidas de um fornecedor específico —
`cnpjs_emitter` filtra pelo CNPJ que emitiu, independente do papel da conta.

### "Notas com CFOP X" (ex.: devolução de venda, transferência)

```
search_nfe(cfop=["<código CFOP>"])
```
**Como ler:** `cfops[]` de cada nota confirma o(s) CFOP(s) presentes; uma nota pode ter mais de um CFOP
por conter produtos com CFOPs diferentes.

### "Notas com finalidade complementar / de ajuste / de crédito / de débito"

```
search_nfe(fin_nfe=[2])   # Complementar
search_nfe(fin_nfe=[3])   # Ajuste
search_nfe(fin_nfe=[5])   # Crédito
search_nfe(fin_nfe=[6])   # Débito
```
Pode combinar mais de um valor na lista, ex. `fin_nfe=[5,6]` para crédito e débito juntos.

## Filtro por data

### "Notas emitidas entre duas datas" (ex.: "de 1º a 15 de junho")

```
search_nfe(emission_date_start="AAAA-MM-DD", emission_date_end="AAAA-MM-DD")
```

**Como ler:** `emission_date_start`/`emission_date_end` são inclusivas (`AAAA-MM-DD`); informar só a
inicial busca dessa data em diante, só a final busca até essa data, e omitir as duas usa o default de
últimos 90 dias. Combine com outros filtros (`cnpjs`, `roles`, `status`, etc.) normalmente. Confira
`emission_date` de cada nota retornada para validar que está dentro do período pedido, e use
`paginator` se `total` for maior que 10.

## Status e manifestação

### "Notas aguardando processamento / inutilizadas / não autorizadas"

```
search_nfe(status=["waiting"])
search_nfe(status=["unused"])
search_nfe(status=["not_authorized"])
```
**Como ler:** confira `status` de cada nota retornada bate com o filtro pedido.

### "Notas confirmadas (manifestação do destinatário)"

```
search_nfe(manifestation_type=["210200"])
```
**Como ler:** `210200` = Confirmada. Para "notas cientes" use `210210`; para "desconhecidas", `210220`;
para "não realizadas", `210240`.

### "Notas que ainda não manifestei" (aproximação)

Não existe filtro de "sem manifestação" — o cliente precisa comparar as notas recebidas
(`roles=["received"]`) com as que já têm algum `manifestation_type` registrado. Workaround: buscar
`search_nfe(roles=["received"])` e, para cada nota, verificar se algum evento de manifestação já foi
registrado (fora do escopo direto de `search_nfe`; sinalizar ao cliente que essa comparação é manual).
O filtro `event_type` de `search_nfe` é real e pode ajudar a restringir a busca por tipo de evento
específico, mas não substitui a comparação manual — não existe um filtro pronto de "sem manifestação".

## Sincronização e CC-e

### "Notas com Carta de Correção (CC-e)"

Não existe filtro direto de `has_cce` em `search_nfe` — é só campo de resposta, limitação conhecida das
tools do MCP da Qive. Workaround: paginar o recorte relevante (ex.: por CNPJ ou período recente) e
checar `has_cce == true` nota a nota. Se o volume esperado for grande, avisar o cliente do custo
(várias chamadas).

```
search_nfe(cnpjs=["<14 dígitos>"])
# inspecionar has_cce em cada item de nfes[]
```

### "Notas não sincronizadas com o ERP"

Mesma limitação: `flag_erp` só existe na resposta, sem filtro. Workaround idêntico ao de CC-e —
paginar e checar `flag_erp == false`.

```
search_nfe(cnpjs=["<14 dígitos>"])
# inspecionar flag_erp em cada item de nfes[]
```

## Contagens e totais (`statistics_nfe`)

### "Quantas notas por estado de origem?"

```
statistics_nfe(operation="count", group_by=["state_origin"])
```
**Como ler:** cada `bucket` em `aggregation.buckets[]` representa uma UF; `bucket.value` é a UF e
`bucket.hits` é a contagem de notas.

### "Quantas notas por status?"

```
statistics_nfe(operation="count", group_by=["status"])
```
**Como ler:** um bucket por status (`authorized`, `canceled`, etc.); `hits` = quantidade em cada um.

### "Quantas notas por mês?"

```
statistics_nfe(operation="count", group_by=["month"])
```
**Como ler:** um bucket por mês; útil para ver evolução de volume ao longo do tempo. Combine com
`emission_date_start`/`emission_date_end` para restringir o período.

### "Quantas notas por CNPJ emitente?"

```
statistics_nfe(operation="count", group_by=["cnpj_emitter"])
```
**Como ler:** um bucket por CNPJ emitente; útil para ranking de fornecedores por volume de notas.

### "Quantas notas por CFOP?" / "Totalizar por tipo de operação (CFOP)"

```
statistics_nfe(operation="count", group_by=["cfop"])
```
**Como ler:** um bucket por CFOP presente nas notas; `bucket.value` é o código CFOP e `bucket.hits` a
contagem. **Atenção:** o CFOP é multivalorado por nota — uma NF-e com produtos de CFOPs diferentes é
contabilizada em cada bucket correspondente, então a soma de todos os `hits` pode ultrapassar o `total`
de notas do filtro. Isso é esperado, não um erro. Para restringir a contagem a um CFOP específico em vez
de agrupar por todos, use o filtro `cfop=["<código>"]` (ver `search_nfe(cfop=[...])` acima) combinado com
`operation="count"` sem `group_by`.

### "Valor total emitido por empresa no mês"

```
statistics_nfe(
  operation="sum",
  group_by=["cnpj"],
  emission_date_start="AAAA-MM-01",
  emission_date_end="AAAA-MM-DD"
)
```
**Como ler:** cada `bucket.value` é o CNPJ e `bucket.hits` é a CONTAGEM de notas daquele CNPJ (não a
soma). O valor somado fica no campo `value` (float) da `aggregation` daquele bucket — confirme o
campo exato na resposta real da tool, pois o nome pode variar. Ajuste `emission_date_end` para o
último dia do mês (considerando o mês corrente, use a data atual se o mês ainda não terminou).

### "Total de notas por status e por mês" (agrupamento aninhado)

```
statistics_nfe(operation="count", group_by=["month", "status"])
```
**Como ler:** a ordem de `group_by` define o aninhamento — aqui, cada bucket de `month` (nível
externo) tem uma `aggregation` interna com buckets de `status`. Percorra recursivamente
`bucket.aggregation.buckets[]` para chegar ao segundo nível.

### "Quanto vendi (valor total) no período X a Y?"

```
statistics_nfe(
  operation="sum",
  roles=["emitted"],
  emission_date_start="AAAA-MM-DD",
  emission_date_end="AAAA-MM-DD"
)
```
**Como ler:** sem `group_by`, a resposta traz só `total` (contagem) — o valor somado aparece dentro de
`aggregation` apenas quando há pelo menos um `group_by`. Se o cliente quer só o total agregado sem
quebra, adicione um `group_by` "neutro" (ex.: `role`) para obter o valor somado em `aggregation`, ou
comunique isso ao cliente: o campo de somatório direto é o `value` (float) do nó de `aggregation`
(nunca `bucket.hits`, que é sempre contagem) — confirme o campo exato na resposta real da tool, pois
o nome pode variar.

### "Quantas notas de devolução recebi este mês?"

```
statistics_nfe(
  operation="count",
  fin_nfe=[4],
  roles=["received"],
  emission_date_start="AAAA-MM-01",
  emission_date_end="AAAA-MM-DD"
)
```
**Como ler:** sem `group_by`, o número está direto em `total`.

## Paginação

### "Próxima página" / "mostra mais resultados"

Repetir a **mesma chamada anterior**, adicionando `paginator="<valor do campo paginator retornado>"`
e mantendo **exatamente os mesmos filtros** da chamada original (CNPJs, status, roles, etc. — mudar
qualquer filtro invalida o cursor).

```
search_nfe(<mesmos filtros da chamada anterior>, paginator="<paginator retornado>")
```
**Como ler:** repita até `paginator` voltar vazio — isso indica que não há mais páginas.

### "Quero ver todas as notas de devolução deste CNPJ" (paginação completa)

```
search_nfe(cnpjs=["<14 dígitos>"], fin_nfe=[4])
# repetir com paginator="<retornado>" até paginator vir vazio
```
**Como ler:** acumule `nfes[]` de cada página; `total` informado na 1ª resposta já indica quantas
notas existem no total, então dá para estimar quantas chamadas serão necessárias (`total / 10`,
arredondando para cima).

## Armadilhas rápidas específicas destas receitas

- `search_nfe` aceita `emission_date_start`/`emission_date_end` (`AAAA-MM-DD`, inclusive, mesma
  semântica do `statistics_nfe`) para mirar ou ampliar o período buscado — 90 dias é só o default
  quando nenhuma das duas é informada. Veja a receita "Notas emitidas entre duas datas" abaixo.
- Não existe filtro por NCM em `search_nfe` nem dimensão `ncm` em `group_by` — limitação conhecida das
  tools do MCP da Qive. (`cfop` **já é** uma dimensão válida de `group_by`, ver receita acima.)
- Não existe filtro de faixa de valor (`value_min`/`value_max`) — mesma limitação.
