# Catálogo de buscas — reforma tributária (IBS/CBS)

Catálogo `pergunta do cliente → chamada da tool` para os 4 focos do domínio de reforma tributária:
**Apuração IBS/CBS**, **Período de transição**, **Classificação NCM/CFOP** e **Conformidade**. Todos os
parâmetros usados aqui existem literalmente nas tools — conferir contra
`references/apuracao-ibs-cbs.md` (seção 1) e o design doc (seção 2) antes de estender esta lista.
Quando o workaround exigir paginar tudo, o aviso de custo vem sempre junto.

## Foco 1 — Apuração IBS/CBS

### "Quanto de CBS/IBS minhas notas geraram em julho?"

**WORKAROUND** (não existe forma direta — ver Armadilhas no SKILL.md):

```
search_nfe(emission_date_start="2026-07-01", emission_date_end="2026-07-31", ...)
```

Paginar com o mesmo `paginator` até esgotar, somando `total.ibscbs.cbs` (CBS) e
`total.ibscbs.ibs_uf + total.ibscbs.ibs_mun` (IBS total) de cada `nfe`. Filtrar/restringir por `cnpjs`
e/ou `roles` quando possível para reduzir o número de páginas.

**Como ler o retorno:** cada `nfe` tem `total.ibscbs.{bc_ibs_cbs, ibs_uf, ibs_mun, cbs}` em R$. Acumule
nota a nota; ignore (ou reporte à parte) notas com os quatro campos zerados — não apuram reforma.

**AVISO:** `statistics_nfe(operation="sum", ...)` **não** soma CBS/IBS — soma apenas `Value` (valor da
nota). Não existe atalho de agregação para esta pergunta hoje
— é uma limitação conhecida das tools do MCP da Qive. `emission_date_start`/`emission_date_end` em `search_nfe` já miram
julho diretamente, sem depender do default de 90 dias. Custo: paginação completa do período buscado
(10 notas/página).

### "Qual a base de cálculo do IBS/CBS das minhas notas recebidas este mês?"

**WORKAROUND:**

```
search_nfe(roles=["received"])
```

Paginar e somar `total.ibscbs.bc_ibs_cbs` de cada nota. Use `emission_date_start`/`emission_date_end`
para restringir ao mês pedido. Mesmas ressalvas de custo (escala com o período buscado) da receita
anterior.

### "Qual o IBS estadual (UF) x municipal das minhas notas emitidas?"

**WORKAROUND:**

```
search_nfe(roles=["emitted"])
```

**Como ler o retorno:** somar `total.ibscbs.ibs_uf` e `total.ibscbs.ibs_mun` **separadamente** — são
parcelas distintas, não confundir com o "IBS total" (que é a soma das duas). Mesma ressalva de custo das
receitas anteriores.

## Foco 2 — Período de transição

### "Comparar o volume de notas mês a mês na transição da reforma"

```
statistics_nfe(operation="count", group_by=["month"])
```

Adicionar `emission_date_start`/`emission_date_end` (`AAAA-MM-DD`) para restringir o intervalo, e outros
filtros (`cnpjs`, `roles`, `status`) conforme a pergunta do cliente.

**Como ler o retorno:** `aggregation.buckets[]`, cada bucket com `value` (rótulo do mês) e `hits`
(quantidade de notas naquele mês). Isso mostra **volume**, não valores de IBS/CBS.

**AVISO:** esta chamada funciona bem para contagem/volume. Para "quanto de IBS/CBS foi apurado em cada
mês", não há equivalente — cai na mesma limitação de soma de IBS/CBS via `statistics_nfe` descrita no
Foco 1 (primeira receita).

### "Quais notas ainda estão fora da reforma (zeradas) neste período?"

Ver Foco 4 (Conformidade) — mesma receita.

### "Quanto do meu volume de notas já apura IBS/CBS vs. quanto ainda não apura?"

**WORKAROUND:** não há filtro de presença de IBS/CBS em nenhuma tool — é uma limitação conhecida das
tools do MCP da Qive. Paginar `search_nfe` (com os filtros de escopo desejados) e classificar cada nota
em "apura" (algum dos quatro campos de `total.ibscbs` diferente de zero) ou "não apura" (todos zerados),
contando os dois grupos. Custo escala com o volume total de notas no período buscado — avisar o
cliente antes de rodar em contas grandes.

## Foco 3 — Classificação NCM/CFOP

### "Meus produtos do NCM 2203.00.00"

**WORKAROUND** (não existe filtro de NCM — limitação conhecida das tools):

```
search_nfe(...)  // sem filtro de NCM disponível
```

Paginar e inspecionar `products[].ncm` de cada nota, filtrando manualmente as que contêm o NCM
desejado.

**Como ler o retorno:** `products[]` é uma lista por nota; cada produto tem `ncm` (8 dígitos) e `cfop`
próprios (podem diferir do CFOP da nota). Reúna os itens cujo `ncm` bate com o procurado.

**AVISO:** custo cresce com o volume total de notas da conta no período buscado, não com a
quantidade de produtos daquele NCM.

### "Notas com CFOP de venda para consumidor final (ex.: 5.102)"

```
search_nfe(cfop=["5102"])
```

`cfop` **é** filtro nativo de `search_nfe` (no nível da nota). Ler `cfops[]` da nota retornada para
confirmar; se precisar do CFOP por item, inspecionar `products[].cfop`.

### "Totalizar minhas notas por NCM ou por CFOP" (contagem/soma agregada)

**Não é possível diretamente** — o `group_by` de `statistics_nfe` não tem dimensão `ncm` nem `cfop`,
limitação conhecida das tools. Workaround: paginar `search_nfe` inteiro e agregar `products[].ncm` /
`products[].cfop` manualmente no cliente. Caro para contas com centenas de notas.

## Foco 4 — Conformidade

### "Quais notas ainda não trazem IBS/CBS (deveriam apurar e vieram zeradas)?"

**WORKAROUND** (não existe filtro de presença de IBS/CBS — limitação conhecida das tools):

```
search_nfe(...)  // filtros de escopo conforme a pergunta (cnpjs, roles, status, etc.)
```

Paginar com o mesmo `paginator` e filtrar, no cliente, as notas em que `total.ibscbs.bc_ibs_cbs`,
`ibs_uf`, `ibs_mun` e `cbs` estão **todos** zerados.

**Como ler o retorno:** lista final de notas "sem apuração" — apresentar `access_key`, `number`,
`emission_date` de cada uma para o cliente investigar.

**AVISO:** varre todas as páginas do período buscado — custo escala com o volume total de notas, não
com a quantidade de notas "sem IBS/CBS" (que é geralmente o que interessa). Lembrar: zerado não é
necessariamente erro — pode ser esperado durante a transição (ver `apuracao-ibs-cbs.md`, seção 3).

### "Essa nota específica (chave de acesso X) já apura IBS/CBS?"

```
search_nfe(access_keys=["<44 dígitos>"])
```

Barato — não precisa paginar (uma nota só). Ler `total.ibscbs` da nota retornada; se os quatro campos
forem zero, não apura.

### "Notas canceladas que geraram IBS/CBS" (checagem de consistência)

```
search_nfe(status=["canceled"])
```

Paginar e inspecionar `total.ibscbs` das notas canceladas retornadas — notas canceladas com IBS/CBS
diferente de zero podem indicar necessidade de ajuste/estorno; reportar ao cliente para validação
fiscal, a skill não decide se é erro.
