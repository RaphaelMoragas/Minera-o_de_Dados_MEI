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

Tabela 2.2a — AUC-PR na validação de cada fold, por modelo e grupo de variáveis

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

Tabela 2.2b — Ganho de AUC-PR ao acrescentar cada grupo (diferença para o grupo anterior)

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


### 2.5 O histórico do utilizador traz benefício adicional?

O histórico do usuário acaba por trazer um benefício pequeno, mas consistente. Ao passar do grupo 3 para o grupo 4, a AUC-PR sobe em média +0,017 na LR e +0,019 no HistGB (tabela 2.2b), bem menos do que o histórico do posto, mas o ganho é positivo nas 6 comparações (3 folds × 2 modelos). Porém, esse ganho acaba por diminuir do fold 1 para o fold 3 (de +0,026 para cerca de +0,01 nos dois modelos). Uma hipótese é que, com o tempo, entram mais usuários novos, sem histórico, e assim o histórico do usuário acaba por ajudar menos, mas isso não foi verificado

### 2.6 O efeito é consistente entre períodos?

Tabela 2.6 — AUC-PR a dividir pela prevalência de cada fold (quantas vezes melhor do que o acaso)

| Modelo | Grupo | Fold 1 | Fold 2 | Fold 3 |
|---|---|---|---|---|
| Regra | — | 2,85× | 3,06× | 2,81× |
| LR | G1 | 1,58× | 1,58× | 1,59× |
| LR | G2 | 1,58× | 1,58× | 1,59× |
| LR | G3 | 3,21× | 3,23× | 2,93× |
| LR | G4 | 3,37× | 3,31× | 2,98× |
| HistGB | G1 | 1,47× | 1,53× | 1,63× |
| HistGB | G2 | 1,48× | 1,53× | 1,65× |
| HistGB | G3 | 3,41× | 3,36× | 3,05× |
| HistGB | G4 | 3,57× | 3,45× | 3,10× |

Prevalência de cada fold (= AUC-PR do no-skill): 15,8% · 19,7% · 21,0%.

No geral, sim, o efeito acaba por ser consistente. Olhando a tabela 2.2b, todos os ganhos têm o mesmo sinal nos 3 folds, ou seja, nenhum grupo de variáveis piora o modelo em algum período. O histórico do posto e o histórico do usuário sobem a AUC-PR nos 3 folds e nos dois modelos, e a tarifa fica praticamente em zero em todos eles.

Já na tabela 2.6, a AUC-PR foi dividida pela prevalência de cada fold, pois a AUC-PR do acaso muda de fold para fold (15,8%, 19,7% e 21,0%), e assim temos quantas vezes o modelo é melhor do que o acaso. No fold 3, os modelos com o histórico do posto acabam por cair (o HistGB no grupo 3 passa de 3,36× para 3,05×), mas a regra também cai (de 3,06× para 2,81×). Como a regra não é treinada, isso indica que a queda vem principalmente do próprio período, e não dos modelos em si.

### 2.7 Preocupações operacionais e éticas

Tabela 2.7 — HistGB no G3, avaliado em subgrupos de sessões (o modelo é o mesmo; muda só o conjunto avaliado)

|  | Fold 1 | Fold 2 | Fold 3 |
|---|---|---|---|
| AUC-PR — todas as sessões | 0,538 | 0,661 | 0,642 |
| AUC-PR — sem as sessões de frota | 0,539 | 0,660 | 0,643 |
| AUC-PR — só as sessões de frota | 0,504 | 0,685 | 0,618 |
| Prevalência — todas as sessões | 15,8% | 19,7% | 21,0% |
| Prevalência — só as sessões de frota | 14,0% | 19,9% | 21,8% |
| Todas as sessões ÷ prevalência | 3,41× | 3,36× | 3,05× |
| Só frotas ÷ prevalência das frotas | 3,61× | 3,44× | 2,83× |
| Nº de sessões de frota | 2097 | 2410 | 2481 |

