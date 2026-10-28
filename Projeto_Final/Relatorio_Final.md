# Relatório final — Project 1: Classification of Abnormal Electric-Vehicle Charging Sessions

## Participantes

- 1220665 - Nuno Marinho
- 1221083 - Rodrigo Cardoso
- 1260464 - Raphael Moragas

Todos os números citados vêm do Projeto_MINDD_Final.ipynb. No notebook, quando a numeração de uma secção não coincide com a do relatório, o título indica entre parênteses a secção correspondente do relatório.

---

# Task 1 — Compreensão, auditoria e preparação dos dados

## 1.1 Carregamento e visão geral

O dataset tem 441 077 linhas e 47 colunas. É possivel observar que 4 colunas de datas (Start time, End Time, Payment Time e Order creation time) foram lidas como texto (string) e precisam de ser convertidas para alguma forma de data antes que possamos fazer qualquer tipo de conta com a mesma. Também é possivel observar que apenas 4 colunas têm valores ausentes, todas do grupo de atributos históricos (post_hours_since_previous_completed_session, post_hours_since_previous_completed_abnormal, user_days_since_previous_completed_session, user_days_since_previous_completed_abnormal).

---

## 1.2 Qualidade dos dados

### 1.2.1 Valores ausentes

Sabemos então quantos valores faltam mas não o pq desses valores estarem faltando. Por exemplo: Se um usuário nunca teve uma falha ele terá NaN no seu estado e não há valor para se inventar nesse caso. O NaN por sí só já é a informação final.

#### post_hours_since_previous_completed_session × post_no_previous_completed_session

As linhas indicam se o valor está ausente (True) ou não (False); as colunas indicam a flag (1 = o posto nunca teve uma sessão concluída antes).

Existem 92 ids unicos de posto, ou seja, 92 postos diferentes entram nesta avaliação.

- 92 sessões têm NaN e o posto não tinha histórico (flag 1). É o esperado pois essas seriam as primeiras sessões de um posto que acabou de iniciar o armazenamento dos dados.
- 440985 sessões têm valor e o posto já tinha histórico (flag 0). É o esperado pois os postos já tiveram uma primeira sessão antes.
- As outras duas células são 0: não existe nenhum NaN num posto com histórico (False 1), nem nenhum valor preenchido num posto sem histórico (True 0).

Os 92 casos correspondem exatamente aos 92 postos do dataset, ou seja, na primeira sessão de cada posto, que não tem uma sessão anterior com que se comparar. O NaN é, portanto, estrutural e não um erro.

#### post_hours_since_previous_completed_abnormal × post_no_previous_completed_abnormal

- 440710 sessões têm valor e o posto já tinha tido pelo menos uma falha.
- 367 sessões têm NaN e o posto ainda não tinha registado nenhuma falha.
- Fora da diagonal, 0 casos.

As 367 sessões são as que ocorreram em cada posto antes da sua primeira falha. Sem uma falha anterior, não existe "tempo desde a última falha". Este NaN também é estrutural.

#### user_days_since_previous_completed_session × user_no_previous_completed_session

- 384430 sessões têm valor e o usuário já tinha pelo menos uma sessão concluída.
- 56647 sessões têm NaN e o usuário não tinha nenhuma sessão concluída antes.
- Fora da diagonal, 0 casos.

Destas 56647, 56115 são a primeira sessão de cada usuário (o dataset tem exatamente 56115 usuários distintos). As restantes 532 são sessões iniciadas enquanto a primeira sessão do mesmo usuário ainda decorria: como o enunciado explica, sessões ainda ativas não contam como "concluídas", por isso continua a não existir uma sessão anterior concluída.
Novamente os resultados acabam por ser um NaN estrutural e não precisamos modificar nada nestas linhas.

#### user_days_since_previous_completed_abnormal × user_no_previous_completed_abnormal

- 304798 sessões têm valor e o usuário já tinha tido pelo menos uma falha.
- 136279 sessões (cerca de 31%) têm NaN e o usuário nunca tinha tido uma falha.
- Fora da diagonal, 0 casos.

É a coluna com mais ausentes, o que faz sentido: muitos usuários têm poucas sessões e nunca passaram por uma falha. E, pela ultima vez, o NaN que encontramos aqui continua sendo estrutural e, logo, não precisamos modificar ou alterar os dados.

#### Conclusão e decisão para as 4 colunas

Em todos os casos, os valores ausentes coincidem exatamente com as flags _no_previous_. Acabam apenas por ser ausencias estruturais, não erros no registo ou adição errônea de informação. Por isso, não serão modificados com média ou mediana de valores no mesmo escopo, porque isso inventaria um histórico inexistente e apagaria a diferença entre "falhou há muito tempo" e "nunca falhou".

### 1.2.2 Dados duplicados

Verificando-se pelo comando, não existem linhas duplicadas no dataset, por isso não é necessária nenhuma ação para esse tipo de problema.

### 1.2.3 Espaços escondidos no texto e conversão de datas

Fizemos então a conversão dos valores de datas de string para date a fim de podermos realizar contas com os valores da tabela.

Um ponto importante é que apenas itens com tipo object ou string podem ter um tab final escondido no meio do texto, como o tipo das datas já foram modificadas e perderam automaticamente o tab final iremos procurar nos outros itens que tenhas os tipos em que esse problema possa aparecer.

Após modificados os itens problemáticos vemos que nenhum item com o tipo object ou string tem mais erros de escrita que possam dividir resultados, trazendo incossitencia nos dados.

### 1.2.4 Consistência e valores extremos

#### Pagamento antes do fim da sessão

Em 87202 sessões (~20%) o pagamento aparece registado antes do fim da sessão. Olhando a diferença entre os dois horários, ela é pequena (mediana de 24 segundos), então parece ser uma diferença de registo entre sistemas e não um erro grave, mas podemos estar errados.

Como Payment time é uma coluna que só existe depois da sessão e não vai ser usada no modelo, não fazemos nenhuma correção. Fica só registado como uma inconsistência dos dados.

#### Valores monetários e sessões com 0 kWh

Os valores monetários parecem ser consistentes: em nenhuma sessão o total é diferente da soma de eletricidade + serviço, e em nenhuma o valor pago é maior que o total pelo que podemos verificar.

Existem 54621 sessões (aproximadamente 12%) com 0 kWh carregados. Isto não é um erro, são sessões que terminaram sem carregar energia, mais a frente iremos verificar se se elas são de um erro (se são sessões anormais). Por isso essas linhas serão mantidas por enquanto.

#### Clima

Verificamos se os valores do clima fazem sentido para Jiaxing (leste da China, clima subtropical):

- Temperatura entre -3,2 °C e 33,6 °C: plausível para inverno e verão da região.
- Umidade entre 32% e 99%: dentro do intervalo possível (0–100%).
- Precipitação entre 0 e 152,1 mm: a maioria das sessões não tem chuva (a mediana é 0), e os valores mais altos podem ser dias de chuva forte. Nada que pareça ser anormal para a região e o tipo de clima.

Nenhum valor parece impossível, então não removemos e nem alteramos nada. Os valores mais altos de precipitação são extremos, mas verossímeis.

#### Valores negativos

A única coluna numérica com valores negativos é a temperatura (com 2052 sessões), e todos esses casos acontecem em dezembro de 2020, ou seja, no inverno. Temperaturas abaixo de zero são normais nessa época, então não parece ser um erro.

Nenhuma outra coluna (energia, custos, contagens do histórico) tem valores negativos nos seus mínimos, o que era esperado porque não faria sentido.

#### Conclusão da 1.2.4

Não encontramos inconsistências que exijam corrigir ou remover linhas. As pequenas diferenças de horário afetam apenas colunas que não irão entrar no modelo, e os valores extremos de clima (mínimos e máximos) são plausíveis para a região dos quais os dados estão sendo retirados.

### 1.2.5 Outliers nas variáveis preditoras

#### 1.2.5.1 Outliers do histórico do posto = sinal, não erro


Os valores que nós vemos de outliers acabam por vir de postos com uma proporção alta de sessões anormais recentes. No post_previous_abnormal_rate_1D acabamos por ter 6,4% de outliers que são de sessões em que o posto teve mais de 46% de sessões anormais no dia anterior.

Já no post_previous_abnormal_rate_7D conseguimos observar que 6,7% das sessões ocorreram num posto com mais de 47% de sessões anormais num período de 7 dias e com o post_previous_abnormal_rate_30D vemos que 5,7% ocorreram em um posto com mais de 49% de sessões anormais em 30 dias.

Lembrando que os três valores são rates, então dá para comparar as três janelas entre si. Se quisermos ver os valores em absoluto seria só utilizar as medições de post_previous_abnormal_1D, post_previous_abnormal_7D e post_previous_abnormal_30D. Ao utilizar esses itens teriamos o valor absoluto, X% de sessões anormais em postos com mais de Y sessões anormais.

#### 1.2.5.2 Tempo desde a última sessão do posto: cauda muito longa

Ao observarmos a variavel post_hours_since_previous_completed_session conseguimos observar uma escala extrema, sendo apenas possivel observar utilizando a logaritmica no plot. A variável tem uma mediana de 0,48h ou 29 min de tempo entre sessões completas, um máximo de 5137h ou praticamente 7 meses parado e com uma assimetria ou skew de praticamente 110, a maior em todo o dataset. 

Ou seja, a maior parte dos nossos valores são pequenos e os poucos valores extremos podem indicar que o posto em questão possa ter ficado parado devido a uma manutenção ou alguma impossibilidade que impediu os condutores de abastecer naquele local. Em geral os valores encontrados parecem verossímeis. Novamente, não é um erro, apenas uma escala extrema.


#### 1.2.5.3 Contadores do usuário: dominados pelas contas de frota

Nosso user_previous_sessions_730D tem uma mediana de 21, um p99 de 10172 e um máximo de 14566. Isso indicaria que, assim como a distribuição anterior, a distribuição é muito assimétrica, onde a ponta da direita, os maiores números de carregamentos, depende de uma ou 2 contas.

