# ovbwr\_capital\_impinfo — AR(1) Productivity Process, Services Sector

> This repository estimates a first-order autoregressive (AR(1)) process for cyclical log
> labor productivity in the **services sector** for up to 31 countries, using two chained
> historical datasets. The outputs — parameter estimates, a LaTeX table, and diagnostic
> figures — are intended to calibrate or validate a structural macroeconomic model.
> A replicator with access to the two raw data files and a standard Python environment
> can reproduce every output by running one notebook.

---

## Table of Contents

- [ovbwr\_capital\_impinfo — AR(1) Productivity Process, Services Sector](#ovbwr_capital_impinfo--ar1-productivity-process-services-sector)
  - [Table of Contents](#table-of-contents)
  - [1. Folder Structure](#1-folder-structure)
  - [2. Execution Order](#2-execution-order)
  - [3. Notebook Description](#3-notebook-description)
    - [`all_countries_prod.ipynb`](#all_countries_prodipynb)
  - [4. Input Data](#4-input-data)
  - [5. Outputs](#5-outputs)
    - [Processed datasets](#processed-datasets)
    - [LaTeX tables](#latex-tables)
    - [JSON parameters](#json-parameters)
    - [Figures](#figures)
  - [6. Software Requirements](#6-software-requirements)
  - [7. Methodological Notes](#7-methodological-notes)
    - [7.1 Labor Productivity](#71-labor-productivity)
    - [7.2 Hamilton Filter](#72-hamilton-filter)
    - [7.3 AR(1) Process](#73-ar1-process)

---

## 1. Folder Structure

```text
ovbwr_capital_impinfo/
├── code/
│   └── empirical/
│       └── all_countries_prod.ipynb   Main analysis notebook (only file to run)
├── data/
│   ├── raw/
│   │   ├── 10sd_jan15_2014.xlsx       10-Sector Dataset (Timmer et al., 2005 USD)
│   │   └── ETD_230918.xlsx            Economic Transformation Dataset (2015 USD)
│   └── processed/                     (empty — no intermediate files produced)
├── output/
│   ├── figures/                       PDF figures produced by the notebook
│   └── tables/                        LaTeX table, JSON parameters, panel Excel file
├── paper/                             (manuscript folder, not part of replication)
└── readme.md                          This file
```

---

## 2. Execution Order

There is only one notebook. No preprocessing step is required.

```text
Step 1  code/empirical/all_countries_prod.ipynb
        reads  data/raw/10sd_jan15_2014.xlsx
               data/raw/ETD_230918.xlsx
        writes output/figures/all_prod_growth_{1..6}.pdf
               output/figures/all_productivity_{1..6}.pdf
               output/figures/all_productivity_detrended_{1..6}.pdf
               output/figures/all_ar1_diagnostics_hamilton.pdf
               output/figures/all_ar1_diagnostics_hp.pdf
               output/tables/all_ar1_productivity.tex
               output/tables/all_prod_ar1_params.json
               output/tables/all_prod_data.xlsx
```

The notebook must be run from the `code/empirical/` working directory (the default when
Jupyter opens the file). All file paths inside the notebook use `../../` to navigate to
`data/` and `output/` relative to that location.

---

## 3. Notebook Description

### `all_countries_prod.ipynb`

Estimates AR(1) processes for log labor productivity in services for every country that
appears in both raw datasets. The notebook performs data loading, country discovery,
sector aggregation, series chaining, detrending, AR(1) estimation, and export in a single
sequential run. No manual intervention is needed between cells.

**Services sector definition.** Services is defined as all sectors that are not Agriculture,
Mining, or Manufacturing. The exact columns summed from each dataset are:

| Dataset   | Columns summed                                                                                                                  |
| --------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 10-Sector | Utilities, Construction, Trade restaurants & hotels, Transport storage & communication, Government services, Community services |
| ETD       | Utilities, Construction, Trade services, Transport services, Government services, Other services                                |

**Data splicing logic.** The 10-Sector series (valued at constant 2005 USD) is the primary
source. For each country, the splice point is the last year with a non-missing 10-Sector
observation. From the following year through 2018, the series is extended using the ETD
annual growth rate of productivity: $a_t = a_{t-1} \cdot (1 + g_t^{\text{ETD}})$, where
$g_t^{\text{ETD}}$ is the ETD year-on-year growth rate. This preserves the 2005 USD
level base while borrowing ETD dynamics for the more recent period.

Missing values in the 10-Sector dataset that appear as annotation strings (e.g.,
`"incl in gov. services"`) are coerced to `NaN` before aggregation; `min_count=1` is
used in the row sum so that a fully missing observation remains `NaN` rather than `0`.

**Country filtering.** After chaining, any country with fewer than $h + p + 2 = 8$ valid
positive observations is excluded from the entire notebook. A second guard removes any
country for which the Hamilton filter produces no valid cycle values.

**Key methods.** See Section 7 for full mathematical specifications.

- Hamilton (2018) filter with $h=2$, $p=4$ — primary detrending method.
- Hodrick-Prescott filter with $\lambda=100$ — shown for comparison only.
- Per-country AR(1): conditional MLE via `statsmodels.AutoReg`.
- Pooled AR(1): OLS on stacked within-country lag pairs with HC1-robust standard errors; variance estimated as MLE (RSS$/N$).

**Key outputs.**

| File                                                   | Content                                                                                                                                          |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `output/tables/all_ar1_productivity.tex`               | LaTeX table of $\hat\rho$, se$(\hat\rho)$, $\hat\sigma_\varepsilon$, $\hat\sigma^2_\varepsilon$ for every country (both filters) plus pooled row |
| `output/tables/all_prod_ar1_params.json`               | Machine-readable AR(1) parameters: pooled and per-country, for both filters                                                                      |
| `output/tables/all_prod_data.xlsx`                     | Panel dataset with spliced productivity level and both detrended cycles per country                                                              |
| `output/figures/all_prod_growth_{1..6}.pdf`            | Six panels comparing annual productivity growth rates from the two source datasets                                                               |
| `output/figures/all_productivity_{1..6}.pdf`           | Six panels of spliced productivity levels in 2005 USD per worker                                                                                 |
| `output/figures/all_productivity_detrended_{1..6}.pdf` | Six panels of Hamilton and HP cycles overlaid per country                                                                                        |
| `output/figures/all_ar1_diagnostics_hamilton.pdf`      | Pooled AR(1) residual diagnostics for Hamilton filter (residual plot, ACF, PACF)                                                                 |
| `output/figures/all_ar1_diagnostics_hp.pdf`            | Pooled AR(1) residual diagnostics for HP filter (residual plot, ACF, PACF)                                                                       |

---

## 4. Input Data

| File                   | Variable | Description                                    | Source                                            | Period     | Coverage           |
| ---------------------- | -------- | ---------------------------------------------- | ------------------------------------------------- | ---------- | ------------------ |
| `10sd_jan15_2014.xlsx` | `VA_Q05` | Real value added, constant 2005 USD (millions) | 10-Sector Database (Timmer et al., GGDC)          | ~1950–2011 | Up to 42 countries |
| `10sd_jan15_2014.xlsx` | `EMP`    | Employment (thousands of persons)              | 10-Sector Database (Timmer et al., GGDC)          | ~1950–2011 | Up to 42 countries |
| `ETD_230918.xlsx`      | `VA_Q15` | Real value added, constant 2015 USD (millions) | Economic Transformation Database (GGDC/UNU-WIDER) | 1990–2018  | Up to 51 countries |
| `ETD_230918.xlsx`      | `EMP`    | Employment (thousands of persons)              | Economic Transformation Database (GGDC/UNU-WIDER) | 1990–2018  | Up to 51 countries |

Both files must be placed in `data/raw/` exactly as named. The notebook reads the sheet
named `"dataset"` from `10sd_jan15_2014.xlsx` and the sheet named `"Data"` from
`ETD_230918.xlsx`.

Labor productivity for each country-year is computed as:

$$a_{i,t} = \frac{\text{VA}_{i,t}}{\text{EMP}_{i,t}}$$

where both numerator and denominator are the sum across the services-sector columns
listed in Section 3. Zero employment values are replaced with `NaN` before division.

---

## 5. Outputs

### Processed datasets

| File                               | Produced by                | Content                                                                         |
| ---------------------------------- | -------------------------- | ------------------------------------------------------------------------------- |
| `output/tables/all_prod_data.xlsx` | `all_countries_prod.ipynb` | Country, Year, spliced productivity (2005 USD/worker), Hamilton cycle, HP cycle |

### LaTeX tables

| File                                     | Produced by                | Content                                                                                                                           |
| ---------------------------------------- | -------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `output/tables/all_ar1_productivity.tex` | `all_countries_prod.ipynb` | AR(1) estimates ($\hat\rho$, se, $\hat\sigma_\varepsilon$, $\hat\sigma^2_\varepsilon$) for all countries and pooled, both filters |

### JSON parameters

| File                                     | Produced by                | Content                                                                                                                                                           |
| ---------------------------------------- | -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `output/tables/all_prod_ar1_params.json` | `all_countries_prod.ipynb` | Nested dict: `all_countries → {hamilton, hp} → {pooled, by_country}` with `rho`, `se_rho`, `sigma`, `var`, `intercept` (pooled only), `yr_min`, `yr_max`, `n_obs` |

### Figures

All figures are saved as PDF at 300 dpi.

| File                                                   | Content                                                              |
| ------------------------------------------------------ | -------------------------------------------------------------------- |
| `output/figures/all_prod_growth_{1..6}.pdf`            | Annual productivity growth rates: 10-Sector (solid) vs. ETD (dashed) |
| `output/figures/all_productivity_{1..6}.pdf`           | Spliced productivity levels in 2005 USD per worker                   |
| `output/figures/all_productivity_detrended_{1..6}.pdf` | Hamilton cycle (solid) and HP cycle (dashed) per country             |
| `output/figures/all_ar1_diagnostics_hamilton.pdf`      | Pooled residuals, ACF, PACF — Hamilton filter                        |
| `output/figures/all_ar1_diagnostics_hp.pdf`            | Pooled residuals, ACF, PACF — HP filter                              |

The six-panel figure sets display six countries each (`CHUNK_SIZE = 6`), in the alphabetical
ISO3 order of the final sample.

---

## 6. Software Requirements

| Package       | Tested version | Role                                                                      |
| ------------- | -------------- | ------------------------------------------------------------------------- |
| `python`      | 3.10+          | Runtime                                                                   |
| `pandas`      | —              | Data loading, merging, panel construction                                 |
| `numpy`       | —              | Numerical operations, array handling                                      |
| `matplotlib`  | —              | Figure production                                                         |
| `statsmodels` | —              | `AutoReg` (per-country AR(1)), `OLS` (pooled), `hpfilter`, ACF/PACF plots |
| `scipy`       | —              | Sparse matrix efficiency (warning suppression only)                       |
| `openpyxl`    | —              | Writing `.xlsx` output via `pandas.to_excel`                              |
| `jupyter`     | —              | Notebook runtime                                                          |

Install all dependencies with:

```bash
pip install pandas numpy matplotlib statsmodels scipy openpyxl jupyter
```

**LaTeX requirement.** The notebook sets `text.usetex = True` and uses the Computer
Modern font family with `\usepackage{amsmath}`. A working LaTeX installation (e.g.,
TeX Live or MacTeX) must be present on the system PATH for figures to render. If LaTeX
is not available, disable it before running by changing the `rcParams` block in cell 2:

```python
plt.rcParams.update({
    'text.usetex': False,
    # remove 'font.family', 'font.serif', 'text.latex.preamble' entries
})
```

---

## 7. Methodological Notes

### 7.1 Labor Productivity

Labor productivity in services for country $i$ in year $t$ is:

$$a_{i,t} = \frac{\sum_{s \in \mathcal{S}} \text{VA}_{i,s,t}}{\sum_{s \in \mathcal{S}} \text{EMP}_{i,s,t}}$$

where $\mathcal{S}$ is the set of services subsectors (see Section 3), value added is in
millions of USD (2005 base), and employment is in thousands of persons.

### 7.2 Hamilton Filter

The Hamilton (2018) filter extracts the business-cycle component of $\log a_{i,t}$ by
running, for each country separately, the OLS regression:

$$\log a_{i,t} = \alpha_0 + \sum_{j=0}^{p-1} \alpha_{j+1} \log a_{i,t-h-j} + u_{i,t}$$

The cycle estimate is the residual $\hat{z}_{i,t} = \log a_{i,t} - \hat{\log a}_{i,t}$.
The first $h + p - 1 = 5$ observations per country are used as regressors and receive
`NaN` for the cycle (no endpoint bias).

Parameter choices: $h = 2$ (forecast horizon) and $p = 4$ (number of lags), following
Hamilton's (2018) recommendation for annual data.

> Hamilton, J.D. (2018). Why You Should Never Use the Hodrick-Prescott Filter.
> *Review of Economics and Statistics*, 100(5), 831–843.

### 7.3 AR(1) Process

The cyclical component follows the first-order autoregressive model:

$$z_{i,t} = \rho_i \, z_{i,t-1} + \varepsilon_{i,t}, \qquad \varepsilon_{i,t} \sim \mathcal{N}(0, \sigma^2_{\varepsilon,i})$$

**Per-country estimation.** Parameters $(\hat\rho_i, \hat\sigma_{\varepsilon,i})$ are
estimated by conditional MLE using `statsmodels.tsa.ar_model.AutoReg` with one lag. The
variance $\hat\sigma^2_{\varepsilon,i}$ equals `fit.sigma2`, which is the MLE estimator
RSS$/N$ (not the unbiased OLS estimator RSS$/N-k$).

**Pooled estimation.** All within-country consecutive lag pairs $(z_{i,t}, z_{i,t-1})$
are stacked and a single OLS regression is estimated:

$$z_{i,t} = \alpha + \rho \, z_{i,t-1} + \varepsilon_{i,t}$$

Standard errors on $\hat\rho$ are HC1-robust to account for heteroskedasticity across
countries. The pooled variance $\hat\sigma^2_\varepsilon$ is computed as MLE (RSS$/N$
over all pooled pairs).

**Reported quantities.** Both the JSON export and the LaTeX table report $\hat\rho$,
se$(\hat\rho)$, $\hat\sigma_\varepsilon = \sqrt{\hat\sigma^2_\varepsilon}$, and
$\hat\sigma^2_\varepsilon$. The pooled JSON entry additionally includes the OLS intercept
$\hat\alpha$.