Com o G4, temos um grande problema, o risco de uma sessão acabar por depender também de quem está carregando o veículo. Isso pode marcar usuários por falhas que não causaram e deixar as contas de frota dominarem o padrão. Como o alerta vai apenas para o operador e não bloqueia a sessão, um erro custa no máximo uma inspeção a mais, mas mesmo assim foi um dos motivos para escolhermos o G3.

O histórico do posto também tem os seus problemas: falhas que vêm do carro (como o Vehicle charging faults, que tbm acaba por não ser culpa do usuário) acabam por contar contra o posto, e postos novos ou com poucas sessões têm um histórico pouco confiável. Já a tabela 2.7 mostra que, no G3, tirar as sessões de frota quase não muda a AUC-PR, ou seja, o modelo funciona de forma parecida para as frotas e para o resto. O limite é que são apenas 2 contas de frota.

### 2.8 Variáveis adicionais

Tabela 2.8 — HistGB no G3, com e sem o `Charging Post ID` (em one-hot)

| Modelo | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra |
|---|---|---|---|---|---|---|---|
| HistGB G3 | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| HistGB G3 + `Charging Post ID` | 0,538 | 0,661 | 0,639 | 0,613 | 0,066 | 0,538 | 3 |
| Diferença (com − sem) | +0,0001 | −0,0001 | −0,0024 | −0,0008 | — | — | — |

A hora e o dia da semana já foram justificados na secção 1. O Charging Post ID foi testado no HistGB com o G3 e não trouxe ganho (tabela 2.8), o que faz sentido, já que o histórico do posto já descreve cada posto e o ID não ajuda com postos novos. O UserID não foi testado, por questões de privacidade, e foi usado apenas para identificar as contas de frota (39 e 23). Na Task 1, estas foram as únicas contas com mais de 2 sessões abertas ao mesmo tempo, com milhares de carregamentos.


### 2.9 Grupo escolhido para os modelos

Acabamos por escolher o G3 para os modelos (já separamos o G4 como o final por esse motivo). O G4 acaba por ser um pouco melhor (cerca de +0,02 de AUC-PR), mas o ganho é pequeno e traz os problemas éticos da secção 2.7, além de ir contra a exclusão do UserID. O protocolo da Task 2 diz quais variáveis podem ser usadas, mas não obriga a usar todas.


---

## 3. Modelos individuais

### 3.1 Algoritmos escolhidos

Tabela 3.1 — Princípios de modelação

| Princípio | Algoritmo | Usado |
|---|---|---|
| Linear | Regressão logística | ✅ |
| Probabilístico | Naive Bayes (versão mista) | ✅ |
| Árvores de decisão | Árvore de decisão (profundidade 6) | ✅ |
| Baseado em instâncias | kNN | — |
| Baseado em kernels | SVM | — |

Escolhemos três modelos com princípios diferentes: a regressão logística (linear), o Naive Bayes misto (probabilístico) e a árvore de decisão (árvores). Cada um acaba por olhar os dados de um jeito diferente, são rápidos de treinar e fáceis de interpretar. 

O kNN e o SVM acabram por ficar de fora devido ao tamanho dos dados: com cerca de 180 000 sessões por treino, o kNN precisaria guardar todas as sessões e comparar cada sessão nova com todas elas, e o SVM acaba por ficar muito lento para treinar. Como o próprio enunciado diz, usar muitos algoritmos sem análise não é melhor do que uma comparação menor e bem justificada.

### 3.2 Justificação pelos seis critérios

A regressão logística foi escolhida pela interpretabilidade, já que os coeficientes mostram o sinal e o tamanho do efeito de cada variável no risco de anomalia. 

Já o Naive Bayes misto acaba por ser o modelo mais barato, pois treinar é apenas contar frequências nas categóricas e calcular médias e desvios nas numéricas. 

Por dim a árvore de decisão consegue apanhar limiares e interações entre variáveis (por exemplo, local × histórico do posto) sem precisar de escala.

O pressuposto de independência do Naive Bayes acaba por ser violado aqui, o que aparece nas probabilidades extremas (72%, tabela 3.4).

### 3.3 Profundidade da árvore

Tabela 3.3 — Árvore de decisão no G3 com várias profundidades

