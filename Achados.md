# Achados — Task 1: Compreensão, auditoria e preparação dos dados

Estrutura espelhada das secções do Projeto_MINDD.ipynb.

---

## 1. Carregamento e visão geral

O dataset tem 441 077 linhas e 47 colunas. É possivel observar que 4 colunas de datas (Start time, End Time, Payment Time e Order creation time) foram lidas como texto (string) e precisam de ser convertidas para alguma forma de data antes que possamos fazer qualquer tipo de conta com a mesma. Também é possivel observar que apenas 4 colunas têm valores ausentes, todas do grupo de atributos históricos (post_hours_since_previous_completed_session, post_hours_since_previous_completed_abnormal, user_days_since_previous_completed_session, user_days_since_previous_completed_abnormal).

---

## 2. Qualidade dos dados

### 2.1 Valores ausentes

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

### 2.2 Dados duplicados

Verificando-se pelo comando, não existem linhas duplicadas no dataset, por isso não é necessária nenhuma ação para esse tipo de problema.

### 2.3 Espaços escondidos no texto e conversão de datas

Fizemos então a conversão dos valores de datas de string para date a fim de podermos realizar contas com os valores da tabela.

Um ponto importante é que apenas itens com tipo object ou string podem ter um tab final escondido no meio do texto, como o tipo das datas já foram modificadas e perderam automaticamente o tab final iremos procurar nos outros itens que tenhas os tipos em que esse problema possa aparecer.

Após modificados os itens problemáticos vemos que nenhum item com o tipo object ou string tem mais erros de escrita que possam dividir resultados, trazendo incossitencia nos dados.

### 2.4 Consistência e valores extremos

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

#### Conclusão da 2.4

Não encontramos inconsistências que exijam corrigir ou remover linhas. As pequenas diferenças de horário afetam apenas colunas que não irão entrar no modelo, e os valores extremos de clima (mínimos e máximos) são plausíveis para a região dos quais os dados estão sendo retirados.

### 2.5 Outliers nas variáveis preditoras

#### 2.5.1. Outliers do histórico do posto = sinal, não erro


Os valores que nós vemos de outliers acabam por vir de postos com uma proporção alta de sessões anormais recentes. No post_previous_abnormal_rate_1D acabamos por ter 6,4% de outliers que são de sessões em que o posto teve mais de 46% de sessões anormais no dia anterior.

Já no post_previous_abnormal_rate_7D conseguimos observar que 6,7% das sessões ocorreram num posto com mais de 47% de sessões anormais num período de 7 dias e com o post_previous_abnormal_rate_30D vemos que 5,7% ocorreram em um posto com mais de 49% de sessões anormais em 30 dias.

Lembrando que os três valores são rates, então dá para comparar as três janelas entre si. Se quisermos ver os valores em absoluto seria só utilizar as medições de post_previous_abnormal_1D, post_previous_abnormal_7D e post_previous_abnormal_30D. Ao utilizar esses itens teriamos o valor absoluto, X% de sessões anormais em postos com mais de Y sessões anormais.

#### 2.5.2. Tempo desde a última sessão do posto: cauda muito longa

Ao observarmos a variavel post_hours_since_previous_completed_session conseguimos observar uma escala extrema, sendo apenas possivel observar utilizando a logaritmica no plot. A variável tem uma mediana de 0,48h ou 29 min de tempo entre sessões completas, um máximo de 5137h ou praticamente 7 meses parado e com uma assimetria ou skew de praticamente 110, a maior em todo o dataset. 

Ou seja, a maior parte dos nossos valores são pequenos e os poucos valores extremos podem indicar que o posto em questão possa ter ficado parado devido a uma manutenção ou alguma impossibilidade que impediu os condutores de abastecer naquele local. Em geral os valores encontrados parecem verossímeis. Novamente, não é um erro, apenas uma escala extrema.


#### 2.5.3. Contadores do usuário: dominados pelas contas de frota

