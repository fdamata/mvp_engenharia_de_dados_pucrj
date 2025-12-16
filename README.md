# MVP - Ciência de Dados e Analytics - PUC-RJ
MVP da SPRINT de Engenharia de Dados da pós-graduação em Ciência de Dados e Analytcs da PUC-RJ

## Disciplina: Engenharia de Dados

**Autor:** Fabiano da Mata Almeida  
**Dataset:** Dados de potência nominal de Unidades Geradoras de Usinas despachadas pelo ONS
<br>

## 1. Introdução / Objetivo

Este projeto implementa uma solução de ingestão e persistência de dados provenientes da base de dados abertos do Operador Nacional do Sistema Elétrico (ONS), utilizando a plataforma Databricks e seguindo as melhores práticas de engenharia de dados.

O objetivo é responder algumas perguntas, quem incluem: 
- Quais os tipos de usinas e combustíveis foram desativadas após 2000? 
- Quais os tipos de usinas e combustíveis mais cresceram desde 1990? 
- Evolução das participação de combustíveis renováveis/não renováveis ao longo das décadas?
- Qual a potência média das usinas ativas e desativadas?
- Existe relação entre idade da usina e desativação? 

## 2. Preparação do Ambiente

A arquitetura utiliza o Unity Catalog do Databricks para governança e organização dos dados, com a seguinte estrutura hierárquica:

Unity Catalog

    └── Catalog: ons
        ├── Schema: staging
        └── Schema: dados

### Schemas e suas Finalidades

2.1. Schema de Staging (staging)

Propósito: Área temporária para armazenamento inicial dos dados brutos coletados da base do ONS.<br>
Natureza: Transiente e volátil.<br>
Formato: Dados brutos, sem transformações significativas.<br>
Retenção: Curto prazo (dados podem ser sobrescritos ou limpos periodicamente).<br>
Qualidade: Dados "como recebidos" da fonte original.

Responsabilidades:

Receber os dados diretamente da fonte externa (ONS).<br>
Servir como buffer para validações iniciais.<br>
Permitir reprocessamento em caso de falhas.<br>
Facilitar troubleshooting e auditoria de dados originais

Exemplo de tabelas: ons.staging.capacidade_geracao

2.2. Schema de Dados (dados)

Propósito: Camada de persistência para dados validados, transformados e prontos para consumo.

Natureza: Persistente e confiável.<br>
Formato: Dados estruturados, limpos e validados.<br>
Retenção: Longo prazo com histórico completo.<br>
Qualidade: Dados curados seguindo regras de negócio

Responsabilidades:

Armazenar dados com qualidade garantida.<br>
Manter histórico completo para análises temporais.<br>
Servir como fonte única da verdade (Single Source of Truth).<br>
Fornecer dados para dashboards, relatórios e modelos analíticos

Exemplo de tabelas: ons.dados.capacidade_geracao

![ONS - Catalogo](img/ons_catalog.png)

## 3. Coleta de Dados

