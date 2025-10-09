---

layout: post
title: "Leveraging Financial Analysis with Google BigQuery and Python: A Financial Big Data Application."
author: "Mario H. Gonzalez-Sauri"
date: "2025-09-10"

---


## 1) Installing Python in RStudio

For this project I will work through the IDE of RStudio given that it is
one of the easieast and fastests ways generate a Markdown document with
snippets of Python. I first installed Python, through the reticulate
package and Conda repositories that will allow to run Python commands
within the R session. The process is simple, first intalling the
reticulate with dependencies, then running the install_miniconda()
command for installing Python. The third step is to accept the
conditions of installing Python in the local machine. And, finally
creating a Python enviroment that will be called (binded) when an
Rsession starts (this happens when I couple the Markdown document).

(If you have already Python installed or you prefer working with
Jupiter, skip to part 2).

``` r
# Install reticulate
install.packages("reticulate", dependencies = T)


# Load Library
library(reticulate)

# Install Python
install_miniconda()

# Accept ToS for all the defaults channels it complained about
system2(conda, c("tos","accept","--override-channels","--channel","https://repo.anaconda.com/pkgs/main"))
system2(conda, c("tos","accept","--override-channels","--channel","https://repo.anaconda.com/pkgs/r"))
system2(conda, c("tos","accept","--override-channels","--channel","https://repo.anaconda.com/pkgs/msys2"))



# Now create the env reticulate wanted
reticulate::conda_create("r-reticulate", packages = c("python=3.10","pip"))

# Install libraries from Needed
reticulate::conda_install("r-reticulate",
                          c("pandas","pyarrow","google-cloud-bigquery",
                            "pandas-gbq","google-cloud-bigquery-storage","db-dtypes"),
                          channel = "conda-forge"
)
```

After the installation is done, is best to start a clean new Rsession
(Ctrl + Alt + F10), and load the environment of Python in the session.
Recall that this snippet will be loaded to bind R with Python within
Rstudio.

``` r
# to bind the env of python to the Rsession is best to restart ctrl + alt + F10
reticulate::use_condaenv("r-reticulate", required = TRUE)

reticulate::py_config()
```

## 2) Setting Up the Cloud Enviroment

For this project I will showcase the use of Google Cloud CLI, that
provide a powerful way to interact with Google BigQuery and Python. The
Google BigQuery has the advantage of using the Cloud Infrastructure from
Google that can run regula SQL and BigQuery if needed. Google Cloud CLI
has also the advantage of monitoring and managing query jobs that run
regularly. Furthermore, the Google BigQuery can directly train, evaluate
and run ML models suitable for prediction and forecast.

The dataset that I will use is public and accessible through Google
BigQuery, called eCommerce, that is rich in tables that contain
inventories, kpis and other financial data typically needed for decision
making.

### 2.1) Install Google Cloud CLI

The first step is install the Google Cloud CLI, I am installing it with
Blunted Python and Beta Commands, and skipping the Cloud Tools for
Power-shell that I do not need at the movement.

