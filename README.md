# Beta Calculator

A bottom-up (peer-group-based) beta estimation tool that derives a target
company's relevered beta from the equity returns of comparable industry
peers, adjusting for leverage and tax effects, then re-levering the result
to the target company's own capital structure.

This methodology is a widely used technique in corporate valuation (DCF,
cost of equity / WACC estimation). It is typically applied when a target
company's own historical return series is too short, too noisy, or when its
capital structure has recently changed significantly — making a direct
regression beta on the company's own stock unreliable.

---

## Calculation pipeline

### 1. Data collection

Weekly (or otherwise configurable frequency) closing prices are downloaded
for the `peer_group` and the `benchmark` from Yahoo Finance (`yfinance`)
over the specified `start_date`–`end_date` window.

### 2. Return calculation

Returns are computed from the closing prices, either linear or logarithmic,
depending on the `return_calc` setting:

- **Linear return:**

  $$r_t = \frac{P_t}{P_{t-1}} - 1$$

- **Logarithmic return:**

  $$r_t = \ln\left(\frac{P_t}{P_{t-1}}\right)$$

### 3. Raw beta estimation via OLS regression

For each peer, a univariate linear regression is run between that company's
return (dependent variable) and the benchmark return (explanatory variable):

$$r_{i,t} = \alpha_i + \beta_i \cdot r_{m,t} + \varepsilon_{i,t}$$

where $\beta_i$ is the peer's raw (levered) market beta and $r_{m,t}$ is the
benchmark return. 
The regression is run using `statsmodels`' OLS
implementation, and alongside the beta, its standard error is also stored —
this is needed later for the Vasicek adjustment.

### 4. Beta adjustment (optional smoothing)

Raw betas can be distorted by sampling noise, so the model offers two
optional smoothing methods, controlled by the `beta_adjustment` parameter:

- **Blume adjustment** — pulls the beta toward the market average of 1.0,
  based on the empirical tendency of betas to mean-revert toward 1.0 over
  time:

  $$\beta_{Blume} = \frac{2}{3}\beta_{raw} + \frac{1}{3}\cdot 1$$

- **Vasicek adjustment** — a Bayesian-weighted smoothing method that pulls
  each company's beta toward the cross-sectional (peer group) mean, weighted
  by the reliability of its own estimate (its standard error). The more
  reliable (lower standard error) a company's beta is, the less it gets
  pulled toward the group average:

  $$w_i = \frac{\sigma^2_{\beta}}{\sigma^2_{\beta} + SE_i^2}, \qquad
  \beta_{Vasicek,i} = w_i \cdot \beta_{raw,i} + (1-w_i)\cdot \bar{\beta}$$

  where $\sigma^2_{\beta}$ is the cross-sectional variance of the peer
  group's raw betas, and $SE_i$ is the regression standard error of company
  $i$'s beta.

- **"none"** — no adjustment; the raw beta is passed through to the next
  step unchanged.

### 5. Unlevering — the Hamada equation

For each peer, the D/E ratio (`Total Debt` / `Stockholders Equity`, taken
from the most recent balance sheet date closest to `end_date`) and the
domestic statutory corporate tax rate (looked up from the OECD database,
based on the company's `country` field) are computed. These are used to
unlever the (adjusted) beta via the Hamada formula:

$$\beta_{unlevered} = \frac{\beta_{levered}}{1 + (1 - t) \cdot \frac{D}{E}}$$

where $t$ is the statutory tax rate and $D/E$ is the company's
debt-to-equity ratio.

The unlevered beta reflects a company's *business* (operating) risk,
independent of its capital structure — this is what makes peers with
different leverage levels comparable to one another.

### 6. Aggregating the peer group beta

The unlevered betas of the peer group are aggregated into a single
representative value, according to the `peer_group_beta_method` parameter:

$$\beta_{peer} = \text{mean}(\beta_{unlevered,i}) \quad \text{or} \quad
\text{median}(\beta_{unlevered,i})$$

The median is more robust to outlier peers.

### 7. Re-levering to the target company

The aggregated peer beta is re-levered using the target company's own D/E
ratio and its own statutory tax rate, producing the target's estimated
levered ("relevered") beta:

$$\beta_{target} = \beta_{peer} \cdot \left(1 + (1 - t_{target}) \cdot
\frac{D_{target}}{E_{target}}\right)$$

`target_relevered_beta` is the model's final output, which can then be used,
e.g. via CAPM, to estimate the target's cost of equity as part of a WACC
calculation.

---

## Input parameters

| Parameter | Description |
|---|---|
| `target_ticker` | Ticker of the target company |
| `benchmark` | Ticker of the index used as a proxy for the market return |
| `peer_group` | Tickers of the comparable companies |
| `start_date` / `end_date` | The analysis period |
| `interval` | Frequency of the price data (e.g. `1wk`) |
| `return_calc` | `linear` or `log` return calculation |
| `beta_adjustment` | `none`, `blume`, or `vasicek` |
| `peer_group_beta_method` | `average` or `median` |

---

## Data sources

- **Price and balance sheet data:** Yahoo Finance (`yfinance` package)
- **Statutory corporate tax rates:** OECD statistical database
  (`data/OECD_statutory_tax_rates/oecd_rates.xlsx`)
- **Country code resolution:** `pycountry` package (ISO alpha-3 codes)

---

## Future development

- **Support for private (non-listed) companies** — the model currently
  sources every input (price data, balance sheet data, country) from Yahoo
  Finance, which assumes both the target and the peer group are publicly
  traded. The goal is to allow the re-levering step to run on manually
  supplied parameters (e.g. an estimated D/E ratio, a known domicile / tax
  rate) instead of a `target_ticker`, so the tool can also be used to value
  private companies.
- **+ Data visualization** 
