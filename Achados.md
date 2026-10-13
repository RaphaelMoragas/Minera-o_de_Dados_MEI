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

#### post_hours_since_previous_completed_abnormal × post_no_previous_completed_abnormal`

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

Os valores monetários são consistentes: em nenhuma sessão o total é diferente da soma de eletricidade + serviço, e em nenhuma o valor pago é maior que o total.

Existem 54621 sessões (~12%) com 0 kWh carregados. Isto não é um erro, são sessões que terminaram sem carregar energia, mais a frente iremos verificar se se elas são de um erro (se são sessões anormais). Por isso essas linhas serão mantidas por enquanto.

#### Clima

Verificamos se os valores do clima fazem sentido para Jiaxing (leste da China, clima subtropical):

- Temperatura entre -3,2 °C e 33,6 °C: plausível para inverno e verão da região.
- Umidade entre 32% e 99%: dentro do intervalo possível (0–100%).
- Precipitação entre 0 e 152,1 mm: a maioria das sessões não tem chuva (a mediana é 0), e os valores mais altos podem ser dias de chuva forte ou tufão, que acontecem na região.

Nenhum valor parece impossível, então não removemos nem alteramos nada. Os valores mais altos de precipitação são extremos, mas reais.

#### Valores negativos

A única coluna numérica com valores negativos é a temperatura (2 052 sessões), e todos esses casos acontecem em dezembro de 2020, ou seja, no inverno. Temperaturas abaixo de zero são normais nessa época, então não é um erro.

Nenhuma outra coluna (energia, custos, contagens do histórico) tem valores negativos, o que era esperado porque não faria sentido.

#### Conclusão da 2.4

Não encontramos inconsistências que exijam corrigir ou remover linhas. As pequenas diferenças de horário afetam apenas colunas que não vão entrar no modelo, e os valores extremos de clima (mínimo e máximo) são plausíveis para a região dos quais os dados estão sendo retirados.

### 2.5 Outliers nas variáveis preditoras

#### 1. Outliers do histórico do posto = sinal, não erro

- `post_previous_abnormal_rate_1D`: 6,4% de outliers. `abnormal_7D`: 9,3%. `abnormal_30D`: 9,7%.
- **O que escrever:** os valores extremos são postos com muitas falhas recentes. A relação com o alvo é vista no fim da secção 4.

#### 2. Tempo desde a última sessão do posto: cauda muito longa

- `post_hours_since_previous_completed_session`: mediana de 0,5 h, máximo de 5 137 h (~7 meses), assimetria de 110 (a maior do dataset).
- **O que escrever:** é plausível, porque um posto pode ficar parado (manutenção, desativação). Não é erro, mas a escala é extrema, então sugere uma transformação log.

#### 3. Contadores do usuário: dominados pelas contas de frota

- `user_previous_sessions_730D`: mediana de 21, p99 de 10 172, máximo de 14 566.
- Usuários 39 e 23: média de 6 050 sessões anteriores. Todos os outros: média de 64.
- `user_previous_sessions_30D` chega a 2 605 em 30 dias (~87 por dia), o que é impossível para uma pessoa.
- `user_active_sessions_at_start`: 95% das sessões têm 0, e o máximo é 14 sessões ao mesmo tempo.
- **O que escrever:** são contas de frota ou empresariais, não erros. A decisão é manter e transformar com log.

#### 4. Clima: sem outliers problemáticos

- Temperatura e umidade: assimetria ≈ −0,4 e praticamente nenhum outlier.
- Precipitação: a regra do IQR marca 19% como outlier, mas é falso alarme. A mediana é 0 (58% das sessões sem chuva), então qualquer chuva vira "outlier".
- **O que escrever:** a regra do IQR não serve para variáveis com muitos zeros.

#### Decisão (para registrar)

- Não remover nenhuma linha: todos os extremos têm explicação real.
- Não limitar (cap) as variáveis de falhas do posto, porque é onde está o sinal (secção 4).
- Aplicar log (log1p) às contagens e aos tempos com assimetria acima de ~3, se o modelo for linear ou baseado em distância (regressão logística, KNN, SVM). Modelos de árvore não precisam.
- Ajustar só no treino: se algum cap for usado, o limite (por exemplo o p99) é calculado apenas no conjunto de treino. O enunciado pede para documentar isso.
- Limitação: as contas de frota podem dominar o aprendizado. Avaliar com e sem elas na fase de modelação.