![](https://github.com/Wario84/blog/raw/main/assets/imgs/google_cli_1.png?raw=true)<!-- -->

After installation, the Google Shell will open and it will require
authentication with a Google Account.

![](https://github.com/Wario84/blog/raw/main/assets/imgs/google_cli_2.png?raw=true)<!-- -->

After selecting Yes

![](https://github.com/Wario84/blog/raw/main/assets/imgs/google_cli_3.png?raw=true)<!-- -->

Make sure the authentification is correct:

![](https://github.com/Wario84/blog/raw/main/assets/imgs/google_cli_4.png?raw=true)<!-- -->

### 2.2) Creating a Project in Gcloud Power-Shell

After accepting the terms, we can go back to to the Power-Shell and
create a new project (Option 3), give a name to the project, I am
calling my project `finance-bigq-demo-2025-mgs`. I there is a problem,
for instance, non-compatible project id name, you can run , to create
the project.

### 2.3) Authentification and Enabling Big Data Services

After the project it is create, the can proceed to authenticate the user
name in the Gcloud, this is done one time and credentials are stored
locall y- `gcloud auth application-default login`. After running this
command, your default browser will open, and just sign in with your
Google Credentials. Then a final step is enabling the services we will
use for this project, namely, `finance-bigq-demo-2025-mgs` and
`finance-bigq-demo-2025-mgs` with this command:
`finance-bigq-demo-2025-mgs`

### 2.4) Sanity Checks

Before continue it is useful to perform some sanity check commands to
verify everything is working in good order. I recommend
`gcloud config list`, for showing the active project, followed by
`gcloud services list --enabled` that verifies that the BigQuery API
services are enabled. An additional step is verify that your account is
set up as the active account, which you can do with `gcloud auth list`.
After running this last command you should be able to see your account
marked with `*`.

![](https://github.com/Wario84/blog/raw/main/assets/imgs/google_cli_5.png?raw=true)<!-- -->

## 3) The `thelook_ecommerce` Dataset

The GCloud services come with public datasets designed to test Google
Service Capabilities processing Big Data. This particular data set have
variables relevant for the context of finance and eCommerce. The data
set has these following tables according to Google:

| Table Name | Number of Rows | Number of Columns | Description |
|----|----|----|----|
| distribution_centers | 5 | 5 | Lists the distribution centers, including their ID and basic location data. |
| events | 1.1 million+ | 14 | Website activity for users (page views, cart events, etc.). |
| inventory_items | 2.5 million+ | 14 | Item-level inventory records, including status (shipped, returned, etc.). |
| order_items | 2.5 million+ | 19 | Links products to orders with item-level details, including sale price. |
| orders | 2.5 million+ | 10 | Transaction headers for each customer order. |
| products | 28,000+ | 10 | Product catalog with category, brand, and cost. |
| users | 100,000 | 12 | User demographics and traffic source. |

## 4) Conecting Gcloud with Python

This query connects to the data thelook_ecommerce schema and retrieves
all tables and column names with their variable type. For later use I am
saving this as CSV to sudy the variables for further analysis.

``` python
# Load libraries
import pandas as pd # to manipulate data
import pandas_gbq as pgbq # to connect to the Gcloud

# Define ENV Variables
PROJECT_ID = "finance-bigq-demo-2025-mgs"         
LOCATION   = "US"                      # theLook public dataset is in US
USE_STORAGE_API = False                 # set to True if Storage API is enabled, faster for big data
SCHEMA_PATH = "look_ecom_schema.csv"

if 'look_ecom_schema' not in globals():
    # Load from CSV if not already defined
    look_ecom_schema = pd.read_csv(SCHEMA_PATH)
    print(look_ecom_schema.head())
else:
  # read the schema of the data
  look_ecom_schema = pgbq.read_gbq("""SELECT table_name, column_name, data_type
  FROM `bigquery-public-data.thelook_ecommerce`.INFORMATION_SCHEMA.COLUMNS
  WHERE table_name IN ('distribution_centers', 'events', 'inventory_items', 'order_items', 'orders',
  'products', 'users', 'products','order_items')
  ORDER BY table_name, column_name""", project_id=PROJECT_ID, location="US")
  look_ecom_schema.to_csv("look_ecom_schema.csv", index=False)
  print(look_ecom_schema.head())
```

    ##              table_name               column_name  data_type
    ## 0  distribution_centers  distribution_center_geom  GEOGRAPHY
    ## 1  distribution_centers                        id      INT64
    ## 2  distribution_centers                  latitude    FLOAT64
    ## 3  distribution_centers                 longitude    FLOAT64
    ## 4  distribution_centers                      name     STRING

## 5) General Apporach for Data Manipulation (ETL)

My approach to handle Big Data is taking advantage of the process of
filtering, aggregation and and joining that are performed efficiently in
the GCloud with SQL/Google BigQuery. Once the data set is ready, save it
locally as a data mart that have be opened for further transformation
and analysis using Python.

### 6.1 Data Filtering, Aggregation and Joining Strategy

For data consistency I use fallbacks when `NULL` values are detected in
key columns. For instance, when filtering the data by timestamp, I first
check when the product was delivered, and if that is `NULL`, I fallback
to the order creation timestamp:  
`DATE(TIMESTAMP_TRUNC(COALESCE(oi.delivered_at, oi.created_at), MONTH))`.

#### Stage 1

To create the P&L data, I join three tables. I start by retrieving
`oi.sale_price` from the `order_items` fact table, which is used to
calculate revenue. For the calculation of cost, I **left join**
`order_items`
(`bigquery-public-data.thelook_ecommerce.order_items AS oi`) with the
`inventory_items` table
(`LEFT JOIN bigquery-public-data.thelook_ecommerce.inventory_items AS ii`)
using the join key `ON ii.id = oi.inventory_item_id`. Then, as a
fallback, I left join with the `products` table
(`LEFT JOIN bigquery-public-data.thelook_ecommerce.products AS p`) using
the join key `ON p.id = oi.product_id`. This fallback ensures we can
retrieve the `unit_cost` when the cost is missing from the inventory
table, by looking it up in the products table:  
`COALESCE(ii.cost, p.cost) AS unit_cost`.

**Tables & columns used**

- Facts: `order_items` → `oi.sale_price`, `oi.delivered_at`,
  `oi.created_at`, `oi.returned_at`

- Cost: `inventory_items` → `ii.cost` (preferred), `products` → `p.cost`
  (fallback)

#### Stage 2

In the second stage, I aggregate gross revenue as
`SUM(sale_price) AS revenue_gross` and cost as `SUM(unit_cost) AS cogs`.
It is important to note that these are net line prices and costs that do
not include taxes, freight, or other operational expenses that may
affect the estimate.

#### Stage 3

I estimate returns in the **month they occur**, which is beneficial for
real-time operational dashboards. I filter using
`WHERE returned_at IS NOT NULL` to locate returned products. The
aggregations are straightforward:  
`SUM(sale_price) AS returns` and `SUM(unit_cost) AS cogs_returns`.

#### Stage 4

In the last stage I leverage BigQuery to perform fast arithmetic
operations with fallbacks for `NULL` values via `COALESCE`. Here it is
worth noting that I use a `FULL OUTER JOIN` to include all months in the
P&L data, even if they contain only revenue or only returns.

``` python
# Import os to check if the file exist otherwise ETL the Data
import os

# Set the working directory
os.chdir(r"R:/PHD/Semester 20/Jobs/Empresas/Solvo/financial_analyst/project")

# define mart file to save the data
PARQUET_PATH = "pnl_monthly_5y_operational.parquet"

if os.path.isfile(PARQUET_PATH):
    # Load cached mart
    df_pnl = pd.read_parquet(PARQUET_PATH)
    print("Loaded cached mart from", PARQUET_PATH)
    print(df_pnl.head())
else:
    # ETL in the GCloud
    sql = """
    -- Base CTE: delivery-based timing, last 5y, and unit cost with fallback
    WITH base AS (
      SELECT
      
      -- month bucket as a DATE (1st of month, UTC)
      
      DATE(TIMESTAMP_TRUNC(COALESCE(oi.delivered_at, oi.created_at), MONTH)) AS revenue_month,

      oi.returned_at,
      DATE(TIMESTAMP_TRUNC(oi.returned_at, MONTH)) AS return_month,

      oi.sale_price,
      COALESCE(ii.cost, p.cost) AS unit_cost
      FROM `bigquery-public-data.thelook_ecommerce.order_items` AS oi
      LEFT JOIN `bigquery-public-data.thelook_ecommerce.inventory_items` AS ii
      ON ii.id = oi.inventory_item_id
      LEFT JOIN `bigquery-public-data.thelook_ecommerce.products` AS p
      ON p.id = oi.product_id
      WHERE
      -- Compare DATE to DATE (last 5 years)
      DATE(COALESCE(oi.delivered_at, oi.created_at))>= DATE_SUB(CURRENT_DATE(), INTERVAL 5 YEAR)),
      
      -- Operational policy rollup 1: revenue & cogs by delivery month
      revenue_cogs AS (
        SELECT
        revenue_month AS month,
        SUM(sale_price) AS revenue_gross,
        SUM(unit_cost)  AS cogs
        FROM base
        GROUP BY month),
        
        -- Operational policy rollup 2: returns by the month they happen
        returns_only AS (
          SELECT
          return_month AS month,
          SUM(sale_price) AS returns,
          SUM(unit_cost) AS cogs_returns
          FROM base
          WHERE returned_at IS NOT NULL
          GROUP BY month)
        
        -- Final monthly P&L (operational)
        SELECT
        m.month,
        COALESCE(m.revenue_gross, 0)                      AS revenue_gross, 
        COALESCE(r.returns, 0)                            AS returns, 
        (COALESCE(m.revenue_gross, 0) - COALESCE(r.returns, 0)) AS revenue_net, 
        COALESCE(m.cogs, 0)                               AS cogs,
        COALESCE(r.cogs_returns, 0)                             AS cogs_returns,
        -- GP = revenue_net - (cogs - cogs_returns)
        (COALESCE(m.revenue_gross, 0) - COALESCE(r.returns, 0)
        - (COALESCE(m.cogs, 0) - COALESCE(r.cogs_returns, 0))) AS gross_profit
        FROM revenue_cogs m
        FULL OUTER JOIN returns_only r USING (month)
        ORDER BY month"""
      
    # GCloud -> pandas DataFrame
    df_pnl = pgbq.read_gbq(
      sql,
      project_id=PROJECT_ID,
      location=LOCATION,
      use_bqstorage_api=USE_STORAGE_API
    )
    
    # Save locally as Parquet (typed, compressed, fast reloads)
    df_pnl.to_parquet(PARQUET_PATH, index=False)
    print(f"Saved monthly P&L mart (operational policy) to {PARQUET_PATH}")
    print(df_pnl.head())
    print(list(df_pnl.columns))
```

    ## Loaded cached mart from pnl_monthly_5y_operational.parquet
    ##        month  revenue_gross  ...  cogs_returns  gross_profit
    ## 0 2020-10-01   31392.780016  ...   1047.665769  15271.909169
    ## 1 2020-11-01   41859.880024  ...   2139.164078  19352.347184
    ## 2 2020-12-01   42759.699989  ...   2774.276256  18981.968990
    ## 3 2021-01-01   47594.270006  ...   2497.217012  22142.032789
    ## 4 2021-02-01   48373.559991  ...   2644.166791  22102.089914
    ## 
    ## [5 rows x 7 columns]

``` python
    
    
# Some sanity checks
df = df_pnl.copy()
df["gm_pct"] = (df["gross_profit"] / df["revenue_net"]).replace([pd.NA, pd.NaT], 0)
df["cogs_pct"] = (df["cogs"] / df["revenue_gross"]).replace([pd.NA, pd.NaT], 0)

print(df.tail(12)[["month","revenue_gross","returns","revenue_net","cogs","cogs_returns","gross_profit","gm_pct","cogs_pct"]])
```

    ##         month  revenue_gross       returns  ...   gross_profit    gm_pct  cogs_pct
    ## 49 2024-11-01  269082.380185  26214.960039  ...  126124.694354  0.519315  0.480289
    ## 50 2024-12-01  273170.510381  29337.700056  ...  126714.060647  0.519676  0.480170
    ## 51 2025-01-01  297889.750371  30776.970042  ...  138763.774925  0.519495  0.479869
    ## 52 2025-02-01  281236.250379  28218.760069  ...  131552.407939  0.519934  0.479882
    ## 53 2025-03-01  312965.760157  28176.590032  ...  146840.430599  0.515611  0.484954
    ## 54 2025-04-01  339263.010250  32180.190041  ...  159081.004919  0.518039  0.481683
    ## 55 2025-05-01  363811.930262  37850.470028  ...  169256.207956  0.519252  0.480367
    ## 56 2025-06-01  377284.620347  35158.139980  ...  177297.917865  0.518223  0.481696
    ## 57 2025-07-01  448945.130368  40269.100062  ...  212147.626249  0.519110  0.481006
    ## 58 2025-08-01  497113.050341  44222.930023  ...  234784.138660  0.518413  0.481715
    ## 59 2025-09-01  607773.990589  58880.459994  ...  285932.294740  0.520925  0.478581
    ## 60 2025-10-01  529830.500402  61318.850029  ...  243232.963535  0.519161  0.481192
    ## 
    ## [12 rows x 9 columns]

``` python
print("Overall GM% (net):", (df["gross_profit"].sum() / df["revenue_net"].sum()))
```

    ## Overall GM% (net): 0.5190164068500782

``` python
print("Median monthly GM%:", df["gm_pct"].median())
```

    ## Median monthly GM%: 0.5190465748341138

``` python
print("Median monthly COGS%:", df["cogs_pct"].median())
```

    ## Median monthly COGS%: 0.481191899780154

``` python
test = (df_pnl["revenue_net"] - df_pnl["gross_profit"]) - (df_pnl["cogs"] - df_pnl["cogs_returns"])
test.abs().max()
```

    ## np.float64(1.4551915228366852e-11)

### 6.2) Visualize the Profit & Loss (P&L) Statement

For nice

``` python
import plotly.express as px
df_pnl["month"] = pd.to_datetime(df_pnl["month"])
long = df_pnl.melt(
    id_vars="month",
    value_vars=["revenue_gross","returns","revenue_net","cogs","cogs_returns","gross_profit"],
    var_name="metric",
    value_name="value"
)

fig = px.line(long, x="month", y="value", color="metric",
              title="Monthly P&L",
              labels={"month": "Month", "value": "USD", "metric": "Series"},
              markers=True)
fig = fig.update_layout(hovermode="x unified")
fig = fig.update_yaxes(tickprefix="$", separatethousands=True)
fig.show()
```

<div>                        <script type="text/javascript">window.PlotlyConfig = {MathJaxConfig: 'local'};</script>
        <script charset="utf-8" src="https://cdn.plot.ly/plotly-3.1.1.min.js" integrity="sha256-HUEFyfiTnZJxCxur99FjbKYTvKSzwDaD3/x5TqHpFu4=" crossorigin="anonymous"></script>                <div id="42fe75f5-05be-467a-9bc5-69474afbbfaa" class="plotly-graph-div" style="height:100%; width:100%;"></div>            <script type="text/javascript">                window.PLOTLYENV=window.PLOTLYENV || {};                                if (document.getElementById("42fe75f5-05be-467a-9bc5-69474afbbfaa")) {                    Plotly.newPlot(                        "42fe75f5-05be-467a-9bc5-69474afbbfaa",                        [{"hovertemplate":"Series=revenue_gross\u003cbr\u003eMonth=%{x}\u003cbr\u003eUSD=%{y}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"revenue_gross","line":{"color":"#636efa","dash":"solid"},"marker":{"symbol":"circle"},"mode":"lines+markers","name":"revenue_gross","orientation":"v","showlegend":true,"x":["2020-10-01T00:00:00.000000000","2020-11-01T00:00:00.000000000","2020-12-01T00:00:00.000000000","2021-01-01T00:00:00.000000000","2021-02-01T00:00:00.000000000","2021-03-01T00:00:00.000000000","2021-04-01T00:00:00.000000000","2021-05-01T00:00:00.000000000","2021-06-01T00:00:00.000000000","2021-07-01T00:00:00.000000000","2021-08-01T00:00:00.000000000","2021-09-01T00:00:00.000000000","2021-10-01T00:00:00.000000000","2021-11-01T00:00:00.000000000","2021-12-01T00:00:00.000000000","2022-01-01T00:00:00.000000000","2022-02-01T00:00:00.000000000","2022-03-01T00:00:00.000000000","2022-04-01T00:00:00.000000000","2022-05-01T00:00:00.000000000","2022-06-01T00:00:00.000000000","2022-07-01T00:00:00.000000000","2022-08-01T00:00:00.000000000","2022-09-01T00:00:00.000000000","2022-10-01T00:00:00.000000000","2022-11-01T00:00:00.000000000","2022-12-01T00:00:00.000000000","2023-01-01T00:00:00.000000000","2023-02-01T00:00:00.000000000","2023-03-01T00:00:00.000000000","2023-04-01T00:00:00.000000000","2023-05-01T00:00:00.000000000","2023-06-01T00:00:00.000000000","2023-07-01T00:00:00.000000000","2023-08-01T00:00:00.000000000","2023-09-01T00:00:00.000000000","2023-10-01T00:00:00.000000000","2023-11-01T00:00:00.000000000","2023-12-01T00:00:00.000000000","2024-01-01T00:00:00.000000000","2024-02-01T00:00:00.000000000","2024-03-01T00:00:00.000000000","2024-04-01T00:00:00.000000000","2024-05-01T00:00:00.000000000","2024-06-01T00:00:00.000000000","2024-07-01T00:00:00.000000000","2024-08-01T00:00:00.000000000","2024-09-01T00:00:00.000000000","2024-10-01T00:00:00.000000000","2024-11-01T00:00:00.000000000","2024-12-01T00:00:00.000000000","2025-01-01T00:00:00.000000000","2025-02-01T00:00:00.000000000","2025-03-01T00:00:00.000000000","2025-04-01T00:00:00.000000000","2025-05-01T00:00:00.000000000","2025-06-01T00:00:00.000000000","2025-07-01T00:00:00.000000000","2025-08-01T00:00:00.000000000","2025-09-01T00:00:00.000000000","2025-10-01T00:00:00.000000000"],"xaxis":"x","y":{"dtype":"f8","bdata":"AADI6zGo3kAAgCgpfHDkQAAKUGb24ORAAADko0g950AAgHHrsZ7nQAAAAJqpiupAAACZM7P960AAgL2a2bfqQAAAOXG9eexAAACbM5MH7kAAQNZm\u002ftHwQADgGM04RPBAAOCxuH4y8kAAwBuk1GHxQACAvz0+4\u002fFAAID6ZiZk80AAICMp1IPxQABAdilQGPZAAIAd9nyR80AABctwZS72QADAVQA81\u002fZAAEA9zVis+UAAwNUegfT4QADAsmbC+\u002flAAEC0PZKG+UAAwL2Psjf7QACAcNcnUPtAAEAXmhEa\u002fUAAwDNxTQD7QAAA4MJB9QBBAIBWSCmv\u002fkAA4GWuCQkAQQBgSymYLwFBAADjcH9FAkEAkKSun5gBQQAgreG0rwJBAIB6XOtAA0EAYI0A7FwCQQDAEHvExQRBAOCNAOLXBUEA4AcVrIYEQYAigdefZQdBAACpmSdFB0EAwObCe2MLQQAA+Uf31AhBAIAXZ\u002fYqCkEAgERSCHUOQYCifs2ujwtBAMj9o5ucEEEAME+FaWwQQQBYoQpKrBBBAGBhAIcuEkEAYGMAUSoRQQCgZgoXGhNBAOB+Cvy0FEEAsJa4jzQWQQBAPHsSBxdBADB\u002fhcRmG0EAgIwzZFceQQB4Lvs7jCJBAKg0AE0rIEE="},"yaxis":"y","type":"scatter"},{"hovertemplate":"Series=returns\u003cbr\u003eMonth=%{x}\u003cbr\u003eUSD=%{y}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"returns","line":{"color":"#EF553B","dash":"solid"},"marker":{"symbol":"circle"},"mode":"lines+markers","name":"returns","orientation":"v","showlegend":true,"x":["2020-10-01T00:00:00.000000000","2020-11-01T00:00:00.000000000","2020-12-01T00:00:00.000000000","2021-01-01T00:00:00.000000000","2021-02-01T00:00:00.000000000","2021-03-01T00:00:00.000000000","2021-04-01T00:00:00.000000000","2021-05-01T00:00:00.000000000","2021-06-01T00:00:00.000000000","2021-07-01T00:00:00.000000000","2021-08-01T00:00:00.000000000","2021-09-01T00:00:00.000000000","2021-10-01T00:00:00.000000000","2021-11-01T00:00:00.000000000","2021-12-01T00:00:00.000000000","2022-01-01T00:00:00.000000000","2022-02-01T00:00:00.000000000","2022-03-01T00:00:00.000000000","2022-04-01T00:00:00.000000000","2022-05-01T00:00:00.000000000","2022-06-01T00:00:00.000000000","2022-07-01T00:00:00.000000000","2022-08-01T00:00:00.000000000","2022-09-01T00:00:00.000000000","2022-10-01T00:00:00.000000000","2022-11-01T00:00:00.000000000","2022-12-01T00:00:00.000000000","2023-01-01T00:00:00.000000000","2023-02-01T00:00:00.000000000","2023-03-01T00:00:00.000000000","2023-04-01T00:00:00.000000000","2023-05-01T00:00:00.000000000","2023-06-01T00:00:00.000000000","2023-07-01T00:00:00.000000000","2023-08-01T00:00:00.000000000","2023-09-01T00:00:00.000000000","2023-10-01T00:00:00.000000000","2023-11-01T00:00:00.000000000","2023-12-01T00:00:00.000000000","2024-01-01T00:00:00.000000000","2024-02-01T00:00:00.000000000","2024-03-01T00:00:00.000000000","2024-04-01T00:00:00.000000000","2024-05-01T00:00:00.000000000","2024-06-01T00:00:00.000000000","2024-07-01T00:00:00.000000000","2024-08-01T00:00:00.000000000","2024-09-01T00:00:00.000000000","2024-10-01T00:00:00.000000000","2024-11-01T00:00:00.000000000","2024-12-01T00:00:00.000000000","2025-01-01T00:00:00.000000000","2025-02-01T00:00:00.000000000","2025-03-01T00:00:00.000000000","2025-04-01T00:00:00.000000000","2025-05-01T00:00:00.000000000","2025-06-01T00:00:00.000000000","2025-07-01T00:00:00.000000000","2025-08-01T00:00:00.000000000","2025-09-01T00:00:00.000000000","2025-10-01T00:00:00.000000000"],"xaxis":"x","y":{"dtype":"f8","bdata":"AADgtx52oUAAAHBnpjexQAAAUNXjUbZAAABwrscvtEAAAKBm5tK1QAAAOB6F\u002f65AAADY1iN3s0AAAMAVLoC2QAAAjJDCz7RAAACAHgV8uEAAACABADi9QAAAdoNrtr9AAADcjUJXu0AAAGhIYSm3QAAAgITrN7tAAABwC9dewEAAAOQKV0S6QAAAvvUoocFAAAAAPkrSv0AAANJmhhDCQAAAIIXrSsRAAADmcJ3twkAAAICGiwTEQAAALK\u002fnSMdAAABI4TpqwkAAANiFKwTAQAAAOAuXfsVAAAD8HkXgw0AAANRmJnTEQAAAgkgBQcxAAABKUvhfzEAAACAVDobMQAAANqQwg8pAAABUXC+v0EAAAPil0NjMQAAAnNfTmdFAAADw\u002f\u002f+b0UAAAOz1iKbIQAAA+h5l5tBAAAC4cY220EAAAOa5vtHOQAAA+D0aA9RAAACzuE550EAAAGwedV3TQAAAeuH6bNNAAAB\u002fAJAp1kAAAPCPwrDXQAAAxHAtU9RAAAANH8Xo2UAAAElxvZnZQAAAuM1sptxAAAAqFT4O3kAAAPmksI7bQAAAF8MlhNtAAACjKQxt30AAAHkKT3viQAAAt3rEKuFAAEC0M6Op40AAgL\u002fC3ZflQAAARbgOwOxAAABxM9vw7UA="},"yaxis":"y","type":"scatter"},{"hovertemplate":"Series=revenue_net\u003cbr\u003eMonth=%{x}\u003cbr\u003eUSD=%{y}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"revenue_net","line":{"color":"#00cc96","dash":"solid"},"marker":{"symbol":"circle"},"mode":"lines+markers","name":"revenue_net","orientation":"v","showlegend":true,"x":["2020-10-01T00:00:00.000000000","2020-11-01T00:00:00.000000000","2020-12-01T00:00:00.000000000","2021-01-01T00:00:00.000000000","2021-02-01T00:00:00.000000000","2021-03-01T00:00:00.000000000","2021-04-01T00:00:00.000000000","2021-05-01T00:00:00.000000000","2021-06-01T00:00:00.000000000","2021-07-01T00:00:00.000000000","2021-08-01T00:00:00.000000000","2021-09-01T00:00:00.000000000","2021-10-01T00:00:00.000000000","2021-11-01T00:00:00.000000000","2021-12-01T00:00:00.000000000","2022-01-01T00:00:00.000000000","2022-02-01T00:00:00.000000000","2022-03-01T00:00:00.000000000","2022-04-01T00:00:00.000000000","2022-05-01T00:00:00.000000000","2022-06-01T00:00:00.000000000","2022-07-01T00:00:00.000000000","2022-08-01T00:00:00.000000000","2022-09-01T00:00:00.000000000","2022-10-01T00:00:00.000000000","2022-11-01T00:00:00.000000000","2022-12-01T00:00:00.000000000","2023-01-01T00:00:00.000000000","2023-02-01T00:00:00.000000000","2023-03-01T00:00:00.000000000","2023-04-01T00:00:00.000000000","2023-05-01T00:00:00.000000000","2023-06-01T00:00:00.000000000","2023-07-01T00:00:00.000000000","2023-08-01T00:00:00.000000000","2023-09-01T00:00:00.000000000","2023-10-01T00:00:00.000000000","2023-11-01T00:00:00.000000000","2023-12-01T00:00:00.000000000","2024-01-01T00:00:00.000000000","2024-02-01T00:00:00.000000000","2024-03-01T00:00:00.000000000","2024-04-01T00:00:00.000000000","2024-05-01T00:00:00.000000000","2024-06-01T00:00:00.000000000","2024-07-01T00:00:00.000000000","2024-08-01T00:00:00.000000000","2024-09-01T00:00:00.000000000","2024-10-01T00:00:00.000000000","2024-11-01T00:00:00.000000000","2024-12-01T00:00:00.000000000","2025-01-01T00:00:00.000000000","2025-02-01T00:00:00.000000000","2025-03-01T00:00:00.000000000","2025-04-01T00:00:00.000000000","2025-05-01T00:00:00.000000000","2025-06-01T00:00:00.000000000","2025-07-01T00:00:00.000000000","2025-08-01T00:00:00.000000000","2025-09-01T00:00:00.000000000","2025-10-01T00:00:00.000000000"],"xaxis":"x","y":{"dtype":"f8","bdata":"AADMFG553EAAgDpch0niQAAKpuu5FuJAAAAWrk+35EAAgJ0eVeTkQACAHEixmuhAAAC+uM6O6UAAgAXY0+fnQACAJx\u002fF3+lAAADLjxL46kAAgIjN\u002fPztQAAAwymkkexAACDUjwp98EAAgCoffd7vQACAd4W\u002fL\u002fBAAICMhUtY8UAAwOlwHb\u002fvQACAvgor5PNAAIA9UliU8UAAxfCjVOzzQADAsY\u002feTfRAAIAgH6VO90AAwAWu73P2QABAzXClEvdAAECL4Uo590AAwAIfLTf5QACACfZUoPhAAMA39gie+kAAQFmkyHH4QADAr1xjYv5AAEANPioj+0AAwCeaUYH8QAAAED7KDv9AAIBYhZkvAEEAIIpIJZb\u002fQACguWZ6fABBAIB8XGsNAUEAoC5xg9IAQQCAMdf3qAJBAOBWUhDBA0EAgGkpkJkCQYAiwo885QRBAKCSwv01BUEAQBkfzfcIQQDAyetXZwZBAKAHZ8RlB0EAgEYA8H4LQYAiZh9JBQlBAPAZpB78DUEAQHVcm6UNQQCwi3vGww1BAMAOH6NNEEEAoCfsy+IOQQAwNa7UYRFBALDkRyu+EkEAkEfXJeUTQQBg5eu54RRBAKgIH5DxGEEAkDR7aKQbQQAoqg87wCBBADD7mX6YHEE="},"yaxis":"y","type":"scatter"},{"hovertemplate":"Series=cogs\u003cbr\u003eMonth=%{x}\u003cbr\u003eUSD=%{y}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"cogs","line":{"color":"#ab63fa","dash":"solid"},"marker":{"symbol":"circle"},"mode":"lines+markers","name":"cogs","orientation":"v","showlegend":true,"x":["2020-10-01T00:00:00.000000000","2020-11-01T00:00:00.000000000","2020-12-01T00:00:00.000000000","2021-01-01T00:00:00.000000000","2021-02-01T00:00:00.000000000","2021-03-01T00:00:00.000000000","2021-04-01T00:00:00.000000000","2021-05-01T00:00:00.000000000","2021-06-01T00:00:00.000000000","2021-07-01T00:00:00.000000000","2021-08-01T00:00:00.000000000","2021-09-01T00:00:00.000000000","2021-10-01T00:00:00.000000000","2021-11-01T00:00:00.000000000","2021-12-01T00:00:00.000000000","2022-01-01T00:00:00.000000000","2022-02-01T00:00:00.000000000","2022-03-01T00:00:00.000000000","2022-04-01T00:00:00.000000000","2022-05-01T00:00:00.000000000","2022-06-01T00:00:00.000000000","2022-07-01T00:00:00.000000000","2022-08-01T00:00:00.000000000","2022-09-01T00:00:00.000000000","2022-10-01T00:00:00.000000000","2022-11-01T00:00:00.000000000","2022-12-01T00:00:00.000000000","2023-01-01T00:00:00.000000000","2023-02-01T00:00:00.000000000","2023-03-01T00:00:00.000000000","2023-04-01T00:00:00.000000000","2023-05-01T00:00:00.000000000","2023-06-01T00:00:00.000000000","2023-07-01T00:00:00.000000000","2023-08-01T00:00:00.000000000","2023-09-01T00:00:00.000000000","2023-10-01T00:00:00.000000000","2023-11-01T00:00:00.000000000","2023-12-01T00:00:00.000000000","2024-01-01T00:00:00.000000000","2024-02-01T00:00:00.000000000","2024-03-01T00:00:00.000000000","2024-04-01T00:00:00.000000000","2024-05-01T00:00:00.000000000","2024-06-01T00:00:00.000000000","2024-07-01T00:00:00.000000000","2024-08-01T00:00:00.000000000","2024-09-01T00:00:00.000000000","2024-10-01T00:00:00.000000000","2024-11-01T00:00:00.000000000","2024-12-01T00:00:00.000000000","2025-01-01T00:00:00.000000000","2025-02-01T00:00:00.000000000","2025-03-01T00:00:00.000000000","2025-04-01T00:00:00.000000000","2025-05-01T00:00:00.000000000","2025-06-01T00:00:00.000000000","2025-07-01T00:00:00.000000000","2025-08-01T00:00:00.000000000","2025-09-01T00:00:00.000000000","2025-10-01T00:00:00.000000000"],"xaxis":"x","y":{"dtype":"f8","bdata":"ykPWAb0qzUAEAXIAw8PTQMnKioGHWdRAlCd9Jms\u002f1kCQHMYoL8jWQOCGYrhQp9lASyFQFnl62kCywAKiydzZQM7LbuChM9tAIVMFO9Xp3EAmP7dTdTHgQFtSckGvbN9A3Sy\u002f2kGJ4UC44t4x4bTgQK0HB10JDOFAJKRGcyjD4kA4pDYMecPgQHnpnPd0XeVA7TaG2c3Q4kBOu7hD\u002fUjlQG8aUOCuxeVA6HBmFuCh6EAe9M8n9yHoQGbuv73gAelAeHFDi6Ot6EDiP8bFLEbqQMKOHsv\u002fROpASWKuGocg7EBEDtZ3QgHqQAxoQrfZLvBApQdSexRh7UDaQy2pA43uQDoqEOJKifBAvAWHt6qG8UCrSQvIvPHwQEA806OX3fFAv3Q96l9z8kBDOu6HYL\u002fxQFA8z505DPRAyw0royon9UBZdFTqUqPzQNK0uLPTifZAZOS6q22Y9kDXGtqDMFf6QNCnylGNDPhAwFeP+gE6+UCHXvS9fDj9QE3lXAOMqfpAUH5qEtHL\u002f0AexlyLVI3\u002fQArJnGACAwBBqXuFnx9zAUGc7PargnkAQdX6oPDuhgJBV\u002fr2VMnyA0GcW7q+WVUFQdxiANNELwZB9Sbr9EpcCkGk\u002fkgZVzsNQUCVVmrVwBFBAdwPKTEfD0E="},"yaxis":"y","type":"scatter"},{"hovertemplate":"Series=cogs_returns\u003cbr\u003eMonth=%{x}\u003cbr\u003eUSD=%{y}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"cogs_returns","line":{"color":"#FFA15A","dash":"solid"},"marker":{"symbol":"circle"},"mode":"lines+markers","name":"cogs_returns","orientation":"v","showlegend":true,"x":["2020-10-01T00:00:00.000000000","2020-11-01T00:00:00.000000000","2020-12-01T00:00:00.000000000","2021-01-01T00:00:00.000000000","2021-02-01T00:00:00.000000000","2021-03-01T00:00:00.000000000","2021-04-01T00:00:00.000000000","2021-05-01T00:00:00.000000000","2021-06-01T00:00:00.000000000","2021-07-01T00:00:00.000000000","2021-08-01T00:00:00.000000000","2021-09-01T00:00:00.000000000","2021-10-01T00:00:00.000000000","2021-11-01T00:00:00.000000000","2021-12-01T00:00:00.000000000","2022-01-01T00:00:00.000000000","2022-02-01T00:00:00.000000000","2022-03-01T00:00:00.000000000","2022-04-01T00:00:00.000000000","2022-05-01T00:00:00.000000000","2022-06-01T00:00:00.000000000","2022-07-01T00:00:00.000000000","2022-08-01T00:00:00.000000000","2022-09-01T00:00:00.000000000","2022-10-01T00:00:00.000000000","2022-11-01T00:00:00.000000000","2022-12-01T00:00:00.000000000","2023-01-01T00:00:00.000000000","2023-02-01T00:00:00.000000000","2023-03-01T00:00:00.000000000","2023-04-01T00:00:00.000000000","2023-05-01T00:00:00.000000000","2023-06-01T00:00:00.000000000","2023-07-01T00:00:00.000000000","2023-08-01T00:00:00.000000000","2023-09-01T00:00:00.000000000","2023-10-01T00:00:00.000000000","2023-11-01T00:00:00.000000000","2023-12-01T00:00:00.000000000","2024-01-01T00:00:00.000000000","2024-02-01T00:00:00.000000000","2024-03-01T00:00:00.000000000","2024-04-01T00:00:00.000000000","2024-05-01T00:00:00.000000000","2024-06-01T00:00:00.000000000","2024-07-01T00:00:00.000000000","2024-08-01T00:00:00.000000000","2024-09-01T00:00:00.000000000","2024-10-01T00:00:00.000000000","2024-11-01T00:00:00.000000000","2024-12-01T00:00:00.000000000","2025-01-01T00:00:00.000000000","2025-02-01T00:00:00.000000000","2025-03-01T00:00:00.000000000","2025-04-01T00:00:00.000000000","2025-05-01T00:00:00.000000000","2025-06-01T00:00:00.000000000","2025-07-01T00:00:00.000000000","2025-08-01T00:00:00.000000000","2025-09-01T00:00:00.000000000","2025-10-01T00:00:00.000000000"],"xaxis":"x","y":{"dtype":"f8","bdata":"5llAv6lekEDYpgICVLagQF30ZXGNrKVAxtVAHG+Co0DtGphlVaikQItLX7qJP55A\u002ftCJ69RQokBlm4WlkrClQEdgHya83aNA2qKZxBDTpkCfSNUQLX+rQKeMssWxUK5AzVh9XT8TqkB\u002fM+WP+UKmQLbfu2K3lapAggEwJ3aEr0CyKixSjIipQEdEsS1vrrBAVwOrZE+xrkCQU+dckKqxQJYjCp6oyrNAZ41uzqlNskDuhHtTsHizQDJyPfqTb7ZAq\u002fuFcYfYsUC4RL10g6OvQADKAZgyXLRAJ5RFhEE3s0BxTtnnYyq0QHx6p4LenLpALgrNP5K5ukBxxG7cUau6QK3eXj9TlblA1DR1RpKev0Dw7NnXg3C7QKzu07ez8MBAHfO9oFbpwEB4sgolKIK3QPyWU2PmocBAPHz6secMwEBEjBnx95O9QERaoCKUSMNAFFoDz+88wEAT9smzagLDQFYch7MYBMNA\u002fP1OLYJ1xUBdkolMg5LGQH09REJz\u002fsNApJjfyhOsyEDggyd0R2fIQAN3XBTGcMtAemc8R3mDzEALVE80oFvKQN2FRGGQAMtAMdwH9KwbzkAuQX\u002fQfaLRQKno+5cCg9BA3h5Fyz320kDw5Mv2OdzUQLN+Ko0HQdtAYKpbU9353EA="},"yaxis":"y","type":"scatter"},{"hovertemplate":"Series=gross_profit\u003cbr\u003eMonth=%{x}\u003cbr\u003eUSD=%{y}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"gross_profit","line":{"color":"#19d3f3","dash":"solid"},"marker":{"symbol":"circle"},"mode":"lines+markers","name":"gross_profit","orientation":"v","showlegend":true,"x":["2020-10-01T00:00:00.000000000","2020-11-01T00:00:00.000000000","2020-12-01T00:00:00.000000000","2021-01-01T00:00:00.000000000","2021-02-01T00:00:00.000000000","2021-03-01T00:00:00.000000000","2021-04-01T00:00:00.000000000","2021-05-01T00:00:00.000000000","2021-06-01T00:00:00.000000000","2021-07-01T00:00:00.000000000","2021-08-01T00:00:00.000000000","2021-09-01T00:00:00.000000000","2021-10-01T00:00:00.000000000","2021-11-01T00:00:00.000000000","2021-12-01T00:00:00.000000000","2022-01-01T00:00:00.000000000","2022-02-01T00:00:00.000000000","2022-03-01T00:00:00.000000000","2022-04-01T00:00:00.000000000","2022-05-01T00:00:00.000000000","2022-06-01T00:00:00.000000000","2022-07-01T00:00:00.000000000","2022-08-01T00:00:00.000000000","2022-09-01T00:00:00.000000000","2022-10-01T00:00:00.000000000","2022-11-01T00:00:00.000000000","2022-12-01T00:00:00.000000000","2023-01-01T00:00:00.000000000","2023-02-01T00:00:00.000000000","2023-03-01T00:00:00.000000000","2023-04-01T00:00:00.000000000","2023-05-01T00:00:00.000000000","2023-06-01T00:00:00.000000000","2023-07-01T00:00:00.000000000","2023-08-01T00:00:00.000000000","2023-09-01T00:00:00.000000000","2023-10-01T00:00:00.000000000","2023-11-01T00:00:00.000000000","2023-12-01T00:00:00.000000000","2024-01-01T00:00:00.000000000","2024-02-01T00:00:00.000000000","2024-03-01T00:00:00.000000000","2024-04-01T00:00:00.000000000","2024-05-01T00:00:00.000000000","2024-06-01T00:00:00.000000000","2024-07-01T00:00:00.000000000","2024-08-01T00:00:00.000000000","2024-09-01T00:00:00.000000000","2024-10-01T00:00:00.000000000","2024-11-01T00:00:00.000000000","2024-12-01T00:00:00.000000000","2025-01-01T00:00:00.000000000","2025-02-01T00:00:00.000000000","2025-03-01T00:00:00.000000000","2025-04-01T00:00:00.000000000","2025-05-01T00:00:00.000000000","2025-06-01T00:00:00.000000000","2025-07-01T00:00:00.000000000","2025-08-01T00:00:00.000000000","2025-09-01T00:00:00.000000000","2025-10-01T00:00:00.000000000"],"xaxis":"x","y":{"dtype":"f8","bdata":"c8epX\u002fTTzUDXU0M4FubSQMMH7gN+idJAJfM2GYKf1UDO5ifBhZXVQNltfHMKctlA1Rid+D7t2kC78rhi8KjYQDsgpOKfB9tAOuEj\u002fbHg20DIKr2V9AbfQDr\u002fyUqvgN1AsOjAOgcS4UCA8EmGy43gQE62EyTR\u002fOBA9FtF+rXl4UBz3tUpLZTgQBA\u002flgOvgORASHk\u002fwfdC4kAkucUP\u002fsTkQASq1FJjT+VAxWCoYR9F6ECA\u002fKo+\u002fjTnQOA\u002fIqNc8edA\u002fc0DJgMA6EBqFIuvZSLqQH6q9HMwh+lAPNBJAnOC60CKm9dNm2fpQDifLxuvuu9AoRnCSHI87EC0FLDGCcvtQLHD9Y\u002fUHvBAkU2Rd3HS8EAkdfy9cFvwQJaBmqBzOfFApUnToqHE8UDksL\u002fcyF3xQJA2\u002ftzyWfNAvAHC95Jc9EBrJJDnDGnzQHabH\u002fC3qfVA3sZK0yvb9UDro9EQt\u002fj5QLu7OZylIvdAAMgpGTdA+EDF0ymss5f8QGPnt6PU4PlAxFQlr+5B\u002f0BeqhIcy8r+QMxcafig7\u002f5AzsoLM17wAEGkqHVDAw8AQYmt3XHD7AFBbOMSCkhrA0GKrOSpQakEQTnayVePpAVB58yOAp3lCUH6nfkbAakMQati0C2xcxFBS\u002flRtQexDUE="},"yaxis":"y","type":"scatter"}],                        {"template":{"data":{"histogram2dcontour":[{"type":"histogram2dcontour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"choropleth":[{"type":"choropleth","colorbar":{"outlinewidth":0,"ticks":""}}],"histogram2d":[{"type":"histogram2d","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmap":[{"type":"heatmap","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"contourcarpet":[{"type":"contourcarpet","colorbar":{"outlinewidth":0,"ticks":""}}],"contour":[{"type":"contour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"surface":[{"type":"surface","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"mesh3d":[{"type":"mesh3d","colorbar":{"outlinewidth":0,"ticks":""}}],"scatter":[{"fillpattern":{"fillmode":"overlay","size":10,"solidity":0.2},"type":"scatter"}],"parcoords":[{"type":"parcoords","line":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolargl":[{"type":"scatterpolargl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"bar":[{"error_x":{"color":"#2a3f5f"},"error_y":{"color":"#2a3f5f"},"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"bar"}],"scattergeo":[{"type":"scattergeo","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolar":[{"type":"scatterpolar","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"histogram":[{"marker":{"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"histogram"}],"scattergl":[{"type":"scattergl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatter3d":[{"type":"scatter3d","line":{"colorbar":{"outlinewidth":0,"ticks":""}},"marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattermap":[{"type":"scattermap","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattermapbox":[{"type":"scattermapbox","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterternary":[{"type":"scatterternary","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattercarpet":[{"type":"scattercarpet","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"carpet":[{"aaxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"baxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"type":"carpet"}],"table":[{"cells":{"fill":{"color":"#EBF0F8"},"line":{"color":"white"}},"header":{"fill":{"color":"#C8D4E3"},"line":{"color":"white"}},"type":"table"}],"barpolar":[{"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"barpolar"}],"pie":[{"automargin":true,"type":"pie"}]},"layout":{"autotypenumbers":"strict","colorway":["#636efa","#EF553B","#00cc96","#ab63fa","#FFA15A","#19d3f3","#FF6692","#B6E880","#FF97FF","#FECB52"],"font":{"color":"#2a3f5f"},"hovermode":"closest","hoverlabel":{"align":"left"},"paper_bgcolor":"white","plot_bgcolor":"#E5ECF6","polar":{"bgcolor":"#E5ECF6","angularaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"radialaxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"ternary":{"bgcolor":"#E5ECF6","aaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"baxis":{"gridcolor":"white","linecolor":"white","ticks":""},"caxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"coloraxis":{"colorbar":{"outlinewidth":0,"ticks":""}},"colorscale":{"sequential":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"sequentialminus":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"diverging":[[0,"#8e0152"],[0.1,"#c51b7d"],[0.2,"#de77ae"],[0.3,"#f1b6da"],[0.4,"#fde0ef"],[0.5,"#f7f7f7"],[0.6,"#e6f5d0"],[0.7,"#b8e186"],[0.8,"#7fbc41"],[0.9,"#4d9221"],[1,"#276419"]]},"xaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"yaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"scene":{"xaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"yaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"zaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2}},"shapedefaults":{"line":{"color":"#2a3f5f"}},"annotationdefaults":{"arrowcolor":"#2a3f5f","arrowhead":0,"arrowwidth":1},"geo":{"bgcolor":"white","landcolor":"#E5ECF6","subunitcolor":"white","showland":true,"showlakes":true,"lakecolor":"white"},"title":{"x":0.05},"mapbox":{"style":"light"}}},"xaxis":{"anchor":"y","domain":[0.0,1.0],"title":{"text":"Month"}},"yaxis":{"anchor":"x","domain":[0.0,1.0],"title":{"text":"USD"},"tickprefix":"$","separatethousands":true},"legend":{"title":{"text":"Series"},"tracegroupgap":0},"title":{"text":"Monthly P&L"},"hovermode":"x unified"},                        {"responsive": true}                    )                };            </script>        </div>