Nosso user_previous_sessions_730D tem uma mediana de 21, um p99 de 10172 e um máximo de 14566. Isso indicaria que, assim como a distribuição anterior, a distribuição é muito assimétrica, onde a ponta da direita, os maiores números de carregamentos, depende de uma ou 2 contas.

Os usuários 39 e 23 tem, em valores absolutos, 14584 e 6748 carregamentos. Uma média de 6050 sessões de carregamento, onde um usuário médio tem uma média de aproximadamente 64 carregamentos.

Na user_previous_sessions_30D um usuário chega a ter 2605 carregamentos em 30 dias, o que seria algo em torno de 87 carregamentos por dia, sendo impossível para uma pessoa carregar o carro quase que 3 vezes por dia.

É no user_active_sessions_at_start que conseguimos entender o que se passa. 95% das sessões tem valor 0, ou seja, na esmagadora maioria das vezes um usuário apenas carrega 1 carro por vez. Já o numero máximo de carregamentos por vez é 14, indicando que provavelmente se trata de carros. 

Essa ideia é confirmada quando vemos que os userIDs com numeros de sessão maiores que 2 são exatamente o 39 e 23 de antes. Isso provavelmente indica que são contas empresariais ou algo do gênero que possuem uma frota de carros. 

Logo, mantemos esses dados e como forma de vizualizar os mesmos utilizamos a escala logarítmica.


#### 2.5.4. Clima: sem outliers problemáticos

Temperatura e umidade tem uma assimetria de aproximadamente −0,4 e praticamente nenhum outlier no dado amostral.

Já na Precipitação temos um problema: a regra do IQR marca 19% como outlier. Porém a mediana é 0 (58% das sessões sem chuva). Pesquisando um pouco mais vê-se que o IRQ como medida não serve para variáveis cujos dados tenham muitos valos iguais a 0


#### Decisão final

Depois de analisar os outliers das variáveis preditoras, decidimos por não remover nenhuma linha do dataset. Como vimos nos pontos anteriores, todos os valores extremos acabam por ter uma explicação real e plausível. Pode ser um posto com muitas sessões anormais, um posto que ficou muito tempo parado ou uma conta que parece pertencer a uma empresa (há de se ficar atento a esse último dado na hora da modelação, visto que eles podem atrapalhar o aprendizado do madelo). Removê-los seria jogar informação verdadeira fora.

Também decidimos não limitar as variáveis de falhas do posto. É justamente nos valores altos dessas variáveis que conseguimos observar o sinal que queremos prever, como vamos ver na secção 4. Já para as contagens e os tempos com assimetria acima de 3, a ideia é aplicar uma transformação logarítmica (log1p) caso o modelo seja linear ou baseado em distância, como a regressão logística, o KNN ou o SVM. Nos modelos de árvore isso não é necessário, já que eles não são afetados pela escala das variáveis.

Lembrando que, se em algum momento formos limitar os dados, o limite (por exemplo o p99) deve ser calculado apenas no conjunto de treino, para não vazar informação do teste para o modelo. O enunciado pede explicitamente para este fato ser documentado, então fica aqui registrado.

Por fim, a única limitação que conseguimos observar, como dito anteriormente, é que as contas de frota podem acabar por dominar o aprendizado do modelo, já que possuem valores muito diferentes de todos os outros usuários. Ou seja, na fase de modelação vale a pena avaliar o modelo com e sem essas contas para avaliar se o resultado muda consideravelmente.

---

## 3. Definição da variável alvo

### Distribuição do alvo

Aproximadamente 81,5% dos carregamentos terminam de forma normal e apenas 18,5% terminam de forma anormal por algum problema.

### Disponibilidade das variáveis no início da sessão

Como a previsão deve ser feita no início da sessão, cada variável foi classificada segundo a sua disponibilidade nesse momento. As variáveis registadas durante ou após a sessão foram excluídas dos preditores para evitar o que se foi pedido no enunciado ("The prediction must represent a decision made when a charging session starts. Consequently, only information that would reasonably be available at that time may be used as a predictor"), sendo usadas apenas na análise exploratória.