---

## 3. Definição da variável alvo

### Distribuição do alvo

Aproximadamente 81,5% dos carregamentos terminam de forma normal, apenas 18,5% terminam de forma anormal por algum problema ou do carro, ou do carregador.

### Disponibilidade das variáveis no início da sessão

Como a previsão deve ser feita no início da sessão, cada variável foi classificada segundo a sua disponibilidade nesse momento. As variáveis registadas durante ou após a sessão foram excluídas dos preditores para evitaro que se foi pedido no enunciado ("The prediction must represent a decision made when a charging session starts. Consequently, only information that would reasonably be available at that time may be used as a predictor"), sendo usadas apenas na análise exploratória.

Outro ponto importante é verificar as analises levando em conta que o ID tanto do posto quanto do usuário pode gerar enviseamento dos dados. O enunciado pede para usarmos com e sem esses IDs para verificar as diferenças nos resultados.

### `electricity_price` × `tariff_period`

- `electricity_price` tem só 3 valores (0,38 / 0,90 / 1,21). Na prática é categórica e repete a informação do `tariff_period`.
- **O que escrever:** o preço é redundante com a tarifa.

### Tabela de classificação das colunas

A tabela acima classifica cada uma das 48 colunas (as 47 originais mais a `is_Abnormal` que criámos) segundo a pergunta: "este valor já é conhecido quando a sessão começa?"

- Início → Feature: informação conhecida no momento da previsão (localização, hora de início, clima, tarifa e atributos históricos). São as candidatas a preditores.
- Depois do fim → Só EDA: valores que só existem depois de a sessão terminar (energia, custos, hora de fim, pagamento e a própria causa de término). Se fossem usados, o modelo estaria a "ver o futuro" e teria um desempenho artificialmente alto. Ficam no dataset apenas para a análise exploratória.
- Discutível → A avaliar: os identificadores de usuário e posto. Estão disponíveis no início, mas podem levar o modelo a decorar IDs específicos, não funcionam para usuários ou postos novos, visto que os dados não existem na primeira vez de um usuário ou posto. A decisão será tomada comparando modelos com e sem estas colunas.
- Alvo: `is_Abnormal`, é a variável que tentaremos prever.

A última categoria do código (`?`) serve de verificação: se alguma coluna caísse aí, significaria que nos esquecemos de a classificar. Com o `is_Abnormal` agora classificado como alvo, nenhuma coluna fica por classificar.

### Listas reutilizadas nas próximas etapas

A partir da tabela, guardamos três listas que vão ser reutilizadas nas próximas etapas:

- `features_inicio`: colunas candidatas a preditores;
- `colunas_depois`: colunas que nunca entram no modelo, só na análise exploratória;
- `colunas_ids`: identificadores a testar com e sem, na fase de modelação.

Assim, a separação entre o que pode e o que não pode entrar no modelo fica definida num único sítio e não precisa de ser repetida à mão.

A célula seguinte mostra, com um exemplo concreto, porque é que as colunas pós-sessão têm de ficar de fora.

### Exemplo: 0 kWh "entrega" a resposta

Exemplo de problema nos dados: das 54 621 sessões com 0 kWh carregados, 83,4% são anormais. Nas sessões que contem energia carregada (> 0 kWh), só 9,4% são anormais.

Faz sentido: se a sessão falha logo no início, não chega a carregar energia. Mas a quantidade de energia só é conhecida depois de a sessão terminar. Se esta coluna entrasse no modelo, ele aprenderia a regra "0 kWh → anormal" e teria ótimos resultados no teste, mas seria inútil na prática, porque no momento da previsão esse valor ainda não existe, estariamos apenas jogando lixo para dentro de nossos dados.

Isto confirma, com evidência, e justifica nossa decisão de excluir dos preditores todas as colunas classificadas como "Depois do fim".

---

## 4. Relação das variáveis com o alvo

