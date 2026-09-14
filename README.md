
> ⚠️ **NOTE:** This document provides an overview of an ongoing private research project. The associated source code is currently restricted and will not be displayed at this time.

# About the Project

This project presents an exploratory data analysis (EDA) and a reproducible data preparation pipeline designed for predicting New York City yellow taxi fares. The objective is to describe the data, identify observations inconsistent with the source definitions, construct time and distance features, and subsequently compare regression models. The results presented here are descriptive; they do not yet constitute a definitive evaluation of predictive performance.

## Objectives

- Understand data structure and quality;
- Identify and handle inconsistent values;
- Engineer useful features for fare prediction;
- Compare different regression models;
- Evaluate and interpret the results.

## Data

Each row represents a single trip made by a yellow taxi. The dataset includes pickup and drop-off dates/times, pickup and drop-off zones, trip distance, passenger count, payment type, and total fare amount.

The project currently utilizes a stratified sample of **100,000 trips** extracted from the twelve official 2018 TLC Parquet files. Sampling was performed locally with a fixed random seed of `42`, proportionally across the `month × day of week × hour` strata of the full source dataset. This value ensures the reproducibility of the sampling process; it carries no scientific hypothesis and could be replaced by any other fixed value. This methodology preserves the observed temporal structure, including weekends, without depending on row ordering or Socrata API limits.

