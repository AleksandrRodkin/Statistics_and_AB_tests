# A/B Test Analysis with Ratio Metrics
When running A/B tests we often deal with metrics that are expressed as a ratio of two quantities - for example, 
the average session duration (total time / number of sessions) or CTR (clicks / views).

Such metrics are called ratio metrics, and they often cause trouble because the usual formulas for confidence 
intervals and p-values don’t behave correctly in this case.

In this notebook I’ll walk through several approaches that allow to estimate the effect for ratio metrics properly.

## Project Overview

This project analyzes the change in average session duration between control and test groups.
It focuses on ratio metrics - I explore several methods that provide more reliable and interpretable estimates of 
the experimental effect.

## Goals

- Show why standard mean comparison methods can produce biased results for ratio metrics.
- Apply and compare several statistical techniques that are used when working with ratio metrics:
  - Bootstrap
  - Bucket Method
  - Linearization
  - Linearization with CUPED (Controlled Pre-Experiment Data)


## 1. Data Generation

Used synthetic data, that contains: 
- Historical data — collected before the experiment
- Experimental data — collected during a 2-week test:
  - Group A (test group): includes new changes
  - Group B (control group): no changes applied

Each record represents a session of an independent user.

## 2. Demonstrating Incorrect Approaches

Before applying advanced statistical methods, I first illustrate why naive analysis can lead to false conclusions.
A common mistake in A/B testing is to treat individual sessions as independent observations.
However, sessions belonging to the same user are not independent.

As a result, performing statistical tests directly on session-level data leads to inflated Type I error rates 
(false positives), and incorrect inferences about the experimental effect.

## 3. Performing Correct A/B Tests
    Note:
    Before each A/B test, a corresponding A/A test is conducted to verify that the chosen method produces 
a correct false positive rate (FPR)

## 3.1 Bootstrap

Bootstrap empirically estimates the sampling distribution of a statistic by repeatedly resampling the data 
with replacement.

Procedure:
- Resample users (not sessions) many times with replacement
- For each sample, calculate the average session duration
- Compute the distribution of the difference in means between groups
- Derive the confidence interval and p-value


Advantages:
- This method makes no assumptions about the underlying distribution of the data
- Provides an interpretable empirical distribution of the metric

Limitations:
- Computationally expensive, especially on large datasets
- Requires many iterations (tens of thousands) for stable results
- Does not allow the use of dispersion reduction methods

## 3.2 Bucket Method

The bucket method improves computational efficiency by aggregating users into randomly assigned buckets 
(for example, using a hash of user_id).
All user sessions within a bucket are combined, and the metric is computed per bucket rather than per user.

Procedure:
- Randomly assign users to a fixed number of buckets.
- Aggregate all sessions of users within each bucket.
- Perform statistical comparison of bucket-level averages between groups.

Advantages:
- Significantly faster and more scalable than bootstrap
- Enables online or streaming analysis — bucket weights can be updated on the fly
- Preserves independence between buckets, since each user belongs to exactly one bucket

Limitations:
- Sensitive to uneven bucket sizes if hashing is poorly implemented
- Does not allow the use of dispersion reduction methods
 
## 3.3 Linearization

Linearization converts a complex ratio metric into a linear form that preserves its meaning but reduces variance, 
making it suitable for standard statistical tests

Formula:

$$Lin_u = total\\_duration_u - \kappa * session\\_count_u$$

where:
- $total\\_duration_u$ - total time spent across all sessions of user u,
- $\kappa$ - a coefficient chosen to minimize the variance of the new metric (typically estimated as the target 
- ratio metric computed on the control group),
- $session\\_count_u$ - number of sessions for user u.

The coefficient $\kappa$ is estimated using the control group and then applied to both groups.

Procedure:

- Compute $\kappa$ on the control group as the average ratio $total\\_duration_u / session\\_count_u$
- Apply this transformation to both groups.
- Compare mean $Lin_u$ values between groups using standard t-tests or nonparametric tests.

Advantages:
- Works at the user level, enable using of dispersion reduction methods
- Efficient and scalable for large data

Limitations:
- Assumes the relationship between numerator and denominator is approximately linear.

### 3.4 Linearization with CUPED

CUPED (Controlled Experiment Using Pre-Experiment Data) is a variance reduction technique that leverages historical 
(pre-experiment) user data.
It adjusts the target metric by removing the portion of its variance explained by correlated pre-experiment features.

Formula:

$$Y_{cuped}=Y−\theta(X−\overline{X})$$

where:
- Y - outcome measured during the experiment
- X - pre-experiment data (a covariate)
- $\theta$ - adjustment coefficient:
$$\theta = cov(Y, X) / var(X)$$


Procedure:
- Compute correlation between pre-experiment and in-experiment metrics.
- Estimate $\theta$
- Apply the CUPED adjustment to obtain a new CUPED-metric
- Perform statistical testing on the adjusted metric.

Advantages:
- Reduces variance, improving statistical power.
- Enhances precision of the estimated effect without increasing sample size.
- Works well with linearized metrics and user-level aggregation.

Limitations:
- Requires reliable pre-experiment data
- Effectiveness depends on strong correlation between linearized metric and the covariate


## Results
- All tests showed a p-value less than the significance level, which allows to reject the null hypothesis of equality 
of means
- The average session duration in Group A is significantly higher than in Group B
- With using of CUPED p-value decreased by 2–3× compared to the baseline methods



## Conclusions

- Bootstrap — easy to implement and statistically valid, but computationally heavy.
- Bucket method — much faster and scalable, suitable for streaming or online analysis, but doesn’t support 
- variance reduction.
- Linearization — converts ratio metrics into a form compatible with standard statistical tests.
- CUPED — effectively reduces variance, improving test sensitivity without increasing sample size.
