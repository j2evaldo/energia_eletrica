# Grid Intelligence — PRD

Plataforma de dados e IA da **Luz do Vale Distribuidora S.A.**, distribuidora de energia
elétrica fictícia regulada pela ANEEL. O projeto se chama **Grid Intelligence** e é
construído em 8 etapas: setup, bronze, silver (números e texto), gold, dashboard, agente
conversacional e relatório executivo.

Este documento é a fonte de contexto para todas as etapas. Leia a diretriz de simplicidade
antes de escrever qualquer código.

## Contexto de negócio

Três dores simultâneas, todas mensuráveis em dinheiro:

1. **Continuidade do fornecimento.** Interromper além do limite regulatório gera
   compensação financeira automática ao consumidor. Em 2024 as distribuidoras brasileiras
   pagaram mais de R$ 1,1 bilhão em compensações.
2. **Perdas não técnicas (PNT).** Energia distribuída e não faturada — furto, fraude e erro
   de medição. Em 2025 somaram ~R$ 11,5 bilhões, dos quais ~R$ 7,9 bilhões foram repassados
   à tarifa.
3. **Atendimento ao cliente.** Milhares de ligações viram transcrição e ninguém lê. Dentro
   delas está o aviso antecipado da reclamação na ouvidoria.

## Glossário — use estes termos, sem sinônimo aproximado

| Termo | Definição |
|---|---|
| **UC — Unidade Consumidora** | O ponto de entrega de energia. **Não é sinônimo de cliente**: um cliente pode ter várias UCs |
| **Conjunto** | Subdivisão geográfica da área de concessão. É a unidade de agregação do regulador |
| **DEC** | Duração Equivalente de Interrupção por UC — tempo médio, em horas, sem fornecimento |
| **FEC** | Frequência Equivalente de Interrupção por UC — número médio de interrupções |
| **PNT** | Perdas Não Técnicas |
| **Prioridade de inspeção** | Classificação de quanto uma UC merece visita técnica |

## Regras de negócio

- **RN-01** — Só interrupções com **3 minutos ou mais** entram na apuração de DEC e FEC.
  A regra vive na camada de transformação (silver), nunca no relatório.
- **RN-02** — O numerador do DEC é `duracao_horas × qtd_ucs_afetadas` (UC-horas
  interrompidas). É quantidade aditiva e por isso é calculada no grão do evento, na silver,
  não depois da agregação.
- **RN-03** — O numerador do FEC é `qtd_ucs_afetadas`, somado no grão do evento. O
  denominador de DEC e FEC é o total de UCs do conjunto, inclusive as não atingidas.
- **RN-04** — DEC e FEC têm **uma única definição** no sistema, consumida por dashboard,
  agente e relatório. A definição vive na metric view; a tabela gold guarda apenas os
  insumos aditivos.
- **RN-05** — DEC e FEC são apurados por mês: `mes_apuracao = date_trunc('MONTH', inicio)`.
- **RN-06** — Uma UC é candidata a inspeção quando o consumo cai em **dois eixos
  simultâneos**: contra o próprio baseline **e** enquanto a vizinhança permanece estável.
  Só o primeiro eixo apontaria o bairro inteiro depois de um apagão.
- **RN-07** — A saída da detecção é **prioridade de inspeção** — alta, média, baixa.
  **Nunca** um rótulo de fraude, furto ou culpa. Queda de consumo também é mudança de
  morador, imóvel desocupado, defeito de medidor e erro de leitura. Em setor regulado a
  diferença é jurídica. Nenhuma coluna, variável ou comentário do projeto pode usar as
  palavras "fraude", "furto" ou "culpado" como rótulo de saída.
- **RN-08** — A prioridade de inspeção é uma **regra explicável, não um modelo treinado**.
  Não existe rótulo confiável de irregularidade no dataset; qualquer pessoa tem de
  reconstruir, linha a linha, por que uma UC entrou na lista.
- **RN-09** — Dado pessoal em transcrição é **mascarado antes** de qualquer análise de
  conteúdo.
- **RN-10** — O texto original identificado é preservado com **acesso restrito**. Duas
  proteções que coexistem: uma na leitura, outra na análise.
- **RN-11** — A camada de ingestão **não filtra, não corrige e não aplica regra de
  negócio**. Leitura inválida de medidor é um fato sobre o medidor. Filtrar na ingestão
  parece limpeza; é destruição de evidência.
- **RN-12** — Todo registro ingerido carrega origem e momento de ingestão.
- **RN-13** — O agente conversacional acessa **apenas a camada de negócio** (gold).
- **RN-14** — As entidades da **camada gold** carregam descrição em linguagem de negócio —
  é o que o agente lê. Bronze e silver não precisam: são passagem.

## Diretriz de simplicidade (vale para todas as etapas)

O projeto é construído ao vivo, em aula. Código que ninguém lê na tela em dez segundos não
serve, mesmo que seja mais robusto.