| `max_depth` | Treino (média) | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra |
|---|---|---|---|---|---|---|---|---|
| 3 | 0,505 | 0,437 | 0,568 | 0,544 | 0,516 | 0,070 | 0,437 | 0 |
| 4 | 0,542 | 0,469 | 0,602 | 0,582 | 0,551 | 0,072 | 0,469 | 1 |
| **6** | 0,576 | 0,503 | 0,626 | 0,606 | 0,578 | 0,066 | 0,503 | 3 |
| 8 | 0,599 | 0,504 | 0,637 | 0,585 | 0,575 | 0,067 | 0,504 | 2 |
| 10 | 0,621 | 0,485 | 0,611 | 0,561 | 0,552 | 0,063 | 0,485 | 2 |
| 12 | 0,652 | 0,433 | 0,554 | 0,522 | 0,503 | 0,063 | 0,433 | 0 |
| Sem limite | 1,000 | 0,199 | 0,274 | 0,276 | 0,249 | 0,044 | 0,199 | 0 |

A profundidade da árvore foi escolhida na validação, testando vários valores e vendo qual tinha o melhor resultado nos folds. Com profundidade 3 e 4, a AUC-PR é baixa tanto no treino como na validação, ou seja, a árvore é simples demais (underfitting). A partir da 8, o treino continua a subir, mas a validação acaba por cair, o que indica overfitting. A 6 e a 8 ficam praticamente empatadas na média (0,578 e 0,575) e no pior fold, mas só a 6 passa a regra nos 3 folds, que é o critério do protocolo, por isso escolhemos a 6.

A árvore sem limite cai para 0,249 muito provavelmente por decorar o treino. A limitação é que a profundidade foi escolhida nos mesmos folds onde depois comparamos os modelos, o que pode deixar a árvore um pouco favorecida.

### 3.4 Naive Bayes: versão gaussiana e versão mista

Tabela 3.4 — Naive Bayes no G3: versão gaussiana (antiga) e versão mista

| Versão | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra | Prob. extremas (fold 3) |
|---|---|---|---|---|---|---|---|---|
| GaussianNB (one-hot + normal em todas as colunas) | 0,444 | 0,586 | 0,566 | 0,532 | 0,077 | 0,444 | 0 | 76,8% |
| NB misto (categóricas por frequências, numéricas pela normal) | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 | 72,0% |
| NB misto sem `tariff_period` | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 | 71,9% |

Na versão gaussiana, todas as colunas eram tratadas como números com distribuição normal, até as categóricas, que só valem 0 ou 1. Por isso passamos para o NB misto, onde as categóricas usam frequências e as numéricas continuam com a normal. Com isso, a AUC-PR média subiu de 0,532 para 0,551 nas médias.

Mesmo assim, 72% das probabilidades continuam extremas (abaixo de 1% ou acima de 99%), e tirar a tarifa não muda nada. Logo, o problema acaba por ser o pressuposto de independência: as variáveis do posto estão correlacionadas e o modelo conta a mesma informação várias vezes.

OBS: Prob. extremas = percentagem de probabilidades abaixo de 1% ou acima de 99% na validação do fold 3. Na regressão logística é 0,3%.

### 3.5 Resultados

Tabela 3.5 — Modelos individuais no G3

| Modelo | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra |
|---|---|---|---|---|---|---|---|
| No-skill | 0,158 | 0,197 | 0,210 | 0,188 | 0,027 | 0,158 | — |
| Regra | 0,450 | 0,602 | 0,590 | 0,548 | 0,084 | 0,450 | — |
| Regressão logística | 0,506 | 0,636 | 0,617 | 0,586 | 0,070 | 0,506 | 3 |
| Naive Bayes misto | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 |
| Árvore de decisão (profundidade 6) | 0,503 | 0,626 | 0,606 | 0,578 | 0,066 | 0,503 | 3 |

