# Referência — apuração de IBS/CBS na NF-e

Referência fiel dos campos de IBS/CBS expostos pelas tools de DFe do MCP da Qive (`search_nfe` e
`statistics_nfe`). **Nenhum campo ou enum listado aqui pode ser inventado ou estendido** — se um campo
não está nesta referência, ele não existe nas tools.

## 1. Os campos de IBS/CBS na resposta de `search_nfe`

Cada `NFe` retornada por `search_nfe` traz um objeto `total.ibscbs` com quatro campos, todos em
**reais (R$)**, todos **floats**:

| Campo | O que é |
|---|---|
| `total.ibscbs.bc_ibs_cbs` | Base de cálculo do IBS/CBS da nota |
| `total.ibscbs.ibs_uf` | IBS — parcela estadual (UF) |
| `total.ibscbs.ibs_mun` | IBS — parcela municipal |
| `total.ibscbs.cbs` | CBS — Contribuição sobre Bens e Serviços (federal) |

**Regras de leitura:**

- **IBS total da nota** = `ibs_uf + ibs_mun`. Não existe um campo único "IBS total" na resposta — sempre
  some as duas parcelas ao apresentar "quanto de IBS" para o cliente.
- **CBS** já vem como valor único em `cbs` — não precisa somar nada.
- Quando a nota **não apura** IBS/CBS (ex.: emitida antes da vigência da reforma para aquele
  contribuinte, ou fora do escopo da cobrança), os quatro campos vêm **zerados** (`0` ou `0.0`). Isso é
  esperado durante o período de transição e **não é um erro de dado** — ver seção 3.
- Esses campos existem **apenas por nota**, dentro de `search_nfe`. Não há um campo equivalente agregado
  em `statistics_nfe` (ver seção 5 — limitação importante).

## 2. Como somar por nota e por conjunto de notas

Não existe hoje nenhuma tool que devolva o total de IBS/CBS já somado para um conjunto de notas. A soma
é sempre feita **no lado do cliente**, iterando sobre as notas retornadas por `search_nfe`:

1. Chamar `search_nfe` com os filtros desejados (`cnpjs`, `roles`, `status`, etc.), usando
   `emission_date_start`/`emission_date_end` para mirar o período exato da apuração (ex.: o mês pedido
   pelo cliente) — ver seção 4.
2. Para cada `nfe` da página, ler `nfe.total.ibscbs.cbs`, `nfe.total.ibscbs.ibs_uf`,
   `nfe.total.ibscbs.ibs_mun` (e `bc_ibs_cbs` se o cliente pedir a base de cálculo).
3. Acumular os totais: `total_cbs += cbs`, `total_ibs += ibs_uf + ibs_mun`.
4. Se `paginator` vier preenchido na resposta, repetir a chamada com o mesmo `paginator` e os
   **mesmos filtros**, até `paginator` vir vazio.
5. Apresentar o total acumulado ao cliente, deixando claro o período coberto (por padrão os últimos 90
   dias se `emission_date_start`/`emission_date_end` não forem informados a `search_nfe`).

Esse é o único caminho disponível hoje. É funcional para conjuntos pequenos de notas (dezenas), mas fica
caro para contas com muito volume — ver a limitação importante na seção 5.

## 3. Período de transição — distinguir notas que já apuram das que vêm zeradas

Durante a transição da reforma tributária, é normal conviver com dois grupos de notas na mesma conta e
no mesmo período: as que já apuram IBS/CBS (campos preenchidos) e as que ainda não apuram (campos
zerados). O cliente frequentemente quer:

- **Comparar volume de notas mês a mês** — usar `statistics_nfe` com `operation="count"` e
  `group_by=["month"]` (opcionalmente combinado com `emission_date_start`/`emission_date_end` para
  restringir o intervalo, e outros filtros como `cnpjs` ou `roles`). Isso dá a contagem de notas por mês,
  útil para acompanhar a evolução do volume ao longo da transição — mas **não** dá o total de IBS/CBS por
  mês (essa soma não existe hoje via `statistics_nfe`, ver seção 4).
- **Separar notas que já apuram das que ainda não apuram** — não existe filtro dedicado para isso, é
  uma limitação conhecida das tools do MCP da Qive. O caminho disponível é paginar `search_nfe` e
  verificar, nota a nota, se `total.ibscbs.bc_ibs_cbs`, `ibs_uf`, `ibs_mun` e `cbs` estão **todos
  zerados** — se estiverem, a nota não apura reforma; caso contrário, apura (mesmo que parcialmente).

