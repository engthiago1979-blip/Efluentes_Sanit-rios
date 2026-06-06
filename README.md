# Documentação Técnica e Arquitetura de Dados: Modelagem de Efluentes Sanitários — Norte Energia S.A.

Este documento descreve o pipeline em Python (Jupyter Notebooks) para análise estatística, controle de qualidade (QA/QC) e modelagem hidrodinâmica das Estações de Tratamento de Esgoto (ETEs) das UHEs Belo Monte e Pimental.

O pipeline processa bases brutas de monitoramento e gera produtos executivos (planilhas e gráficos) que avaliam, de forma estatística e transparente, o impacto dos lançamentos de efluente sanitário no Rio Xingu frente aos padrões da Resolução CONAMA 430/11.

> 🔗 **Repositório Oficial:** [https://github.com/engthiago1979-blip/Efluentes_Sanit-rios](https://github.com/engthiago1979-blip/Efluentes_Sanit-rios)

> 🧩 **Skill associada:** O desenvolvimento segue a skill `analista-python-ambiental` (Claude Code) — Analista Ambiental + Desenvolvedor Python Sênior, padronizando stack, QA/QC, identidade visual Norte Energia e rastreabilidade regulatória.

---

## 0. Histórico de Versões

| Versão | Notebook | Principais mudanças |
|---|---|---|
| **v1** | `pipeline_etes_nesa.ipynb`, `analise_estatistica_nesa.ipynb` | Versão original: ETL/QA-QC, P95, Mann-Kendall, Kruskal-Wallis, balanço de massa e gráficos. |
| **v2** | `pipeline_etes_nesa_v2.ipynb`, `analise_estatistica_nesa_v2.ipynb` | Correções de engenharia: caminhos via `pathlib`, fim da fragmentação de DataFrame, supressão de warnings direcionada, *fallback* de fonte. Gráficos redesenhados (data storytelling). Meses em PT-BR, acentuação e unidade `°C`. |
| **v2.1** | `analise_estatistica_nesa_v2.ipynb` (cél. 3) | **P95 automatizado e segmentado por UHE** (BM = ETE 01+02; PM = ETE PM+Compacta), substituindo os valores fixos ("hardcoded"). |
| **v3** | `analise_estatistica_nesa_v3.ipynb` | **4 camadas analíticas novas**: pós-hoc de Dunn, detecção de ruptura (`ruptures`), validação de schema (`pandera`) e outliers por IQR. VMPs marcados como *"conferir na base"*. |

> Os notebooks v1 são mantidos por referência histórica. **O fluxo recomendado é: `pipeline_etes_nesa_v2.ipynb` → `analise_estatistica_nesa_v3.ipynb`.**

---

## 1. Origem dos Dados de Entrada (Inputs)

1.  **Série Histórica Laboratorial:** laudos físico-químicos das 4 ETEs (01, 02, PM e Compacta). Lida pelo `pipeline_etes_nesa_v2.ipynb` a partir do arquivo bruto em `base_dados/`.
2.  **Relatório Global de Sustentabilidade (GRI 303-4):** volumes mensais de descarte (m³/mês) por ETE, Jan/24 a Mar/26, embutidos no dicionário `dados_gri`. *(Atualmente digitados no código; ver "Limitações conhecidas".)*
3.  **Planilha de Vazão Turbinada (`vazão turbinada.xlsx`):** vazões oficiais do ONS (m³/s) por Casa de Força (Belo Monte e Pimental).

---

## 2. Módulo de QA/QC e Tratamento de Dados Censurados

### 2.1. Tratamento Matemático ($LQ / \sqrt{2}$)
Valores reportados como "< LQ" (ex.: `< 0,01`) são identificados por expressão regular e substituídos pelo Limite de Quantificação dividido por $\sqrt{2}$ — recomendação da US EPA para minimizar a distorção da variância. **Nunca se converte para zero.** Cada imputação é rastreada em colunas `_IMPUTADO_LQ`.

### 2.2. Produtos
* 📊 **`Heatmap_QAQC_Censurados.png`** — mapa de calor (`seaborn`) com a fração de amostras censuradas por ETE/parâmetro, ordenado por incidência. Células escuras = maior percentual de não-detecção.
* 📄 **`Relatorio_QAQC_Dados_Censurados.xlsx`** — tabela ETE × parâmetro com o percentual de censura.

---

## 3. Módulo de Estatística (Básica, Avançada e Camadas v3)

Roda sobre a base tratada e cobre todo o escopo CONAMA 430/11.

### 3.1. Percentil 95 (P95)
Calcula N, média, mediana, desvio e o **P95** (`numpy.percentile`), usado como "pior cenário crônico" — a concentração em que a ETE opera em 95% do tempo.

### 3.2. Mann-Kendall (tendência temporal)
`pymannkendall` aplicado à série cronológica de cada ETE; classifica em *Crescente (Piora)*, *Decrescente (Melhora)* ou *Estável* com o respectivo `p-value`. Protegido contra séries sem variância.

### 3.3. Kruskal-Wallis (comparação entre ETEs)
`scipy.stats.kruskal` testa se as ETEs pertencem à mesma população estatística.

### 3.4. 🆕 Camadas analíticas da v3
* **Pós-hoc de Dunn** (`scikit-posthocs`, ajuste de Holm): executado **somente quando o Kruskal-Wallis é significativo**, identifica **quais pares de ETEs** diferem entre si — o que o teste omnibus sozinho não revela.
* **Detecção de ruptura / regime-shift** (`ruptures`, PELT/L2 sobre o sinal padronizado): identifica mudanças de patamar na série, indo além da tendência monotônica do Mann-Kendall. As datas das rupturas são marcadas nos gráficos de série temporal.
* **Validação de schema** (`pandera`): valida tipos e domínios (ETE textual, DATA temporal, parâmetros numéricos ≥ 0) antes da análise. Resultado registrado em aba própria.
* **Outliers por IQR** (Tukey 1,5×): contagem de outliers por ETE/parâmetro, robusta para dados assimétricos.

> As bibliotecas das camadas novas têm **import defensivo**: se ausentes, a camada é pulada e o restante do pipeline continua.

### 3.5. Produto
* 📄 **`Relatorio_Analitico_Mestre_v3.xlsx`** — múltiplas abas:
  `Estatistica_e_Tendencia` (descritiva + P95 + Outliers IQR + Mann-Kendall + nº de rupturas) · `Kruskal_Wallis` · `PostHoc_Dunn` · `Pontos_de_Ruptura` · `Validacao_Schema`.
  *(O `Relatorio_Analitico_Mestre.xlsx` é o produto equivalente da v2.)*

---

## 4. Módulo de Modelagem Hidrodinâmica e Balanço de Massa

### 4.1. Parser temporal e Left Join
* **`extrair_data_segura()`** converte tanto `Datetime` do Excel quanto strings tipo `Jan/24` para um `Timestamp` absoluto.
* **Left Join** do GRI com a vazão (`how='left'`) fixa a linha do tempo Jan/24–Mar/26, gerando `NaN` controlados para meses ainda não reportados.

### 4.2. Segregação Geográfica
* **Segmento BM:** $Q_{BeloMonte}$ × (ETE 01 + ETE 02).
* **Segmento PM:** $Q_{Pimental}$ (TVR) × (ETE PM + ETE Compacta).

### 4.3. 🆕 P95 Automatizado e Segmentado (v2.1)
O P95 de DBO, N-Amoniacal e Fósforo usado no balanço **deixou de ser um número fixo** e passa a ser **derivado da base processada, por segmento** (`calcular_p95_segmento`), usando apenas as ETEs daquele segmento — coerente com a segregação dos volumes. A função imprime a **rastreabilidade** (coluna de origem e N) e mantém *fallback* caso a coluna não exista. Isso revelou, por exemplo, que o efluente de Pimental é mais concentrado em DBO que o de Belo Monte — contraste antes mascarado por um P95 único.

### 4.4. Matemática do Balanço de Massa
1.  **Conversão volumétrica:** $V_{rio\,(m^3/m\hat{e}s)} = Q_{m^3/s} \times 86400 \times 30{,}44$.
2.  **Fator de Diluição:** $FD = V_{rio} / V_{efluente}$.
3.  **Adição de carga efetiva:** $C_{adic} = (V_{efluente} \times P95) / V_{rio}$.
> Premissas explícitas: concentração de fundo do rio = 0 e mistura completa (estimativa conservadora de primeira ordem).

### 4.5. Produto
* 📄 **`Relatorio_Diluicao_Mestre_Segregado.xlsx`** — vazão, volume mensal, Fator de Diluição e acréscimos por parâmetro, separados por UHE, incluindo o P95 segmentado utilizado.

---

## 5. Módulo de Geração Visual (Data Storytelling)

Gráficos em `matplotlib` seguindo boas práticas de narrativa de dados: hierarquia tipográfica (manchete + subtítulo com o *insight* quantitativo), *decluttering* (remoção de molduras/grades supérfluas), rotulagem direta, cor com propósito (paleta Norte Energia) e fonte no rodapé. **Eixo temporal em português** (formatador independente de *locale*), acentuação completa e unidade `°C`.

### 5.1. Proporção Hidrodinâmica (ex.: `Produto1A_Diluicao_BeloMonte.png`)
Barras com o Fator de Diluição (em milhões); o **pior mês** é destacado em cor de alerta com anotação direta. O subtítulo traz o pior cenário histórico.

### 5.2. Impacto vs. Limite de Detecção (ex.: `Produto2A_Impacto_DBO_BM.png`)
* **Escala fixa** (formato consagrado para o laudo), travada no limite de detecção do laboratório.
* **Subtítulo honesto:** declara explicitamente o **valor real da adição máxima** e **quantas vezes** ele está abaixo da detecção (ex.: *"adição máxima de 0,00005 mg/L — 40.085× abaixo do limite de detecção"*).
* Anotação do limite CONAMA, com nota de isenção do §1º do Art. 21 para Nitrogênio Amoniacal.

### 5.3. Séries Temporais (ex.: `Plot_DEMANDA_BIOQUIMICA_DE_OXIGENIO_ETE_01.png`)
Linha com VMP e P95 rotulados diretamente, último ponto anotado, faixa de conformidade e **marcação vertical dos pontos de ruptura** detectados.

---

## 6. Qualidade de Código e Reprodutibilidade

* **Caminhos:** centralizados numa única `RAIZ` (`pathlib`); basta alterar uma linha para mudar de máquina.
* **Performance:** colunas de flag/status acumuladas e adicionadas em um único `pd.concat` (fim do `PerformanceWarning` de fragmentação).
* **Warnings:** supressão **direcionada** (categorias/mensagens específicas), não global.
* **Portabilidade visual:** fonte com *fallback* (`Segoe UI → Calibri → Arial → DejaVu Sans`) e meses PT-BR sem depender de *locale*.
* **Exportação:** PNG a 600 dpi; planilhas via `openpyxl`.

### Execução (headless)
```powershell
& "C:\Users\thiago\anaconda3\python.exe" -m jupyter nbconvert --to notebook --execute --inplace `
  --ExecutePreprocessor.kernel_name=python3 pipeline_etes_nesa_v2.ipynb
& "C:\Users\thiago\anaconda3\python.exe" -m jupyter nbconvert --to notebook --execute --inplace `
  --ExecutePreprocessor.kernel_name=python3 analise_estatistica_nesa_v3.ipynb
```

### Dependências
`pandas`, `numpy`, `scipy`, `matplotlib`, `seaborn`, `openpyxl`, `pymannkendall`,
`scikit-posthocs`, `ruptures`, `pandera` *(camadas v3)*.

---

## 7. Nota Regulatória (importante)

Os Valores Máximos Permitidos (VMP) referenciam a Resolução CONAMA 430/11 (Art. 16), porém nos produtos atuais são marcados como **"conferir na base do projeto"** e **não são validados automaticamente** nesta execução. A consulta obrigatória à base documental de normas (prevista na skill `analista-python-ambiental`) depende de um acervo que deve ser disponibilizado localmente; enquanto isso, os limites devem ser conferidos por um responsável técnico antes de qualquer conclusão de enquadramento.

## 8. Limitações Conhecidas / Próximos Passos
* Volumes GRI (`dados_gri`) ainda digitados no código — migrar para leitura de planilha.
* Balanço de massa de primeira ordem (sem concentração de fundo do rio).
* P95 segmentado por *pool* simples de amostras — evolução possível: P95 ponderado pela vazão de cada ETE.
* Validação regulatória automatizada pendente de acervo documental local.
