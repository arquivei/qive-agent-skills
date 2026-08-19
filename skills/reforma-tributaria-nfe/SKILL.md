---
name: reforma-tributaria-nfe
description: |
  Cookbook para buscas de reforma tributária (IBS e CBS) sobre NF-e via as tools de DFe do MCP da
  Qive (search_nfe e statistics_nfe). Use quando o cliente pergunta sobre IBS, CBS, apuração dos novos
  tributos, base de cálculo, período de transição, evolução mês a mês da reforma, enquadramento por
  NCM/CFOP, ou conformidade dos novos campos fiscais nas notas. Para buscas operacionais sem recorte
  de reforma, use dfe-busca-nfe.
compatibility: Requer o MCP da Qive conectado (gateway mcp.qive.com.br, via OAuth), expondo as tools search_nfe e statistics_nfe.
metadata:
  author: Qive
  version: "1.0"
  mcp: qive-gateway
  dominio: reforma-tributaria
---

# Reforma tributária na NF-e (IBS/CBS)

## Papel & quando ativar

Esta skill é um **cookbook de uso + interpretação** das tools de DFe do MCP da Qive (`search_nfe` e
`statistics_nfe`) recortado para o tema de **reforma tributária**: IBS (Imposto sobre Bens e Serviços) e CBS (Contribuição
sobre Bens e Serviços). Ela **não gera código** — traduz a pergunta do cliente para a chamada certa da
tool, com os parâmetros corretos, e orienta como ler e somar o retorno.

**Ative esta skill quando o cliente mencionar:** IBS, CBS, reforma tributária, apuração dos novos
tributos, base de cálculo do IBS/CBS, período de transição, evolução mês a mês da reforma, enquadramento
por NCM/CFOP (para fins de reforma), ou conformidade dos novos campos fiscais nas notas.

**Fronteira com `dfe-busca-nfe`:** buscas e totalizações operacionais **sem** esse recorte de
reforma — busca por CNPJ/chave de acesso, devoluções, canceladas, notas emitidas vs. recebidas, totais
por estado/status/empresa, sincronização com ERP, CC-e — pertencem a `dfe-busca-nfe`. Se a pergunta do
cliente não cita IBS/CBS/reforma/transição/NCM-CFOP-para-enquadramento, prefira aquela skill.

## Os 4 focos

1. **Apuração IBS/CBS** — quanto de CBS, IBS-UF, IBS-Município ou base de cálculo as notas do cliente
   geraram, por nota ou por conjunto de notas.
2. **Período de transição** — acompanhar a evolução mês a mês (volume de notas) e distinguir notas que
   já apuram IBS/CBS das que ainda vêm zeradas.
3. **Classificação NCM/CFOP** — inspecionar o enquadramento fiscal dos produtos das notas.
4. **Conformidade** — localizar notas que deveriam apurar IBS/CBS e vieram zeradas, ou checar uma nota
   específica.

Receitas completas (pergunta → chamada da tool → como ler o retorno) para os 4 focos estão em
`references/catalogo-buscas.md`. Resumo rápido:

| Pergunta típica | Chamada | Observação |
|---|---|---|
| "Quanto de CBS gerei em julho?" | `search_nfe(...)` paginando + somar `total.ibscbs.cbs` no cliente | **Workaround** — `statistics_nfe` não soma CBS/IBS (ver Armadilhas) |
| "Comparar volume de notas mês a mês na transição" | `statistics_nfe(operation="count", group_by=["month"])` | Dá volume, não valores de IBS/CBS |
| "Meus produtos do NCM 2203.00.00" | `search_nfe(...)` paginando + inspecionar `products[].ncm` | Sem filtro de NCM na tool |
| "Quais notas ainda não trazem IBS/CBS?" | `search_nfe(...)` paginando + filtrar `total.ibscbs` todo zerado | Custo cresce com o volume total de notas |

## Como ler o retorno

