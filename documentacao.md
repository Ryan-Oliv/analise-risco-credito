## Principais achados da exploração SQL

Todas as consultas desta etapa foram feitas diretamente em SQL sobre a tabela
`clientes`, sem uso de Pandas para os cálculos.

### Base fortemente desbalanceada

| Classe | Registros | % |
|---|---|---|
| Adimplente (0) | 97.855 | 93,4% |
| Inadimplente (1) | 6.950 | 6,6% |

Apenas 6,6% da base é inadimplente. Isso define a estratégia de avaliação da etapa
de modelagem: acurácia não é métrica confiável aqui — um modelo que classificasse
todos como adimplentes acertaria 93,4% e seria inútil.

### Histórico de atraso é o preditor mais forte

| Já atrasou 90+ dias | Taxa de inadimplência |
|---|---|
| Não | 4,6% |
| Sim | 41,6% |

Quem já ficou 90 dias ou mais em atraso tem risco **9 vezes maior**. Foi a relação
mais forte encontrada na base.

### Risco cai de forma consistente com a idade

| Faixa etária | Taxa de inadimplência |
|---|---|
| 18-29 | 11,4% |
| 30-44 | 9,4% |
| 45-59 | 7,1% |
| 60+ | 3,0% |

### Linhas de crédito abertas: relação em "U"

| Linhas abertas | Taxa de inadimplência |
|---|---|
| Nenhuma | 24,7% |
| 6 a 10 | 5,5% |
| 11 ou mais | 6,5% |

Clientes sem nenhuma linha de crédito aberta apresentam o maior risco da base —
prová