Os usuários 39 e 23 tem, em valores absolutos, 14584 e 6748 carregamentos. Uma média de 6050 sessões de carregamento, onde um usuário médio tem uma média de aproximadamente 64 carregamentos.

Na user_previous_sessions_30D um usuário chega a ter 2605 carregamentos em 30 dias, o que seria algo em torno de 87 carregamentos por dia, sendo impossível para uma pessoa carregar o carro quase que 3 vezes por dia.

É no user_active_sessions_at_start que conseguimos entender o que se passa. 95% das sessões tem valor 0, ou seja, na esmagadora maioria das vezes um usuário apenas carrega 1 carro por vez. Já o numero máximo de carregamentos por vez é 14, indicando que provavelmente se trata de carros. 

Essa ideia é confirmada quando vemos que os userIDs com numeros de sessão maiores que 2 são exatamente o 39 e 23 de antes. Isso provavelmente indica que são contas empresariais ou algo do gênero que possuem uma frota de carros. 

Logo, mantemos esses dados e como forma de vizualizar os mesmos utilizamos a escala logarítmica.


#### 1.2.5.4 Clima: sem outliers problemáticos

Temperatura e umidade tem uma assimetria de aproximadamente −0,4 e praticamente nenhum outlier no dado amostral.

Já na Precipitação temos um problema: a regra do IQR marca 19% como outlier. Porém a mediana é 0 (58% das sessões sem chuva). Pesquisando um pouco mais vê-se que o IRQ como medida não serve para variáveis cujos dados tenham muitos valos iguais a 0


#### Decisão final

Depois de analisar os outliers das variáveis preditoras, decidimos por não remover nenhuma linha do dataset. Como vimos nos pontos anteriores, todos os valores extremos acabam por ter uma explicação real e plausível. Pode ser um posto com muitas sessões anormais, um posto que ficou muito tempo parado ou uma conta que parece pertencer a uma empresa (há de se ficar atento a esse último dado na hora da modelação, visto que eles podem atrapalhar o aprendizado do madelo). Removê-los seria jogar informação verdadeira fora.

Também decidimos não limitar as variáveis de falhas do posto. É justamente nos valores altos dessas variáveis que conseguimos observar o sinal que queremos prever, como vamos ver na secção 1.4. Já para as contagens e os tempos com assimetria acima de 3, a ideia é aplicar uma transformação logarítmica (log1p) caso o modelo seja linear ou baseado em distância, como a regressão logística, o KNN ou o SVM. Nos modelos de árvore isso não é necessário, já que eles não são afetados pela escala das variáveis.

Lembrando que, se em algum momento formos limitar os dados, o limite (por exemplo o p99) deve ser calculado apenas no conjunto de treino, para não vazar informação do teste para o modelo. O enunciado pede explicitamente para este fato ser documentado, então fica aqui registrado.

Por fim, a única limitação que conseguimos observar, como dito anteriormente, é que as contas de frota podem acabar por dominar o aprendizado do modelo, já que possuem valores muito diferentes de todos os outros usuários. Ou seja, na fase de modelação vale a pena avaliar o modelo com e sem essas contas para avaliar se o resultado muda consideravelmente.

---

## 1.3 Definição da variável alvo

### 1.3.1 Disponibilidade das variáveis no início da sessão

Como a previsão deve ser feita no início da sessão, cada variável foi classificada segundo a sua disponibilidade nesse momento. As variáveis registadas durante ou após a sessão foram excluídas dos preditores para evitar o que se foi pedido no enunciado ("The prediction must represent a decision made when a charging session starts. Consequently, only information that would reasonably be available at that time may be used as a predictor"), sendo usadas apenas na análise exploratória.

### 1.3.2 electricity_price × tariff_period

A variável electricity_price tem só 3 valores (0,38 / 0,90 / 1,21). Na prática é categórica e repete a informação do tariff_period. Logo são informações redundantes por definição das mesmas. Ou seja, caso usemos uma não precisamos da outra.


### 1.3.3 Tabela de classificação das colunas

Para cada uma das 48 colunas (as 47 originais mais a is_Abnormal que criámos) fizemos a mesma pergunta: "este valor já é conhecido quando a sessão começa?". A resposta fica na coluna 'disponibilidade' da tabela e, a partir dela, já a coluna 'uso' indica o que vamos fazer com cada variável. No fim ficamos com cinco grupos:

| Disponibilidade | Uso | Colunas | Porquê |
|---|---|---|---|
| Início | Feature | localização, hora de início, clima, tariff_period e histórico do posto e do usuário | já são conhecidas quando a sessão começa |
| Depois do fim | Só EDA | energia, custos, hora de fim, pagamento e End cause | só existem depois de a sessão terminar |
| Discutível | A avaliar | UserID e Charging Post ID | conhecidos no início, mas o modelo pode decorar IDs |
| Redundante | Não usar | Order creation time e electricity_price | repetem a informação do Start Time e do tariff_period |
| Alvo | Alvo | is_Abnormal | é o que queremos prever |

As variáveis do grupo "Depois do fim" são as mais importantes de deixar de fora. Se fossem usadas, o modelo estaria a "ver o futuro" e teria um desempenho artificialmente alto. Por isso ficam no dataset apenas para a análise exploratória (EDA, do inglês Exploratory Data Analysis).

Já os IDs de usuário e de posto estão disponíveis no início, mas podem levar o modelo a decorar usuários e postos específicos. Também não ajudam em nada quando aparece um usuário ou posto novo, visto que não existe histórico na primeira vez. Como o enunciado pede, a decisão vai ser tomada comparando modelos com e sem essas colunas.

O Order creation time ficou de fora porque em 56% das sessões é registado depois do Start Time (mediana de 55 s) e não traz nenhuma informação nova. Já o electricity_price tem só 3 valores e cada tariff_period tem sempre o mesmo preço, então sabendo o período já sabemos o preço. O contrário não acontece, porque o mesmo preço aparece em vários períodos, por isso ficamos com o tariff_period, que tem mais informação.

Por fim, a categoria "?" do código serve de verificação: se alguma coluna caísse aí, significaria que nos esquecemos de a classificar. No nosso caso nenhuma coluna ficou por classificar.

### 1.3.4 Listas reutilizadas nas próximas etapas

A partir da tabela, guardamos três listas que vão ser reutilizadas nas próximas etapas:

- features_inicio: colunas candidatas a preditores;
- colunas_depois: colunas que nunca entram no modelo, só na análise exploratória;
- colunas_ids: identificadores para se testar com e sem sua presença, na fase de modelação.

Assim, a separação entre o que pode e o que não pode entrar no modelo fica definida num único local e não precisa ser repetida à mão depois.

A célula seguinte mostra, com um exemplo concreto, porque é que as colunas pós-sessão têm de ficar de fora.

### 1.3.5 Exemplo: 0 kWh "entrega" a resposta

Exemplo de problema nos dados: das 54621 sessões com 0 kWh carregados, 83,4% são anormais. Nas sessões que contem energia carregada (> 0 kWh), só 9,4% são anormais.

Faz sentido se pensarmos que a sessão que falha logo no início, não chega a carregar energia. Mas a quantidade de energia só é conhecida depois de uma sessão terminar. Se esta coluna entrasse no modelo, ele aprenderia a regra "0 kWh → anormal" e teria ótimos resultados no teste, mas seria inútil na prática, porque no momento da previsão esse valor ainda não existe, estariamos apenas jogando lixo para dentro de nossos dados.

Isto confirma, com evidência, e justifica a decisão de excluir dos preditores todas as colunas classificadas como "Depois do fim".

Já as sessões com 0 kWh, que na secção 1.2.4 tinham sido mantidas por enquanto, continuam no dataset. Elas não são erros, mas sim sessões reais que falharam antes de carregar energia, e cerca de 45,5 mil delas são anormais, o que representa mais da metade de todas as anomalias do dataset (cerca de 82 mil). Se removêssemos essas linhas, estaríamos apagando justamente a maior parte dos casos que o modelo precisa aprender a prever. O problema estava apenas na coluna de energia, que denuncia a resposta, e como ela já foi excluída dos preditores, as linhas podem ficar sem trazer nenhuma fuga de informação.

---

## 1.4 Relação das variáveis com o alvo

### 1.4.1 Histórico do posto (heatmap 1)

- Forte com o alvo: as taxas de anomalia do posto: rate_1D 0,45, rate_7D 0,43, abnormal_1D 0,37, rate_30D 0,37.
- Fraco ou nulo: número de sessões (7D 0,07, 30D 0,01) e horas desde a última sessão (0,01).
- Negativo: hours_since_previous_completed_abnormal −0,12. Quanto mais tempo desde a última falha, menor o risco.
- Redundâncias: sessions_7D × 30D 0,90; abnormal_1D × rate_1D 0,85; rate_7D × rate_30D 0,82.

Do heatmap 1 conseguimos tirar a informação de que, umm posto que falhou recentemente, tende a falhar de novo. A taxa de anomalias do posto no último dia (rate_1d) é a variável que mais se correlaciona com o alvo (0,45), seguida da taxa de anomalias do posto na ultima semana (rate_7D) (0,43). 

Sendo que é visivel que essa correlação cai a medida que a janela de tempo aumenta. Sendo assim o histórico mais recente é o que melhor informa e prediz uma proxima falha no posto. Sendo o inverso tambem informativo, onde quanto mais tempo passa desde a ultima falha, menor o risco (hours_since_previous_completed_abnormalcom uma taxa de −0,12). 

### 1.4.2 Histórico do usuário (heatmap 2)

- Forte com o alvo: só as taxas: rate_30D 0,30, rate_180D e rate_730D 0,28.
- Nulo: contagens de sessões e de falhas (≈ −0,02 a 0,01) e is_new_user 0,02.
- Redundância extrema:
  - abnormal_730D × sessions_730D 1,00;
  - abnormal_180D × sessions_180D 0,99;
  - is_new_user × user_no_previous_completed_session 0,99;
  - rate_180D × rate_730D 0,97.

