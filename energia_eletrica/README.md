# Grid Intelligence

Plataforma de dados e IA da **Luz do Vale Distribuidora S.A.**, construída no Databricks
com Declarative Automation Bundles. O contexto de negócio, o glossário, as regras de negócio
(RN-01 a RN-14) e o modelo de informação estão em [`.llm/prd.md`](.llm/prd.md).

## Estrutura

```
databricks.yml                 # bundle, variavel de catalogo e targets dev/prod
resources/grid_schemas.yml     # schemas raw/bronze/silver/gold + volume raw.landing
.llm/prd.md                    # contexto que todos os prompts consultam
```

## Ambientes

O catálogo isola os ambientes; os schemas têm os mesmos nomes nos dois lados.

| Ambiente | Catálogo | Como rodar |
|---|---|---|
| `dev` | `grid_dev` | `databricks bundle deploy -t dev --profile grid_inteligence` |
| `prod` | `grid_intelligence` | `databricks bundle deploy -t prod --profile grid_inteligence` |

Catálogos **não** são criados pelo bundle. Para criá-los:

```bash
databricks experimental aitools tools query --profile grid_inteligence \
  "CREATE CATALOG IF NOT EXISTS grid_dev COMMENT 'Catalogo de desenvolvimento da Grid Intelligence'"

databricks experimental aitools tools query --profile grid_inteligence \
  "CREATE CATALOG IF NOT EXISTS grid_intelligence COMMENT 'Catalogo de producao da Grid Intelligence'"
```

## Deploy

```bash
databricks bundle validate --profile grid_inteligence
databricks bundle deploy -t dev --profile grid_inteligence
```

O deploy cria os quatro schemas (`raw`, `bronze`, `silver`, `gold`) e o volume managed
`raw.landing`. Em `mode: development`, `experimental.skip_name_prefix_for_schema: true`
impede que o nome do usuário vire prefixo dos schemas.

## Onde subir os dados

Uma pasta por base, cada uma com um único arquivo Parquet dentro:

```
/Volumes/grid_dev/raw/landing/unidades_consumidoras/
/Volumes/grid_dev/raw/landing/interrupcoes/
/Volumes/grid_dev/raw/landing/consumo_diario/
/Volumes/grid_dev/raw/landing/chamados/
```

Pelo terminal, subindo tudo de uma vez (o prefixo `dbfs:` é obrigatório mesmo para Volume):

```bash
databricks fs cp -r "/caminho/local/landing" dbfs:/Volumes/grid_dev/raw/landing \
  --overwrite --profile grid_inteligence
```

Neste repositório a pasta local é
`05 - Projeto/05 - Projeto/landing`, então o comando fica:

```bash
databricks fs cp -r "05 - Projeto/05 - Projeto/landing" dbfs:/Volumes/grid_dev/raw/landing \
  --overwrite --profile grid_inteligence
```

A pasta, e não o arquivo, é o contrato com a camada bronze: se uma base precisar ser
particionada no futuro, o pipeline não muda.

## Recomeçar do zero (DESTRUTIVO)

`CASCADE` remove schemas, tabelas, volumes e **os arquivos dentro deles**, sem pedir
confirmação duas vezes. Confirme antes de rodar.

```bash
databricks experimental aitools tools query --profile grid_inteligence \
  "DROP CATALOG IF EXISTS grid_dev CASCADE"
```

Para recriar depois: rode os dois `CREATE CATALOG` acima e `databricks bundle deploy -t dev
--profile grid_inteligence`.

## Observação sobre o perfil da CLI

O prompt do projeto usa o perfil `grid_intelligence`, mas o perfil autenticado neste
ambiente está gravado como `grid_inteligence` (sem o segundo "i"). Todos os comandos deste
README usam o nome realmente configurado em `~/.databrickscfg`. Para alinhar ao nome do
prompt, reautentique uma vez:

```bash
databricks auth login --host https://dbc-9a5431a6-5f39.cloud.databricks.com \
  --profile grid_intelligence
```
