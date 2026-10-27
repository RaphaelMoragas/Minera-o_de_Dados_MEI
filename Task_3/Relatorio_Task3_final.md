# Relatório — Task 3: Modelos individuais e ensemble


Estrutura espelhada nas secções pedidas no enunciado da Task 3. Todos os números citados vêm do Projeto_MINDD_Task3.ipynb, com os folds, a métrica principal (AUC-PR), os baselines e a agregação definidos no protocolo da Task 2.

---

## 1. Configuração comum

Tabela 1 — Pré-processamento por tipo de modelo (ajustado só no treino de cada fold)

| | Regressão logística | Naive Bayes misto | Árvores (DT, RF, HistGB) |
|---|---|---|---|
| Categóricas (local, distrito, tarifa) | One-hot | Frequências com Laplace (`CategoricalNB`) | One-hot |
| `hora`, `dia_da_semana` | One-hot | Categóricas | Numéricas |
| Contagens e tempos muito assimétricos (skew > 3) | `log1p` + `StandardScaler` | `log1p` + normal por classe (`GaussianNB`) | Sem transformação |
| Restantes numéricas | `StandardScaler` | Normal por classe (`GaussianNB`) | Sem transformação |
| NaN | Máximo do treino | Máximo do treino | Mantidos (a implementação suporta-os) |


> Aulas: Aula 4, slides 30 e 64 · regra de ouro da Aula 3 · Guia 5.3 e 8.

Nesta parte da Task 3, estamos apenas reutilizando o protocolo definido na Task 2 (3 folds temporais, AUC-PR, regra e no-skill, média simples + desvio + pior fold + folds > regra). Como mudamos de notebook, esses dados acabaram por se perder e, logo, precisamos definir as variáveis e guardá-las novamente.

Na Logistic Regression (LR), cada feature contribui com peso × valor, e isso acaba por ser um problema quando temos uma variável como a hora, que vai de 0 a 23: o modelo só consegue aprender um efeito monótono (o risco só sobe ou só desce ao longo do dia). Além disso, os valores mínimo e máximo da hora estão lado a lado, o que não acontece nos números (0 fica longe de 23 na numeração, mas, em relação à hora, estão lado a lado). A solução acaba por ser usar one-hot, que dá um peso independente para cada hora. Esse mesmo one-hot acaba sendo desnecessário e até contraproducente nas árvores, já que elas fazem cortes do tipo Hora >= X e conseguem separar intervalos de horas a partir da variável numérica. Com one-hot, cada corte só separaria uma hora das restantes.

Outro ponto que é bom explicar é que o log1p não causa leakage, diferente do StandardScaler. Isso se dá pela forma como as fórmulas funcionam: o log1p transforma cada valor sozinho (log(1 + x)), sem usar informação das outras sessões, por isso pode ser aplicado a qualquer momento sem gerar leakage. Já o StandardScaler, por subtrair a média e dividir pelo desvio padrão, usa informação que precisa ser extraída dos dados. Ou seja, existe um momento certo para aplicá-lo: ele é ajustado só no treino de cada fold (dentro do Pipeline) e depois aplicado à validação. Se fosse ajustado com todos os dados, a média e o desvio já incluiriam informação da validação.

E, finalmente, chegamos aos NaN, que recebem o valor máximo do treino de cada fold. Esses NaN aparecem nas colunas do tipo "horas desde o último evento" quando o evento nunca aconteceu. Caso recebessem 0, entrariam com o significado oposto: o evento teria acabado de acontecer, quando na verdade nunca aconteceu. Dessa forma, receber o valor máximo do treino nos dá uma variável utilizável e coerente (quanto maior o valor, mais tempo sem o evento), e a flag *_no_previous_* continua a separar os casos em que o evento nunca aconteceu.

---

## 2. Contribuição dos grupos de variáveis (ablação)


### 2.1 Grupos e modelos

Tabela 2.1 — Grupos de variáveis

| Grupo | Variáveis acrescentadas | Nº total |
|---|---|---|
| G1 — contexto e clima | `Location Information`, `District Name`, `hora`, `dia_da_semana`, temperatura, humidade, precipitação | 7 |
| G2 — G1 + tarifa | `tariff_period` | 8 |
| G3 — G2 + histórico do posto | 12 variáveis `post_*` | 20 |
| G4 — G3 + histórico do utilizador | 9 variáveis `user_*` e `user_active_sessions_at_start` | 30 |

Por que acabamos por escolher, em princípio, 2 modelos para a ablação e não apenas um? Além de serem modelos com princípios diferentes. A ideia principal era a de verificar se os nossos resultados dependem ou não dos modelos que utilizamos. A ideia geral é que a nossa conclusão não dependa do tipo de modelo. Se uma variável ou um grupo de variáveis melhora os dois modelos, isso indica que o ganho vem da informação e não do modelo.

