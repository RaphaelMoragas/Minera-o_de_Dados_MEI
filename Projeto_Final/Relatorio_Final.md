# Final report — Project 1: Classification of Abnormal Electric-Vehicle Charging Sessions

## Participants

- 1220665 - Nuno Marinho
- 1221083 - Rodrigo Cardoso
- 1260464 - Raphael Moragas

All the numbers cited come from Projeto_MINDD_Final.ipynb. In the notebook, when the numbering of a section does not match that of the report, the title indicates in parentheses the corresponding section of the report.

---

# Task 1 — Understanding, auditing and preparing the data

## 1.1 Loading and overview

The dataset has 441077 rows and 47 columns. It is possible to observe that 4 date columns (Start time, End Time, Payment Time and Order creation time) were read as text (string) and need to be converted to some form of date before we can do any kind of calculation with them. It is also possible to observe that only 4 columns have missing values, all from the group of historical attributes (post_hours_since_previous_completed_session, post_hours_since_previous_completed_abnormal, user_days_since_previous_completed_session, user_days_since_previous_completed_abnormal).

---

## 1.2 Data quality

### 1.2.1 Missing values

So we know how many values are missing but not why those values are missing. For example: If a user never had a failure they will have NaN in their state and there is no value to invent in that case. The NaN by itself is already the final information.

#### post_hours_since_previous_completed_session × post_no_previous_completed_session

The rows indicate whether the value is missing (True) or not (False); the columns indicate the flag (1 = the station never had a completed session before).

There are 92 unique station ids, that is, 92 different stations enter this evaluation.

- 92 sessions have NaN and the station had no history (flag 1). This is expected because these would be the first sessions of a station that has just started storing the data.
- 440985 sessions have a value and the station already had history (flag 0). This is expected because the stations already had a first session before.
- The other two cells are 0: there is no NaN in a station with history (False 1), nor any filled value in a station without history (True 0).

The 92 cases correspond exactly to the 92 stations of the dataset, that is, in the first session of each station, which does not have a previous session to compare itself with. The NaN is, therefore, structural and not an error.

#### post_hours_since_previous_completed_abnormal × post_no_previous_completed_abnormal

- 440710 sessions have a value and the station had already had at least one failure.
- 367 sessions have NaN and the station had not yet recorded any failure.
- Off the diagonal, 0 cases.

The 367 sessions are the ones that occurred at each station before its first failure. Without a previous failure, there is no "time since the last failure". This NaN is also structural.

#### user_days_since_previous_completed_session × user_no_previous_completed_session

- 384430 sessions have a value and the user already had at least one completed session.
- 56647 sessions have NaN and the user had no completed session before.
- Off the diagonal, 0 cases.

Of these 56647, 56115 are the first session of each user (the dataset has exactly 56115 distinct users). The remaining 532 are sessions started while the first session of the same user was still in progress: as the assignment statement explains, sessions that are still active do not count as "completed", so there is still no previous completed session.
Again the results end up being a structural NaN and we do not need to modify anything in these rows.

#### user_days_since_previous_completed_abnormal × user_no_previous_completed_abnormal

- 304798 sessions have a value and the user had already had at least one failure.
- 136279 sessions (about 31%) have NaN and the user had never had a failure.
- Off the diagonal, 0 cases.

It is the column with the most missing values, which makes sense: many users have few sessions and never went through a failure. And, for the last time, the NaN that we find here is still structural and, therefore, we do not need to modify or alter the data.

#### Conclusion and decision for the 4 columns

In all cases, the missing values coincide exactly with the _no_previous_ flags. They end up being just structural absences, not errors in the recording or erroneous addition of information. Therefore, they will not be modified with the mean or median of values in the same scope, because that would invent a non-existent history and would erase the difference between "failed a long time ago" and "never failed".

### 1.2.2 Duplicate data

Checking with the command, there are no duplicate rows in the dataset, so no action is needed for this type of problem.

### 1.2.3 Hidden spaces in the text and date conversion

We then did the conversion of the date values from string to date in order to be able to perform calculations with the values of the table.

An important point is that only items with type object or string can have a hidden trailing tab in the middle of the text; since the type of the dates has already been modified and they automatically lost the trailing tab, we will search in the other items that have the types in which this problem can appear.

After modifying the problematic items we see that no item with type object or string has any more writing errors that could split results, bringing inconsistency to the data.

### 1.2.4 Consistency and extreme values

#### Payment before the end of the session

In 87202 sessions (~20%) the payment appears recorded before the end of the session. Looking at the difference between the two times, it is small (median of 24 seconds), so it seems to be a recording difference between systems and not a serious error, but we may be wrong.

Since Payment time is a column that only exists after the session and will not be used in the model, we do not make any correction. It is just recorded as an inconsistency of the data.

#### Monetary values and sessions with 0 kWh

The monetary values seem to be consistent: in no session is the total different from the sum of electricity + service, and in none is the amount paid greater than the total as far as we can verify.

There are 54621 sessions (approximately 12%) with 0 kWh charged. This is not an error, they are sessions that ended without charging energy; further ahead we will check whether they come from an error (whether they are abnormal sessions). Therefore these rows will be kept for now.

#### Weather

We checked whether the weather values make sense for Jiaxing (eastern China, subtropical climate):

- Temperature between -3,2 °C and 33,6 °C: plausible for the winter and summer of the region.
- Humidity between 32% and 99%: within the possible range (0–100%).
- Precipitation between 0 and 152,1 mm: most sessions have no rain (the median is 0), and the highest values may be days of heavy rain. Nothing that seems to be abnormal for the region and the type of climate.

No value seems impossible, so we neither removed nor altered anything. The highest precipitation values are extreme, but believable.

#### Negative values

The only numeric column with negative values is temperature (with 2052 sessions), and all of these cases happen in December 2020, that is, in winter. Temperatures below zero are normal at that time of year, so it does not seem to be an error.

No other column (energy, costs, history counts) has negative values in its minimums, which was expected because it would not make sense.

#### Conclusion of 1.2.4

We did not find inconsistencies that require correcting or removing rows. The small time differences affect only columns that will not enter the model, and the extreme weather values (minimums and maximums) are plausible for the region from which the data are being taken.

### 1.2.5 Outliers in the predictor variables

#### 1.2.5.1 Station history outliers = signal, not error


The values that we see as outliers end up coming from stations with a high proportion of recent abnormal sessions. In post_previous_abnormal_rate_1D we end up having 6,4% of outliers that are from sessions in which the station had more than 46% abnormal sessions on the previous day.

As for post_previous_abnormal_rate_7D we can observe that 6,7% of the sessions occurred at a station with more than 47% abnormal sessions in a period of 7 days and with post_previous_abnormal_rate_30D we see that 5,7% occurred at a station with more than 49% abnormal sessions in 30 days.

Keeping in mind that the three values are rates, so it is possible to compare the three windows with each other. If we want to see the values in absolute terms it would just be a matter of using the measurements of post_previous_abnormal_1D, post_previous_abnormal_7D and post_previous_abnormal_30D. By using these items we would have the absolute value, X% of abnormal sessions at stations with more than Y abnormal sessions.

#### 1.2.5.2 Time since the station's last session: very long tail

When we observe the variable post_hours_since_previous_completed_session we can observe an extreme scale, being only possible to observe it using the logarithmic scale in the plot. The variable has a median of 0,48h or 29 min of time between completed sessions, a maximum of 5137h or practically 7 months stopped and an asymmetry or skew of practically 110, the largest in the whole dataset.

In other words, most of our values are small and the few extreme values may indicate that the station in question may have been stopped due to maintenance or some impossibility that prevented drivers from charging at that location. In general the values found seem believable. Again, it is not an error, just an extreme scale.


#### 1.2.5.3 User counters: dominated by fleet accounts

Our user_previous_sessions_730D has a median of 21, a p99 of 10172 and a maximum of 14566. This would indicate that, like the previous distribution, the distribution is very asymmetric, where the right end, the largest numbers of charges, depends on one or 2 accounts.

Users 39 and 23 have, in absolute values, 14584 and 6748 charges. An average of 6050 charging sessions, where an average user has an average of approximately 64 charges.

In user_previous_sessions_30D one user reaches 2605 charges in 30 days, which would be something around 87 charges per day, it being impossible for one person to charge the car almost 3 times per day.

It is in user_active_sessions_at_start that we manage to understand what is going on. 95% of the sessions have value 0, that is, in the overwhelming majority of times a user only charges 1 car at a time. As for the maximum number of charges at a time, it is 14, indicating that it probably concerns cars.

This idea is confirmed when we see that the userIDs with session numbers greater than 2 are exactly 39 and 23 from before. This probably indicates that they are business accounts or something of the kind that own a fleet of cars.

Therefore, we keep these data and as a way of visualizing them we use the logarithmic scale.


#### 1.2.5.4 Weather: no problematic outliers

Temperature and humidity have an asymmetry of approximately −0,4 and practically no outliers in the sample data.

As for Precipitation we have a problem: the IQR rule marks 19% as outliers. However the median is 0 (58% of the sessions without rain). Researching a bit more one sees that the IRQ as a measure does not work for variables whose data have many values equal to 0


#### Final decision