Outro ponto importante é verificar as analises levando em conta que o ID tanto do posto quanto do usuário pode gerar enviseamento dos dados. O enunciado pede para usarmos com e sem esses IDs para verificar as diferenças nos resultados.

### electricity_price × tariff_period

A variável electricity_price tem só 3 valores (0,38 / 0,90 / 1,21). Na prática é categórica e repete a informação do tariff_period. Logo são informações redundantes por definição das mesmas. Ou seja, caso usemos uma não precisamos da outra.


### Tabela de classificação das colunas

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

### Listas reutilizadas nas próximas etapas

A partir da tabela, guardamos três listas que vão ser reutilizadas nas próximas etapas:

- features_inicio: colunas candidatas a preditores;
- colunas_depois: colunas que nunca entram no modelo, só na análise exploratória;
- colunas_ids: identificadores para se testar com e sem sua presença, na fase de modelação.

Assim, a separação entre o que pode e o que não pode entrar no modelo fica definida num único local e não precisa ser repetida à mão depois.

A célula seguinte mostra, com um exemplo concreto, porque é que as colunas pós-sessão têm de ficar de fora.

### Exemplo: 0 kWh "entrega" a resposta

Exemplo de problema nos dados: das 54621 sessões com 0 kWh carregados, 83,4% são anormais. Nas sessões que contem energia carregada (> 0 kWh), só 9,4% são anormais.

Faz sentido se pensarmos que a sessão que falha logo no início, não chega a carregar energia. Mas a quantidade de energia só é conhecida depois de uma sessão terminar. Se esta coluna entrasse no modelo, ele aprenderia a regra "0 kWh → anormal" e teria ótimos resultados no teste, mas seria inútil na prática, porque no momento da previsão esse valor ainda não existe, estariamos apenas jogando lixo para dentro de nossos dados.

Isto confirma, com evidência, e justifica a decisão de excluir dos preditores todas as colunas classificadas como "Depois do fim".

Já as sessões com 0 kWh, que na secção 2.4 tinham sido mantidas por enquanto, continuam no dataset. Elas não são erros, mas sim sessões reais que falharam antes de carregar energia, e cerca de 45,5 mil delas são anormais, o que representa mais da metade de todas as anomalias do dataset (cerca de 82 mil). Se removêssemos essas linhas, estaríamos apagando justamente a maior parte dos casos que o modelo precisa aprender a prever. O problema estava apenas na coluna de energia, que denuncia a resposta, e como ela já foi excluída dos preditores, as linhas podem ficar sem trazer nenhuma fuga de informação.

---

## 4. Relação das variáveis com o alvo

### 4.1 Histórico do posto (heatmap 1)

- Forte com o alvo: as taxas de anomalia do posto: rate_1D 0,45, rate_7D 0,43, abnormal_1D 0,37, rate_30D 0,37.
- Fraco ou nulo: número de sessões (7D 0,07, 30D 0,01) e horas desde a última sessão (0,01).
- Negativo: hours_since_previous_completed_abnormal −0,12. Quanto mais tempo desde a última falha, menor o risco.
- Redundâncias: sessions_7D × 30D 0,90; abnormal_1D × rate_1D 0,85; rate_7D × rate_30D 0,82.

Do heatmap 1 conseguimos tirar a informação de que, umm posto que falhou recentemente, tende a falhar de novo. A taxa de anomalias do posto no último dia (rate_1d) é a variável que mais se correlaciona com o alvo (0,45), seguida da taxa de anomalias do posto na ultima semana (rate_7D) (0,43). 

Sendo que é visivel que essa correlação cai a medida que a janela de tempo aumenta. Sendo assim o histórico mais recente é o que melhor informa e prediz uma proxima falha no posto. Sendo o inverso tambem informativo, onde quanto mais tempo passa desde a ultima falha, menor o risco (hours_since_previous_completed_abnormalcom uma taxa de −0,12). 

