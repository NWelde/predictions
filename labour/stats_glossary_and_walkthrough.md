# Jobless claims forecast: glossary and walkthrough

Companion to `inital_jobless_claim_nov14.ipynb`. The glossary is meant to grow; add terms as they come up.

## The question

Kalshi asks: will initial jobless claims for the week ending Nov 14 be **at least 200,000**?
The last reported value (week ending Nov 7) was **200,000**.

---

## Glossary

**Initial jobless claims (ICSA)**: number of people filing for unemployment insurance for the first time in a week. FRED's series is *seasonally adjusted*, meaning the government already tried to remove predictable calendar effects.

**Level**: the raw number itself (200,000).

**Change (week-over-week change)**: this week's level minus last week's. Example: 200,000 - 191,000 = +9,000. This is the quantity we model, because we want to know how far the number *moves* in one week.

**Log change**: `ln(this week) - ln(last week)`. Roughly the percent change (0.04 is about 4%). Used for the seasonal sample because a 20,000 move means very different things at a level of 500,000 (1970s) vs 200,000 (now).

**Sample**: the handful of past observations we use as evidence. Method A's sample is the last 52 weekly changes. Method B's sample is the change into the same week of November in each past year.

**Mean (average, mu in this notebook when used as a forecast centre)**: sum divided by count. The "typical" value.

**Variance / standard deviation (SD, sigma)**: the SD is the typical distance of a data point from the mean. Compute: take each value's gap from the mean, square it, average the squares (dividing by n-1 for a sample, see *ddof*), then square-root. Bigger SD = more spread = more uncertainty. Sigma = 9,327 means "a normal week moves roughly 9 thousand either way".

**ddof=1 / n-1**: when you estimate the SD from a sample (not the whole population), dividing by n-1 instead of n corrects for the fact that the sample mean is fitted to the same data, which makes the spread look slightly too small.

**Distribution**: the shape describing how often each possible value occurs.

**Histogram**: a bar chart of a sample. The values are sliced into bins, and each bar's height shows how many fall in that bin. With `density=True` the bars are rescaled so total area = 1, which lets you lay a probability curve over them.

**Normal distribution (Gaussian, bell curve)**: a distribution fully described by two numbers: mean and SD. It's symmetric, most values sit near the mean, and extreme values are rare. Rule of thumb: 68% of values within 1 SD, 95% within 2, 99.7% within 3.

**Probability density function (pdf)**: the *height* of the bell curve at each value. A height isn't a probability. The **area under the curve between two values is the probability** of landing between them. The total area is 1. For the normal: `pdf(x) = 1/(sigma*sqrt(2*pi)) * exp(-(x-mu)^2 / (2*sigma^2))`.

**Cumulative distribution function (cdf)**: the area under the pdf from -infinity up to x, i.e. P(X <= x). So P(X >= x) = 1 - cdf(x). Integrating the pdf and using the cdf give the same answer; the cdf is just the integral already worked out.

**z-score**: how many SDs a value is from the mean: `(x - mu) / sigma`. Converts any normal to the standard one (mean 0, SD 1), which is how tables and `norm.cdf` work.

**Continuity correction**: claims are whole numbers but the normal curve is continuous. "At least 200,000" covers everything from 199,500 up when you round, so we integrate from 199,500, not 200,000.

**Skewness**: lopsidedness. 0 = symmetric. Negative = longer tail on the left.

**Kurtosis (excess)**: tail heaviness vs a normal. 0 = normal. Positive = fat tails (extreme weeks more common than a bell curve says). This is the one that matters most for a threshold bet, because fat tails mean the normal would understate the chance of big moves.

**Q-Q plot**: plots your sorted data against where a normal's values would sit. Points hugging the straight line = looks normal. Ends curling away = tails differ.

**Shapiro-Wilk test / p-value**: a formal normality test. Null hypothesis: "the data are normal". A p-value is the chance of seeing data at least this non-normal *if* it really were normal. Small (under 0.05) = evidence against normality. A large p-value does **not prove** normality; with ~50 points the test can't detect modest departures.

**Drift**: a consistent average direction of change. Here: does the series tend to fall a little in this week of November? Mean of the seasonal changes = -0.8%.

**Robust SD (MAD-based)**: SD estimate that one outlier can't inflate much. If it disagrees a lot with the regular SD, outliers are driving the spread.

---

## Step by step

### Step 1. Treat next week as "this week plus a random change"

```
next_week = last_level + change
```

We know `last_level` (200,000). `change` is unknown, so we model it as a random draw from a normal distribution with some mean and SD. Everything after this is about choosing those two numbers and then reading off an area.