Origem: bucket público do ONS (s3a://ons-aws-prod-opendata/dataset/capacidade-geracao), formato Parquet, licença CC-BY.
Script: ons2_buscadados.py.
Snippet:

        source_dir = "s3a://ons-aws-prod-opendata/dataset/capacidade-geracao"
        target_dir = "/Volumes/ons/staging/capacidade_geracao"
        files = dbutils.fs.ls(source_dir)
        for f in files:
            if f.name.endswith(".parquet"):
                dbutils.fs.cp(f.path, f"{target_dir}/{f.name}")

## 4. Importação e Modelagem

Script: ons3_lerdadosbaixados.py.
Após download dos dados em staging, realizou-se a importação. Foram realizadas transformações para padronização (trim, upper), conversão de tipos e gravação como Delta.
Snippet:

        cap_df = spark.read.format("parquet").load(f"{cap_path}/*.parquet")
        cap_df = cap_df.withColumn('id_subsistema', upper(trim(col('id_subsistema')))) \
                    .withColumn('dat_desativacao', try_to_timestamp(col('dat_desativacao'))) \
                    .withColumn('val_potenciaefetiva', col('val_potenciaefetiva').cast('double'))
        cap_df.write.format("delta").mode("overwrite").saveAsTable("ons.dados.capacidade_geracao")

![ONS - Exemplo de Dados](img/ons_sampledata.png)

## 5. Dados

### 5.1. Linhagem

O fluxo simplificado da linhagem: origem S3 → staging → DataFrame Spark → Delta Table.

O processo de ingestão e persistência segue um padrão Flat, adaptado para o contexto:

    ┌─────────────────────────┐
    │     FLUXO DE DADOS      │
    └─────────────────────────┘

    ┌─────────────────────────┐
    │         Base ONS        │  ← Fonte de Dados Externa
    └────────┬────────────────┘
             │ [1. EXTRAÇÃO]
             │ • Coleta via Download
             │ • Dados brutos
             ▼
    ┌─────────────────────────┐
    │         staging         │  ← Área de Staging
    │  ─────────────────────  │
    │  • Dados brutos         │
    │  • Formato original     │
    │  • Retenção temporária  │
    └────────┬────────────────┘
             │ [2. TRANSFORMAÇÃO E VALIDAÇÃO]
             │ • Aplicação de regras de negócio
             │ • Validação de qualidade
             ▼
    ┌─────────────────────────┐
    │           ons           │  ← Camada Confiável
    │  ─────────────────────  │
    │  • Dados curados        │
    │  • Formato padronizado  │
    │  • Qualidade garantida  │
    └────────┬────────────────┘
             │ [3. CONSUMO]
             │ • Relatórios
             │ • Analytics
             ▼
    ┌─────────────────────────┐
    │  Dashboards/Analytics   │  ← Camadas de Consumo
    └─────────────────────────┘

 A extração ocorreu em Dez/2025.

Telas da linhagem dos dados: representação gráfica e tabelas down e upstream.

 ![ONS - Linhagem - Graph](img/ons_lineage_graph.png)
 ![ONS - Linhagem - UP](img/ons_lineage_up.png)
 ![ONS - Linhagem - DOWN - Tabelas](img/ons_lineage_down_tables.png)
 ![ONS - Linhagem - DOWN](img/ons_lineage_down.png)

### 5.2. Qualidade

Os dados utilizados, obtidos de uma base pública do ONS, que é curada e mantida por um órgão oficial. Acredito que por esse motivo não foram identificados problemas relevantes de qualidade, como inconsistências, duplicidades ou valores fora de domínio. 

As verificações realizadas (estatísticas descritivas, contagem de nulos e análise de tipos) confirmaram que os atributos estão consistentes com o dicionário de dados disponibilizado pelo ONS. 

Assim, não houve necessidade de aplicar tratamentos adicionais além das conversões de tipo e ajustes de formato previstos no pipeline.

Código de verificação de qualidade

![ONS - Qualidade - 1/4](img/ons_qualidade_01.png)

Foi possível observar que todos os valores de "count" são os mesmos (5438).

![ONS - Qualidade - 2/4](img/ons_qualidade_02.png)

É possível observar que assim como consta no dicionário de dados, apenas as colunas da datas de teste e desativação apresentaram valores nulos, ou seja, não houve informação de teste e ainda constam 4472 usinas ativas.

![ONS - Qualidade - 3/4](img/ons_qualidade_03.png)

Tipos de dados tal como descrito no dicionário de dados.

![ONS - Qualidade - 4/4](img/ons_qualidade_04.png)

## 6. Evidências das etapas do ETL

### 6.1. Preparação do ambiente

![ONS - Preparação](img/ons1_preparacao.png)

### 6.2. Busca dos dados na fonte: Download para STAGING

![ONS - Buscar Dados](img/ons2_buscadados.png)

### 6.3. Persistência dos dados no UNITY CATALOG

![ONS - Unity Catalog - 1/3](img/ons3_lerdadosbaixados.1.png)
![ONS - Unity Catalog - 2/3](img/ons3_lerdadosbaixados.2.png)
![ONS - Unity Catalog - 3/3](img/ons_catalog.png)

## 7. Perguntas e Respostas (Q&A)

### ✅ Pergunta 1: Qual o perfil de usinas e combustíveis foram desativadas após 2000?

![ONS Q&A - Pergunta 1 - 1/6](img/ons4_Q&A.1.png)
![ONS Q&A - Pergunta 1 - 2/6](img/ons4_Q&A.2.png)
![ONS Q&A - Pergunta 1 - 3/6](img/ons4_Q&A.3.png)
![ONS Q&A - Pergunta 1 - 4/6](img/ons4_Q&A.4.png)
![ONS Q&A - Pergunta 1 - 5/6](img/ons4_Q&A.5.png)
![ONS Q&A - Pergunta 1 - 6/6](img/ons4_Q&A.6.png)

#### Resposta:

A partir dos anos 2000, ocorreram desativações de apenas 2 tipos de usinas. Destacam-se as TÉRMICA com 959 unidades e totalizando 6.211 MW. Tivemos ainda 3 unidades de usinas Fotovoltaica com apenas 50 MW no total. <br>
Por TIPO DE COMBUSTÍVEL, a prevalência foram as usinas a Óleo Diesel, com 772 unidades (80%), somando 2.020 MW (32%). Tivemos ainda 64 unidades (7%) a Gás, que acumularam 1.513 MW (24%). E ainda usinas a Óleo Combustível, Biomassa, Carvão e outros tipos de combustíveis.

#### Discussão:
A desativação concentra-se em usinas térmicas principalmente pelo impacto ambiental pelo uso de combustíveis fósseis.

### ✅ 2. Quais tipos de usinas e combustíveis mais cresceram após 1990?

![ONS Q&A - Pergunta 2 - 1/6](img/ons4_Q&A.7.png)
![ONS Q&A - Pergunta 2 - 2/6](img/ons4_Q&A.8.png)
![ONS Q&A - Pergunta 2 - 3/6](img/ons4_Q&A.9.png)
![ONS Q&A - Pergunta 2 - 4/6](img/ons4_Q&A.10.png)
![ONS Q&A - Pergunta 2 - 5/6](img/ons4_Q&A.11.png)
![ONS Q&A - Pergunta 2 - 6/6](img/ons4_Q&A.12.png)

#### Resposta:

Consegue-se observar que ainda houve entrada de usinas térmicas com relevante participação até os anos 2000, notadamente abastecidas por Óleo Diesel, mas que a partir dos anos 2010 prevalecem usinas com energia renovável: Eólicas e Fotovoltáica.
- Eólicas: explosão após 2010 (1.091 unidades) e 2020 (1.017).
- Fotovoltaicas: 131 (2010) e 900 (2020).
- Térmicas: pico em 2000 (958 unidades) seguida de declínio nas décadas seguintes.

#### Discussão:
Mostra mudança estrutural: renováveis crescem fortemente após 2010, enquanto térmicas dominaram a década de 2000.

### ✅ 3. Evolução da participação renovável vs não renovável por década

![ONS Q&A - Pergunta 3 - 1/3](img/ons4_Q&A.13.png)
![ONS Q&A - Pergunta 3 - 3/3](img/ons4_Q&A.14.png)
![ONS Q&A - Pergunta 3 - 3/3](img/ons4_Q&A.15.png)

#### Resposta:

A análise da inclinação da curva de capacidade dos renováveis por década revela uma tendência relevante de incremento principalmente a partir de 1960. Esse salto marca o início de um ciclo de grandes investimentos em hidrelétricas, consolidando as renováveis como base da matriz elétrica brasileira. A curva começa a se inclinar fortemente, sinalizando expansão acelerada.

Imperioso destacar a contribuição dos não renováveis entre 1990 e 2000, notadamente pelas entradas das usinas térmicas, como já supracitado.

Ainda em relação à curva de renováveis, é possível observar outro incremento líquido significativo a partir dos anos 2000. Esse “bump” coincide com a introdução das primeiras usinas eólicas e fotovoltaicas, além da continuidade das hidrelétricas. É um segundo impulso na transição energética, diversificando a matriz.


Tendência mais flat após 2000 para os não renováveis, indicando uma desaceleração nas implementações desse tipo de usina, com queda na taxa de crescimento.

#### Discussão:
A curva das renováveis mostra um padrão típico de transição energética:

- Fase inicial (1960–1980): crescimento acelerado, dominado por hidrelétricas.
- Fase de diversificação (1990–2010): entrada de novas tecnologias (eólica, solar).
- Fase de maturidade (após 2010): expansão continua, mas com inclinação menor, refletindo desafios de integração e políticas mais equilibradas.

### ✅ 4. Potência média de usinas ativas vs desativadas

![ONS Q&A - Pergunta 4 - 1/1](img/ons4_Q&A.16.png)

#### Resultado:

É possível notar que a potência média das usinas que foram desativadas é bem menor que a potência média das usinas ativas, apenas 15%,
- Ativas: média 43,6 MW.
- Desativadas: média 6,6 MW.

#### Discussão:
Unidades menores são mais vulneráveis à desativação, útil para estudos de vida útil e planejamento.

### ✅ 5. Relação idade vs probabilidade de desativação

![ONS Q&A - Pergunta 5 - 1/3](img/ons4_Q&A.17.png)
![ONS Q&A - Pergunta 5 - 2/3](img/ons4_Q&A.18.png)
![ONS Q&A - Pergunta 5 - 3/3](img/ons4_Q&A.19.png)

#### Resultado:

A anélise da relação idade da usina vs percentual de desativação com os dados agregados por faixa mostra-nos uma correlação de Pearson de −0,46 (correlação negativa moderada).

#### Discussão:

Não há evidência de que usinas de maior idade sejam mais propensas à desativação. Pelo contrário, a tendência geral é ligeiramente negativa, mas isso ocorre porque os dados são muito dispersos:

- Faixa 0–9 anos: ~1,7% desativadas
- Faixa 10–19 anos: ~13,5%
- Faixa 20–29 anos: salta para ~77,3%
- Depois disso, cai nas faixas seguintes.

Esse comportamento não é monotônico, então um modelo linear não explica bem a variação. A maior taxa está concentrada em uma faixa intermediária (20–29 anos), não nas mais velhas.

### ✅ 6. Concentração de capacidade por agente nos últimos 10 anos

![ONS Q&A - Pergunta 6 - 1/1](img/ons4_Q&A.20.png)

#### Resultado:

- Norte Energia SA: 11.233 MW.
- Jirau Energia: 2250 MW.
- Eneva: 1.882 MW.

#### Discussão:

Poucos agentes concentram grande capacidade, importante para análise de dependência e governança.

### ✅ 7. Tendência de diversificação de combustíveis nas térmicas (Índice de Shannon)

![ONS Q&A - Pergunta 7 - 1/4](img/ons4_Q&A.21.png)
![ONS Q&A - Pergunta 7 - 2/4](img/ons4_Q&A.22.png)
![ONS Q&A - Pergunta 7 - 3/4](img/ons4_Q&A.23.png)
![ONS Q&A - Pergunta 7 - 4/4](img/ons4_Q&A.24.png)

#### Resultado:

O índice de Shannon representa a diversidade do mix de combustíveis nas térmicas ao longo das décadas.

Pico de diversidade em 2010 (1,77), queda em 2020 (1,10). E nos anos 1980 o menos valor, representando a menor diversificação.

#### Discussão:

É possível visualizar:

- 1950 (0,69) → 2 combustíveis, quase equilíbrio perfeito.
- 1960 (1,31) → 4 combustíveis, bem distribuídos.
- 1980 (0,16) → diversidade mínima, dominância total.
- 2010 (1,77) → maior diversidade, mix mais equilibrado.

## 8. Discussão Geral

As análises realizadas permitem compreender tendências importantes na evolução do parque gerador brasileiro:


Desativações: Observou-se que as usinas desativadas após 2000 concentram-se em térmicas fósseis e unidades de menor porte. Esse padrão indica vulnerabilidade das plantas menos eficientes e dependentes de combustíveis não renováveis.


Expansão: A partir de 2010, fontes renováveis — especialmente eólica e solar — dominaram a expansão, enquanto as térmicas tiveram papel relevante nos anos 2000, refletindo políticas energéticas e conjunturas econômicas da época.


Sustentabilidade: A participação das renováveis apresentou queda nos anos 2000, mas recuperou-se fortemente na década seguinte, sinalizando uma mudança estrutural em direção à matriz limpa.


Perfil tecnológico: As tecnologias eólica e fotovoltaica estão fortemente associadas aos anos mais recentes, evidenciando inovação e diversificação tecnológica.


Diversidade: A diversidade de combustíveis atingiu seu ápice nos anos 2010, seguida por uma fase de consolidação, indicando amadurecimento do setor e maior previsibilidade na composição da matriz.


Esses achados reforçam a importância de políticas que incentivem fontes renováveis e a necessidade de monitorar a concentração de agentes geradores para garantir equilíbrio competitivo e segurança energética.

## 9. Autoavaliação

Este trabalho representou um desafio significativo, principalmente pelo uso do Databricks — uma ferramenta completamente nova para mim. Nunca havia tido contato com essa plataforma, o que exigiu um esforço inicial para compreender sua lógica e recursos. Além disso, a formulação das questões foi complexa, pois a base de dados era inédita e demandou exploração cuidadosa para identificar padrões relevantes.
Apesar dessas dificuldades, considero que os objetivos foram integralmente atendidos. O aprendizado adquirido sobre engenharia de dados em nuvem, manipulação e integração com ferramentas analíticas foi extremamente valioso. Em muitos momentos, senti-me como um verdadeiro bandeirante, desbravando territórios desconhecidos e construindo caminhos para análises robustas.

Para trabalhos futuros, vislumbro ampliar o escopo para incluir dados de geração horária, métricas de sustentabilidade e integração com modelos preditivos.

Em síntese, este projeto cumpriu seu papel acadêmico: proporcionou aprendizado prático, consolidou conceitos teóricos e abriu novas perspectivas para estudos avançados em gestão e planejamento energético.