Do heatmap 2 vemos que, assim como no posto, o que se relaciona com a falha é a taxa de anomalias do usuário, e não a quantidade de sessões ou de falhas. A taxa dos últimos 30 dias (rate_30D) é a mais forte (0,30), seguida das taxas de 180 e 730 dias (0,28). Já as contagens ficam praticamente em zero, ou seja, um usuário que carrega muito não falha necessariamente mais. Mesmo assim, o histórico do usuário informa menos que o do posto (0,30 contra 0,45).

Tambem é visivel que muitas colunas são praticamente duplicadas, como abnormal_730D e sessions_730D (1,00) e is_new_user e user_no_previous_completed_session (0,99), então não faz sentido manter todas no modelo. Essa correlação de 1,00 entre falhas e sessões é estranha, e provavelmente acontece porque algumas contas com milhares de sessões dominam o cálculo de Pearson (Na próxima etapa iremos verificar melhor o que pode estar acontecendo).

### 1.4.3 Contexto da sessão — clima, preço e sessões ativas do usuário no momento (heatmap 3)

- Tudo ≈ 0 (temperatura 0,03; os demais entre −0,01 e 0,01).

Do heatmap 3 vemos que todas as variáveis de contexto ficaram perto de zero, sendo a temperatura a maior com apenas 0,03. Sendo assim, sozinhos, o clima, o preço e o número de sessões ativas do usuário não explicam a anomalia. Mas como Pearson só mede relação linear, essas variáveis ainda podem ajudar o modelo quando , quem sabe, combinadas com outras variáveis.

### 1.4.4 Correlação de Pearson da localização e do período tarifário com is_Abnormal

#### 1.4.4a Localização (heatmap 4a)

- Maior risco: Expressway C (0,11), Shopping Mall (0,07), Expressway A (0,07), Wholesale Market (0,06).
- Menor risco: Park B (−0,10), Bus Station (−0,09), Technology Park (−0,06).
- Taxa real: vai de 7,9% (Industrial Park) a 32,4% (Expressway A).

Do heatmap 4a conseguimos ver que a localização do posto importa. As áreas de serviço das vias expressas (Expressway C com 0,11 e Expressway A com 0,07) e o do Shopping Mall (0,07) são as que mais se relacionam com falhas, enquanto Park B (−0,10) e Bus Station (−0,09) têm menos risco. Os valores de correlação parecem baixos porque cada localização é só uma pequena parte das sessões, mas olhando a taxa real fica mais claro: vai de 7,9% no Industrial Park até 32,4% na Expressway A, ou seja, as áreas da autoestrada têm cerca de 4× mais falhas que os parques. Não da para saber o porque disso ocorrer, mas é de fato interessante ter uma correlação como essas presente.

#### 1.4.4b Período tarifário (heatmap 4b)

- Tudo proximo de 0 (entre −0,016 e 0,02). A taxa só varia entre 16,7% e 20,5%.

Do heatmap 4b vemos que o período tarifário quase não influencia na falha, já que todas as correlações ficam perto de zero (entre −0,016 e 0,02) e a taxa de anomalia só varia entre 16,7% e 20,5%. É uma variável fraca, mas pode ser mantida no modelo, já que pode ajudar junto com outras variáveis.

### 1.4.5 Eliminação de variáveis redundantes

Seguindo os critérios discutidos com a professora, agora eliminamos as variáveis muito correlacionadas entre si. Quando duas variáveis têm uma correlação de Pearson maior que 0,9 entre elas, ficamos com a que tem maior correlação com o alvo (is_Abnormal) e eliminamos a outra. A justificativa é que, se duas variáveis trazem praticamente a mesma informação, manter as duas não acrescenta nada ao modelo e só torna as próximas análises mais pesadas, tanto quantitativamente quanto visualmente para nós usuários.

A correlação com o alvo é comparada em valor absoluto, porque o que interessa é a força da relação e não o sinal (algumas contagens têm correlação negativa com o alvo).

Os heatmaps arredondam para 2 casas decimais, o que não chega para decidir os casos no limite. Por isso os valores abaixo foram calculados com 4 casas, a partir das mesmas matrizes dos heatmaps.

#### Heatmap 1

Nenhum par passa de 0,9. O mais alto é sessions_7D × sessions_30D, que aparece como 0,90 no heatmap e vale 0,8987. Mesmo não chegando a 0,9, decidimos eliminar uma delas, porque o valor está tão perto do limite que, na prática, as duas trazem a mesma informação. Aplicando o mesmo critério, fica sessions_7D, que tem maior correlação com o alvo (0,0678 contra 0,0143), e sai post_previous_sessions_30D.

Os outros pares altos ficam claramente abaixo do limite (abnormal_1D × rate_1D 0,85; rate_7D × rate_30D 0,82), por isso as restantes variáveis do posto são mantidas.

#### Heatmap 2

Aqui existem cinco pares acima de 0,9:

| Par | Correlação entre elas | Correlação de cada uma com o alvo | Fica | Sai |
|---|---|---|---|---|
| sessions_730D × abnormal_730D | 0,9974 | −0,0206 / −0,0124 | sessions_730D | abnormal_730D |
| is_new_user × user_no_previous_completed_session | 0,9946 | 0,0195 / 0,0206 | user_no_previous_completed_session | is_new_user |
| sessions_180D × abnormal_180D | 0,9880 | −0,0241 / −0,0089 | sessions_180D | abnormal_180D |
| rate_180D × rate_730D | 0,9710 | 0,2821 / 0,2796 | rate_180D | rate_730D |
| sessions_30D × abnormal_30D | 0,9455 | −0,0201 / 0,0148 | sessions_30D | abnormal_30D |

O par sessions_30D × abnormal_30D (0,95 no heatmap) faltava na nossa primeira lista. Já rate_180D e rate_730D pareciam empatadas (0,28 com o alvo), mas com 4 casas a de 180 dias fica ligeiramente à frente (0,2821 contra 0,2796).

Em quatro dos cinco pares as duas variáveis têm correlação quase nula com o alvo (abaixo de 0,03 em valor absoluto), então a escolha de qual fica pouco muda. Mesmo assim seguimos o critério à risca.

Do histórico do usuário saem 5 variáveis: user_previous_abnormal_730D, user_previous_abnormal_180D, user_previous_abnormal_30D, user_previous_abnormal_rate_730D e is_new_user.

#### Heatmap 3

O par com maior correlação tem só 0,35 e, logo, todas as variáveis de contexto são mantidas.

#### Resumo

No total saem 6 variáveis: 1 do histórico do posto (post_previous_sessions_30D) e 5 do histórico do usuário (user_previous_abnormal_730D, user_previous_abnormal_180D, user_previous_abnormal_30D, user_previous_abnormal_rate_730D e is_new_user).


### 1.4.6 Outliers contra o is_Abnormal

Na secção 1.2.5 decidimos manter os outliers das variáveis preditoras, e aqui fomos verificar se eles realmente carregam informação sobre o alvo. Para isso, em cada variável separamos as sessões que ficam acima do limite do IQR (Q3 + 1,5 × IQR) e comparamos a taxa de anomalia delas com a do resto das sessões.

No histórico do posto a diferença é muito grande. Nas sessões em que o posto teve uma taxa de anomalias no dia anterior (post_previous_abnormal_rate_1D) fora do normal, 75% são anormais, contra apenas 14,7% no resto. O mesmo acontece com o número de falhas do posto na última semana (post_previous_abnormal_7D), com 55,3% contra 14,8%, e nos últimos 30 dias (post_previous_abnormal_30D), com 45,7% contra 15,7%. Sendo assim, os outliers do posto não são erros, mas sim os casos que parecem mais informar sobre uma próxima falha, ou seja, se removêssemos ou limitássemos esses valores estaríamos apagando justamente a informação mais preditiva que temos. Quando um posto começa a falhar, obviamente as sessão naquele dia vão sofrer com mais anomalias de carregamento.

No histórico do usuário o padrão se repete nas taxas de anomalia. Os outliers das taxas de 30, 180 e 730 dias têm entre 45,6% e 46,9% de sessões anormais, contra cerca de 16% no resto. Ou seja, a taxa de falhas do usuário também tem sinal, embora mais fraco que a do posto (cerca de 46% contra 75%).

Já no número de sessões do usuário em 730 dias (user_previous_sessions_730D) acontece o contrário, onde os outliers têm até um pouco menos de anomalia (15,4% contra 19%). Esses outliers são as contas com milhares de sessões que vimos na secção 1.2.5.3, que provavelmente pertencem a frotas, e a taxa de anomalia delas é parecida com a do resto, então não carregam nenhum sinal especial. Mesmo assim, são sessões reais e por isso também ficam no dataset.

Com isso confirmamos a decisão da secção 1.2.5 de não remover nem limitar os outliers, já que nas variáveis de falhas eles são exatamente o sinal que queremos que o modelo aprenda.


---

## 1.5 Estabilidade temporal

Nesta secção a ideia principal era verificar se os dados se mantêm estáveis ao longo do tempo ou se o comportamento das sessões muda de um período para o outro, já que, se os dados mudarem muito, um modelo treinado num período pode não funcionar tão bem noutro. Os dados cobrem exatamente 25 meses e vão de 31/12/2019 a 31/12/2021, sendo que dezembro de 2019 tem só 17 sessões, visto que os registos começam às 21h do último dia do mês, por isso não o consideramos como um mês nas análises mensais.