Entre os modelos individuais, a regressão logística acaba por ser a melhor (0,586 de média), seguida da árvore (0,578) e do Naive Bayes (0,551). A LR e a árvore passam a regra nos 3 folds. Já o NB, apesar de ter média um pouco acima da regra (0,551 contra 0,548), perde para ela em 2 folds. Pelo critério do protocolo da Task 2, que pede para passar a regra em todos os folds, o NB acaba por não ser considerado um bom modelo para nossos dados.

### 3.6 Interpretação dos modelos


Tabela 3.6a — Regressão logística (G3, modelo do fold 3): os 10 coeficientes maiores em valor absoluto

| Variável (depois do pré-processamento) | Coeficiente |
|---|---|
| `post_previous_abnormal_rate_1D` (escala) | +0,932 |
| `post_previous_abnormal_1D` (log + escala) | −0,804 |
| `Location Information` = Technology Park | −0,582 |
| `post_hours_since_previous_completed_abnormal` (log + escala) | −0,573 |
| `post_previous_abnormal_rate_7D` (escala) | +0,421 |
| `Location Information` = Tourist Attraction | +0,293 |
| `post_previous_sessions_1D` (escala) | +0,293 |
| `Location Information` = Park A | +0,228 |
| `Location Information` = Industrial Park | −0,226 |
| `Location Information` = Shopping Mall | +0,217 |

Tabela 3.6b — Árvore de decisão (G3, profundidade 6, modelo do fold 3): variáveis mais importantes

| Variável | Importância |
|---|---|
| `post_previous_abnormal_rate_7D` | 0,557 |
| `post_hours_since_previous_completed_abnormal` | 0,171 |
| `post_previous_abnormal_rate_1D` | 0,149 |
| `post_previous_abnormal_rate_30D` | 0,104 |
| `Location Information` = Bus Station | 0,003 |
| Cada uma das restantes variáveis | < 0,002 |
| Soma das 4 variáveis do posto | 0,982 |

Os dois modelos concordam que o que mais pesa é o histórico do posto, o que bate com a secção 2.4: na árvore, as 4 variáveis do posto somam 0,982 da importância, e na LR os maiores coeficientes também são quase todos do posto. A diferença é que a LR se apoia mais na rate_1D, que já era a variável mais correlacionada com o alvo na Task 1. A árvore se apoia mais na rate_7D, que por usar 7 dias acaba por ser mais estável e dar cortes melhores.

Na LR, a rate_1D e a abnormal_1D têm sinais opostos (+0,932 e −0,804). Isso não quer dizer que mais falhas ontem baixam o risco. Como as duas variáveis estão correlacionadas, o modelo acaba por dividir o efeito entre elas, e por isso os coeficientes devem ser lidos em conjunto, e não um a um.

---

## 4. Modelos ensemble


### 4.1 Abordagens e porque podem melhorar os seus modelos base

Tabela 4.1 — Ensembles avaliados

| Ensemble | Princípio (Aula 5) | Modelos base | Configuração |
|---|---|---|---|
| Random Forest | Bagging + sorteio de variáveis em cada divisão | Árvores sem limite de profundidade (`min_samples_leaf=20`), cada uma numa amostra bootstrap | 100 árvores, `class_weight='balanced_subsample'` |
| HistGradientBoosting | Boosting | Árvores pequenas, treinadas em sequência | Valores por defeito, `class_weight='balanced'`, `early_stopping=False` |
| Votação soft | Combinação heterogénea e estática | Regressão logística + NB misto + árvore (profundidade 6) | Média das probabilidades; avaliada também sem o NB |

A Random Forest junta 100 árvores, cada uma treinada numa amostra diferente do treino (bagging), e faz a média das previsões. Ela acaba por reduzir a variância, que é o principal problema de uma árvore sozinha, já que pequenas mudanças nos dados mudam muito a árvore. O sorteio de variáveis em cada divisão serve para as árvores ficarem diferentes entre si. Sem ele, todas começariam pela rate_7D e acabariam parecidas, e a média não ajudaria tanto.

O HistGB treina árvores pequenas em sequência, e cada uma tenta corrigir os erros das anteriores (boosting), o que acaba por reduzir o viés. O early_stopping ficou desligado porque, por defeito, ele separa 10% aleatório do treino para decidir quando parar, o que acabaria por misturar o futuro com o passado. O HistGB não está nos slides, mas funciona como o LightGBM: agrupa os valores em intervalos, o que o deixa rápido com 180 mil sessões.