### 4.1 Histórico do posto (heatmap 1)

- **Forte com o alvo:** as taxas de anomalia do posto: `rate_1D` 0,45, `rate_7D` 0,43, `abnormal_1D` 0,37, `rate_30D` 0,37.
- **Fraco ou nulo:** número de sessões (7D 0,07, 30D 0,01) e horas desde a última sessão (0,01).
- **Negativo:** `hours_since_previous_completed_abnormal` −0,12. Quanto mais tempo desde a última falha, menor o risco.
- **Redundância:** `sessions_7D` × `30D` 0,90; `abnormal_1D` × `rate_1D` 0,85; `rate_7D` × `rate_30D` 0,82.
- **O que escrever:** um posto que falhou recentemente tende a falhar de novo. As janelas repetem a mesma informação, então dá para manter poucas.

### 4.2 Histórico do usuário (heatmap 2)

- **Forte com o alvo:** só as taxas: `rate_30D` 0,30, `rate_180D` e `rate_730D` 0,28.
- **Nulo:** contagens de sessões e de falhas (≈ −0,02 a 0,01) e `is_new_user` 0,02.
- **Redundância extrema:**
  - `abnormal_730D` × `sessions_730D` 1,00;
  - `abnormal_180D` × `sessions_180D` 0,99;
  - `is_new_user` × `user_no_previous_completed_session` 0,99;
  - `rate_180D` × `rate_730D` 0,97.
- **O que escrever:** o que prevê é a proporção de falhas, não a quantidade. Muitas colunas são praticamente duplicadas.
- **Ponto de atenção:** correlação 1,00 entre falhas e sessões é estranha. Provavelmente algumas contas com milhares de sessões dominam o cálculo de Pearson (veja o gráfico 730D na secção 5).

### 4.3 Contexto da sessão — clima, preço e sessões ativas do usuário no momento (heatmap 3)

- Tudo ≈ 0 (temperatura 0,03; os demais entre −0,01 e 0,01).
- **O que escrever:** sozinhos, clima, preço e sessões ativas não explicam a anomalia. Pearson só mede relação linear, então ainda podem ajudar combinados com outras variáveis.

### 4.4 Correlação de Pearson da localização e do período tarifário com `is_Abnormal`

#### 4.4a Localização (heatmap 4a)

- **Mais risco:** Expressway C (0,11), Shopping Mall (0,07), Expressway A (0,07), Wholesale Market (0,06).
- **Menos risco:** Park B (−0,10), Bus Station (−0,09), Technology Park (−0,06).
- **Taxa real (melhor de citar):** vai de 7,9% (Industrial Park) a 32,4% (Expressway A).
- **O que escrever:** a localização importa muito. As áreas de serviço da autoestrada têm cerca de 4× mais falhas que os parques.

#### 4.4b Período tarifário (heatmap 4b)

- Tudo ≈ 0 (entre −0,016 e 0,02). A taxa só varia entre 16,7% e 20,5%.
- **O que escrever:** o horário da tarifa quase não influencia. Uma variável fraca, mas pode ser mantida.

### Outliers contra o alvo

- `post_previous_abnormal_rate_1D`: 75% dos outliers são anormais, contra 14,7% no resto.
- `abnormal_7D`: 55% contra 15%. `abnormal_30D`: 46% contra 16%.
- **O que escrever:** remover ou limitar esses valores apagaria a informação mais preditiva.
- `user_previous_sessions_730D`: 15% contra 19%. Os outliers das contas de frota têm taxa de anomalia parecida com a do resto, então não carregam sinal especial.
- `user_previous_abnormal_rate_30D`: 46% contra 16%, ou seja, a taxa de falhas do usuário também tem sinal, embora mais fraco que a do posto.

---

## 5. Estabilidade temporal

### Sessões por mês

- Dez/2019 tem só 17 sessões, um mês incompleto.
- Fev/2020 cai para 1 391 (COVID e Ano Novo Chinês); fev/2021 também cai.
- Depois o volume cresce: de ~6k para ~27k por mês.

### Taxa de anomalia por mês

