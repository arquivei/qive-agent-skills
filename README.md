# Qive Agent Skills

Repositório **central de skills e agents da Qive** para os nossos **MCPs (Model Context Protocol)**.

Um MCP expõe *tools* (funções que o Claude pode chamar). As **skills** e **agents** deste repositório
são a camada de inteligência por cima dessas tools: transformam ferramentas cruas em uma experiência
em **linguagem natural**, traduzindo a pergunta do usuário na chamada certa (com os parâmetros e
filtros corretos), interpretando o retorno e roteando entre domínios — sem você precisar conhecer os
detalhes técnicos de cada tool.

> **Para quem é:** clientes e equipes da Qive que usam os MCPs da Qive no Claude
> (Claude.ai, Claude Code ou API).

- 🧠 **Inteligência sobre as tools** — a skill sabe qual tool usar, quais filtros aplicar e como
  resumir o resultado.
- 🗣️ **Linguagem natural** — você pergunta como falaria com um analista; o agent faz o resto.
- 🧩 **Central e extensível** — um só lugar para as skills/agents de todos os MCPs da Qive; novos
  domínios entram seguindo o mesmo padrão.

---

## Domínios disponíveis

| Domínio | MCP | O que cobre | Status |
|---------|-----|-------------|--------|
| **DFe (NF-e)** | DFe MCP | Busca e totalização de notas fiscais eletrônicas; reforma tributária (IBS/CBS) | ✅ Disponível |
| _(próximos)_ | _outros MCPs da Qive_ | _à medida que novos MCPs surgirem_ | 🔜 Planejado |

