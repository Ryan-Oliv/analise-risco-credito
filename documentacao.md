# Documentação técnica

Registro de decisões, queries e problemas encontrados ao longo do projeto.
Para a visão geral e os resultados, ver o [README](README.md).

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

**Nota:** a coluna de índice do CSV original foi mantida na carga. Ela não tem valor
analítico e deve ser descartada na etapa de EDA.

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

## Problemas encontrados e como pretendo tratar

### `DebtRatio` fora de escala — 20,7% da base

**O problema:** a variável deveria ser uma proporção (despesas ÷ renda), mas 21.688
registros têm valor acima de 2, chegando a 329.664.

**Hipótese:** onde `MonthlyIncome` é nulo ou zero, a divisão provavelmente resultou no
valor absoluto das despesas em vez da razão. Vale cruzar as duas colunas para
confirmar.

**Tratamento pendente:** confirmar a hipótese antes de decidir. Se a correlação com
nulos de renda se confirmar, o tratamento dos dois problemas pode ser conjunto.

### `MonthlyIncome` nulo — 19,8% da base

**O problema:** volume alto demais para simplesmente descartar as linhas — perderia
um quinto da base.

**Opções em avaliação:**
- Imputar pela mediana (não pela média, dada a assimetria à direita)
- Imputar pela mediana da faixa etária correspondente
- Criar uma flag `renda_informada` e testar se a ausência em si tem poder preditivo

A terceira opção me parece a mais interessante: em crédito, não informar renda pode
ser sinal por si só.

### `RevolvingUtilization` acima de 1 — 243 registros

Volume pequeno. Provável erro de cálculo ou caso limite de uso acima do limite
contratado. Decidir entre truncar em 1 ou remover.

### Idade inválida

Um registro com `age = 0` — erro claro, será removido.
Um registro com `age = 109` — plausível, será mantido.

### Coluna de índice

Descartar na etapa de EDA. Não tem valor analítico e pode ser confundida com feature
pelo modelo.

---

## Observações para a modelagem

- **Desbalanceamento (6,6%)**: avaliar `class_weight='balanced'` ou reamostragem.
  Métrica principal: AUC-ROC. Acurácia não será usada como critério.
- **`NumberOfTimes90DaysLate`** mostrou a relação mais forte com o alvo. Atenção a
  possível vazamento conceitual — verificar se a variável é anterior à janela de
  observação do alvo.
- **Relação em "U"** em `NumberOfOpenCreditLinesAndLoans`: modelo linear não captura
  bem. Considerar binning ou modelo baseado em árvore.

---

## Log de progresso

| Etapa | Status | Notas |
|---|---|---|
| 1. Exploração SQL | Concluída | Achados consolidados no README |
| 2. EDA em Python | Em andamento | Tratamento dos problemas listados acima |
| 3. Modelo (scikit-learn) | Pendente | — |
| 4. Power BI + AWS | Pendente | — |
| 5. Documentação final | Contínuo | — |

---

## Pendências abertas

- [ ] Confirmar hipótese sobre `DebtRatio` × `MonthlyIncome` nulo
- [ ] Decidir estratégia de imputação de renda
- [ ] Remover coluna de índice
- [ ] Verificar vazamento conceitual em `NumberOfTimes90DaysLate`
- [ ] Definir estratégia para o desbalanceamento