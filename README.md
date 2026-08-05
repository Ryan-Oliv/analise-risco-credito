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

| | |
|---|---|
| Registros analisados | 104.805 |
| Variável alvo | `SeriousDlqin2yrs` (1 = inadimplência grave em 2 anos) |
| Natureza | Base desbalanceada — a maioria dos clientes é adimplente |

Principais variáveis: idade, renda mensal, número de dependentes, utilização de
linhas de crédito rotativo, razão entre dívida e renda e histórico de atrasos em
diferentes faixas (30-59, 60-89 e 90+ dias).

---

## Stack

| Etapa | Ferramenta |
|---|---|
| Armazenamento e exploração | SQLite |
| Análise e modelagem | Python — Pandas, NumPy, Matplotlib, scikit-learn |
| Ambiente | Jupyter Notebook (VS Code) |
| Visualização | Power BI |
| Nuvem | AWS — S3 e Athena |

---

## Estrutura do repositório

```
analise-risco-credito/
├── data/               # base bruta (train.csv)
├── scripts/            # notebooks e scripts do pipeline
├── credito.db          # banco SQLite com os dados carregados
├── documentacao.md     # anotações e decisões técnicas
└── README.md
```

---

## Etapas do pipeline

### 1. Modelagem e exploração em SQL — concluído

Carga do CSV bruto para um banco SQLite e exploração via SQL antes de qualquer
análise em Python. A intenção foi tratar o dado como ele chega no mundo real: dentro
de um banco, não em um DataFrame pronto.

Perguntas respondidas nesta etapa:

- Qual a distribuição da variável alvo e o grau de desbalanceamento da base
- Quais colunas possuem valores nulos e em que proporção
- Onde estão os outliers e quais são plausíveis vs. erros de coleta
- Como cada variável financeira se relaciona com a inadimplência

### 2. Análise exploratória em Python — em andamento

Aprofundamento da EDA com Pandas e visualizações, tratamento dos nulos identificados
na etapa anterior e engenharia de atributos.

### 3. Modelo de classificação — planejado

Modelo em scikit-learn para prever `SeriousDlqin2yrs`, com atenção especial ao
desbalanceamento da base. Avaliação por AUC-ROC, matriz de confusão e análise de
precisão x recall — acurácia isolada não é métrica adequada aqui.

### 4. Dashboard e nuvem — planejado

Dashboard de monitoramento de risco em Power BI e disponibilização da base tratada
em AWS (S3 + Athena), reproduzindo em nuvem as consultas feitas localmente.

### 5. Documentação — contínuo

Registro das decisões técnicas e dos achados em `documentacao.md`.

---

## Principais achados até aqui

> Preencher com os resultados reais da exploração SQL — por exemplo:
> percentual de inadimplentes na base, quais colunas tinham nulos e em que volume,
> quais variáveis mostraram maior relação com o alvo.

---

## Como reproduzir

```bash
# clonar o repositório
git clone https://github.com/Ryan-Oliv/analise-risco-credito.git
cd analise-risco-credito

# instalar dependências
pip install pandas numpy matplotlib scikit-learn jupyter

# abrir os notebooks
jupyter notebook
```

O banco `credito.db` já está versionado, então é possível rodar as consultas SQL sem
refazer a carga.

---

## Status

Projeto em desenvolvimento. As etapas concluídas estão sinalizadas acima e o
repositório é atualizado conforme cada fase avança.

---

## Autor

**Ryan Oliveira** — Analista de Dados
[LinkedIn](https://www.linkedin.com/in/ryan-oliv) · [GitHub](https://github.com/Ryan-Oliv)
