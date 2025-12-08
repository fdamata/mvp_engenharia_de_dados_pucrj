# Visão Geral da Arquitetura

Este projeto implementa uma solução de ingestão e persistência de dados provenientes da base de dados abertos do Operador Nacional do Sistema Elétrico (ONS), utilizando a plataforma Databricks e seguindo as melhores práticas de engenharia de dados.

Estrutura do Unity Catalog

A arquitetura utiliza o Unity Catalog do Databricks para governança e organização dos dados, com a seguinte estrutura hierárquica:

Unity Catalog

    └── Catalog: ons
        ├── Schema: staging
        └── Schema: dados

Schemas e suas Finalidades

1. Schema de Staging (staging)

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

2. Schema de Dados (dados)

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

### Fluxo de Dados (Pipeline)

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
## Evidências das etapas do ETL

### Preparação do ambiente

![ONS - Preparação](img/ons1_preparacao.png)

### Busca dos dados na fonte: Download para STAGING

![ONS - Buscar Dados](img/ons2_buscadados.png)

### Persistência dos dados no UNITY CATALOG

![ONS - Unity Catalog - 1/2](img/ons3_lerdadosbaixados.1.png)
![ONS - Unity Catalog - 2/2](img/ons3_lerdadosbaixados.2.png)

✅ Pergunta 1: Qual o perfil de usinas e combustíveis foram desativadas após 2000?

![ONS Q&A - Pergunta 1 - 1/6](img/ons4_Q&A.1.png)
![ONS Q&A - Pergunta 1 - 2/6](img/ons4_Q&A.2.png)
![ONS Q&A - Pergunta 1 - 3/6](img/ons4_Q&A.3.png)
![ONS Q&A - Pergunta 1 - 4/6](img/ons4_Q&A.4.png)
![ONS Q&A - Pergunta 1 - 5/6](img/ons4_Q&A.5.png)
![ONS Q&A - Pergunta 1 - 6/6](img/ons4_Q&A.6.png)

#### Resposta:

Por TIPO DE USINA: TÉRMICA (959 unidades, 6.211 MW), Fotovoltaica (3 unidades, 50 MW). <br>
Por TIPO DE COMBUSTÍVEL: Óleo Diesel (772 unidades (80%), 2.020 MW (32%)), Gás (64 unidades (7%), 1.513 MW (24%)), Óleo Combustível, Biomassa, Carvão etc.

#### Discussão:
A desativação concentra-se em usinas térmicas principalmente pelo impacto ambiental pelo uso de combustíveis fósseis.

✅ 2. Quais tipos de usinas e combustíveis mais cresceram após 1990?

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

✅ 3. Evolução da participação renovável vs não renovável por década

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

✅ 4. Potência média de usinas ativas vs desativadas

![ONS Q&A - Pergunta 4 - 1/1](img/ons4_Q&A.16.png)

#### Resultado:

É possível notar que a potência média das usinas que foram desativadas é bem menor que a potência média das usinas ativas, apenas 15%,
- Ativas: média 43,6 MW.
- Desativadas: média 6,6 MW.

#### Discussão:
Unidades menores são mais vulneráveis à desativação, útil para estudos de vida útil e planejamento.

✅ 5. Relação idade vs probabilidade de desativação

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

✅ 6. Concentração de capacidade por agente nos últimos 10 anos

![ONS Q&A - Pergunta 6 - 1/1](img/ons4_Q&A.20.png)

#### Resultado:

- Norte Energia SA: 11.233 MW.
- Jirau Energia: 2250 MW.
- Eneva: 1.882 MW.

#### Discussão:

Poucos agentes concentram grande capacidade, importante para análise de dependência e governança.

✅ 7. Tendência de diversificação de combustíveis nas térmicas (Índice de Shannon)

![ONS Q&A - Pergunta 7 - 1/4](img/ons4_Q&A.21.png)
![ONS Q&A - Pergunta 7 - 2/4](img/ons4_Q&A.22.png)
![ONS Q&A - Pergunta 7 - 3/4](img/ons4_Q&A.23.png)
![ONS Q&A - Pergunta 7 - 4/4](img/ons4_Q&A.24.png)

#### Resultado:

O índice de Shannon representa a diversidade do mix de combustíveis nas térmicas ao longo das décadas.

Pico de diversidade em 2010 (1,77), queda em 2020 (1,10). E nos anos 1908 o menos valor, representando a menor diversificação.

#### Discussão:

É possível visualizar:

- 1950 (0,69) → 2 combustíveis, quase equilíbrio perfeito.
- 1960 (1,31) → 4 combustíveis, bem distribuídos.
- 1980 (0,16) → diversidade mínima, dominância total.
- 2010 (1,77) → maior diversidade, mix mais equilibrado.

#### Discussão Geral

Essas análises mostram:

- Desativações: foco em térmicas fósseis e pequenas unidades.
- Expansão: renováveis dominam após 2010; térmicas foram relevantes nos anos 2000.
- Sustentabilidade: participação renovável recupera após queda nos anos 2000.
- Perfil tecnológico: eólica e solar correlacionadas com anos recentes.
- Diversidade: maior nos anos 2010, depois consolidação.