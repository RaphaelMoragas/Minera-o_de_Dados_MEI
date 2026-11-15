# Task 1 — Time-Series Construction, Exploratory Analysis, and Forecasting Framework

## 1. Introduction

The objective of this task is to transform the supplied transaction-level electric vehicle (EV) charging dataset into coherent hourly time series suitable for forecasting charging demand at both the complete network and individual district levels. In addition, an operational forecasting formulation and a chronological evaluation protocol are established before proceeding to model development, optimisation and selection.

The forecasting problem is defined as a day-ahead prediction task, in which forecasts are issued at 00:00 for the following 24 hours. This operational constraint determines which historical observations, calendar variables and external information can legitimately be used as predictors.

Four forecasting targets are considered:

- **Network:** Total charging demand across the complete network.
- **Nanhu:** Charging demand in the Nanhu district.
- **Tongxiang:** Charging demand in the Tongxiang district.
- **Xiuzhou:** Charging demand in the Xiuzhou district.

The task is organised into three main stages: data preparation and exploratory analysis, operational forecasting formulation, and evaluation methodology.

## 2. Data Preparation and Exploratory Analysis

### 2.1 Dataset Description and Temporal Coverage

The supplied dataset contains **441,077 transaction records and 24 variables** describing EV charging activity.

The dataset includes transaction identifiers, charging-post information, geographical attributes, timestamps, energy consumption, financial information, weather measurements and operational indicators.

The principal variables used to construct charging demand are:

| Variable | Description |
|---|---|
| `Start Time` | Timestamp associated with the charging transaction |
| `Transaction power/kwh` | Energy consumed during the transaction |
| `District Name` | Geographical district associated with the transaction |
| `Temperature(ºC)` | Recorded temperature |
| `Relative Humidity(%)` | Recorded relative humidity |
| `Precipitation(mm)` | Recorded precipitation |

The transaction timestamps cover the period from **31 December 2019 to 31 December 2021**.

The hourly dataset initially contains a partial first day, beginning at 21:00 on 31 December 2019. Consequently, the exploratory comparison of complete calendar days starts on 1 January 2020.

The observation period includes 731 complete calendar days across 2020 and 2021, including the leap day in February 2020.

### 2.2 Data Quality Assessment

The initial data-quality assessment identified 441,077 transaction records, with no apparent missing values or duplicate rows in the supplied dataset.

The transaction timestamps were converted into datetime values to support chronological aggregation and time-based analysis.

The main preparation decisions are summarised below.

| Data-quality aspect | Finding and treatment |
|---|---|
| Dataset dimensions | 441,077 records and 24 variables |
| Missing transaction values | No apparent missing values |
| Duplicate records | No apparent duplicated rows |
| Timestamp format | Converted to datetime |
| Initial observation | Partial day beginning 31 December 2019 at 21:00 |
| Energy measurements | Positive charging-energy observations used for demand aggregation |
| Missing hourly intervals | Continuous hourly index constructed |
| Zero-demand intervals | Retained as genuine observations |
| District consistency | Network demand checked against the sum of district demand |

The assessment distinguishes data-quality problems from legitimate changes in charging activity.

For example, an hour without recorded charging transactions does not necessarily indicate missing information. It may represent a period during which no charging sessions occurred.

Similarly, unusually low demand during February should not automatically be interpreted as invalid data. Such periods require investigation because they may reflect operational disruptions, unusual external conditions or changes in the underlying charging activity.

### 2.3 Construction of Hourly Charging-Demand Series

The original transaction-level dataset is transformed into hourly charging-demand series using the sum of energy consumed within each hour.

For a given hourly interval $t$, network demand is defined as:

$$
D_{\text{Network},t} = \sum_{i \in T_t} E_i
$$

where:

- **$T_t$** represents the transactions assigned to hour **$t$**.
- **$E_i$** represents the energy consumption of transaction **$i$**, measured in kWh.

The same aggregation procedure is applied independently to Nanhu, Tongxiang and Xiuzhou.

The resulting series are combined into a common hourly DataFrame containing four columns:

$$
D_t =
\begin{bmatrix}
D_{\text{Network},t} &
D_{\text{Nanhu},t} &
D_{\text{Tongxiang},t} &
D_{\text{Xiuzhou},t}
\end{bmatrix}
$$

A complete hourly datetime index is constructed to ensure that all observations are equally spaced.

Intervals without recorded transactions are assigned a demand value of zero.

This is important because zero-demand observations represent legitimate charging conditions and must be retained for the subsequent forecasting analysis.

The network series is also verified against the sum of the three district series:

$$
D_{\text{Network},t}
=
D_{\text{Nanhu},t}
+
D_{\text{Tongxiang},t}
+
D_{\text{Xiuzhou},t}
$$

The equality holds for every hour, confirming consistency between the geographical aggregation levels.

The complete hourly series contains **491 zero-demand observations**, which are retained rather than treated as missing values.

### 2.4 Intraday Seasonality

The first exploratory analysis investigates the average network demand across the 24 hours of the day.

The results reveal a strong intraday pattern, with substantial differences between overnight, morning, midday and evening charging activity.

Network demand reaches its average hourly maximum at **12:00, with 683.16 kWh**, while the lowest average occurs at **05:00, with 88.78 kWh**.

The main characteristics are:

- Charging activity is relatively low during the early morning, particularly between 04:00 and 06:00.
- Demand begins increasing from approximately 06:00.
- A pronounced increase occurs between 08:00 and 11:00.
- The highest average demand occurs around midday, remaining elevated between 11:00 and 15:00.
- A secondary evening increase occurs between approximately 18:00 and 22:00.

The median hourly demand follows a similar pattern, reaching 720.48 kWh at 12:00.

Variability is also higher during periods of stronger charging activity. The largest hourly standard deviation occurs at 11:00, reaching 306.79 kWh.

The maximum observed hourly demand is **1,363.87 kWh**, recorded on 24 November 2021 at 11:00.

These results demonstrate the importance of hourly calendar features and historical observations corresponding to previous days.

### 2.5 Weekly and Calendar Seasonality

Weekly patterns are investigated by comparing demand across individual weekdays and between weekdays and weekends.

The average hourly demand by weekday is relatively stable, although some differences are observable.

| Day | Average hourly demand (kWh) |
|---|---:|
| Monday | 411.51 |
| Tuesday | 405.10 |
| Wednesday | 404.84 |
| Thursday | 408.90 |
| Friday | 425.65 |
| Saturday | 423.78 |
| Sunday | 412.38 |

Friday records the highest average hourly demand, while Wednesday records the lowest.

The comparison between weekdays and weekends produces the following results:

| Day type | Average hourly demand (kWh) | Standard deviation (kWh) |
|---|---:|---:|
| Weekday | 411.21 | 293.73 |
| Weekend | 418.08 | 291.89 |

Weekend demand is slightly higher than weekday demand, although the difference is relatively small.

This suggests that charging activity is distributed throughout the week rather than being strongly concentrated on working days.

Monthly seasonality is also evident.

Average hourly demand is lowest in February, at 139.26 kWh, and reaches its highest monthly average in August, at 529.04 kWh.

The monthly results indicate a general increase in charging activity from spring into the second half of the year.

The standard deviation also increases during higher-demand months, reaching 314.39 kWh in December.

This suggests that demand variability increases alongside the overall level of charging activity.

### 2.6 Changes in Demand Patterns

The temporal analysis reveals substantial changes in charging demand throughout the observation period.

Average hourly network demand increased from **140.34 kWh in January 2020** to **612.64 kWh in December 2021**.

Average daily demand also increased from approximately 3,368.27 kWh to 14,703.32 kWh.

The half-year comparison further illustrates this evolution.

| Period | Average hourly demand (kWh) |
|---|---:|
| 2020-H1 | 225.48 |
| 2020-H2 | 469.78 |
| 2021-H1 | 377.71 |
| 2021-H2 | 577.24 |

The strongest increase occurred during the second half of 2020. Demand continued to grow in 2021, with average hourly demand increasing by approximately 53% between the first and second halves of that year.