A votação faz a média das probabilidades da LR, do NB e da árvore. Para ajudar, os modelos precisam de ser bons sozinhos e de errar em sessões diferentes. Como os três têm princípios diferentes, esperávamos alguma diversidade, mas todos se apoiam principalmente no histórico do posto, então acabam por errar em sessões parecidas. Além disso, o NB, com probabilidades muito extremas, pode acabar por atrapalhar.

### 4.2 Resultados: os ensembles melhoram os seus modelos base?

**Tabela 4.2a** — Ensembles no G3

| Ensemble | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra |
|---|---|---|---|---|---|---|---|
| Random Forest | 0,533 | 0,660 | 0,641 | 0,612 | 0,069 | 0,533 | 3 |
| HistGradientBoosting | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| Votação (LR + NB + árvore) | 0,514 | 0,643 | 0,622 | 0,593 | 0,070 | 0,514 | 3 |
| Votação sem o NB (LR + árvore) | 0,524 | 0,650 | 0,628 | 0,601 | 0,067 | 0,524 | 3 |

**Tabela 4.2b** — Diferença de AUC-PR entre cada ensemble e os seus modelos base (positivo = o ensemble é melhor)

| Comparação | Fold 1 | Fold 2 | Fold 3 |
|---|---|---|---|
| Random Forest − árvore (prof. 6) | +0,0302 | +0,0344 | +0,0354 |
| HistGB − árvore (prof. 6) | +0,0347 | +0,0347 | +0,0355 |
| HistGB − Random Forest | +0,0045 | +0,0003 | +0,0001 |
| Votação − LR | +0,0073 | +0,0070 | +0,0053 |
| Votação − NB misto | +0,0347 | +0,0423 | +0,0493 |
| Votação − árvore (prof. 6) | +0,0104 | +0,0174 | +0,0160 |
| Votação sem NB − LR | +0,0175 | +0,0136 | +0,0112 |
| Votação sem NB − árvore (prof. 6) | +0,0207 | +0,0240 | +0,0220 |
| Votação sem NB − Votação | +0,0102 | +0,0066 | +0,0060 |

**Tabela 4.2c** — Probabilidades extremas (abaixo de 1% ou acima de 99%) na validação do fold 3

| Modelo | Percentagem |
|---|---|
| Naive Bayes misto | 72,0% |
| Regressão logística | 0,3% |

Todos os ensembles acabam por ser melhores do que os seus modelos base nos 3 folds (tabela 4.2b). A Random Forest e o HistGB ganham cerca de +0,03 da melhor árvore individual, e ficam praticamente empatados entre si. Isso é interessante na Random Forest, porque ela é feita de árvores mais profundas, que sozinhas acabariam por fazer overfitting (tabela 3.3). Ou seja, o bagging acaba por corrigir esse problema.

Na votação, tirar o NB melhora o resultado nos 3 folds. Isso acontece porque o NB tem probabilidades muito extremas e, como não usa class_weight='balanced', fica numa escala diferente da LR e da árvore, puxando demais a média. Mesmo sem o NB, a votação não chega à Random Forest nem ao HistGB, porque junta só dois ou três modelos mais fracos, que erram em sessões parecidas.

### 4.3 Stacking

Acabamos por não usar stacking. Para usar sem leakage, os modelos base teriam de ser treinados num período mais antigo e o meta-modelo no período seguinte, sempre antes da validação. O StackingClassifier normal, com cv=5, não serve, porque divide os dados de forma aleatória e acabaria por misturar o futuro com o passado.

---

## 5. Comparação final

**Tabela 5a** — Todos os modelos no G3, por ordem da média