### 4.2 Histórico do usuário (heatmap 2)

- Forte com o alvo: só as taxas: rate_30D 0,30, rate_180D e rate_730D 0,28.
- Nulo: contagens de sessões e de falhas (≈ −0,02 a 0,01) e is_new_user 0,02.
- Redundância extrema:
  - abnormal_730D × sessions_730D 1,00;
  - abnormal_180D × sessions_180D 0,99;
  - is_new_user × user_no_previous_completed_session 0,99;
  - rate_180D × rate_730D 0,97.

Do heatmap 2 vemos que, assim como no posto, o que se relaciona com a falha é a taxa de anomalias do usuário, e não a quantidade de sessões ou de falhas. A taxa dos últimos 30 dias (rate_30D) é a mais forte (0,30), seguida das taxas de 180 e 730 dias (0,28). Já as contagens ficam praticamente em zero, ou seja, um usuário que carrega muito não falha necessariamente mais. Mesmo assim, o histórico do usuário informa menos que o do posto (0,30 contra 0,45).

Tambem é visivel que muitas colunas são praticamente duplicadas, como abnormal_730D e sessions_730D (1,00) e is_new_user e user_no_previous_completed_session (0,99), então não faz sentido manter todas no modelo. Essa correlação de 1,00 entre falhas e sessões é estranha, e provavelmente acontece porque algumas contas com milhares de sessões dominam o cálculo de Pearson (Na próxima etapa iremos verificar melhor o que pode estar acontecendo).

### 4.3 Contexto da sessão — clima, preço e sessões ativas do usuário no momento (heatmap 3)

- Tudo ≈ 0 (temperatura 0,03; os demais entre −0,01 e 0,01).

Do heatmap 3 vemos que todas as variáveis de contexto ficaram perto de zero, sendo a temperatura a maior com apenas 0,03. Sendo assim, sozinhos, o clima, o preço e o número de sessões ativas do usuário não explicam a anomalia. Mas como Pearson só mede relação linear, essas variáveis ainda podem ajudar o modelo quando , quem sabe, combinadas com outras variáveis.

### 4.4 Correlação de Pearson da localização e do período tarifário com is_Abnormal

#### 4.4a Localização (heatmap 4a)

- Maior risco: Expressway C (0,11), Shopping Mall (0,07), Expressway A (0,07), Wholesale Market (0,06).
- Menor risco: Park B (−0,10), Bus Station (−0,09), Technology Park (−0,06).
- Taxa real: vai de 7,9% (Industrial Park) a 32,4% (Expressway A).

Do heatmap 4a conseguimos ver que a localização do posto importa. As áreas de serviço das vias expressas (Expressway C com 0,11 e Expressway A com 0,07) e o do Shopping Mall (0,07) são as que mais se relacionam com falhas, enquanto Park B (−0,10) e Bus Station (−0,09) têm menos risco. Os valores de correlação parecem baixos porque cada localização é só uma pequena parte das sessões, mas olhando a taxa real fica mais claro: vai de 7,9% no Industrial Park até 32,4% na Expressway A, ou seja, as áreas da autoestrada têm cerca de 4× mais falhas que os parques. Não da para saber o porque disso ocorrer, mas é de fato interessante ter uma correlação como essas presente.

#### 4.4b Período tarifário (heatmap 4b)

- Tudo proximo de 0 (entre −0,016 e 0,02). A taxa só varia entre 16,7% e 20,5%.

Do heatmap 4b vemos que o período tarifário quase não influencia na falha, já que todas as correlações ficam perto de zero (entre −0,016 e 0,02) e a taxa de anomalia só varia entre 16,7% e 20,5%. É uma variável fraca, mas pode ser mantida no modelo, já que pode ajudar junto com outras variáveis.

### 4.5 Eliminação de variáveis redundantes

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


### 4.6 Outliers contra o is_Abnormal

