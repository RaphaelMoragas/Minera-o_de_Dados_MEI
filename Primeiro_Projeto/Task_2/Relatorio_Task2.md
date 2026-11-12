# Relatório — Task 2: Definição do problema de previsão e da avaliação

Estrutura espelhada nas secções pedidas no enunciado da Task 2. Todos os números citados vêm do Projeto_MINDD_Task2.ipynb, que usa o Charging_Data_tratado.csv exportado na Task 1.

---

## 1. Definição do problema

Quanto à unidade de observação, cada linha do dataset é uma sessão de carregamento, ou seja, o modelo faz uma previsão por sessão e temos 441 077 sessões no total. Quanto às classes, a classe positiva é a sessão anormal (is_Abnormal = 1) e a negativa é a sessão normal (is_Abnormal = 0). São 81934 sessões anormais (18,6%) contra 359143 normais (81,4%).

Sobre como o alvo sai do End cause, como foi escolhido na Task 1, decidimos por interpretar as sessões que terminaram com "Charging ends normally" ou "User stops charging" como sendo sessões normais e as que terminaram com qualquer um dos outros 13 motivos, todos falhas (do posto, da comunicação com o veículo, do BMS, etc), são as consideradas sessões anormais.

O momento da previsão é o início da sessão (Start Time). Só pode ser usado o que já se sabe naquele momento, ou seja, variáveis que não sabemos no momento do inicio, não serão consideradas nesse ponto. A previsão serve para monitoramento automático e priorização da manutenção dos postos. Prever uma sessão como anormal significa que a sessão tem risco alto de terminar com falha, então o sistema gera um alerta para o operador acompanhar a sessão e o posto sobe na fila de inspeção. O alerta não bloqueia a sessão nem avisa o usuário, funciona apenas como um alerta ao usuário do carro para que este fique atento.

Consideramos o "User stops charging" como normal porque foi o próprio usuário que decidiu parar, e não uma falha técnica. Mas isso acaba por ser uma limitação do proprio dataset e como a informação ali existe, visto que alguns usuários podem ter parado justamente porque algo estava errado (um carregamento lento, por exemplo), e os dados não nos permitem separar esses casos para uma previsão mais precisa. Infelizmente esse é um problema que não há como contornar. 

---

## 2. Variáveis disponíveis no momento da previsão

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

A categoria 3, geradas durante ou depois da sessão, é a mais importante de deixar de fora. Como vimos durante a Task 1, das sessões com 0 kWh 83,4% são anormais, ou seja, se a energia entrasse no modelo ele aprenderia a regra "0 kWh → anormal" e teria ótimos resultados, mas seria inútil na prática, já que esse valor ainda não existe no momento da previsão.

Na categoria 4, o Order creation time é registado depois do Start Time em 56% das sessões e não traz nada de novo ou importante, e o electricity_price tem só 3 valores, onde sabendo o tariff_period já sabemos o preço. As 6 variáveis redundantes saíram seguindo o critério indicado pela professora (de cada par com correlação acima de 0,9 fica a mais correlacionada com o alvo). Já os IDs são conhecidos no início, mas podem levar o modelo a decorar usuários e postos específicos e não ajudam com usuários ou postos novos, por isso a decisão vai ser tomada na modelação, comparando o modelo com e sem eles.

---

## 3. Consequências dos erros

Um falso positivo (FP) é um alerta numa sessão que seria normal. O custo é baixo (um usuário fica de olho numa sessão sem necessidade ou um técnico verifica um posto que estava bom), mas se acontecer demais os operadores passam a ignorar os alertas. Já um falso negativo (FN) é uma sessão anormal sem alerta. O usuário fica sem carregar e o posto com problema continua a ser usado sem ninguém saber.

Para o uso escolhido, o FN é o erro mais importante. Como vimos durante a Task 1, um posto que falhou recentemente tende a falhar novamente (a taxa de falhas do posto no último dia é a variável mais correlacionada com o alvo, 0,45), ou seja, deixar passar uma falha acaba por afetar também os próximos usuários daquele posto além da não manutenção necessária do mesmo. Mesmo assim o FP não deve ser ignorado: em cada 1 000 sessões existem  cerca de 186 sessões que são anormais, e um modelo que desse alerta em todas apanharia as 186 mas geraria cerca de 814 alertas falsos. Logo, a avaliação deve dar mais peso ao recall sem deixar a precision cair demais.

