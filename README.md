# Documentação Técnica e Arquitetura de Dados: Modelagem de Efluentes Sanitários - Norte Energia S.A.

Este documento apresenta o memorial descritivo completo dos scripts em Python (Jupyter Notebooks) desenvolvidos para a análise estatística, controle de qualidade (QA/QC) e modelagem hidrodinâmica das Estações de Tratamento de Esgoto (ETEs) das UHEs Belo Monte e Pimental.

O objetivo do pipeline é processar bases de dados brutas e transformá-las em produtos executivos (planilhas e gráficos) que comprovem, matematicamente e visualmente, que o impacto dos lançamentos no Rio Xingu é analiticamente indetectável, atendendo aos padrões da IFC e JGP.

> 🔗 **Repositório Oficial:** Todos os scripts originais, bases de dados de exemplo e produtos finais gerados descritos neste documento estão integralmente disponibilizados no repositório: [https://github.com/engthiago1979-blip/Efluentes_Sanit-rios](https://github.com/engthiago1979-blip/Efluentes_Sanit-rios).

---

## 1. Origem dos Dados de Entrada (Inputs)

Os scripts não inventam cenários; eles processam três matrizes de dados estritamente oficiais:

1.  **Série Histórica Laboratorial:** Laudos físicos e químicos emitidos por laboratórios credenciados, compreendendo o monitoramento das 4 ETEs (01, 02, PM e Compacta). Estes dados alimentam o motor estatístico.
2.  **Relatório Global de Sustentabilidade (GRI 303-4):** Embutido no dicionário Python (`dados_gri`), traz os volumes mensais exatos (em $m^3/mês$) de descarte de cada ETE, cobrindo a janela temporal do projeto (Jan/24 a Mar/26). São dados assegurados por auditoria externa.
3.  **Planilha de Vazão Turbinada (`vazão turbinada.xlsx`):** Lida via `pandas.read_excel`, traz os dados oficiais do Operador Nacional do Sistema (ONS), com a vazão em $m^3/s$ turbinada individualmente nas Casas de Força Principal (Belo Monte) e Complementar (Pimental).

---

## 2. Módulo de QA/QC e Tratamento de Dados Censurados

A primeira camada do script lida com a higienização da base de dados laboratorial, focando na eliminação de viés analítico em parâmetros não detectados.

### 2.1. O Tratamento Matemático ($LQ / \sqrt{2}$)
Muitos elementos (Metais, VOCs, BTEX) não existem em esgoto doméstico. O laboratório reporta esses valores como "< LQ" (ex: `< 0,01`). O código Python identifica essas strings (Expressões Regulares) e aplica a regra de substituição pelo Limite de Quantificação dividido pela raiz quadrada de 2.
* **Por que o script faz isso?** Substituir por "zero" subestimaria o risco, e substituir pelo valor integral do LQ puniria a ETE injustamente. A métrica $LQ / \sqrt{2}$ é a recomendação da US EPA para minimizar a distorção da variância (MSE) em análises estatísticas subsequentes.

### 2.2. Produtos Gerados por este Módulo:
* 📊 **`Heatmap_QAQC_Censurados.png`:** O script gera um Mapa de Calor (usando `seaborn.heatmap`). Nele, cores escuras representam 100% de censura. A função deste gráfico é provar visualmente ao auditor que a matriz é 100% livre de passivos industriais (solventes, metais pesados crônicos).
* 📄 **`Relatorio_QAQC_Dados_Censurados.xlsx`:** Uma tabela de contingência que cruza as ETEs com os parâmetros, quantificando o percentual exato de censura ao longo da série histórica.

---

## 3. Módulo de Estatística Básica e Avançada

Este módulo analisa a performance interna do tratamento de esgoto, rodando sobre a base de dados tratada no módulo QA/QC.

### 3.1. Estatística Básica (O Racional do Percentil 95)
O script calcula o Tamanho da Amostra (N), a Média e a Mediana, mas a modelagem extrai especificamente o **Percentil 95 (P95)** (`numpy.percentile(x, 95)`).
* **Por que o P95?** Em auditorias de risco socioambiental, a média aritmética é rejeitada por mascarar picos pontuais de falha. O P95 atesta a concentração limite em que a ETE opera em 95% do tempo. É o "Pior Cenário Crônico".

### 3.2. Estatística Avançada: Teste de Mann-Kendall
* **Como o script faz:** Utiliza o módulo `pymannkendall` para avaliar a série cronológica de resultados de cada ETE.
* **Por que faz:** É um teste de *Tendência Temporal Não-Paramétrico*. O objetivo é gerar um `p-value`. Se o p-value for $< 0.05$, o script acusa "Tendência de Degradação" ou "Melhora". Isso prova ao auditor se a planta está sofrendo desgaste sazonal ou mantendo a estabilidade.

### 3.3. Estatística Avançada: Kruskal-Wallis (ANOVA Não-Paramétrico)
* **Como o script faz:** Utiliza `scipy.stats.kruskal` agrupando os resultados das 4 ETEs no mesmo período.
* **Por que faz:** O script testa se as 4 ETEs pertencem à mesma "população estatística" baseada em ranqueamento. A função atesta se há uniformidade operacional entre as diferentes plantas e canteiros da Norte Energia.

### 3.4. Produto Gerado por este Módulo:
* 📄 **`Relatorio_Analitico_Mestre.xlsx`:** Planilha consolidada com múltiplas abas (`Estatistica_e_Tendencia` e `Comparativo_Kruskal_Wallis`). Confronta o P95 diretamente com o VMP da Resolução CONAMA 430/11, declarando automaticamente o *status* de "Conforme" ou "Não Conforme".

---

## 4. Módulo de Modelagem Hidrodinâmica e Balanço de Massa

Esta é a espinha dorsal do projeto. O algoritmo une o P95 extraído do módulo estatístico com os volumes (GRI) e as vazões turbinadas (ONS).

### 4.1. Lógica do Código (Parser Temporal e Left Join)
* **Super-Parser de Data:** O Excel frequentemente oculta formatos de data americanos. O script possui a função `extrair_data_segura()`, que força a leitura de `Jan/24` ou de um objeto *Datetime* do Excel para um formato Python absoluto.
* **Left Join:** No momento de fundir (`pd.merge`) o DataFrame do GRI com o DataFrame do Rio, usa-se um `how='left'`. Isso obriga o script a desenhar a linha do tempo estática de Jan/24 a Mar/26, gerando NaNs controlados caso a vazão de Mar/26 ainda não tenha sido reportada. Isso cria gráficos com espaço reservado de forma elegante, provando governança futura.

### 4.2. Segregação Geográfica (Avanço Metodológico)
Para evitar que a vazão massiva de Belo Monte mascare o cenário em Pimental, o script divide o processamento:
* **Segmento BM:** $Q_{BeloMonte}$ cruza *apenas* com ETE 01 + ETE 02.
* **Segmento PM:** $Q_{Pimental}$ (TVR) cruza *apenas* com ETE PM + Compacta.

### 4.3. Matemática do Balanço de Massa
1.  **Conversão Volumétrica:** A vazão do rio é convertida no script usando: $V_{rio\ (m^3/m\hat{e}s)} = Q_{m^3/s} \times 86400 \times 30,44$.
2.  **Fator de Diluição (FD):** $FD = V_{rio} / V_{efluente}$. Quantifica quantos milhões de litros de rio diluem 1L de efluente.
3.  **Adição de Carga Efetiva ($\Delta C$):** Calcula a concentração real adicionada à coluna d'água: $C_{adicionada\ (mg/L)} = (V_{efluente} \times Concentra\text{ç}\tilde{a}o_{P95}) / V_{rio}$.

### 4.4. Produto Gerado por este Módulo:
* 📄 **`Relatorio_Diluicao_Mestre_Segregado.xlsx`:** A exportação em Excel desta modelagem, exibindo lado a lado a vazão, o volume mensal, o Fator de Diluição e os acréscimos na sexta casa decimal ($0,0000x\ mg/L$) para DBO, Fósforo e Amônia, separados por UHE.

---

## 5. Módulo de Geração Visual (A "Bala de Prata" da Auditoria)

A biblioteca `matplotlib.pyplot` foi arquitetada com técnicas de engenharia visual cognitiva, traduzindo a planilha descrita acima em laudos gráficos irrefutáveis.

### 5.1. Gráficos de Proporção Hidrodinâmica (Ex: `Produto1A_Diluicao_BeloMonte.png`)
* **Como é gerado:** Um gráfico de barras (`ax.bar`) onde a altura é o Fator de Diluição dividido por $10^6$ (escala de milhões). A função `ax.bar_label` estampa o número exato flutuando sobre a barra.
* **Objetivo:** Provar que mesmo no pior mês de seca extrema (linha tracejada calculada pelo `min()` do array), o Xingu oferece proporções multimilionárias para a diluição do efluente sanitário.

### 5.2. Gráficos de Impacto Físico vs Risco (A "Bala de Prata Linear")
O script gera gráficos separados para DBO, Nitrogênio Amoniacal e Fósforo (Ex: `Produto2A_Impacto_DBO_BM.png`), abandonando a escala logarítmica para evidenciar visualmente a insignificância do risco.
* **O Truque do Eixo Y (`ax.set_ylim`):** O script força o limite vertical superior do gráfico a coincidir com o **Limite de Detecção do Laboratório** (ex: travado em $2,5\ mg/L$ para DBO, onde a detecção começa em $2,0\ mg/L$).
* **Linha de Efluente (`ax.plot` com `fill_between`):** Como o balanço de massa calculou valores próximos a $0,00002\ mg/L$, a linha e a área preenchida do impacto da ETE ficam perfeitamente coladas no eixo $0.0$.
* **Anotações Dinâmicas:** A função `ax.annotate` cria uma flecha apontando para o céu indicando: *"Limite CONAMA (120 mg/L) -> Fora de Escala"*. Para o Nitrogênio Amoniacal, a anotação inclui automaticamente o subtexto de isenção legal com base no §1º do Art. 21 da Resolução CONAMA 430/11.
* **Objetivo Final:** Quando o auditor visualiza o gráfico, ele percebe imediatamente que o impacto orgânico da ETE não apenas atende à lei com folga, mas é **fisicamente incapaz de ser lido por instrumentação analítica moderna**, zerando qualquer alegação de risco à ictiofauna ou risco de eutrofização no TVR e no canal de fuga.