After analyzing the outliers of the predictor variables, we decided not to remove any row from the dataset. As we saw in the previous points, all the extreme values end up having a real and plausible explanation. It may be a station with many abnormal sessions, a station that was stopped for a long time or an account that seems to belong to a company (one must pay attention to this last data point at modeling time, since they may hinder the learning of the model). Removing them would be throwing true information away.

We also decided not to cap the station failure variables. It is precisely in the high values of these variables that we can observe the signal we want to predict, as we will see in section 1.4. As for the counts and times with asymmetry above 3, the idea is to apply a logarithmic transformation (log1p) in case the model is linear or distance-based, such as logistic regression, KNN or SVM. In tree models this is not necessary, since they are not affected by the scale of the variables.

Keeping in mind that, if at some point we are going to cap the data, the limit (for example the p99) must be calculated only on the training set, so as not to leak information from the test to the model. The assignment statement explicitly asks for this fact to be documented, so it is recorded here.

Finally, the only limitation that we can observe, as said before, is that the fleet accounts may end up dominating the learning of the model, since they have values very different from all the other users. That is, in the modeling phase it is worth evaluating the model with and without these accounts to assess whether the result changes considerably.

---

## 1.3 Definition of the target variable

### 1.3.1 Availability of the variables at the start of the session

Since the prediction must be made at the start of the session, each variable was classified according to its availability at that moment. The variables recorded during or after the session were excluded from the predictors to avoid what was asked in the assignment statement ("The prediction must represent a decision made when a charging session starts. Consequently, only information that would reasonably be available at that time may be used as a predictor"), being used only in the exploratory analysis.

### 1.3.2 electricity_price × tariff_period

The variable electricity_price has only 3 values (0,38 / 0,90 / 1,21). In practice it is categorical and repeats the information of tariff_period. Therefore they are redundant information by their very definition. That is, if we use one we do not need the other.


### 1.3.3 Column classification table

For each of the 48 columns (the 47 original ones plus the is_Abnormal that we created) we asked the same question: "is this value already known when the session starts?". The answer is in the 'availability' column of the table and, from it, the 'use' column indicates what we are going to do with each variable. In the end we are left with five groups:

| Availability | Use | Columns | Why |
|---|---|---|---|
| Start | Feature | location, start time, weather, tariff_period and station and user history | they are already known when the session starts |
| After the end | EDA only | energy, costs, end time, payment and End cause | they only exist after the session ends |
| Debatable | To be evaluated | UserID and Charging Post ID | known at the start, but the model may memorize IDs |
| Redundant | Do not use | Order creation time and electricity_price | they repeat the information of Start Time and tariff_period |
| Target | Target | is_Abnormal | it is what we want to predict |

The variables of the "After the end" group are the most important to leave out. If they were used, the model would be "seeing the future" and would have an artificially high performance. That is why they stay in the dataset only for the exploratory analysis (EDA, from the English Exploratory Data Analysis).

As for the user and station IDs, they are available at the start, but they may lead the model to memorize specific users and stations. They also do not help at all when a new user or station appears, since there is no history the first time. As the assignment statement asks, the decision will be made by comparing models with and without these columns.

Order creation time was left out because in 56% of the sessions it is recorded after the Start Time (median of 55 s) and it does not bring any new information. As for electricity_price, it has only 3 values and each tariff_period always has the same price, so knowing the period we already know the price. The opposite does not happen, because the same price appears in several periods, so we keep tariff_period, which has more information.

Finally, the "?" category of the code serves as a check: if any column fell there, it would mean that we forgot to classify it. In our case no column was left unclassified.

### 1.3.4 Lists reused in the next steps

From the table, we save three lists that will be reused in the next steps:

- features_inicio: candidate columns for predictors;
- colunas_depois: columns that never enter the model, only the exploratory analysis;
- colunas_ids: identifiers to be tested with and without their presence, in the modeling phase.

This way, the separation between what can and what cannot enter the model is defined in a single place and does not need to be repeated by hand later.

The next cell shows, with a concrete example, why the post-session columns have to stay out.

### 1.3.5 Example: 0 kWh "gives away" the answer

Example of a problem in the data: of the 54621 sessions with 0 kWh charged, 83,4% are abnormal. In the sessions that contain charged energy (> 0 kWh), only 9,4% are abnormal.

It makes sense if we think that the session that fails right at the start does not get to charge energy. But the amount of energy is only known after a session ends. If this column entered the model, it would learn the rule "0 kWh → abnormal" and would have great results in the test, but it would be useless in practice, because at the moment of the prediction that value does not exist yet, we would just be throwing garbage into our data.

This confirms, with evidence, and justifies the decision to exclude from the predictors all the columns classified as "After the end".

As for the sessions with 0 kWh, which in section 1.2.4 had been kept for now, they remain in the dataset. They are not errors, but rather real sessions that failed before charging energy, and about 45,5 thousand of them are abnormal, which represents more than half of all the anomalies in the dataset (about 82 thousand). If we removed these rows, we would be erasing precisely most of the cases that the model needs to learn to predict. The problem was only in the energy column, which gives away the answer, and since it has already been excluded from the predictors, the rows can stay without bringing any information leakage.

---

## 1.4 Relationship of the variables with the target

### 1.4.1 Station history (heatmap 1)

- Strong with the target: the station's anomaly rates: rate_1D 0,45, rate_7D 0,43, abnormal_1D 0,37, rate_30D 0,37.
- Weak or null: number of sessions (7D 0,07, 30D 0,01) and hours since the last session (0,01).
- Negative: hours_since_previous_completed_abnormal −0,12. The more time since the last failure, the lower the risk.
- Redundancies: sessions_7D × 30D 0,90; abnormal_1D × rate_1D 0,85; rate_7D × rate_30D 0,82.

From heatmap 1 we can take the information that a station that failed recently tends to fail again. The station's anomaly rate on the last day (rate_1d) is the variable that most correlates with the target (0,45), followed by the station's anomaly rate in the last week (rate_7D) (0,43).

With it being visible that this correlation drops as the time window increases. Thus the most recent history is what best informs and predicts a next failure at the station. The inverse also being informative, where the more time passes since the last failure, the lower the risk (hours_since_previous_completed_abnormal with a rate of −0,12).

### 1.4.2 User history (heatmap 2)

- Strong with the target: only the rates: rate_30D 0,30, rate_180D and rate_730D 0,28.
- Null: counts of sessions and of failures (≈ −0,02 to 0,01) and is_new_user 0,02.
- Extreme redundancy:
  - abnormal_730D × sessions_730D 1,00;
  - abnormal_180D × sessions_180D 0,99;
  - is_new_user × user_no_previous_completed_session 0,99;
  - rate_180D × rate_730D 0,97.

From heatmap 2 we see that, just as with the station, what relates to the failure is the user's anomaly rate, and not the number of sessions or of failures. The rate of the last 30 days (rate_30D) is the strongest (0,30), followed by the 180 and 730 day rates (0,28). As for the counts, they stay practically at zero, that is, a user who charges a lot does not necessarily fail more. Even so, the user history informs less than the station's (0,30 against 0,45).

It is also visible that many columns are practically duplicated, such as abnormal_730D and sessions_730D (1,00) and is_new_user and user_no_previous_completed_session (0,99), so it does not make sense to keep all of them in the model. This correlation of 1,00 between failures and sessions is strange, and probably happens because some accounts with thousands of sessions dominate the Pearson calculation (In the next step we will check better what may be happening).

### 1.4.3 Session context — weather, price and the user's active sessions at the moment (heatmap 3)

- Everything ≈ 0 (temperature 0,03; the others between −0,01 and 0,01).

From heatmap 3 we see that all the context variables were close to zero, temperature being the largest with only 0,03. Thus, alone, the weather, the price and the user's number of active sessions do not explain the anomaly. But since Pearson only measures linear relationship, these variables may still help the model when , who knows, combined with other variables.

### 1.4.4 Pearson correlation of the location and of the tariff period with is_Abnormal

#### 1.4.4a Location (heatmap 4a)

- Highest risk: Expressway C (0,11), Shopping Mall (0,07), Expressway A (0,07), Wholesale Market (0,06).
- Lowest risk: Park B (−0,10), Bus Station (−0,09), Technology Park (−0,06).
- Real rate: goes from 7,9% (Industrial Park) to 32,4% (Expressway A).

From heatmap 4a we can see that the location of the station matters. The service areas of the expressways (Expressway C with 0,11 and Expressway A with 0,07) and that of the Shopping Mall (0,07) are the ones that most relate to failures, while Park B (−0,10) and Bus Station (−0,09) have less risk. The correlation values seem low because each location is only a small part of the sessions, but looking at the real rate it becomes clearer: it goes from 7,9% in the Industrial Park up to 32,4% in Expressway A, that is, the motorway areas have about 4× more failures than the parks. It is not possible to know why this occurs, but it is indeed interesting to have a correlation like this present.

#### 1.4.4b Tariff period (heatmap 4b)

- Everything close to 0 (between −0,016 and 0,02). The rate only varies between 16,7% and 20,5%.

From heatmap 4b we see that the tariff period hardly influences the failure, since all the correlations stay close to zero (between −0,016 and 0,02) and the anomaly rate only varies between 16,7% and 20,5%. It is a weak variable, but it can be kept in the model, since it may help together with other variables.

### 1.4.5 Elimination of redundant variables