Comparing the second halves of both years indicates an additional increase of approximately 23%.

February represents an important period of unusually low demand in both years.

In February 2020, average hourly demand fell to only 29.68 kWh. February 2021 also recorded substantially lower demand than surrounding months, at 252.75 kWh.

These periods may represent operational disruptions, abnormal external conditions or data-quality effects. However, the available analysis does not establish a definitive cause.

Peak demand also increased throughout the observation period.

The highest hourly demand reached 1,363.87 kWh on 24 November 2021 at 11:00, while the highest daily demand reached 17,180.71 kWh on 1 October 2021.

The standard deviation of hourly demand increased from 117.61 kWh in January 2020 to 333.42 kWh in December 2021.

These observations indicate that the demand process is not stationary in its overall level and variability.

Consequently, forecasting models must account for long-term changes in demand, calendar seasonality and potential structural disruptions.

### 2.7 District-Level Demand Comparison

The three districts exhibit different demand magnitudes and temporal characteristics.

The following table summarises their main descriptive statistics.

| Measure | Nanhu | Tongxiang | Xiuzhou |
|---|---:|---:|---:|
| Total demand (kWh) | 1,249,915.68 | 3,819,981.38 | 2,179,434.81 |
| Average hourly demand (kWh) | 71.24 | 217.74 | 124.23 |
| Network contribution (%) | 17.24 | 52.69 | 30.06 |
| Peak hourly demand (kWh) | 551.50 | 845.30 | 632.32 |
| Hourly standard deviation (kWh) | 70.97 | 164.26 | 106.76 |
| Coefficient of variation | 1.00 | 0.75 | 0.86 |

Tongxiang is the dominant district, contributing 52.69% of total network demand.

Xiuzhou contributes 30.06%, while Nanhu contributes 17.24%.

Tongxiang therefore represents the largest component of network charging activity and is particularly important for network-level forecasting.

However, the districts also exhibit different temporal characteristics.

The average hourly profiles show that:

- Nanhu reaches its highest average demand at 11:00, with 161.47 kWh.
- Tongxiang reaches its highest average demand at 14:00, with 342.69 kWh.
- Xiuzhou reaches its highest average demand at 12:00, with 216.77 kWh.

All three districts generally experience their lowest demand during the early morning, particularly between 04:00 and 06:00.

Demand increases substantially after 09:00, although the magnitude and timing of the peaks differ.

Nanhu has the highest coefficient of variation, approximately 1.00, indicating the greatest relative variability.

Tongxiang has the lowest coefficient of variation, approximately 0.75, suggesting comparatively more stable demand relative to its average level.

These differences justify forecasting the network and districts separately rather than assuming that the same demand profile applies across all geographical areas.

### 2.8 Exploratory Analysis Conclusions

The exploratory analysis identifies five important characteristics:

1. **Strong intraday seasonality:** Demand is lowest during the early morning, peaks around midday and exhibits a secondary evening increase.
2. **Moderate weekly variation:** Weekend demand is slightly higher than weekday demand, but the differences are relatively small.
3. **Long-term demand growth:** Charging demand increases substantially throughout the observation period.
4. **Structural variations:** February contains unusually low-demand periods that require particular attention.
5. **Geographical differences:** Tongxiang dominates network demand, while Nanhu exhibits the greatest relative variability and district-specific peak times.

These findings support the use of calendar features, historical demand predictors, trend information and district-specific forecasting models.

## 3. Operational Forecasting Formulation

### 3.1 Forecast Issuance Schedule

The forecasting problem is defined as a day-ahead prediction task.

Forecasts are issued at precisely **00:00** for the following 24 hourly intervals.

The prediction horizon covers:

$$
h = 1, 2, \ldots, 24
$$

The information cutoff corresponds to **23:00 on the previous day**, representing the latest completed hourly observation available when the forecast is issued.

For example, a forecast issued at 00:00 on 1 January uses historical information available through 23:00 on 31 December.

The forecasting system must produce 24 predictions for each of the four targets:

- Network demand.
- Nanhu demand.
- Tongxiang demand.
- Xiuzhou demand.

### 3.2 Multi-Step Forecasting Strategy

A direct multi-step forecasting strategy is adopted.

Under this approach, one model is trained for each forecasting horizon, from one to 24 hours ahead.

This allows each model to learn the characteristics associated with its specific prediction horizon.

All predictions must use information available at the same forecast issuance time.

This restriction prevents the use of actual demand observations from earlier hours of the forecast day when predicting later hours.

The strategy therefore reproduces a realistic day-ahead forecasting scenario rather than a sequence of independent one-hour-ahead predictions.

### 3.3 Predictor Selection and Feature Engineering

Predictors are selected according to their availability at the 00:00 forecast issuance time.

The available information is divided into known future information, historical observations and external variables.

#### 3.3.1 Calendar and Temporal Features

Calendar variables are known in advance for every hour of the forecast day.

The selected features include:

- Hour of the day.
- Day of the week.
- Weekend indicator.
- Day of the month.
- Month.
- Quarter.
- Cyclical hour encoding.
- Cyclical weekday encoding.
- Linear time trend.

Cyclical encoding represents recurring temporal variables using sine and cosine transformations:

$$
x_{\sin} = \sin\left(\frac{2\pi x}{P}\right)
$$

$$
x_{\cos} = \cos\left(\frac{2\pi x}{P}\right)
$$

where $P$ represents the corresponding seasonal period.

For example, the hourly period uses $P = 24$, while the weekly period uses $P = 7$.

The linear time trend is included to represent the substantial long-term growth observed throughout the dataset.

#### 3.3.2 Historical Demand Features

Historical demand is considered one of the most important sources of predictive information.

The selected lag periods are:

| Lag | Historical reference |
|---|---|
| 1 hour | Most recent completed observation |
| 24 hours | Same hour on the previous day |
| 48 hours | Same hour two days earlier |
| 168 hours | Same hour one week earlier |
| 336 hours | Same hour two weeks earlier |
| 672 hours | Same hour four weeks earlier |

Additional historical aggregates include:

- Previous-day total demand.
- Previous-week total demand.
- Rolling mean.
- Rolling minimum.
- Rolling maximum.
- Rolling standard deviation.
- Number of zero-demand hours.

Rolling statistics are calculated over 24-hour, 168-hour and 672-hour windows.

All historical features are anchored to the 23:00 information cutoff and are calculated using observations available before forecast issuance.

For the direct forecasting strategy, the same historical information cutoff must be maintained across all 24 prediction horizons.

#### 3.3.3 Weather Features

The original dataset includes three observed weather variables:

- Temperature.
- Relative humidity.
- Precipitation.

Weather information requires particular attention because future observed weather conditions are not available at the time of day-ahead forecast issuance.

The following historical weather summaries are constructed:

| Variable | Aggregation |
|---|---|
| Temperature | Previous-day mean |
| Temperature | Previous-day minimum |
| Temperature | Previous-day maximum |
| Relative humidity | Previous-day mean |
| Precipitation | Previous-day total |

These daily summaries are shifted by one day before being aligned with the hourly forecasting index.

Longer historical weather summaries may also be used, provided that their calculation does not incorporate information from the forecast day.

Future observed weather measurements are excluded.

They may only be incorporated if actual weather forecasts corresponding to the 00:00 issuance time are available and can be used without information leakage.

The current methodology therefore uses historical weather information rather than retrospective observations of the target day.

#### 3.3.4 Operational Features

The dataset also contains operational indicators describing charging outcomes and potential faults.

The selected variables include:

- `is_full_charge`
- `is_user_stop`
- `is_abnormal`
- `is_nonsystem_fault`
- `is_system_issue`
- `is_EV_fault`
- `is_other_fault`

For each variable, daily sums and averages are calculated.

The resulting operational features are shifted by one day before being incorporated into the hourly predictor dataset.

This ensures that the models use historical operational information rather than information generated during the forecast period.