Official Source: [New York City Taxi and Limousine Commission (TLC)](https://nyc.gov)

Socrata Dataset: [2018 Yellow Taxi Trip Data](https://cityofnewyork.us)

## Project Organization

```text
data/
└── taxi_2018_sample.csv
notebooks/
└── 01_exploration_initiale.ipynb
scripts/
└── 01_exploration_initiale.py
```
*(Note: Code files are restricted for the time being)*

## Installation and Usage

Install the core dependencies:

```bash
python -m pip install pandas matplotlib
```

Run the exploration script:

```bash
python scripts/01_exploration_initiale.py
```

The notebook can be opened in VS Code using the Jupyter extension.

## Engineered Variables

The data cleaning pipeline prepares the following features:

- Trip duration in minutes;
- Pickup hour, day of the week, and month;
- Weekend indicator;
- Average speed in miles per hour;
- Distance converted to kilometers for dependency analysis with `fare_amount`;
- Hourly median compared separately for weekdays and weekends.

The cleaning process filters out trips with dates outside of 2018, unknown vendor IDs (valid `VendorID` values for the 2018 TLC source are 1, 2, and 4), passenger counts outside the 1–6 range, non-positive distances, non-positive fares or total amounts, negative charge components, and trip durations outside the 1–180 minute interval. Rate codes, payment types, zones, and `Y/N` indicator values are validated against the official categories present in the dataset. Discrepancies between `total_amount` and the sum of its individual components are flagged but not used independently to delete records, as some fee components may not be fully represented in the sample.

**Source Verification Note.** Validations were executed using the official 2018 TLC/Socrata metadata along with the categories actually observed in the sample. These controls verify date formatting, the calendar year, `VendorID` codes, zones, payment categories, distances, durations, and monetary amounts. Outliers are thus distinguished from definitive data errors without applying arbitrary corrections.

**Methodological Support.** The sample is drawn proportionally within the `month × day of week × hour` strata, preserving the temporal framework of the comprehensive source dataset. The selection of `random_state=42` holds no specific scientific meaning; it merely anchors the random seed so that the exact same sample can be reproduced. Hours are classified into `Low`, `Medium`, and `High` density levels separately for weekdays and weekends based on the quantiles of hourly trip volume. The medians of descriptive variables are then calculated within each group to mitigate the influence of skewed distributions and extreme outliers.

The core mathematical formulations are:

$$
n_h \approx n \times \frac{N_h}{N}, \qquad \sum_h n_h = n
$$

where $N_h$ represents the number of trips within the `month × day × hour` stratum, $N$ is the total number of trips in 2018, $n=100,000$ is the target sample size, and $n_h$ is its assigned quota. Integer quotas are obtained via truncation, and any remaining units are allocated to the strata displaying the largest fractional remainders to guarantee the sum equals exactly $n$.

The hourly density used to categorize time slots is defined as:

$$
D_h = \frac{C_h}{\Delta t}
$$

where $C_h$ is the number of trips during hour $h$ and $\Delta t=1$ hour. The `Low`, `Medium`, and `High` tiers represent the three quantile groups of $D_h$, computed separately for weekdays and weekends. For any given variable $X$, the value displayed in group $g$ is formulated as:

$$
\widetilde{X}_g = \text{median}\{X_h : h \in g\}
$$


For `distance_km` and `duration_minutes`, $X_h$ denotes the mean of the trips observed at hour $h$. For `fare_amount`, $X_h$ represents the hourly median. The resulting visualization plots the median of these hourly metrics $X_h$ within each density group $g$. This distinction is critical: the time slots are defined by trip volume density, whereas the remaining variables serve as descriptive indicators mapped to those slots.

This approach is grounded in academic literature addressing these specific statistical operations:

- Malec, D. (2012). *Stratified Sampling, Allocation in*.  [Wiley Statistics](https://www.wiley.com/en-gb/): This reference covers stratified sampling and allocation across strata.

- Oosterhoff, J. (1994). *Trimmed mean or sample median?* Direct Article Link: [ScienceDirect](https://www.sciencedirect.com/science/article/pii/0167715294901325). This study directly evaluates trimmed means versus medians as robust measures of central tendency.
- Kim, T. H. (1992). *The Metrically Trimmed Mean as a Robust Estimator of Location*. Direct Article Link: [Project Euclid](https://projecteuclid.org/journals/annals-of-statistics/volume-20/issue-3/The-Metrically-Trimmed-Mean-as-a-Robust-Estimator-of-Location/10.1214/aos/1176348783.full). This paper provides the rationale for utilizing a trimmed mean as a robust location estimator.


For the hourly plots, no hours are filtered, and sample sizes are omitted from the figure. The median was selected as the representative metric for typical hourly fares due to its robustness against extreme values, yielding a more continuous trendline than the mean across this heavily skewed distribution. This design choice aligns with established frameworks on robust location estimators and asymmetric distributions.

**Distance-Fare Visualization Layout.** The plot `fare_amount_distribution.png` utilizes a dual-axis layout to compare trip counts and fare metrics simultaneously without confounding their respective units:

- The horizontal axis displays distance brackets (`0-1 km`, `1-2 km`, etc.);
- Blue bars represent the trip count within each bracket;
- Orange bars indicate the standard mean fare in USD for each bracket;
- The right vertical axis is denominated in USD and scales both fare trendlines;
- The red curve maps the median fare;
- The green curve maps the 10% trimmed mean fare.

This visualization choice allows concurrent evaluation of the observation distribution and the evolution of fares relative to distance. The distance brackets are retained on the horizontal axis rather than plotting a median distance per bracket, as the latter would be redundant with the variable defining the brackets themselves. The median and trimmed mean are presented alongside one another to contrast a fully robust location estimator with a mean that is less sensitive to extreme outliers. While the dual-axis format streamlines the readability of variables with conflicting scales, it should not be interpreted as definitive evidence of causality; volume and monetary values remain independently described by their respective axes and legends.

This structure builds upon Oosterhoff's comparison of trimmed means and medians, Kim's research on robust estimation, and taxi fare prediction studies that incorporate distance as a core explanatory variable (Huang, 2023). Full references are cited below. None of these foundational works leverage a dual-axis layout to imply causality; here, it serves exclusively to overlay volume distributions and monetary amounts across different metrics.

## Academic References

- Cleveland, W. S., & McGill, R. (1984). *Graphical Perception: Theory, Experimentation, and Application to the Development of Graphical Methods*. Journal of the American Statistical Association, 79(387), 531-554. Direct Article Link: [JSTOR Stable Archive](https://www.jstor.org/stable/2288400). This reference supports the rigorous selection of graphical encodings to maximize visual clarity.
- Oosterhoff, J. (1994). *Trimmed mean or sample median?* Statistics & Probability Letters, 20(1), 77-79. Direct Article Link: [ScienceDirect](https://www.sciencedirect.com/science/article/pii/0167715294901325). This study focuses directly on comparing the properties of trimmed means and sample medians.
- Kim, T. H. (1992). *The Metrically Trimmed Mean as a Robust Estimator of Location*. Annals of Statistics, 20(3), 1524-1531. Direct Article Link: [Project Euclid](https://projecteuclid.org/journals/annals-of-statistics/volume-20/issue-3/The-Metrically-Trimmed-Mean-as-a-Robust-Estimator-of-Location/10.1214/aos/1176348783.full). This paper addresses the application of the metrically trimmed mean as a robust estimator.
- Huang, H. (2023). *Taxi fare prediction based on multiple machine learning models*. Applied and Computational Engineering, 16, 123-129. Direct Publication Link: [ResearchGate Preprint](https://www.researchgate.net/publication/382370761_Taxi_fare_prediction_based_on_multiple_machine_learning_models). This study utilizes distance as a primary predictive feature for fare calculation and contrasts various regression models.

These references serve to document the specific analytical and statistical steps implemented in this workflow. They do not imply a validation of final predictive accuracy; target datasets, fare definitions, and cross-validation protocols must be rigorously calibrated before conducting any model comparisons.
