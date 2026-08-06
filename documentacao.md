# Documentação técnica

Registro de decisões, queries e problemas encontrados ao longo do projeto.
Para a visão geral e os resultados, ver o [README](https://github.com/Ryan-Oliv/analise-risco-credito/blob/main/README.md).

---

## Decisões de arquitetura

### Por que SQLite e não PostgreSQL

Optei por SQLite por ser um banco em arquivo único, sem servidor e sem configuração.
Para um projeto de portfólio com ~100 mil registros e uma única tabela, um servidor
Postgres traria custo de setup sem trazer benefício analítico.

A sintaxe SQL usada aqui é praticamente portável — a decisão não limita a migração
futura para Postgres ou para o Athena na etapa de nuvem.

**Custo dessa escolha:** o SQLite não tem tipagem forte (type affinity), então valores
inconsistentes entram sem reclamar. Precisei validar tipos manualmente na exploração.

### Por que SQL puro na etapa 1, e não Pandas

Seria mais rápido carregar o CSV em um DataFrame e fazer tudo com `groupby`. Optei
deliberadamente pelo caminho mais longo: carregar em banco e explorar via SQL.

Dois motivos:

1. É como o dado chega na prática — dentro de um banco, não em um arquivo pronto
2. A etapa serve para demonstrar domínio de SQL, que é o requisito mais comum nas
vagas de dados

Pandas entra na etapa 2, para o que ele faz melhor: visualização e transformação.

### Por que segui com 104.805 linhas

O dataset original do Kaggle tem 150.000 registros; o arquivo que utilizei contém
104.805. Como o objetivo é demonstrar o processo — não competir por acurácia na
competição original — segui com o volume disponível.

O desbalanceamento e a distribuição das variáveis se mantêm representativos.

### Ambiente

Jupyter Notebook dentro do VS Code, em vez do Jupyter no navegador. Facilita manter
notebooks, scripts SQL e controle de versão na mesma janela.

---

## Carga dos dados

Importação feita com Pandas apenas para o transporte CSV → banco. Nenhum cálculo
nessa etapa.

```python
import pandas as pd
import sqlite3

df = pd.read_csv('data/train.csv')
conn = sqlite3.connect('credito.db')
df.to_sql('clientes', conn, if_exists='replace', index=False)
conn.close()
```

**Nota:** diferente do planejado inicialmente, a coluna de índice do CSV original
**não** foi mantida na carga — o parâmetro `index=False` já a descarta no momento da
gravação. A pendência de "remover coluna de índice" registrada abaixo não se aplica.

---

## Queries da exploração (etapa 1)

Todas executadas via `sqlite3` sobre a tabela `clientes`.

### Distribuição da variável alvo

```sql
SELECT SeriousDlqin2yrs,
       COUNT(*) AS qtd,
       ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM clientes), 2) AS pct
FROM clientes
GROUP BY SeriousDlqin2yrs;
```

### Contagem de nulos

```sql
SELECT COUNT(*) AS total,
       SUM(CASE WHEN MonthlyIncome IS NULL THEN 1 ELSE 0 END) AS nulos_renda,
       SUM(CASE WHEN NumberOfDependents IS NULL THEN 1 ELSE 0 END) AS nulos_dep
FROM clientes;
```

### Inadimplência por faixa etária

```sql
SELECT CASE
         WHEN age < 30 THEN '18-29'
         WHEN age < 45 THEN '30-44'
         WHEN age < 60 THEN '45-59'
         ELSE '60+'
       END AS faixa,
       COUNT(*) AS clientes,
       ROUND(AVG(SeriousDlqin2yrs) * 100, 2) AS taxa
FROM clientes
GROUP BY faixa
ORDER BY faixa;
```

### Impacto de atraso de 90+ dias

```sql
SELECT CASE WHEN NumberOfTimes90DaysLate = 0 THEN 'nunca' ELSE 'ja atrasou' END AS hist,
       COUNT(*) AS clientes,
       ROUND(AVG(SeriousDlqin2yrs) * 100, 2) AS taxa
FROM clientes
GROUP BY hist;
```

### Verificação de escala das variáveis proporcionais

```sql
SELECT
  SUM(CASE WHEN DebtRatio > 2 THEN 1 ELSE 0 END) AS debtratio_fora,
  MAX(DebtRatio) AS debtratio_max,
  SUM(CASE WHEN RevolvingUtilizationOfUnsecuredLines > 2 THEN 1 ELSE 0 END) AS revol_fora,
  MAX(RevolvingUtilizationOfUnsecuredLines) AS revol_max
FROM clientes;
```

---

## Tratamento dos dados (etapa 2 — EDA em Python)

Todas as transformações abaixo foram aplicadas sobre o DataFrame carregado a partir
de `clientes` e salvas na nova tabela `clientes_tratados`, preservando a tabela
original intacta.

### Idade inválida

```python
df = df[df["age"] > 0]
```

Removido o único registro com `age = 0`. O registro com `age = 109` foi mantido —
plausível, não necessariamente erro.

### `MonthlyIncome`: nulos e outliers

Confirmada a hipótese de que o `DebtRatio` corrompido estava ligado à ausência de
renda antes de decidir a estratégia de imputação (ver seção seguinte). Optei por
imputar pela mediana da faixa etária, já que a renda varia de forma consistente com
a idade (visto na etapa 1), em vez de uma mediana única para toda a base.

```python
df["faixa_etaria"] = pd.cut(df["age"], bins=[0, 29, 44, 59, 120],
                             labels=["18-29", "30-44", "45-59", "60+"])

mediana_por_faixa = df.groupby("faixa_etaria")["MonthlyIncome"].transform("median")
df["MonthlyIncome"] = df["MonthlyIncome"].fillna(mediana_por_faixa)

teto_renda = df["MonthlyIncome"].quantile(0.99)
df["MonthlyIncome"] = df["MonthlyIncome"].clip(upper=teto_renda)
```

Outliers extremos (ex.: um registro de R$ 3.008.750,00) tratados com teto no
percentil 99 (R$ 23.266,70), em vez de um valor de corte arbitrário.

### `DebtRatio`: causa raiz identificada e corrigida

Cruzando `DebtRatio` com a informação de renda nula:

| Grupo                  | Mediana `DebtRatio` |
| ----------------------- | --------------------- |
| Renda informada         | 0,296                |
| Renda nula (original)   | 1.164                |

Quando a renda era nula, o valor de `DebtRatio` correspondia às despesas em valor
absoluto — o cálculo despesas ÷ renda não pôde ser feito, e o numerador ficou
registrado "cru" no lugar do resultado da divisão. Não é um problema de outliers
aleatórios: é sistemático, atingindo 20,7% da base.

**Tratamento:** para os registros afetados, recalculei `DebtRatio` = valor original
(despesas) ÷ nova renda (já imputada). Para os outliers remanescentes (inclusive
entre quem já tinha renda válida), apliquei teto no percentil 99.

```python
mask_corrompido = df["renda_informada"] == False
df.loc[mask_corrompido, "DebtRatio"] = (
    df.loc[mask_corrompido, "DebtRatio"] / df.loc[mask_corrompido, "MonthlyIncome"]
)

teto_debtratio = df["DebtRatio"].quantile(0.99)
df["DebtRatio"] = df["DebtRatio"].clip(upper=teto_debtratio)
```

**Validação:** após o tratamento, a mediana do `DebtRatio` nos casos antes
corrompidos passou a 0,28 — praticamente igual à mediana de quem sempre teve renda
válida (0,296). Isso indica que a correção resolveu a causa do problema, e não
apenas mascarou o sintoma com um corte arbitrário.

### `RevolvingUtilizationOfUnsecuredLines`

Sem uma causa identificável como no `DebtRatio` — outliers tratados diretamente com
teto no percentil 99 (1,09).

```python
teto_revolving = df["RevolvingUtilizationOfUnsecuredLines"].quantile(0.99)
df["RevolvingUtilizationOfUnsecuredLines"] = (
    df["RevolvingUtilizationOfUnsecuredLines"].clip(upper=teto_revolving)
)
```

### Matriz de correlação

Identificada multicolinearidade forte entre as três variáveis de atraso (0,98-0,99
entre si) — relevante para a etapa de modelagem, já que um modelo linear pode sofrer
com essa redundância. Correlação linear de todas as variáveis com o alvo é fraca
isoladamente (máx. 0,12), o que é esperado dado que os padrões mais fortes
observados em SQL (ex.: 4,6% → 41,6% em atraso de 90+ dias) são relações não
lineares — a correlação de Pearson não as captura bem.

### Persistência

```python
conexao = sqlite3.connect("../credito.db")
df.to_sql("clientes_tratados", conexao, if_exists="replace", index=False)
conexao.close()
```

---

## Observações para a modelagem

- **Desbalanceamento (6,6%)**: avaliar `class_weight='balanced'` ou reamostragem.
Métrica principal: AUC-ROC. Acurácia não será usada como critério.
- **`NumberOfTimes90DaysLate`** mostrou a relação mais forte com o alvo. Atenção a
possível vazamento conceitual — verificar se a variável é anterior à janela de
observação do alvo.
- **Relação em "U"** em `NumberOfOpenCreditLinesAndLoans`: modelo linear não captura
bem. Considerar binning ou modelo baseado em árvore.
- **Multicolinearidade** entre as três variáveis de atraso: avaliar remoção de
redundância caso o modelo final seja linear.

---

## Log de progresso

| Etapa                    | Status       | Notas                                                      |
| ------------------------- | ------------- | ------------------------------------------------------------ |
| 1. Exploração SQL        | Concluída     | Achados consolidados no README                              |
| 2. EDA em Python         | Concluída     | Tratamentos aplicados, base salva em `clientes_tratados`     |
| 3. Modelo (scikit-learn) | Em andamento  | Regressão Logística — `04_modelo_classificacao.ipynb`        |
| 4. Power BI + AWS        | Pendente      | —                                                            |
| 5. Documentação final    | Contínuo      | —                                                            |

---

## Pendências abertas

- [x] Confirmar hipótese sobre `DebtRatio` × `MonthlyIncome` nulo
- [x] Decidir estratégia de imputação de renda
- [x] ~~Remover coluna de índice~~ — não aplicável, `index=False` já resolvia isso
- [ ] Verificar vazamento conceitual em `NumberOfTimes90DaysLate`
- [ ] Definir estratégia para o desbalanceamento
- [ ] Treinar e avaliar Regressão Logística