Cada domínio traz um ou mais **agents** (roteadores especialistas) e **skills** (os "livros de
receita" que ensinam a montar cada busca). O repositório é pensado para crescer: novos MCPs ganham
suas próprias skills/agents aqui, com a mesma experiência de linguagem natural.

---

## Domínio DFe (NF-e) — o primeiro disponível

Converse em linguagem natural sobre suas **notas fiscais eletrônicas (NF-e)** e sobre a
**reforma tributária (IBS/CBS)**.

### O que você pode perguntar

Dia a dia de NF-e:

- "Quais foram minhas **notas de devolução** dos últimos dias?"
- "Me mostra as **notas canceladas** do CNPJ 12345678000199."
- "**Quantas notas** eu tenho por estado de origem?"
- "**Valor total** emitido por empresa em julho."
- "Notas emitidas **entre 01/06 e 15/06**."

Reforma tributária (IBS/CBS):

- "**Quanto de CBS** minhas notas geraram em julho?"
- "Quais notas **ainda não trazem IBS/CBS**?" (conformidade)
- "Meus produtos com **NCM 2203.00.00**."
- "Compara o **volume de notas mês a mês** na transição."

### Agents e skills deste domínio

| Tipo | Nome | Para quê |
|------|------|----------|
| 🤖 Agent | `especialista-nfe` | Buscas e totalizações de NF-e no dia a dia |
| 🤖 Agent | `especialista-reforma-tributaria` | IBS/CBS, apuração, período de transição, NCM/CFOP, conformidade |
| 📘 Skill | `dfe-busca-nfe` | Receitas de busca/totalização de NF-e |
| 📘 Skill | `reforma-tributaria-nfe` | Receitas de reforma tributária (IBS/CBS) |

Os dois agents se complementam: pergunte sobre IBS/CBS ao especialista de NF-e e ele encaminha para o
de reforma tributária, e vice-versa.

### As tools do DFe MCP por trás

- **`search_nfe`** — busca NF-e com filtros (CNPJ, chave de acesso, papel, status, finalidade,
  manifestação, CFOP, período de emissão). Retorna as notas em páginas de 10.
- **`statistics_nfe`** — conta (`count`) ou soma (`sum`) notas, com agrupamentos (por mês, papel,
  status, estado, CNPJ, etc.).

> **Reforma tributária hoje:** as NF-e já trazem os campos de IBS/CBS (base de cálculo, IBS estadual,
> IBS municipal e CBS) e as skills ajudam a buscar, somar e conferir esses valores, acompanhar o
> período de transição e checar conformidade. Algumas consolidações mais pesadas (ex.: total de IBS/CBS
> somado por período) ainda dependem de melhorias em andamento nas tools.

---

## Pré-requisitos

1. Um cliente Claude: **Claude.ai** (plano pago), **Claude Code** ou a **API**.
2. Acesso ao **MCP da Qive** (gateway `https://mcp.qive.com.br/mcp`, autenticação via **OAuth**) — é ele
   que expõe as tools usadas pelas skills (para o domínio DFe, `search_nfe` e `statistics_nfe`). No
   **Claude Code**, o próprio plugin já registra esse MCP (veja abaixo) e o login OAuth é feito na
   primeira conexão. Fale com a Qive (contato@qive.com.br) se ainda não tiver acesso.

---

## Como instalar e usar

### Claude Code (plugin)

Este repositório é um **marketplace de plugins** do Claude Code (`qive-agent-skills`) com o plugin
`qive-skills`, que reúne todas as skills e agents da Qive **e já registra o MCP da Qive** (gateway
`mcp.qive.com.br`) — ou seja, instalar o plugin entrega skills + agents + a conexão com o MCP num passo só.

1. Adicione o marketplace (por caminho local, após clonar, ou pela URL do repositório):
   ```
   /plugin marketplace add ./qive-agent-skills
   ```
2. Instale o plugin:
   ```
   /plugin install qive-skills@qive-agent-skills
   ```
3. Na primeira busca, o Claude Code abre o **login OAuth** do MCP da Qive (nenhum token/header manual).
4. Depois é só pedir em linguagem natural, por exemplo:
   > "Usando o especialista de reforma tributária, me diz quanto de CBS minhas notas geraram em julho."

### Claude.ai

Suba as skills da pasta `skills/` como skills personalizadas seguindo
[Usando skills no Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude).
Com o MCP do domínio conectado, basta conversar normalmente.

### API

Skills personalizadas também podem ser usadas via API — veja o
[Skills API Quickstart](https://docs.claude.com/en/api/skills-guide#creating-a-skill).

---

## Estrutura do repositório

```
.
├── agents/                         # Agents especialistas (roteadores), por domínio
│   ├── especialista-nfe.md
│   └── especialista-reforma-tributaria.md
├── skills/                         # Skills (livros de receita), por domínio
│   ├── dfe-busca-nfe/
│   │   ├── SKILL.md
│   │   └── references/             # Referência de filtros/enums + catálogo de buscas
│   └── reforma-tributaria-nfe/
│       ├── SKILL.md
│       └── references/             # Deep-dive IBS/CBS + catálogo de buscas
└── .claude-plugin/                 # Manifests do plugin/marketplace
    ├── marketplace.json
    └── plugin.json
```

À medida que novos MCPs entram, seus agents/skills seguem a mesma convenção (nomeados pelo domínio) e
são registrados no `.claude-plugin/plugin.json`.

---

## Contribuindo (equipe Qive)

Para adicionar inteligência a um novo MCP (ou ampliar um domínio existente):

1. **Skill** — crie uma pasta em `skills/<nome>/` com um `SKILL.md`. É o "livro de receita": mapeia a
   pergunta do usuário → chamada da tool (parâmetros corretos) → como ler/resumir o retorno.
   ```markdown
   ---
   name: nome-da-skill
   description: O que a skill faz e quando o Claude deve usá-la (inclua frases-gatilho em PT-BR).
   ---

   # Título da skill

   Receitas (pergunta do usuário → chamada da tool) e orientações de interpretação.
   ```
   Só `name` e `description` são obrigatórios no frontmatter.

2. **Agent** — crie um roteador em `agents/<nome>.md` (frontmatter com `description` + `skills:` que
   ele orquestra), que identifica a intenção do usuário e invoca a skill certa.

3. **Registro** — adicione a skill/agent ao `.claude-plugin/plugin.json`.

4. **Feedback para o MCP** — se uma skill precisa de muitas chamadas para consolidar uma resposta
   comum, isso costuma indicar uma lacuna de filtro na tool. Registre a oportunidade e leve para o time
   do MCP responsável.

Padrão completo de Agent Skills: [especificação oficial](https://agentskills.io/specification).

---

## Suporte

Dúvidas ou acesso aos MCPs da Qive: **contato@qive.com.br**.