A primeira coisa que conseguimos ver é que o volume de sessões cresce bastante, passando de cerca de 6 mil para cerca de 27 mil sessões por mês, e 2021 acaba por ter mais dados que 2020 (255 mil contra 186 mil sessões). A exceção é fevereiro, que cai nos dois anos (as 2 grandes quedas que acontecem no primeiro gráfico "Número de sessões por mes"), chegando a só 1 391 sessões em fev/2020, o que pode estar possivelmente ligado ao Ano Novo Chinês e, em 2020, ao início da COVID, mas os dados não nos permitem confirmar isso. Junto com o volume cresce também a base de usuários, que vai de cerca de 2 mil (2020-01) para cerca de 7,5 mil (2021-10) usuários ativos por mês, lembrando que 43% dos usuários têm uma única sessão em todo o dataset. Já o número de postos fica estável em 91–92 durante quase todo o período e só cai para 86–88 no fim de 2021, porque 4 postos (17, 18, 19 e 20) param de ter sessões em 28/09/2021 e outros param em dezembro. Não dá para saber se foram desativados ou se entraram em manutenção, mas isso explica a queda.

A taxa de anomalia também não é constante. Ela começa em cerca de 25%, cai para 16–17% no fim de 2020, chega ao mínimo de 13,7% em mar/2021 e volta a subir para cerca de 23% em set/2021. Curiosamente, olhando por ano a taxa quase não muda (18,7% em 2020 e 18,5% em 2021), ou seja, a variação só aparece quando olhamos mês a mês. As variáveis de histórico acompanham essa mudança da forma que esperávamos: a média mensal da taxa de falhas do posto em 7 dias vai de cerca de 0,23 para 0,15 e volta para 0,22, e a do usuário em 30 dias faz o mesmo, porém de forma mais suave (entre 0,17 e 0,25), o que bate com o que vimos na secção 1.4, onde o histórico do usuário informa menos que o do posto.

Já a percentagem de novos usuários é muito alta no início (94% em dez/2019 e 31% em jan/2020), o que faz sentido, visto que no começo dos registos quase todos os usuários aparecem pela primeira vez, e depois estabiliza em 10–15% por mês. Esses novos usuários têm um pouco mais de anomalia que os restantes (20,6% contra 18,3%). Ou seja, os primeiros meses são um "início frio", onde o histórico dos usuários e dos postos ainda está sendo construído e as variáveis de histórico têm pouca informação. A média de sessões anteriores do usuário em 730 dias cresce de forma natural, já que a janela vai acumulando sessões, mas tem um pico estranho em jun/2020 (626) seguido de uma queda em ago/2020 (132). Como a média é calculada sobre as sessões de cada mês, isso provavelmente acontece nos meses em que as contas grandes (os usuários 39 e 23, que somam cerca de 21 mil sessões, ou seja ≈5% do total) carregam muito e puxam a média para cima. São essas mesmas contas que explicam a correlação de 1,00 entre user_previous_sessions_730D e user_previous_abnormal_730D no heatmap 2. Por fim, a temperatura segue o ciclo normal das estações, com cerca de 6 °C no inverno e 30 °C no verão, e a anomalia sobe um pouco nos verões (ago/2020 e set/2021), mas como vimos na secção 1.4.3 a correlação linear com o alvo é fraca (0,03), então existe uma sazonalidade, mas sozinha ela não explica a anomalia.

De forma geral, o volume de sessões, a taxa de anomalia mensal e a base de usuários mudam ao longo do tempo, ou seja, os dados não parecem ser estáveis, e isso traz algumas consequências para a modelação. A divisão entre treino e teste tem de ser temporal, treinando com o passado e testando com os meses finais, já que se fizéssemos uma divisão aleatória estaríamos misturando sessões do futuro no treino e o resultado seria otimista demais. Como a taxa de anomalia do período de teste pode ser diferente da do treino, também não vamos avaliar o modelo só pela acurácia, mas por métricas como F1, PR-AUC e o recall da classe anormal. Temos ainda de ter atenção ao início frio, podendo descartar jan–fev/2020 do treino ou, pelo menos, citar isso como uma limitação, e às contas grandes, que podem dominar o aprendizado do modelo, por isso a ideia é verificar o erro com e sem elas. Por fim, como alguns postos param no fim de 2021, o período de teste vai ter menos postos que o de treino, o que também é uma limitação a citar.


### 1.5.1 Decisões para a Task 2

- **Início frio:** as sessões de dez/2019 e jan/2020 vão ser retiradas do treino. Nesses meses o histórico dos usuários e dos postos ainda está sendo construído (94% e 31% de usuários novos, contra 10–15% nos outros meses), então as variáveis de histórico têm pouca informação. São cerca de 6 200 sessões (~1,4% do total) das mais de 400mil que temos no total.
- **Fevereiro:** a queda de sessões em fev/2020 não será removida. A mesma queda acontece em fev/2021, então parece ser um evento que se repete todo ano, muito mais provavelmente ligado ao Ano Novo Chinês do que à COVID como antes idealizado. Além disso, a taxa de anomalia desse mês (26,3%) é parecida com a de janeiro (25,0%) e março (24,8%).
- O corte é feito só no treino, na Task 2. O CSV exportado no notebook manterá todas as sessões não modificadas.

---

# Task 2 — Definição do problema de previsão e da avaliação

## 2.1 Definição do problema

Quanto à unidade de observação, cada linha do dataset é uma sessão de carregamento, ou seja, o modelo faz uma previsão por sessão e temos 441 077 sessões no total. Quanto às classes, a classe positiva é a sessão anormal (is_Abnormal = 1) e a negativa é a sessão normal (is_Abnormal = 0). São 81934 sessões anormais (18,6%) contra 359143 normais (81,4%).

Sobre como o alvo sai do End cause, como foi escolhido na Task 1, decidimos por interpretar as sessões que terminaram com "Charging ends normally" ou "User stops charging" como sendo sessões normais e as que terminaram com qualquer um dos outros 13 motivos, todos falhas (do posto, da comunicação com o veículo, do BMS, etc), são as consideradas sessões anormais.

O momento da previsão é o início da sessão (Start Time). Só pode ser usado o que já se sabe naquele momento, ou seja, variáveis que não sabemos no momento do inicio, não serão consideradas nesse ponto. A previsão serve para monitoramento automático e priorização da manutenção dos postos. Prever uma sessão como anormal significa que a sessão tem risco alto de terminar com falha, então o sistema gera um alerta para o operador acompanhar a sessão e o posto sobe na fila de inspeção. O alerta não bloqueia a sessão nem avisa o usuário, funciona apenas como um alerta ao usuário do carro para que este fique atento.

Consideramos o "User stops charging" como normal porque foi o próprio usuário que decidiu parar, e não uma falha técnica. Mas isso acaba por ser uma limitação do proprio dataset e como a informação ali existe, visto que alguns usuários podem ter parado justamente porque algo estava errado (um carregamento lento, por exemplo), e os dados não nos permitem separar esses casos para uma previsão mais precisa. Infelizmente esse é um problema que não há como contornar. 

---

## 2.2 Variáveis disponíveis no momento da previsão

Para cada uma das 47 colunas do dataset original fizemos a mesma pergunta da Task 1: "este valor já é conhecido quando a sessão é iniciada?".

| Categoria | Colunas | Porquê |
|---|---|---|
| 1. Preditoras (29) | Location Information, District Name, Start Time, clima (3), tariff_period, histórico do posto (12), histórico do usuário (9) e user_active_sessions_at_start | já são conhecidas quando a sessão começa |
| 2. Só para o alvo (1) | End cause | é de onde sai o is_Abnormal |
| 3. Geradas durante ou depois da sessão (7) | Transaction power/kwh, Electricity cost, Service charge, Transaction Amount, Actual Payment, End Time e Payment time | só existem depois de a sessão terminar |
| 4. Outro motivo (10) | Order creation time e electricity_price | repetem a informação do Start Time e do tariff_period |
| | post_previous_sessions_30D, user_previous_abnormal_730D, user_previous_abnormal_180D, user_previous_abnormal_30D, user_previous_abnormal_rate_730D e is_new_user | correlação acima de 0,9 com outra variável |
| | UserID e Charging Post ID | o modelo pode decorar IDs |

O Start Time entra como preditora, mas não como data, e sim para tirar a hora e o dia da semana. O End cause é praticamente a própria resposta e só é conhecido no fim da sessão, então nunca poderia ser preditor.

---

## 2.3 Consequências dos erros

Um falso positivo (FP) é um alerta numa sessão que seria normal. O custo é baixo (um usuário fica de olho numa sessão sem necessidade ou um técnico verifica um posto que estava bom), mas se acontecer demais os operadores passam a ignorar os alertas. Já um falso negativo (FN) é uma sessão anormal sem alerta. O usuário fica sem carregar e o posto com problema continua a ser usado sem ninguém saber.

Para o uso escolhido, o FN é o erro mais importante. Como vimos durante a Task 1, um posto que falhou recentemente tende a falhar novamente (a taxa de falhas do posto no último dia é a variável mais correlacionada com o alvo, 0,45), ou seja, deixar passar uma falha acaba por afetar também os próximos usuários daquele posto além da não manutenção necessária do mesmo. Mesmo assim o FP não deve ser ignorado: em cada 1 000 sessões existem  cerca de 186 sessões que são anormais, e um modelo que desse alerta em todas apanharia as 186 mas geraria cerca de 814 alertas falsos. Logo, a avaliação deve dar mais peso ao recall sem deixar a precision cair demais.

Também consideramos a intervenção no usuário (avisar para trocar de posto), mas aí os falsos positivos passariam a ser o erro mais caro, já que cada alerta falso poderia encomodar um cliente real. Por isso fica apenas como uma possível extensão para alertas com probabilidade muito alta.

---

## 2.4 Métricas de avaliação

Um modelo que prevê sempre normal acertaria cerca de 81% das sessões, mas teria recall de 0%. É o accuracy paradox, ou seja, com  aproximadamente 18% de anomalias a acurácia mede mais o desbalanceamento do período do que a qualidade do modelo, e por isso não vai ser usada.

A métrica principal seria a AUC-PR. Pois ela é indicada quando a classe positiva é minoritária e quando os falsos positivos e falsos negativos acabam por ter custos diferentes, que são os nossos dois casos em questão. Como não depende do threshold, serve para comparar modelos e escolher hiperparâmetros antes do threshold acabar por ser definido. A limitação fianel é que o valor de um modelo sem skill é igual à taxa de anomalia, que muda de fold para fold, então cada fold acaba por ter que ser comparado com o seu próprio baseline.