Following the criteria discussed with the professor, we now eliminate the variables highly correlated with each other. When two variables have a Pearson correlation greater than 0,9 between them, we keep the one that has the higher correlation with the target (is_Abnormal) and eliminate the other. The justification is that, if two variables bring practically the same information, keeping both adds nothing to the model and only makes the next analyses heavier, both quantitatively and visually for us users.

The correlation with the target is compared in absolute value, because what matters is the strength of the relationship and not the sign (some counts have negative correlation with the target).

The heatmaps round to 2 decimal places, which is not enough to decide the borderline cases. That is why the values below were calculated with 4 places, from the same matrices as the heatmaps.

#### Heatmap 1

No pair goes above 0,9. The highest is sessions_7D × sessions_30D, which appears as 0,90 in the heatmap and is worth 0,8987. Even though it does not reach 0,9, we decided to eliminate one of them, because the value is so close to the limit that, in practice, both bring the same information. Applying the same criterion, sessions_7D stays, which has a higher correlation with the target (0,0678 against 0,0143), and post_previous_sessions_30D leaves.

The other high pairs are clearly below the limit (abnormal_1D × rate_1D 0,85; rate_7D × rate_30D 0,82), so the remaining station variables are kept.

#### Heatmap 2

Here there are five pairs above 0,9:

| Pair | Correlation between them | Correlation of each one with the target | Stays | Leaves |
|---|---|---|---|---|
| sessions_730D × abnormal_730D | 0,9974 | −0,0206 / −0,0124 | sessions_730D | abnormal_730D |
| is_new_user × user_no_previous_completed_session | 0,9946 | 0,0195 / 0,0206 | user_no_previous_completed_session | is_new_user |
| sessions_180D × abnormal_180D | 0,9880 | −0,0241 / −0,0089 | sessions_180D | abnormal_180D |
| rate_180D × rate_730D | 0,9710 | 0,2821 / 0,2796 | rate_180D | rate_730D |
| sessions_30D × abnormal_30D | 0,9455 | −0,0201 / 0,0148 | sessions_30D | abnormal_30D |

The pair sessions_30D × abnormal_30D (0,95 in the heatmap) was missing from our first list. As for rate_180D and rate_730D, they seemed tied (0,28 with the target), but with 4 places the 180-day one is slightly ahead (0,2821 against 0,2796).

In four of the five pairs both variables have almost null correlation with the target (below 0,03 in absolute value), so the choice of which one stays changes little. Even so we followed the criterion to the letter.

From the user history 5 variables leave: user_previous_abnormal_730D, user_previous_abnormal_180D, user_previous_abnormal_30D, user_previous_abnormal_rate_730D and is_new_user.

#### Heatmap 3

The pair with the highest correlation has only 0,35 and, therefore, all the context variables are kept.

#### Summary

In total 6 variables leave: 1 from the station history (post_previous_sessions_30D) and 5 from the user history (user_previous_abnormal_730D, user_previous_abnormal_180D, user_previous_abnormal_30D, user_previous_abnormal_rate_730D and is_new_user).


### 1.4.6 Outliers against is_Abnormal

In section 1.2.5 we decided to keep the outliers of the predictor variables, and here we went to check whether they really carry information about the target. For this, in each variable we separated the sessions that are above the IQR limit (Q3 + 1,5 × IQR) and compared their anomaly rate with that of the rest of the sessions.

In the station history the difference is very large. In the sessions in which the station had an out-of-the-ordinary anomaly rate on the previous day (post_previous_abnormal_rate_1D), 75% are abnormal, against only 14,7% in the rest. The same happens with the station's number of failures in the last week (post_previous_abnormal_7D), with 55,3% against 14,8%, and in the last 30 days (post_previous_abnormal_30D), with 45,7% against 15,7%. Thus, the station outliers are not errors, but rather the cases that seem to most inform about a next failure, that is, if we removed or capped these values we would be erasing precisely the most predictive information we have. When a station starts to fail, obviously the sessions on that day will suffer from more charging anomalies.

In the user history the pattern repeats in the anomaly rates. The outliers of the 30, 180 and 730 day rates have between 45,6% and 46,9% of abnormal sessions, against about 16% in the rest. That is, the user's failure rate also has signal, although weaker than the station's (about 46% against 75%).

As for the user's number of sessions in 730 days (user_previous_sessions_730D) the opposite happens, where the outliers have even a bit less anomaly (15,4% against 19%). These outliers are the accounts with thousands of sessions that we saw in section 1.2.5.3, which probably belong to fleets, and their anomaly rate is similar to that of the rest, so they do not carry any special signal. Even so, they are real sessions and that is why they also stay in the dataset.

With this we confirm the decision of section 1.2.5 not to remove or cap the outliers, since in the failure variables they are exactly the signal that we want the model to learn.


---

## 1.5 Temporal stability

In this section the main idea was to check whether the data remain stable over time or whether the behavior of the sessions changes from one period to another, since, if the data change a lot, a model trained in one period may not work as well in another. The data cover exactly 25 months and go from 31/12/2019 to 31/12/2021, with December 2019 having only 17 sessions, since the records start at 21h on the last day of the month, so we do not consider it as a month in the monthly analyses.

The first thing we can see is that the volume of sessions grows a lot, going from about 6 thousand to about 27 thousand sessions per month, and 2021 ends up having more data than 2020 (255 thousand against 186 thousand sessions). The exception is February, which drops in both years (the 2 big drops that happen in the first chart "Number of sessions per month"), reaching only 1 391 sessions in Feb/2020, which may possibly be linked to the Chinese New Year and, in 2020, to the start of COVID, but the data do not allow us to confirm this. Along with the volume the user base also grows, going from about 2 thousand (2020-01) to about 7,5 thousand (2021-10) active users per month, keeping in mind that 43% of the users have a single session in the whole dataset. As for the number of stations, it stays stable at 91–92 during almost the whole period and only drops to 86–88 at the end of 2021, because 4 stations (17, 18, 19 and 20) stop having sessions on 28/09/2021 and others stop in December. It is not possible to know whether they were deactivated or went into maintenance, but this explains the drop.

The anomaly rate is also not constant. It starts at about 25%, drops to 16–17% at the end of 2020, reaches the minimum of 13,7% in Mar/2021 and goes back up to about 23% in Sep/2021. Curiously, looking by year the rate hardly changes (18,7% in 2020 and 18,5% in 2021), that is, the variation only appears when we look month by month. The history variables follow this change in the way we expected: the monthly mean of the station's 7-day failure rate goes from about 0,23 to 0,15 and back to 0,22, and the user's 30-day one does the same, but in a smoother way (between 0,17 and 0,25), which matches what we saw in section 1.4, where the user history informs less than the station's.

As for the percentage of new users, it is very high at the start (94% in Dec/2019 and 31% in Jan/2020), which makes sense, since at the beginning of the records almost all users appear for the first time, and then it stabilizes at 10–15% per month. These new users have a bit more anomaly than the rest (20,6% against 18,3%). That is, the first months are a "cold start", where the history of the users and of the stations is still being built and the history variables have little information. The mean of the user's previous sessions in 730 days grows naturally, since the window keeps accumulating sessions, but it has a strange peak in Jun/2020 (626) followed by a drop in Aug/2020 (132). Since the mean is calculated over the sessions of each month, this probably happens in the months in which the large accounts (users 39 and 23, which add up to about 21 thousand sessions, that is ≈5% of the total) charge a lot and pull the mean up. It is these same accounts that explain the correlation of 1,00 between user_previous_sessions_730D and user_previous_abnormal_730D in heatmap 2. Finally, the temperature follows the normal cycle of the seasons, with about 6 °C in winter and 30 °C in summer, and the anomaly rises a bit in the summers (Aug/2020 and Sep/2021), but as we saw in section 1.4.3 the linear correlation with the target is weak (0,03), so there is a seasonality, but alone it does not explain the anomaly.

In general, the volume of sessions, the monthly anomaly rate and the user base change over time, that is, the data do not seem to be stable, and this brings some consequences for the modeling. The split between training and test has to be temporal, training with the past and testing with the final months, since if we made a random split we would be mixing sessions from the future into the training and the result would be too optimistic. Since the anomaly rate of the test period may be different from that of the training, we will also not evaluate the model only by accuracy, but by metrics such as F1, PR-AUC and the recall of the abnormal class. We still have to pay attention to the cold start, possibly discarding Jan–Feb/2020 from the training or, at least, citing this as a limitation, and to the large accounts, which may dominate the learning of the model, so the idea is to check the error with and without them. Finally, since some stations stop at the end of 2021, the test period will have fewer stations than the training one, which is also a limitation to cite.


### 1.5.1 Decisions for Task 2

- **Cold start:** the sessions of Dec/2019 and Jan/2020 will be removed from the training. In these months the history of the users and of the stations is still being built (94% and 31% of new users, against 10–15% in the other months), so the history variables have little information. They are about 6 200 sessions (~1,4% of the total) out of the more than 400 thousand that we have in total.
- **February:** the drop in sessions in Feb/2020 will not be removed. The same drop happens in Feb/2021, so it seems to be an event that repeats every year, much more likely linked to the Chinese New Year than to COVID as previously thought. In addition, the anomaly rate of that month (26,3%) is similar to that of January (25,0%) and March (24,8%).
- The cut is made only in the training, in Task 2. The CSV exported in the notebook will keep all the sessions unmodified.