Na secção 2.5 decidimos manter os outliers das variáveis preditoras, e aqui fomos verificar se eles realmente carregam informação sobre o alvo. Para isso, em cada variável separamos as sessões que ficam acima do limite do IQR (Q3 + 1,5 × IQR) e comparamos a taxa de anomalia delas com a do resto das sessões.

No histórico do posto a diferença é muito grande. Nas sessões em que o posto teve uma taxa de anomalias no dia anterior (post_previous_abnormal_rate_1D) fora do normal, 75% são anormais, contra apenas 14,7% no resto. O mesmo acontece com o número de falhas do posto na última semana (post_previous_abnormal_7D), com 55,3% contra 14,8%, e nos últimos 30 dias (post_previous_abnormal_30D), com 45,7% contra 15,7%. Sendo assim, os outliers do posto não são erros, mas sim os casos que parecem mais informar sobre uma próxima falha, ou seja, se removêssemos ou limitássemos esses valores estaríamos apagando justamente a informação mais preditiva que temos. Quando um posto começa a falhar, obviamente as sessão naquele dia vão sofrer com mais anomalias de carregamento.

No histórico do usuário o padrão se repete nas taxas de anomalia. Os outliers das taxas de 30, 180 e 730 dias têm entre 45,6% e 46,9% de sessões anormais, contra cerca de 16% no resto. Ou seja, a taxa de falhas do usuário também tem sinal, embora mais fraco que a do posto (cerca de 46% contra 75%).

Já no número de sessões do usuário em 730 dias (user_previous_sessions_730D) acontece o contrário, onde os outliers têm até um pouco menos de anomalia (15,4% contra 19%). Esses outliers são as contas com milhares de sessões que vimos na secção 2.5.3, que provavelmente pertencem a frotas, e a taxa de anomalia delas é parecida com a do resto, então não carregam nenhum sinal especial. Mesmo assim, são sessões reais e por isso também ficam no dataset.

Com isso confirmamos a decisão da secção 2.5 de não remover nem limitar os outliers, já que nas variáveis de falhas eles são exatamente o sinal que queremos que o modelo aprenda.


---

## 5. Estabilidade temporal

Nesta secção a ideia principal era verificar se os dados se mantêm estáveis ao longo do tempo ou se o comportamento das sessões muda de um período para o outro, já que, se os dados mudarem muito, um modelo treinado num período pode não funcionar tão bem noutro. Os dados cobrem exatamente 25 meses e vão de 31/12/2019 a 31/12/2021, sendo que dezembro de 2019 tem só 17 sessões, visto que os registos começam às 21h do último dia do mês, por isso não o consideramos como um mês nas análises mensais.

A primeira coisa que conseguimos ver é que o volume de sessões cresce bastante, passando de cerca de 6 mil para cerca de 27 mil sessões por mês, e 2021 acaba por ter mais dados que 2020 (255 mil contra 186 mil sessões). A exceção é fevereiro, que cai nos dois anos (as 2 grandes quedas que acontecem no primeiro gráfico "Número de sessões por mes"), chegando a só 1 391 sessões em fev/2020, o que pode estar possivelmente ligado ao Ano Novo Chinês e, em 2020, ao início da COVID, mas os dados não nos permitem confirmar isso. Junto com o volume cresce também a base de usuários, que vai de cerca de 2 mil (2020-01) para cerca de 7,5 mil (2021-10) usuários ativos por mês, lembrando que 43% dos usuários têm uma única sessão em todo o dataset. Já o número de postos fica estável em 91–92 durante quase todo o período e só cai para 86–88 no fim de 2021, porque 4 postos (17, 18, 19 e 20) param de ter sessões em 28/09/2021 e outros param em dezembro. Não dá para saber se foram desativados ou se entraram em manutenção, mas isso explica a queda.

