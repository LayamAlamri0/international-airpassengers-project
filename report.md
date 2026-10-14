# Air Passenger Forecasting Report

## Recommendation

I would choose seasonal AutoARIMA for the point forecasts. It gave the lowest average error in the eight-window comparison, with a MASE of 0.647, compared with 1.313 for SeasonalNaive. This is about a 51% reduction in average scaled absolute error.

There is still a limitation: AutoARIMA's 80% prediction intervals covered only 69.8% of the observations. I would therefore use its forecasts as a starting point for planning, while allowing for more uncertainty than the original bands suggest.

## What I found in the data

The data contains 144 monthly observations from January 1949 to December 1960. Passenger numbers are measured in thousands, and the calendar check found no missing months.

Passenger numbers increase over time, with a similar pattern repeating each year. The highest values tend to occur around the middle of the year. The seasonal rises and falls also become larger as passenger numbers grow.

The STL decomposition separates the rising trend from the yearly seasonal pattern. Its remainder is much smaller than these two components, although some spikes remain. Trend strength rounds to 1.00 and seasonal strength to 0.99. The raw-series ACF also has a slow decline and local peaks at 12 and 24 months. These findings made a seasonal model that can handle growth a reasonable choice to test.

## How I compared the models

I started with SeasonalNaive, which repeats the last year's monthly values. This keeps the yearly pattern but misses continued growth. I also included Naive, which uses the latest observation for every future month.

For AutoGluon, I converted the data into its TimeSeriesDataFrame format. The training input contained 132 months, with the last 12 months kept for a separate holdout comparison. The final model search included Naive, SeasonalNaive, AutoETS, and AutoARIMA. AutoGluon also built a weighted ensemble.

Theta was tried initially, but it failed and used SeasonalNaive as a fallback. I removed it from the final search because its result did not represent a successful independent Theta forecast.

AutoGluon shows negative MASE scores, so closer to zero means better. WeightedEnsemble had the best internal validation score, while AutoARIMA performed best on the final-year holdout. Since the internal ranking used only one validation window, I did not treat it as enough evidence on its own.

I then compared the StatsForecast seasonal AutoARIMA model with both benchmarks using eight rolling origins. Each origin forecast the next 12 months, with a 12-month step between origins. This evaluated the years 1953 to 1960. At each cutoff, the models used only the history available then. The scoring function also used each fold's training history to calculate the MASE and RMSSE denominators.

| Model | MASE | RMSSE | Scaled CRPS | 80% coverage | Lowest fold MASE | Highest fold MASE |
|---|---:|---:|---:|---:|---:|---:|
| SeasonalNaive | 1.313 | 1.246 | 0.062 | 51.0% | 0.412 | 1.959 |
| Naive | 2.119 | 2.452 | 0.110 | 68.8% | 1.422 | 3.196 |
| AutoARIMA | 0.647 | 0.683 | 0.034 | 69.8% | 0.330 | 1.594 |

AutoARIMA was best on all three error measures, where lower is better. Its lower scaled CRPS also supports its forecast distribution over the benchmarks, despite the coverage problem. Naive had higher errors than SeasonalNaive on all three measures.

AutoARIMA's MASE ranged from 0.330 to 1.594 across the windows, so its performance was not equally strong every year. These results support it over the tested benchmarks, but do not prove it would beat every model in every period. The rolling evaluation also includes the final year already used during shortlisting, so it is not a completely untouched final test.

## Forecasts and uncertainty

After evaluation, I refitted AutoARIMA to the full series and forecast the next 24 months. The table shows three points in that forecast. All values are in thousands of passengers.

| Month | Forecast | 80% lower bound | 80% upper bound | 80% interval width |
|---|---:|---:|---:|---:|
| January 1961 | 444.31 | 429.25 | 459.37 | 30.12 |
| December 1961 | 459.24 | 418.45 | 500.02 | 81.57 |
| December 1962 | 486.47 | 396.20 | 576.74 | 180.54 |

The intervals become wider further into the future. For January 1961, the 95% interval width is 46.06 thousand passengers, compared with 30.12 thousand for the 80% interval. By December 1961, the widths are 124.74 and 81.57 thousand respectively.

However, wider intervals are not automatically reliable. In the rolling evaluation, the 80% bands covered 51.0% of observations for SeasonalNaive, 68.8% for Naive, and 69.8% for AutoARIMA. All were below the intended 80%.

AutoARIMA had the best coverage of the three, but still missed the target by about ten percentage points. Its bands suggest more certainty than the evaluation supports. This coverage result is for 12-month forecasts; it does not establish coverage for the second forecast year. I have reported the original intervals without claiming to have calibrated them.

## What the models missed

SeasonalNaive's residuals, calculated as the observed value minus the value 12 months earlier, were mostly positive. This shows how it often underestimated passenger numbers by missing growth. Their ACF had strong positive correlation at short lags, but no clear positive spike at lag 12. There was still a pattern in these errors that a stronger model could use.

The AutoARIMA residual ACF had no spikes outside the approximate confidence bounds through lag 24. This suggests that it removed much of the dependence left by SeasonalNaive. It does not prove that every aspect of the errors is satisfactory. In particular, the interval coverage still needs improvement.

## What I would try next

My next step would be to test interval calibration using errors from earlier rolling origins. At each new origin, I would estimate an adjustment from earlier available errors, then check coverage and scaled CRPS on the next unseen window.

The aim would be coverage closer to 80%, although this would likely make the intervals wider. I would keep that change only if the later evaluation supported it. A separate 24-month evaluation would also be needed to assess second-year coverage.

For now, AutoARIMA is my preferred point forecasting model among those tested in the rolling comparison. Its uncertainty estimates need further work before being relied on for decisions where underestimating demand would be costly.
