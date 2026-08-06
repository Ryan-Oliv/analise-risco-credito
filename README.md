![Análise de Risco de Crédito e Previsão de Inadimplência](capa-projeto.png)

# Análise de Risco de Crédito e Previsão de Inadimplência

Pipeline de dados ponta a ponta para prever a probabilidade de um cliente entrar em
inadimplência grave nos próximos 24 meses.

Projeto de portfólio construído para demonstrar o ciclo completo de um problema de
dados: da modelagem em banco até a entrega em dashboard e nuvem.

---

## Problema de negócio

Instituições financeiras precisam decidir a quem conceder crédito. Conceder a um
cliente que não vai pagar gera perda direta; negar a um bom pagador gera receita
perdida e insatisfação.

O objetivo aqui é construir um modelo que estime a probabilidade de inadimplência a
partir do histórico financeiro do cliente, permitindo ordenar a carteira por risco em
vez de tratar todos os solicitantes da mesma forma.

---

## Dataset

**Give Me Some Credit** — competição do Kaggle.

|                      |                                                                  |
| -------------------- | ---------------------------------------------------------------- |
| Registros analisados | 104.804 (após remoção de 1 registro com idade inválida)         |
| Colunas              | 12                                                               |
| Variável alvo        | `SeriousDlqin2yrs` (1 = 90+ dias de atraso nos 2 anos seguintes) |
| Natureza             | Base desbalanceada — 6,6% de inadimplentes                       |

O dataset original do Kaggle possui 150.000 linhas; o arquivo utilizado neste projeto
contém um subconjunto de 104.805 registros (104.804 após limpeza).

### Dicionário de dados

| Coluna                                 | Descrição                                               |
| -------------------------------------- | ------------------------------------------------------- |
| `SeriousDlqin2yrs`                     | **Alvo.** Ficou 90+ dias em atraso nos 2 anos seguintes |
| `RevolvingUtilizationOfUnsecuredLines` | Saldo usado ÷ limite disponível (rotativo sem garantia) |
| `age`                                  | Idade do cliente                                        |
| `NumberOfTime30-59DaysPastDueNotWorse` | Nº de atrasos de 30 a 59 dias                           |
| `DebtRatio`                            | Despesas mensais ÷ renda mensal                         |
| `MonthlyIncome`                        | Renda mensal                                            |
| `NumberOfOpenCreditLinesAndLoans`      | Produtos de crédito ativos                              |
| `NumberOfTimes90DaysLate`              | Nº de atrasos de 90+ dias                               |
| `NumberRealEstateLoansOrLines`         | Nº de financiamentos imobiliários                       |
| `NumberOfTime60-89DaysPastDueNotWorse` | Nº de atrasos de 60 a 89 dias                           |
| `NumberOfDependents`                   | Nº de dependentes                                       |

---

## Stack

| Etapa                      | Ferramenta                                                 |
| --------------------------- | ----------------------------------------------------------- |
| Armazenamento e exploração | SQLite                                                       |
| Análise e modelagem        | Python — Pandas, NumPy, Matplotlib, Seaborn, scikit-learn   |
| Ambiente                   | Jupyter Notebook (VS Code)                                   |
| Visualização               | Power BI                                                     |
| Nuvem                      | AWS — S3 e Athena                                            |

---

## Estrutura do repositório
analise-risco-credito/
├── data/ # base bruta (train.csv)
├── scripts/ # notebooks e scripts do pipeline
├── credito.db # banco SQLite (tabelas clientes e clientes_tratados)
├── documentacao.md # anotações e decisões técnicas
└── README.md

---

## Principais achados da exploração SQL

Todas as consultas desta etapa foram feitas diretamente em SQL sobre a tabela `clientes`, sem uso de Pandas para os cálculos.

### Base fortemente desbalanceada

| Classe           | Registros | %     |
| ---------------- | --------- | ----- |
| Adimplente (0)   | 97.855    | 93,4% |
| Inadimplente (1) | 6.950     | 6,6%  |

Apenas 6,6% da base é inadimplente. Isso define a estratégia de avaliação da etapa de
modelagem: acurácia não é métrica confiável aqui — um modelo que classificasse todos
os clientes como adimplentes acertaria 93,4% e seria inútil.

### Histórico de atraso é o preditor mais forte

| Já atrasou 90+ dias | Taxa de inadimplência |
| ------------------- | --------------------- |
| Não                 | 4,6%                  |
| Sim                 | 41,6%                 |

Quem já ficou 90 dias ou mais em atraso apresenta risco **9 vezes maior**. Foi a
relação mais forte encontrada na base.

### Risco cai de forma consistente com a idade

| Faixa etária | Taxa de inadimplência |
| ------------ | --------------------- |
| 18-29        | 11,4%                 |
| 30-44        | 9,4%                  |
| 45-59        | 7,1%                  |
| 60+          | 3,0%                  |

### Linhas de crédito abertas: relação em "U"

| Linhas abertas | Taxa de inadimplência |
| -------------- | --------------------- |
| Nenhuma        | 24,7%                 |
| 6 a 10         | 5,5%                  |
| 11 ou mais     | 6,5%                  |

Clientes sem nenhuma linha de crédito aberta apresentam o maior risco da base —
provável efeito de ausência de histórico creditício, não de mau pagamento. O risco
mínimo aparece na faixa de 6 a 10 linhas e volta a subir levemente acima disso.

### Outros sinais relevantes