The operational variables provide additional context about historical charging activity and the occurrence of abnormal charging conditions.

### 3.4 Exogenous Predictor Groups

The selected predictors are organised into four groups:

| Group | Main information |
|---|---|
| Calendar | Hour, weekday, weekend, month, quarter, cyclical features and trend |
| Demand | Historical demand lags, daily and weekly totals, rolling statistics and zero-demand counts |
| Weather | Historical temperature, humidity and precipitation summaries |
| Operational | Historical charging outcomes and fault-related aggregates |

These groups are combined into a single exogenous predictor dataset for subsequent SARIMAX experiments.

The initial feature-engineering process produces 122 predictor columns after removing duplicated columns.

The first complete day for which all predictors are available is 29 January 2020. This results from the historical lookback requirements, particularly the 672-hour demand lag.

The resulting modelling dataset contains:

- 16,872 hourly observations.
- 122 predictor columns.
- Four hourly demand targets.

The exclusion of initial observations is necessary because the longest historical lag cannot be calculated before sufficient historical information is available.

### 3.5 Information Leakage Prevention

The principal requirement is that every predictor must be available at the 00:00 issuance time.

The following restrictions are therefore applied:

- Actual demand during the forecast day is never used as a predictor.
- Transaction-level information recorded during future charging sessions is excluded.
- Future observed weather is excluded.
- Historical demand features use only observations available up to 23:00.
- Weather and operational aggregates are shifted to use previous-day information.
- Calendar features are generated deterministically for the future forecast hours.

These restrictions ensure that the forecasting problem represents a realistic operational scenario.

## 4. Evaluation Protocol

### 4.1 Chronological Data Partitioning

The dataset is divided chronologically into development and independent testing periods.

The first 80% of the available forecasting days are allocated to model development, while the final 20% are reserved for independent testing.

The division is performed using complete forecasting days rather than randomly selecting individual hourly observations.

This preserves the chronological structure of the dataset and prevents future observations from entering the training period.

For the complete hourly series, the split is based on the 731 complete calendar days.

For the predictor-complete SARIMAX dataset, the split is performed independently after the initial historical lookback period has been removed.

### 4.2 Expanding-Window Validation

Model development uses four expanding-window validation folds.

Each validation fold contains 30 consecutive days.

The training window expands as additional historical observations become available.

For the predictor-complete dataset, the chronological periods are:

| Fold | Training period | Validation period |
|---|---|---|
| 1 | 29 Jan 2020 – 14 Apr 2021 | 15 Apr – 14 May 2021 |
| 2 | 29 Jan 2020 – 14 May 2021 | 15 May – 13 Jun 2021 |
| 3 | 29 Jan 2020 – 13 Jun 2021 | 14 Jun – 13 Jul 2021 |
| 4 | 29 Jan 2020 – 13 Jul 2021 | 14 Jul – 12 Aug 2021 |

The corresponding independent test period extends from **13 August 2021 to 31 December 2021**.

This expanding-window structure reflects an operational setting in which the forecasting system progressively accumulates historical information.

### 4.3 Independent Final Test Period

The final 20% of forecasting days are reserved for independent testing.

This period must not be used for:

- Predictor selection.
- Model optimisation.
- Parameter tuning.
- Forecasting-method selection.

The final test period is used only after the model-development decisions have been completed.

This provides an independent assessment of forecasting performance on previously unseen chronological observations.

### 4.4 Baseline Forecasting Methods

Three baseline methods are established.

**Naïve**

The latest available demand observation is used as the prediction:

$$
\widehat{D}_{T+h} = D_{T-1}
$$

**Seasonal Naïve (24h)**

Demand from the corresponding hour on the previous day is used:

$$
\widehat{D}_{T+h} = D_{T+h-24}
$$

**Seasonal Naïve (168h)**

Demand from the corresponding hour one week earlier is used:

$$
\widehat{D}_{T+h} = D_{T+h-168}
$$

The 24-hour seasonal baseline represents daily recurring demand patterns, while the 168-hour baseline represents weekly recurring patterns.