---

# Task 2 — Definition of the prediction problem and of the evaluation

## 2.1 Problem definition

As for the unit of observation, each row of the dataset is a charging session, that is, the model makes one prediction per session and we have 441 077 sessions in total. As for the classes, the positive class is the abnormal session (is_Abnormal = 1) and the negative is the normal session (is_Abnormal = 0). There are 81934 abnormal sessions (18,6%) against 359143 normal ones (81,4%).

About how the target comes out of End cause, as was chosen in Task 1, we decided to interpret the sessions that ended with "Charging ends normally" or "User stops charging" as being normal sessions and the ones that ended with any of the other 13 reasons, all failures (of the station, of the communication with the vehicle, of the BMS, etc), are the ones considered abnormal sessions.

The moment of the prediction is the start of the session (Start Time). Only what is already known at that moment can be used, that is, variables that we do not know at the moment of the start will not be considered at that point. The prediction serves for automatic monitoring and prioritization of the maintenance of the stations. Predicting a session as abnormal means that the session has a high risk of ending with a failure, so the system generates an alert for the operator to follow the session and the station moves up in the inspection queue. The alert does not block the session nor warn the user, it works only as an alert to the user of the car so that they stay attentive.

We consider "User stops charging" as normal because it was the user themselves who decided to stop, and not a technical failure. But this ends up being a limitation of the dataset itself and as the information there exists, since some users may have stopped precisely because something was wrong (a slow charge, for example), and the data do not allow us to separate these cases for a more precise prediction. Unfortunately this is a problem that there is no way to get around.

---

## 2.2 Variables available at the moment of the prediction

For each of the 47 columns of the original dataset we asked the same question as in Task 1: "is this value already known when the session is started?".

| Category | Columns | Why |
|---|---|---|
| 1. Predictors (29) | Location Information, District Name, Start Time, weather (3), tariff_period, station history (12), user history (9) and user_active_sessions_at_start | they are already known when the session starts |
| 2. Only for the target (1) | End cause | it is where is_Abnormal comes from |
| 3. Generated during or after the session (7) | Transaction power/kwh, Electricity cost, Service charge, Transaction Amount, Actual Payment, End Time and Payment time | they only exist after the session ends |
| 4. Other reason (10) | Order creation time and electricity_price | they repeat the information of Start Time and tariff_period |
| | post_previous_sessions_30D, user_previous_abnormal_730D, user_previous_abnormal_180D, user_previous_abnormal_30D, user_previous_abnormal_rate_730D and is_new_user | correlation above 0,9 with another variable |
| | UserID and Charging Post ID | the model may memorize IDs |

Start Time enters as a predictor, but not as a date, but rather to extract the hour and the day of the week. End cause is practically the answer itself and is only known at the end of the session, so it could never be a predictor.

---

## 2.3 Consequences of the errors

A false positive (FP) is an alert on a session that would be normal. The cost is low (a user keeps an eye on a session unnecessarily or a technician checks a station that was fine), but if it happens too much the operators start to ignore the alerts. As for a false negative (FN), it is an abnormal session without an alert. The user is left without charging and the station with a problem continues to be used without anyone knowing.