**Métricas complementares:**

| Métrica | Porquê | Limitação |
|---|---|---|
| Recall | controla os FN, o nosso erro mais grave | alertar tudo dá 100% |
| Precision | controla os alertas falsos | sozinha também acaba por enganar |
| F1 | junta precision e recall | dá o mesmo peso aos dois erros |
| Balanced accuracy | corrige o desbalanceamento das classes | dá a mesma importância às duas classes |
| MCC | usa as 4 células da matriz de confusão e é forte contra o desbalanceamento que vemos em nosso dataset | acaba por ser menos intuitivo |
| ROC-AUC | mede a separação entre as classes | otimista demais quando as classes são desbalanceadas |

Para o threshold, fica o de maior precision entre os que garantem um recall mínimo (por exemplo, 70%), escolhido com as 3 validações juntas e nunca com o teste. Como métricas operacionais usamos os alertas corretos, alertas falsos e anomalias perdidas a cada 1 000 sessões.

Para a estabilidade entre folds usamos o desvio-padrão da AUC-PR e o pior fold. nas aulas vimos o teste de Wilcoxon para comparar modelos nos folds, mas com só 3 folds ele nunca consegue dar p < 0,05 (o menor p possível acaba sendo 0,25), então nos folds a comparação vai ser feita pela média, pelo desvio e por quantos folds cada modelo ganha. No teste final, como são cerca de 88 mil sessões, aí sim daria para utilizar o teste de McNemar.

---

## 2.5 Estratégia de validação temporal

Não usamos k-fold aleatório, porque ele misturaria sessões do futuro no treino. Primeiro, com os dados ordenados por data, separamos os 80% mais antigos para treino e os 20% mais recentes para teste. Depois, os folds são feitos só dentro dos 80% de treino, com rolling-origin, como indicado no enunciado do projeto, onde o treino tem sempre o mesmo tamanho e desliza para a frente junto com a validação.

Para isso criamos uma função (dividir_dados) que faz o corte 80/20 e depois usa o TimeSeriesSplit para dividir o treino em folds. Assim nenhuma data é escolhida à mão e a mesma função pode ser usada em qualquer outro dataset.

| Conjunto | Treino | Validação | Sessões na validação | Anomalias |
|---|---|---|---|---|
| Fold 1 | 01/04/2020–13/01/2021 | 13/01–03/05/2021 | 52 929 | 8 358 |
| Fold 2 | 14/07/2020–03/05/2021 | 03/05–18/07/2021 | 52 929 | 10 416 |
| Fold 3 | 22/09/2020–18/07/2021 | 18/07–22/09/2021 | 52 929 | 11 134 |
| Teste final | as 180 000 sessões antes do teste | 22/09–31/12/2021 | 88 216 | — |

O teste fica com exatamente 20% das sessões (88 216 de 441 077) e começa em 22/09/2021. As validações ocupam os últimos 45% do treino, divididos em 3 folds com o mesmo número de sessões. Escolhemos 3 folds porque assim cada validação fica com mais de 8 mil anomalias. Com mais folds cada validação ficaria menor, e aumentar a parte do treino usada nas validações faria o primeiro treino começar mais cedo, justamente no período em que o histórico ainda está se formando.

Cada treino tem 180 000 sessões, que é mais ou menos o tamanho do primeiro treino. Com isso o treino do fold 1 começa em abr/2020 e os meses de dez/2019 e jan/2020, identificados como "outliers" na Task 1 (94% e 31% de usuários novos, respectivamente), ficam de fora de todos os folds.

Como vimos na secção 1.5 da Task 1, a taxa de anomalia muda bastante ao longo do tempo (podendo variar de 25% para 13,7% e de volta para 23%). Com o rolling o modelo aprende só com o passado recente e os treinos ficam todos com o mesmo tamanho. As 180 000 sessões correspondem a cerca de 9 a 10 meses, ou seja, quase um ciclo inteiro de estações. A desvantagem é deixar de usar os meses mais antigos.

Como os folds são feitos pelo número de sessões, todas as validações têm o mesmo tamanho, mas não a mesma duração (cerca de 3,6, 2,5 e 2,2 meses), porque o volume de sessões cresce ao longo de 2021. As validações não se sobrepõem entre si, então cada sessão de 13/01/2021 até 22/09/2021 é avaliada uma única vez apenas.

Tudo o que o modelo aprende dos dados (scaler, one-hot, limite de outliers, etc) é ajustado só no treino de cada fold e depois apenas aplicado na validação, que fica sempre com a distribuição original. O log1p não aprende nada dos dados, por isso pode ser aplicado em tudo.

O teste foi separado logo no início e não entra em nenhuma decisão. Modelos, parametros e threshold são escolhidos só com os 3 folds. No fim, o modelo final é treinado nas 180 000 sessões antes do teste e avaliado uma única vez, sem voltar atrás caso o resultado não seja bom, senão o teste acabaria por virar mais uma validação e o resultado deixaria de ser imparcial.

---

## 2.6 Agregação entre folds

Cada modelo vai ter um resultado por fold, e precisamos decidir como juntar esses 3 valores num resumo. As formas possíveis são a média simples, a média ponderada pelo tamanho da validação e a pooled, onde juntamos as 3 validações numa só e calculamos a métrica uma única vez.

Como as 3 validações têm o mesmo número de sessões, a média ponderada acaba por dar exatamente o mesmo valor que a média simples. Já na pooled os folds 2 e 3, que têm mais anomalias, acabam por pesar mais do que o fold 1. Na média simples cada período conta o mesmo.

Logo, para comparar os modelos vamos usar a média simples da AUC-PR, já que queremos que o modelo funcione bem em todos os períodos e não só no maior. Junto com ela vamos sempre mostrar o desvio-padrão, o pior fold e o valor de cada fold, porque a média sozinha pode esconder muita variação entre os períodos. Também vamos contar em quantos dos 3 folds o modelo passa da regra, visto que um modelo que ganha na média mas perde em algum fold não é confiável. Já a forma pooled vai ser usada para escolher o threshold e para as métricas operacionais por 1 000 sessões, já que aí faz sentido juntar todas as sessões de validação.

Quanto à evolução no tempo, a taxa de anomalia sobe de 15,8% no fold 1 para 19,7% e 21,0% nos folds 2 e 3. Como a AUC-PR de um modelo sem skill acompanha essa taxa, um modelo pode parecer que melhora ao longo do tempo só porque o período tem mais anomalias, e por isso a evolução no tempo tem que ser lida sempre em relação ao baseline de cada fold.

---

## 2.7 Baseline

Para sabermos se um modelo é bom precisamos de uma régua para comparação, e por isso definimos dois baselines, ambos medidos com a AUC-PR, que é a nossa métrica principal.

| Fold | Taxa de anomalia | No-skill (AUC-PR esperada) |
|---|---|---|
| 1 | 15,8% | 0,158 |
| 2 | 19,7% | 0,197 |
| 3 | 21,0% | 0,210 |

O primeiro é o no-skill, ou seja, o mesmo modelo que prevê sempre normal que usamos no teste da acurácia. Como ele não ordena as sessões de nenhuma forma, a sua AUC-PR acaba por ser exatamente a taxa de anomalia de cada fold, como já tínhamos comentado nas métricas. Qualquer modelo que não passe desses valores não aprendeu nada.

O segundo é uma regra simples, onde usamos diretamente a taxa de falhas do posto no dia anterior (post_previous_abnormal_rate_1D), que foi a variável mais correlacionada com o alvo (is_Abnormal) na Task 1 (0,45), como pontuação de risco. Ou seja, quanto mais o posto falhou ontem, maior o risco da sessão de hoje ser anormal. Ela não precisa de treino nem de threshold, por isso é uma régua simples de calcular em cada fold.

Logo, a régua que realmente importa é a regra e não o no-skill. Se o modelo final não passar claramente da regra, significa que as outras variáveis (clima, localização, histórico do usuário, etc.) não estão a acrescentar nada ao que o histórico do posto já diz sozinho. A comparação tem sempre que ser feita fold a fold, visto que tanto a taxa de anomalia quanto o resultado da regra mudam de um período para outro.

---

## 2.8 Protocolo de avaliação

A fim de fecharmos a Task 2, juntamos aqui tudo o que foi decidido. Este protocolo fica definido antes de qualquer modelo ser treinado, ou seja, não vamos mudar estas regras depois de ver os resultados que teremos na Task 3.

| Item | Decisão |
|---|---|
| Problema | prever, no início de cada sessão (Start Time), se ela vai ser anormal (is_Abnormal = 1) ou normal (0) |
| Uso | monitoramento automático e priorização da manutenção dos postos |
| Preditores admitidos | as 28 colunas do CSV tratado (local, clima, tariff_period, histórico do posto e do usuário, user_active_sessions_at_start) e o Start Time, de onde vamos tirar a hora e o dia da semana |
| Excluídos | End cause (só serve para o alvo), as 7 colunas geradas durante ou depois da sessão, Order creation time e electricity_price (redundantes) e as 6 variáveis com correlação acima de 0,9 |
| A avaliar na modelação | UserID e Charging Post ID, comparando o modelo com e sem eles |
| Divisão temporal | 80% treino e 20% teste, por ordem cronológica; rolling-origin só dentro do treino, com treinos de 180 000 sessões e 3 folds com validações de 52 929 sessões (13/01 a 22/09/2021); teste final de 22/09 a 31/12/2021, usado uma única vez |
| Pré-processamento | scaler, one-hot, limite de outliers, etc., ajustados só no treino de cada fold |
| Erro mais grave | o FN (anomalia não detetada), sem ignorar o FP (alertas falsos demais) |
| Métrica principal | AUC-PR |
| Métricas complementares | recall, precision, F1, balanced accuracy, MCC e ROC-AUC (a acurácia não é usada) |
| Threshold | o de maior precision entre os que garantem um recall mínimo, escolhido com as 3 validações juntas |
| Métricas operacionais | alertas corretos, alertas falsos e anomalias perdidas a cada 1 000 sessões |
| Baselines | no-skill (sempre normal, AUC-PR = taxa de anomalia do fold) e regra com a post_previous_abnormal_rate_1D |
| Agregação | média simples da AUC-PR para comparar modelos, sempre com o desvio-padrão, o pior fold e quantos folds o modelo ganha da regra; pooled para o threshold e as métricas operacionais |
| Comparação estatística | nos folds pela média, desvio e folds ganhos (o Wilcoxon não serve com 3 folds); no teste final com o McNemar |

