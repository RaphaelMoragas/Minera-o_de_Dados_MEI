# 4. Otimização de Modelos e Seleção do Limiar

## 4.1. Otimização de hiperparâmetros

Foram otimizados os modelos **Random Forest** e **HistGradientBoosting**, utilizando apenas os dados de desenvolvimento e validação cruzada temporal. A pesquisa foi realizada com **BayesSearchCV**, tendo o F1-score da classe positiva (`is_Abnormal = 1`) como métrica de otimização.

| Modelo | Hiperparâmetro pesquisado |
|---|---|
| Random Forest | `min_samples_split` |
| HistGradientBoosting | `learning_rate` |

As configurações finais foram escolhidas pelo melhor F1-score médio nas divisões temporais.

| Modelo | F1-score inicial | F1-score otimizado | Variação |
|---|---:|---:|---:|
| Random Forest | 0,4483 | 0,5464 | +0,0981 |
| HistGradientBoosting | 0,5211 | 0,5218 | +0,0007 |

Valores positivos na coluna “Variação” indicam melhoria após a otimização. O modelo selecionado foi **Random Forest**, por apresentar o melhor equilíbrio entre desempenho, estabilidade temporal e deteção de anomalias.

## 4.2. Seleção do limiar

O limiar foi selecionado a partir de previsões *out-of-fold* temporais, maximizando o F1-score da classe positiva. O valor escolhido foi **0,64** e foi mantido fixo na avaliação final.

| Métrica | Limiar 0,50 | Limiar selecionado (0,64) |
|---|---:|---:|
| Precisão | 0,4246 | 0,5774 |
| Recall | 0,6914 | 0,5423 |
| F1-score | 0,5261 | 0,5593 |
| Taxa de alertas (%) | 30,67% | 17,69% |
| Alertas corretos por 1 000 sessões | 130,2185 | 102,1368 |
| Falsos alertas por 1 000 sessões | 176,4628 | 74,7479 |
| Sessões anormais não detetadas por 1 000 sessões | 58,1345 | 86,2161 |

A alteração do limiar representa um compromisso entre aumentar a deteção de sessões anormais e limitar os falsos alertas. A escolha final deve considerar tanto o F1-score como o impacto operacional dos alertas.

## 4.3. Interpretação do modelo final

Foi utilizada a importância das variáveis (`feature_importances_`) do **Random Forest** para identificar as variáveis mais relevantes nas previsões do modelo final. As importâncias foram associadas aos nomes gerados pelo `ColumnTransformer` por meio de `get_feature_names_out()` e ordenadas em ordem decrescente.

| Posição | Variável | Importância/efeito |
|---:|---|---:|
| 1 | `remainder__post_previous_abnormal_rate_7D` | 0,1513 |
| 2 | `remainder__post_previous_abnormal_rate_1D` | 0,1460 |
| 3 | `remainder__post_hours_since_previous_completed...` | 0,1320 |

As importâncias indicam associações utilizadas pelo modelo e não relações causais. Assim, os resultados devem ser interpretados em conjunto com o contexto dos dados e com as limitações do método aplicado.