- Cada `nfe` retornada por `search_nfe` traz `total.ibscbs.{bc_ibs_cbs, ibs_uf, ibs_mun, cbs}`, todos em
  R$. **IBS total da nota = `ibs_uf + ibs_mun`** (não existe campo único de "IBS total"). **CBS** é
  direto em `cbs`.
- Ao somar para um conjunto de notas, acumule nota a nota enquanto pagina (`paginator`), mantendo os
  mesmos filtros entre chamadas.
- Os quatro campos **zerados** numa nota significam que ela **não apura** IBS/CBS — isso é esperado
  durante o período de transição e **não é, por si só, um erro**. Antes de reportar "inconsistência",
  confirme se a nota realmente deveria apurar.
- `products[].ncm` (8 dígitos) e `products[].cfop` ficam dentro de cada nota, em `products[]` — inspecione
  aí para enquadramento fiscal.
- Aprofundamento completo dos campos e da lógica de soma: `references/apuracao-ibs-cbs.md`.

## Armadilhas

- **Limitação importante: não existe soma de IBS/CBS via `statistics_nfe`.** O parâmetro
  `operation="sum"` totaliza **apenas o campo `Value`** (valor total da nota), independentemente de
  `group_by` ou filtros usados — a soma de IBS/CBS ainda não está disponível no servidor. Não há como obter
  "total de CBS/IBS no mês" diretamente de nenhuma tool hoje. O único caminho é paginar `search_nfe` e
  somar `total.ibscbs.*` no cliente — sempre avise sobre o custo (muitas chamadas em contas de alto
  volume) antes de executar. É uma limitação conhecida das tools do MCP da Qive. Detalhes e o workaround
  completo em `references/apuracao-ibs-cbs.md`.
- **Janela de datas em `search_nfe`.** Por padrão, `search_nfe` usa os últimos 90 dias, MAS aceita
  `emission_date_start`/`emission_date_end` (mesma semântica do `statistics_nfe`) para mirar ou ampliar
  o período — use-os para restringir o workaround de paginação ao período exato da apuração (ex.: o mês
  pedido pelo cliente). Isso não resolve a lacuna de soma de IBS/CBS (continua manual), mas reduz o
  volume de notas a percorrer.
- **IBS/CBS zerado ≠ erro.** É o comportamento esperado para notas fora do escopo da reforma naquele
  momento da transição.
- **Sem filtro de NCM e sem `group_by` de NCM/CFOP.** Qualquer busca por NCM ou totalização por
  NCM/CFOP exige paginar `search_nfe` por completo e agregar no cliente — limitação conhecida das tools.
- **Sem filtro de presença de IBS/CBS.** Checagens de conformidade ("quais notas não apuram") exigem
  inspecionar nota a nota, também por limitação das tools disponíveis hoje.
- Paginação sempre com o **mesmo conjunto de filtros** entre chamadas, usando o `paginator` da resposta
  anterior.
- **Permissão de acesso a NF-e.** Antes de retornar dados, `search_nfe`/`statistics_nfe` verificam a
  permissão da conta. Se a conta não tiver acesso liberado, a tool responde com **erro** ("Você não tem
  permissão para consultar notas fiscais…") — repasse a orientação de procurar o administrador da conta,
  não é falha técnica. O filtro `roles` fica restrito aos papéis concedidos (papéis: `received`,
  `emitted`, `transporter`, `authorized`=citada via autXML): pedir um papel não concedido gera erro, e
  sem nenhum papel concedido a busca volta **vazia**. Ao paginar para somar IBS/CBS, um retorno vazio
  pode ser ausência de papéis liberados — não confunda com "nenhuma nota apura IBS/CBS".

## Ponteiros

- `references/apuracao-ibs-cbs.md` — deep-dive dos campos `total.ibscbs`, como somar por nota e por
  conjunto de notas, lógica do período de transição, NCM/CFOP para enquadramento, e o detalhamento
  completo dessa limitação.
- `references/catalogo-buscas.md` — catálogo de perguntas do cliente → chamada da tool, para os 4
  focos (Apuração IBS/CBS, Período de transição, Classificação NCM/CFOP, Conformidade).