Lembrando que um modelo só vai ser considerado bom se passar claramente da regra em todos os folds, e não só na média.

---

# Task 3 — Modelos individuais e ensemble

## 3.1 Configuração comum

Tabela 3.1 — Pré-processamento por tipo de modelo (ajustado só no treino de cada fold)

| | Regressão logística | Naive Bayes misto | Árvores (DT, RF, HistGB) |
|---|---|---|---|
| Categóricas (local, distrito, tarifa) | One-hot | Frequências com Laplace (`CategoricalNB`) | One-hot |
| `hora`, `dia_da_semana` | One-hot | Categóricas | Numéricas |
| Contagens e tempos muito assimétricos (skew > 3) | `log1p` + `StandardScaler` | `log1p` + normal por classe (`GaussianNB`) | Sem transformação |
| Restantes numéricas | `StandardScaler` | Normal por classe (`GaussianNB`) | Sem transformação |
| NaN | Máximo do treino | Máximo do treino | Mantidos (a implementação suporta-os) |


> Aulas: Aula 4, slides 30 e 64 · regra de ouro da Aula 3 · Guia 5.3 e 8.

Nesta parte da Task 3, estamos apenas reutilizando o protocolo definido na Task 2 (3 folds temporais, AUC-PR, regra e no-skill, média simples + desvio + pior fold + folds > regra).

Na Logistic Regression (LR), cada feature contribui com peso × valor, e isso acaba por ser um problema quando temos uma variável como a hora, que vai de 0 a 23: o modelo só consegue aprender um efeito monótono (o risco só sobe ou só desce ao longo do dia). Além disso, os valores mínimo e máximo da hora estão lado a lado, o que não acontece nos números (0 fica longe de 23 na numeração, mas, em relação à hora, estão lado a lado). A solução acaba por ser usar one-hot, que dá um peso independente para cada hora. Esse mesmo one-hot acaba sendo desnecessário e até contraproducente nas árvores, já que elas fazem cortes do tipo Hora >= X e conseguem separar intervalos de horas a partir da variável numérica. Com one-hot, cada corte só separaria uma hora das restantes.

Outro ponto que é bom explicar é que o log1p não causa leakage, diferente do StandardScaler. Isso se dá pela forma como as fórmulas funcionam: o log1p transforma cada valor sozinho (log(1 + x)), sem usar informação das outras sessões, por isso pode ser aplicado a qualquer momento sem gerar leakage. Já o StandardScaler, por subtrair a média e dividir pelo desvio padrão, usa informação que precisa ser extraída dos dados. Ou seja, existe um momento certo para aplicá-lo: ele é ajustado só no treino de cada fold (dentro do Pipeline) e depois aplicado à validação. Se fosse ajustado com todos os dados, a média e o desvio já incluiriam informação da validação.

E, finalmente, chegamos aos NaN, que recebem o valor máximo do treino de cada fold. Esses NaN aparecem nas colunas do tipo "horas desde o último evento" quando o evento nunca aconteceu. Caso recebessem 0, entrariam com o significado oposto: o evento teria acabado de acontecer, quando na verdade nunca aconteceu. Dessa forma, receber o valor máximo do treino nos dá uma variável utilizável e coerente (quanto maior o valor, mais tempo sem o evento), e a flag *_no_previous_* continua a separar os casos em que o evento nunca aconteceu.

---

## 3.2 Contribuição dos grupos de variáveis (ablação)


### 3.2.1 Grupos e modelos

Tabela 3.2.1 — Grupos de variáveis

| Grupo | Variáveis acrescentadas | Nº total |
|---|---|---|
| G1 — contexto e clima | `Location Information`, `District Name`, `hora`, `dia_da_semana`, temperatura, humidade, precipitação | 7 |
| G2 — G1 + tarifa | `tariff_period` | 8 |
| G3 — G2 + histórico do posto | 12 variáveis `post_*` | 20 |
| G4 — G3 + histórico do utilizador | 9 variáveis `user_*` e `user_active_sessions_at_start` | 30 |

Por que acabamos por escolher, em princípio, 2 modelos para a ablação e não apenas um? Além de serem modelos com princípios diferentes. A ideia principal era a de verificar se os nossos resultados dependem ou não dos modelos que utilizamos. A ideia geral é que a nossa conclusão não dependa do tipo de modelo. Se uma variável ou um grupo de variáveis melhora os dois modelos, isso indica que o ganho vem da informação e não do modelo.

A variável user_active_sessions_at_start acabou por ficar no grupo do usuário, pois ela conta quantas sessões abertas o usuário tem no momento, descrevendo assim o usuário, e não o posto. Caso ficasse no grupo 3, esse grupo já teria informação do usuário, e a comparação entre o grupo 3 e o grupo 4 deixaria de medir quanto vale o histórico do usuário.

Já a variável UserID ficou de fora por questões de privacidade, visto que basear a previsão de falha no usuário poderia acabar por marcar usuários específicos como responsáveis pelas falhas. Já o Charging Post ID ficou de fora, pois o modelo pode decorar postos específicos, além de não ajudar na previsão para postos novos (secção 3.2.8).

### 3.2.2 Resultados

Tabela 3.2.2a — AUC-PR na validação de cada fold, por modelo e grupo de variáveis

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

Tabela 3.2.2b — Ganho de AUC-PR ao acrescentar cada grupo (diferença para o grupo anterior)

| Modelo | Variáveis acrescentadas | Fold 1 | Fold 2 | Fold 3 | Média |
|---|---|---|---|---|---|
| LR | Tarifa (G2 − G1) | +0,0000 | +0,0000 | +0,0001 | +0,0001 |
| LR | Histórico do posto (G3 − G2) | +0,2572 | +0,3263 | +0,2822 | +0,2885 |
| LR | Histórico do utilizador (G4 − G3) | +0,0258 | +0,0158 | +0,0104 | +0,0174 |
| HistGB | Tarifa (G2 − G1) | +0,0005 | +0,0016 | +0,0045 | +0,0022 |
| HistGB | Histórico do posto (G3 − G2) | +0,3048 | +0,3587 | +0,2950 | +0,3195 |
| HistGB | Histórico do utilizador (G4 − G3) | +0,0258 | +0,0187 | +0,0114 | +0,0186 |

### 3.2.3 A tarifa traz informação além da hora?

No fim das contas não, não traz. Isso porque cada hora acaba sempre por cair num único período de tarifa. Logo, o ganho que se tem de juntar a tarifa não adiciona muita diferença entre os folds, pois sabendo a hora já se sabe a tarifa. O ganho em LR é praticamente nenhum (explicado pelo fato já abordado anteriormente onde a hora aqui entra como one-hot). Já no HistGB a hora entra mesmo como número, precisando de dois cortes enquanto a tarifa já junta esses cortes em uma variável.

A tarifa acaba por ser um facilitador do trabalho da árvore, mas é apenas uma outra forma de se apresentar a mesma informação já existente na hora, não uma informação nova. Esse ponto já havia sido discutido em notebooks anteriores.

### 3.2.4 Quanto contribui o histórico do posto?

No fundo o histórico do posto acaba por ser, de longe, o grupo que mais contribui nas análises. Em relação ao acaso, os modelos passam de cerca de 1,5× para cerca de 3× (tabela 3.2.6). Ao passar do G2 para o G3, a AUC-PR sobe em média +0,29 na LR e +0,32 no HistGB (tabela 3.2.2b), enquanto a tarifa não traz quase nada e o histórico do usuário traz cerca de +0,02.

Além disso, com o G3, os dois modelos passam a ganhar da regra nos 3 folds, o que não acontecia com os grupos 1 e 2. Como a regra usa apenas a rate_1D, isso mostra que juntar o histórico completo do posto (7 e 30 dias, horas desde a última falha, entre outras variáveis) acaba por trazer mais informação além da do dia anterior.


### 3.2.5 O histórico do utilizador traz benefício adicional?

O histórico do usuário acaba por trazer um benefício pequeno, mas consistente. Ao passar do grupo 3 para o grupo 4, a AUC-PR sobe em média +0,017 na LR e +0,019 no HistGB (tabela 3.2.2b), bem menos do que o histórico do posto, mas o ganho é positivo nas 6 comparações (3 folds × 2 modelos). Porém, esse ganho acaba por diminuir do fold 1 para o fold 3 (de +0,026 para cerca de +0,01 nos dois modelos). Uma hipótese é que, com o tempo, entram mais usuários novos, sem histórico, e assim o histórico do usuário acaba por ajudar menos, mas isso não foi verificado

### 3.2.6 O efeito é consistente entre períodos?

Tabela 3.2.6 — AUC-PR a dividir pela prevalência de cada fold (quantas vezes melhor do que o acaso)

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

No geral, sim, o efeito acaba por ser consistente. Olhando a tabela 3.2.2b, todos os ganhos têm o mesmo sinal nos 3 folds, ou seja, nenhum grupo de variáveis piora o modelo em algum período. O histórico do posto e o histórico do usuário sobem a AUC-PR nos 3 folds e nos dois modelos, e a tarifa fica praticamente em zero em todos eles.