- Começa em ~25% e cai para ~16–17% no fim de 2020.
- Mínimo de 13,7% em mar/2021; volta a ~23% em set/2021.
- **O que escrever:** a prevalência muda com o tempo, então a divisão treino/teste deve ser temporal e não aleatória.

### Usuários ativos por mês

- Crescem de ~2k para ~7,5k. A base de usuários se expande.

### Postos ativos por mês

- Estável em ~91–92 e cai para 86–88 no fim de 2021 (postos desativados?).

### Novos usuários por mês (%)

- Alto no início (94% em dez/19, 31% em jan/20), depois estável em ~10–15%.
- **O que escrever:** os primeiros meses são "frios" (sem histórico), e o histórico ainda está sendo construído.

### Média mensal da taxa de falhas do posto (7D)

- Acompanha a taxa de anomalia mensal (~0,23 → ~0,15 → ~0,22).
- **O que escrever:** a variável reage à mudança, o que é coerente.

### Média mensal da taxa de falhas do usuário (30D)

- Mesmo padrão, porém mais suave (0,17–0,25).

### Média mensal de sessões anteriores do usuário (730D)

- Cresce de forma natural, já que a janela vai acumulando.
- Pico estranho em jun/2020 (626), seguido de queda em ago (132).
- **O que escrever:** indica contas com volume enorme (frotas, táxis?). Ligue isso à redundância 1,00 do heatmap 2 (secção 4.2).

### Temperatura média por mês

- Ciclo sazonal normal: ~6 °C no inverno, ~30 °C no verão.
- A anomalia sobe um pouco nos verões (ago/20, set/21).
- **O que escrever:** existe sazonalidade, mas a correlação linear é fraca (0,03).

**Conclusão geral para fechar a secção 5:** volume, prevalência e base de usuários mudam ao longo do tempo. Por isso, treinar com o passado e testar com os meses mais recentes, e avaliar o efeito do cold-start.

### Conclusão da estabilidade temporal

> Ainda por escrever no notebook: colocar uma célula markdown depois do gráfico da temperatura (última célula da secção 5), com estes pontos.

#### Cobertura

- De 31/12/2019 a 31/12/2021, 25 meses.
- Dez/2019 tem só 17 sessões (começa às 21h do último dia). Não serve como mês e deve ser ignorado nas análises mensais.

#### O que muda com o tempo

- **Volume:** passa de ~6k para ~27k sessões por mês. 2021 tem mais dados que 2020 (255k contra 186k).
- **Prevalência:** a taxa anual quase não muda (18,7% em 2020, 18,5% em 2021), mas a mensal oscila entre 13,7% e 25%.
- **Usuários:** a base cresce de ~2k para ~7,5k ativos por mês. 43% dos usuários têm uma única sessão.
- **Novos usuários:** são muitos no início e depois ficam em 10–15% por mês. Têm um pouco mais de anomalia (20,6% contra 18,3%).
- **Postos:** 4 postos (17, 18, 19, 20) param em 28/09/2021, e outros param em dezembro de 2021. Isso explica a queda para 86–88 postos ativos.
- **Contas grandes:** os usuários 39 e 23 somam ~21k sessões (≈5% do total). Explicam o pico do gráfico 730D e a correlação de 1,00 no heatmap 2.

#### Consequências para a modelação

- **Divisão temporal:** treinar com o passado e testar com os meses finais. Uma divisão aleatória misturaria o futuro no treino e daria um resultado otimista.
- **Prevalência no teste:** a taxa do período de teste pode ser diferente da do treino. Avaliar com métricas que não dependem só da acurácia (por exemplo F1, PR-AUC ou recall da classe anormal).
- **Início frio:** os primeiros meses têm o histórico quase vazio. Considerar descartar jan–fev/2020 do treino, ou ao menos citar isso como limitação.
- **Contas grandes:** podem dominar o aprendizado. Citar como risco e verificar o erro com e sem elas.
- **Postos desativados:** se saírem do conjunto de teste, o modelo é avaliado em menos postos. Citar como limitação.

**Nota:** na conclusão da secção, não dizer que fevereiro é "COVID/Ano Novo Chinês" como facto, porque os dados não provam isso. Escrever "possivelmente".
