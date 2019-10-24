Pre/Post post-hoc meter underregistration detector workflow

Step 1: Identify &quot;Meter Lines&quot; (Note: Step can be skipped if given list of meter replacements)

1. Group meters by shared **location, size,** and **occupant**.
2. Identify groups of meters with **adjacent** , **non-overlapping** time series.
3. Give each identified pair of meters a unique identifier to create a concatenated time series I call &quot;Meter Line&quot;.
4. Remove all readings that occur after a threshold gap in readings. I choose 93 days (3 months)

Step 2: Shift all time series to regular monthly interval where possible

1. Normalize metered volumes by number of days represented in reading
2. Expand time series to daily frequency (This is a computationally expensive step, there are probably better ways you might know to do this)
3. Summarize/collapse all time series to common monthly interval.

Step 3: Match &quot;old&quot; meter time series to most similar _k_ meter time series of non-replaced meters

1. Separate data into &quot;Meter Lines&quot; and &quot;Non-replaced meters&quot;
2. Spread Non-replaced meter data so each row is a month, and each column represented one meter
3. Loop: For each Meter Line-

1.
  1. Left join all Non-replaced meter time series, matching by month
  2.
ii.Remove any Non-replaced meter time series columns that have ANY missing values.1
  3. Remove all rows corresponding to observations from the &quot;new&quot; meter in the Meter Line
  4. Scale/ standardize all the columns
  5. Search all columns to identify _k_ nearest Non-replaced meter time series to the Meter Line in terms of Euclidean Distance. Use the [UCR Suite](https://www.cs.ucr.edu/~eamonn/UCRsuite.html) (has [R](https://cran.r-project.org/web/packages/rucrdtw/index.html) and [Python](https://github.com/klon/ucrdtw) bindings) for fast performance.
  6. Using the above information, wrangle a data frame like so (in the original unstandardized units)

| Timestamp | Meter Line ID | Meter Line Volume | New\_Meter\_indicator | Match-1 volume | Match-2 volume | Match-_k_ volume |
| --- | --- | --- | --- | --- | --- | --- |
| May 2019 | 1 | 10 | 0 | 4 | 9 | 13 |
| Jun 2019 | 1 | 12 | 0 | 5 | 12 | 15 |
| Jul 2019 | 1 | 10 | 0 | 4 | 11 | 14 |
| Aug 2019 | 1 | 9 | 1 | 3 | 9 | 12 |
| Sept 2019 | 1 | 8 | 1 | 2 | 8 | 12 |

1.
  1. Repeat i-vi, binding all Meter Lines and Matches 1-k into one data frame

| Timestamp | Meter Line ID | Meter Line Volume | New\_Meter\_indicator | Match-1 volume | Match-2 volume | Match-_k_ volume |
| --- | --- | --- | --- | --- | --- | --- |
| May 2019 | 1 | 10 | 0 | 4 | 9 | 13 |
| Jun 2019 | 1 | 12 | 0 | 5 | 12 | 15 |
| Jul 2019 | 1 | 10 | 0 | 4 | 11 | 14 |
| Aug 2019 | 1 | 9 | 1 | 3 | 9 | 12 |
| Sept 2019 | 1 | 8 | 1 | 2 | 8 | 12 |
| May 2019 | 2 | 2 | 0 | 2 | 3 | 4 |
| Jun 2019 | 2 | 2 | 0 | 2 | 3 | 3 |
| Jul 2019 | 2 | 4 | 1 | 2 | 3 | 2 |
| Aug 2019 | 2 | 4 | 1 | 1 | 3 | 3 |
| Sept 2019 | 2 | 4 | 1 | 2 | 3 | 2 |



Step 4: Forecast &quot;old&quot; meters within each Meter Line into the &quot;future&quot; with confidence bands

1. For each Meter Line:
  1. Filter data from step 3.3.vii to that Meter Line
  2. Decide to proceed if and only if:

1.
  1.
    1. There are &quot;enough&quot; readings from the &quot;old&quot; meter (I chose 24 months)
    2. There is at least one non-zero reading from the &quot;old&quot; meter
    3. There is at least 2 readings from the &quot;new&quot; meter

1.
  1. Construct Bayesian Structure Time Series Forecast, where:

1.
  1.
    1. The response is the Meter Line volumes **from the &quot;old&quot; meter**
    2. The possible predictors are the Match volumes
    3. The training period does not include the last reading of the &quot;old&quot; meter, which is often aberrant.
    4. The prediction/ comparison period does not include the first reading of the &quot;new&quot; meter, which is also often aberrant
    5. RECCOMENDED: Restrict training period to &#39;representative period&#39;. I choose only up to 4 years before. I would do up to just one year for Daily data. THIS CAN BE IMPROVED. I WOULD LOOK INTO IDENTIFYING STRUCTURAL BREAKS IN THE OLD METER SERIES AND RESTRICTING THE TRAINING PERIOD TO TIME AFTER THE MOST RECENT BREAK
    6. All variables are log-transformed to give a lower bound of 0 to predictions
    7. An appropriate random-walk component is specified (I choose semi-local linear trend)
    8. An appropriate seasonality component is specified (I choose month indicators). Could be a harmonic function, or more complex seasonalities with higher-frequency data)

1.
  1. Infer if the forecast is &quot;significantly&quot; less than the actual time series from the &quot;new&quot; meter.
    1. Step 1 produces many MCMC forecasts, from which confidence intervals (width and confidence of your choice, I pick 95% intervals at alpha=0.05) around the mean forecast are constructed. If in the &quot;new&quot; meter period, the actual new meter readings on average fall outside the confidence interval, we judge the forecast significantly different from the actual time series. If on average the actual readings are **greater** than the forecast, we judge this to be a potential under-registering mete.
    2. If the penultimate reading of the old meter is 0, we judge this to be the case of a dead meter rather than an underregistering meter.

STEP 4.1.iii can be done with a mild assist from the _bsts_ R package. Alternatively, Steps 4.1.iii and 4.1.iv can be wrapped in one step with the _CausalImpact_ R package. I am unaware if either have been implemented in python. Both are completely open source so could be ported in principle.

1.
  1. Calculate revenue consequences
    1. If in step iv we identify an under-registering meter replacement, calculate the average monthly volume recovered by subtracting the mean forecast values from the actual &quot;new&quot; meter readings, and taking the average of these differences.
    2. Calculate the monthly revenue recovered by applying the water rate structure to
      1. The average of the actual &quot;new&quot; meter readings
      2. The average of the forecasted &quot;old&quot; meter readings
      3. Subtract the value from (1) from the value of (2)





1

#
 This ensures that we only match Replaced meters to Non-replaced meters that have at least the same temporal coverage.

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