These methods provide reference performance levels against which more complex forecasting models can be evaluated.

### 4.5 Evaluation Metrics

Forecasting performance is assessed using MAE, RMSE and MASE.

#### Mean Absolute Error

$$
MAE =
\frac{1}{n}
\sum_{i=1}^{n}|y_i-\widehat{y}_i|
$$

MAE measures the average absolute forecasting error in kWh.

Lower values indicate better predictive accuracy.

#### Root Mean Squared Error

$$
RMSE =
\sqrt{
\frac{1}{n}
\sum_{i=1}^{n}(y_i-\widehat{y}_i)^2
}
$$

RMSE penalises larger errors more strongly and is particularly relevant when substantial forecasting errors could affect infrastructure management.

#### Mean Absolute Scaled Error

MASE provides a scale-independent measure of forecasting accuracy.

The adopted seasonal period is 24 hours because the forecasting task concerns hourly demand and a complete day-ahead prediction horizon.

$$
MASE =
\frac{MAE}
{
\frac{1}{N-24}
\sum_{t=25}^{N}|y_t-y_{t-24}|
}
$$

The scaling denominator is calculated exclusively from the training observations.

A MASE value below 1 indicates that the model achieves lower scaled absolute error than the corresponding daily seasonal naïve reference.

The 168-hour seasonal naïve method is retained as an additional benchmark to evaluate weekly recurring demand patterns.

### 4.6 Preliminary Baseline Evaluation

The notebook includes preliminary baseline experiments using the four forecasting targets and the three benchmark methods.

The following table reports the mean validation performance across the four folds.

| Model | Target | MAE (kWh) | RMSE (kWh) | MASE |
|---|---|---:|---:|---:|
| Naïve | Network | 138.313 | 180.044 | 1.364 |
| Seasonal Naïve (24h) | Network | 117.530 | 156.243 | 1.159 |
| Seasonal Naïve (168h) | Network | 125.191 | 166.380 | 1.235 |
| Naïve | Nanhu | 43.695 | 58.543 | 1.054 |
| Seasonal Naïve (24h) | Nanhu | 41.831 | 56.392 | 1.009 |
| Seasonal Naïve (168h) | Nanhu | 40.972 | 55.598 | 0.988 |
| Naïve | Tongxiang | 94.334 | 121.653 | 1.366 |
| Seasonal Naïve (24h) | Tongxiang | 80.664 | 107.192 | 1.168 |
| Seasonal Naïve (168h) | Tongxiang | 85.792 | 113.599 | 1.242 |
| Naïve | Xiuzhou | 73.575 | 96.685 | 1.383 |
| Seasonal Naïve (24h) | Xiuzhou | 68.982 | 91.266 | 1.297 |
| Seasonal Naïve (168h) | Xiuzhou | 70.194 | 93.324 | 1.320 |

The preliminary results suggest that seasonal naïve methods outperform the simple naïve method across all four targets.

Seasonal Naïve (24h) achieves the lowest MAE for Network, Tongxiang and Xiuzhou.

Seasonal Naïve (168h) achieves the lowest MAE for Nanhu.

The results demonstrate that historical daily and weekly demand patterns provide useful forecasting information.

However, these values should be interpreted as preliminary experimental results rather than final day-ahead forecasting performance, because the current baseline implementation uses actual observations from within the validation period for subsequent predictions. The strict issuance-time restriction must be applied before these metrics can be considered operationally valid.

### 4.7 Preliminary Statistical Model Evaluation

The notebook also evaluates a SARIMA model using the following configuration:

$$
SARIMA(1,1,0)(1,0,0)_{24}
$$

The model is fitted independently for each forecasting target.

Its mean validation performance is:

| Target | MAE (kWh) | RMSE (kWh) | MASE |
|---|---:|---:|---:|
| Network | 242.929 | 295.987 | 2.396 |
| Nanhu | 55.545 | 65.885 | 1.340 |
| Tongxiang | 145.319 | 172.781 | 2.104 |
| Xiuzhou | 97.319 | 116.930 | 1.826 |