A taxa de anomalia também não é constante. Ela começa em cerca de 25%, cai para 16–17% no fim de 2020, chega ao mínimo de 13,7% em mar/2021 e volta a subir para cerca de 23% em set/2021. Curiosamente, olhando por ano a taxa quase não muda (18,7% em 2020 e 18,5% em 2021), ou seja, a variação só aparece quando olhamos mês a mês. As variáveis de histórico acompanham essa mudança da forma que esperávamos: a média mensal da taxa de falhas do posto em 7 dias vai de cerca de 0,23 para 0,15 e volta para 0,22, e a do usuário em 30 dias faz o mesmo, porém de forma mais suave (entre 0,17 e 0,25), o que bate com o que vimos na secção 4, onde o histórico do usuário informa menos que o do posto.

Já a percentagem de novos usuários é muito alta no início (94% em dez/2019 e 31% em jan/2020), o que faz sentido, visto que no começo dos registos quase todos os usuários aparecem pela primeira vez, e depois estabiliza em 10–15% por mês. Esses novos usuários têm um pouco mais de anomalia que os restantes (20,6% contra 18,3%). Ou seja, os primeiros meses são um "início frio", onde o histórico dos usuários e dos postos ainda está sendo construído e as variáveis de histórico têm pouca informação. A média de sessões anteriores do usuário em 730 dias cresce de forma natural, já que a janela vai acumulando sessões, mas tem um pico estranho em jun/2020 (626) seguido de uma queda em ago/2020 (132). Como a média é calculada sobre as sessões de cada mês, isso provavelmente acontece nos meses em que as contas grandes (os usuários 39 e 23, que somam cerca de 21 mil sessões, ou seja ≈5% do total) carregam muito e puxam a média para cima. São essas mesmas contas que explicam a correlação de 1,00 entre user_previous_sessions_730D e user_previous_abnormal_730D no heatmap 2. Por fim, a temperatura segue o ciclo normal das estações, com cerca de 6 °C no inverno e 30 °C no verão, e a anomalia sobe um pouco nos verões (ago/2020 e set/2021), mas como vimos na secção 4.3 a correlação linear com o alvo é fraca (0,03), então existe uma sazonalidade, mas sozinha ela não explica a anomalia.

De forma geral, o volume de sessões, a taxa de anomalia mensal e a base de usuários mudam ao longo do tempo, ou seja, os dados não parecem ser estáveis, e isso traz algumas consequências para a modelação. A divisão entre treino e teste tem de ser temporal, treinando com o passado e testando com os meses finais, já que se fizéssemos uma divisão aleatória estaríamos misturando sessões do futuro no treino e o resultado seria otimista demais. Como a taxa de anomalia do período de teste pode ser diferente da do treino, também não vamos avaliar o modelo só pela acurácia, mas por métricas como F1, PR-AUC e o recall da classe anormal. Temos ainda de ter atenção ao início frio, podendo descartar jan–fev/2020 do treino ou, pelo menos, citar isso como uma limitação, e às contas grandes, que podem dominar o aprendizado do modelo, por isso a ideia é verificar o erro com e sem elas. Por fim, como alguns postos param no fim de 2021, o período de teste vai ter menos postos que o de treino, o que também é uma limitação a citar.


### Decisões para a Task 2

- **Início frio:** as sessões de dez/2019 e jan/2020 vão ser retiradas do treino. Nesses meses o histórico dos usuários e dos postos ainda está sendo construído (94% e 31% de usuários novos, contra 10–15% nos outros meses), então as variáveis de histórico têm pouca informação. São cerca de 6 200 sessões (~1,4% do total).
- **Fevereiro:** a queda de sessões em fev/2020 não é removida. A mesma queda acontece em fev/2021, então parece ser um evento que se repete todo ano, muito mais provavelmente ligado ao Ano Novo Chinês do que à COVID como antes pensado. Além disso, a taxa de anomalia desse mês (26,3%) é parecida com a de janeiro (25,0%) e março (24,8%).
- O corte é feito só no treino, na Task 2. O CSV exportado no notebook manterá todas as sessões não modificadas.
