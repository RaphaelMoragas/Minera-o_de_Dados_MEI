# 4. Otimização de Modelos e Seleção do Limiar de Classificação

## 4.1. Otimização de Hiperparâmetros

Após a avaliação inicial dos modelos, foram selecionados três modelos promissores para otimização de hiperparâmetros: **Random Forest** e **HistGradientBoosting**. A seleção teve em consideração o desempenho inicial na identificação da classe positiva (`is_Abnormal = 1`), a capacidade de generalização e a adequação dos modelos ao problema em estudo.

O objetivo desta etapa consiste em identificar configurações que permitam melhorar o desempenho preditivo sem comprometer a capacidade de generalização para sessões futuras.

### 4.1.1. Procedimento de pesquisa

A otimização foi realizada exclusivamente com os dados de desenvolvimento, sem recorrer ao conjunto de teste final. Para evitar a utilização de informação futura na construção dos modelos, foi adotada uma estratégia de validação cruzada temporal, na qual os dados de treino de cada divisão precedem cronologicamente os dados de validação.

Foi utilizado o método **RandomizedSearchCV** no Random Forest, e o **BayesSearchCV** no HistGradientBoosting, permitindo explorar de forma eficiente os espaços de pesquisa definidos para cada modelo. O número de iterações foi definido de forma a garantir uma exploração adequada do espaço de hiperparâmetros, mantendo um equilíbrio entre o tempo de execução e a qualidade da otimização.

A avaliação do desempenho de cada configuração foi realizada com base na métrica **F1-score da classe positiva**, já que não podemos utilizar a Average Precision como métrica de otimização, dado que esta não é suportada diretamente pelos métodos de pesquisa utilizados. A métrica F1-score permite equilibrar a precisão e a recuperação da classe positiva, sendo adequada para problemas com classes desbalanceadas.

Os espaços de pesquisa foram definidos de acordo com os hiperparâmetros relevantes de cada algoritmo:

| Modelo | Hiperparâmetros explorados |
|---|---|
| Random Forest | `n_estimators`: 100, 200, 300, 400 ou 500; `max_depth`: None, 10, 20 ou 30; `min_samples_split`: 2, 5 ou 10; `max_features`: 1.0, `sqrt` ou `log2` |
| HistGradientBoosting | `max_iter`: inteiro entre 100 e 500; `max_depth`: None, 10, 20 ou 30; `learning_rate`: valor real entre 0,01 e 0,2, com distribuição log-uniforme; `min_samples_leaf`: inteiro entre 20 e 150 |

A configuração selecionada para cada modelo corresponde à que obteve o melhor resultado médio na métrica de otimização durante a validação temporal. O número de iterações, as divisões temporais utilizadas e as restantes opções do procedimento de pesquisa foram mantidos documentados, de forma a garantir a reprodutibilidade dos resultados.

### 4.1.2. Comparação com as configurações iniciais

Após a otimização, os resultados dos modelos foram comparados com os obtidos pelas configurações iniciais, utilizando as mesmas divisões temporais e métricas de avaliação.

A tabela seguinte resume a comparação:

| Modelo | F1-score inicial | F1-score otimizado | Variação |
|---|---:|---:|---:|
| [Modelo 1] | [Valor] | [Valor] | [Valor] |
| [Modelo 2] | [Valor] | [Valor] | [Valor] |

A variação absoluta da métrica foi calculada através da diferença entre o resultado otimizado e o resultado inicial. Valores positivos indicam uma melhoria da Average Precision.

A análise dos resultados permite identificar se a otimização produziu melhorias relevantes ou se o desempenho permaneceu semelhante ao das configurações iniciais. No entanto, uma melhoria na média não garante, por si só, maior capacidade de generalização. Por esse motivo, foram também considerados os resultados obtidos nas diferentes divisões temporais, procurando identificar variações acentuadas entre períodos.

O modelo selecionado para a etapa seguinte foi **[modelo selecionado]**, por apresentar o melhor compromisso entre desempenho preditivo, estabilidade temporal e capacidade de identificação das sessões anormais.

## 4.2. Seleção do Limiar de Classificação

Os classificadores probabilísticos utilizam habitualmente um limiar de 0,50 para converter as probabilidades previstas em classes. Contudo, este valor não é necessariamente o mais adequado para o problema em estudo, uma vez que o objetivo operacional envolve a identificação de sessões anormais e a redução das consequências associadas a previsões incorretas.

Assim, para o modelo probabilístico selecionado, foi investigado o efeito da alteração do limiar de classificação, procurando encontrar um equilíbrio adequado entre a precisão e a recuperação da classe positiva.

### 4.2.1. Critério de seleção

Foi utilizado o critério de **maximização do F1-score da classe positiva**, que corresponde à média harmónica entre a precisão e a recuperação (*recall*).

Este critério permite equilibrar a proporção de alertas corretos com a capacidade de identificar as sessões efetivamente anormais. Um limiar demasiado elevado pode reduzir o número de falsos alertas, mas aumentar o número de sessões anormais não detetadas. Inversamente, um limiar mais baixo tende a aumentar a deteção de casos positivos, podendo também aumentar o número de falsos alertas.