In these preliminary experiments, SARIMA produces higher average errors than the seasonal naïve methods for all four targets.

Its average MASE exceeds 1 across every target, indicating that it does not outperform the daily seasonal naïve reference.

The results suggest that the simple recurring demand patterns captured by the seasonal naïve models provide a strong reference that more complex statistical models must improve upon.

Nevertheless, the current SARIMA experiment forecasts entire 30-day validation blocks rather than generating independent 24-hour forecasts from daily 00:00 issuance times. Consequently, the results are useful for preliminary model exploration but cannot yet establish the final ranking of day-ahead forecasting methods.

## 5. Final Methodological Protocol and Implementation Requirements

The methodology established in this task defines the following requirements for subsequent model development.

| Component | Established methodology |
|---|---|
| Forecasting targets | Network, Nanhu, Tongxiang and Xiuzhou |
| Target frequency | Hourly |
| Forecast issuance | 00:00 |
| Forecast horizon | 24 hours |
| Information cutoff | 23:00 on the previous day |
| Forecasting strategy | Direct multi-step |
| Historical demand lags | 1, 24, 48, 168, 336 and 672 hours |
| Historical aggregates | Previous-day and previous-week totals |
| Rolling windows | 24, 168 and 672 hours |
| Calendar features | Hour, weekday, weekend, day, month, quarter, cyclical encodings and trend |
| Weather information | Historical observations only, unless genuine forecasts are available |
| Operational information | Previous-day aggregates |
| Data partition | 80% development, 20% independent test |
| Validation | Four expanding-window folds |
| Validation horizon | 30 days per fold |
| Baselines | Naïve, Seasonal Naïve (24h), Seasonal Naïve (168h) |
| Performance metrics | MAE, RMSE and MASE |
| MASE seasonal period | 24 hours |

An important implementation requirement is that the evaluation code must follow the operational protocol exactly.

For each forecast origin:

1. Select training observations available up to 23:00 on the previous day.
2. Construct all predictors using only that historical information and known future calendar variables.
3. Generate the 24 predictions for the following day.
4. Compare those predictions against the corresponding 24 actual hourly observations.
5. Repeat for every eligible validation day.
6. Aggregate the results by forecasting horizon, validation period and geographical target.

In particular, seasonal naïve predictions must be generated using the corresponding historical observations relative to the forecast origin. Actual observations from earlier hours of the forecast day must never be used to predict later hours of that same day.

Similarly, statistical models must be evaluated using daily 24-hour forecasts rather than treating an entire 30-day validation block as a single forecasting operation.

The final independent test period must remain untouched until model development and selection have been completed.

## 6. Conclusion

Task 1 establishes the data preparation, exploratory analysis and methodological framework required for day-ahead EV charging-demand forecasting.

The original dataset contains 441,077 transaction records, which were transformed into four continuous hourly demand series representing the complete network and the three geographical districts.

The exploratory analysis identifies strong intraday seasonality, substantial long-term demand growth, relatively modest weekday/weekend differences and distinct geographical demand patterns. Tongxiang represents the largest contribution to network demand, while Nanhu exhibits the greatest relative variability.

The forecasting problem is defined as a 24-hour direct multi-step prediction task, with forecasts issued at 00:00 and all predictors restricted to information available through 23:00 on the previous day.

Calendar, historical demand, weather and operational features are organised into separate predictor groups. Particular attention is given to excluding future observed weather and future charging activity to prevent information leakage.

The evaluation methodology establishes an 80% development period, an independent 20% final test period, four expanding-window validation folds and three baseline forecasting methods. Performance is assessed using MAE, RMSE and MASE, with a 24-hour seasonal period adopted for MASE.

Preliminary experiments indicate that seasonal naïve methods provide strong reference performance and currently outperform the tested SARIMA configuration.

However, the existing experimental implementation must be aligned with the strict daily issuance schedule before its performance results can be treated as valid day-ahead comparisons.

**The methodological protocol is therefore established, with the operational forecasting and evaluation constraints explicitly defined. These requirements provide the foundation for subsequent model development, optimisation and selection.**