Já na tabela 3.2.6, a AUC-PR foi dividida pela prevalência de cada fold, pois a AUC-PR do acaso muda de fold para fold (15,8%, 19,7% e 21,0%), e assim temos quantas vezes o modelo é melhor do que o acaso. No fold 3, os modelos com o histórico do posto acabam por cair (o HistGB no grupo 3 passa de 3,36× para 3,05×), mas a regra também cai (de 3,06× para 2,81×). Como a regra não é treinada, isso indica que a queda vem principalmente do próprio período, e não dos modelos em si.

### 3.2.7 Preocupações operacionais e éticas

Tabela 3.2.7 — HistGB no G3, avaliado em subgrupos de sessões (o modelo é o mesmo; muda só o conjunto avaliado)

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

O histórico do posto também tem os seus problemas: falhas que vêm do carro (como o Vehicle charging faults, que tbm acaba por não ser culpa do usuário) acabam por contar contra o posto, e postos novos ou com poucas sessões têm um histórico pouco confiável. Já a tabela 3.2.7 mostra que, no G3, tirar as sessões de frota quase não muda a AUC-PR, ou seja, o modelo funciona de forma parecida para as frotas e para o resto. O limite é que são apenas 2 contas de frota.

### 3.2.8 Variáveis adicionais

Tabela 3.2.8 — HistGB no G3, com e sem o `Charging Post ID` (em one-hot)

| Modelo | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra |
|---|---|---|---|---|---|---|---|
| HistGB G3 | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| HistGB G3 + `Charging Post ID` | 0,538 | 0,661 | 0,639 | 0,613 | 0,066 | 0,538 | 3 |
| Diferença (com − sem) | +0,0001 | −0,0001 | −0,0024 | −0,0008 | — | — | — |

A hora e o dia da semana já foram justificados na secção 3.1. O Charging Post ID foi testado no HistGB com o G3 e não trouxe ganho (tabela 3.2.8), o que faz sentido, já que o histórico do posto já descreve cada posto e o ID não ajuda com postos novos. O UserID não foi testado, por questões de privacidade, e foi usado apenas para identificar as contas de frota (39 e 23).


### 3.2.9 Grupo escolhido para os modelos

Acabamos por escolher o G3 para os modelos (já separamos o G4 como o final por esse motivo). O G4 acaba por ser um pouco melhor (cerca de +0,02 de AUC-PR), mas o ganho é pequeno e traz os problemas éticos da secção 3.2.7, além de ir contra a exclusão do UserID. O protocolo da Task 2 diz quais variáveis podem ser usadas, mas não obriga a usar todas.


---

## 3.3 Modelos individuais

### 3.3.1 Algoritmos escolhidos

Tabela 3.3.1 — Princípios de modelação

| Princípio | Algoritmo | Usado |
|---|---|---|
| Linear | Regressão logística | ✅ |
| Probabilístico | Naive Bayes (versão mista) | ✅ |
| Árvores de decisão | Árvore de decisão (profundidade 6) | ✅ |
| Baseado em instâncias | kNN | — |
| Baseado em kernels | SVM | — |

Escolhemos três modelos com princípios diferentes: a regressão logística (linear), o Naive Bayes misto (probabilístico) e a árvore de decisão (árvores). Cada um acaba por olhar os dados de um jeito diferente, são rápidos de treinar e fáceis de interpretar. 

O kNN e o SVM acabram por ficar de fora devido ao tamanho dos dados: com cerca de 180 000 sessões por treino, o kNN precisaria guardar todas as sessões e comparar cada sessão nova com todas elas, e o SVM acaba por ficar muito lento para treinar. Como o próprio enunciado diz, usar muitos algoritmos sem análise não é melhor do que uma comparação menor e bem justificada.

### 3.3.2 Justificação pelos seis critérios

A regressão logística foi escolhida pela interpretabilidade, já que os coeficientes mostram o sinal e o tamanho do efeito de cada variável no risco de anomalia. 

Já o Naive Bayes misto acaba por ser o modelo mais barato, pois treinar é apenas contar frequências nas categóricas e calcular médias e desvios nas numéricas. 

Por dim a árvore de decisão consegue apanhar limiares e interações entre variáveis (por exemplo, local × histórico do posto) sem precisar de escala.

O pressuposto de independência do Naive Bayes acaba por ser violado aqui, o que aparece nas probabilidades extremas (72%, tabela 3.3.4).

### 3.3.3 Profundidade da árvore

Tabela 3.3.3 — Árvore de decisão no G3 com várias profundidades

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

### 3.3.4 Naive Bayes: versão gaussiana e versão mista

Tabela 3.3.4 — Naive Bayes no G3: versão gaussiana (antiga) e versão mista

| Versão | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra | Prob. extremas (fold 3) |
|---|---|---|---|---|---|---|---|---|
| GaussianNB (one-hot + normal em todas as colunas) | 0,444 | 0,586 | 0,566 | 0,532 | 0,077 | 0,444 | 0 | 76,8% |
| NB misto (categóricas por frequências, numéricas pela normal) | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 | 72,0% |
| NB misto sem `tariff_period` | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 | 71,9% |

Na versão gaussiana, todas as colunas eram tratadas como números com distribuição normal, até as categóricas, que só valem 0 ou 1. Por isso passamos para o NB misto, onde as categóricas usam frequências e as numéricas continuam com a normal. Com isso, a AUC-PR média subiu de 0,532 para 0,551 nas médias.

Mesmo assim, 72% das probabilidades continuam extremas (abaixo de 1% ou acima de 99%), e tirar a tarifa não muda nada. Logo, o problema acaba por ser o pressuposto de independência: as variáveis do posto estão correlacionadas e o modelo conta a mesma informação várias vezes.

OBS: Prob. extremas = percentagem de probabilidades abaixo de 1% ou acima de 99% na validação do fold 3. Na regressão logística é 0,3%.

### 3.3.5 Resultados

Tabela 3.3.5 — Modelos individuais no G3

| Modelo | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra |
|---|---|---|---|---|---|---|---|
| No-skill | 0,158 | 0,197 | 0,210 | 0,188 | 0,027 | 0,158 | — |
| Regra | 0,450 | 0,602 | 0,590 | 0,548 | 0,084 | 0,450 | — |
| Regressão logística | 0,506 | 0,636 | 0,617 | 0,586 | 0,070 | 0,506 | 3 |
| Naive Bayes misto | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 |
| Árvore de decisão (profundidade 6) | 0,503 | 0,626 | 0,606 | 0,578 | 0,066 | 0,503 | 3 |

Entre os modelos individuais, a regressão logística acaba por ser a melhor (0,586 de média), seguida da árvore (0,578) e do Naive Bayes (0,551). A LR e a árvore passam a regra nos 3 folds. Já o NB, apesar de ter média um pouco acima da regra (0,551 contra 0,548), perde para ela em 2 folds. Pelo critério do protocolo da Task 2, que pede para passar a regra em todos os folds, o NB acaba por não ser considerado um bom modelo para nossos dados.

### 3.3.6 Interpretação dos modelos


Tabela 3.3.6a — Regressão logística (G3, modelo do fold 3): os 10 coeficientes maiores em valor absoluto

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

Tabela 3.3.6b — Árvore de decisão (G3, profundidade 6, modelo do fold 3): variáveis mais importantes

| Variável | Importância |
|---|---|
| `post_previous_abnormal_rate_7D` | 0,557 |
| `post_hours_since_previous_completed_abnormal` | 0,171 |
| `post_previous_abnormal_rate_1D` | 0,149 |
| `post_previous_abnormal_rate_30D` | 0,104 |
| `Location Information` = Bus Station | 0,003 |
| Cada uma das restantes variáveis | < 0,002 |
| Soma das 4 variáveis do posto | 0,982 |

Os dois modelos concordam que o que mais pesa é o histórico do posto, o que bate com a secção 3.2.4: na árvore, as 4 variáveis do posto somam 0,982 da importância, e na LR os maiores coeficientes também são quase todos do posto. A diferença é que a LR se apoia mais na rate_1D, que já era a variável mais correlacionada com o alvo na Task 1. A árvore se apoia mais na rate_7D, que por usar 7 dias acaba por ser mais estável e dar cortes melhores.

Na LR, a rate_1D e a abnormal_1D têm sinais opostos (+0,932 e −0,804). Isso não quer dizer que mais falhas ontem baixam o risco. Como as duas variáveis estão correlacionadas, o modelo acaba por dividir o efeito entre elas, e por isso os coeficientes devem ser lidos em conjunto, e não um a um.

---

## 3.4 Modelos ensemble


### 3.4.1 Abordagens e porque podem melhorar os seus modelos base

Tabela 3.4.1 — Ensembles avaliados

| Ensemble | Princípio (Aula 5) | Modelos base | Configuração |
|---|---|---|---|
| Random Forest | Bagging + sorteio de variáveis em cada divisão | Árvores sem limite de profundidade (`min_samples_leaf=20`), cada uma numa amostra bootstrap | 100 árvores, `class_weight='balanced_subsample'` |
| HistGradientBoosting | Boosting | Árvores pequenas, treinadas em sequência | Valores por defeito, `class_weight='balanced'`, `early_stopping=False` |
| Votação soft | Combinação heterogénea e estática | Regressão logística + NB misto + árvore (profundidade 6) | Média das probabilidades; avaliada também sem o NB |

A Random Forest junta 100 árvores, cada uma treinada numa amostra diferente do treino (bagging), e faz a média das previsões. Ela acaba por reduzir a variância, que é o principal problema de uma árvore sozinha, já que pequenas mudanças nos dados mudam muito a árvore. O sorteio de variáveis em cada divisão serve para as árvores ficarem diferentes entre si. Sem ele, todas começariam pela rate_7D e acabariam parecidas, e a média não ajudaria tanto.

O HistGB treina árvores pequenas em sequência, e cada uma tenta corrigir os erros das anteriores (boosting), o que acaba por reduzir o viés. O early_stopping ficou desligado porque, por defeito, ele separa 10% aleatório do treino para decidir quando parar, o que acabaria por misturar o futuro com o passado. O HistGB não está nos slides, mas funciona como o LightGBM: agrupa os valores em intervalos, o que o deixa rápido com 180 mil sessões.