- **Renda:** inadimplentes têm renda média de $5.616 contra $6.763 dos adimplentes
(~17% menor)
- **Financiamento imobiliário:** quem tem imóvel financiado apresenta risco ~30% menor
(5,7% vs 8,2%)
- **Distribuição de renda:** média de $6.684 e mediana de $5.400 — assimetria à
direita, então a mediana representa melhor o cliente típico
- **Dependentes:** 60,5% dos clientes sem dependentes declarados

### Problemas de qualidade identificados

| Problema                                             | Volume                   |
| ---------------------------------------------------- | ------------------------ |
| `MonthlyIncome` nulo                                 | 20.781 registros (19,8%) |
| `NumberOfDependents` nulo                            | 2.749 registros (2,6%)   |
| `DebtRatio` fora da escala esperada (> 2)            | 21.688 registros (20,7%) |
| `RevolvingUtilization` fora da escala esperada (> 2) | 243 registros            |
| Idade inválida (`age = 0`)                           | 1 registro               |

`DebtRatio` e `RevolvingUtilizationOfUnsecuredLines` deveriam ser proporções, mas
apresentam valores de até 329.664 e 29.110 respectivamente. O volume de 20,7% em
`DebtRatio` indicava problema sistemático de escala, não outliers isolados — tratado
na etapa de EDA (ver seção abaixo).

### Variáveis priorizadas para a modelagem

`NumberOfTimes90DaysLate` · `NumberOfOpenCreditLinesAndLoans` · `age` · `NumberRealEstateLoansOrLines` · `MonthlyIncome`

---

## Tratamentos aplicados na EDA (Python)

Investigação em Pandas confirmou que o `DebtRatio` corrompido estava diretamente
ligado à ausência de `MonthlyIncome`: quando a renda era nula, o valor registrado em
`DebtRatio` correspondia às despesas em valor absoluto (não à proporção
despesas ÷ renda). Mediana de `DebtRatio` nesse grupo: 1.164, contra 0,296 no grupo
com renda informada.

| Problema                            | Tratamento                                                 |
| ------------------------------------ | ------------------------------------------------------------ |
| Idade inválida (`age = 0`)          | Registro removido                                            |
| `MonthlyIncome` nulo (19,8%)        | Imputado pela mediana da faixa etária correspondente          |
| `MonthlyIncome` outliers extremos   | Teto no percentil 99 (R$ 23.266,70)                          |
| `DebtRatio` corrompido (renda nula) | Recalculado (despesas ÷ nova renda) para os casos afetados     |
| `DebtRatio` outliers remanescentes  | Teto no percentil 99 (274,0)                                 |
| `RevolvingUtilization` outliers     | Teto no percentil 99 (1,09)                                  |

Após o tratamento, a mediana do `DebtRatio` nos casos corrigidos passou a 0,28 —
praticamente igual à do grupo que já tinha renda válida, confirmando que o
tratamento resolveu a causa do problema, não só o sintoma.

A matriz de correlação mostrou forte multicolinearidade entre as três variáveis de
atraso (30-59, 60-89 e 90+ dias — correlação de 0,98-0,99 entre si), e correlação
linear fraca de todas as variáveis com o alvo isoladamente (máx. 0,12, com `age`) —
esperado, já que os padrões mais fortes identificados em SQL (como o salto de 4,6%
para 41,6%) são relações não lineares.

Base tratada salva na tabela `clientes_tratados`, no mesmo banco `credito.db`.

---

## Etapas do pipeline

### 1. Modelagem e exploração em SQL — concluído

Carga do CSV bruto para um banco SQLite e exploração via SQL antes de qualquer análise
em Python. A intenção foi tratar o dado como ele chega no mundo real: dentro de um
banco, não em um DataFrame pronto. Resultados na seção acima.

### 2. Análise exploratória em Python — concluído

Visualizações (distribuição do alvo, idade, renda, boxplots de outliers, matriz de
correlação) e tratamento dos problemas identificados na etapa anterior. Detalhes na
seção "Tratamentos aplicados na EDA" acima.

### 3. Modelo de classificação — em andamento

Modelo em scikit-learn para prever `SeriousDlqin2yrs`, começando por Regressão
Logística como linha de base, com atenção ao desbalanceamento da base. Avaliação por
AUC-ROC, matriz de confusão e análise de precisão x recall.

### 4. Dashboard e nuvem — planejado

Dashboard de monitoramento de risco em Power BI e disponibilização da base tratada em
AWS (S3 + Athena), reproduzindo em nuvem as consultas feitas localmente.

### 5. Documentação — contínuo

Registro das decisões técnicas e dos achados em `documentacao.md`.

---

## Como reproduzir

```bash
# clonar o repositório
git clone https://github.com/Ryan-Oliv/analise-risco-credito.git
cd analise-risco-credito

# instalar dependências
pip install pandas numpy matplotlib seaborn scikit-learn jupyter

# abrir os notebooks
jupyter notebook
```

O banco `credito.db` já está versionado, com as tabelas `clientes` (dados brutos) e
`clientes_tratados` (após limpeza), então é possível rodar as consultas SQL sem
refazer a carga.

---

## Status

Projeto em desenvolvimento. As etapas concluídas estão sinalizadas acima e o
repositório é atualizado conforme cada fase avança.

---

## Autor

**Ryan Oliveira** — Analista de Dados
[LinkedIn](https://www.linkedin.com/in/ryan-oliv) · [GitHub](https://github.com/Ryan-Oliv)