O limiar foi selecionado utilizando previsões de validação geradas a partir dos dados de desenvolvimento, sem utilizar as observações empregues no ajuste do modelo correspondente. Para esse efeito, foram utilizadas previsões *out-of-fold* obtidas através de divisões temporais, garantindo que cada previsão de validação é produzida por um modelo treinado apenas com dados cronologicamente anteriores.

Foram avaliados diferentes limiares candidatos, calculando o F1-score correspondente a cada um. O limiar escolhido foi aquele que maximizou esta métrica nos dados de desenvolvimento.

O valor selecionado foi **[limiar selecionado]**, tendo sido posteriormente mantido fixo durante a avaliação no conjunto de teste final. Desta forma, o teste final não influenciou a escolha do limiar, evitando uma avaliação excessivamente otimista do desempenho.

### 4.2.2. Comparação entre limiares

O comportamento do modelo foi comparado utilizando o limiar predefinido de 0,50 e o limiar selecionado durante a validação.

Foram consideradas as seguintes métricas:

- **Precisão:** proporção de alertas emitidos que correspondem efetivamente a sessões anormais.
- **Recall:** proporção de sessões anormais corretamente identificadas.
- **F1-score:** equilíbrio entre precisão e recall.
- **Taxa de alertas:** proporção de sessões classificadas como anormais.
- **Alertas corretos por 1 000 sessões:** número de sessões anormais corretamente identificadas por cada 1 000 sessões avaliadas.
- **Falsos alertas por 1 000 sessões:** número de sessões normais incorretamente classificadas como anormais por cada 1 000 sessões.
- **Sessões anormais não detetadas por 1 000 sessões:** número de sessões anormais classificadas incorretamente como normais por cada 1 000 sessões.

Os resultados obtidos encontram-se resumidos na tabela seguinte.

| Métrica | Limiar 0,50 | Limiar selecionado |
|---|---:|---:|
| Precisão | [Valor] | [Valor] |
| Recall | [Valor] | [Valor] |
| F1-score | [Valor] | [Valor] |
| Taxa de alertas (%) | [Valor] | [Valor] |
| Alertas corretos por 1 000 sessões | [Valor] | [Valor] |
| Falsos alertas por 1 000 sessões | [Valor] | [Valor] |
| Sessões anormais não detetadas por 1 000 sessões | [Valor] | [Valor] |

A comparação permite avaliar o impacto da alteração do limiar no comportamento operacional do classificador. Uma diminuição do limiar poderá aumentar o recall e o número de alertas corretos, mas também poderá elevar a quantidade de falsos alertas. Por outro lado, um aumento do limiar poderá reduzir o volume de alertas emitidos, à custa de uma eventual diminuição da capacidade de deteção.

A interpretação dos resultados deve, por isso, considerar não apenas as métricas preditivas, mas também os recursos necessários para analisar os alertas e as consequências das sessões anormais não identificadas. O limiar selecionado representa o compromisso obtido segundo o critério definido, não sendo necessariamente o mais adequado para todos os cenários operacionais.

## 4.3. Avaliação Final

Depois de concluída a seleção dos hiperparâmetros e do limiar, o modelo final foi avaliado no conjunto de teste reservado para esse efeito. Este conjunto representa um período temporal posterior ao utilizado no desenvolvimento e permaneceu excluído de todas as decisões de otimização.

O modelo foi avaliado com a configuração selecionada e com o limiar fixado durante a validação. O desempenho foi analisado através das métricas preditivas e operacionais anteriormente definidas, permitindo avaliar a capacidade de generalização para sessões futuras.

Os resultados do teste final devem ser interpretados como uma estimativa do desempenho em dados não utilizados no desenvolvimento, tendo em consideração que alterações temporais na distribuição das observações podem afetar a capacidade de generalização do modelo.

## 4.4. Interpretação do Modelo Final

Para compreender os fatores associados às previsões do modelo final, foi utilizada uma técnica de interpretação adequada ao algoritmo selecionado: **[método de interpretação]**.

A análise procurou identificar as variáveis ou grupos de variáveis que mais influenciam as previsões de anomalia, permitindo compreender quais os fatores que contribuem para a identificação de sessões potencialmente anormais.

Os resultados da interpretação foram analisados considerando a natureza do modelo e as limitações do método utilizado. A importância de uma variável não implica necessariamente uma relação causal com a ocorrência de uma anomalia, podendo refletir associações presentes nos dados. Do mesmo modo, a interpretação da importância global das variáveis não permite, por si só, concluir se o aumento de uma variável conduz a uma maior ou menor probabilidade de classificação como anormal.

A análise contribui, assim, para uma melhor compreensão do comportamento do classificador e para a identificação dos fatores mais relevantes para as suas previsões, sem ultrapassar as conclusões suportadas pelos dados e pelo método de interpretação adotado.