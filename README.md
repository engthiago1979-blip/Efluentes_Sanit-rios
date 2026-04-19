# Efluentes_Sanitários
Tratamento dos resultados do efluente sanitário
# EcoAnalítica NESA - Sistema de Monitoramento e Auditoria de Efluentes 🌊

![Python](https://img.shields.io/badge/python-3.8+-009CA7.svg)
![Status](https://img.shields.io/badge/Auditoria-Pronto%20para%20IFC-52BD8B.svg)
![Regulatório](https://img.shields.io/badge/Conformidade-CONAMA%20430/11-F25A3C.svg)

Sistema especializado em análise estatística avançada e auditoria regulatória para os efluentes sanitários das ETES do Complexo de Belo Monte (**ETE 01**, **ETE 02**) e Pimental (**ETE PM**, **ETE Compacta**) da **Norte Energia S.A.**

## 📋 Descrição do Projeto

Este repositório contém o pipeline automatizado para processamento de dados históricos de monitoramento ambiental. O foco principal é assegurar a robustez operacional frente aos padrões de lançamento estabelecidos pela **Resolução CONAMA 430/2011**, utilizando métricas de rigor internacional exigidas por auditores de bancos financiadores.

## 🚀 Funcionalidades Principais

* **ETL Dinâmico:** Extração e unificação de dados das abas BM e PM com normalização de nomes via Regex.
* **Tratamento de Dados Censurados:** Aplicação da regra estatística institucional para limites de quantificação ($LQ / \sqrt{2}$).
* **Auditoria de Robustez (P95):** Cálculo do **Percentil 95** e Margem de Segurança Operacional, superando a análise simples de média aritmética.
* **Estatística Avançada:** * **Mann-Kendall:** Detecção de tendências temporais (Melhora/Piora/Estabilidade).
    * **Kruskal-Wallis:** Teste de hipótese não-paramétrico para comparação de performance entre diferentes estações.
* **Relatório de QA/QC:** Mapeamento de incidência de dados censurados por parâmetro e unidade.
* **Visualização Institucional:** Geração automática de séries temporais em alta resolução (600 DPI) com identidade visual NESA.

## 📂 Estrutura de Diretórios

```text
etes_nesa/
├── base_dados/                  # Bancos de dados brutos e processados (.xlsx)
├── Efluentes_Sanitários/
│   └── produtos/                # Outputs: Gráficos PNG, Relatórios de Auditoria e QA/QC
├── scripts/                     # Scripts Python (.py ou .ipynb)
└── README.md