Também consideramos a intervenção no usuário (avisar para trocar de posto), mas aí os falsos positivos passariam a ser o erro mais caro, já que cada alerta falso poderia encomodar um cliente real. Por isso fica apenas como uma possível extensão para alertas com probabilidade muito alta.

---

## 4. Métricas de avaliação

Um modelo que prevê sempre normal acertaria cerca de 81% das sessões, mas teria recall de 0%. É o accuracy paradox, ou seja, com  aproximadamente 18% de anomalias a acurácia mede mais o desbalanceamento do período do que a qualidade do modelo, e por isso não vai ser usada.

A métrica principal seria a AUC-PR. Pois ela é indicada quando a classe positiva é minoritária e quando os falsos positivos e falsos negativos acabam por ter custos diferentes, que são os nossos dois casos em questão. Como não depende do threshold, serve para comparar modelos e escolher hiperparâmetros antes do threshold acabar por ser definido. A limitação fianel é que o valor de um modelo sem skill é igual à taxa de anomalia, que muda de fold para fold, então cada fold acaba por ter que ser comparado com o seu próprio baseline.

Obs: Há uma limitação visto que num modelo sem skill, a AUC-PR é igual à taxa de anomalia, que muda de fold para fold. Por isso o mesmo valor de AUC-PR não quer dizer o mesmo em todos os folds. Cada fold deve ser lido em relação ao seu próprio baseline.

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

## 5. Estratégia de validação temporal

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

Como vimos na secção 5 da Task 1, a taxa de anomalia muda bastante ao longo do tempo (podendo variar de 25% para 13,7% e de volta para 23%). Com o rolling o modelo aprende só com o passado recente e os treinos ficam todos com o mesmo tamanho. As 180 000 sessões correspondem a cerca de 9 a 10 meses, ou seja, quase um ciclo inteiro de estações. A desvantagem é deixar de usar os meses mais antigos.

Como os folds são feitos pelo número de sessões, todas as validações têm o mesmo tamanho, mas não a mesma duração (cerca de 3,6, 2,5 e 2,2 meses), porque o volume de sessões cresce ao longo de 2021. As validações não se sobrepõem entre si, então cada sessão de 13/01/2021 até 22/09/2021 é avaliada uma única vez apenas.

Tudo o que o modelo aprende dos dados (scaler, one-hot, limite de outliers, etc) é ajustado só no treino de cada fold e depois apenas aplicado na validação, que fica sempre com a distribuição original. O log1p não aprende nada dos dados, por isso pode ser aplicado em tudo.

O teste foi separado logo no início e não entra em nenhuma decisão. Modelos, parametros e threshold são escolhidos só com os 3 folds. No fim, o modelo final é treinado nas 180 000 sessões antes do teste e avaliado uma única vez, sem voltar atrás caso o resultado não seja bom, senão o teste acabaria por virar mais uma validação e o resultado deixaria de ser imparcial.

---

## 6. Agregação entre folds

Cada modelo vai ter um resultado por fold, e precisamos decidir como juntar esses 3 valores num resumo. As formas possíveis são a média simples, a média ponderada pelo tamanho da validação e a pooled, onde juntamos as 3 validações numa só e calculamos a métrica uma única vez.

Como as 3 validações têm o mesmo número de sessões, a média ponderada acaba por dar exatamente o mesmo valor que a média simples. Já na pooled os folds 2 e 3, que têm mais anomalias, acabam por pesar mais do que o fold 1. Na média simples cada período conta o mesmo.

Logo, para comparar os modelos vamos usar a média simples da AUC-PR, já que queremos que o modelo funcione bem em todos os períodos e não só no maior. Junto com ela vamos sempre mostrar o desvio-padrão, o pior fold e o valor de cada fold, porque a média sozinha pode esconder muita variação entre os períodos. Também vamos contar em quantos dos 3 folds o modelo passa da regra, visto que um modelo que ganha na média mas perde em algum fold não é confiável. Já a forma pooled vai ser usada para escolher o threshold e para as métricas operacionais por 1 000 sessões, já que aí faz sentido juntar todas as sessões de validação.

Quanto à evolução no tempo, a taxa de anomalia sobe de 15,8% no fold 1 para 19,7% e 21,0% nos folds 2 e 3. Como a AUC-PR de um modelo sem skill acompanha essa taxa, um modelo pode parecer que melhora ao longo do tempo só porque o período tem mais anomalias, e por isso a evolução no tempo tem que ser lida sempre em relação ao baseline de cada fold.

---

## 7. Baseline

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

## 8. Protocolo de avaliação

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