A variável user_active_sessions_at_start acabou por ficar no grupo do usuário, pois ela conta quantas sessões abertas o usuário tem no momento, descrevendo assim o usuário, e não o posto. Caso ficasse no grupo 3, esse grupo já teria informação do usuário, e a comparação entre o grupo 3 e o grupo 4 deixaria de medir quanto vale o histórico do usuário.

Já a variável UserID ficou de fora por questões de privacidade, visto que basear a previsão de falha no usuário poderia acabar por marcar usuários específicos como responsáveis pelas falhas. O Start Time acabou servindo apenas para tirar as variáveis hora e dia da semana. Já o Charging Post ID ficou de fora, pois o modelo pode decorar postos específicos, além de não ajudar na previsão para postos novos (secção 2.8).

### 2.2 Resultados

**Tabela 2.2a** — AUC-PR na validação de cada fold, por modelo e grupo de variáveis

| Modelo | Grupo | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra |
|---|---|---|---|---|---|---|---|---|
| No-skill | — | 0,158 | 0,197 | 0,210 | 0,188 | 0,027 | 0,158 | — |
| Regra | — | 0,450 | 0,602 | 0,590 | 0,548 | 0,084 | 0,450 | — |
| LR | G1 | 0,249 | 0,310 | 0,334 | 0,298 | 0,044 | 0,249 | 0 |
| LR | G2 | 0,249 | 0,310 | 0,335 | 0,298 | 0,044 | 0,249 | 0 |
| LR | G3 | 0,506 | 0,636 | 0,617 | 0,586 | 0,070 | 0,506 | 3 |
| LR | G4 | 0,532 | 0,652 | 0,627 | 0,604 | 0,063 | 0,532 | 3 |
| HistGB | G1 | 0,232 | 0,300 | 0,342 | 0,292 | 0,055 | 0,232 | 0 |
| HistGB | G2 | 0,233 | 0,302 | 0,347 | 0,294 | 0,057 | 0,233 | 0 |
| HistGB | G3 | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| HistGB | G4 | 0,564 | 0,679 | 0,653 | 0,632 | 0,061 | 0,564 | 3 |

**Tabela 2.2b** — Ganho de AUC-PR ao acrescentar cada grupo (diferença para o grupo anterior)

| Modelo | Variáveis acrescentadas | Fold 1 | Fold 2 | Fold 3 | Média |
|---|---|---|---|---|---|
| LR | Tarifa (G2 − G1) | +0,0000 | +0,0000 | +0,0001 | +0,0001 |
| LR | Histórico do posto (G3 − G2) | +0,2572 | +0,3263 | +0,2822 | +0,2885 |
| LR | Histórico do utilizador (G4 − G3) | +0,0258 | +0,0158 | +0,0104 | +0,0174 |
| HistGB | Tarifa (G2 − G1) | +0,0005 | +0,0016 | +0,0045 | +0,0022 |
| HistGB | Histórico do posto (G3 − G2) | +0,3048 | +0,3587 | +0,2950 | +0,3195 |
| HistGB | Histórico do utilizador (G4 − G3) | +0,0258 | +0,0187 | +0,0114 | +0,0186 |

### 2.3 A tarifa traz informação além da hora?

No fim das contas não, não traz. Isso porque cada hora acaba sempre por cair num único período de tarifa. Logo, o ganho que se tem de juntar a tarifa não adiciona muita diferença entre os folds, pois sabendo a hora já se sabe a tarifa. O ganho em LR é praticamente nenhum (explicado pelo fato já abordado anteriormente onde a hora aqui entra como one-hot). Já no HistGB a hora entra mesmo como número, precisando de dois cortes enquanto a tarifa já junta esses cortes em uma variável.

A tarifa acaba por ser um facilitador do trabalho da árvore, mas é apenas uma outra forma de se apresentar a mesma informação já existente na hora, não uma informação nova. Esse ponto já havia sido discutido em notebooks anteriores.

### 2.4 Quanto contribui o histórico do posto?

No fundo o histórico do posto acaba por ser, de longe, o grupo que mais contribui nas análises. Em relação ao acaso, os modelos passam de cerca de 1,5× para cerca de 3× (tabela 2.6). Ao passar do G2 para o G3, a AUC-PR sobe em média +0,29 na LR e +0,32 no HistGB (tabela 2.2b), enquanto a tarifa não traz quase nada e o histórico do usuário traz cerca de +0,02.

Além disso, com o G3, os dois modelos passam a ganhar da regra nos 3 folds, o que não acontecia com os grupos 1 e 2. Como a regra usa apenas a rate_1D, isso mostra que juntar o histórico completo do posto (7 e 30 dias, horas desde a última falha, entre outras variáveis) acaba por trazer mais informação além da do dia anterior.