A votação faz a média das probabilidades da LR, do NB e da árvore. Para ajudar, os modelos precisam de ser bons sozinhos e de errar em sessões diferentes. Como os três têm princípios diferentes, esperávamos alguma diversidade, mas todos se apoiam principalmente no histórico do posto, então acabam por errar em sessões parecidas. Além disso, o NB, com probabilidades muito extremas, pode acabar por atrapalhar.

### 3.4.2 Resultados: os ensembles melhoram os seus modelos base?

**Tabela 3.4.2a** — Ensembles no G3

| Ensemble | Fold 1 | Fold 2 | Fold 3 | Média | Desvio | Pior fold | Folds > regra |
|---|---|---|---|---|---|---|---|
| Random Forest | 0,533 | 0,660 | 0,641 | 0,612 | 0,069 | 0,533 | 3 |
| HistGradientBoosting | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| Votação (LR + NB + árvore) | 0,514 | 0,643 | 0,622 | 0,593 | 0,070 | 0,514 | 3 |
| Votação sem o NB (LR + árvore) | 0,524 | 0,650 | 0,628 | 0,601 | 0,067 | 0,524 | 3 |

**Tabela 3.4.2b** — Diferença de AUC-PR entre cada ensemble e os seus modelos base (positivo = o ensemble é melhor)

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

Todos os ensembles acabam por ser melhores do que os seus modelos base nos 3 folds (tabela 3.4.2b). A Random Forest e o HistGB ganham cerca de +0,03 da melhor árvore individual, e ficam praticamente empatados entre si. Isso é interessante na Random Forest, porque ela é feita de árvores mais profundas, que sozinhas acabariam por fazer overfitting (tabela 3.3.3). Ou seja, o bagging acaba por corrigir esse problema.

Na votação, tirar o NB melhora o resultado nos 3 folds. Isso acontece porque o NB tem probabilidades muito extremas e, como não usa class_weight='balanced', fica numa escala diferente da LR e da árvore, puxando demais a média. Mesmo sem o NB, a votação não chega à Random Forest nem ao HistGB, porque junta só dois ou três modelos mais fracos, que erram em sessões parecidas.

### 3.4.3 Stacking

Acabamos por não usar stacking. Para usar sem leakage, os modelos base teriam de ser treinados num período mais antigo e o meta-modelo no período seguinte, sempre antes da validação. O StackingClassifier normal, com cv=5, não serve, porque divide os dados de forma aleatória e acabaria por misturar o futuro com o passado.

---

## 3.5 Comparação final

**Tabela 3.5a** — Todos os modelos no G3, por ordem da média

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

**Tabela 3.5b** — Diferença para a regra em cada fold (positivo = melhor do que a regra)

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

Isso mostra que olhar só para a média não chega. O HistGB e a Random Forest ficam praticamente empatados: o HistGB ganha nos 3 folds, mas por muito pouco (tabela 3.4.2b). Pelo protocolo da Task 2, o HistGB fica em primeiro por ter a média e o pior fold um pouco melhores, mas como a diferença é muito pequena, os dois seguem para a próxima etapa (afinação, threshold e avaliação final).

---

## 3.6 Limitações

Escolhas feitas nos mesmos folds da comparação: a profundidade da árvore, a versão do NB e a votação sem o NB foram escolhidas olhando para a validação dos mesmos folds onde depois comparamos os modelos. Isso pode deixar esses modelos um pouco favorecidos, já que acabaram por ser ajustados para esses períodos. O ideal seria escolher num período separado, mas com apenas 3 folds isso deixaria menos dados para a comparação.

Sem teste estatístico: com apenas 3 folds, o teste de Wilcoxon nunca consegue dar significativo, porque o menor p-valor possível é 0,25. Por isso, a comparação acaba por ser feita pela consistência fold a fold (o modelo ganha nos 3 folds ou não?). Diferenças muito pequenas, como a entre o HistGB e a Random Forest, não podem ser consideradas uma diferença real.

Poucas sessões de frota: a análise das frotas usa apenas 2 contas e poucas sessões, então a AUC-PR desse subgrupo pode mudar bastante com poucas sessões a mais ou a menos. Não dá para tirar conclusões fortes dela.

Probabilidades do NB mal calibradas: 72% das probabilidades do NB ficam abaixo de 1% ou acima de 99%, por isso não podem ser lidas como o risco real de uma sessão. É também por isso que o NB atrapalha a votação.

Ensembles sem afinação: a Random Forest e o HistGB ainda usam os valores por defeito, mudando apenas o class_weight e o early_stopping. Os resultados podem melhorar com a afinação, e a ordem entre os dois pode mudar. Isso fica para a próxima etapa.

Falhas do carro no histórico do posto: algumas falhas, como as de BMS ou Vehicle charging faults, vêm do carro e não do posto, mas acabam por contar no histórico do posto. Essa divisão foi feita só pelo nome de cada motivo e não foi verificada.


---

# Task 4 — Optimização de hiperparâmetros e validação cruzada

## 4.1 Hiperparâmetros escolhidos

Tabela 4.1 — Melhores hiperparâmetros no G3, procurados nos 3 folds temporais da Task 2 com o F1 como pontuação

| Modelo | Busca | Melhores hiperparâmetros | F1 (média nos folds) |
|---|---|---|---|
| Random Forest | `RandomizedSearchCV`, 10 combinações | `n_estimators=500`, `max_depth=None`, `min_samples_split=2`, `max_features='sqrt'` | nan (as 30 fits falharam por falta de memória) |
| HistGradientBoosting | `BayesSearchCV`, 40 iterações | `learning_rate=0,037`, `max_depth=None`, `max_iter=393`, `min_samples_leaf=20` | 0,5243 |

## 4.2 Comparação dos thresholds

Tabela 4.2 — Métricas nas previsões OOF das 3 validações, com o threshold padrão (0,50) e com o threshold que maximiza o F1 (procurado de 0,05 a 0,95, de 0,01 em 0,01)

| Modelo | Threshold | Precision | Recall | F1 | Taxa de alertas | Alertas corretos / 1 000 | Alertas falsos / 1 000 | Anomalias perdidas / 1 000 |
|---|---|---|---|---|---|---|---|---|
| Random Forest | 0,50 | 0,7936 | 0,3268 | 0,4629 | 7,76% | 61,55 | 16,01 | 126,81 |
| Random Forest | 0,29 | 0,5723 | 0,5263 | 0,5484 | 17,32% | 99,14 | 74,08 | 89,21 |
| HistGradientBoosting | 0,50 | 0,4293 | 0,6873 | 0,5285 | 30,15% | 129,46 | 172,08 | 58,89 |
| HistGradientBoosting | 0,63 | 0,5710 | 0,5455 | 0,5580 | 17,99% | 102,74 | 77,19 | 85,61 |

## 4.3 Análise dos resultados

- O **HistGradientBoosting com threshold otimizado (0,63)** apresenta o melhor desempenho global, com **F1-score de 0,5580**.
- A otimização do threshold melhorou o F1-score de ambos os modelos:
	- Random Forest: **0,4629 → 0,5484**
	- HistGradientBoosting: **0,5285 → 0,5580**
- No HistGradientBoosting, aumentar o threshold de **0,50 para 0,63** reduziu significativamente os falsos alertas (**172,08 → 77,19 por 1.000 sessões**), mantendo um recall razoável (**0,5455**).
- O threshold padrão do HistGradientBoosting oferece o maior recall (**0,6873**), mas gera muitos falsos alertas e apresenta menor precisão.
- O Random Forest otimizado tem desempenho equilibrado, mas fica ligeiramente abaixo do HistGradientBoosting otimizado.
- Assim, o modelo recomendado é o **HistGradientBoosting com threshold 0,63**, por apresentar o maior F1-score e um compromisso mais adequado entre deteção de anomalias e número de falsos alertas.

Os resultados foram calculados sobre as previsões OOF temporais; portanto, o threshold otimizado deve ser posteriormente confirmado num conjunto de teste temporal independente.

---

# Task 5 — Seleção final do modelo

## 5.1 Avaliação no teste final

Tabela 5.1 — HistGradientBoosting (threshold 0,63) e baseline no-skill no teste final (os 20% de sessões mais recentes, 88 216 sessões)

| Modelo | PR-AUC | Precision | Recall | F1 | Balanced accuracy | MCC | Taxa de alertas | Alertas corretos / 1 000 | Alertas falsos / 1 000 | Anomalias perdidas / 1 000 |
|---|---|---|---|---|---|---|---|---|---|---|
| Baseline (no-skill) | 0,1810 | 0,0000 | 0,0000 | 0,0000 | 0,5000 | 0,0000 | 0,00% | 0,00 | 0,00 | 181,00 |
| HistGradientBoosting (threshold 0,63) | 0,3223 | 0,4898 | 0,4577 | 0,4732 | 0,6762 | 0,3619 | 16,91% | 82,84 | 86,30 | 98,16 |

## 5.2 Análise dos resultados

O **HistGradientBoosting com threshold otimizado** apresenta desempenho claramente superior ao baseline:

- O modelo identifica **45,77% das sessões anormais** (*recall*).
- Entre os alertas emitidos, **48,98% são realmente anomalias** (*precision*).
- Em cada 1.000 sessões, gera aproximadamente:
    - **82,8 alertas corretos**;
    - **86,3 falsos alertas**;
    - **98,2 anomalias não detetadas**.
- A taxa de alertas é de **16,91%** das sessões.
- O *Balanced Accuracy* (**0,6762**) e o MCC (**0,3619**) indicam capacidade preditiva moderada, superior ao acaso.

O baseline não gera alertas porque a probabilidade estimada (**0,1869**) é inferior ao threshold padrão de 0,5. Por isso, obtém precisão, recall e F1 iguais a zero.

Assim, o modelo é útil como sistema de triagem, mas ainda deixa escapar cerca de metade das sessões anormais e produz uma quantidade semelhante de falsos alertas.