Zerado não significa erro de integração ou de dado: pode ser exatamente o comportamento esperado para
uma nota fora do escopo da cobrança de IBS/CBS naquele momento da transição. Antes de reportar
"inconsistência" ao cliente, confirme se a nota realmente deveria apurar (data de emissão, natureza da
operação) antes de tratar o zero como problema.

## 4. NCM/CFOP para enquadramento

Para entender o enquadramento fiscal de um produto (necessário para saber se/como ele é afetado pela
reforma), use os campos por produto dentro de cada `NFe`:

- `products[].ncm` — NCM do produto, 8 dígitos.
- `products[].cfop` — CFOP do produto.

Esses campos só existem dentro da resposta de `search_nfe` (dentro de `products[]` de cada nota) — não
há filtro nem dimensão de agrupamento por **NCM** em nenhuma das duas tools hoje (ver seção 5).

O `cfop`, ao contrário do NCM, tem suporte mais completo — mas sempre no **nível da nota** (campo
`CFOPs`), não do produto: (1) filtro `cfop` em `search_nfe` e `statistics_nfe`; (2) dimensão `cfop` de
`group_by` em `statistics_nfe`, que totaliza notas por CFOP. Como uma nota pode ter itens com CFOPs
distintos (campo multivalorado), `group_by=["cfop"]` conta a mesma nota em cada CFOP presente nela — a
soma dos `hits` de todos os buckets pode superar o `total` de notas do filtro, o que é esperado, não
erro. Nada disso agrupa **produtos** por CFOP — para o CFOP por item (`products[].cfop`, que pode
diferir do CFOP da nota), é preciso inspecionar nota a nota como descrito abaixo.

Para inspecionar NCM/CFOP de produtos, portanto: chamar `search_nfe` com os filtros que restringem o
universo de notas relevante (ex.: `cfop`, `roles`, `cnpjs`), paginar, e olhar `products[].ncm` /
`products[].cfop` de cada nota retornada.

## 5. Limitação importante — total de IBS/CBS por período ainda não é consolidável via `statistics_nfe`

Esta é a limitação mais relevante do domínio de reforma tributária hoje, e deve ser comunicada com
honestidade sempre que o cliente perguntar por um total agregado de IBS/CBS.

**O que se esperaria funcionar (mas não funciona):**

```
statistics_nfe(operation="sum", group_by=["month"], emission_date_start="2026-07-01", emission_date_end="2026-07-31")
```

Seria natural supor que isso soma `cbs`, `ibs_uf` ou `ibs_mun` por mês. **Não soma.** O parâmetro
`operation="sum"` do `statistics_nfe` totaliza **exclusivamente o campo `Value`** da
nota (o valor total da NF-e) — a agregação de soma é sempre sobre `Value` e não há parâmetro para trocar
o campo somado. Ou seja: `statistics_nfe` com `operation="sum"`
**nunca** soma `bc_ibs_cbs`, `ibs_uf`, `ibs_mun` ou `cbs` — independentemente dos filtros ou do
`group_by` usados.

**Consequência prática:** não existe hoje nenhuma forma de obter "o total de CBS/IBS gerado no mês" (ou
em qualquer período) diretamente de uma tool. A única alternativa é o workaround manual descrito na
seção 2: paginar `search_nfe` (10 notas por página, podendo mirar o período via
`emission_date_start`/`emission_date_end`) e somar `total.ibscbs.cbs` / `ibs_uf` / `ibs_mun` nota a
nota no lado do cliente.

**Custo do workaround:** escala linearmente com o volume de notas da conta no período. Para uma conta
com centenas ou milhares de notas por mês, isso significa dezenas a centenas de chamadas de
`search_nfe` só para responder a uma pergunta de totalização — sempre avise o cliente/usuário sobre esse
custo antes de executar o workaround em contas de alto volume, e ofereça restringir por `cnpjs`/`roles`
para reduzir o universo de notas quando possível.

**Isto é uma limitação conhecida das tools do MCP da Qive, não uma limitação da skill.** A melhoria
natural seria as tools passarem a permitir somar o campo desejado (CBS/IBS) em vez de sempre `Value`.
Enquanto isso não estiver disponível, esta referência e o catálogo de buscas (`catalogo-buscas.md`)
apontam para o workaround manual.