### Step 2. Estimate the spread (sigma) from data

- Method A: SD of the last 52 weekly changes = **9,327**.
- Method B: SD of the log change into the Nov 11-17 week across past years = **0.0399**, times today's level of 200,000 = **7,977**.

Different samples, different noise estimates. That's the whole difference between the methods.

### Step 3. Check the bell-curve assumption

We're about to claim the changes follow a normal. The histogram with the pdf over it and the Q-Q plot let us eyeball it; skewness, kurtosis and Shapiro give numbers. Both samples passed: no fat tails, p-values 0.48 and 0.25. Caveat: this means "no evidence against", not "proven".

### Step 4. Choose the centre (mu) of next week's distribution

This is the "start point". Two natural choices:

- **No drift**: centre = last level = 200,000. (Random-walk assumption: the best guess for next week is this week.)
- **With seasonal drift**: centre = 200,000 x exp(-0.0084) = about 198,300.

### Step 5. Turn the distribution into a probability

We want P(claims >= 199,500) under Normal(mu, sigma):

1. z = (199,500 - mu) / sigma
2. P = area under the standard bell curve to the right of z = 1 - cdf(z)

Worked example (mu = 200,000, sigma = 9,327): z = -500 / 9,327 = -0.054. The area to the right of -0.054 is about **52.1%**. The threshold sits almost exactly at the centre of the bell, so about half the area is above it.

Worked example (mu = 198,318, sigma = 7,977): z = (199,500 - 198,318) / 7,977 = +0.148. Area to the right of +0.148 is about **44.1%**. The threshold is now slightly *right* of the centre, so a bit less than half the area is above it.

---

## How the start point changes the probability

"Start point" can mean two things here, and they behave very differently.

### A. The starting level / centre (mu): a strong lever

Because the threshold (200k) is sitting right at the current level, the probability is ultra-sensitive to where you put the centre. The pdf's height at the centre is the exchange rate: each 1,000 claims you shift mu moves the probability by about **4 to 6 percentage points** near 200k.

P(claims >= 200,000), by centre and sigma:

| mu | sigma 7,000 | sigma 7,977 | sigma 9,327 |
|---:|---:|---:|---:|
| 194,000 | 21.6% | 24.5% | 27.8% |
| 196,000 | 30.9% | 33.0% | 35.4% |
| 198,000 | 41.5% | 42.5% | 43.6% |
| **200,000** | **52.8%** | **52.5%** | **52.1%** |
| 202,000 | 64.0% | 62.3% | 60.6% |
| 204,000 | 74.0% | 71.4% | 68.5% |

Reading across a row (changing sigma) moves the answer a little. Reading down a column (changing mu) moves it a lot. That's why the drift question (200,000 vs about 198,300) matters more than almost anything else in the notebook: it's the difference between 52% and 44%.

### B. The starting year of the seasonal sample (START_YEAR): a weak lever

This only changes **sigma** (the spread), via which years go into the sample:

| start year | n | SD of log change | sigma at 200k |
|---:|---:|---:|---:|
| 1967 | 57 | 0.0399 | 7,977 |
| 1990 | 34 | 0.0351 | 7,022 |
| 2000 | 24 | 0.0355 | 7,095 |
| 2010 | 14 | 0.0406 | 8,114 |

Why sigma barely matters here: when mu is at the threshold, the threshold is at the *middle* of the bell, and the middle splits 50/50 no matter how wide or narrow the bell is. Widening the bell only changes the chance by shifting a small sliver of area. When mu is *away* from the threshold, sigma matters more (see the 204,000 row: 74% vs 68.5%), because a narrower bell concentrates more probability on the side mu is on.

Also: fewer years means a noisier estimate of sigma itself (n=14 is thin), and the SD wobbles up and down with the era (the 2010+ number is bigger than the 1990+ one). That wobble is estimation noise, not a real change in the world.

### Bottom line

- Start-year choice: changes sigma by about +/-1,000, which moves the answer by roughly 1 point near mu = 200k.
- Centre choice: changes the answer by about 5 points per 1,000 claims.
- So the decision worth thinking hardest about is *what to believe about next week's centre*, not which years to sample.

## Open caveats

- Normality tests with ~50 points have low power; a "pass" isn't proof.
- The -0.8% seasonal drift is about a fifth of an SD. It is weak evidence and could be noise. With 57 years of data its standard error is about 0.04 / sqrt(57) = 0.5%, so the drift (0.84%) is only about 1.6 standard errors from zero: suggestive, not conclusive.
- The ICSA series is revised after release; the 200,000 "last level" is the revised FRED number and may differ from what Kalshi settles against.