| Modelo | Tipo | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra |
|---|---|---|---|---|---|---|---|---|
| HistGradientBoosting | Ensemble | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| Random Forest | Ensemble | 0,533 | 0,660 | 0,641 | 0,612 | 0,069 | 0,533 | 3 |
| Votação sem o NB | Ensemble | 0,524 | 0,650 | 0,628 | 0,601 | 0,067 | 0,524 | 3 |
| Votação (LR + NB + árvore) | Ensemble | 0,514 | 0,643 | 0,622 | 0,593 | 0,070 | 0,514 | 3 |
| Regressão logística | Individual | 0,506 | 0,636 | 0,617 | 0,586 | 0,070 | 0,506 | 3 |
| Árvore de decisão (prof. 6) | Individual | 0,503 | 0,626 | 0,606 | 0,578 | 0,066 | 0,503 | 3 |
| Naive Bayes misto | Individual | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 |
| Regra | Baseline | 0,450 | 0,602 | 0,590 | 0,548 | 0,084 | 0,450 | — |
| No-skill | Baseline | 0,158 | 0,197 | 0,210 | 0,188 | 0,027 | 0,158 | — |

**Tabela 5b** — Diferença para a regra em cada fold (positivo = melhor do que a regra)

| Modelo | Fold 1 | Fold 2 | Fold 3 |
|---|---|---|---|
| HistGradientBoosting | +0,087 | +0,058 | +0,051 |
| Random Forest | +0,083 | +0,058 | +0,051 |
| Votação sem o NB | +0,073 | +0,048 | +0,038 |
| Votação (LR + NB + árvore) | +0,063 | +0,041 | +0,032 |
| Regressão logística | +0,056 | +0,034 | +0,027 |
| Árvore de decisão (prof. 6) | +0,053 | +0,024 | +0,016 |
| Naive Bayes misto | +0,028 | −0,001 | −0,017 |

No fim, os ensembles acabam por ficar à frente dos modelos individuais, com o HistGB (0,613) e a Random Forest (0,612) no topo. Todos os modelos passam a regra nos 3 folds, menos o NB, que tem média um pouco acima da regra mas perde para ela em 2 folds. 

Isso mostra que olhar só para a média não chega. O HistGB e a Random Forest ficam praticamente empatados: o HistGB ganha nos 3 folds, mas por muito pouco (tabela 4.2b). Pelo protocolo da Task 2, o HistGB fica em primeiro por ter a média e o pior fold um pouco melhores, mas como a diferença é muito pequena, os dois seguem para a próxima etapa (afinação, threshold e avaliação final).

---

## 6. Limitações

Escolhas feitas nos mesmos folds da comparação: a profundidade da árvore, a versão do NB e a votação sem o NB foram escolhidas olhando para a validação dos mesmos folds onde depois comparamos os modelos. Isso pode deixar esses modelos um pouco favorecidos, já que acabaram por ser ajustados para esses períodos. O ideal seria escolher num período separado, mas com apenas 3 folds isso deixaria menos dados para a comparação.

Sem teste estatístico: com apenas 3 folds, o teste de Wilcoxon nunca consegue dar significativo, porque o menor p-valor possível é 0,25. Por isso, a comparação acaba por ser feita pela consistência fold a fold (o modelo ganha nos 3 folds ou não?). Diferenças muito pequenas, como a entre o HistGB e a Random Forest, não podem ser consideradas uma diferença real.

Poucas sessões de frota: a análise das frotas usa apenas 2 contas e poucas sessões, então a AUC-PR desse subgrupo pode mudar bastante com poucas sessões a mais ou a menos. Não dá para tirar conclusões fortes dela.

Probabilidades do NB mal calibradas: 72% das probabilidades do NB ficam abaixo de 1% ou acima de 99%, por isso não podem ser lidas como o risco real de uma sessão. É também por isso que o NB atrapalha a votação.

Ensembles sem afinação: a Random Forest e o HistGB ainda usam os valores por defeito, mudando apenas o class_weight e o early_stopping. Os resultados podem melhorar com a afinação, e a ordem entre os dois pode mudar. Isso fica para a próxima etapa.

Falhas do carro no histórico do posto: algumas falhas, como as de BMS ou Vehicle charging faults, vêm do carro e não do posto, mas acabam por contar no histórico do posto. Essa divisão foi feita só pelo nome de cada motivo e não foi verificada.