For the chosen use, the FN is the most important error. As we saw during Task 1, a station that failed recently tends to fail again (the station's failure rate on the last day is the variable most correlated with the target, 0,45), that is, letting a failure pass ends up also affecting the next users of that station besides the non-maintenance needed for it. Even so the FP should not be ignored: in every 1 000 sessions there are  about 186 sessions that are abnormal, and a model that gave an alert on all of them would catch the 186 but would generate about 814 false alerts. Therefore, the evaluation should give more weight to recall without letting precision drop too much.

We also considered the intervention on the user (warning them to switch station), but then the false positives would become the most expensive error, since each false alert could bother a real customer. That is why it remains only as a possible extension for alerts with very high probability.

---

## 2.4 Evaluation metrics

A model that always predicts normal would get about 81% of the sessions right, but would have a recall of 0%. It is the accuracy paradox, that is, with  approximately 18% of anomalies the accuracy measures more the imbalance of the period than the quality of the model, and that is why it will not be used.

The main metric would be AUC-PR. Because it is indicated when the positive class is the minority and when false positives and false negatives end up having different costs, which are our two cases in question. Since it does not depend on the threshold, it serves to compare models and choose hyperparameters before the threshold ends up being defined. The final limitation is that the value of a model without skill is equal to the anomaly rate, which changes from fold to fold, so each fold ends up having to be compared with its own baseline.

**Complementary metrics:**

| Metric | Why | Limitation |
|---|---|---|
| Recall | controls the FN, our most serious error | alerting everything gives 100% |
| Precision | controls the false alerts | alone it also ends up deceiving |
| F1 | combines precision and recall | gives the same weight to both errors |
| Balanced accuracy | corrects the class imbalance | gives the same importance to both classes |
| MCC | uses the 4 cells of the confusion matrix and is strong against the imbalance that we see in our dataset | ends up being less intuitive |
| ROC-AUC | measures the separation between the classes | too optimistic when the classes are imbalanced |

For the threshold, we keep the one with the highest precision among those that guarantee a minimum recall (for example, 70%), chosen with the 3 validations together and never with the test. As operational metrics we use the correct alerts, false alerts and missed anomalies per 1 000 sessions.

For the stability between folds we use the standard deviation of the AUC-PR and the worst fold. in the lectures we saw the Wilcoxon test to compare models across the folds, but with only 3 folds it can never give p < 0,05 (the smallest possible p ends up being 0,25), so across the folds the comparison will be made by the mean, by the deviation and by how many folds each model wins. In the final test, since there are about 88 thousand sessions, then yes it would be possible to use the McNemar test.

---

## 2.5 Temporal validation strategy

We do not use random k-fold, because it would mix sessions from the future into the training. First, with the data sorted by date, we separate the oldest 80% for training and the most recent 20% for test. Then, the folds are made only within the 80% of training, with rolling-origin, as indicated in the project statement, where the training always has the same size and slides forward together with the validation.

For this we created a function (dividir_dados) that makes the 80/20 cut and then uses TimeSeriesSplit to divide the training into folds. This way no date is chosen by hand and the same function can be used on any other dataset.

| Set | Training | Validation | Sessions in the validation | Anomalies |
|---|---|---|---|---|
| Fold 1 | 01/04/2020–13/01/2021 | 13/01–03/05/2021 | 52 929 | 8 358 |
| Fold 2 | 14/07/2020–03/05/2021 | 03/05–18/07/2021 | 52 929 | 10 416 |
| Fold 3 | 22/09/2020–18/07/2021 | 18/07–22/09/2021 | 52 929 | 11 134 |
| Final test | the 180 000 sessions before the test | 22/09–31/12/2021 | 88 216 | — |

The test has exactly 20% of the sessions (88 216 of 441 077) and starts on 22/09/2021. The validations occupy the last 45% of the training, divided into 3 folds with the same number of sessions. We chose 3 folds because that way each validation has more than 8 thousand anomalies. With more folds each validation would be smaller, and increasing the part of the training used in the validations would make the first training start earlier, precisely in the period in which the history is still being formed.

Each training has 180 000 sessions, which is more or less the size of the first training. With this the training of fold 1 starts in Apr/2020 and the months of Dec/2019 and Jan/2020, identified as "outliers" in Task 1 (94% and 31% of new users, respectively), are left out of all the folds.

As we saw in section 1.5 of Task 1, the anomaly rate changes a lot over time (being able to vary from 25% to 13,7% and back to 23%). With rolling the model learns only from the recent past and the trainings all have the same size. The 180 000 sessions correspond to about 9 to 10 months, that is, almost an entire cycle of seasons. The disadvantage is no longer using the oldest months.

Since the folds are made by the number of sessions, all the validations have the same size, but not the same duration (about 3,6, 2,5 and 2,2 months), because the volume of sessions grows over 2021. The validations do not overlap each other, so each session from 13/01/2021 to 22/09/2021 is evaluated one single time only.

Everything that the model learns from the data (scaler, one-hot, outlier limit, etc) is fitted only on the training of each fold and then only applied to the validation, which always keeps the original distribution. log1p does not learn anything from the data, so it can be applied to everything.

The test was separated right at the start and does not enter any decision. Models, parameters and threshold are chosen only with the 3 folds. In the end, the final model is trained on the 180 000 sessions before the test and evaluated a single time, without going back in case the result is not good, otherwise the test would end up becoming one more validation and the result would no longer be impartial.

---

## 2.6 Aggregation across folds

Each model will have one result per fold, and we need to decide how to combine these 3 values into a summary. The possible ways are the simple mean, the mean weighted by the size of the validation and the pooled, where we combine the 3 validations into a single one and calculate the metric one single time.

Since the 3 validations have the same number of sessions, the weighted mean ends up giving exactly the same value as the simple mean. As for the pooled, folds 2 and 3, which have more anomalies, end up weighing more than fold 1. In the simple mean each period counts the same.

Therefore, to compare the models we will use the simple mean of the AUC-PR, since we want the model to work well in all periods and not only in the largest one. Along with it we will always show the standard deviation, the worst fold and the value of each fold, because the mean alone can hide a lot of variation between the periods. We will also count in how many of the 3 folds the model beats the rule, since a model that wins on the mean but loses in some fold is not reliable. As for the pooled form, it will be used to choose the threshold and for the operational metrics per 1 000 sessions, since there it makes sense to combine all the validation sessions.

As for the evolution over time, the anomaly rate rises from 15,8% in fold 1 to 19,7% and 21,0% in folds 2 and 3. Since the AUC-PR of a no-skill model follows this rate, a model may seem to improve over time just because the period has more anomalies, and that is why the evolution over time always has to be read relative to the baseline of each fold.

---

## 2.7 Baseline

To know whether a model is good we need a yardstick for comparison, and that is why we defined two baselines, both measured with the AUC-PR, which is our main metric.

| Fold | Anomaly rate | No-skill (expected AUC-PR) |
|---|---|---|
| 1 | 15,8% | 0,158 |
| 2 | 19,7% | 0,197 |
| 3 | 21,0% | 0,210 |

The first is the no-skill, that is, the same model that always predicts normal that we used in the accuracy test. Since it does not rank the sessions in any way, its AUC-PR ends up being exactly the anomaly rate of each fold, as we had already commented in the metrics. Any model that does not beat these values has not learned anything.

The second is a simple rule, where we directly use the station's failure rate on the previous day (post_previous_abnormal_rate_1D), which was the variable most correlated with the target (is_Abnormal) in Task 1 (0,45), as a risk score. That is, the more the station failed yesterday, the higher the risk of today's session being abnormal. It does not need training or a threshold, so it is a simple yardstick to calculate in each fold.

Therefore, the yardstick that really matters is the rule and not the no-skill. If the final model does not clearly beat the rule, it means that the other variables (weather, location, user history, etc.) are not adding anything to what the station history already says on its own. The comparison always has to be made fold by fold, since both the anomaly rate and the result of the rule change from one period to another.

---

## 2.8 Evaluation protocol

In order to close Task 2, we gather here everything that was decided. This protocol is defined before any model is trained, that is, we will not change these rules after seeing the results that we will have in Task 3.

| Item | Decision |
|---|---|
| Problem | predict, at the start of each session (Start Time), whether it will be abnormal (is_Abnormal = 1) or normal (0) |
| Use | automatic monitoring and prioritization of the maintenance of the stations |
| Admitted predictors | the 28 columns of the treated CSV (location, weather, tariff_period, station and user history, user_active_sessions_at_start) and Start Time, from which we will extract the hour and the day of the week |
| Excluded | End cause (it only serves for the target), the 7 columns generated during or after the session, Order creation time and electricity_price (redundant) and the 6 variables with correlation above 0,9 |
| To evaluate in the modeling | UserID and Charging Post ID, comparing the model with and without them |
| Temporal split | 80% training and 20% test, in chronological order; rolling-origin only within the training, with trainings of 180 000 sessions and 3 folds with validations of 52 929 sessions (13/01 to 22/09/2021); final test from 22/09 to 31/12/2021, used one single time |
| Preprocessing | scaler, one-hot, outlier limit, etc., fitted only on the training of each fold |
| Most serious error | the FN (undetected anomaly), without ignoring the FP (too many false alerts) |
| Main metric | AUC-PR |
| Complementary metrics | recall, precision, F1, balanced accuracy, MCC and ROC-AUC (accuracy is not used) |
| Threshold | the one with the highest precision among those that guarantee a minimum recall, chosen with the 3 validations together |
| Operational metrics | correct alerts, false alerts and missed anomalies per 1 000 sessions |
| Baselines | no-skill (always normal, AUC-PR = anomaly rate of the fold) and rule with post_previous_abnormal_rate_1D |
| Aggregation | simple mean of the AUC-PR to compare models, always with the standard deviation, the worst fold and in how many folds the model beats the rule; pooled for the threshold and the operational metrics |
| Statistical comparison | in the folds by the mean, deviation and folds won (Wilcoxon does not work with 3 folds); in the final test with McNemar |

Keeping in mind that a model will only be considered good if it clearly beats the rule in all folds, and not only on the mean.

---

# Task 3 — Individual models and ensemble

## 3.1 Common configuration

Table 3.1 — Preprocessing by model type (fitted only on the training of each fold)

| | Logistic regression | Mixed Naive Bayes | Trees (DT, RF, HistGB) |
|---|---|---|---|
| Categorical (location, district, tariff) | One-hot | Frequencies with Laplace (`CategoricalNB`) | One-hot |
| `hora`, `dia_da_semana` | One-hot | Categorical | Numeric |
| Very asymmetric counts and times (skew > 3) | `log1p` + `StandardScaler` | `log1p` + normal per class (`GaussianNB`) | No transformation |
| Remaining numeric | `StandardScaler` | Normal per class (`GaussianNB`) | No transformation |
| NaN | Training maximum | Training maximum | Kept (the implementation supports them) |


> Lectures: Lecture 4, slides 30 and 64 · golden rule of Lecture 3 · Guide 5.3 and 8.

In this part of Task 3, we are only reusing the protocol defined in Task 2 (3 temporal folds, AUC-PR, rule and no-skill, simple mean + deviation + worst fold + folds > rule).

In Logistic Regression (LR), each feature contributes with weight × value, and this ends up being a problem when we have a variable like the hour, which goes from 0 to 23: the model can only learn a monotonic effect (the risk only rises or only falls throughout the day). In addition, the minimum and maximum values of the hour are side by side, which does not happen in the numbers (0 is far from 23 in the numbering, but, in relation to the hour, they are side by side). The solution ends up being to use one-hot, which gives an independent weight to each hour. This same one-hot ends up being unnecessary and even counterproductive in the trees, since they make cuts of the type Hour >= X and can separate hour intervals from the numeric variable. With one-hot, each cut would only separate one hour from the rest.

Another point that is good to explain is that log1p does not cause leakage, unlike StandardScaler. This is due to the way the formulas work: log1p transforms each value alone (log(1 + x)), without using information from the other sessions, so it can be applied at any moment without generating leakage. As for StandardScaler, by subtracting the mean and dividing by the standard deviation, it uses information that needs to be extracted from the data. That is, there is a right moment to apply it: it is fitted only on the training of each fold (inside the Pipeline) and then applied to the validation. If it were fitted with all the data, the mean and the deviation would already include information from the validation.

And, finally, we come to the NaN, which receive the maximum value of the training of each fold. These NaN appear in the columns of the type "hours since the last event" when the event never happened. If they received 0, they would enter with the opposite meaning: the event would have just happened, when in fact it never happened. This way, receiving the maximum value of the training gives us a usable and coherent variable (the larger the value, the more time without the event), and the flag *_no_previous_* continues to separate the cases in which the event never happened.

---

## 3.2 Contribution of the variable groups (ablation)


### 3.2.1 Groups and models

Table 3.2.1 — Variable groups

| Group | Added variables | Total no. |
|---|---|---|
| G1 — context and weather | `Location Information`, `District Name`, `hora`, `dia_da_semana`, temperature, humidity, precipitation | 7 |
| G2 — G1 + tariff | `tariff_period` | 8 |
| G3 — G2 + station history | 12 `post_*` variables | 20 |
| G4 — G3 + user history | 9 `user_*` variables and `user_active_sessions_at_start` | 30 |

Why did we end up choosing, in principle, 2 models for the ablation and not just one? Besides being models with different principles. The main idea was to check whether or not our results depend on the models we use. The general idea is that our conclusion does not depend on the type of model. If a variable or a group of variables improves both models, this indicates that the gain comes from the information and not from the model.

The variable user_active_sessions_at_start ended up staying in the user group, because it counts how many open sessions the user has at the moment, thus describing the user, and not the station. If it stayed in group 3, that group would already have user information, and the comparison between group 3 and group 4 would no longer measure how much the user history is worth.

As for the variable UserID, it was left out for privacy reasons, since basing the failure prediction on the user could end up marking specific users as responsible for the failures. As for the Charging Post ID, it was left out, because the model may memorize specific stations, besides not helping in the prediction for new stations (section 3.2.8).

### 3.2.2 Results

Table 3.2.2a — AUC-PR in the validation of each fold, by model and variable group

| Model | Group | Fold 1 | Fold 2 | Fold 3 | Mean | Deviation | Worst fold | Folds > rule |
|---|---|---|---|---|---|---|---|---|
| No-skill | — | 0,158 | 0,197 | 0,210 | 0,188 | 0,027 | 0,158 | — |
| Rule | — | 0,450 | 0,602 | 0,590 | 0,548 | 0,084 | 0,450 | — |
| LR | G1 | 0,249 | 0,310 | 0,334 | 0,298 | 0,044 | 0,249 | 0 |
| LR | G2 | 0,249 | 0,310 | 0,335 | 0,298 | 0,044 | 0,249 | 0 |
| LR | G3 | 0,506 | 0,636 | 0,617 | 0,586 | 0,070 | 0,506 | 3 |
| LR | G4 | 0,532 | 0,652 | 0,627 | 0,604 | 0,063 | 0,532 | 3 |
| HistGB | G1 | 0,232 | 0,300 | 0,342 | 0,292 | 0,055 | 0,232 | 0 |
| HistGB | G2 | 0,233 | 0,302 | 0,347 | 0,294 | 0,057 | 0,233 | 0 |
| HistGB | G3 | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| HistGB | G4 | 0,564 | 0,679 | 0,653 | 0,632 | 0,061 | 0,564 | 3 |

Table 3.2.2b — AUC-PR gain when adding each group (difference to the previous group)

| Model | Added variables | Fold 1 | Fold 2 | Fold 3 | Mean |
|---|---|---|---|---|---|
| LR | Tariff (G2 − G1) | +0,0000 | +0,0000 | +0,0001 | +0,0001 |
| LR | Station history (G3 − G2) | +0,2572 | +0,3263 | +0,2822 | +0,2885 |
| LR | User history (G4 − G3) | +0,0258 | +0,0158 | +0,0104 | +0,0174 |
| HistGB | Tariff (G2 − G1) | +0,0005 | +0,0016 | +0,0045 | +0,0022 |
| HistGB | Station history (G3 − G2) | +0,3048 | +0,3587 | +0,2950 | +0,3195 |
| HistGB | User history (G4 − G3) | +0,0258 | +0,0187 | +0,0114 | +0,0186 |

### 3.2.3 Does the tariff bring information beyond the hour?

In the end no, it does not. This is because each hour always ends up falling into a single tariff period. Therefore, the gain from adding the tariff does not add much difference between the folds, because knowing the hour one already knows the tariff. The gain in LR is practically none (explained by the fact already addressed previously where the hour here enters as one-hot). As for HistGB, the hour actually enters as a number, needing two cuts while the tariff already joins these cuts in one variable.

The tariff ends up being a facilitator of the tree's work, but it is just another way of presenting the same information already existing in the hour, not new information. This point had already been discussed in previous notebooks.

### 3.2.4 How much does the station history contribute?

Deep down the station history ends up being, by far, the group that contributes the most in the analyses. Relative to chance, the models go from about 1,5× to about 3× (table 3.2.6). When going from G2 to G3, the AUC-PR rises on average +0,29 in LR and +0,32 in HistGB (table 3.2.2b), while the tariff brings almost nothing and the user history brings about +0,02.

In addition, with G3, both models start to beat the rule in the 3 folds, which did not happen with groups 1 and 2. Since the rule uses only rate_1D, this shows that combining the complete station history (7 and 30 days, hours since the last failure, among other variables) ends up bringing more information beyond that of the previous day.


### 3.2.5 Does the user history bring additional benefit?

The user history ends up bringing a small, but consistent, benefit. When going from group 3 to group 4, the AUC-PR rises on average +0,017 in LR and +0,019 in HistGB (table 3.2.2b), much less than the station history, but the gain is positive in the 6 comparisons (3 folds × 2 models). However, this gain ends up decreasing from fold 1 to fold 3 (from +0,026 to about +0,01 in both models). One hypothesis is that, over time, more new users come in, without history, and thus the user history ends up helping less, but this was not verified

### 3.2.6 Is the effect consistent across periods?

Table 3.2.6 — AUC-PR divided by the prevalence of each fold (how many times better than chance)

| Model | Group | Fold 1 | Fold 2 | Fold 3 |
|---|---|---|---|---|
| Rule | — | 2,85× | 3,06× | 2,81× |
| LR | G1 | 1,58× | 1,58× | 1,59× |
| LR | G2 | 1,58× | 1,58× | 1,59× |
| LR | G3 | 3,21× | 3,23× | 2,93× |
| LR | G4 | 3,37× | 3,31× | 2,98× |
| HistGB | G1 | 1,47× | 1,53× | 1,63× |
| HistGB | G2 | 1,48× | 1,53× | 1,65× |
| HistGB | G3 | 3,41× | 3,36× | 3,05× |
| HistGB | G4 | 3,57× | 3,45× | 3,10× |

Overall, yes, the effect ends up being consistent. Looking at table 3.2.2b, all the gains have the same sign in the 3 folds, that is, no variable group makes the model worse in any period. The station history and the user history raise the AUC-PR in the 3 folds and in both models, and the tariff stays practically at zero in all of them.

As for table 3.2.6, the AUC-PR was divided by the prevalence of each fold, because the AUC-PR of chance changes from fold to fold (15,8%, 19,7% and 21,0%), and thus we have how many times the model is better than chance. In fold 3, the models with the station history end up dropping (HistGB in group 3 goes from 3,36× to 3,05×), but the rule also drops (from 3,06× to 2,81×). Since the rule is not trained, this indicates that the drop comes mainly from the period itself, and not from the models themselves.

### 3.2.7 Operational and ethical concerns

Table 3.2.7 — HistGB on G3, evaluated on subgroups of sessions (the model is the same; only the evaluated set changes)

|  | Fold 1 | Fold 2 | Fold 3 |
|---|---|---|---|
| AUC-PR — all sessions | 0,538 | 0,661 | 0,642 |
| AUC-PR — without the fleet sessions | 0,539 | 0,660 | 0,643 |
| AUC-PR — only the fleet sessions | 0,504 | 0,685 | 0,618 |
| Prevalence — all sessions | 15,8% | 19,7% | 21,0% |
| Prevalence — only the fleet sessions | 14,0% | 19,9% | 21,8% |
| All sessions ÷ prevalence | 3,41× | 3,36× | 3,05× |
| Only fleets ÷ prevalence of the fleets | 3,61× | 3,44× | 2,83× |
| No. of fleet sessions | 2097 | 2410 | 2481 |

With G4, we have a big problem, the risk of a session ending up also depending on who is charging the vehicle. This can mark users for failures they did not cause and let the fleet accounts dominate the pattern. Since the alert goes only to the operator and does not block the session, an error costs at most one extra inspection, but even so it was one of the reasons for us to choose G3.

The station history also has its problems: failures that come from the car (such as Vehicle charging faults, which also ends up not being the user's fault) end up counting against the station, and new stations or ones with few sessions have an unreliable history. As for table 3.2.7, it shows that, on G3, removing the fleet sessions hardly changes the AUC-PR, that is, the model works in a similar way for the fleets and for the rest. The limit is that there are only 2 fleet accounts.

### 3.2.8 Additional variables

Table 3.2.8 — HistGB on G3, with and without the `Charging Post ID` (in one-hot)

| Model | Fold 1 | Fold 2 | Fold 3 | Mean | Deviation | Worst fold | Folds > rule |
|---|---|---|---|---|---|---|---|
| HistGB G3 | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| HistGB G3 + `Charging Post ID` | 0,538 | 0,661 | 0,639 | 0,613 | 0,066 | 0,538 | 3 |
| Difference (with − without) | +0,0001 | −0,0001 | −0,0024 | −0,0008 | — | — | — |

The hour and the day of the week were already justified in section 3.1. Charging Post ID was tested in HistGB with G3 and brought no gain (table 3.2.8), which makes sense, since the station history already describes each station and the ID does not help with new stations. UserID was not tested, for privacy reasons, and was used only to identify the fleet accounts (39 and 23).


### 3.2.9 Group chosen for the models

We ended up choosing G3 for the models (we already separated G4 as the final one for this reason). G4 ends up being a bit better (about +0,02 of AUC-PR), but the gain is small and brings the ethical problems of section 3.2.7, besides going against the exclusion of UserID. The Task 2 protocol says which variables can be used, but does not require using all of them.


---

## 3.3 Individual models

### 3.3.1 Chosen algorithms

Table 3.3.1 — Modeling principles

| Principle | Algorithm | Used |
|---|---|---|
| Linear | Logistic regression | ✅ |
| Probabilistic | Naive Bayes (mixed version) | ✅ |
| Decision trees | Decision tree (depth 6) | ✅ |
| Instance-based | kNN | — |
| Kernel-based | SVM | — |

We chose three models with different principles: logistic regression (linear), mixed Naive Bayes (probabilistic) and the decision tree (trees). Each one ends up looking at the data in a different way, they are quick to train and easy to interpret. 

kNN and SVM ended up being left out due to the size of the data: with about 180 000 sessions per training, kNN would need to store all the sessions and compare each new session with all of them, and SVM ends up being very slow to train. As the assignment statement itself says, using many algorithms without analysis is not better than a smaller and well-justified comparison.

### 3.3.2 Justification by the six criteria

Logistic regression was chosen for the interpretability, since the coefficients show the sign and the size of the effect of each variable on the anomaly risk. 

As for mixed Naive Bayes, it ends up being the cheapest model, since training is just counting frequencies in the categorical ones and calculating means and deviations in the numeric ones. 

Finally the decision tree can capture thresholds and interactions between variables (for example, location × station history) without needing scaling.

The independence assumption of Naive Bayes ends up being violated here, which appears in the extreme probabilities (72%, table 3.3.4).

### 3.3.3 Tree depth

Table 3.3.3 — Decision tree on G3 with several depths

| `max_depth` | Training (mean) | Fold 1 | Fold 2 | Fold 3 | Mean | Deviation | Worst fold | Folds > rule |
|---|---|---|---|---|---|---|---|---|
| 3 | 0,505 | 0,437 | 0,568 | 0,544 | 0,516 | 0,070 | 0,437 | 0 |
| 4 | 0,542 | 0,469 | 0,602 | 0,582 | 0,551 | 0,072 | 0,469 | 1 |
| **6** | 0,576 | 0,503 | 0,626 | 0,606 | 0,578 | 0,066 | 0,503 | 3 |
| 8 | 0,599 | 0,504 | 0,637 | 0,585 | 0,575 | 0,067 | 0,504 | 2 |
| 10 | 0,621 | 0,485 | 0,611 | 0,561 | 0,552 | 0,063 | 0,485 | 2 |
| 12 | 0,652 | 0,433 | 0,554 | 0,522 | 0,503 | 0,063 | 0,433 | 0 |
| No limit | 1,000 | 0,199 | 0,274 | 0,276 | 0,249 | 0,044 | 0,199 | 0 |

The depth of the tree was chosen on the validation, testing several values and seeing which had the best result in the folds. With depth 3 and 4, the AUC-PR is low both in the training and in the validation, that is, the tree is too simple (underfitting). From 8 on, the training keeps rising, but the validation ends up dropping, which indicates overfitting. 6 and 8 are practically tied on the mean (0,578 and 0,575) and on the worst fold, but only 6 beats the rule in the 3 folds, which is the protocol criterion, so we chose 6.

The tree without limit drops to 0,249 most probably because it memorizes the training. The limitation is that the depth was chosen on the same folds where we later compare the models, which may leave the tree a bit favored.

### 3.3.4 Naive Bayes: Gaussian version and mixed version

Table 3.3.4 — Naive Bayes on G3: Gaussian version (old) and mixed version

| Version | Fold 1 | Fold 2 | Fold 3 | Mean | Deviation | Worst fold | Folds > rule | Extreme prob. (fold 3) |
|---|---|---|---|---|---|---|---|---|
| GaussianNB (one-hot + normal on all columns) | 0,444 | 0,586 | 0,566 | 0,532 | 0,077 | 0,444 | 0 | 76,8% |
| Mixed NB (categorical by frequencies, numeric by the normal) | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 | 72,0% |
| Mixed NB without `tariff_period` | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 | 71,9% |

In the Gaussian version, all the columns were treated as numbers with normal distribution, even the categorical ones, which are only worth 0 or 1. That is why we moved to the mixed NB, where the categorical ones use frequencies and the numeric ones continue with the normal. With this, the mean AUC-PR rose from 0,532 to 0,551 in the means.

Even so, 72% of the probabilities remain extreme (below 1% or above 99%), and removing the tariff changes nothing. Therefore, the problem ends up being the independence assumption: the station variables are correlated and the model counts the same information several times.

NOTE: Extreme prob. = percentage of probabilities below 1% or above 99% in the validation of fold 3. In logistic regression it is 0,3%.

### 3.3.5 Results

Table 3.3.5 — Individual models on G3

| Model | Fold 1 | Fold 2 | Fold 3 | Mean | Deviation | Worst fold | Folds > rule |
|---|---|---|---|---|---|---|---|
| No-skill | 0,158 | 0,197 | 0,210 | 0,188 | 0,027 | 0,158 | — |
| Rule | 0,450 | 0,602 | 0,590 | 0,548 | 0,084 | 0,450 | — |
| Logistic regression | 0,506 | 0,636 | 0,617 | 0,586 | 0,070 | 0,506 | 3 |
| Mixed Naive Bayes | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 |
| Decision tree (depth 6) | 0,503 | 0,626 | 0,606 | 0,578 | 0,066 | 0,503 | 3 |

Among the individual models, logistic regression ends up being the best (0,586 mean), followed by the tree (0,578) and Naive Bayes (0,551). LR and the tree beat the rule in the 3 folds. As for NB, despite having a mean slightly above the rule (0,551 against 0,548), it loses to it in 2 folds. By the criterion of the Task 2 protocol, which asks to beat the rule in all folds, NB ends up not being considered a good model for our data.

### 3.3.6 Interpretation of the models


Table 3.3.6a — Logistic regression (G3, fold 3 model): the 10 largest coefficients in absolute value

| Variable (after preprocessing) | Coefficient |
|---|---|
| `post_previous_abnormal_rate_1D` (scale) | +0,932 |
| `post_previous_abnormal_1D` (log + scale) | −0,804 |
| `Location Information` = Technology Park | −0,582 |
| `post_hours_since_previous_completed_abnormal` (log + scale) | −0,573 |
| `post_previous_abnormal_rate_7D` (scale) | +0,421 |
| `Location Information` = Tourist Attraction | +0,293 |
| `post_previous_sessions_1D` (scale) | +0,293 |
| `Location Information` = Park A | +0,228 |
| `Location Information` = Industrial Park | −0,226 |
| `Location Information` = Shopping Mall | +0,217 |

Table 3.3.6b — Decision tree (G3, depth 6, fold 3 model): most important variables

| Variable | Importance |
|---|---|
| `post_previous_abnormal_rate_7D` | 0,557 |
| `post_hours_since_previous_completed_abnormal` | 0,171 |
| `post_previous_abnormal_rate_1D` | 0,149 |
| `post_previous_abnormal_rate_30D` | 0,104 |
| `Location Information` = Bus Station | 0,003 |
| Each one of the remaining variables | < 0,002 |
| Sum of the 4 station variables | 0,982 |

Both models agree that what weighs the most is the station history, which matches section 3.2.4: in the tree, the 4 station variables add up to 0,982 of the importance, and in LR the largest coefficients are also almost all from the station. The difference is that LR relies more on rate_1D, which was already the variable most correlated with the target in Task 1. The tree relies more on rate_7D, which, by using 7 days, ends up being more stable and giving better cuts.

In LR, rate_1D and abnormal_1D have opposite signs (+0,932 and −0,804). This does not mean that more failures yesterday lower the risk. Since the two variables are correlated, the model ends up dividing the effect between them, and that is why the coefficients must be read together, and not one by one.

---

## 3.4 Ensemble models


### 3.4.1 Approaches and why they can improve their base models

Table 3.4.1 — Evaluated ensembles

| Ensemble | Principle (Lecture 5) | Base models | Configuration |
|---|---|---|---|
| Random Forest | Bagging + random selection of variables at each split | Trees without depth limit (`min_samples_leaf=20`), each one on a bootstrap sample | 100 trees, `class_weight='balanced_subsample'` |
| HistGradientBoosting | Boosting | Small trees, trained in sequence | Default values, `class_weight='balanced'`, `early_stopping=False` |
| Soft voting | Heterogeneous and static combination | Logistic regression + mixed NB + tree (depth 6) | Mean of the probabilities; also evaluated without the NB |

Random Forest combines 100 trees, each one trained on a different sample of the training (bagging), and averages the predictions. It ends up reducing the variance, which is the main problem of a single tree, since small changes in the data change the tree a lot. The random selection of variables at each split serves to make the trees different from each other. Without it, all of them would start with rate_7D and would end up similar, and the mean would not help as much.

HistGB trains small trees in sequence, and each one tries to correct the errors of the previous ones (boosting), which ends up reducing the bias. early_stopping was turned off because, by default, it separates a random 10% of the training to decide when to stop, which would end up mixing the future with the past. HistGB is not in the slides, but it works like LightGBM: it groups the values into intervals, which makes it fast with 180 thousand sessions.

The voting averages the probabilities of LR, NB and the tree. To help, the models need to be good on their own and to err on different sessions. Since the three have different principles, we expected some diversity, but all of them rely mainly on the station history, so they end up erring on similar sessions. In addition, NB, with very extreme probabilities, may end up getting in the way.

### 3.4.2 Results: do the ensembles improve their base models?

**Table 3.4.2a** — Ensembles on G3

| Ensemble | Fold 1 | Fold 2 | Fold 3 | Mean | Deviation | Worst fold | Folds > rule |
|---|---|---|---|---|---|---|---|
| Random Forest | 0,533 | 0,660 | 0,641 | 0,612 | 0,069 | 0,533 | 3 |
| HistGradientBoosting | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| Voting (LR + NB + tree) | 0,514 | 0,643 | 0,622 | 0,593 | 0,070 | 0,514 | 3 |
| Voting without the NB (LR + tree) | 0,524 | 0,650 | 0,628 | 0,601 | 0,067 | 0,524 | 3 |

**Table 3.4.2b** — AUC-PR difference between each ensemble and its base models (positive = the ensemble is better)

| Comparison | Fold 1 | Fold 2 | Fold 3 |
|---|---|---|---|
| Random Forest − tree (depth 6) | +0,0302 | +0,0344 | +0,0354 |
| HistGB − tree (depth 6) | +0,0347 | +0,0347 | +0,0355 |
| HistGB − Random Forest | +0,0045 | +0,0003 | +0,0001 |
| Voting − LR | +0,0073 | +0,0070 | +0,0053 |
| Voting − mixed NB | +0,0347 | +0,0423 | +0,0493 |
| Voting − tree (depth 6) | +0,0104 | +0,0174 | +0,0160 |
| Voting without NB − LR | +0,0175 | +0,0136 | +0,0112 |
| Voting without NB − tree (depth 6) | +0,0207 | +0,0240 | +0,0220 |
| Voting without NB − Voting | +0,0102 | +0,0066 | +0,0060 |

All the ensembles end up being better than their base models in the 3 folds (table 3.4.2b). Random Forest and HistGB gain about +0,03 over the best individual tree, and are practically tied with each other. This is interesting in the Random Forest, because it is made of deeper trees, which alone would end up overfitting (table 3.3.3). That is, bagging ends up correcting this problem.

In the voting, removing the NB improves the result in the 3 folds. This happens because NB has very extreme probabilities and, since it does not use class_weight='balanced', it is on a different scale from LR and the tree, pulling the mean too much. Even without the NB, the voting does not reach the Random Forest or the HistGB, because it combines only two or three weaker models, which err on similar sessions.

### 3.4.3 Stacking

We ended up not using stacking. To use it without leakage, the base models would have to be trained on an older period and the meta-model on the following period, always before the validation. The normal StackingClassifier, with cv=5, does not work, because it splits the data randomly and would end up mixing the future with the past.

---

## 3.5 Final comparison

**Table 3.5a** — All the models on G3, in order of the mean

| Model | Type | Fold 1 | Fold 2 | Fold 3 | Mean | Deviation | Worst fold | Folds > rule |
|---|---|---|---|---|---|---|---|---|
| HistGradientBoosting | Ensemble | 0,538 | 0,661 | 0,642 | 0,613 | 0,066 | 0,538 | 3 |
| Random Forest | Ensemble | 0,533 | 0,660 | 0,641 | 0,612 | 0,069 | 0,533 | 3 |
| Voting without the NB | Ensemble | 0,524 | 0,650 | 0,628 | 0,601 | 0,067 | 0,524 | 3 |
| Voting (LR + NB + tree) | Ensemble | 0,514 | 0,643 | 0,622 | 0,593 | 0,070 | 0,514 | 3 |
| Logistic regression | Individual | 0,506 | 0,636 | 0,617 | 0,586 | 0,070 | 0,506 | 3 |
| Decision tree (depth 6) | Individual | 0,503 | 0,626 | 0,606 | 0,578 | 0,066 | 0,503 | 3 |
| Mixed Naive Bayes | Individual | 0,479 | 0,601 | 0,573 | 0,551 | 0,064 | 0,479 | 1 |
| Rule | Baseline | 0,450 | 0,602 | 0,590 | 0,548 | 0,084 | 0,450 | — |
| No-skill | Baseline | 0,158 | 0,197 | 0,210 | 0,188 | 0,027 | 0,158 | — |

**Table 3.5b** — Difference to the rule in each fold (positive = better than the rule)

| Model | Fold 1 | Fold 2 | Fold 3 |
|---|---|---|---|
| HistGradientBoosting | +0,087 | +0,058 | +0,051 |
| Random Forest | +0,083 | +0,058 | +0,051 |
| Voting without the NB | +0,073 | +0,048 | +0,038 |
| Voting (LR + NB + tree) | +0,063 | +0,041 | +0,032 |
| Logistic regression | +0,056 | +0,034 | +0,027 |
| Decision tree (depth 6) | +0,053 | +0,024 | +0,016 |
| Mixed Naive Bayes | +0,028 | −0,001 | −0,017 |

In the end, the ensembles end up ahead of the individual models, with HistGB (0,613) and Random Forest (0,612) at the top. All the models beat the rule in the 3 folds, except NB, which has a mean slightly above the rule but loses to it in 2 folds. 

This shows that looking only at the mean is not enough. HistGB and Random Forest are practically tied: HistGB wins in the 3 folds, but by very little (table 3.4.2b). By the Task 2 protocol, HistGB comes first for having a slightly better mean and worst fold, but since the difference is very small, both go on to the next step (tuning, threshold and final evaluation).

---

## 3.6 Limitations

Choices made on the same folds as the comparison: the depth of the tree, the NB version and the voting without the NB were chosen by looking at the validation of the same folds where we later compare the models. This may leave these models a bit favored, since they ended up being tuned to these periods. The ideal would be to choose on a separate period, but with only 3 folds this would leave less data for the comparison.

No statistical test: with only 3 folds, the Wilcoxon test can never give significant, because the smallest possible p-value is 0,25. Therefore, the comparison ends up being made by the fold-by-fold consistency (does the model win in the 3 folds or not?). Very small differences, such as the one between HistGB and Random Forest, cannot be considered a real difference.

Few fleet sessions: the fleet analysis uses only 2 accounts and few sessions, so the AUC-PR of this subgroup can change a lot with a few sessions more or less. It is not possible to draw strong conclusions from it.

Poorly calibrated NB probabilities: 72% of the NB probabilities are below 1% or above 99%, so they cannot be read as the real risk of a session. That is also why NB gets in the way of the voting.

Ensembles without tuning: Random Forest and HistGB still use the default values, changing only class_weight and early_stopping. The results may improve with tuning, and the order between the two may change. This is left for the next step.

Car failures in the station history: some failures, such as the BMS ones or Vehicle charging faults, come from the car and not from the station, but end up counting in the station history. This division was made only by the name of each reason and was not verified.


---

# Task 4 — Hyperparameter optimization and cross-validation

## 4.1 Chosen hyperparameters

Table 4.1 — Best hyperparameters on G3, searched on the 3 temporal folds of Task 2 with F1 as the score

| Model | Search | Best hyperparameters | F1 (mean across the folds) |
|---|---|---|---|
| Random Forest | `RandomizedSearchCV`, 10 combinations | `n_estimators=500`, `max_depth=None`, `min_samples_split=2`, `max_features='sqrt'` | nan (the 30 fits failed due to lack of memory) |
| HistGradientBoosting | `BayesSearchCV`, 40 iterations | `learning_rate=0,037`, `max_depth=None`, `max_iter=393`, `min_samples_leaf=20` | 0,5243 |

## 4.2 Comparison of the thresholds

Table 4.2 — Metrics on the OOF predictions of the 3 validations, with the default threshold (0,50) and with the threshold that maximizes F1 (searched from 0,05 to 0,95, in steps of 0,01)

| Model | Threshold | Precision | Recall | F1 | Alert rate | Correct alerts / 1 000 | False alerts / 1 000 | Missed anomalies / 1 000 |
|---|---|---|---|---|---|---|---|---|
| Random Forest | 0,50 | 0,7936 | 0,3268 | 0,4629 | 7,76% | 61,55 | 16,01 | 126,81 |
| Random Forest | 0,29 | 0,5723 | 0,5263 | 0,5484 | 17,32% | 99,14 | 74,08 | 89,21 |
| HistGradientBoosting | 0,50 | 0,4293 | 0,6873 | 0,5285 | 30,15% | 129,46 | 172,08 | 58,89 |
| HistGradientBoosting | 0,63 | 0,5710 | 0,5455 | 0,5580 | 17,99% | 102,74 | 77,19 | 85,61 |

## 4.3 Analysis of the results

- **HistGradientBoosting with optimized threshold (0,63)** presents the best overall performance, with an **F1-score of 0,5580**.
- The threshold optimization improved the F1-score of both models:
	- Random Forest: **0,4629 → 0,5484**
	- HistGradientBoosting: **0,5285 → 0,5580**
- In HistGradientBoosting, increasing the threshold from **0,50 to 0,63** significantly reduced the false alerts (**172,08 → 77,19 per 1.000 sessions**), while maintaining a reasonable recall (**0,5455**).
- The default threshold of HistGradientBoosting offers the highest recall (**0,6873**), but generates many false alerts and presents lower precision.
- The optimized Random Forest has balanced performance, but stays slightly below the optimized HistGradientBoosting.
- Thus, the recommended model is **HistGradientBoosting with threshold 0,63**, for presenting the highest F1-score and a more adequate trade-off between anomaly detection and number of false alerts.

The results were calculated on the temporal OOF predictions; therefore, the optimized threshold should later be confirmed on an independent temporal test set.

---

# Task 5 — Final model selection

## 5.1 Evaluation on the final test

Table 5.1 — HistGradientBoosting (threshold 0,63) and no-skill baseline on the final test (the most recent 20% of sessions, 88 216 sessions)

| Model | PR-AUC | Precision | Recall | F1 | Balanced accuracy | MCC | Alert rate | Correct alerts / 1 000 | False alerts / 1 000 | Missed anomalies / 1 000 |
|---|---|---|---|---|---|---|---|---|---|---|
| Baseline (no-skill) | 0,1810 | 0,0000 | 0,0000 | 0,0000 | 0,5000 | 0,0000 | 0,00% | 0,00 | 0,00 | 181,00 |
| HistGradientBoosting (threshold 0,63) | 0,3223 | 0,4898 | 0,4577 | 0,4732 | 0,6762 | 0,3619 | 16,91% | 82,84 | 86,30 | 98,16 |

## 5.2 Analysis of the results

**HistGradientBoosting with optimized threshold** presents clearly superior performance to the baseline:

- The model identifies **45,77% of the abnormal sessions** (*recall*).
- Among the alerts issued, **48,98% are really anomalies** (*precision*).
- In every 1.000 sessions, it generates approximately:
    - **82,8 correct alerts**;
    - **86,3 false alerts**;
    - **98,2 undetected anomalies**.
- The alert rate is **16,91%** of the sessions.
- The *Balanced Accuracy* (**0,6762**) and the MCC (**0,3619**) indicate moderate predictive capacity, superior to chance.

The baseline does not generate alerts because the estimated probability (**0,1869**) is lower than the default threshold of 0,5. Therefore, it obtains precision, recall and F1 equal to zero.

Thus, the model is useful as a triage system, but it still lets about half of the abnormal sessions slip through and produces a similar amount of false alerts.
