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

Fluxo de Dados (Pipeline)

O processo de ingestão e persistência segue um padrão Flat, adaptado para o contexto:

┌─────────────────────────────────────────────────────────────────┐
│                         FLUXO DE DADOS                          │
└─────────────────────────────────────────────────────────────────┘

    ┌─────────────────┐
    │     Base ONS    │  ← Fonte de Dados Externa
    └────────┬────────┘
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
             │ • Enriquecimento (metadados)
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
             │ • Dashboards
             │ • Relatórios
             │ • Analytics
             ▼
    ┌─────────────────────────┐
    │  Dashboards/Analytics   │  ← Camadas de Consumo
    └─────────────────────────┘