- **Sem expectations e sem `CONSTRAINT`.** Regra de negócio vira `WHERE`.
- **Sem `COMMENT` em bronze e silver.** Só gold e metric view levam descrição (RN-14).
- **Sem `TBLPROPERTIES`, `CLUSTER BY` e afins**, a menos que sem eles quebre.
- **Sem tratamento de caso que o dataset não tem.**
- **Um `SELECT` direto vence um `WITH` de quatro CTEs.** Só quebre em CTE quando a query
  não couber de cabeça.
- Comentário só onde a decisão não é óbvia pelo código.
- Achando que o certo é acrescentar validação: não acrescente; diga em uma linha e o
  responsável decide.

## Convenções

- Todo texto e todo nome em **português do Brasil**; entidades, colunas e arquivos **sem
  acentuação**.
- **Nenhum nome de catálogo fixado em código.** Catálogo vem de `${var.catalog}`;
  schema vem de `${resources.schemas.<camada>.name}`.
- **Nenhum nome de modelo de IA fixado no código.** Use as funções `ai_*` do Databricks.
- Perfil da CLI: `grid_intelligence`. Todo comando leva `--profile grid_intelligence`.

## Ambientes

O catálogo isola os ambientes. Os schemas mantêm os nomes exatos `raw`, `bronze`, `silver`
e `gold` nos dois lados, para que o mesmo SQL rode sem alteração.

| Ambiente | Catálogo | Uso |
|---|---|---|
| `dev` | `grid_dev` | dia a dia |
| `prod` | `grid_intelligence` | ambiente final, depois de validado em dev |

O bundle **não** cria catálogos (recurso de conta); schemas e volume ficam no bundle.

## Modelo de informação

Quatro bases sintéticas em Parquet, quatro anos de histórico, crescimento de 30% ao ano na
carga e sazonalidade de verão no atendimento (dezembro a março recebem quase o dobro de
ligações). O dataset carrega defeitos propositais — consumo nulo, negativo, fisicamente
impossível, linhas duplicadas, interrupções abaixo de 3 minutos — para as validações e a
RN-01 terem o que fazer.

### `unidades_consumidoras` — 4.000 linhas (uma por UC)

`id_uc`, `id_cliente`, `id_conjunto`, `nome_conjunto`, `municipio`, `bairro`, `logradouro`,
`classe_consumo`, `subgrupo_tarifario`, `tensao_nominal_v`, `situacao`, `data_ligacao`,
`possui_medidor_inteligente`, `consumo_medio_kwh_dia`, `latitude`, `longitude`, `origem`,
`ingerido_em`.

### `interrupcoes` — 1.383 linhas (uma por evento de rede)

`id_interrupcao`, `id_conjunto`, `inicio`, `fim`, `duracao_minutos`, `tipo`, `causa`,
`equipamento`, `qtd_ucs_afetadas`, `origem`, `ingerido_em`.

### `consumo_diario` — ~5.854.587 linhas (uma por UC por dia)

`id_uc`, `data`, `consumo_kwh`, `horas_com_leitura`, `sinal_violacao_medidor`, `origem`,
`ingerido_em`.

### `chamados` — 2.801 linhas (uma por ligação, com transcrição)

`id_chamado`, `id_uc`, `abertura`, `canal`, `nome_solicitante`, `telefone_solicitante`,
`transcricao`, `duracao_segundos`, `origem`, `ingerido_em`.

## Arquitetura medalhão

| Camada | Papel | Regra |
|---|---|---|
| `raw.landing` | Volume com os Parquet originais | contrato por pasta, não por arquivo |
| `bronze` | Cópia fiel da origem, append-only | RN-11: não filtra, não corrige, não aplica regra |
| `silver` | Cadastro limpo, RN-01, dedup, baseline, texto enriquecido | regra de negócio vira `WHERE` |
| `gold` | Entregas de negócio e metric view | RN-14: `COMMENT` em linguagem de negócio |

## As quatro entregas de negócio (camada gold)

| Entrega | Grão | Pergunta que responde |
|---|---|---|
| `continuidade_conjunto_mes` | conjunto × mês | Estamos dentro do limite regulatório este mês? |
| `prioridade_inspecao_uc` | uma UC candidata | Quais UCs merecem visita técnica primeiro? |
| `saude_cliente` | um cliente | Quais clientes estão em rota de reclamação na ouvidoria? |
| `painel_operacional_dia` | conjunto × dia | O que aconteceu ontem e o que precisa de ação hoje? |

Além das quatro tabelas, a gold expõe a **metric view** `continuidade_metricas`, que é a
definição única de DEC e FEC (RN-04), consultada com `MEASURE()` e `GROUP BY ALL`.

## Roadmap

| # | Etapa | Entrega |
|---|---|---|
| 00 | Setup | Contexto, regras, catálogos, schemas e volume |
| 01 | Bronze | Ingestão fiel do volume, sem regra de negócio |
| 02 | Silver — números | Cadastro limpo, regra dos 3 minutos, dedup e baseline |
| 03 | Silver — texto | Anonimização e enriquecimento das transcrições |
| 04 | Gold | As quatro entregas e a metric view de DEC/FEC |
| 05 | Dashboard | Painel AI/BI sobre a gold |
| 06 | Agente | Copiloto conversacional em português |
| 07 | Relatório | Relatório executivo e as queries que provam tudo |
