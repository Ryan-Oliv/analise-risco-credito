# Análise de Risco de Crédito e Inadimplência — Documentação do Projeto

Projeto de portfólio 

## Sobre o dataset

- **Fonte:** Kaggle — "Give Me Some Credit"
- **Arquivo:** `train.csv`
- **Linhas importadas:** 104.805 (o dataset original do Kaggle tem 150.000 linhas; o arquivo utilizado neste projeto contém um subconjunto — decisão consciente de seguir com esse volume)
- **Colunas:** 12 (variável alvo + 10 variáveis explicativas + 1 coluna de índice)

### Dicionário de dados

| Coluna | Tradução | Descrição |
|---|---|---|
| `SeriousDlqin2yrs` | Inadimplência grave em 2 anos | **Variável alvo.** 1 = ficou 90+ dias em atraso nos 2 anos seguintes; 0 = não ficou |
| `RevolvingUtilizationOfUnsecuredLines` | Utilização do limite rotativo (sem garantia) | Saldo usado ÷ limite total disponível (cartão de crédito, cheque especial) |
| `age` | Idade | Idade do cliente em anos |
| `NumberOfTime30-59DaysPastDueNotWorse` | Nº de atrasos de 30-59 dias | Quantidade de atrasos nessa faixa |
| `DebtRatio` | Índice de endividamento | Despesas mensais ÷ renda mensal |
| `MonthlyIncome` | Renda mensal | Renda mensal do cliente |
| `NumberOfOpenCreditLinesAndLoans` | Nº de linhas de crédito/empréstimos abertos | Produtos de crédito ativos |
| `NumberOfTimes90DaysLate` | Nº de atrasos de 90+ dias | Atrasos graves |
| `NumberRealEstateLoansOrLines` | Nº de financiamentos imobiliários | Empréstimos ligados a imóveis |
| `NumberOfTime60-89DaysPastDueNotWorse` | Nº de atrasos de 60-89 dias | Quantidade de atrasos nessa faixa |
| `NumberOfDependents` | Nº de dependentes | Quantidade de dependentes do cliente |

## Stack utilizada

- **Banco de dados:** SQLite (`credito.db`)
- **Ambiente:** Notebook Jupyter dentro do VSCode
- **Importação dos dados:** pandas (`read_csv` + `to_sql`)

## Semana 1 — Exploração SQL

Todas as consultas foram feitas diretamente em SQL (via `sqlite3` em Python) sobre a tabela `clientes`, sem uso de pandas para os cálculos — o objetivo desta etapa é demonstrar domínio de SQL.

### 1. Distribuição da variável alvo

| Classe | Quantidade | Percentual |
|---|---|---|
| Não inadimplente (0) | 97.855 | 93,4% |
| Inadimplente (1) | 6.950 | 6,6% |

**Achado:** dataset desbalanceado. Relevante para a etapa de modelagem (Semana 3) — acurácia sozinha não será uma métrica confiável.

### 2. Qualidade dos dados

- **Valores nulos:** `MonthlyIncome` com 20.781 nulos (~19,8% da base); `NumberOfDependents` com 2.749 nulos (~2,6% da base)
- **Outliers de idade:** 1 linha com `age = 0` (erro de dado) e 1 linha com `age = 109` (outlier plausível, não necessariamente erro). Idade mínima 0, máxima 109, média 52,35 anos
- **Valores fora do esperado:** `RevolvingUtilizationOfUnsecuredLines` (deveria ser uma proporção, 0 a 1) varia de 0 a 29.110 — 243 linhas acima de 2; `DebtRatio` (também deveria ser uma proporção) varia de 0 a 329.664 — 21.688 linhas acima de 2 (~20,7% da base, sugerindo problema mais amplo de escala/cálculo, não apenas outliers isolados)

### 3. Relação entre variáveis e inadimplência

| Variável | Achado |
|---|---|
| Renda mensal | Inadimplentes têm renda média de R$ 5.616 vs R$ 6.763 dos não inadimplentes (~17% menor) |
| Faixa etária | Taxa de inadimplência cai de forma consistente com a idade: 18-29 anos (11,4%) → 30-44 (9,4%) → 45-59 (7,1%) → 60+ (3,0%) |
| Atraso de 90+ dias no passado | Preditor mais forte identificado: 4,6% de inadimplência entre quem nunca atrasou 90+ dias vs 41,6% entre quem já atrasou |
| Nº de linhas de crédito abertas | Relação em "U": sem nenhuma linha aberta, risco de 24,7% (provável falta de histórico); risco mínimo na faixa 6-10 linhas (5,5%); volta a subir levemente em 11+ linhas (6,5%) |
| Imóvel financiado | Quem tem financiamento imobiliário tem risco ~30% menor (5,7% vs 8,2%) |

### 4. Perfil geral da base

- **Renda:** média de R$ 6.684,45, mediana de R$ 5.400,00 — distribuição assimétrica à direita (poucos clientes com renda muito alta puxam a média para cima; a mediana representa melhor o cliente típico)
- **Dependentes:** 60,5% dos clientes sem dependentes (ou não informado), 39,5% com 1 ou mais

### Conclusões da Semana 1 (para tratamento na Semana 2)

- Tratar valores nulos em `MonthlyIncome` e `NumberOfDependents`
- Decidir tratamento para os 2 outliers de idade
- Investigar e tratar a escala anômala de `RevolvingUtilizationOfUnsecuredLines` e `DebtRatio`
- Variáveis com maior potencial preditivo observado até aqui: `NumberOfTimes90DaysLate`, `NumberOfOpenCreditLinesAndLoans`, `age`, `NumberRealEstateLoansOrLines`, `MonthlyIncome`

---

## Semana 2 — EDA em Python
*(em andamento)*

## Semana 3 — Modelo de classificação
*(pendente)*

## Semana 4 — Dashboard Power BI + AWS
*(pendente)*

## Semana 5 — Documentação final + GitHub
*(pendente)*
