Project 2T - Working with Financial Data API
================
Cassio Monti & Smitali Patnaik
10/12/2022

-   <a href="#goal-and-specifications" id="toc-goal-and-specifications">Goal
    and Specifications</a>
-   <a href="#required-packages" id="toc-required-packages">Required
    Packages</a>
-   <a href="#api-querying-functions" id="toc-api-querying-functions">API
    Querying Functions</a>
    -   <a href="#aggregate-endpoint" id="toc-aggregate-endpoint">Aggregate
        EndPoint</a>
    -   <a href="#grouped-daily-endpoint"
        id="toc-grouped-daily-endpoint">Grouped Daily EndPoint</a>
    -   <a href="#technical-indicators-endpoint"
        id="toc-technical-indicators-endpoint">Technical Indicators EndPoint</a>
    -   <a href="#ticker-endpoint" id="toc-ticker-endpoint">Ticker EndPoint</a>
    -   <a href="#wrapper-function" id="toc-wrapper-function">Wrapper
        Function</a>
-   <a href="#eda" id="toc-eda">EDA</a>
    -   <a href="#summary-wrapper-function"
        id="toc-summary-wrapper-function">Summary Wrapper Function</a>
    -   <a href="#for-several-tickers-data---df"
        id="toc-for-several-tickers-data---df">For several tickers data - df</a>
        -   <a href="#modified-data-and-related-plots"
            id="toc-modified-data-and-related-plots">Modified Data and Related
            Plots.</a>
-   <a href="#timeseries-data-frame--time_df"
    id="toc-timeseries-data-frame--time_df">Timeseries Data frame-
    Time_df</a>

# Goal and Specifications

The main goal of this vignette is to provide a set of functions that may
assist in accessing specific information contained in the [Financial
API](https://polygon.io/docs/stocks). These functions aim to provide
data sets for further exploratory data analysis (EDA).  
The companies selected for this analysis belong to two major groups,
technology and real-state companies of the interest of the authors of
this vignette. The companies related to the technology group are Apple,
Microsoft, and Google. The companies related to real-state group are
Weyerhaeuser and Rayonier, both are timberland investment groups.  
Four functions were created querying data from 4 end points, or
searching keywords, present in the the API. Further, three data sets
were created and the EDA was executed for each of them separately. The
EDA encompassed categorical and quantitative analyses, numerical and
graphical, for each data according to their relevance to the
[objective](#eda) of this analysis.  
In order for this analysis to happen, some initial contact with the API
and an access key are required.

# Required Packages

Some packages are necessary to run the code throughout this vignette.
They are related to reading and parsing the API format, JSON format, to
a more friendly and simplified view scheme, rectangular format, through
the `jsonlite` package. The widely used `tidyverse` for data management,
specifically using `dplyr`, for nice correlation plots, `corrplot`, and
for nice table printing, `knitr`.

-   [jsonlite](https://cran.r-project.org/web/packages/jsonlite/):
    Performs the interaction between R and the API that uses the JSON
    file format.

-   [tidyverse](https://www.tidyverse.org/): This package loads several
    other packages with several useful functions addressing data
    management, reshaping, reading, plotting and some more.

-   [corrplot](https://cran.r-project.org/web/packages/corrplot/vignettes/corrplot-intro.html):
    This package has specific functions to display correlations among
    variables in a data set as intuitive plots.

-   [knitr](https://cran.r-project.org/web/packages/knitr/index.html):
    provides nice table printing formats, mainly for the contingency
    tables used herein.

# API Querying Functions

There are four functions that are responsible for querying specific
information from the mentioned financial API. The first function queries
data from the *Aggregate endpoint*, which collects mainly price metrics
for a single ticker over a pre-defined period of time. The second
function, *Grouped endpoint*, queries prices related metrics for all
tickers in a particular day. The function *Technical Indicator
endpoint*, queries information corresponding to the moving average
convergence/divergence (macd) and its related metrics. The forth, and
last function, *ticker endpoint*, queries the overall information about
the tickers, for instance, official names, country of origin, type of
market, and others.  
As operating specificity, the *ticker* and *grouped* endpoints are
working together to unify overall information about the tickers and
price related metrics in a single data set. Additionally, *ticker* and
*aggregate* endpoints are working together for the same purpose, but, in
this case, returning a timely dependent data set.

The function querying the *Technical Indicator endpoint* extracts
information for a single ticker and works with *ticker endpoint* to get
the historical ‘macd’ information with other details included.

## Aggregate EndPoint

This function takes some modifiers defined in the financial API as the
ticker ID `stocksTicker` (default is “Apple”), the time frame arguments
`from` and `to` (these arguments have default values corresponding to
one year period from July 22nd 2021 to July 22nd 2022), the `multiplier`
(default is 30) is the value associated to the `timespan` (default is
“day”) argument and they both together define the time frame in which
the ticker will be obtained. For instance, if `timespan` is “minute” and
multiplier is 1, then the returned data set will be aggregated every
1-minute interval across the time frame selected in `from` and `to`
arguments. Also, `ky` (to define the key ID for the call) is an
argument.  
This function takes all these arguments and passes them to the text
format through the function `paste0()` associating each element of the
URL required to form the query string that goes to the API. The parts of
this URL are the `base_endpoint` (provides initial piece of the URL with
the endpoint call), `last_code` (provides some other modifiers that are
not going to be considered in this function as arguments), and `key`
(provides the key ID to access the API - since API has a limited free
version, more keys are necessary to run a more complex query analysis),
besides the input arguments which are all required. It is important to
notice that this function, and the ones that use single input for
company names, have the input `stocksTicker` as the full name of the
listed company, for example, “Apple” should be provided instead of
“AAPL”. The same for “Microsoft”, “Google”, “Weyerhaeuser”, and
“Rayonier”. Therefore, it was used the function `tolower()` to return
the lower case of the name of the company to avoid errors due to
matching. Associated to the latter function, the `switch()` function
assigns the specific company name to the ticker symbol.  
The function `fromJSON()` has the URL call as input and returns a
simplified object that can be stored as a data frame. The arguments
`from` and `to` are converted into date format and then, through the
function `as.Date()` and `seq()`, a monthly vector is created and joint
with the “results” and “ticker”, from the `fromJSON()`, using `tibble()`
function. The function is defined below.

``` r
# create the URL for aggregate endpoint:
# This function has some default values.
agg_endpoint = function(stocksTicker="Apple Inc.", from = "2021-07-22", to = "2022-07-22",mltplr=30, timespan="day", ky, ...){

  # converting the full name into the ticker symbol
  stocksTicker = switch(tolower(stocksTicker),
                        "apple" = "AAPL",
                        "google" = "GOOGL",
                        "microsoft" = "MSFT",
                        "weyerhaeuser" = "WY",
                        "rayonier" = "RYN",
                        stop("You are allowed to call only a limited number of companies: Apple, GOOGLE, Microsoft, Weyerhaeuser, Rayonier!"))
  
  # passing the components of the URL for the API:
  # base + endpoint 1
  base_endpoint = "https://api.polygon.io/v2/aggs/"
  
  # last part of the URL defining some defaults
  last_code = "?adjusted=true&sort=asc&limit=5000"
  
  # key for accessing API
  key = paste0("&apiKey=", key_id[ky])
  
  # converting the multiplier to character
  mltplr = as.character(mltplr)
  
  # creating the URL call
  call = paste0(base_endpoint,"ticker/",stocksTicker,"/range/",mltplr,"/",
                timespan,"/",from,"/",to,last_code,key)
  
  # assigning the call to an object
  p = fromJSON(call)

  # getting results from the object
  tb = p$results
  tckr = p$ticker
  
  # working with the dates
  d1 = as.Date(from) # transforms initial date from char to date format
  d2 = as.Date(to) # transforms last date from char to date format
  d = seq(d1,d2, by ="month") # sequence by month
  
  # combining the final object with ticker name, date, and metrics
  out = tibble(tckr,d,tb)
  
  # returning the final tibble object
  return(out)
  
}
```

## Grouped Daily EndPoint

This end point returns price related metrics for all tickers in a
particular date. This end point is important to provide data for
analyzing the frequency of tickers by available market type and ticker
type. The function has some inputs: `date` (a specific day for which the
user is interested in knowing the prices), `otc` (to include OTC
securities in the response, other wise no OTC will be added), the
over-the-counter securities are used in further analysis. Also, `ky` (to
define the key ID for the call) is an argument.  
This function begins converting the `otc` argument to lower case through
the function `tolower()` for standardizing purpose. Then, a validation
step was done on `otc` for unauthorized inputs by using `if-else`
statement and a message would come out in case of any unintended input.
The next part assembles the pieces of the URL call and uses the
functions `paste0()` and `fromJSON()` to extract specific information
from the API. The function `tibble()` converts the produced data frame
into a tibble and outputs this object.  
The argument `date` has a default value corresponding to July 22nd 2022,
the same end date defined in the previous function. The function
definition is below.

``` r
# creates the URL for group endpoint:
# This function has some default values.
grouped_endpoint = function(date= "2022-07-22", otc = "true", ky, ...){
  
  # this code sets to lower case the arguments otc.
  otc = tolower(otc)
  
  # check if otc is correctly assigned
  if(otc != "true" && otc != "false"){
    stop("Only true or false allowed")
  }

  # base + endpoint 1
  base="https://api.polygon.io/v2/aggs/grouped/locale/us/market/stocks/"
  
  # key for accessing API
  key = paste0("&apiKey=", key_id[ky])
  
  # creating the URL call
  call = paste0(base,date,"?adjusted=true&include_otc=",otc,key)
  
  # assigning the call to an object
  p = fromJSON(call)

  # transforming the data frame into tibble for better printing
  out = tibble(p$results)
  
  return(out)
}
```

## Technical Indicators EndPoint

This function extracts information from the Technical Indicators end
point, specifically related to the Moving Average Convergence/Divergence
(MACD). This end point gets moving average convergence/divergence (MACD)
data for a ticker symbol over a given time range. The function that runs
the call for this end point starts returning the lower case of the
company name `stocksTicker` and assigning the company name to its
specific ticker symbol through the functions `tolower()` and `switch()`.
Only the available companies mentioned previously are authorized here as
well. Then, the function takes the `date` argument (the same date
defined in the previous function was used here as default as well) and
assembles the the pieces of the URL call using the function `paste0()`
and make the call using `fromJSON()` function. Then, `tibble()` is used
to convert the data frame into a tibble object to be output. The
function definition is below.

``` r
# defining the function.
# some default arguments.
macd_endpoint = function(stocksTicker="Apple", date = "2022-07-22", ky, ..) {

   # converting the full name into the ticker symbol
    stocksTicker = switch(tolower(stocksTicker),
                        "apple" = "AAPL",
                        "google" = "GOOGL",
                        "microsoft" = "MSFT",
                        "weyerhaeuser" = "WY",
                        "rayonier" = "RYN",
                         stop("You are allowed to call only a limited number of companies: Apple, GOOGLE, Microsoft, Weyerhaeuser, Rayonier!"))

  # base + endpoint
  base="https://api.polygon.io/v1/indicators/macd/"
  
  # key for accessing API
  key = paste0("&apiKey=", key_id[ky])
  
  # creating the URL call
  call = paste0(base,stocksTicker,"?timestamp=",date,"&timespan=hour&adjusted=true&short_window=12&long_window=26&signal_window=9&series_type=close&order=desc",key)
  
  
  # assigning the call to an object
  p = fromJSON(call)
  
  # converting the result to a tibble object.
  outd = tibble(p$results$values)
  
  return(outd)
}
```

## Ticker EndPoint

This function a more complex one in order to give the user more
flexibility in relation to the information that can be obtained from the
API. The arguments `type` (default NULL) is an option for looking up to
specific stock type, in which the allowed values are Common Stock (CS),
Investment Fund (FUND), Exchanged-Traded Fund (ETF), and Standard &
Poors (SP). These values are converted to lower case (`tolower()`) to
standardize the inputs and the function returns an error in case of no
concordance with the allowed inputs. The `ticker`(default NULL) is an
options for looking up to specific ticker names, the same aforementioned
(Google, Microsoft, Weyerhaeuser, Rayonir, and Apple). These arguments
are options to the user to get specific information for a particular
group of ticker type or company.  
The argument `market` (default “stocks”) allows the user to choose from
“Stocks” and “OTC” options and return all tickers for a particular
market type. The mechanics of this function is conditional to the
existence of some arguments, for instance, `market` argument will not be
allowed in the URL call if `ticker` exists. This control is necessary in
order to make the function return existing information, because the user
could specify a ticker name that does not exist in a particular market
type, so the search by ticker name is enough in this case. Although in
our function, the ticker symbols are pre-defined, the user can use our
code to make a personalized function with other ticker names, then this
engine would assist in returning the correct result. For the
relationship between `type` and `market`, this issue is not common, so
the user can call `type` and `market` that it will always return the
correct result, then this control is not necessary. This function is
defined below.

``` r
# tickers endpoint= get ticker names and other information
# create the URL for the ticker endpoint - two calls of market:
# i) stocks; and
# ii) otc

ticker_endpoint = function(type = NULL, market = "stocks", limit = 1000, ticker = NULL, ky, ...){
  
  # checking the limit of the call for the free version of the API key.
  if(limit > 1000){
    limit = 1000
    message("Warning: the max limit is 1000 for free access!")
  }

  # assigning the some other components of the URL
  last_code = "&active=true&sort=locale&order=asc&limit="
  
  # assigning the key component of the URL
  key = paste0("&apiKey=", key_id[ky])

  # checking existence for ticker argument
  if(!is.null(ticker)){
    
          # converting the full name into the ticker symbol
          ticker = switch(tolower(ticker),
                        "apple" = "AAPL",
                        "google" = "GOOGL",
                        "microsoft" = "MSFT",
                        "weyerhaeuser" = "WY",
                        "rayonier" = "RYN",
                         stop("You are allowed to call only a limited number of companies: Apple, GOOGLE, Microsoft, Weyerhaeuser, Rayonier!"))

      # base + endpoint
      base_endpoint = "https://api.polygon.io/v3/reference/tickers?ticker="
      
      # creating the URL call
      call = paste0(base_endpoint,ticker)

  }else{
    
      # base + endpoint
      base_endpoint = "https://api.polygon.io/v3/reference/tickers?market="
      
      # converting to lower case
      market = tolower(market)
      
        # check if market is correctly assigned
      if(market != "stocks" && market != "otc"){
          stop("Only true or false allowed for market. Options are: stocks or otc")
      }

      # creating the URL call
      call = paste0(base_endpoint,market)

  }
  
  # checking for existence of type
  if(!is.null(type)){
    
    # if type exists, convert the input name to the correct code for the URL
    type = switch(tolower(type),
               "common stock" = "CS",
               "investment fund" = "FUND",
               "exchanged-traded fund" = "ETF",
               "standard & poors" = "SP",
                stop("This is not one of the allowed options!"))
    
    # create the partial call for the URL
    call = paste0(call, "&type=", type, last_code, limit, key)
    
  }else{
    
    # in case of type is null, then create an alternative URL call
    call = paste0(call, last_code, limit, key)
    
  }

  # accesses the API
  p = fromJSON(call)
  
  return(p$results)

}
```

## Wrapper Function

This function takes the previous functions in its body and runs all the
API calls that they produce. The only argument of this function is
`tickerID` in which the user must provide the company name in order to
return the correct calls. Some objects were created inside of this
wrapper function and they are used within it for data manipulation such
as inner join, through `inner_join()` and `select()` to join data sets
and select important variables to be used in the EDA part. Some renaming
was done as well to provide more comprehensive meaning to the variables.
The created objects **CompanyName** and **agg_data** have the purpose of
making obtaining the specific ticker symbols for the previously selected
companies, via `sapply()`, and obtain a list of data frames encompassing
the aggregate data over time for those companies. The use of the
function `lapply()` here is strategic so that each list element is a
data frame with the same number of columns and rows, which makes them
easy to be merged through `cbind()` function. These objects are used
within the `Combining_calls()` function, defined below. Other objects
are created and editted within the function so that in the end the
output is a list of named elements corresponding to each important
object that will be used in the EDA part. These objects are **df**
corresponding to the tibble that assembles tickers for OTC market and
Stocks market with the grouped daily price derived data set for a
particular day. The object **time_df** corresponds to the merge of
company names and the timely structured data for the five tickers
selected for this analysis. The object **macd_df** corresponds to the
merge between company name and macd end point, that returns the the
Moving Average Convergence/Divergence, for further analysis. The
described wrapper function is defined below.

``` r
# definition of the wrapper function - tickerID = company names for the 
# analysis. These company names have to be within the previously defined.
Combining_calls = function(tickerID, ...){

  # calling the full name of the companies
  CompanyName = sapply(tickers, function(x){
  return(ticker_endpoint(ticker = x, ky = 1)$name)
  })

  # call multiple tickers from agg_endpoint and return sa df
  agg_data = lapply(tickers, agg_endpoint, ky = 1)

  # grouping quantitative EDA data - time analysis
  time_df = lapply(1:length(agg_data), function(x){
    
    return(cbind(Company_Name = CompanyName[x], agg_data[[x]]))
  })
  
  # merging the list elemets in a data frame
  time_df <- do.call("rbind", time_df)
  
  # converting the data frame into a tibble object
  time_df = as_tibble(time_df)
  
  # renaming the final tibble
  time_df = time_df%>%
    rename(
    Closing_price=c,
    Highest_price=h,
    Lowest_price=l, 
    Transactions=n,
    Open_price=o, 
    Unix_time=t,
    Trading_volume=v,
    Volume_wt_avg_price=vw,
    Date=d
   )

  # generating company names for stocks market
  tout = ticker_endpoint(market = "stocks", limit = 1000, ky=2)

  # generating company names for otc market
  tout2 = ticker_endpoint(market = "otc", limit = 1000, ky=2)
  
  # # generating price defived data for both stocks and otc market for the
  # default date.
  gout = grouped_endpoint(otc = "true",ky=2)

  # performing the inner join between otc market info and the price 
  # derived data set
  df1 = inner_join(tout2, gout, by = c("ticker" = "T"))

  # selecting variables of interest
  df11 = df1 %>%
  select(ticker, name, market, type, composite_figi,share_class_figi, v:n)

  # performing the inner join between stocks market info and the price 
  # derived data set
  df2 = inner_join(tout, gout, by = c("ticker" = "T"))

  # selecting variables of interest
  df22 = df2 %>%
    select(ticker, name, market, type,composite_figi,share_class_figi, v:n)

  # combining the data sets.
  df = rbind(df11, df22)

  # dropping any NA cells and renaming important variables
  df = df %>% drop_na() %>%
      rename(
      Closing_price=c,
      Highest_price=h,
      Lowest_price=l, 
      Transactions=n,
      Open_price=o, 
      Unix_time=t,
      Trading_volume=v,
      Volume_wt_avg_price=vw
    )
  
  # macd data extracting
  macd_data = lapply(tickers, macd_endpoint, ky = 3)
  macd_df = lapply(1:length(macd_data), function(x){
    return(cbind(Company_Name = CompanyName[x], macd_data[[x]]))
  
    })
  
  # merging the list elemets in a data frame
  macd_df <- do.call("rbind", macd_df)
  
  # converting to tibble
  macd_df = as_tibble(macd_df)
  
  # return multiple object through list
  return(list(df = df, time_df = time_df,macd_df=macd_df))
  
}
```

# EDA

The main objective of the EDA presented herein is to analyze data sets
and extract meaningful information to the user as if the reader of this
vignette was a beginner in investments seeking for a starting point upon
which market and industry type are more interesting in terms of risk and
return. We are to compare companies listed in the stock market with
over-the-counter or off-exchange trading (OTC) securities. These
companies classified as OTC securities are not listed on a major
exchange in the United States and are instead traded via a broker-dealer
network, usually because many are smaller companies and do not meet the
requirements to be listed on a formal exchange. So, the idea behind this
analysis is to compare how many OTCs are present in the market in
relation to stocks. Additionally, technical indicators for moving
average convergence/divergence and timely data were analyzed for trends
and patterns.

The other aspect of the EDA also explores how prices , transactions, ,
volume etc relate with one other for the given tickers/companies, that
is if any relation or trend can be seen from the historical data or not.

The important objects for the remaining of this analysis are created in
the code below.

``` r
# ticker vector to call the API
# tickers = c("AAPL","GOOGL", "MSFT","WY","RYN")
tickers = c("Apple", "Google", "Microsoft", "Weyerhaeuser", "Rayonier")

# list object containing all the data sets for the analysis
out = Combining_calls(tickerID = tickers)

# assigning the list elements to objects with short preview
df = out$df
df
```

| ticker | name                                                                                                                                                                            | market | type | composite_figi | share_class_figi | Trading_volume | Volume_wt_avg_price |  Open_price | Closing_price | Highest_price | Lowest_price |   Unix_time | Transactions |
|:-------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-------|:-----|:---------------|:-----------------|---------------:|--------------------:|------------:|--------------:|--------------:|-------------:|------------:|-------------:|
| CNTMF  | CANSORTIUM INC                                                                                                                                                                  | otc    | OS   | BBG00NB00924   | BBG00NB00933     |   1.076110e+05 |           0.1849900 |    0.185000 |      0.185000 |      0.188000 |     0.180000 | 1.65852e+12 |           11 |
| OODH   | ORION DIVERSIFIED HLDG                                                                                                                                                          | otc    | CS   | BBG0035BC278   | BBG0035BC2Z7     |   2.043750e+05 |           0.0382130 |    0.043000 |      0.036000 |      0.043000 |     0.035000 | 1.65852e+12 |           20 |
| HEOFF  | H2O INNOVATION INC                                                                                                                                                              | otc    | OS   | BBG000N5WFY6   | BBG001SGQYB6     |   2.752000e+03 |           1.5227000 |    1.545000 |      1.520000 |      1.545000 |     1.520000 | 1.65852e+12 |            7 |
| BNVIF  | BINOVI TECHNOLOGIES CORP                                                                                                                                                        | otc    | OS   | BBG002XSKFG4   | BBG002XSKG81     |   1.042000e+03 |           0.0624500 |    0.064900 |      0.060000 |      0.064900 |     0.060000 | 1.65852e+12 |            2 |
| BDWBY  | BUDWEISER BRWNG UNSP/ADR                                                                                                                                                        | otc    | ADRC | BBG00QYWH6Z4   | BBG00QYWH7Q2     |   5.970000e+02 |          11.4746000 |   11.565000 |     11.565000 |     11.565000 |    11.565000 | 1.65852e+12 |            7 |
| HKMPY  | HIKMA PHARMS PLC S/ADR                                                                                                                                                          | otc    | ADRC | BBG000W5N809   | BBG001T6D0K5     |   1.350000e+03 |          42.1867000 |   43.042000 |     41.673000 |     43.320000 |    41.673000 | 1.65852e+12 |           11 |
| NHMD   | NATE’S FOOD CO                                                                                                                                                                  | otc    | CS   | BBG000R0M2T9   | BBG001SSXL11     |   1.223667e+06 |           0.0014739 |    0.001400 |      0.001500 |      0.001500 |     0.001400 | 1.65852e+12 |            7 |
| HSNGY  | HANG SENG BANK LTD S/ADR                                                                                                                                                        | otc    | ADRC | BBG000BCR233   | BBG001S6BYB4     |   4.143800e+04 |          16.3412000 |   16.325000 |     16.370000 |     16.390000 |    16.270000 | 1.65852e+12 |           76 |
| PDER   | PARDEE RESOURCES CO INC                                                                                                                                                         | otc    | CS   | BBG000BDGHL0   | BBG001S6XND2     |   1.400000e+02 |         257.9900000 |  257.990000 |    257.990000 |    257.990000 |   257.990000 | 1.65852e+12 |           19 |
| YAMHY  | YAMAHA MOTOR CO UNSP/ADR                                                                                                                                                        | otc    | ADRC | BBG000NVZ9N2   | BBG001T3P109     |   6.910000e+02 |           9.6094000 |    9.614000 |      9.455000 |      9.719000 |     9.455000 | 1.65852e+12 |            7 |
| ABIT   | ATHENA BITCOIN GLOBAL                                                                                                                                                           | otc    | CS   | BBG000CKBF87   | BBG001S6Y023     |   2.096000e+03 |           0.3653000 |    0.380000 |      0.320000 |      0.380000 |     0.320000 | 1.65852e+12 |            7 |
| EDTXF  | SPECTRAL MEDICAL INC ORD                                                                                                                                                        | otc    | OS   | BBG000BZX3J6   | BBG001S609G6     |   5.300000e+04 |           0.4292200 |    0.429100 |      0.423140 |      0.432218 |     0.422254 | 1.65852e+12 |           10 |
| LNVGY  | LENOVO GROUP LTD S/ADR                                                                                                                                                          | otc    | ADRC | BBG000BLWMN1   | BBG001S966T6     |   1.896600e+04 |          18.7215000 |   19.380000 |     18.670000 |     19.380000 |    18.620000 | 1.65852e+12 |           89 |
| HTRE   | H3 ENTERPRISES INC                                                                                                                                                              | otc    | CS   | BBG000DZCHB4   | BBG001SFH3T7     |   2.000000e+04 |           0.0000010 |    0.000001 |      0.000001 |      0.000001 |     0.000001 | 1.65852e+12 |            1 |
| WDGJY  | WOOD GROUP JOHN UNSP/ADR                                                                                                                                                        | otc    | ADRC | BBG000KWKCP0   | BBG001T3QNY3     |   5.740000e+02 |           3.2100000 |    3.210000 |      3.210000 |      3.210000 |     3.210000 | 1.65852e+12 |            1 |
| PTOAF  | PIERIDAE ENERGY LIMITED                                                                                                                                                         | otc    | OS   | BBG000LHKG28   | BBG001SK0MJ3     |   1.000000e+04 |           0.7402000 |    0.740200 |      0.740200 |      0.740200 |     0.740200 | 1.65852e+12 |            1 |
| IPIX   | INNOVATION PHARMACEUTICAL                                                                                                                                                       | otc    | CS   | BBG000BF2Q94   | BBG001S8STW0     |   2.764643e+06 |           0.0416890 |    0.041000 |      0.038700 |      0.048000 |     0.037500 | 1.65852e+12 |          173 |
| LNGT   | LASER ENERGETICS INC                                                                                                                                                            | otc    | CS   | BBG000BCM9Z8   | BBG001S6NC65     |   1.000000e+06 |           0.0000010 |    0.000001 |      0.000001 |      0.000001 |     0.000001 | 1.65852e+12 |            1 |
| MWSNF  | MAWSON GOLD LTD                                                                                                                                                                 | otc    | OS   | BBG000P1CSJ4   | BBG001SL2CQ3     |   2.800000e+04 |           0.0752070 |    0.075000 |      0.077000 |      0.077000 |     0.075000 | 1.65852e+12 |            3 |
| ESALY  | EISAI CO S/ADR                                                                                                                                                                  | otc    | ADRC | BBG000BXH728   | BBG001S98X88     |   1.405700e+04 |          46.2843000 |   46.530000 |     46.240000 |     46.530000 |    46.160000 | 1.65852e+12 |           80 |
| GUYGF  | G2 GOLDFIELDS INC                                                                                                                                                               | otc    | CS   | BBG001730W02   | BBG001SR4TD5     |   4.100000e+04 |           0.5572500 |    0.560000 |      0.555000 |      0.561000 |     0.551000 | 1.65852e+12 |            6 |
| ICNB   | ICONIC BRANDS INC                                                                                                                                                               | otc    | CS   | BBG000THC4F2   | BBG001T05T15     |   3.673400e+04 |           0.3800000 |    0.380000 |      0.380000 |      0.380000 |     0.380000 | 1.65852e+12 |            9 |
| ICCT   | ICORECONNECT INC                                                                                                                                                                | otc    | CS   | BBG006TM0XQ5   | BBG006TM0XX7     |   2.397120e+05 |           0.0709920 |    0.071200 |      0.070300 |      0.071200 |     0.070300 | 1.65852e+12 |            6 |
| MNDJF  | MANDALAY RES CORP                                                                                                                                                               | otc    | OS   | BBG000DBGJQ3   | BBG001SHZ3L1     |   6.580000e+02 |           1.8916000 |    1.891000 |      1.891000 |      1.891000 |     1.891000 | 1.65852e+12 |            4 |
| BLRDY  | BILLERUD KORSNAS UNSP/ADR                                                                                                                                                       | otc    | ADRC | BBG000BYTWK8   | BBG001SJLDS3     |   1.500000e+02 |          28.7500000 |   28.750000 |     28.750000 |     28.750000 |    28.750000 | 1.65852e+12 |            1 |
| JUVAF  | JUVA LIFE INC                                                                                                                                                                   | otc    | OS   | BBG00XYJ9YF8   | BBG00XYJ9Z83     |   9.929500e+04 |           0.1312900 |    0.135500 |      0.133000 |      0.141450 |     0.125000 | 1.65852e+12 |           30 |
| ATRWF  | ALTIUS RENEWABLE ROYALTIS                                                                                                                                                       | otc    | OS   | BBG00Y0B1HZ3   | BBG00Y0B1J07     |   3.195000e+03 |           7.3309000 |    7.710000 |      7.000000 |      7.710000 |     7.000000 | 1.65852e+12 |           36 |
| MXROF  | MAX RESOURCE CRP                                                                                                                                                                | otc    | OS   | BBG000BS2XT9   | BBG001S6Z246     |   4.190000e+04 |           0.2741800 |    0.277700 |      0.261100 |      0.277700 |     0.261100 | 1.65852e+12 |            8 |
| ALNPY  | ANA HOLDINGS INC S/ADR                                                                                                                                                          | otc    | ADRC | BBG000BV8G31   | BBG001S7QNR3     |   1.653000e+03 |           3.5003000 |    3.460000 |      3.510000 |      3.510000 |     3.460000 | 1.65852e+12 |            8 |
| SITKF  | SITKA GOLD CORP                                                                                                                                                                 | otc    | OS   | BBG00GXJ53M0   | BBG00GXJ53N9     |   3.100000e+03 |           0.1209200 |    0.124330 |      0.119040 |      0.124330 |     0.119040 | 1.65852e+12 |            3 |
| ROYTL  | PACIFIC COAST OIL TR UTS                                                                                                                                                        | otc    | CS   | BBG002D68SD3   | BBG002D68T41     |   8.265000e+03 |           0.3144200 |    0.300000 |      0.300000 |      0.340000 |     0.300000 | 1.65852e+12 |           17 |
| QBAN   | TELCO CUBA INC                                                                                                                                                                  | otc    | CS   | BBG000BLZC96   | BBG001SP8TQ1     |   2.298706e+08 |           0.0005811 |    0.000700 |      0.000700 |      0.000700 |     0.000500 | 1.65852e+12 |          227 |
| AIQUY  | L’AIR LIQUIDE SA UNSP/ADR                                                                                                                                                       | otc    | ADRC | BBG000BKTQ12   | BBG001S8R1D3     |   1.947760e+05 |          26.3166000 |   26.300000 |     26.200000 |     26.470500 |    26.150000 | 1.65852e+12 |          499 |
| GBTC   | GRAYSCALE BITCOIN TRUST                                                                                                                                                         | otc    | CS   | BBG008748J88   | BBG008748J97     |   3.589472e+06 |          14.9721000 |   15.330000 |     14.480000 |     15.470000 |    14.480000 | 1.65852e+12 |        10218 |
| BERI   | BLUE EARTH RES INC                                                                                                                                                              | otc    | CS   | BBG000DPD8B5   | BBG001SDWCT5     |   2.000000e+04 |           0.1390000 |    0.140000 |      0.136000 |      0.140000 |     0.136000 | 1.65852e+12 |            2 |
| MSBN   | MESSABEN CORP                                                                                                                                                                   | otc    | CS   | BBG000JKSNW8   | BBG001T3NLH9     |   2.513000e+04 |           0.2844200 |    0.276000 |      0.291975 |      0.339900 |     0.276000 | 1.65852e+12 |           12 |
| WYPH   | WAYPOINT BIOMED HLDGS INC                                                                                                                                                       | otc    | CS   | BBG000P36FP8   | BBG001SL3GQ3     |   5.970000e+04 |           0.0213240 |    0.020000 |      0.023200 |      0.023200 |     0.020000 | 1.65852e+12 |            7 |
| HANNF  | HANNAN METALS LTD ORD                                                                                                                                                           | otc    | OS   | BBG000BBFJH6   | BBG001S5Y4W4     |   4.266000e+03 |           0.1663600 |    0.186800 |      0.165000 |      0.186800 |     0.165000 | 1.65852e+12 |            5 |
| WHGOF  | WHITE GOLD CORP ORD                                                                                                                                                             | otc    | OS   | BBG000BZ3586   | BBG001SBL687     |   5.000000e+02 |           0.2948000 |    0.294800 |      0.294800 |      0.294800 |     0.294800 | 1.65852e+12 |            1 |
| MNZO   | MANZO PHARMACEUTICALS INC                                                                                                                                                       | otc    | CS   | BBG000FS8KP1   | BBG001SBVWN2     |   1.080000e+03 |           0.0002000 |    0.000200 |      0.000200 |      0.000200 |     0.000200 | 1.65852e+12 |            2 |
| ZACAF  | ZACAPA RES LTD                                                                                                                                                                  | otc    | OS   | BBG014T47DY7   | BBG014T47G34     |   1.073000e+04 |           0.1820200 |    0.180000 |      0.173000 |      0.188800 |     0.173000 | 1.65852e+12 |            7 |
| OSCI   | OSCEOLA GOLD INC                                                                                                                                                                | otc    | CS   | BBG000Q09Q58   | BBG001T66TW7     |   1.993500e+05 |           0.0372530 |    0.036500 |      0.039900 |      0.039900 |     0.036000 | 1.65852e+12 |           19 |
| LTMCF  | LITHIUM CHILE INC                                                                                                                                                               | otc    | OS   | BBG0018NR8Y4   | BBG001TFBJY1     |   4.132900e+04 |           0.4128900 |    0.424000 |      0.428000 |      0.432750 |     0.401900 | 1.65852e+12 |           20 |
| CLGPF  | CLEAN SEED CAP GROUP LTD                                                                                                                                                        | otc    | OS   | BBG001J47HB1   | BBG001V0GSZ5     |   1.000000e+03 |           0.1614000 |    0.161400 |      0.161400 |      0.161400 |     0.161400 | 1.65852e+12 |            1 |
| CANOF  | CALIFORNIA NANOTECHS CORP                                                                                                                                                       | otc    | OS   | BBG000QMYCQ0   | BBG001SN7G49     |   3.000000e+03 |           0.0725000 |    0.072500 |      0.072500 |      0.072500 |     0.072500 | 1.65852e+12 |            1 |
| MFON   | MOBIVITY HOLDINGS CORP                                                                                                                                                          | otc    | CS   | BBG000PZ4V38   | BBG001T657V7     |   1.300000e+03 |           1.0267000 |    1.030000 |      1.040000 |      1.040000 |     1.010000 | 1.65852e+12 |            5 |
| JBAXY  | JULIUS BAER GRP UNSP/ADR                                                                                                                                                        | otc    | ADRC | BBG000PT2PG3   | BBG001T5ZDQ9     |   5.870200e+04 |           9.3284000 |    9.400000 |      9.310000 |      9.450000 |     9.257500 | 1.65852e+12 |          116 |
| HQGE   | HQ GLOBAL EDUCATION INC                                                                                                                                                         | otc    | CS   | BBG000BBL491   | BBG001S7R9W7     |   5.739960e+05 |           0.0001000 |    0.000100 |      0.000100 |      0.000100 |     0.000100 | 1.65852e+12 |            7 |
| MRNJ   | METATRON INC                                                                                                                                                                    | otc    | CS   | BBG000BHNQY9   | BBG001SB3X33     |   3.425000e+06 |           0.0001854 |    0.000200 |      0.000200 |      0.000200 |     0.000100 | 1.65852e+12 |           12 |
| MNTR   | MENTOR CAPITAL INC                                                                                                                                                              | otc    | CS   | BBG000C1KW64   | BBG001SC0BC2     |   5.640000e+03 |           0.0402590 |    0.040000 |      0.040280 |      0.040280 |     0.040000 | 1.65852e+12 |            4 |
| FLOOF  | FLOWER ONE HOLDINGS INC                                                                                                                                                         | otc    | OS   | BBG000R2WB32   | BBG001ST12P0     |   3.379000e+03 |           0.0230940 |    0.018100 |      0.023400 |      0.023900 |     0.018100 | 1.65852e+12 |            4 |
| HWNI   | HIGH WIRE NETWORKS INC.                                                                                                                                                         | otc    | CS   | BBG000BFMGM9   | BBG001S99Y12     |   1.061250e+05 |           0.1201900 |    0.124000 |      0.123300 |      0.124000 |     0.109100 | 1.65852e+12 |           21 |
| REPO   | NATIONAL ASSET RECOVERY                                                                                                                                                         | otc    | CS   | BBG000BDRS43   | BBG001SDB559     |   1.300000e+02 |           0.0729500 |    0.072950 |      0.072950 |      0.072950 |     0.072950 | 1.65852e+12 |            1 |
| DFIFF  | DIAMOND FIELDS RES INC                                                                                                                                                          | otc    | OS   | BBG000BCJS71   | BBG001S64NZ0     |   9.620000e+02 |           0.1057300 |    0.105700 |      0.105700 |      0.105700 |     0.105700 | 1.65852e+12 |            3 |
| VGLS   | VG LIFE SCIENCES INC                                                                                                                                                            | otc    | CS   | BBG000CPS8S2   | BBG001SFC5F2     |   3.887883e+08 |           0.0002074 |    0.000200 |      0.000300 |      0.000300 |     0.000200 | 1.65852e+12 |          108 |
| BMTM   | BRIGHT MOUNTAIN MEDIA INC                                                                                                                                                       | otc    | CS   | BBG005Q529S6   | BBG005Q529T5     |   1.500000e+02 |           0.2002000 |    0.200200 |      0.200200 |      0.200200 |     0.200200 | 1.65852e+12 |            1 |
| CNNEF  | CANACOL ENERGY ORD                                                                                                                                                              | otc    | OS   | BBG000C12RY4   | BBG001S65X32     |   7.968000e+03 |           1.8680000 |    1.885990 |      1.840000 |      1.885990 |     1.840000 | 1.65852e+12 |           15 |
| ISVG   | INTEGRATED SVCS GRP INC                                                                                                                                                         | otc    | CS   | BBG000N9QK74   | BBG001S6Y8Y1     |   2.000000e+04 |           0.0000010 |    0.000001 |      0.000001 |      0.000001 |     0.000001 | 1.65852e+12 |            1 |
| TWMIF  | TIDEWATER MIDSTRM & INFRA                                                                                                                                                       | otc    | OS   | BBG008871P56   | BBG008871P65     |   2.000000e+03 |           1.0204000 |    1.020400 |      1.020400 |      1.020400 |     1.020400 | 1.65852e+12 |            1 |
| CNIKF  | CANADA NICKEL CO INC                                                                                                                                                            | otc    | OS   | BBG00QGG7B34   | BBG00QGG7B43     |   2.533100e+04 |           1.1336000 |    1.209990 |      1.110000 |      1.209990 |     1.110000 | 1.65852e+12 |           32 |
| IDCBY  | INDUSTRIAL & COM UNSP/ADR                                                                                                                                                       | otc    | ADRC | BBG000RLS431   | BBG001T3SGX7     |   9.518080e+05 |          10.3184000 |   10.220000 |     10.280000 |     10.440000 |    10.220000 | 1.65852e+12 |         1012 |
| OWRDF  | ONE WORLD LITHIUM INC                                                                                                                                                           | otc    | OS   | BBG000FJQPG0   | BBG001SB8YM5     |   1.000000e+02 |           0.0590000 |    0.059000 |      0.059000 |      0.059000 |     0.059000 | 1.65852e+12 |            1 |
| NLLSY  | NEL ASA UNSP/ADR                                                                                                                                                                | otc    | ADRC | BBG00M5B6QS7   | BBG00M5B6RJ5     |   1.730000e+02 |          48.3000000 |   48.300000 |     48.300000 |     48.300000 |    48.300000 | 1.65852e+12 |            1 |
| UNCRY  | UNICREDITO SPA UNSP/ADR                                                                                                                                                         | otc    | ADRC | BBG00HNL0GJ4   | BBG00HNL0H75     |   2.911890e+05 |           4.2913000 |    4.365000 |      4.255000 |      4.410000 |     4.240000 | 1.65852e+12 |          401 |
| PHBI   | PHARMAGREEN BIOTECH INC                                                                                                                                                         | otc    | CS   | BBG000H1B586   | BBG001T33Z26     |   1.275553e+06 |           0.0091643 |    0.009800 |      0.009250 |      0.009800 |     0.009000 | 1.65852e+12 |           11 |
| EKTAY  | ELEKTA B SHS UNSP/ADR                                                                                                                                                           | otc    | ADRC | BBG000KK09J2   | BBG001T0K066     |   1.522500e+04 |           7.0917000 |    7.100000 |      7.120000 |      7.130000 |     7.040000 | 1.65852e+12 |           34 |
| ITOX   | IIOT-OXYS INC                                                                                                                                                                   | otc    | CS   | BBG000BYF8X2   | BBG001SJR6S2     |   1.952556e+06 |           0.0056177 |    0.005800 |      0.006000 |      0.006000 |     0.005600 | 1.65852e+12 |           16 |
| CGRW   | CANNAGROW HOLDINGS INC                                                                                                                                                          | otc    | CS   | BBG000CM3V23   | BBG001S82H27     |   1.023500e+04 |           0.0199890 |    0.020000 |      0.020000 |      0.020000 |     0.020000 | 1.65852e+12 |            4 |
| EJPRY  | EAST JAPAN RWY UNSP/ADR                                                                                                                                                         | otc    | ADRC | BBG000FMZ899   | BBG001T2HYY1     |   9.507100e+04 |           8.0796000 |    8.134000 |      8.100000 |      8.134000 |     8.060000 | 1.65852e+12 |          104 |
| LQWC   | LIFEQUEST WORLD CP                                                                                                                                                              | otc    | CS   | BBG000BXJH06   | BBG001SCWKB8     |   1.370000e+03 |           0.0397440 |    0.041100 |      0.037500 |      0.041100 |     0.037500 | 1.65852e+12 |            6 |
| BUKS   | BUTLER NATL CORP                                                                                                                                                                | otc    | CS   | BBG000D96KF8   | BBG001S7R2V3     |   1.680000e+04 |           0.8300000 |    0.830000 |      0.830000 |      0.830000 |     0.830000 | 1.65852e+12 |            9 |
| GRNF   | GRN HOLDING CORPORATION                                                                                                                                                         | otc    | CS   | BBG001CMFL48   | BBG001TYLPX6     |   2.000000e+03 |           0.0218000 |    0.021800 |      0.021800 |      0.021800 |     0.021800 | 1.65852e+12 |            1 |
| THCBF  | THC BIOMED INTL LTD ORD                                                                                                                                                         | otc    | OS   | BBG000CG9T98   | BBG001S9F3N9     |   2.800700e+04 |           0.0374360 |    0.035000 |      0.038900 |      0.048600 |     0.034200 | 1.65852e+12 |            7 |
| IDGC   | IDGLOBAL CORP                                                                                                                                                                   | otc    | CS   | BBG000DQ70X3   | BBG001SJBJK9     |   3.714781e+08 |           0.0003786 |    0.000300 |      0.000400 |      0.000400 |     0.000200 | 1.65852e+12 |          175 |
| FPWM   | CHARLESTOWNE PREMIUM BEV                                                                                                                                                        | otc    | CS   | BBG000BCN5R4   | BBG001S66NX0     |   1.020000e+02 |           0.0042000 |    0.004200 |      0.004200 |      0.004200 |     0.004200 | 1.65852e+12 |            1 |
| MARUY  | MARUBENI CORP UNSP/ADR                                                                                                                                                          | otc    | ADRC | BBG000BWSLZ1   | BBG001S8ZNM6     |   7.470000e+03 |          89.4293000 |   89.540000 |     89.280000 |     89.920000 |    88.590000 | 1.65852e+12 |          102 |
| FUJIY  | FUJIFILM HLDGS CORP ADR                                                                                                                                                         | otc    | ADRC | BBG000BB7D24   | BBG001S5RDY0     |   9.858000e+03 |          56.3212000 |   56.670000 |     56.190000 |     56.930000 |    55.970000 | 1.65852e+12 |          112 |
| VFRM   | VERITAS FARMS INC                                                                                                                                                               | otc    | CS   | BBG00CGCW1H3   | BBG00CGCW1J1     |   3.538000e+04 |           0.0305140 |    0.030500 |      0.030500 |      0.030600 |     0.030500 | 1.65852e+12 |            5 |
| CNGT   | CANNAGISTICS INC                                                                                                                                                                | otc    | CS   | BBG000JZ4T49   | BBG001SPMVV4     |   5.700000e+04 |           0.0038000 |    0.003800 |      0.003800 |      0.003800 |     0.003800 | 1.65852e+12 |            3 |
| EQMEF  | EQUITY METALS CORPORATION                                                                                                                                                       | otc    | OS   | BBG000BZR4T0   | BBG001S5ZSH7     |   8.350000e+04 |           0.0650290 |    0.065100 |      0.063050 |      0.065300 |     0.063050 | 1.65852e+12 |            5 |
| SHERF  | SHERRITT INTL CORP                                                                                                                                                              | otc    | OS   | BBG000GXW4R6   | BBG001S925B1     |   3.800000e+03 |           0.2810200 |    0.280000 |      0.284300 |      0.284300 |     0.280000 | 1.65852e+12 |            3 |
| CPWR   | OCEAN THERMAL ENERGY CORP                                                                                                                                                       | otc    | CS   | BBG000Q7WSQ0   | BBG001SRWJJ0     |   7.047300e+04 |           0.0102200 |    0.010500 |      0.009300 |      0.010500 |     0.009300 | 1.65852e+12 |            5 |
| SNWR   | SANWIRE CORPORATION                                                                                                                                                             | otc    | CS   | BBG000GNQGW2   | BBG001SKRCV2     |   5.400000e+04 |           0.0070330 |    0.006600 |      0.008100 |      0.008100 |     0.004630 | 1.65852e+12 |            8 |
| DECN   | DECISION DIAGNOSTICS CORP                                                                                                                                                       | otc    | CS   | BBG000F8N562   | BBG001SFWTN9     |   9.000000e+03 |           0.0003000 |    0.000300 |      0.000300 |      0.000300 |     0.000300 | 1.65852e+12 |            2 |
| OBTX   | OBITX INC                                                                                                                                                                       | otc    | CS   | BBG00J03L8K8   | BBG00J03L980     |   2.807000e+03 |           3.1705000 |    3.170000 |      3.180000 |      3.180000 |     3.170000 | 1.65852e+12 |            9 |
| HXLTF  | HELIOSX LITHIUM & TECH                                                                                                                                                          | otc    | OS   | BBG000GLTDG8   | BBG001SD33H0     |   1.609000e+03 |           0.4383100 |    0.402500 |      0.450000 |      0.450000 |     0.402500 | 1.65852e+12 |            9 |
| COWI   | CARBONMETA TECHS INC                                                                                                                                                            | otc    | CS   | BBG000DNX294   | BBG001SJRVH9     |   6.853515e+07 |           0.0005958 |    0.000700 |      0.000600 |      0.000700 |     0.000500 | 1.65852e+12 |          127 |
| VWFB   | VWF BANCORP INC                                                                                                                                                                 | otc    | CS   | BBG0160DXJ08   | BBG0160DXJ17     |   2.825000e+03 |          13.9529000 |   13.530000 |     14.000000 |     14.000000 |    13.530000 | 1.65852e+12 |           13 |
| SOUHY  | SOUTH32 LTD SPNS/ADR                                                                                                                                                            | otc    | ADRC | BBG008MWGWS9   | BBG008MWGWT8     |   5.178100e+04 |          12.3380000 |   12.255000 |     12.170000 |     12.420000 |    12.090000 | 1.65852e+12 |          141 |
| AIMLF  | AI / ML INNOVATIONS INC                                                                                                                                                         | otc    | OS   | BBG000CNRFH4   | BBG001S706N2     |   9.730000e+02 |           0.1422900 |    0.138000 |      0.147174 |      0.147174 |     0.138000 | 1.65852e+12 |            6 |
| NCLTY  | NITORI HOLDINGS CO U/ADR                                                                                                                                                        | otc    | ADRC | BBG00RN65MJ7   | BBG00RN65N87     |   8.571100e+04 |          10.6112000 |   10.410000 |     10.580000 |     10.690000 |    10.250000 | 1.65852e+12 |          172 |
| VWAGY  | VOLKSWAGEN AG UNSP/ADR                                                                                                                                                          | otc    | ADRC | BBG00LPF36F9   | BBG00LPF3758     |   3.419130e+05 |          19.2466000 |   19.420000 |     19.040000 |     19.570000 |    19.010000 | 1.65852e+12 |         1287 |
| ELRFF  | EASTERN PLATINUM LTD NEW                                                                                                                                                        | otc    | OS   | BBG000BPPY90   | BBG001SFXK75     |   1.150000e+04 |           0.1515500 |    0.151000 |      0.155200 |      0.155200 |     0.151000 | 1.65852e+12 |            4 |
| REZNF  | HASH CORP                                                                                                                                                                       | otc    | OS   | BBG00KXCQFD6   | BBG00KXCQFF4     |   1.150000e+02 |           0.0078000 |    0.007800 |      0.007800 |      0.007800 |     0.007800 | 1.65852e+12 |            1 |
| AHKSY  | ASAHI KAISEI CRP UNSP/ADR                                                                                                                                                       | otc    | ADRC | BBG000CKGF37   | BBG001SHFMK2     |   8.480800e+04 |          15.7183000 |   15.330000 |     15.660000 |     16.127100 |    15.330000 | 1.65852e+12 |          239 |
| VMHG   | VICTORY MARINE HLDGS CORP                                                                                                                                                       | otc    | CS   | BBG000BDJHH2   | BBG001S6XS32     |   1.652000e+03 |           0.0140500 |    0.014050 |      0.014050 |      0.014050 |     0.014050 | 1.65852e+12 |            1 |
| TSCC   | TECHNOLOGY SLTNS CO                                                                                                                                                             | otc    | CS   | BBG000CCLR97   | BBG001S6T1P2     |   1.780000e+02 |           0.0007000 |    0.000700 |      0.000700 |      0.000700 |     0.000700 | 1.65852e+12 |            1 |
| EGTYF  | EGUANA TECHS INC ORD                                                                                                                                                            | otc    | OS   | BBG000DKK276   | BBG001SDLQH9     |   1.357690e+05 |           0.2021400 |    0.190000 |      0.200000 |      0.203100 |     0.190000 | 1.65852e+12 |           32 |
| APRU   | APPLE RUSH COMP INC                                                                                                                                                             | otc    | CS   | BBG000TKJLW1   | BBG001T0B360     |   2.275620e+05 |           0.0017560 |    0.001800 |      0.001800 |      0.001800 |     0.001750 | 1.65852e+12 |            4 |
| ENDTF  | CANOE EIT INCOME FD                                                                                                                                                             | otc    | UNIT | BBG000BSYXJ5   | BBG001SB5N24     |   3.553000e+03 |           9.6656000 |    9.720000 |      9.658000 |      9.720000 |     9.658000 | 1.65852e+12 |            6 |
| LTGHY  | LIFE HLTHCRE GRP USP/ADR                                                                                                                                                        | otc    | ADRC | BBG004BJ4QL2   | BBG004BJ4RB1     |   3.988000e+03 |           4.4998000 |    4.495000 |      4.495000 |      4.570000 |     4.420000 | 1.65852e+12 |           17 |
| HDII   | HYPERTENSION DIAGNSTCS                                                                                                                                                          | otc    | CS   | BBG000C4T1L0   | BBG001SBZ8N1     |   4.610010e+05 |           0.0044392 |    0.004500 |      0.004000 |      0.005900 |     0.003700 | 1.65852e+12 |           14 |
| GXXFF  | GOLD BASIN RESOURCES CORP                                                                                                                                                       | otc    | OS   | BBG00PVP2573   | BBG00PVP2582     |   1.000000e+03 |           0.1264000 |    0.126400 |      0.126400 |      0.126400 |     0.126400 | 1.65852e+12 |            1 |
| MURGY  | MUENCHENER RE GP UNSP/ADR                                                                                                                                                       | otc    | ADRC | BBG000GCM2Q8   | BBG001T3P010     |   8.835700e+04 |          22.3125000 |   22.452000 |     22.270000 |     22.480000 |    22.220000 | 1.65852e+12 |          237 |
| KMTUY  | KOMATSU LTD S/ADR                                                                                                                                                               | otc    | ADRC | BBG000BYKR00   | BBG001S9Z160     |   1.884580e+05 |          21.5526000 |   21.750000 |     21.570000 |     22.060000 |    21.130000 | 1.65852e+12 |          451 |
| GCXXF  | GRANITE CREEK COPPER LTD                                                                                                                                                        | otc    | OS   | BBG001V59KB5   | BBG001V59KY0     |   2.200000e+04 |           0.0589090 |    0.058800 |      0.060000 |      0.060000 |     0.058800 | 1.65852e+12 |            2 |
| RUPRF  | RUPERT RESOURCES LTD                                                                                                                                                            | otc    | OS   | BBG000DP83K2   | BBG001S89XP0     |   9.050000e+02 |           3.2248000 |    3.270000 |      3.050000 |      3.280000 |     3.050000 | 1.65852e+12 |            5 |
| OLCLY  | ORIENTAL LAND CO LTD ADR                                                                                                                                                        | otc    | ADRC | BBG002S9ZGP4   | BBG002S9ZHF3     |   1.466000e+03 |          28.3887000 |   28.731000 |     28.390000 |     28.731000 |    28.160000 | 1.65852e+12 |            9 |
| CMWAY  | CMNWLTH BK AUSTRLA S/ADR                                                                                                                                                        | otc    | ADRC | BBG000C70884   | BBG001SFSZT4     |   2.022600e+04 |          67.9843000 |   69.210000 |     67.720000 |     69.210000 |    67.530000 | 1.65852e+12 |          145 |
| ANRGF  | ANAERGIA INC SUB VTG SHS                                                                                                                                                        | otc    | OS   | BBG002GMD9X9   | BBG002GMD9Y8     |   1.870000e+02 |           5.8000000 |    5.800000 |      5.800000 |      5.800000 |     5.800000 | 1.65852e+12 |            2 |
| BYSD   | BAYSIDE CORP                                                                                                                                                                    | otc    | CS   | BBG000D02K98   | BBG001SCR581     |   5.000000e+02 |           0.2100000 |    0.210000 |      0.210000 |      0.210000 |     0.210000 | 1.65852e+12 |            1 |
| CODYY  | COMPAGNIE ST GBN UNSP/ADR                                                                                                                                                       | otc    | ADRC | BBG000C2K6K3   | BBG001SM7N73     |   1.681150e+05 |           8.7672000 |    8.820300 |      8.740000 |      8.890000 |     8.700000 | 1.65852e+12 |          198 |
| PRKA   | PARKS! AMERICA INC                                                                                                                                                              | otc    | CS   | BBG000C28XY3   | BBG001SCGCB4     |   1.670000e+02 |           0.4150000 |    0.415000 |      0.415000 |      0.415000 |     0.415000 | 1.65852e+12 |            1 |
| DXBRF  | BELLROCK BRANDS INC                                                                                                                                                             | otc    | OS   | BBG00MP45DW6   | BBG00MP45FX0     |   1.200000e+03 |           0.0001000 |    0.000100 |      0.000100 |      0.000100 |     0.000100 | 1.65852e+12 |            1 |
| PHPPY  | SIGNIFY NV UNSP/ADR                                                                                                                                                             | otc    | ADRC | BBG00HLJ8Y04   | BBG00HLJ8YQ6     |   4.600000e+02 |          18.0500000 |   18.050000 |     18.050000 |     18.050000 |    18.050000 | 1.65852e+12 |            1 |
| BKTPF  | CRUZ BATTERY METALS CORP                                                                                                                                                        | otc    | OS   | BBG000RMF3F2   | BBG001STX5B3     |   6.020200e+04 |           0.1215400 |    0.118600 |      0.127000 |      0.128700 |     0.114600 | 1.65852e+12 |           25 |
| DRCMF  | DORE COPPER MNG CORP                                                                                                                                                            | otc    | OS   | BBG00LT3NLH5   | BBG00LT3NMH3     |   1.300000e+03 |           0.2910800 |    0.289570 |      0.296100 |      0.296100 |     0.289570 | 1.65852e+12 |            2 |
| NOCSF  | NORSEMAN SILVER INC                                                                                                                                                             | otc    | OS   | BBG000BJWBZ7   | BBG001S5RCF3     |   8.500000e+03 |           0.1160000 |    0.116900 |      0.108360 |      0.116900 |     0.108360 | 1.65852e+12 |            2 |
| SDVKY  | SANDVIK AB S/ADR                                                                                                                                                                | otc    | ADRC | BBG000BN3280   | BBG001SC9KM2     |   1.225610e+05 |          17.1567000 |   17.267000 |     17.090000 |     17.350000 |    16.990000 | 1.65852e+12 |          309 |
| CANSF  | WILLOW BIOSCIENCES INC                                                                                                                                                          | otc    | OS   | BBG000BXWNR9   | BBG001S5Y4J9     |   1.120000e+04 |           0.1205200 |    0.121400 |      0.120000 |      0.122000 |     0.120000 | 1.65852e+12 |            7 |
| ESKYF  | ESKAY MINING CORP                                                                                                                                                               | otc    | OS   | BBG000H8C1H9   | BBG001SDQP59     |   2.541800e+04 |           1.4626000 |    1.470000 |      1.440000 |      1.486900 |     1.425000 | 1.65852e+12 |           34 |
| SVNLY  | SVENSKA HANDELSBK UNS/ADR                                                                                                                                                       | otc    | ADRC | BBG000P2S4W3   | BBG001T3RTK4     |   6.140430e+05 |           4.2884000 |    4.315000 |      4.270000 |      4.315000 |     4.250000 | 1.65852e+12 |          218 |
| LUMB   | LUMBEE GUARANTY BK                                                                                                                                                              | otc    | CS   | BBG000BNQKG6   | BBG001S9J970     |   2.000000e+02 |          10.4500000 |   10.450000 |     10.450000 |     10.450000 |    10.450000 | 1.65852e+12 |            2 |
| CPPMF  | COPPER MTN MNG CORP                                                                                                                                                             | otc    | OS   | BBG000QQ07K5   | BBG001SSG953     |   9.844200e+04 |           1.0973000 |    1.150000 |      1.090000 |      1.150000 |     1.080000 | 1.65852e+12 |          128 |
| CKHUY  | CK HUTCH HLD LTD UNSP/ADR                                                                                                                                                       | otc    | ADRC | BBG008D4TY21   | BBG008D4TYW8     |   2.983710e+05 |           6.5536000 |    6.570000 |      6.540000 |      6.575000 |     6.530000 | 1.65852e+12 |          138 |
| NBMFF  | NEO BATTERY MATLS LTD                                                                                                                                                           | otc    | OS   | BBG000GPLSR2   | BBG001SNZTD0     |   1.780000e+03 |           0.1243400 |    0.124000 |      0.124000 |      0.124000 |     0.124000 | 1.65852e+12 |            2 |
| ORKLY  | ORKLA AS A S/ADR                                                                                                                                                                | otc    | ADRC | BBG000BN2RR5   | BBG001SBVXF9     |   9.696300e+04 |           8.2836000 |    8.210000 |      8.260000 |      8.304000 |     8.210000 | 1.65852e+12 |           60 |
| CRCUF  | CANAGOLD RES LTD                                                                                                                                                                | otc    | OS   | BBG000BXYCT9   | BBG001S5Y5R7     |   1.000000e+03 |           0.2200000 |    0.220000 |      0.220000 |      0.220000 |     0.220000 | 1.65852e+12 |            1 |
| BSTO   | BLUE STAR OPPTYS CORP                                                                                                                                                           | otc    | CS   | BBG000CJS7B6   | BBG001S81V09     |   5.323000e+04 |           0.0544820 |    0.045000 |      0.046000 |      0.056055 |     0.045000 | 1.65852e+12 |           10 |
| TAKOF  | DRONE DELIVRY CDA COM&VAR                                                                                                                                                       | otc    | OS   | BBG001PG5LL9   | BBG001V187W1     |   1.272900e+04 |           0.4262300 |    0.420000 |      0.430000 |      0.432930 |     0.419700 | 1.65852e+12 |           10 |
| SNWV   | SANUWAVE HEALTH INC                                                                                                                                                             | otc    | CS   | BBG000C33X21   | BBG001SV44J5     |   1.994000e+05 |           0.0713850 |    0.071000 |      0.070000 |      0.077000 |     0.070000 | 1.65852e+12 |           17 |
| PEGRY  | PENNON GROUP PLC UNSP/ADR                                                                                                                                                       | otc    | ADRC | BBG000K37GQ9   | BBG001T4B031     |   1.900000e+02 |          24.3703000 |   24.315000 |     24.315000 |     24.315000 |    24.315000 | 1.65852e+12 |            5 |
| TCRI   | TECHCOM INC                                                                                                                                                                     | otc    | CS   | BBG000F08YN4   | BBG001SKKK23     |   2.950500e+04 |           0.0798830 |    0.076900 |      0.061400 |      0.086000 |     0.058500 | 1.65852e+12 |           16 |
| ZNOG   | ZION OIL & GAS INC                                                                                                                                                              | otc    | CS   | BBG000RFZLM7   | BBG001SSCT99     |   1.082253e+06 |           0.2481300 |    0.255000 |      0.242000 |      0.256500 |     0.240000 | 1.65852e+12 |          228 |
| ASCK   | AUSCRETE CORP                                                                                                                                                                   | otc    | CS   | BBG001QCG512   | BBG001V1FTM6     |   1.000000e+03 |           0.0284000 |    0.028400 |      0.028400 |      0.028400 |     0.028400 | 1.65852e+12 |            1 |
| BRLXF  | BORALEX INC A                                                                                                                                                                   | otc    | OS   | BBG000BXSFH2   | BBG001S5Y1Q7     |   4.910000e+02 |          33.9955000 |   34.330000 |     33.790000 |     34.410000 |    33.670000 | 1.65852e+12 |            5 |
| RIII   | RENAVOTIO INC                                                                                                                                                                   | otc    | CS   | BBG004JLQY47   | BBG004JLQYX5     |   4.000000e+04 |           0.0194840 |    0.016000 |      0.020000 |      0.020000 |     0.016000 | 1.65852e+12 |            6 |
| JENGQ  | JUST ENERGY GROUP INC                                                                                                                                                           | otc    | OS   | BBG000D8RJ97   | BBG001SHTXC2     |   2.424480e+05 |           0.2407800 |    0.279900 |      0.243700 |      0.279900 |     0.216250 | 1.65852e+12 |           34 |
| LEAS   | STRATEGIC ASSET LEASING                                                                                                                                                         | otc    | CS   | BBG000D1JK07   | BBG001SJQSW0     |   2.268338e+06 |           0.0010588 |    0.001000 |      0.001200 |      0.001200 |     0.001000 | 1.65852e+12 |           10 |
| DSNY   | DESTINY MEDIA TECHS INC                                                                                                                                                         | otc    | OS   | BBG000BNMTH0   | BBG001SDPPH7     |   3.010000e+03 |           0.6299500 |    0.630000 |      0.630000 |      0.630000 |     0.630000 | 1.65852e+12 |            2 |
| CQRLF  | CONQUEST RESOURCES LTD                                                                                                                                                          | otc    | OS   | BBG000LMLCR8   | BBG001S5YDG2     |   2.290000e+03 |           0.0222000 |    0.022200 |      0.022200 |      0.022200 |     0.022200 | 1.65852e+12 |            1 |
| VDMCY  | VODACOM GROUP LTD S/ADR                                                                                                                                                         | otc    | ADRC | BBG004SHX9T6   | BBG004SHXHH1     |   9.830000e+04 |           8.4516000 |    8.550000 |      8.440000 |      8.550000 |     8.368000 | 1.65852e+12 |           80 |
| SRUTF  | SPROUTLY CDA INC                                                                                                                                                                | otc    | OS   | BBG009S509J8   | BBG009S509K6     |   5.074200e+04 |           0.0110020 |    0.011370 |      0.010100 |      0.012000 |     0.010100 | 1.65852e+12 |            9 |
| PCLOF  | PHARMACIELO LTD                                                                                                                                                                 | otc    | OS   | BBG00J4XKS07   | BBG00J4XKSY0     |   1.385000e+03 |           0.3395900 |    0.350000 |      0.331770 |      0.350000 |     0.331770 | 1.65852e+12 |            4 |
| NXTTF  | LIFEIST WELLNESS INC                                                                                                                                                            | otc    | OS   | BBG000C7ZY25   | BBG001SLP413     |   1.761370e+05 |           0.0333170 |    0.035000 |      0.031000 |      0.035600 |     0.029150 | 1.65852e+12 |           38 |
| KTPPF  | KATIPULT TECH CORP ORD                                                                                                                                                          | otc    | OS   | BBG00J7KY6J5   | BBG00J7KY7F7     |   2.000000e+03 |           0.0562700 |    0.056270 |      0.056270 |      0.056270 |     0.056270 | 1.65852e+12 |            2 |
| MYCOF  | MYDECINE INNOVATNS GP INC                                                                                                                                                       | otc    | OS   | BBG0069PSQH5   | BBG0069PSQJ3     |   3.338800e+04 |           0.5238900 |    0.550000 |      0.560000 |      0.560000 |     0.410000 | 1.65852e+12 |           42 |
| STHFF  | STELMINE CDA LTD ORD                                                                                                                                                            | otc    | OS   | BBG000QLLRJ1   | BBG001SSBLS6     |   7.346500e+04 |           0.1422800 |    0.118000 |      0.151800 |      0.155600 |     0.118000 | 1.65852e+12 |            9 |
| ATZAF  | ARITZIA INC ORD                                                                                                                                                                 | otc    | OS   | BBG00DR7R5K3   | BBG00DR7R5L2     |   1.134000e+03 |          30.8261000 |   30.800300 |     30.163900 |     31.060000 |    30.163900 | 1.65852e+12 |           12 |
| VACNY  | VAT GROUP AG UNSP/ADR                                                                                                                                                           | otc    | ADRC | BBG00K17Z0H1   | BBG00K17Z161     |   1.026000e+03 |          27.6178000 |   26.710000 |     27.450000 |     28.190000 |    26.710000 | 1.65852e+12 |           16 |
| WAYN   | WAYNE SAVINGS BNCSHS INC                                                                                                                                                        | otc    | CS   | BBG000BHRMK9   | BBG001S7D881     |   5.000000e+02 |          26.0000000 |   26.000000 |     26.000000 |     26.000000 |    26.000000 | 1.65852e+12 |            2 |
| APLIF  | APPILI THERAPEUTICS INC                                                                                                                                                         | otc    | OS   | BBG00CZT89F7   | BBG00CZT89G6     |   6.203000e+03 |           0.0486410 |    0.047520 |      0.050300 |      0.050300 |     0.047520 | 1.65852e+12 |            3 |
| CMGR   | CLUBHOUSE MEDIA GROUP INC                                                                                                                                                       | otc    | CS   | BBG000TJHBF7   | BBG001T07JD2     |   2.124224e+07 |           0.0019223 |    0.002000 |      0.002000 |      0.002200 |     0.001700 | 1.65852e+12 |          115 |
| RAMPF  | POLARIS RENEWABLE ENERGY                                                                                                                                                        | otc    | OS   | BBG000CC0V00   | BBG001S6SR67     |   1.814000e+03 |          16.6358000 |   16.680000 |     16.600000 |     16.730100 |    16.600000 | 1.65852e+12 |            7 |
| LUDG   | LUDWIG ENTERPRISES                                                                                                                                                              | otc    | CS   | BBG000R8VRF2   | BBG001STBDG5     |   8.004400e+04 |           0.0373790 |    0.037000 |      0.038000 |      0.038000 |     0.037000 | 1.65852e+12 |            5 |
| PTRDF  | ROK RESOURCES INC                                                                                                                                                               | otc    | OS   | BBG000JTCLD1   | BBG001SPLDY2     |   8.000000e+02 |           0.1600000 |    0.160000 |      0.160000 |      0.160000 |     0.160000 | 1.65852e+12 |            1 |
| VBHI   | VERDE BIO HLDGS INC                                                                                                                                                             | otc    | CS   | BBG0025YXBD0   | BBG0025YXC48     |   6.170406e+06 |           0.0035977 |    0.003700 |      0.003400 |      0.003900 |     0.003100 | 1.65852e+12 |           60 |
| BDRFY  | BEIERDORF AG UNSP/ADR                                                                                                                                                           | otc    | ADRC | BBG000PK4PR8   | BBG001T41KY4     |   4.368020e+05 |          20.4437000 |   20.480000 |     20.250000 |     20.490000 |    20.240000 | 1.65852e+12 |          166 |
| SMFKY  | SMURFIT KAPPA GROUP ADR                                                                                                                                                         | otc    | ADRC | BBG002CHFW52   | BBG002CHFXG8     |   4.765700e+04 |          33.3463000 |   33.750000 |     32.870000 |     34.070000 |    32.820000 | 1.65852e+12 |          261 |
| KELTF  | KELT EXPLORATION LTD                                                                                                                                                            | otc    | OS   | BBG003NPPZK5   | BBG003NPPZL4     |   5.978000e+03 |           4.6508000 |    4.330000 |      4.570000 |      4.695000 |     4.330000 | 1.65852e+12 |           15 |
| KSRYY  | KOSE CORP UNSP/ADR                                                                                                                                                              | otc    | ADRC | BBG00CDNB951   | BBG00CDNB9W1     |   1.387900e+04 |          17.9130000 |   17.990000 |     17.860000 |     18.050000 |    17.830000 | 1.65852e+12 |           68 |
| PAYD   | PAID INC                                                                                                                                                                        | otc    | CS   | BBG000K8C8Q0   | BBG001SC6YL6     |   7.840000e+02 |           1.9197000 |    1.920000 |      1.925000 |      1.925000 |     1.920000 | 1.65852e+12 |            6 |
| CSCCF  | CAPSTONE COPPER CORP                                                                                                                                                            | otc    | OS   | BBG00CTTHYR6   | BBG017Z2HGQ8     |   2.926000e+04 |           2.0412000 |    2.020000 |      1.940000 |      2.078500 |     1.940000 | 1.65852e+12 |           24 |
| SAENF  | SOLAR ALLIANCE ENERGY NEW                                                                                                                                                       | otc    | OS   | BBG00JGBJJW9   | BBG00JGBJJX8     |   7.473000e+03 |           0.0532190 |    0.059000 |      0.055100 |      0.059000 |     0.048000 | 1.65852e+12 |            8 |
| ARTM   | AMER NORTEL COMMUN INC                                                                                                                                                          | otc    | CS   | BBG000D3WZK3   | BBG001S7F9Q7     |   6.155000e+04 |           0.0272450 |    0.028000 |      0.028000 |      0.028000 |     0.025000 | 1.65852e+12 |            7 |
| ELRA   | ELRAY RESOURCES INC                                                                                                                                                             | otc    | CS   | BBG000DC6181   | BBG001SMJ7S2     |   6.249777e+08 |           0.0022075 |    0.001500 |      0.002300 |      0.002800 |     0.001400 | 1.65852e+12 |         1392 |
| IFLM   | INDEPENDENT FILM DEV CORP                                                                                                                                                       | otc    | CS   | BBG000NN8CK6   | BBG001SQP3W0     |   1.714400e+04 |           0.0000010 |    0.000001 |      0.000001 |      0.000001 |     0.000001 | 1.65852e+12 |            1 |
| WPFH   | WORLD POKER FD HLDGS                                                                                                                                                            | otc    | CS   | BBG000PR5X04   | BBG001SR9W89     |   1.800000e+03 |           0.0786000 |    0.078600 |      0.078600 |      0.078600 |     0.078600 | 1.65852e+12 |            1 |
| GOOLF  | AQUARIUS AI INC                                                                                                                                                                 | otc    | OS   | BBG0046YVGQ2   | BBG0046YVHC5     |   5.905000e+03 |           0.0379680 |    0.026000 |      0.100000 |      0.100000 |     0.026000 | 1.65852e+12 |            4 |
| FTEG   | FOR THE EARTH CORP                                                                                                                                                              | otc    | CS   | BBG000HPQBH4   | BBG001S9BL53     |   2.645723e+07 |           0.0000959 |    0.000100 |      0.000100 |      0.000100 |     0.000001 | 1.65852e+12 |           30 |
| SPHRY  | STARPHARMA HLDGS S/ADR                                                                                                                                                          | otc    | ADRC | BBG000G4ZF47   | BBG001SNC510     |   4.030000e+02 |           4.4560000 |    4.456000 |      4.456000 |      4.456000 |     4.456000 | 1.65852e+12 |            1 |
| ERKH   | EUREKA HOMESTEAD BNCRP                                                                                                                                                          | otc    | CS   | BBG00NKWF5K6   | BBG00NKWF5L5     |   2.200000e+03 |          14.9091000 |   15.000000 |     15.000000 |     15.000000 |    14.800000 | 1.65852e+12 |            6 |
| BIOF   | BLUE BIOFUELS INC                                                                                                                                                               | otc    | CS   | BBG0032F1309   | BBG0032F13S9     |   2.929770e+05 |           0.1654000 |    0.167800 |      0.172800 |      0.173800 |     0.158000 | 1.65852e+12 |           54 |
| GBRCF  | GOLD BULL RESOURCES CORP                                                                                                                                                        | otc    | OS   | BBG000N8RCD6   | BBG001SGXVD2     |   9.394300e+04 |           0.0558190 |    0.059000 |      0.056050 |      0.059200 |     0.053100 | 1.65852e+12 |           24 |
| AMYZF  | RECYCLICO BATTERY MATLS                                                                                                                                                         | otc    | OS   | BBG000FVBD30   | BBG001SBYCZ0     |   9.822200e+04 |           0.3984900 |    0.408350 |      0.395000 |      0.420200 |     0.381800 | 1.65852e+12 |           59 |
| GLUC   | GLUCOSE HEALTH INC                                                                                                                                                              | otc    | CS   | BBG000GCNJS8   | BBG001SNWTM3     |   9.428000e+03 |           0.7655800 |    0.861000 |      0.731000 |      0.861000 |     0.731000 | 1.65852e+12 |           12 |
| ECTM   | ECA MARCELLUS TRUST I                                                                                                                                                           | otc    | UNIT | BBG000QM53N5   | BBG001T7V1Z5     |   2.137500e+04 |           2.2802000 |    2.280000 |      2.290000 |      2.300000 |     2.250000 | 1.65852e+12 |           41 |
| MEDIF  | MEDIPHARM LABS CORP                                                                                                                                                             | otc    | OS   | BBG00HZZLFW4   | BBG00HZZLFX3     |   4.209400e+04 |           0.0577990 |    0.067000 |      0.056350 |      0.067000 |     0.053900 | 1.65852e+12 |           18 |
| GRCAF  | GOLD SPRNGS RESOURCE CORP                                                                                                                                                       | otc    | OS   | BBG000BPF3M4   | BBG001SFRL43     |   1.000000e+04 |           0.1464000 |    0.146400 |      0.146400 |      0.146400 |     0.146400 | 1.65852e+12 |            1 |
| SLGWF  | SLANG WORLDWIDE INC                                                                                                                                                             | otc    | OS   | BBG00N5QF0L6   | BBG00N5QF1M3     |   3.600800e+04 |           0.0804320 |    0.076200 |      0.080000 |      0.081850 |     0.076200 | 1.65852e+12 |            6 |
| ETST   | EARTH SCIENCE TECH INC                                                                                                                                                          | otc    | CS   | BBG002GP6T11   | BBG002GP6TW7     |   1.066700e+04 |           0.0230920 |    0.025350 |      0.021100 |      0.025350 |     0.021100 | 1.65852e+12 |            2 |
| VLEEY  | VALEO SE S/ADR                                                                                                                                                                  | otc    | ADRC | BBG000BK7549   | BBG001S87289     |   3.323100e+04 |          10.1457000 |   10.170000 |     10.120000 |     10.250000 |    10.070000 | 1.65852e+12 |          104 |
| XISHY  | XINYI SOLAR HLDG UNSP/ADR                                                                                                                                                       | otc    | ADRC | BBG00QXRHG84   | BBG00QXRHH00     |   4.851000e+03 |          31.0569000 |   31.400000 |     30.993000 |     31.400000 |    30.970000 | 1.65852e+12 |           18 |
| ADDYY  | ADIDAS AG S/ADR                                                                                                                                                                 | otc    | ADRC | BBG000PX9P13   | BBG001SNFNL6     |   8.305300e+04 |          90.3688000 |   91.420000 |     89.890000 |     91.880000 |    89.760000 | 1.65852e+12 |          769 |
| CURR   | CURE PHARMA HLDG CORP                                                                                                                                                           | otc    | CS   | BBG009CXG2M8   | BBG009CXG2N7     |   1.354420e+05 |           0.3298500 |    0.320000 |      0.340000 |      0.340000 |     0.315100 | 1.65852e+12 |          119 |
| GRPS   | GOLD RIVER PRODS INC                                                                                                                                                            | otc    | CS   | BBG000J403M9   | BBG001S64K40     |   2.910000e+04 |           0.0023552 |    0.002200 |      0.002350 |      0.002700 |     0.002200 | 1.65852e+12 |            3 |
| SOTGY  | SUNNY OPTICAL TECH ADR                                                                                                                                                          | otc    | ADRC | BBG0064SSWJ9   | BBG0064SSX89     |   2.570000e+02 |         140.5720000 |  141.660000 |    139.770000 |    141.660000 |   139.080000 | 1.65852e+12 |           16 |
| CBULF  | GRATOMIC INC                                                                                                                                                                    | otc    | OS   | BBG000TQZ9B0   | BBG001T0D135     |   1.714050e+05 |           0.2000400 |    0.200000 |      0.201800 |      0.201800 |     0.200000 | 1.65852e+12 |           26 |
| RMHB   | ROCKY MTN HIGH BRAND INC                                                                                                                                                        | otc    | CS   | BBG000CM3WP6   | BBG001SGB561     |   8.486900e+04 |           0.0260130 |    0.027250 |      0.026750 |      0.029400 |     0.025000 | 1.65852e+12 |           10 |
| ZAIRF  | ZINC8 ENERGY SOLUTINS INC                                                                                                                                                       | otc    | OS   | BBG00KT8NYW1   | BBG00KT8NYX0     |   1.660040e+05 |           0.1517300 |    0.162900 |      0.156800 |      0.176000 |     0.130000 | 1.65852e+12 |           41 |
| WCBR   | WisdomTree Cybersecurity Fund                                                                                                                                                   | stocks | ETF  | BBG00YZ669B5   | BBG00YZ66B57     |   3.208000e+03 |          19.1943000 |   19.495000 |     19.050000 |     19.600000 |    18.951000 | 1.65852e+12 |           59 |
| PALC   | Pacer Lunt Large Cap Multi-Factor Alternator ETF                                                                                                                                | stocks | ETF  | BBG00VP235V0   | BBG00VP236R3     |   5.113300e+04 |          35.3845000 |   35.400000 |     35.310000 |     35.560000 |    35.110000 | 1.65852e+12 |          230 |
| JBSS   | John B. Sanfilippo & SON                                                                                                                                                        | stocks | CS   | BBG000CHPMY5   | BBG001S6WMC6     |   4.800200e+04 |          74.2103000 |   73.590000 |     74.410000 |     74.670000 |    73.590000 | 1.65852e+12 |         1141 |
| IR     | Ingersoll Rand Inc. Common Stock                                                                                                                                                | stocks | CS   | BBG002R1CW27   | BBG002R1CW36     |   2.200156e+06 |          44.5779000 |   45.000000 |     44.540000 |     45.190000 |    44.180000 | 1.65852e+12 |        22818 |
| ICVT   | iShares Convertible Bond ETF                                                                                                                                                    | stocks | ETF  | BBG009BKDML7   | BBG009BKDMM6     |   7.601010e+05 |          71.6487000 |   72.090000 |     71.620000 |     72.270000 |    71.270000 | 1.65852e+12 |         4379 |
| SATS   | EchoStar Corporation                                                                                                                                                            | stocks | CS   | BBG000TGLV00   | BBG001T048Z5     |   1.992820e+05 |          19.4878000 |   19.440000 |     19.590000 |     19.620000 |    19.310000 | 1.65852e+12 |         4056 |
| DFJ    | WisdomTree Japan SmallCap Dividend Fund                                                                                                                                         | stocks | ETF  | BBG000BT2803   | BBG001SHL7C7     |   4.072000e+03 |          61.1468000 |   61.180000 |     61.000000 |     61.450000 |    60.812100 | 1.65852e+12 |           73 |
| KPRX   | Kiora Pharmaceuticals, Inc. Common Stock                                                                                                                                        | stocks | CS   | BBG0083Y1PH9   | BBG0083Y1Q78     |   2.654362e+05 |           6.1264000 |    6.400000 |      5.920000 |      6.400000 |     5.644000 | 1.65852e+12 |        14095 |
| PETV   | PetVivo Holdings, Inc. Common Stock                                                                                                                                             | stocks | CS   | BBG00363ZK61   | BBG00363ZKY0     |   1.427600e+04 |           1.7453000 |    1.839900 |      1.820000 |      1.840000 |     1.730000 | 1.65852e+12 |           42 |
| IBTJ   | iShares iBonds Dec 2029 Term Treasury ETF                                                                                                                                       | stocks | ETF  | BBG00RYR4QK9   | BBG00RYR4R90     |   4.841100e+04 |          23.0541000 |   23.100000 |     23.056300 |     23.100000 |    23.020000 | 1.65852e+12 |           19 |
| CORP   | PIMCO Investment Grade Corporate Bond Index Exchange-Traded Fund                                                                                                                | stocks | ETF  | BBG001660LJ0   | BBG001TCGQY7     |   6.711300e+04 |          98.6670000 |   98.680000 |     98.690000 |     99.140000 |    98.340000 | 1.65852e+12 |          693 |
| FXR    | First Trust Industrials/Producer Durables AlphaDEX Fund                                                                                                                         | stocks | ETF  | BBG000BCZ751   | BBG001S7RKF1     |   4.523900e+04 |          51.2362000 |   51.520000 |     51.140000 |     51.710000 |    50.850400 | 1.65852e+12 |          493 |
| SFET   | Safe-T Group Ltd. American Depositary Share                                                                                                                                     | stocks | ADRC | BBG00GVM3KM9   | BBG00GVM3LB9     |   6.370000e+04 |           0.4929500 |    0.509700 |      0.484000 |      0.515000 |     0.480000 | 1.65852e+12 |          179 |
| LD     | iPath Bloomberg Lead Subindex Total Return ETN due June 24, 2038                                                                                                                | stocks | ETN  | BBG000GKPP40   | BBG001T2YDC2     |   9.525000e+03 |          43.3339000 |   42.800000 |     42.750100 |     44.500000 |    42.750100 | 1.65852e+12 |          117 |
| AFL    | Aflac Inc.                                                                                                                                                                      | stocks | CS   | BBG000BBBNC6   | BBG001S5NGJ4     |   1.802352e+06 |          55.1574000 |   55.220000 |     55.280000 |     55.610000 |    54.750000 | 1.65852e+12 |        23631 |
| ASAN   | Asana, Inc. Class A Common Stock                                                                                                                                                | stocks | CS   | BBG00WYHL732   | BBG00WYHL741     |   2.404757e+06 |          18.1661000 |   19.510000 |     17.660000 |     20.020000 |    17.510000 | 1.65852e+12 |        25593 |
| NRC    | National Research Corporation Common Stock (Delaware)                                                                                                                           | stocks | CS   | BBG004K1F8W7   | BBG004K1F9R1     |   2.200500e+04 |          37.9060000 |   38.230000 |     37.710000 |     38.590000 |    37.610000 | 1.65852e+12 |          660 |
| VINO   | Gaucho Group Holdings, Inc. Common Stock                                                                                                                                        | stocks | CS   | BBG000J23D93   | BBG001T3H1B6     |   1.616113e+06 |           0.3926700 |    0.399700 |      0.388000 |      0.409600 |     0.368700 | 1.65852e+12 |         2530 |
| TLIS   | Talis Biomedical Corporation Common Stock                                                                                                                                       | stocks | CS   | BBG00Y9D99K6   | BBG00Y9D99L5     |   2.423400e+04 |           0.8376900 |    0.854000 |      0.831300 |      0.854000 |     0.819900 | 1.65852e+12 |          150 |
| NXP    | NUVEEN SELECT TAX-FREE INC                                                                                                                                                      | stocks | FUND | BBG000CPMYC8   | BBG001S70MD8     |   9.424400e+04 |          13.9555000 |   13.890000 |     13.860000 |     14.080000 |    13.860000 | 1.65852e+12 |          399 |
| HCICU  | Hennessy Capital Investment Corp. V Units                                                                                                                                       | stocks | UNIT | BBG00YN8HR22   | BBG00YN8HS02     |   6.832000e+03 |           9.9894000 |   10.000000 |     10.000000 |     10.000000 |    10.000000 | 1.65852e+12 |           10 |
| LBRDK  | Liberty Broadband Corporation Class C                                                                                                                                           | stocks | CS   | BBG006GNSZW5   | BBG006GNSZX4     |   6.576160e+05 |         123.1818000 |  123.910000 |    123.320000 |    124.810000 |   122.090000 | 1.65852e+12 |        16184 |
| BCX    | BLACKROCK RESOURCES & COMMODITIES STRATEGY TRUST                                                                                                                                | stocks | FUND | BBG0019VQB06   | BBG001TG5XN7     |   2.389140e+05 |           8.6510000 |    8.670000 |      8.550000 |      8.740000 |     8.540000 | 1.65852e+12 |          979 |
| PLAG   | Planet Green Holdings Corp.                                                                                                                                                     | stocks | CS   | BBG000GL6QR2   | BBG001SPJN46     |   5.726000e+03 |           0.8239100 |    0.800000 |      0.800000 |      0.840000 |     0.800000 | 1.65852e+12 |           25 |
| RM     | REGIONAL MANAGEMENT CORP                                                                                                                                                        | stocks | CS   | BBG001PJFM76   | BBG001V1C4G1     |   2.009400e+04 |          39.8331000 |   40.320000 |     39.870000 |     40.399400 |    39.540000 | 1.65852e+12 |          650 |
| UGA    | United States Gasoline Fund, LP                                                                                                                                                 | stocks | ETV  | BBG000R22FF3   | BBG001SSZR43     |   6.915600e+04 |          61.1089000 |   60.380000 |     60.770000 |     61.900000 |    60.380000 | 1.65852e+12 |          719 |
| GIW    | GigInternational1, Inc. Common Stock                                                                                                                                            | stocks | CS   | BBG0105WRBH0   | BBG0105WRBK6     |   4.909000e+03 |          10.0802000 |   10.080000 |     10.090000 |     10.090000 |    10.080000 | 1.65852e+12 |           53 |
| RCS    | PIMCO STRATEGIC INCOME FUND, INC.                                                                                                                                               | stocks | FUND | BBG000B9ZPB1   | BBG001S5QG01     |   4.906800e+04 |           5.2035000 |    5.210000 |      5.230000 |      5.230000 |     5.160000 | 1.65852e+12 |          173 |
| TSVT   | 2seventy bio, Inc. Common Stock                                                                                                                                                 | stocks | CS   | BBG00YV1J622   | BBG00YV1J631     |   3.508870e+05 |          13.1760000 |   13.340000 |     13.180000 |     13.355000 |    12.790000 | 1.65852e+12 |         5003 |
| UCO    | ProShares Ultra Bloomberg Crude Oil                                                                                                                                             | stocks | ETV  | BBG000CSVPZ6   | BBG001T0D0D6     |   1.625203e+06 |          36.2980000 |   36.120000 |     35.480000 |     37.340000 |    35.370000 | 1.65852e+12 |         9355 |
| NPCE   | Neuropace, Inc. Common Stock                                                                                                                                                    | stocks | CS   | BBG001KWZD52   | BBG001V0SVB1     |   2.392000e+04 |           5.7287000 |    5.760000 |      5.550000 |      5.840100 |     5.500000 | 1.65852e+12 |          689 |
| NUVB   | Nuvation Bio Inc.                                                                                                                                                               | stocks | CS   | BBG00VHJ0CC1   | BBG00VHJ0CD0     |   8.301190e+05 |           3.1079000 |    3.240000 |      3.060000 |      3.270000 |     3.060000 | 1.65852e+12 |         5073 |
| ABOS   | Acumen Pharmaceuticals, Inc. Common Stock                                                                                                                                       | stocks | CS   | BBG0058YJ317   | BBG0058YJ326     |   3.890600e+04 |           6.3178000 |    6.210000 |      6.280000 |      6.490000 |     6.120000 | 1.65852e+12 |          824 |
| LEXX   | Lexaria Bioscience Corp. Common Stock                                                                                                                                           | stocks | CS   | BBG000JQHT18   | BBG001SBJ1Q1     |   7.185800e+04 |           2.7739000 |    2.835000 |      2.800000 |      2.849900 |     2.740000 | 1.65852e+12 |          217 |
| TRMK   | Trustmark Corp                                                                                                                                                                  | stocks | CS   | BBG000C3SB31   | BBG001S6JGT6     |   2.084170e+05 |          30.2663000 |   30.500000 |     30.310000 |     30.610000 |    29.130000 | 1.65852e+12 |         3871 |
| ILTB   | iShares Core 10+ Year USD Bond ETFof iShares Trust                                                                                                                              | stocks | ETF  | BBG000PGL182   | BBG001T5MKJ5     |   3.015300e+04 |          57.4814000 |   57.450000 |     57.530000 |     57.700000 |    57.240000 | 1.65852e+12 |          212 |
| ESOA   | Energy Services of America Corporation Common Stock                                                                                                                             | stocks | CS   | BBG000FSY520   | BBG001SNQ748     |   1.128500e+04 |           2.0679000 |    2.060000 |      2.050000 |      2.089900 |     2.050000 | 1.65852e+12 |           93 |
| FRDM   | Freedom 100 Emerging Markets ETF                                                                                                                                                | stocks | ETF  | BBG00P7KKM01   | BBG00P7KKMR2     |   3.050100e+04 |          28.0167000 |   28.390000 |     27.840000 |     28.410000 |    27.788300 | 1.65852e+12 |          278 |
| FRLA   | Fortune Rise Acquisition Corporation Class A Common Stock                                                                                                                       | stocks | CS   | BBG011XR74K2   | BBG011XR74L1     |   1.925400e+04 |          10.1000000 |   10.100000 |     10.100000 |     10.100000 |    10.100000 | 1.65852e+12 |           21 |
| COUR   | Coursera, Inc.                                                                                                                                                                  | stocks | CS   | BBG002WLDMW5   | BBG002WLDMY3     |   5.413930e+05 |          16.8840000 |   17.360000 |     16.780000 |     17.590000 |    16.640000 | 1.65852e+12 |         6805 |
| HESM   | Hess Midstream LP Class A Share representing a limited partner Interest                                                                                                         | stocks | CS   | BBG00R02H8D5   | BBG00R02H8F3     |   2.325260e+05 |          28.2473000 |   28.600000 |     28.110000 |     28.939000 |    27.895000 | 1.65852e+12 |         4285 |
| NMR    | Nomura Holdings, Inc                                                                                                                                                            | stocks | ADRC | BBG000BZPXB7   | BBG001S6HZT6     |   5.948780e+05 |           3.7250000 |    3.720000 |      3.740000 |      3.740000 |     3.710000 | 1.65852e+12 |         2847 |
| JPMB   | JPMorgan USD Emerging Markets Sovereign Bond ETF                                                                                                                                | stocks | ETF  | BBG00JWVJXF5   | BBG00JWVJY45     |   9.340000e+02 |          38.6306000 |   38.660000 |     38.727500 |     38.727500 |    38.600000 | 1.65852e+12 |           13 |
| EVH    | Evolent Health, Inc Class A Common Stock                                                                                                                                        | stocks | CS   | BBG005CHLM96   | BBG005CHLMB3     |   6.317380e+05 |          33.1841000 |   33.840000 |     32.980000 |     34.160000 |    32.720000 | 1.65852e+12 |         8923 |
| PLTM   | GraniteShares Platinum Shares                                                                                                                                                   | stocks | ETV  | BBG00JRYLN55   | BBG00JRYLNX4     |   1.476500e+04 |           8.6247000 |    8.670000 |      8.529900 |      8.710000 |     8.529900 | 1.65852e+12 |          130 |
| SWTX   | SpringWorks Therapeutics, Inc. Common Stock                                                                                                                                     | stocks | CS   | BBG00Q011TW9   | BBG00Q011TX8     |   4.652730e+05 |          28.6001000 |   30.600000 |     28.270000 |     31.020000 |    28.130000 | 1.65852e+12 |         7486 |
| WSTG   | Wayside Technology Group, Inc.                                                                                                                                                  | stocks | CS   | BBG000BCJVD7   | BBG001S5T4D1     |   1.632600e+04 |          31.0438000 |   29.560000 |     31.110000 |     31.610000 |    29.560000 | 1.65852e+12 |          234 |
| BITF   | Bitfarms Ltd. Common Stock                                                                                                                                                      | stocks | CS   | BBG00PZTS4J3   | BBG00PGMZVM7     |   5.217681e+06 |           1.3973000 |    1.540000 |      1.340000 |      1.570000 |     1.300000 | 1.65852e+12 |         7870 |
| VTWG   | Vanguard Russell 2000 Growth ETF                                                                                                                                                | stocks | ETF  | BBG0016LDNH1   | BBG001TCH7Y8     |   1.518600e+04 |         161.1353000 |  163.210000 |    160.040000 |    163.700000 |   159.070100 | 1.65852e+12 |          402 |
| CIG.C  | Companhia Energetica De Minas Gerais-CEMIG                                                                                                                                      | stocks | ADRC | BBG000GN0XD4   | BBG001STLY96     |   3.215200e+04 |           3.0174000 |    2.960000 |      3.050000 |      3.050000 |     2.960000 | 1.65852e+12 |          187 |
| NEOG   | Neogen Corp                                                                                                                                                                     | stocks | CS   | BBG000C1BCK2   | BBG001S67B47     |   1.054530e+06 |          22.1644000 |   22.650000 |     22.010000 |     22.780000 |    21.800000 | 1.65852e+12 |        11516 |
| BLBX   | Blackboxstocks Inc. Common Stock                                                                                                                                                | stocks | CS   | BBG007GGHNQ4   | BBG007GGHNR3     |   4.620000e+04 |           1.3362000 |    1.410100 |      1.350000 |      1.410100 |     1.280000 | 1.65852e+12 |          169 |
| IVRA   | Invesco Real Assets ESG ETF                                                                                                                                                     | stocks | ETF  | BBG00YJV0GW5   | BBG00YJV0HQ0     |   1.940000e+02 |          13.9084000 |   13.880000 |     13.934800 |     13.934800 |    13.880000 | 1.65852e+12 |           21 |
| RFDI   | First Trust RiverFront Dynamic Developed International ETF                                                                                                                      | stocks | ETF  | BBG00CNPNL73   | BBG00CNPNL82     |   3.247000e+03 |          55.4403000 |   55.730100 |     55.210000 |     55.730100 |    55.030000 | 1.65852e+12 |           32 |
| MHD    | Blackrock Muniholdings Fund, Inc.                                                                                                                                               | stocks | FUND | BBG000BBX3V5   | BBG001S66J14     |   1.066150e+05 |          13.0486000 |   13.140000 |     13.040000 |     13.150000 |    12.950000 | 1.65852e+12 |          528 |
| JBLU   | JetBlue Airways Corp                                                                                                                                                            | stocks | CS   | BBG000BRQ6L2   | BBG001S9YPC1     |   7.748623e+06 |           8.4479000 |    8.660000 |      8.370000 |      8.700000 |     8.330000 | 1.65852e+12 |        28704 |
| REK    | ProShares Short Real Estate                                                                                                                                                     | stocks | ETF  | BBG000QJ4C08   | BBG001T7CNH8     |   6.267200e+04 |          18.6432000 |   18.680000 |     18.600000 |     18.730000 |    18.460000 | 1.65852e+12 |          358 |
| DIVS   | SmartETFs Dividend Builder ETF                                                                                                                                                  | stocks | ETF  | BBG00ZV186P6   | BBG00ZV187J1     |   2.615000e+03 |          23.5739000 |   23.730100 |     23.567400 |     23.730100 |    23.524900 | 1.65852e+12 |           10 |
| FMN    | Federated Hermes Premier Municipal Income Fund                                                                                                                                  | stocks | FUND | BBG000CTZS48   | BBG001SFSHM1     |   4.205700e+04 |          11.3752000 |   11.340000 |     11.420000 |     11.440000 |    11.320000 | 1.65852e+12 |          226 |
| HYZN   | Hyzon Motors Inc. Class A Common Stock                                                                                                                                          | stocks | CS   | BBG00Y2JHBG1   | BBG00Y2JHBH0     |   8.693310e+05 |           3.5858000 |    3.730000 |      3.580000 |      3.730000 |     3.525000 | 1.65852e+12 |         6772 |
| NOCT   | Innovator Growth-100 Power Buffer ETF- October                                                                                                                                  | stocks | ETF  | BBG00QFP0615   | BBG00QFP06T5     |   1.540000e+03 |          37.0278000 |   37.111400 |     36.875000 |     37.111400 |    36.820000 | 1.65852e+12 |           11 |
| KLR    | Kaleyra, Inc.                                                                                                                                                                   | stocks | CS   | BBG00J7STCT0   | BBG00J7STCX5     |   3.402380e+05 |           2.2609000 |    2.330000 |      2.180000 |      2.460000 |     2.140000 | 1.65852e+12 |         2512 |
| ES     | Eversource Energy                                                                                                                                                               | stocks | CS   | BBG000BQ87N0   | BBG001S5TRL1     |   1.149714e+06 |          83.8168000 |   83.410000 |     84.060000 |     84.080000 |    83.220000 | 1.65852e+12 |        17699 |
| ETW    | Eaton Vance Tax-Managed Global Buy-Write Opportunities Fund                                                                                                                     | stocks | FUND | BBG000BQJ0Y1   | BBG001SGGKG2     |   1.965440e+05 |           8.6731000 |    8.710000 |      8.630000 |      8.711600 |     8.610000 | 1.65852e+12 |          746 |
| IDAT   | iShares Cloud 5G and Tech ETF                                                                                                                                                   | stocks | ETF  | BBG00ZHZJFG5   | BBG00ZHZJG91     |   2.600000e+01 |          21.7510000 |   21.725400 |     21.725400 |     21.725400 |    21.725400 | 1.65852e+12 |           14 |
| EIRL   | iShares MSCI Ireland ETF                                                                                                                                                        | stocks | ETF  | BBG000QVLXY9   | BBG001S56P85     |   3.458000e+03 |          41.1960000 |   41.506500 |     41.217600 |     41.550000 |    41.120000 | 1.65852e+12 |           35 |
| SEA    | U.S. Global Sea to Sky Cargo ETF                                                                                                                                                | stocks | ETF  | BBG013Y2V4Z9   | BBG013Y2V5W9     |   1.199000e+03 |          19.2305000 |   19.250000 |     19.090200 |     19.250000 |    19.090200 | 1.65852e+12 |           11 |
| BHR    | Braemar Hotels & Resorts Inc. Common Stock                                                                                                                                      | stocks | CS   | BBG004PW84G9   | BBG004PW8567     |   5.089010e+05 |           5.1547000 |    5.300000 |      5.120000 |      5.320000 |     5.025000 | 1.65852e+12 |         4114 |
| PBUG   | Pacer iPath Gold Trendpilot Exchange-Traded Notes                                                                                                                               | stocks | ETN  | BBG011F5ZCT2   | BBG011F5ZDP4     |   7.580000e+02 |          31.5497000 |   31.990000 |     31.010000 |     32.000000 |    31.010000 | 1.65852e+12 |           17 |
| RVPH   | Reviva Pharmaceuticals Holdings, Inc. Common Stock                                                                                                                              | stocks | CS   | BBG00LJZ8FW8   | BBG00LJZ8FX7     |   4.395700e+04 |           0.9936400 |    1.000000 |      0.981000 |      1.030000 |     0.969900 | 1.65852e+12 |          280 |
| DFSD   | Dimensional Short-Duration Fixed Income ETF                                                                                                                                     | stocks | ETF  | BBG01254JD10   | BBG01254JDW6     |   2.613210e+05 |          47.1724000 |   47.150000 |     47.190000 |     47.348200 |    47.100000 | 1.65852e+12 |          767 |
| VICE   | AdvisorShares Vice ETF                                                                                                                                                          | stocks | ETF  | BBG00JGGJ4N1   | BBG00JGGJ5C0     |   3.350000e+02 |          26.4220000 |   26.470000 |     26.378800 |     26.470000 |    26.378800 | 1.65852e+12 |           12 |
| DRIV   | Global X Autonomous & Electric Vehicles ETF                                                                                                                                     | stocks | ETF  | BBG00KLHY7D7   | BBG00KLHY836     |   1.464600e+05 |          22.9623000 |   23.360000 |     22.760000 |     23.400000 |    22.671200 | 1.65852e+12 |         1067 |
| NTCO   | Natura &Co Holding S.A. American Depositary Shares (each representing two Common Shares)                                                                                        | stocks | ADRC | BBG00R4ZKPN5   | BBG00R4ZKQD4     |   8.189890e+05 |           5.8511000 |    5.870000 |      5.810000 |      5.975000 |     5.710000 | 1.65852e+12 |         4726 |
| PJP    | Invesco Dynamic Pharmaceuticals ETF                                                                                                                                             | stocks | ETF  | BBG000DVJ728   | BBG001SN6GP7     |   3.040800e+04 |          75.3472000 |   76.300000 |     75.290000 |     76.300000 |    75.080000 | 1.65852e+12 |          161 |
| MLPX   | Global X MLP & Energy Infrastructure ETF                                                                                                                                        | stocks | ETF  | BBG004Y67XL7   | BBG004Y67XK8     |   7.993400e+04 |          39.6651000 |   39.930000 |     39.450000 |     40.260000 |    39.230000 | 1.65852e+12 |          796 |
| ECOR   | electroCore, Inc. Common Stock                                                                                                                                                  | stocks | CS   | BBG006R93N12   | BBG006R93QQ8     |   1.031350e+05 |           0.5934200 |    0.613800 |      0.594100 |      0.613800 |     0.585000 | 1.65852e+12 |          210 |
| SFBC   | Sound Financial Bancorp, Inc.                                                                                                                                                   | stocks | CS   | BBG000BD3F92   | BBG001S7WQB6     |   3.770000e+02 |          37.2549000 |   37.230000 |     37.180000 |     37.230000 |    37.180000 | 1.65852e+12 |           22 |
| BYM    | BLACKROCK MUNICIPAL INCOME QUALITY TRUST                                                                                                                                        | stocks | FUND | BBG000BLWDZ8   | BBG001S99KD9     |   4.200800e+04 |          12.8957000 |   12.950000 |     12.890000 |     12.950000 |    12.840000 | 1.65852e+12 |          111 |
| JAGX   | Jaguar Health, Inc.                                                                                                                                                             | stocks | CS   | BBG005Z20406   | BBG005Z205T2     |   4.000561e+06 |           0.3267700 |    0.330000 |      0.317000 |      0.338300 |     0.315000 | 1.65852e+12 |         3704 |
| SLAB   | Silicon Laboratories Inc                                                                                                                                                        | stocks | CS   | BBG000BB99S3   | BBG001S5W492     |   3.075180e+05 |         140.0601000 |  143.830000 |    140.340000 |    143.830000 |   138.800000 | 1.65852e+12 |         8542 |
| CSM    | ProShares Large Cap Core Plus                                                                                                                                                   | stocks | ETF  | BBG000NDJYJ0   | BBG001T551S6     |   7.353000e+03 |          46.8133000 |   47.210000 |     46.800000 |     47.210000 |    46.570000 | 1.65852e+12 |           46 |
| FBIO   | Fortress Biotech, Inc.                                                                                                                                                          | stocks | CS   | BBG001J2MFH6   | BBG001V0GB64     |   2.352790e+05 |           0.9871800 |    1.030000 |      0.950100 |      1.030000 |     0.950000 | 1.65852e+12 |         1172 |
| ALE    | ALLETE, Inc.                                                                                                                                                                    | stocks | CS   | BBG000CYW7N5   | BBG001S77110     |   2.297510e+05 |          58.1986000 |   58.440000 |     58.420000 |     58.590000 |    57.750000 | 1.65852e+12 |         5525 |
| HTH    | HILLTOP HOLDINGS INC.                                                                                                                                                           | stocks | CS   | BBG000GM73Y2   | BBG001SH9HH4     |   9.488340e+05 |          28.2741000 |   28.480000 |     28.200000 |     29.250000 |    27.910000 | 1.65852e+12 |        10712 |
| ADERU  | 26 Capital Acquisition Corp. Unit                                                                                                                                               | stocks | UNIT | BBG00YNKD509   | BBG00YNKD5V5     |   9.010000e+02 |           9.9533000 |    9.970000 |      9.960000 |      9.970000 |     9.950000 | 1.65852e+12 |            7 |
| XLF    | Financial Select Sector SPDR Fund                                                                                                                                               | stocks | ETF  | BBG000BJ29X7   | BBG001S7T223     |   2.574122e+07 |          32.7647000 |   32.990000 |     32.750000 |     33.160000 |    32.500000 | 1.65852e+12 |        63058 |
| XL     | XL Fleet Corp.                                                                                                                                                                  | stocks | CS   | BBG00PP0KRL1   | BBG00PP0KRM0     |   8.976920e+05 |           1.2636000 |    1.300000 |      1.270000 |      1.330000 |     1.240000 | 1.65852e+12 |         4466 |
| DMO    | Western Asset Mortgage Opportunity Fund Inc.                                                                                                                                    | stocks | FUND | BBG000D912M5   | BBG001T0VQ03     |   3.544900e+04 |          11.9870000 |   11.920000 |     11.960000 |     12.069900 |    11.900000 | 1.65852e+12 |          216 |
| QLGN   | Qualigen Therapeutics, Inc. Common Stock                                                                                                                                        | stocks | CS   | BBG008D03RS4   | BBG008D03RT3     |   3.060630e+05 |           0.4476300 |    0.468000 |      0.453800 |      0.468000 |     0.431300 | 1.65852e+12 |          623 |
| WAB    | Wabtec Inc.                                                                                                                                                                     | stocks | CS   | BBG000BDD940   | BBG001S5XBT3     |   7.725530e+05 |          85.8795000 |   86.320000 |     85.910000 |     87.090000 |    85.110000 | 1.65852e+12 |        13162 |
| ADIL   | Adial Pharmaceuticals, Inc                                                                                                                                                      | stocks | CS   | BBG00HNGDTR4   | BBG00HNGDTS3     |   1.677467e+06 |           0.7623000 |    0.810100 |      0.740200 |      0.820000 |     0.730400 | 1.65852e+12 |         2663 |
| LXEH   | Lixiang Education Holding Co., Ltd. American Depositary Shares                                                                                                                  | stocks | ADRC | BBG00X7L4LC9   | BBG00X7L4M64     |   5.130000e+02 |           2.1425000 |    2.160000 |      2.250000 |      2.250000 |     2.070000 | 1.65852e+12 |           11 |
| BFAM   | BRIGHT HORIZONS FAMILY SOLUTIONS INC.                                                                                                                                           | stocks | CS   | BBG003LFWP05   | BBG003LFWPT4     |   4.728110e+05 |          90.3344000 |   90.670000 |     90.480000 |     92.310000 |    89.750000 | 1.65852e+12 |         7752 |
| PTEU   | Pacer TrendpilotTM European Index ETF                                                                                                                                           | stocks | ETF  | BBG00BLVPY10   | BBG00BLVPY01     |   9.726000e+03 |          22.2294000 |   22.160000 |     22.340000 |     22.350000 |    22.160000 | 1.65852e+12 |           25 |
| ING    | ING Groep N.V. American Depositary Shares                                                                                                                                       | stocks | ADRC | BBG000BM0LB9   | BBG001SB1FR8     |   4.167866e+06 |           9.2212000 |    9.240000 |      9.210000 |      9.300000 |     9.155000 | 1.65852e+12 |        10191 |
| ADRT.U | Ault Disruptive Technologies Corporation Units, each consisting of one share of Common Stock, and three-fourths of one Redeemable Warrant to purchase one share of Common Stock | stocks | UNIT | BBG013DRFC78   | BBG013DRFD21     |   1.800000e+03 |          10.0400000 |   10.040000 |     10.040000 |     10.040000 |    10.040000 | 1.65852e+12 |           18 |
| FDTS   | First Trust Developed Markets ex-US Small Cap AlphaDEX Fund                                                                                                                     | stocks | ETF  | BBG002NCPVF1   | BBG002NCPW50     |   7.900000e+01 |          37.3275000 |   35.650000 |     35.650000 |     35.650000 |    35.650000 | 1.65852e+12 |            4 |
| PPI    | AXS Astoria Inflation Sensitive ETF                                                                                                                                             | stocks | ETF  | BBG0149PN8N7   | BBG0149PN9S0     |   1.203200e+04 |          23.5711000 |   24.000000 |     23.540000 |     24.050000 |    23.420000 | 1.65852e+12 |           41 |
| MXI    | iShares Global Materials ETF                                                                                                                                                    | stocks | ETF  | BBG000G81ZY8   | BBG001SNVXP2     |   2.884600e+04 |          72.9500000 |   73.610000 |     72.650000 |     73.830000 |    72.488200 | 1.65852e+12 |          529 |
| TBLD   | Thornburg Income Builder Opportunities Trust Common Stock                                                                                                                       | stocks | CS   | BBG011K2MTD7   | BBG011K2MTF5     |   8.614200e+04 |          15.1383000 |   15.390000 |     15.060000 |     15.530000 |    15.000000 | 1.65852e+12 |          506 |
| BMAQ   | Blockchain Moon Acquisition Corp. Common Stock                                                                                                                                  | stocks | CS   | BBG012Q928N1   | BBG012Q928P9     |   3.094000e+03 |           9.9200000 |    9.920000 |      9.920000 |      9.920000 |     9.920000 | 1.65852e+12 |            6 |
| NSPI   | Nationwide S&P 500 Risk Managed Income ETF                                                                                                                                      | stocks | ETF  | BBG0145JDM28   | BBG0145JDMY3     |   1.341000e+03 |          21.9572000 |   22.000000 |     21.931100 |     22.000000 |    21.900000 | 1.65852e+12 |           22 |
| PNQI   | Invesco NASDAQ Internet ETF                                                                                                                                                     | stocks | ETF  | BBG000DDK8Y9   | BBG001T10BD4     |   1.116800e+04 |         128.9977000 |  131.715000 |    127.980000 |    131.715000 |   127.447900 | 1.65852e+12 |          413 |
| IEV    | iShares Europe ETF                                                                                                                                                              | stocks | ETF  | BBG000BXV152   | BBG001SFGXD9     |   2.133990e+05 |          43.2141000 |   43.380000 |     43.050000 |     43.630000 |    42.900000 | 1.65852e+12 |         1292 |
| QUOT   | Quotient Technology Inc                                                                                                                                                         | stocks | CS   | BBG001QYNR63   | BBG001V1S8S2     |   5.757730e+05 |           2.9979000 |    3.120000 |      2.950000 |      3.180000 |     2.905000 | 1.65852e+12 |         4221 |
| SPWR   | SunPower Corporation Common Stock                                                                                                                                               | stocks | CS   | BBG000FVQ185   | BBG001SN52J6     |   1.688026e+06 |          16.8192000 |   17.150000 |     16.440000 |     17.510000 |    16.365000 | 1.65852e+12 |        18462 |
| FAD    | First Trust Multi Cap Growth AlphaDEX Fund                                                                                                                                      | stocks | ETF  | BBG000R6FJ99   | BBG001ST6952     |   1.043000e+03 |          97.0568000 |   97.470000 |     96.460000 |     97.470000 |    96.460000 | 1.65852e+12 |           30 |
| TRV    | The Travelers Companies, Inc.                                                                                                                                                   | stocks | CS   | BBG000BJ81C1   | BBG001S5R103     |   8.809920e+05 |         156.6688000 |  157.140000 |    156.420000 |    158.209900 |   155.480000 | 1.65852e+12 |        18385 |
| ORC    | Orchid Island Capital, Inc.                                                                                                                                                     | stocks | CS   | BBG001P2KSC8   | BBG001V18CW0     |   3.011794e+05 |          15.2995000 |   15.450000 |     15.350000 |     15.450000 |    15.100000 | 1.65852e+12 |         6256 |
| EWA    | iShares MSCI Australia ETF                                                                                                                                                      | stocks | ETF  | BBG000BDNJ29   | BBG001S6ZW40     |   4.012362e+06 |          21.8499000 |   21.950000 |     21.750000 |     22.070000 |    21.660000 | 1.65852e+12 |         9217 |
| ETX    | EATON VANCE MUNICIPAL INCOME 2028 TERM TRUST                                                                                                                                    | stocks | FUND | BBG003PQCQ52   | BBG003PQCQX1     |   4.358000e+03 |          20.0524000 |   20.000000 |     20.050000 |     20.180000 |    19.990000 | 1.65852e+12 |           29 |
| CNXN   | PC Connection Inc                                                                                                                                                               | stocks | CS   | BBG000BX74M4   | BBG001S96MG5     |   2.743300e+04 |          45.9149000 |   46.460000 |     45.760000 |     46.460000 |    45.550000 | 1.65852e+12 |         1967 |
| HEDJ   | WisdomTree Europe Hedged Equity Fund                                                                                                                                            | stocks | ETF  | BBG000Q3NW62   | BBG001T6DPF6     |   3.122400e+04 |          68.1128000 |   68.560000 |     68.050000 |     68.680000 |    67.863000 | 1.65852e+12 |          361 |
| PTNR   | Partner Communications                                                                                                                                                          | stocks | ADRC | BBG000DGFK99   | BBG001SDF181     |   7.370000e+02 |           7.8217000 |    7.780000 |      7.790000 |      7.860100 |     7.780000 | 1.65852e+12 |           12 |
| SUPV   | Grupo Supervielle S.A.                                                                                                                                                          | stocks | CS   | BBG00BTHHZ55   | BBG00BTHJ014     |   9.837650e+05 |           1.3954000 |    1.340000 |      1.490000 |      1.490000 |     1.270000 | 1.65852e+12 |         2450 |
| DC     | Dakota Gold Corp.                                                                                                                                                               | stocks | CS   | BBG0120W17D2   | BBG0120W17F0     |   3.026750e+05 |           3.7176000 |    3.710000 |      3.780000 |      3.800000 |     3.580000 | 1.65852e+12 |         3170 |
| KRMD   | KORU Medical Systems, Inc. Common Stock                                                                                                                                         | stocks | CS   | BBG000D94144   | BBG001S7QTX3     |   6.975000e+03 |           2.6695000 |    2.820000 |      2.580000 |      2.820000 |     2.580000 | 1.65852e+12 |           86 |
| GDX    | VanEck Gold Miners ETF                                                                                                                                                          | stocks | ETF  | BBG000PLNQN7   | BBG001SR42Z0     |   2.742484e+07 |          25.8403000 |   25.850000 |     25.410000 |     26.540000 |    25.300000 | 1.65852e+12 |       130329 |
| GIII   | G-Iii Apparel Group Ltd                                                                                                                                                         | stocks | CS   | BBG000C2YZ60   | BBG001S6F0Z8     |   4.963910e+05 |          22.5006000 |   22.730000 |     22.530000 |     23.160000 |    22.160000 | 1.65852e+12 |         8947 |
| BIOC   | Biocept, Inc.                                                                                                                                                                   | stocks | CS   | BBG000RRRD66   | BBG001T0B985     |   6.617600e+04 |           1.0191000 |    1.070000 |      1.005000 |      1.070000 |     1.000000 | 1.65852e+12 |          233 |
| FRSH   | Freshworks Inc. Class A Common Stock                                                                                                                                            | stocks | CS   | BBG0029KYGL5   | BBG0029KYGM4     |   1.193351e+06 |          13.2429000 |   13.620000 |     13.030000 |     14.000000 |    12.960000 | 1.65852e+12 |        12487 |
| AAWW   | Atlas Air Worldwide Holdings, Inc.                                                                                                                                              | stocks | CS   | BBG000Q57YP0   | BBG001SM7BG9     |   5.669320e+05 |          69.1280000 |   68.560000 |     69.630000 |     69.705000 |    68.350000 | 1.65852e+12 |        11385 |
| CRNC   | Cerence Inc. Common Stock                                                                                                                                                       | stocks | CS   | BBG00MMDJG84   | BBG00MMDJG93     |   2.767640e+05 |          27.0078000 |   27.730000 |     27.060000 |     27.809100 |    26.540000 | 1.65852e+12 |         4480 |
| EVLV   | Evolv Technologies Holdings, Inc. Class A Common Stock                                                                                                                          | stocks | CS   | BBG00W1PPKX4   | BBG00W1PPKY3     |   3.927010e+05 |           2.6740000 |    2.740000 |      2.640000 |      2.790000 |     2.610000 | 1.65852e+12 |         3508 |
| AIRT   | Air T Inc                                                                                                                                                                       | stocks | CS   | BBG000BBH316   | BBG001S5NJZ0     |   5.953000e+03 |          15.3474000 |   14.830000 |     15.100000 |     15.750000 |    14.830000 | 1.65852e+12 |           85 |
| KFYP   | KraneShares CICC China Leaders 100 Index ETF                                                                                                                                    | stocks | ETF  | BBG004VDL3G1   | BBG004VDL3H0     |   7.150000e+02 |          25.1439000 |   25.130000 |     25.179900 |     25.179900 |    25.130000 | 1.65852e+12 |            6 |
| GQRE   | FlexShares Global Quality Real Estate Index Fund                                                                                                                                | stocks | ETF  | BBG005JYLGR5   | BBG005JYLGS4     |   2.939000e+03 |          59.1216000 |   59.150000 |     58.919700 |     59.210000 |    58.750000 | 1.65852e+12 |           47 |
| SNN    | Smith & Nephew plc                                                                                                                                                              | stocks | ADRC | BBG000C2W4K5   | BBG001SDCF10     |   4.566590e+05 |          28.4899000 |   28.590000 |     28.480000 |     28.780000 |    28.255000 | 1.65852e+12 |         6073 |
| MDEV   | First Trust Indxx Medical Devices ETF                                                                                                                                           | stocks | ETF  | BBG011FJ2MH9   | BBG011FJ2NB3     |   2.250000e+02 |          19.3900000 |   19.390000 |     19.223400 |     19.390000 |    19.223400 | 1.65852e+12 |            1 |
| KGHG   | KraneShares Global Carbon Transformation ETF                                                                                                                                    | stocks | ETF  | BBG015WR9FB7   | BBG015WR9G52     |   8.000000e+01 |          22.3363000 |   22.298900 |     22.298900 |     22.298900 |    22.298900 | 1.65852e+12 |            2 |
| COW    | iPath Series B Bloomberg Livestock Subindex Total ReturnSM ETN                                                                                                                  | stocks | ETN  | BBG00JRY3TZ9   | BBG00JRY3VP5     |   7.489000e+03 |          38.6536000 |   38.515500 |     38.880000 |     38.880000 |    38.500100 | 1.65852e+12 |           92 |
| CRSR   | Corsair Gaming, Inc. Common Stock                                                                                                                                               | stocks | CS   | BBG00HMSHL83   | BBG00HMSHL92     |   1.007167e+06 |          13.9447000 |   13.770000 |     13.850000 |     14.800000 |    13.490000 | 1.65852e+12 |        13019 |
| LARK   | Landmark Bancorp Inc                                                                                                                                                            | stocks | CS   | BBG000BJYZD6   | BBG001S742Q4     |   2.001000e+03 |          24.9312000 |   25.070000 |     24.650000 |     25.070000 |    24.650000 | 1.65852e+12 |           93 |
| BB     | BlackBerry Limited                                                                                                                                                              | stocks | CS   | BBG000CR90K4   | BBG001S60618     |   5.561477e+06 |           5.9899000 |    6.210000 |      5.930000 |      6.251000 |     5.880000 | 1.65852e+12 |        14015 |
| ZTEK   | Zentek Ltd. Common Stock                                                                                                                                                        | stocks | CS   | BBG002V3D8N7   | BBG001TFBYY8     |   1.020700e+04 |           2.3251000 |    2.290000 |      2.340000 |      2.360000 |     2.225000 | 1.65852e+12 |          120 |
| SPUU   | Direxion Daily S&P 500 Bull 2X Shares                                                                                                                                           | stocks | ETF  | BBG006K8XS17   | BBG006K8XS08     |   3.655600e+04 |          78.7550000 |   79.790000 |     78.190000 |     80.150000 |    77.280000 | 1.65852e+12 |          339 |
| FTAG   | First Trust Exchange-Traded Fund II First Trust Indxx Global Agriculture ETF                                                                                                    | stocks | ETF  | BBG000QH5HZ2   | BBG001T71BT3     |   4.710000e+02 |          27.7042000 |   27.905000 |     27.570000 |     27.905000 |    27.570000 | 1.65852e+12 |           13 |
| EWX    | SPDR S&P Emerging Markets Small Cap ETF                                                                                                                                         | stocks | ETF  | BBG000FT16M0   | BBG001T2PHC3     |   3.000900e+04 |          48.3581000 |   48.580000 |     48.370000 |     48.709500 |    48.180100 | 1.65852e+12 |          235 |
| MEAR   | BlackRock Short Maturity Municipal Bond ETF                                                                                                                                     | stocks | ETF  | BBG0087DRND2   | BBG0087DRNC3     |   6.530400e+04 |          49.8211000 |   49.820000 |     49.820000 |     49.840000 |    49.810000 | 1.65852e+12 |          244 |
| UEIC   | Universal Electronics Inc                                                                                                                                                       | stocks | CS   | BBG000CZ07G5   | BBG001S77GV4     |   3.693400e+04 |          26.9509000 |   27.600000 |     26.990000 |     28.007400 |    26.700000 | 1.65852e+12 |          998 |
| INAQ.U | Insight Acquisition Corp. Units, each consisting of one share of Class A common stock and one-half of one redeemable warrant                                                    | stocks | UNIT | BBG0123NSPY7   | BBG0123NSQS2     |   1.190000e+04 |           9.9427000 |    9.980000 |      9.890000 |      9.980000 |     9.890000 | 1.65852e+12 |           22 |
| QLD    | ProShares Ultra QQQ                                                                                                                                                             | stocks | ETF  | BBG000PNK918   | BBG001SR6H76     |   4.411633e+06 |          48.4986000 |   49.280000 |     47.810000 |     49.950000 |    47.210000 | 1.65852e+12 |        29814 |
| REMX   | VanEck Rare Earth/Strategic Metals ETF                                                                                                                                          | stocks | ETF  | BBG0018555F4   | BBG001TCVD33     |   3.128600e+04 |          86.5352000 |   87.440000 |     85.510000 |     88.140000 |    85.220000 | 1.65852e+12 |          982 |
| ALTL   | Pacer Lunt Large Cap Alternator ETF                                                                                                                                             | stocks | ETF  | BBG00VP22TF7   | BBG00VP22V71     |   1.947650e+05 |          40.8730000 |   40.910000 |     40.970000 |     41.100000 |    40.740000 | 1.65852e+12 |          597 |
| VBNK   | VersaBank Common Shares                                                                                                                                                         | stocks | CS   | BBG00FZBRHK4   | BBG00FWZQ4P9     |   2.530600e+04 |           7.1020000 |    7.110000 |      7.080000 |      7.130000 |     7.030000 | 1.65852e+12 |          121 |
| CBZ    | CBIZ, Inc.                                                                                                                                                                      | stocks | CS   | BBG000FQD1Z0   | BBG001S8MB55     |   1.337330e+05 |          41.9463000 |   42.000000 |     41.920000 |     42.370000 |    41.680000 | 1.65852e+12 |         2461 |
| SSB    | SouthState Corporation Common Stock                                                                                                                                             | stocks | CS   | BBG000BNPYN9   | BBG001S9J7Z3     |   2.745740e+05 |          79.0238000 |   80.000000 |     79.040000 |     80.185000 |    78.520000 | 1.65852e+12 |         6961 |
| PG     | Procter & Gamble Company                                                                                                                                                        | stocks | CS   | BBG000BR2TH3   | BBG001S5V4L9     |   5.292834e+06 |         142.5100000 |  140.760000 |    143.020000 |    143.170000 |   140.670000 | 1.65852e+12 |        70444 |
| TCOM   | Trip.com Group Limited American Depositary Shares                                                                                                                               | stocks | ADRC | BBG000CWKYS8   | BBG001S6VL20     |   1.961486e+06 |          26.5827000 |   26.770000 |     26.510000 |     27.140000 |    26.290000 | 1.65852e+12 |        15621 |
| PMCB   | PharmaCyte Biotech, Inc. Common Stock                                                                                                                                           | stocks | CS   | BBG000BBVY94   | BBG001S69496     |   3.130070e+05 |           2.4275000 |    2.470000 |      2.420000 |      2.470000 |     2.400000 | 1.65852e+12 |         1204 |
| THCX   | The Cannabis ETF                                                                                                                                                                | stocks | ETF  | BBG00PNSG6L3   | BBG00PNSG7B2     |   2.219090e+05 |           3.7226000 |    3.980000 |      3.760000 |      4.040000 |     3.670000 | 1.65852e+12 |          568 |
| ALPAU  | Alpha Healthcare Acquisition Corp. III Units                                                                                                                                    | stocks | UNIT | BBG00ZK0HGQ7   | BBG00ZK0HHK1     |   5.000000e+03 |           9.7800000 |    9.780000 |      9.780000 |      9.780000 |     9.780000 | 1.65852e+12 |           50 |
| DCT    | Duck Creek Technologies, Inc. Common Stock                                                                                                                                      | stocks | CS   | BBG00PPWYWJ8   | BBG00PPWYWK6     |   4.230040e+05 |          14.2694000 |   14.980000 |     14.220000 |     14.980000 |    14.050000 | 1.65852e+12 |         5379 |
| IMGN   | Immunogen Inc                                                                                                                                                                   | stocks | CS   | BBG000C2JTB5   | BBG001S6CTN1     |   3.215009e+06 |           4.9886000 |    5.240000 |      4.940000 |      5.260000 |     4.905000 | 1.65852e+12 |        15839 |
| HAS    | Hasbro, Inc.                                                                                                                                                                    | stocks | CS   | BBG000BKVJK4   | BBG001S5RSQ6     |   1.261654e+06 |          81.6768000 |   84.110000 |     81.130000 |     84.110000 |    80.560000 | 1.65852e+12 |        21700 |
| SMN    | ProShares UltraShort Basic Materials                                                                                                                                            | stocks | ETF  | BBG000QXJD09   | BBG001SSTD23     |   1.866800e+04 |          13.4914000 |   13.290000 |     13.730000 |     13.849900 |    13.121600 | 1.65852e+12 |          124 |
| ROCI   | ROC ETF                                                                                                                                                                         | stocks | ETF  | BBG014JCX337   | BBG014JCX408     |   6.405000e+03 |          22.4911000 |   22.520000 |     22.427900 |     22.520000 |    22.365000 | 1.65852e+12 |           19 |
| MPV    | Barings Participation Investors                                                                                                                                                 | stocks | FUND | BBG000BBNTQ5   | BBG001S63V47     |   5.627000e+03 |          12.2907000 |   12.370000 |     12.250000 |     12.370000 |    12.250000 | 1.65852e+12 |           55 |
| GENE   | Genetic Technologies Ltd.                                                                                                                                                       | stocks | ADRC | BBG000P9PKY3   | BBG001SK4X23     |   2.076000e+04 |           1.4564000 |    1.495600 |      1.449000 |      1.500000 |     1.410000 | 1.65852e+12 |          149 |
| MVRL   | ETRACS Monthly Pay 1.5X Leveraged Mortgage REIT ETN                                                                                                                             | stocks | ETN  | BBG00V775QQ7   | BBG00V775RH5     |   4.147900e+04 |          30.1072000 |   30.140000 |     29.885600 |     30.390000 |    29.620000 | 1.65852e+12 |           59 |
| IIIN   | Insteel Industries, Inc.                                                                                                                                                        | stocks | CS   | BBG000KFLDK9   | BBG001SK9GV3     |   3.336580e+05 |          32.4698000 |   34.210000 |     31.950000 |     34.460000 |    31.680000 | 1.65852e+12 |         5717 |
| APPS   | Digital Turbine, Inc.                                                                                                                                                           | stocks | CS   | BBG000HZ3562   | BBG001SPBG12     |   2.886925e+06 |          19.8872000 |   20.390000 |     19.500000 |     20.910000 |    19.330000 | 1.65852e+12 |        27803 |
| GOOG   | Alphabet Inc. Class C Capital Stock                                                                                                                                             | stocks | CS   | BBG009S3NB30   | BBG009S3NB21     |   4.445528e+07 |         109.4594000 |  111.810000 |    108.360000 |    113.180000 |   107.600000 | 1.65852e+12 |       441009 |
| EXD    | Eaton Vance TaxManaged Buy-Write Strategy Fund                                                                                                                                  | stocks | FUND | BBG000QYNPP8   | BBG001T8RYH6     |   5.221000e+03 |          10.5721000 |   10.560000 |     10.440000 |     10.612000 |    10.430000 | 1.65852e+12 |           52 |
| GERM   | ETFMG Treatments Testing and Advancements ETF                                                                                                                                   | stocks | ETF  | BBG00VCZBR80   | BBG00VCZBS15     |   1.420000e+03 |          24.7944000 |   25.088000 |     24.584100 |     25.088000 |    24.562000 | 1.65852e+12 |           23 |
| PSA    | Public Storage                                                                                                                                                                  | stocks | CS   | BBG000BPPN67   | BBG001S5TH79     |   4.171040e+05 |         319.6106000 |  320.050000 |    319.330000 |    323.862500 |   317.570000 | 1.65852e+12 |        12593 |
| RAYS   | Global X Solar ETF                                                                                                                                                              | stocks | ETF  | BBG012DVX111   | BBG012DVX1X6     |   8.631000e+03 |          22.0713000 |   22.405000 |     22.130000 |     22.405000 |    21.986100 | 1.65852e+12 |           76 |
| PWS    | Pacer WealthShield ETF                                                                                                                                                          | stocks | ETF  | BBG00JGBRQ18   | BBG00JGBRQR0     |   4.040000e+02 |          30.7291000 |   30.730000 |     30.720000 |     30.730000 |    30.720000 | 1.65852e+12 |            6 |
| ICLK   | iClick Interactive Asia Group Limited                                                                                                                                           | stocks | ADRC | BBG00J0BKBX0   | BBG00J0BKBY9     |   8.884400e+04 |           0.5849400 |    0.580000 |      0.571000 |      0.607900 |     0.571000 | 1.65852e+12 |          201 |
| KRP    | Kimbell Royalty Partners, LP Common Units representing Limited Partner Interests                                                                                                | stocks | CS   | BBG00FQH6MJ5   | BBG00FQH6N85     |   1.898880e+05 |          16.3003000 |   16.420000 |     16.250000 |     16.550000 |    16.035000 | 1.65852e+12 |         1837 |
| NHTC   | Natural Health Trends Corp.                                                                                                                                                     | stocks | CS   | BBG000FCF697   | BBG001S8JTM1     |   1.058600e+04 |           5.1630000 |    5.180000 |      5.165000 |      5.180000 |     5.150000 | 1.65852e+12 |          126 |
| NMI    | Nuveen Municipal Income                                                                                                                                                         | stocks | FUND | BBG000BQ0Q01   | BBG001S5TNH5     |   8.398000e+03 |           9.3441000 |    9.430000 |      9.320000 |      9.430000 |     9.300000 | 1.65852e+12 |           32 |
| CWS    | AdvisorShares Focused Equity ETF                                                                                                                                                | stocks | ETF  | BBG00DVZRVF5   | BBG00DVZRVH3     |   1.025000e+03 |          44.6502000 |   44.800000 |     44.591100 |     44.800000 |    44.480000 | 1.65852e+12 |           83 |
| IMV    | IMV Inc. Common Shares                                                                                                                                                          | stocks | CS   | BBG000W29615   | BBG001T08Q81     |   5.012300e+04 |           0.5165600 |    0.522500 |      0.508900 |      0.528000 |     0.500800 | 1.65852e+12 |          230 |
| JSMD   | Janus Detroit Street Trust Janus Henderson Small/Mid Cap Growth Alpha ETF                                                                                                       | stocks | ETF  | BBG00C9Y3BC8   | BBG00C9Y3BD7     |   3.224100e+04 |          55.4223000 |   55.850000 |     55.020000 |     55.900000 |    54.690000 | 1.65852e+12 |          161 |
| LBPH   | Longboard Pharmaceuticals, Inc. Common Stock                                                                                                                                    | stocks | CS   | BBG00Y0BJ9N5   | BBG00Y0BJ9Q2     |   6.219000e+03 |           3.3616000 |    3.305000 |      3.500000 |      3.500000 |     3.095000 | 1.65852e+12 |           85 |
| CONX   | CONX Corp. Class A Common Stock                                                                                                                                                 | stocks | CS   | BBG00XS14F06   | BBG00XS14F15     |   2.351800e+04 |           9.9107000 |    9.910200 |      9.915000 |      9.920000 |     9.910000 | 1.65852e+12 |           74 |
| STK    | Columbia Seligman Premium Technology Growth Fund, Inc.                                                                                                                          | stocks | FUND | BBG000PNVDY1   | BBG001T5SVR6     |   4.603600e+04 |          29.1511000 |   29.730000 |     29.010000 |     29.980000 |    28.730000 | 1.65852e+12 |          388 |
| MTGP   | WisdomTree Mortgage Plus Bond Fund                                                                                                                                              | stocks | ETF  | BBG00QTQ1895   | BBG00QTQ1911     |   6.737000e+03 |          46.4390000 |   46.440000 |     46.445000 |     46.445000 |    46.440000 | 1.65852e+12 |            4 |
| WRBY   | Warby Parker Inc.                                                                                                                                                               | stocks | CS   | BBG005DWN8K8   | BBG005DWN8L7     |   1.473356e+06 |          12.5294000 |   12.550000 |     12.590000 |     12.860000 |    12.200000 | 1.65852e+12 |        12259 |
| SCHG   | Schwab U.S. Large-Cap Growth ETF                                                                                                                                                | stocks | ETF  | BBG000Q0CS41   | BBG001T66WN0     |   1.455479e+06 |          62.3846000 |   63.250000 |     62.200000 |     63.620000 |    61.850200 | 1.65852e+12 |         7739 |
| BRK.B  | BERKSHIRE HATHAWAY Class B                                                                                                                                                      | stocks | CS   | BBG000DWG505   | BBG001S90346     |   2.877341e+06 |         286.2264000 |  288.100000 |    285.930000 |    289.400000 |   283.615000 | 1.65852e+12 |        53925 |
| WPP    | WPP PLC                                                                                                                                                                         | stocks | ADRC | BBG000BBVP39   | BBG001S5XJ89     |   1.908670e+05 |          52.3721000 |   52.680000 |     52.220000 |     52.920000 |    51.880000 | 1.65852e+12 |         3379 |
| ADSK   | Autodesk Inc                                                                                                                                                                    | stocks | CS   | BBG000BM7HL0   | BBG001S5SCD4     |   1.457698e+06 |         196.5435000 |  199.190000 |    195.950000 |    203.550000 |   194.480000 | 1.65852e+12 |        32934 |
| WINC   | Western Asset Short Duration Income ETF                                                                                                                                         | stocks | ETF  | BBG00N8V1KX0   | BBG00N8V1LN9     |   1.035000e+04 |          23.9227000 |   23.941000 |     23.940000 |     23.970000 |    23.920000 | 1.65852e+12 |          125 |
| HTUS   | Hull Tactical US ETF                                                                                                                                                            | stocks | ETF  | BBG009H5D6N7   | BBG009H5D6P5     |   3.030000e+02 |          29.7114000 |   29.760000 |     29.497700 |     29.760000 |    29.497700 | 1.65852e+12 |            4 |
| EMGD   | Simplify Emerging Markets Equity PLUS Downside Convexity ETF                                                                                                                    | stocks | ETF  | BBG014FNK4X6   | BBG014FNK5R0     |   2.209000e+03 |          19.4920000 |   19.569900 |     19.390000 |     19.569900 |    19.390000 | 1.65852e+12 |            9 |
| BUFB   | Innovator Laddered Allocation Buffer ETF                                                                                                                                        | stocks | ETF  | BBG0154NXWC2   | BBG0154NXX67     |   7.990000e+02 |          23.2521000 |   23.270000 |     23.105600 |     23.280000 |    23.105600 | 1.65852e+12 |           12 |
| PYN    | PIMCO New York Municipal Income Fund III                                                                                                                                        | stocks | FUND | BBG000BM28W3   | BBG001S9C5C9     |   9.843000e+03 |           8.2050000 |    8.240000 |      8.180000 |      8.315000 |     8.095000 | 1.65852e+12 |           46 |
| GOVX   | GeoVax Labs, Inc. New                                                                                                                                                           | stocks | CS   | BBG000BGGJD8   | BBG001S6ZC26     |   8.774490e+05 |           0.6522400 |    0.670000 |      0.635000 |      0.697900 |     0.620000 | 1.65852e+12 |         1445 |
| DYNT   | Dynatronics Corp                                                                                                                                                                | stocks | CS   | BBG000BHJCZ4   | BBG001S5QPV7     |   2.870000e+03 |           0.6488100 |    0.655000 |      0.642000 |      0.655000 |     0.640100 | 1.65852e+12 |           32 |
| PTA    | Cohen & Steers Tax-Advantaged Preferred Securities and Income Fund                                                                                                              | stocks | FUND | BBG00RP7J328   | BBG00RP7J3T9     |   2.731220e+05 |          19.3786000 |   19.350000 |     19.400000 |     19.600000 |    19.255000 | 1.65852e+12 |         1319 |
| VIRI   | Virios Therapeutics, Inc. Common Stock                                                                                                                                          | stocks | CS   | BBG00X0SZKJ2   | BBG00X0SZLB8     |   1.029600e+04 |           6.8321000 |    6.890000 |      6.790000 |      6.890000 |     6.520000 | 1.65852e+12 |          123 |
| NCPL   | Netcapital Inc. Common Stock                                                                                                                                                    | stocks | CS   | BBG000BYGJV9   | BBG001SJRK47     |   7.096300e+04 |           3.0895000 |    3.130000 |      3.017600 |      3.310000 |     2.930000 | 1.65852e+12 |          273 |
| LMNR   | Limoneira Co                                                                                                                                                                    | stocks | CS   | BBG000BDPC13   | BBG001S775Q4     |   2.720900e+04 |          12.8407000 |   13.380000 |     12.760000 |     13.380000 |    12.710000 | 1.65852e+12 |          604 |
| GLYC   | GlycoMimetics, Inc.                                                                                                                                                             | stocks | CS   | BBG001J1BQN9   | BBG001V0FSK2     |   6.836100e+04 |           0.7209000 |    0.730000 |      0.716300 |      0.751600 |     0.710000 | 1.65852e+12 |          284 |
| CRKN   | Crown Electrokinetics Corp. Common Stock                                                                                                                                        | stocks | CS   | BBG00PNBLK15   | BBG00PNBLKC3     |   5.487500e+04 |           0.8755600 |    0.885000 |      0.850000 |      0.900000 |     0.850000 | 1.65852e+12 |          189 |
| BJUN   | Innovator U.S. Equity Buffer ETF - June                                                                                                                                         | stocks | ETF  | BBG00PBD5ZW5   | BBG00PBD60M2     |   2.860000e+03 |          30.9759000 |   31.120000 |     30.900200 |     31.120000 |    30.860000 | 1.65852e+12 |           24 |
| BKLN   | Invesco Senior Loan ETF                                                                                                                                                         | stocks | ETF  | BBG001K94N28   | BBG001V0MBC0     |   6.051701e+06 |          20.9458000 |   21.010000 |     20.910000 |     21.090000 |    20.850000 | 1.65852e+12 |        10880 |
| MODV   | ModivCare Inc. Common Stock                                                                                                                                                     | stocks | CS   | BBG000P1B7C8   | BBG001SL2C10     |   9.266800e+04 |          97.7390000 |   99.040000 |     98.470000 |     99.040000 |    95.810000 | 1.65852e+12 |         2077 |
| DEMZ   | The Advisors’ Inner Circle Fund III Democratic Large Cap Core ETF                                                                                                               | stocks | ETF  | BBG00Y29K750   | BBG00Y29K803     |   7.599000e+03 |          24.4695000 |   24.600000 |     24.460000 |     24.600000 |    24.330000 | 1.65852e+12 |           42 |
| LOOP   | Loop Industries, Inc. Common Stock                                                                                                                                              | stocks | CS   | BBG003FMFRS2   | BBG003FMFSJ0     |   1.033840e+05 |           6.1241000 |    5.960000 |      6.130000 |      6.340000 |     5.835000 | 1.65852e+12 |          996 |
| AMX    | America Movil S.A.B de C.V                                                                                                                                                      | stocks | ADRC | BBG000CYPVX9   | BBG001SGW6H4     |   1.331498e+06 |          18.6177000 |   18.480000 |     18.680000 |     18.760000 |    18.440000 | 1.65852e+12 |        11312 |
| BCE    | BCE, Inc.                                                                                                                                                                       | stocks | CS   | BBG000BCXNS3   | BBG001S5P2C0     |   1.289733e+06 |          48.9443000 |   48.890000 |     49.110000 |     49.205000 |    48.640000 | 1.65852e+12 |        13094 |
| TTMI   | TTM Technologies Inc                                                                                                                                                            | stocks | CS   | BBG000BYQ0B1   | BBG001SFJKF2     |   5.721380e+05 |          12.8374000 |   13.140000 |     12.790000 |     13.230000 |    12.740000 | 1.65852e+12 |         5648 |
| CHMG   | Chemung Financial Corp                                                                                                                                                          | stocks | CS   | BBG000BHQ7B3   | BBG001S7RT53     |   9.959000e+03 |          45.1869000 |   44.670000 |     45.790000 |     46.910000 |    44.340000 | 1.65852e+12 |          301 |
| UMMA   | Wahed Dow Jones Islamic World ETF                                                                                                                                               | stocks | ETF  | BBG014FFM4C4   | BBG014FFM577     |   1.967000e+03 |          19.9320000 |   19.970000 |     19.730000 |     19.970000 |    19.730000 | 1.65852e+12 |           23 |
| INBK   | First Internet Bancorp                                                                                                                                                          | stocks | CS   | BBG000LF59X1   | BBG001SJZH45     |   4.742000e+04 |          35.5519000 |   36.730000 |     35.310000 |     36.730000 |    35.000000 | 1.65852e+12 |         1006 |
| YYY    | Amplify High Income ETF                                                                                                                                                         | stocks | ETF  | BBG0033KWW36   | BBG0033KWWV5     |   1.425180e+05 |          12.9373000 |   12.930000 |     12.920000 |     13.000000 |    12.872700 | 1.65852e+12 |          722 |
| RNAZ   | TransCode Therapeutics, Inc. Common Stock                                                                                                                                       | stocks | CS   | BBG00ZGYHV03   | BBG00ZGYHVW8     |   1.076600e+04 |           1.3285000 |    1.370000 |      1.290000 |      1.370100 |     1.270000 | 1.65852e+12 |           77 |
| SYPR   | Sypris Solutions Inc                                                                                                                                                            | stocks | CS   | BBG000BBQZV2   | BBG001S6BZF7     |   9.047700e+04 |           2.0816000 |    2.210000 |      2.070000 |      2.230000 |     2.010000 | 1.65852e+12 |          369 |
| EVX    | VanEck Environmental Services ETF                                                                                                                                               | stocks | ETF  | BBG000Q55682   | BBG001SRRWS7     |   1.490000e+02 |         135.0266000 |  134.336900 |    134.336900 |    134.336900 |   134.336900 | 1.65852e+12 |           30 |
| TELL   | Tellurian Inc.                                                                                                                                                                  | stocks | CS   | BBG000CS2ZS4   | BBG001S5TB09     |   1.145225e+07 |           3.6976000 |    3.670000 |      3.610000 |      3.820000 |     3.610000 | 1.65852e+12 |        26890 |
| FCO    | abrdn Global Income Fund, Inc.                                                                                                                                                  | stocks | FUND | BBG000BB41S5   | BBG001S5R2X5     |   2.851500e+04 |           5.2190000 |    5.220000 |      5.190000 |      5.290000 |     5.190000 | 1.65852e+12 |          267 |
| SPXN   | ProShares S&P 500 Ex-Financials ETF                                                                                                                                             | stocks | ETF  | BBG00B2VZX94   | BBG00B2VZXB1     |   1.120000e+02 |          84.1680000 |   83.898500 |     83.898500 |     83.898500 |    83.898500 | 1.65852e+12 |            3 |
| SOXL   | Direxion Daily Semiconductor Bull 3X Shares                                                                                                                                     | stocks | ETF  | BBG000QG1D78   | BBG001T6YQM3     |   6.568772e+07 |          17.9272000 |   18.840000 |     17.650000 |     18.910000 |    17.150000 | 1.65852e+12 |       267216 |
| FTS    | Fortis Inc. Common Shares                                                                                                                                                       | stocks | CS   | BBG000K9SSS5   | BBG001S5YVM5     |   5.885070e+05 |          46.3616000 |   46.050000 |     46.550000 |     46.560000 |    46.050000 | 1.65852e+12 |         7547 |
| GSIG   | Goldman Sachs Access Investment Grade Corporate 1-5 Year Bond ETF                                                                                                               | stocks | ETF  | BBG00VY18832   | BBG00VY188X9     |   4.160000e+02 |          46.9054000 |   46.900000 |     46.928500 |     46.928500 |    46.900000 | 1.65852e+12 |           15 |
| PKI    | PerkinElmer, Inc.                                                                                                                                                               | stocks | CS   | BBG000FXW512   | BBG001SBKS35     |   3.899820e+05 |         146.4784000 |  147.960000 |    146.670000 |    148.780000 |   144.840000 | 1.65852e+12 |        12538 |
| IDEX   | Ideanomics, Inc. Common Stock                                                                                                                                                   | stocks | CS   | BBG000BM2S95   | BBG001S9D128     |   4.198899e+06 |           0.7020600 |    0.719700 |      0.700000 |      0.719800 |     0.690000 | 1.65852e+12 |         8615 |
| DHIL   | Diamond Hill Investment Group                                                                                                                                                   | stocks | CS   | BBG000K7DJG8   | BBG001SCH7V2     |   5.849000e+03 |         188.4778000 |  190.340000 |    187.200000 |    190.440000 |   186.840000 | 1.65852e+12 |          345 |
| AAN    | The Aaron’s Company, Inc.                                                                                                                                                       | stocks | CS   | BBG00WCNDCZ6   | BBG00WCNDDH4     |   2.643900e+05 |          16.0277000 |   16.380000 |     16.010000 |     16.450000 |    15.720000 | 1.65852e+12 |         3823 |
| FLAU   | Franklin FTSE Australia ETF                                                                                                                                                     | stocks | ETF  | BBG00J3M8WZ9   | BBG00J3M8XP8     |   1.136000e+03 |          25.8744000 |   26.000000 |     25.705500 |     26.000000 |    25.690000 | 1.65852e+12 |           24 |
| SNDR   | Schneider National, Inc.                                                                                                                                                        | stocks | CS   | BBG000DR87M7   | BBG001SF1R75     |   3.664090e+05 |          23.9867000 |   24.070000 |     24.030000 |     24.160000 |    23.840000 | 1.65852e+12 |         5696 |
| VIS    | Vanguard Industrials ETF                                                                                                                                                        | stocks | ETF  | BBG000HX9TN0   | BBG001SHVTX5     |   5.371400e+04 |         170.4190000 |  171.660000 |    170.340000 |    172.140000 |   169.620000 | 1.65852e+12 |          759 |
| CME    | CME Group Inc.                                                                                                                                                                  | stocks | CS   | BBG000BHLYP4   | BBG001S86547     |   9.512020e+05 |         204.5621000 |  205.430000 |    204.480000 |    206.550000 |   202.920000 | 1.65852e+12 |        19622 |
| MIST   | Milestone Pharmaceuticals Inc. Common Shares                                                                                                                                    | stocks | CS   | BBG009DR4MN8   | BBG009DR4MP6     |   8.507000e+04 |           7.2257000 |    7.155000 |      7.160000 |      7.430000 |     7.060000 | 1.65852e+12 |          707 |
| JRS    | Nuveen Real Estate Income Fund                                                                                                                                                  | stocks | FUND | BBG000DQ3HW1   | BBG001SJX6L2     |   1.499380e+05 |           9.4076000 |    9.500000 |      9.430000 |      9.550000 |     9.280100 | 1.65852e+12 |          758 |
| SPTL   | SPDR Portfolio Long Term Treasury ETF                                                                                                                                           | stocks | ETF  | BBG000RFRG83   | BBG001STKCY7     |   8.294830e+06 |          34.0331000 |   34.035000 |     34.110000 |     34.290000 |    33.930000 | 1.65852e+12 |        20832 |
| SAA    | ProShares Ulta SmallCap600                                                                                                                                                      | stocks | ETF  | BBG000BB6SX8   | BBG001S5VV75     |   4.653000e+03 |          22.6454000 |   22.980000 |     22.610000 |     23.130000 |    22.270000 | 1.65852e+12 |           86 |
| HYLB   | Xtrackers USD High Yield Corporate Bond ETF                                                                                                                                     | stocks | ETF  | BBG00FGWY3Q6   | BBG00FGWY4G5     |   2.799508e+06 |          35.2987000 |   35.420000 |     35.210000 |     35.520000 |    35.100000 | 1.65852e+12 |         7736 |
| TCX    | Tucows, Inc                                                                                                                                                                     | stocks | CS   | BBG000L69KL5   | BBG001S978L9     |   3.874600e+04 |          47.3038000 |   47.470000 |     47.790000 |     47.880000 |    46.900000 | 1.65852e+12 |         1131 |
| CIA    | Citizens, Inc.                                                                                                                                                                  | stocks | CS   | BBG000DJ3D29   | BBG001S81V90     |   2.943800e+04 |           3.9103000 |    3.840000 |      3.880000 |      3.980000 |     3.840000 | 1.65852e+12 |          591 |
| WNC    | Wabash National Corp.                                                                                                                                                           | stocks | CS   | BBG000CGM9H8   | BBG001S6W2K1     |   3.188530e+05 |          15.9610000 |   16.050000 |     15.950000 |     16.170000 |    15.810000 | 1.65852e+12 |         3794 |
| SBFM   | Sunshine Biopharma Inc.                                                                                                                                                         | stocks | CS   | BBG000RVLVB7   | BBG001SVBT66     |   4.052870e+05 |           1.0554000 |    1.110000 |      1.040000 |      1.110000 |     1.030000 | 1.65852e+12 |          968 |
| AMPH   | Amphastar Pharmaceuticals, Inc.                                                                                                                                                 | stocks | CS   | BBG000BW90S6   | BBG001SCM4W2     |   1.707040e+05 |          35.9151000 |   36.090000 |     35.960000 |     36.300000 |    35.550000 | 1.65852e+12 |         4235 |
| O      | Realty Income Corporation                                                                                                                                                       | stocks | CS   | BBG000DHPN63   | BBG001S884K0     |   2.224750e+06 |          71.2120000 |   71.000000 |     71.350000 |     71.465000 |    70.690000 | 1.65852e+12 |        36795 |
| CGC    | Canopy Growth Corporation Common Shares                                                                                                                                         | stocks | CS   | BBG0069FB082   | BBG001T6MH82     |   1.068356e+07 |           2.6406000 |    2.710000 |      2.565000 |      2.840000 |     2.520000 | 1.65852e+12 |        24118 |
| TBK    | Triumph Bancorp, Inc.                                                                                                                                                           | stocks | CS   | BBG000QS6MN9   | BBG001SNKPZ0     |   1.420580e+05 |          69.6040000 |   70.750000 |     69.080000 |     71.360000 |    68.490000 | 1.65852e+12 |         3903 |
| AMWD   | American Woodmark Corp                                                                                                                                                          | stocks | CS   | BBG000BBX657   | BBG001S5NQ57     |   6.528000e+04 |          47.7689000 |   48.030000 |     48.000000 |     48.480000 |    46.950000 | 1.65852e+12 |         1780 |
| PRVB   | Provention Bio, Inc. Common Stock                                                                                                                                               | stocks | CS   | BBG00H1L1L88   | BBG00H1L1L97     |   2.861060e+05 |           4.0080000 |    4.180000 |      3.990000 |      4.180000 |     3.931700 | 1.65852e+12 |         2588 |
| VCNX   | Vaccinex, Inc. Common Stock                                                                                                                                                     | stocks | CS   | BBG00469NFK4   | BBG00469NFM2     |   3.247100e+04 |           1.1362000 |    1.160000 |      1.120000 |      1.160000 |     1.110000 | 1.65852e+12 |          236 |
| FNWD   | Finward Bancorp Common Stock                                                                                                                                                    | stocks | CS   | BBG000BRR418   | BBG001S9SXD0     |   2.710000e+03 |          37.4937000 |   37.140000 |     37.680000 |     37.990000 |    37.040000 | 1.65852e+12 |           62 |
| LETB   | AdvisorShares Let Bob AI Powered Momentum ETF                                                                                                                                   | stocks | ETF  | BBG015780X81   | BBG015780YW2     |   4.688000e+03 |          23.1792000 |   23.230000 |     23.170000 |     23.230000 |    23.137600 | 1.65852e+12 |           19 |
| RRR    | Red Rock Resorts, Inc. Class A Common Stock                                                                                                                                     | stocks | CS   | BBG00B6G8077   | BBG00B6G8068     |   6.254790e+05 |          36.9097000 |   36.960000 |     36.690000 |     37.550000 |    36.175000 | 1.65852e+12 |        11384 |
| LEA    | Lear Corporation                                                                                                                                                                | stocks | CS   | BBG000PTLGZ1   | BBG001T60092     |   5.183280e+05 |         141.4731000 |  143.680000 |    141.170000 |    144.130000 |   139.855000 | 1.65852e+12 |        11722 |
| QCLN   | First Trust NASDAQ Clean Edge Green Energy Index Fund                                                                                                                           | stocks | ETF  | BBG000QX2HH0   | BBG001SSSWQ6     |   3.141030e+05 |          54.8984000 |   56.320000 |     54.530000 |     56.560000 |    54.185300 | 1.65852e+12 |         2322 |
| AXSM   | Axsome Therapeutics, Inc                                                                                                                                                        | stocks | CS   | BBG00B6G7GL7   | BBG00B6G7GM6     |   7.632580e+05 |          41.0553000 |   43.260000 |     40.310000 |     43.982800 |    40.210000 | 1.65852e+12 |        11168 |
| MTD    | Mettler-Toledo International                                                                                                                                                    | stocks | CS   | BBG000BZCKH3   | BBG001SB87G1     |   1.138210e+05 |        1228.6399000 | 1249.090000 |   1224.140000 |   1255.320000 |  1219.450000 | 1.65852e+12 |         8738 |
| KRNY   | Kearny Financial Corporation                                                                                                                                                    | stocks | CS   | BBG008N1HXP6   | BBG008N1HXQ5     |   1.937070e+05 |          11.3559000 |   11.410000 |     11.370000 |     11.430000 |    11.260000 | 1.65852e+12 |         2800 |
| PBI    | Pitney Bowes Inc.                                                                                                                                                               | stocks | CS   | BBG000BQTMJ9   | BBG001S5V171     |   1.014660e+06 |           4.1067000 |    4.190000 |      4.110000 |      4.190000 |     4.040000 | 1.65852e+12 |         7253 |
| PHI    | PLDT Inc.                                                                                                                                                                       | stocks | ADRC | BBG000BB8KT8   | BBG001S5V5J9     |   5.693800e+04 |          29.7671000 |   29.440000 |     29.690000 |     30.040000 |    29.440000 | 1.65852e+12 |         1184 |
| EDOC   | Global X Telemedicine & Digital Health ETF                                                                                                                                      | stocks | ETF  | BBG00WCNXFP8   | BBG00WCNXGG6     |   5.622200e+04 |          12.6722000 |   12.910000 |     12.610000 |     13.000000 |    12.530000 | 1.65852e+12 |          251 |
| JT     | Jianpu Technology Inc. American Depositary Shares, each representing twenty Class A Ordinary Shares                                                                             | stocks | ADRC | BBG00J0BKGB3   | BBG00J0BKGC2     |   2.165770e+05 |           1.6583000 |    1.600000 |      1.610000 |      1.720000 |     1.530000 | 1.65852e+12 |          493 |
| XSHQ   | Invesco S&P SmallCap Quality ETF                                                                                                                                                | stocks | ETF  | BBG00GBGV9L9   | BBG00GBGVB98     |   8.760000e+03 |          33.0785000 |   33.610000 |     33.202000 |     33.610000 |    33.020000 | 1.65852e+12 |           29 |
| GMTX   | Gemini Therapeutics, Inc. Common Stock                                                                                                                                          | stocks | CS   | BBG00W9MJK79   | BBG00W9MJL04     |   6.274000e+04 |           1.5875000 |    1.570000 |      1.590000 |      1.610000 |     1.560000 | 1.65852e+12 |          478 |
| RNEM   | First Trust Emerging Markets Equity Select ETF                                                                                                                                  | stocks | ETF  | BBG00GY1HY38   | BBG00GY1HYT0     |   2.940000e+02 |          42.3751000 |   42.370000 |     42.590000 |     42.590000 |    42.370000 | 1.65852e+12 |            7 |
| WFCF   | Where Food Comes From, Inc. Common Stock                                                                                                                                        | stocks | CS   | BBG000Q889B3   | BBG001SRX4Z4     |   5.166000e+03 |          10.9367000 |   11.230200 |     10.450000 |     11.300700 |    10.450000 | 1.65852e+12 |           64 |
| TRKA   | Troika Media Group, Inc. Common Stock                                                                                                                                           | stocks | CS   | BBG00LLKH1Z6   | BBG00LLKH202     |   2.227940e+05 |           0.8584100 |    0.870000 |      0.860100 |      0.885300 |     0.811100 | 1.65852e+12 |          637 |
| VRME   | VerifyMe, Inc. Common Stock                                                                                                                                                     | stocks | CS   | BBG000BL5RQ7   | BBG001SD4YY2     |   3.704300e+04 |           1.7170000 |    1.800000 |      1.710000 |      1.890000 |     1.680000 | 1.65852e+12 |          184 |
| BYD    | Boyd Gaming Corporation                                                                                                                                                         | stocks | CS   | BBG000BHX9P6   | BBG001S7DMW3     |   8.498930e+05 |          54.6026000 |   55.790000 |     54.640000 |     55.999900 |    54.010000 | 1.65852e+12 |        14606 |
| MTEX   | Mannatech Inc.                                                                                                                                                                  | stocks | CS   | BBG000C24TS3   | BBG001SBT652     |   8.663000e+03 |          16.1185000 |   16.490000 |     15.750000 |     16.600000 |    15.650000 | 1.65852e+12 |          178 |
| FLMI   | Franklin Dynamic Municipal Bond ETF                                                                                                                                             | stocks | ETF  | BBG00HLJ4L24   | BBG00HLJ4LS6     |   2.330400e+04 |          24.1147000 |   24.090000 |     24.130000 |     24.150000 |    24.050000 | 1.65852e+12 |          170 |
| OVL    | Overlay Shares Large Cap Equity ETF                                                                                                                                             | stocks | ETF  | BBG00QFNW9L7   | BBG00QFNWBB3     |   1.329700e+04 |          32.5686000 |   32.810000 |     32.407300 |     32.810000 |    32.260000 | 1.65852e+12 |           44 |
| NDLS   | Noodles & Company Class A                                                                                                                                                       | stocks | CS   | BBG001CN6Q81   | BBG001TYMSZ7     |   1.251740e+05 |           5.1203000 |    5.210000 |      5.040000 |      5.300000 |     5.020000 | 1.65852e+12 |         1800 |
| NBCT   | Neuberger Berman Carbon Transition & Infrastructure ETF                                                                                                                         | stocks | ETF  | BBG016KF7SS0   | BBG016KF7TM4     |   4.000000e+01 |          22.3800000 |   22.436200 |     22.436200 |     22.436200 |    22.436200 | 1.65852e+12 |            1 |
| SPYG   | SPDR Portfolio S&P 500 Growth ETF                                                                                                                                               | stocks | ETF  | BBG000BLH653   | BBG001SD7RB9     |   2.441551e+06 |          56.1335000 |   56.810000 |     55.990000 |     57.070000 |    55.670000 | 1.65852e+12 |        12141 |
| CVR    | Chicago Rivet & Machine Co.                                                                                                                                                     | stocks | CS   | BBG000BGRMB1   | BBG001S5QBC9     |   9.740000e+02 |          27.9830000 |   27.970000 |     28.090000 |     28.090000 |    27.970000 | 1.65852e+12 |           39 |
| CCU    | Compania Cervecerias Unidas S.A.                                                                                                                                                | stocks | ADRC | BBG000BBCT50   | BBG001S5VY42     |   4.234620e+05 |          11.0447000 |   11.240000 |     11.040000 |     11.260000 |    10.990000 | 1.65852e+12 |         3096 |
| BOND   | PIMCO Active Bond Exchange-Traded Fund                                                                                                                                          | stocks | ETF  | BBG002N12B91   | BBG002N12C17     |   2.473570e+05 |          96.2695000 |   96.330000 |     96.390000 |     96.643000 |    96.142000 | 1.65852e+12 |         1487 |
| VINC   | Vincerx Pharma, Inc. Common Stock                                                                                                                                               | stocks | CS   | BBG00RRQXMX2   | BBG00RRQXMY1     |   6.597900e+04 |           1.4937000 |    1.530000 |      1.490000 |      1.530000 |     1.460000 | 1.65852e+12 |          215 |
| DJIA   | Global X Dow 30 Covered Call ETF                                                                                                                                                | stocks | ETF  | BBG015GTQ3H3   | BBG015GTQ4F3     |   2.021700e+04 |          23.0930000 |   23.170000 |     23.050000 |     23.170000 |    23.030000 | 1.65852e+12 |          280 |
| KMX    | CarMax Inc.                                                                                                                                                                     | stocks | CS   | BBG000BLMZK6   | BBG001SD9561     |   8.547500e+05 |          94.8981000 |   96.490000 |     94.350000 |     98.490000 |    93.700000 | 1.65852e+12 |        15963 |
| VERX   | Vertex, Inc. Class A Common Stock                                                                                                                                               | stocks | CS   | BBG00VVT2F25   | BBG00VVT2FV3     |   5.040500e+04 |          10.7995000 |   11.110000 |     10.720000 |     11.350000 |    10.500000 | 1.65852e+12 |         1234 |
| IXSE   | WisdomTree India Ex-State-Owned Enterprises Fund                                                                                                                                | stocks | ETF  | BBG00NSBVNF3   | BBG00NSBVP59     |   5.400000e+02 |          31.5486000 |   31.630000 |     31.400000 |     31.630000 |    31.400000 | 1.65852e+12 |            5 |
| FCFS   | FirstCash Holdings, Inc. Common Stock                                                                                                                                           | stocks | CS   | BBG0145KL747   | BBG0145KL756     |   1.458950e+05 |          67.4812000 |   68.040000 |     67.620000 |     68.468300 |    66.530000 | 1.65852e+12 |         4024 |
| FRXB   | Forest Road Acquisition Corp. II                                                                                                                                                | stocks | CS   | BBG0108B3FX2   | BBG0108B3GR7     |   5.150800e+04 |           9.8300000 |    9.830000 |      9.820000 |      9.830000 |     9.820000 | 1.65852e+12 |           29 |
| RISR   | FolioBeyond Rising Rate ETF                                                                                                                                                     | stocks | ETF  | BBG012S2TSG9   | BBG012S2TTB2     |   5.726970e+05 |          31.2934000 |   31.800000 |     31.500000 |     31.810000 |    31.200000 | 1.65852e+12 |          626 |
| GLTO   | Galecto, Inc. Common Stock                                                                                                                                                      | stocks | CS   | BBG00XMLGWC5   | BBG00XMLGWD4     |   1.092400e+04 |           1.6717000 |    1.737900 |      1.670000 |      1.737900 |     1.620000 | 1.65852e+12 |           78 |
| MCK    | McKesson Corporation                                                                                                                                                            | stocks | CS   | BBG000DYGNW7   | BBG001S8F8P8     |   8.353480e+05 |         330.3868000 |  330.320000 |    330.440000 |    333.420000 |   327.830000 | 1.65852e+12 |        17092 |
| XOM    | Exxon Mobil Corporation                                                                                                                                                         | stocks | CS   | BBG000GZQ728   | BBG001S69V32     |   1.547948e+07 |          87.4367000 |   87.550000 |     87.080000 |     88.470000 |    86.630000 | 1.65852e+12 |       138102 |
| URE    | ProShares Ultra Real Estate                                                                                                                                                     | stocks | ETF  | BBG000QXLPJ9   | BBG001SSTDC2     |   8.420000e+03 |          76.6160000 |   76.961900 |     76.095000 |     76.961900 |    75.270000 | 1.65852e+12 |           73 |
| BAD    | B.A.D. ETF                                                                                                                                                                      | stocks | ETF  | BBG0147BLY91   | BBG0147BLZ43     |   6.470000e+02 |          12.3274000 |   12.380000 |     12.225700 |     12.380000 |    12.225700 | 1.65852e+12 |           22 |
| GP     | GreenPower Motor Company Inc. Common Shares                                                                                                                                     | stocks | CS   | BBG008366HR2   | BBG001TC2Y20     |   1.179350e+05 |           3.8207000 |    4.020000 |      3.620000 |      4.080000 |     3.620000 | 1.65852e+12 |         1058 |
| BERY   | Berry Global Group, Inc.                                                                                                                                                        | stocks | CS   | BBG000Q1R1Y9   | BBG001T6B4W6     |   6.351380e+05 |          56.7304000 |   57.080000 |     56.560000 |     57.630000 |    56.355000 | 1.65852e+12 |         9967 |
| BSJT   | Invesco BulletShares 2029 High Yield Corporate Bond ETF                                                                                                                         | stocks | ETF  | BBG012JBJ2Y7   | BBG012JBJ3V8     |   3.607800e+04 |          21.0689000 |   21.060000 |     20.925000 |     21.090000 |    20.925000 | 1.65852e+12 |          180 |
| CIK    | Credit Suisse Asset Management Income Fund Inc.                                                                                                                                 | stocks | FUND | BBG000G894K4   | BBG001SCVJQ6     |   1.189730e+05 |           2.7517000 |    2.750000 |      2.760000 |      2.765000 |     2.740000 | 1.65852e+12 |          368 |
| GOLD   | Barrick Gold Corp.                                                                                                                                                              | stocks | CS   | BBG000BB07P9   | BBG001S5N9P3     |   2.882583e+07 |          15.5063000 |   15.650000 |     15.330000 |     15.897100 |    15.180000 | 1.65852e+12 |        87171 |
| RFL    | Rafael Holdings, Inc. Class B Common Stock                                                                                                                                      | stocks | CS   | BBG00J3YYS92   | BBG00J3YYSB9     |   9.323300e+04 |           2.2120000 |    2.280000 |      2.190000 |      2.319900 |     2.170000 | 1.65852e+12 |          483 |
| STRR   | Star Equity Holdings, Inc. Common Stock                                                                                                                                         | stocks | CS   | BBG000BVZVL8   | BBG001SFBBM2     |   7.993400e+04 |           1.0366000 |    1.030000 |      1.030000 |      1.060000 |     1.020000 | 1.65852e+12 |          178 |
| WM     | Waste Management, Inc.                                                                                                                                                          | stocks | CS   | BBG000BWVSR1   | BBG001S5XH47     |   1.145188e+06 |         155.1760000 |  155.710000 |    154.930000 |    156.530000 |   154.310000 | 1.65852e+12 |        24904 |
| IGR    | CBRE Global Real Estate Income Fund                                                                                                                                             | stocks | FUND | BBG000JSF386   | BBG001SJ9SR5     |   2.558310e+05 |           7.4017000 |    7.380000 |      7.390000 |      7.440000 |     7.340000 | 1.65852e+12 |          901 |
| HACK   | ETFMG Prime Cyber Security ETF                                                                                                                                                  | stocks | ETF  | BBG007HXLJS8   | BBG007HXLJT7     |   3.010940e+05 |          48.7279000 |   49.260000 |     48.620000 |     49.650000 |    48.350000 | 1.65852e+12 |         2595 |
| METC   | Ramaco Resources, Inc. Common Stock                                                                                                                                             | stocks | CS   | BBG00BCQJ2X3   | BBG00BCQJ2W4     |   6.105430e+05 |          11.2085000 |   11.360000 |     11.030000 |     11.605000 |    11.010000 | 1.65852e+12 |         7578 |
| NAZ    | Nuveen Arizona Quality Municipal Income Fund                                                                                                                                    | stocks | FUND | BBG000BGNXH5   | BBG001S75CC6     |   8.149000e+03 |          12.9885000 |   12.990000 |     12.980000 |     13.039900 |    12.940000 | 1.65852e+12 |           45 |
| RAVI   | FlexShares Ready Access Variable Income Fund                                                                                                                                    | stocks | ETF  | BBG003GP3VC5   | BBG003GP3W33     |   4.249500e+04 |          74.4792000 |   74.410000 |     74.478000 |     74.500000 |    74.410000 | 1.65852e+12 |           77 |
| PATH   | UiPath, Inc.                                                                                                                                                                    | stocks | CS   | BBG00GKS1G03   | BBG00GKS1G12     |   3.527951e+06 |          19.7698000 |   21.050000 |     19.410000 |     21.110000 |    19.210000 | 1.65852e+12 |        37210 |
| FLSW   | Franklin FTSE Switzerland ETF                                                                                                                                                   | stocks | ETF  | BBG00JYXTYC9   | BBG00JYXTZ27     |   9.048000e+03 |          29.2195000 |   29.400000 |     29.264700 |     29.400000 |    29.190000 | 1.65852e+12 |           68 |
| URBN   | Urban Outfitters Inc                                                                                                                                                            | stocks | CS   | BBG000BL79J3   | BBG001S7H9K1     |   1.539938e+06 |          20.9521000 |   21.150000 |     20.860000 |     21.640000 |    20.730000 | 1.65852e+12 |        18191 |
| JNPR   | Juniper Networks Inc                                                                                                                                                            | stocks | CS   | BBG000BY33P5   | BBG001SCTT05     |   3.696848e+06 |          28.6089000 |   29.250000 |     28.610000 |     29.370000 |    28.265000 | 1.65852e+12 |        30043 |
| CASI   | CASI Pharmaceuticals, Inc.                                                                                                                                                      | stocks | CS   | BBG000GW8ML1   | BBG001S91RM2     |   4.875700e+04 |           2.7531000 |    2.760000 |      2.730000 |      2.844600 |     2.680000 | 1.65852e+12 |          322 |
| ESHY   | Xtrackers J.P. Morgan ESG USD High Yield Corporate Bond ETF                                                                                                                     | stocks | ETF  | BBG0086BG290   | BBG0086BG2B7     |   1.030000e+02 |          18.5326000 |   18.530000 |     18.495000 |     18.530000 |    18.495000 | 1.65852e+12 |            3 |
| ADBE   | Adobe Inc.                                                                                                                                                                      | stocks | CS   | BBG000BB5006   | BBG001S5NCQ5     |   2.413224e+06 |         402.9262000 |  410.030000 |    401.900000 |    414.620000 |   398.631000 | 1.65852e+12 |        57241 |
| PDLB   | Ponce Financial Group, Inc. Common Stock                                                                                                                                        | stocks | CS   | BBG014XDMDV8   | BBG014XDMDW7     |   7.517000e+03 |           9.2816000 |    9.310200 |      9.310000 |      9.320000 |     9.210000 | 1.65852e+12 |          142 |
| ICVX   | Icosavax, Inc. Common Stock                                                                                                                                                     | stocks | CS   | BBG00QHK2J87   | BBG00QHK2J96     |   6.316900e+04 |           7.9331000 |    8.510000 |      7.620000 |      8.510000 |     7.610000 | 1.65852e+12 |         1385 |
| FOXA   | Fox Corporation Class A Common Stock                                                                                                                                            | stocks | CS   | BBG00JHNJW99   | BBG00JHNJX06     |   1.594247e+06 |          35.0519000 |   35.000000 |     35.160000 |     35.350000 |    34.754500 | 1.65852e+12 |        20360 |
| IAG    | IAMGold Corporation                                                                                                                                                             | stocks | CS   | BBG000LL9LQ5   | BBG001S8SMV6     |   4.698921e+06 |           1.4615000 |    1.480000 |      1.430000 |      1.530000 |     1.420000 | 1.65852e+12 |         6761 |
| TPHS   | Trinity Place Holdings Inc.com                                                                                                                                                  | stocks | CS   | BBG003D482W2   | BBG003D482X1     |   3.067000e+03 |           1.1087000 |    1.100000 |      1.130000 |      1.130000 |     1.085000 | 1.65852e+12 |           45 |
| MED    | Medifast, Inc.                                                                                                                                                                  | stocks | CS   | BBG000BWBW76   | BBG001SD45H4     |   1.284390e+05 |         172.5215000 |  177.090000 |    172.530000 |    177.090000 |   170.855000 | 1.65852e+12 |         5213 |
| DFND   | Siren DIVCON Dividend Defender ETF                                                                                                                                              | stocks | ETF  | BBG00BT79C75   | BBG00BT79C84     |   4.091000e+03 |          34.3077000 |   33.884200 |     33.495000 |     34.370000 |    33.495000 | 1.65852e+12 |           11 |
| GSY    | Invesco Ultra Short Duration ETF                                                                                                                                                | stocks | ETF  | BBG00KJR1SL9   | BBG00KJR1T91     |   2.651190e+05 |          49.5407000 |   49.520000 |     49.550000 |     49.550000 |    49.520000 | 1.65852e+12 |          796 |
| FQAL   | Fidelity Quality Factor ETF                                                                                                                                                     | stocks | ETF  | BBG00DRGLM68   | BBG00DRGLM86     |   3.540100e+04 |          46.3551000 |   46.880000 |     46.290000 |     46.880000 |    46.129200 | 1.65852e+12 |          225 |
| WKLY   | SoFi Weekly Dividend ETF                                                                                                                                                        | stocks | ETF  | BBG010Z8G059   | BBG010Z8G102     |   2.183000e+03 |          44.4637000 |   44.490700 |     44.224500 |     44.630000 |    44.180000 | 1.65852e+12 |          195 |
| GLIF   | AGFiQ Global Infrastructure ETF                                                                                                                                                 | stocks | ETF  | BBG00P83CJ22   | BBG00P83CJV0     |   9.000000e+00 |          26.6982000 |   26.590400 |     26.590400 |     26.590400 |    26.590400 | 1.65852e+12 |            2 |
| DOMA   | Doma Holdings, Inc.                                                                                                                                                             | stocks | CS   | BBG00YVJXDT3   | BBG00YVJXFN4     |   7.836580e+05 |           0.7374300 |    0.796400 |      0.718600 |      0.806883 |     0.710000 | 1.65852e+12 |         2953 |
| EMR    | Emerson Electric Co.                                                                                                                                                            | stocks | CS   | BBG000BHX7N2   | BBG001S5QVT7     |   1.932428e+06 |          83.1144000 |   84.110000 |     83.100000 |     84.330000 |    82.490000 | 1.65852e+12 |        26657 |
| XSLV   | Invesco S&P SmallCap Low Volatility ETF                                                                                                                                         | stocks | ETF  | BBG00449DVG7   | BBG00449DVH6     |   1.177960e+05 |          45.6127000 |   45.790000 |     45.710000 |     46.000000 |    45.380000 | 1.65852e+12 |          552 |
| RCEL   | Avita Medical, Inc. Common Stock                                                                                                                                                | stocks | ADRC | BBG00TD18T43   | BBG00TD18TF1     |   5.365400e+04 |           5.8980000 |    6.070000 |      5.830000 |      6.190000 |     5.730000 | 1.65852e+12 |         1129 |
| HSCZ   | iShares Currency Hedged MSCI EAFE Small-Cap ETF                                                                                                                                 | stocks | ETF  | BBG009HYWS28   | BBG009HYWS37     |   7.671000e+03 |          32.0752000 |   32.250000 |     31.961000 |     32.250000 |    31.920000 | 1.65852e+12 |           61 |
| WAT    | Waters Corp                                                                                                                                                                     | stocks | CS   | BBG000FQRVM3   | BBG001S8MDG9     |   2.537570e+05 |         346.5149000 |  351.400000 |    345.680000 |    353.140000 |   342.530000 | 1.65852e+12 |         8465 |
| DFAI   | Dimensional International Core Equity Market ETF                                                                                                                                | stocks | ETF  | BBG00Y2PGCR4   | BBG00Y2PGDN6     |   1.042155e+06 |          24.4917000 |   24.620000 |     24.450000 |     24.730000 |    24.340000 | 1.65852e+12 |         2535 |
| ONCT   | Oncternal Therapeutics, Inc. Common Stock                                                                                                                                       | stocks | CS   | BBG000BX50F2   | BBG001SCTS61     |   5.148290e+05 |           1.0899000 |    1.150000 |      1.060000 |      1.150000 |     1.050000 | 1.65852e+12 |         1471 |
| ASTC   | Astrotech Corporation (DE) Common Stock                                                                                                                                         | stocks | CS   | BBG000FNC0Z0   | BBG001S8LVC4     |   9.174500e+04 |           0.4629000 |    0.472000 |      0.459600 |      0.477213 |     0.450000 | 1.65852e+12 |          173 |
| AUGX   | Augmedix, Inc. Common Stock                                                                                                                                                     | stocks | CS   | BBG006674MJ1   | BBG006674MM7     |   9.273000e+03 |           1.5423000 |    1.580000 |      1.570000 |      1.640000 |     1.480000 | 1.65852e+12 |           89 |
| BOLT   | Bolt Biotherapeutics, Inc. Common Stock                                                                                                                                         | stocks | CS   | BBG00N8MY5V9   | BBG00N8MY5W8     |   7.131600e+04 |           2.5071000 |    2.660000 |      2.460000 |      2.660000 |     2.430000 | 1.65852e+12 |          759 |
| PSAG   | Property Solutions Acquisition Corporation II Class A Common Stock                                                                                                              | stocks | CS   | BBG00ZSKXF25   | BBG00ZSKXFZ9     |   1.120000e+02 |           9.8190000 |    9.820000 |      9.820000 |      9.820000 |     9.820000 | 1.65852e+12 |            4 |
| GGZ    | The Gabelli Global Small and Mid Cap Value Trust                                                                                                                                | stocks | FUND | BBG00562K7K7   | BBG00562K7L6     |   2.989000e+03 |          11.6196000 |   11.690000 |     11.540000 |     11.700000 |    11.540000 | 1.65852e+12 |           49 |
| ENZ    | Enzo Biochem, Inc.                                                                                                                                                              | stocks | CS   | BBG000BHY9Q4   | BBG001S5QWQ8     |   2.323200e+04 |           2.5829000 |    2.600100 |      2.580000 |      2.620000 |     2.560000 | 1.65852e+12 |          333 |
| PXH    | Invesco FTSE RAFI Emerging Markets ETF                                                                                                                                          | stocks | ETF  | BBG000QSJX15   | BBG001SSJYS0     |   1.182377e+06 |          17.5644000 |   17.620000 |     17.490000 |     17.650000 |    17.420100 | 1.65852e+12 |         1170 |
| PAA    | Plains All American Pipeline, L.P. Common Units representing Limited Partner Interests                                                                                          | stocks | CS   | BBG000BP63C5   | BBG001S985K5     |   2.782151e+06 |          10.5822000 |   10.640000 |     10.510000 |     10.890000 |    10.445000 | 1.65852e+12 |        11696 |
| ERTH   | Invesco MSCI Sustainable Future ETF                                                                                                                                             | stocks | ETF  | BBG000Q4YTD7   | BBG001SRRFG7     |   6.261000e+03 |          54.6974000 |   55.240000 |     54.390000 |     55.310000 |    54.180000 | 1.65852e+12 |          160 |
| ABEO   | Abeona Therapeutics Inc. Common Stock                                                                                                                                           | stocks | CS   | BBG000DT5D52   | BBG001S8T7K0     |   5.541900e+04 |           4.8847000 |    5.140000 |      4.840000 |      5.140000 |     4.790000 | 1.65852e+12 |          350 |
| CLXT   | Calyxt, Inc. Common Stock                                                                                                                                                       | stocks | CS   | BBG00H1CJS11   | BBG00H1CJSR3     |   7.173790e+05 |           0.2448100 |    0.252700 |      0.240000 |      0.259000 |     0.223200 | 1.65852e+12 |         1164 |
| PRTY   | Party City Holdco Inc.                                                                                                                                                          | stocks | CS   | BBG005WX31L1   | BBG005WX31M0     |   2.201567e+06 |           1.3009000 |    1.380000 |      1.290000 |      1.390000 |     1.260000 | 1.65852e+12 |         7798 |
| INST   | Instructure Holdings, Inc.                                                                                                                                                      | stocks | CS   | BBG011MGVB34   | BBG011MGVC50     |   1.850490e+05 |          23.7656000 |   23.950000 |     23.790000 |     24.480000 |    23.420000 | 1.65852e+12 |         2711 |
| VEGI   | iShares MSCI Global Agriculture Producers ETF                                                                                                                                   | stocks | ETF  | BBG002GKQWJ4   | BBG002GKQX93     |   6.145200e+04 |          39.4169000 |   39.760000 |     39.320000 |     39.955600 |    39.225000 | 1.65852e+12 |          638 |
| OSIS   | OSI Systems Inc                                                                                                                                                                 | stocks | CS   | BBG000BWPR54   | BBG001SB1J54     |   7.185500e+04 |          93.0366000 |   93.470000 |     93.130000 |     93.940000 |    92.360000 | 1.65852e+12 |         1828 |
| GLAD   | Gladstone Capital Corp                                                                                                                                                          | stocks | CS   | BBG000DJYTQ4   | BBG001SJH738     |   7.849400e+04 |          10.8851000 |   10.910000 |     10.870000 |     10.970000 |    10.810000 | 1.65852e+12 |         1349 |
| CHNA   | Loncar China BioPharma ETF                                                                                                                                                      | stocks | ETF  | BBG00LPXSYJ8   | BBG00LPXSZ96     |   3.250000e+02 |          19.3287000 |   19.250000 |     19.292800 |     19.422400 |    19.250000 | 1.65852e+12 |            9 |
| OPOF   | Old Point Financial Corp                                                                                                                                                        | stocks | CS   | BBG000BSBQ78   | BBG001SB1L31     |   3.568000e+03 |          24.3006000 |   22.750000 |     24.500000 |     25.019900 |    22.750000 | 1.65852e+12 |          101 |
| TME    | Tencent Music Entertainment Group American Depositary Shares, each representing two Class A Ordinary Shares                                                                     | stocks | ADRC | BBG00LDC5RK5   | BBG00M5B6BG3     |   5.119727e+06 |           4.5722000 |    4.600000 |      4.490000 |      4.730000 |     4.480000 | 1.65852e+12 |        12558 |
| ZUO    | Zuora, Inc.                                                                                                                                                                     | stocks | CS   | BBG000BT3HG5   | BBG001SS0FH3     |   8.038090e+05 |           8.9192000 |    9.230000 |      8.960000 |      9.260000 |     8.750000 | 1.65852e+12 |         8294 |
| AKTX   | Akari Therapeutics plc ADR (0.01 USD)                                                                                                                                           | stocks | ADRC | BBG003PMV495   | BBG003PMV510     |   5.128800e+04 |           0.8802900 |    0.830000 |      0.910100 |      0.910100 |     0.830000 | 1.65852e+12 |          146 |
| DE     | Deere & Company                                                                                                                                                                 | stocks | CS   | BBG000BH1NH9   | BBG001S5QFF7     |   1.110725e+06 |         313.6991000 |  317.840000 |    312.260000 |    320.470000 |   310.460000 | 1.65852e+12 |        24252 |
| UTMD   | Utah Medical Products Inc                                                                                                                                                       | stocks | CS   | BBG000HFDNW7   | BBG001S6PWD1     |   1.069200e+04 |          87.1325000 |   87.960000 |     85.600000 |     90.030000 |    85.600000 | 1.65852e+12 |          473 |
| B      | Barnes Group Inc.                                                                                                                                                               | stocks | CS   | BBG000BCSCB1   | BBG001S5P0S7     |   1.256600e+05 |          32.5354000 |   32.700000 |     32.560000 |     32.900000 |    32.200000 | 1.65852e+12 |         2730 |
| TBI    | Trueblue, Inc.                                                                                                                                                                  | stocks | CS   | BBG000BP69N0   | BBG001SC5XD8     |   1.166240e+05 |          18.9093000 |   19.580000 |     18.900000 |     19.610000 |    18.680000 | 1.65852e+12 |         2270 |
| POAI   | Predictive Oncology Inc. Common Stock                                                                                                                                           | stocks | CS   | BBG000PVXTF7   | BBG001T619N6     |   3.432780e+05 |           0.4164400 |    0.449300 |      0.412100 |      0.449300 |     0.400600 | 1.65852e+12 |          941 |
| CCD    | Calamos Dynamic Convertible & Income Fund                                                                                                                                       | stocks | CS   | BBG0065XMFL7   | BBG0065XMFM6     |   4.608500e+04 |          23.1524000 |   23.390000 |     23.070000 |     23.530000 |    22.947300 | 1.65852e+12 |          342 |
| UFEB   | Innovator U.S. Equity Ultra Buffer ETF - February                                                                                                                               | stocks | ETF  | BBG00RHZ8M98   | BBG00RHZ8N23     |   1.459400e+04 |          26.4678000 |   26.520000 |     26.459000 |     26.520000 |    26.440000 | 1.65852e+12 |           49 |
| QTT    | Qutoutiao Inc. American Depositary Shares                                                                                                                                       | stocks | ADRC | BBG00LQM8ZT9   | BBG00LQM90K4     |   2.640900e+04 |           1.0920000 |    1.080000 |      1.100000 |      1.126100 |     1.070000 | 1.65852e+12 |          157 |
| EXAS   | Exact Sciences Corp                                                                                                                                                             | stocks | CS   | BBG000CWL0F5   | BBG001SGCLB9     |   1.521737e+06 |          45.9885000 |   48.130000 |     45.390000 |     48.300000 |    45.170000 | 1.65852e+12 |        23683 |
| SIXH   | ETC 6 Meridian Hedged Equity Index Option ETF                                                                                                                                   | stocks | ETF  | BBG00TQWHCD8   | BBG00TQWHD55     |   8.263000e+03 |          29.7782000 |   29.780000 |     29.673400 |     29.780000 |    29.673400 | 1.65852e+12 |           13 |
| RCMT   | RCM Technologies Inc                                                                                                                                                            | stocks | CS   | BBG000BRYSR9   | BBG001S5VLS4     |   9.275000e+04 |          17.6275000 |   18.850000 |     17.350000 |     18.850000 |    17.155000 | 1.65852e+12 |         1570 |
| ASPY   | ASYMshares ASYMmetric S&P 500 ETF                                                                                                                                               | stocks | ETF  | BBG00ZL7QQ07   | BBG00ZL7QQV3     |   4.064000e+03 |          26.8157000 |   26.820000 |     26.833600 |     26.850000 |    26.800000 | 1.65852e+12 |           20 |
| IZEA   | IZEA Worldwide, Inc. Common Stock                                                                                                                                               | stocks | CS   | BBG0018XMPL3   | BBG001STRP71     |   1.411080e+05 |           0.9953000 |    1.030000 |      0.962400 |      1.050000 |     0.940000 | 1.65852e+12 |          424 |
| CQQQ   | Invesco China Technology ETF                                                                                                                                                    | stocks | ETF  | BBG00KXH51H5   | BBG00KXH5265     |   3.617370e+05 |          46.7695000 |   47.400000 |     46.660000 |     47.470000 |    46.530000 | 1.65852e+12 |         1698 |
| CDNS   | Cadence Design Systems                                                                                                                                                          | stocks | CS   | BBG000C13CD9   | BBG001S65YK1     |   1.804334e+06 |         168.1360000 |  169.460000 |    167.710000 |    171.110000 |   167.230000 | 1.65852e+12 |        33427 |
| CURV   | Torrid Holdings Inc.                                                                                                                                                            | stocks | CS   | BBG011C0PCJ8   | BBG011C0PDC3     |   2.841780e+05 |           4.5480000 |    4.660000 |      4.350000 |      5.058800 |     4.320000 | 1.65852e+12 |         3380 |
| SYBX   | Synlogic, Inc. Common Stock                                                                                                                                                     | stocks | CS   | BBG000R33JD4   | BBG001T92245     |   3.313100e+04 |           1.0298000 |    1.010000 |      1.040000 |      1.040000 |     1.010000 | 1.65852e+12 |           99 |
| CURI   | CuriosityStream Inc. Class A Common Stock                                                                                                                                       | stocks | CS   | BBG00QS5N3V4   | BBG00QS5N3W3     |   1.221260e+05 |           1.9172000 |    1.960000 |      1.920000 |      1.990000 |     1.870000 | 1.65852e+12 |         1149 |
| YCL    | ProShares Ultra Yen                                                                                                                                                             | stocks | ETV  | BBG000CT88J6   | BBG001T0D5T8     |   6.051000e+03 |          33.2569000 |   33.027700 |     33.256900 |     33.420000 |    33.027700 | 1.65852e+12 |           76 |
| AEMD   | AETHLON MEDICAL INC                                                                                                                                                             | stocks | CS   | BBG000L78XT8   | BBG001SDDHJ6     |   1.000960e+05 |           1.0460000 |    1.080000 |      1.060000 |      1.080000 |     1.030000 | 1.65852e+12 |          267 |
| IAU    | iShares Gold Trust                                                                                                                                                              | stocks | ETV  | BBG000QLKDR4   | BBG001SJK6D5     |   5.483335e+06 |          32.8425000 |   32.718600 |     32.750000 |     33.036600 |    32.670100 | 1.65852e+12 |        11176 |
| FATE   | Fate Therapeutics, Inc.                                                                                                                                                         | stocks | CS   | BBG000QP35H2   | BBG001T888S2     |   1.277417e+06 |          31.8227000 |   33.590000 |     31.040000 |     33.840000 |    30.975000 | 1.65852e+12 |        25001 |
| RGNX   | REGENXBIO Inc.                                                                                                                                                                  | stocks | CS   | BBG007Z9V591   | BBG007Z9V5C7     |   3.153370e+05 |          31.2997000 |   32.260000 |     31.170000 |     32.260000 |    31.050000 | 1.65852e+12 |         4596 |
| PW     | Power REIT                                                                                                                                                                      | stocks | CS   | BBG000BRS4P1   | BBG001S5VH49     |   2.441300e+04 |          17.4299000 |   17.690000 |     17.380000 |     18.000000 |    17.000000 | 1.65852e+12 |          595 |
| OLP    | One Liberty Properties, Inc.                                                                                                                                                    | stocks | CS   | BBG000BQJHF5   | BBG001S5TWY6     |   2.831800e+04 |          26.4165000 |   26.220000 |     26.460000 |     26.721400 |    26.200000 | 1.65852e+12 |          650 |
| SPXU   | ProShares UltraPro Short S&P 500                                                                                                                                                | stocks | ETF  | BBG000N2ZYJ6   | BBG001T51PV3     |   2.622238e+07 |          17.3066000 |   16.980000 |     17.470000 |     17.760000 |    16.805000 | 1.65852e+12 |        33966 |
| NRO    | Neuberger Berman Real Estate Sec. Income Fund Inc.                                                                                                                              | stocks | FUND | BBG000C2VH63   | BBG001SDQJG1     |   4.087500e+04 |           4.1258000 |    4.090000 |      4.100000 |      4.150000 |     4.090000 | 1.65852e+12 |          159 |
| DFAX   | Dimensional World ex U.S. Core Equity 2 ETF                                                                                                                                     | stocks | ETF  | BBG012G2PC71   | BBG012G2PD33     |   1.141059e+06 |          21.4769000 |   21.620000 |     21.460000 |     21.690000 |    21.390000 | 1.65852e+12 |         1925 |
| FLJH   | Franklin FTSE Japan Hedged ETF                                                                                                                                                  | stocks | ETF  | BBG00J3MMYR8   | BBG00J3MMZG7     |   8.162000e+03 |          30.2417000 |   30.390000 |     30.140000 |     30.390000 |    30.100000 | 1.65852e+12 |           41 |
| PHAR   | Pharming Group N.V. ADS, each representing 10 ordinary shares                                                                                                                   | stocks | CS   | BBG00YN70VB3   | BBG00YN70W58     |   1.370000e+02 |           8.0874000 |    8.100000 |      8.100000 |      8.100000 |     8.100000 | 1.65852e+12 |            8 |
| CBFV   | CB Financial Services, Inc. (PA)                                                                                                                                                | stocks | CS   | BBG000BKT8B1   | BBG001S825Y8     |   8.140000e+02 |          22.9475000 |   22.770200 |     23.130000 |     23.130000 |    22.770200 | 1.65852e+12 |           35 |
| HYPR   | Hyperfine, Inc. Class A Common Stock                                                                                                                                            | stocks | CS   | BBG00YVTN9J2   | BBG00YVTN9K0     |   2.019500e+05 |           1.5559000 |    1.690000 |      1.490000 |      1.710000 |     1.480000 | 1.65852e+12 |         1073 |
| KINS   | Kingstone Companies, Inc.                                                                                                                                                       | stocks | CS   | BBG000G21FS2   | BBG001SCHWG4     |   1.865500e+04 |           3.8175000 |    3.730000 |      3.850000 |      3.990000 |     3.724400 | 1.65852e+12 |          113 |
| INTC   | Intel Corp                                                                                                                                                                      | stocks | CS   | BBG000C0G1D1   | BBG001S5SF65     |   4.134997e+07 |          39.4340000 |   40.370000 |     39.200000 |     40.510000 |    38.940000 | 1.65852e+12 |       223720 |
| BSCQ   | Invesco BulletShares 2026 Corporate Bond ETF                                                                                                                                    | stocks | ETF  | BBG00KJR2JD7   | BBG00KJR2K35     |   2.034870e+05 |          19.4807000 |   19.470000 |     19.485000 |     19.515000 |    19.458600 | 1.65852e+12 |          846 |
| ZD     | Ziff Davis, Inc. Common Stock                                                                                                                                                   | stocks | CS   | BBG000F3CWW7   | BBG001SD21P6     |   3.519000e+05 |          81.4928000 |   82.150000 |     81.220000 |     83.420000 |    80.510000 | 1.65852e+12 |         8794 |
| SSTK   | SHUTTERSTOCK, INC.                                                                                                                                                              | stocks | CS   | BBG002ZCK2V9   | BBG002ZCK3M7     |   3.000080e+05 |          60.3583000 |   62.450000 |     60.090000 |     63.000000 |    59.630000 | 1.65852e+12 |         7938 |
| JPRE   | JPMorgan Realty Income ETF                                                                                                                                                      | stocks | ETF  | BBG016CBQ1J6   | BBG016CBQ2K2     |   6.621000e+03 |          49.8365000 |   49.890000 |     49.832300 |     50.290000 |    49.620000 | 1.65852e+12 |           55 |
| MZZ    | ProShares UltraShort MidCap400                                                                                                                                                  | stocks | ETF  | BBG000PTHM74   | BBG001SRDT90     |   1.255000e+03 |          18.6447000 |   18.515000 |     18.886500 |     18.886500 |    18.441000 | 1.65852e+12 |           25 |
| NVVE   | Nuvve Holding Corp. Common Stock                                                                                                                                                | stocks | CS   | BBG00Z49PKM2   | BBG00Z49PKN1     |   2.343210e+05 |           4.0459000 |    4.300000 |      3.890000 |      4.330000 |     3.880000 | 1.65852e+12 |         1764 |
| TMX    | Terminix Global Holdings, Inc.                                                                                                                                                  | stocks | CS   | BBG002WMH2F2   | BBG002WMH2G1     |   1.380274e+06 |          42.9378000 |   42.920000 |     42.730000 |     43.230000 |    42.715000 | 1.65852e+12 |         5452 |
| DFAE   | Dimensional Emerging Core Equity Market ETF                                                                                                                                     | stocks | ETF  | BBG00Y2PHV55   | BBG00Y2PHW26     |   4.046680e+05 |          22.7396000 |   22.920000 |     22.750000 |     22.920000 |    22.645000 | 1.65852e+12 |         1074 |
| PINK   | Simplify Health Care ETF                                                                                                                                                        | stocks | ETF  | BBG012VCQ1M8   | BBG012VCQ2K8     |   1.687000e+03 |          25.1669000 |   25.150000 |     25.131700 |     25.200000 |    25.131700 | 1.65852e+12 |           14 |
| NRXP   | NRX Pharmaceuticals, Inc. Common Stock                                                                                                                                          | stocks | CS   | BBG00JCWDN63   | BBG00JCWDNX3     |   3.012780e+05 |           0.5347900 |    0.540000 |      0.536000 |      0.560000 |     0.520000 | 1.65852e+12 |          591 |
| AVB    | AvalonBay Communities, Inc.                                                                                                                                                     | stocks | CS   | BBG000BLPBL5   | BBG001S7J2H8     |   4.723600e+05 |         198.3286000 |  196.940000 |    198.250000 |    199.520000 |   196.010000 | 1.65852e+12 |        14056 |
| JPEM   | JPMorgan Diversified Return Emerging Markets Equity ETF                                                                                                                         | stocks | ETF  | BBG007VDWC18   | BBG007VDWC09     |   1.742400e+04 |          48.4136000 |   48.479900 |     48.260000 |     48.580000 |    48.160000 | 1.65852e+12 |          135 |
| MHO    | M/I Homes, Inc.                                                                                                                                                                 | stocks | CS   | BBG000BL9MZ4   | BBG001S7HGC4     |   2.266100e+05 |          46.1729000 |   45.950000 |     46.230000 |     46.900000 |    45.310000 | 1.65852e+12 |         5511 |
| LFVN   | Lifevantage Corporation Common Stock (Delaware)                                                                                                                                 | stocks | CS   | BBG000CDJHR0   | BBG001S6TDX7     |   3.827300e+04 |           4.2466000 |    4.230000 |      4.240000 |      4.340000 |     4.230000 | 1.65852e+12 |          387 |
| CARG   | CarGurus, Inc. Class A Common Stock                                                                                                                                             | stocks | CS   | BBG00HQ77DS2   | BBG00HQ77DT1     |   8.256380e+05 |          23.8812000 |   25.020000 |     23.770000 |     25.140000 |    23.570800 | 1.65852e+12 |        10711 |
| ROM    | ProShares Ultra Technology                                                                                                                                                      | stocks | ETF  | BBG000QXKPR1   | BBG001SSTD78     |   1.041770e+05 |          32.6848000 |   33.650000 |     32.310000 |     33.880000 |    31.956100 | 1.65852e+12 |          866 |
| HTY    | JOHN HANCOCK TAX-ADVANTAGED GLOBAL SHAREHOLDER YIELD FUND                                                                                                                       | stocks | FUND | BBG000RRMJ04   | BBG001SV51C7     |   2.724600e+04 |           5.3361000 |    5.380000 |      5.300000 |      5.390000 |     5.300000 | 1.65852e+12 |          185 |
| ORGO   | Organogenesis Holdings Inc. Class A Common Stock                                                                                                                                | stocks | CS   | BBG00FGGD295   | BBG00FGGD311     |   6.634090e+05 |           5.4027000 |    5.550000 |      5.370000 |      5.600000 |     5.300000 | 1.65852e+12 |         4076 |
| NUE    | Nucor Corporation                                                                                                                                                               | stocks | CS   | BBG000BQ8KV2   | BBG001S5TRV0     |   3.445227e+06 |         122.8929000 |  129.200000 |    119.860000 |    130.720000 |   119.270000 | 1.65852e+12 |        57502 |
| PECO   | Phillips Edison & Company, Inc. Common Stock                                                                                                                                    | stocks | CS   | BBG011RJSSB1   | BBG011RJST56     |   6.040430e+05 |          33.7685000 |   33.880000 |     33.580000 |     34.090000 |    33.450000 | 1.65852e+12 |         7504 |
| EVTC   | EVERTEC, INC.                                                                                                                                                                   | stocks | CS   | BBG000J187K0   | BBG001SX2XR0     |   1.746040e+05 |          37.7559000 |   38.110000 |     37.830000 |     38.250000 |    37.390000 | 1.65852e+12 |         4220 |
| VHC    | VirnetX Holding Corporation                                                                                                                                                     | stocks | CS   | BBG000BP25X1   | BBG001S7FPB7     |   5.251350e+05 |           1.7390000 |    1.650000 |      1.810000 |      1.840000 |     1.620000 | 1.65852e+12 |         3651 |
| ESML   | iShares ESG Aware MSCI USA Small-Cap ETF                                                                                                                                        | stocks | ETF  | BBG00KK876G7   | BBG00KK87757     |   8.296600e+04 |          33.4016000 |   33.830000 |     33.360000 |     33.893700 |    33.165000 | 1.65852e+12 |          546 |
| IUSV   | iShares Core S&P U.S. Value ETF                                                                                                                                                 | stocks | ETF  | BBG000C183Y1   | BBG001SFQL80     |   5.979480e+05 |          68.6308000 |   68.970000 |     68.560000 |     69.159600 |    68.190000 | 1.65852e+12 |         1755 |
| MRKR   | Marker Therapeutics, Inc. Common Stock                                                                                                                                          | stocks | CS   | BBG000GMZP09   | BBG001SKLLN7     |   9.827200e+04 |           0.3411100 |    0.346500 |      0.339300 |      0.350000 |     0.335000 | 1.65852e+12 |          214 |
| NUWE   | Nuwellis, Inc. Common Stock                                                                                                                                                     | stocks | CS   | BBG000QVFRH8   | BBG001SNT6T0     |   1.972200e+04 |           0.6310900 |    0.639100 |      0.638037 |      0.639100 |     0.626100 | 1.65852e+12 |           82 |
| IBHE   | iShares iBonds 2025 Term High Yield and Income ETF                                                                                                                              | stocks | ETF  | BBG00P3CSJR3   | BBG00P3CSKH1     |   1.024600e+04 |          23.0685000 |   22.990000 |     22.905000 |     23.160000 |    22.905000 | 1.65852e+12 |           80 |
| IE     | Ivanhoe Electric Inc.                                                                                                                                                           | stocks | CS   | BBG012WFRMG0   | BBG012WFRMH9     |   1.582020e+05 |           8.2686000 |    7.010000 |      8.500000 |      8.650000 |     7.010000 | 1.65852e+12 |         1546 |
| SONN   | Sonnet BioTherapeutics Holdings, Inc. Common Stock                                                                                                                              | stocks | CS   | BBG000BVZ4N6   | BBG001SJ3JN5     |   4.149729e+04 |           4.1175000 |    4.419800 |      4.128600 |      4.419800 |     3.920000 | 1.65852e+12 |          554 |
| BGNE   | BeiGene, Ltd. American Depositary Shares                                                                                                                                        | stocks | ADRC | BBG00B6WF7T5   | BBG00B6WF7V2     |   2.376910e+05 |         182.0008000 |  184.090000 |    180.570000 |    187.750000 |   179.100000 | 1.65852e+12 |         5997 |
| CMCSA  | Comcast Corp                                                                                                                                                                    | stocks | CS   | BBG000BFT2L4   | BBG001S5PXL2     |   2.409857e+07 |          42.5382000 |   42.410000 |     42.600000 |     42.910000 |    42.220000 | 1.65852e+12 |       116309 |
| WSC    | WillScot Mobile Mini Holdings Corp. Class A Common Stock                                                                                                                        | stocks | CS   | BBG00B0FS947   | BBG00B0FS9V7     |   1.362282e+06 |          35.7272000 |   36.190000 |     35.700000 |     36.400000 |    34.390000 | 1.65852e+12 |        17856 |
| EBTC   | Enterprise Bancorp Inc.                                                                                                                                                         | stocks | CS   | BBG000BLT2T3   | BBG001SB6470     |   6.695000e+03 |          32.4161000 |   32.540000 |     32.160000 |     32.690000 |    32.160000 | 1.65852e+12 |          194 |
| XMTR   | Xometry, Inc. Class A Common Stock                                                                                                                                              | stocks | CS   | BBG00BDCCV93   | BBG00BDCCVB0     |   2.943200e+05 |          37.3367000 |   38.060000 |     36.890000 |     39.210000 |    36.430000 | 1.65852e+12 |         6123 |
| WFG    | West Fraser Timber Co. Ltd                                                                                                                                                      | stocks | CS   | BBG000D6PCT6   | BBG001S60WD8     |   4.342910e+05 |          97.4820000 |   97.770000 |     97.880000 |     98.950000 |    96.206000 | 1.65852e+12 |         7347 |
| PGR    | Progressive Corporation                                                                                                                                                         | stocks | CS   | BBG000BR37X2   | BBG001S5V509     |   1.697854e+06 |         111.3109000 |  111.580000 |    111.190000 |    112.270000 |   110.750000 | 1.65852e+12 |        22258 |
| PAG    | Penske Automotive Group, Inc.                                                                                                                                                   | stocks | CS   | BBG000H6K1B0   | BBG001S96493     |   3.859350e+05 |         110.4567000 |  110.100000 |    110.060000 |    112.660000 |   109.560000 | 1.65852e+12 |        10274 |
| HITI   | High Tide Inc. Common Shares                                                                                                                                                    | stocks | CS   | BBG00N6WFFQ4   | BBG00MF4XPP7     |   3.106830e+05 |           1.5851000 |    1.600000 |      1.550000 |      1.640000 |     1.550000 | 1.65852e+12 |         1260 |
| GRES   | IQ ARB Global Resources                                                                                                                                                         | stocks | ETF  | BBG000PR0MS3   | BBG001T5WCN7     |   1.451000e+03 |          30.3485000 |   30.392800 |     30.270000 |     30.392800 |    30.200000 | 1.65852e+12 |           41 |
| CPRX   | Catalyst Pharmaceutical Inc.                                                                                                                                                    | stocks | CS   | BBG000GZDC67   | BBG001SRHS16     |   2.203677e+06 |           9.6560000 |    9.880000 |      9.550000 |      9.882600 |     9.535000 | 1.65852e+12 |        17468 |
| ACRX   | AcelRx Pharmaceuticals, Inc                                                                                                                                                     | stocks | CS   | BBG0018YYFX7   | BBG001TFZCK5     |   1.044848e+06 |           0.2390100 |    0.254400 |      0.234400 |      0.258900 |     0.225501 | 1.65852e+12 |         1527 |
| PACB   | Pacific Biosciences of California, Inc.                                                                                                                                         | stocks | CS   | BBG000QKXH20   | BBG001SS9G64     |   8.954032e+06 |           4.5415000 |    4.840000 |      4.470000 |      5.045000 |     4.280000 | 1.65852e+12 |        40885 |
| UDI    | USCF Dividend Income Fund                                                                                                                                                       | stocks | ETF  | BBG0180D79X9   | BBG0180D7BT9     |   1.500000e+01 |          24.3500000 |   24.187900 |     24.187900 |     24.187900 |    24.187900 | 1.65852e+12 |            1 |
| ACHR   | Archer Aviation Inc.                                                                                                                                                            | stocks | CS   | BBG00XRTC910   | BBG00XRTC929     |   1.206484e+06 |           3.2367000 |    3.470000 |      3.150000 |      3.490000 |     3.140000 | 1.65852e+12 |         6600 |
| IOT    | Samsara Inc.                                                                                                                                                                    | stocks | CS   | BBG0099PW5P1   | BBG0099PW5Q0     |   1.000634e+06 |          14.3485000 |   15.060000 |     14.220000 |     15.185000 |    14.020100 | 1.65852e+12 |        10282 |
| DSTX   | Distillate International Fundamental Stability & Value ETF                                                                                                                      | stocks | ETF  | BBG00YJ721V0   | BBG00YJ722P5     |   8.350000e+02 |          21.2221000 |   21.250000 |     21.182800 |     21.250000 |    21.140000 | 1.65852e+12 |            2 |
| LEVI   | Levi Strauss & Co. Class A Common Stock                                                                                                                                         | stocks | CS   | BBG000BQDF10   | BBG001S9NVN8     |   1.500001e+06 |          19.3050000 |   19.380000 |     19.400000 |     19.450000 |    19.143800 | 1.65852e+12 |        16789 |
| CNXT   | VanEck ChiNext ETF                                                                                                                                                              | stocks | ETF  | BBG006W461W8   | BBG006W461V9     |   9.316000e+03 |          37.4229000 |   37.600000 |     37.367600 |     37.600000 |    37.330000 | 1.65852e+12 |           64 |
| POWL   | Powell Industries Inc                                                                                                                                                           | stocks | CS   | BBG000BRGWN4   | BBG001S5VBT5     |   2.519300e+04 |          23.9743000 |   24.200000 |     23.930000 |     24.410000 |    23.530000 | 1.65852e+12 |          622 |
| TRFM   | AAM Transformers ETF                                                                                                                                                            | stocks | ETF  | BBG018QSQ1S0   | BBG018QSQ2M4     |   1.200000e+02 |          25.9000000 |   25.900000 |     24.964700 |     25.900000 |    24.964700 | 1.65852e+12 |            1 |
| ATXI   | Avenue Therapeutics, Inc. Common Stock                                                                                                                                          | stocks | CS   | BBG00GLTS4V4   | BBG00GLTS5K3     |   8.875200e+03 |           4.4504000 |    4.650000 |      4.374000 |      4.662000 |     4.201500 | 1.65852e+12 |          235 |
| QIPT   | Quipt Home Medical Corp. Ordinary Shares                                                                                                                                        | stocks | CS   | BBG000HV7H25   | BBG001SHK710     |   2.384500e+04 |           5.3537000 |    5.400000 |      5.390000 |      5.400000 |     5.338900 | 1.65852e+12 |          118 |
| SRTY   | ProShares UltraPro Short Russell2000                                                                                                                                            | stocks | ETF  | BBG000QBF8X6   | BBG001T6S4P6     |   1.419916e+06 |          58.8670000 |   56.580000 |     59.720000 |     61.010000 |    56.250000 | 1.65852e+12 |        11643 |
| FACA   | Figure Acquisition Corp. I                                                                                                                                                      | stocks | CS   | BBG00Z6BFKL7   | BBG00Z6BFKP3     |   4.024000e+04 |           9.8473000 |    9.850000 |      9.850000 |      9.850000 |     9.830000 | 1.65852e+12 |           20 |
| MPC    | MARATHON PETROLEUM CORPORATION                                                                                                                                                  | stocks | CS   | BBG001DCCGR8   | BBG001S169P1     |   3.950018e+06 |          86.0291000 |   86.590000 |     85.650000 |     87.780000 |    85.140000 | 1.65852e+12 |        50592 |
| ATNF   | 180 Life Sciences Corp. Common Stock                                                                                                                                            | stocks | CS   | BBG00H1CP0B4   | BBG00H1CP113     |   1.396360e+05 |           1.2132000 |    1.270000 |      1.200000 |      1.299400 |     1.150000 | 1.65852e+12 |          659 |
| CPS    | Cooper-Standard Automotive Inc.                                                                                                                                                 | stocks | CS   | BBG000PSXT64   | BBG001SRCY63     |   2.360130e+05 |           5.0249000 |    5.450000 |      5.000000 |      5.520000 |     4.760000 | 1.65852e+12 |         2741 |
| NCA    | Nuveen California Municipal Value Fund                                                                                                                                          | stocks | FUND | BBG000BPV792   | BBG001S5TJV8     |   1.988400e+04 |           8.9203000 |    8.880000 |      8.940000 |      8.949900 |     8.868200 | 1.65852e+12 |           95 |
| GBCI   | Glacier Bancorp Inc                                                                                                                                                             | stocks | CS   | BBG000C3KB84   | BBG001S6HGM5     |   3.457870e+05 |          48.3326000 |   49.570000 |     48.040000 |     49.570000 |    47.690000 | 1.65852e+12 |         7393 |
| LVS    | Las Vegas Sands Corp.                                                                                                                                                           | stocks | CS   | BBG000JWD753   | BBG001SJCGP9     |   5.934052e+06 |          38.9868000 |   39.410000 |     38.990000 |     39.650000 |    38.660000 | 1.65852e+12 |        52240 |
| RCI    | Rogers Communications, Inc.                                                                                                                                                     | stocks | CS   | BBG000BSZ3J0   | BBG001S604S4     |   2.880060e+05 |          46.7682000 |   46.620000 |     46.780000 |     47.270000 |    46.440000 | 1.65852e+12 |         4393 |
| EVEN   | Direxion Daily S&P 500 Equal Weight Bull 2X Shares                                                                                                                              | stocks | ETF  | BBG014L04T62   | BBG014L04VD9     |   5.140000e+02 |          19.6542000 |   19.670000 |     19.436300 |     19.740000 |    19.380000 | 1.65852e+12 |           11 |
| IBKR   | Interactive Brokers Group, Inc. Class A Common Stock                                                                                                                            | stocks | CS   | BBG000LV0836   | BBG001SQ4YC7     |   6.689600e+05 |          56.4991000 |   56.620000 |     56.440000 |     57.240000 |    55.900000 | 1.65852e+12 |        13608 |
| EZPW   | Ezcorp Inc                                                                                                                                                                      | stocks | CS   | BBG000C93HM1   | BBG001S6RN67     |   1.924860e+05 |           7.4162000 |    7.500000 |      7.470000 |      7.560000 |     7.320000 | 1.65852e+12 |         2877 |
| LAC    | Lithium Americas Corp. Common Shares                                                                                                                                            | stocks | CS   | BBG000BGM5P8   | BBG001SJCRY5     |   3.216109e+06 |          23.4360000 |   24.090000 |     22.620000 |     24.210000 |    22.513700 | 1.65852e+12 |        23980 |
| WEYS   | Weyco Group Inc                                                                                                                                                                 | stocks | CS   | BBG000BWQ4C6   | BBG001S5XF05     |   1.129600e+04 |          25.9193000 |   25.600000 |     25.810000 |     26.170000 |    25.600000 | 1.65852e+12 |          687 |
| OPHC   | OptimumBank Holdings, Inc.                                                                                                                                                      | stocks | CS   | BBG000PQV9V6   | BBG001SLNQD3     |   8.400000e+03 |           3.5208000 |    3.480000 |      3.612900 |      3.612900 |     3.410000 | 1.65852e+12 |          111 |
| VSPY   | VectorShares Min Vol ETF                                                                                                                                                        | stocks | ETF  | BBG011WH0KP1   | BBG011WH0LJ6     |   1.167420e+05 |           9.5312000 |    9.540000 |      9.495000 |      9.540000 |     9.430000 | 1.65852e+12 |           26 |
| INMB   | INmune Bio Inc. Common stock                                                                                                                                                    | stocks | CS   | BBG00LW94L98   | BBG00LW94M14     |   6.091400e+04 |           9.4048000 |   10.060000 |      9.330000 |     10.060000 |     9.100000 | 1.65852e+12 |          390 |
| ISDR   | Issuer Direct Corporation                                                                                                                                                       | stocks | CS   | BBG000C1CK87   | BBG001S67C09     |   5.959000e+03 |          25.1624000 |   24.850000 |     25.280000 |     25.510000 |    24.850000 | 1.65852e+12 |          296 |
| CPII   | Ionic Inflation Protection ETF                                                                                                                                                  | stocks | ETF  | BBG018CNTGT9   | BBG018CNTHQ0     |   1.201000e+03 |          20.1148000 |   20.090000 |     20.057500 |     20.119900 |    20.057500 | 1.65852e+12 |            4 |
| NET    | Cloudflare, Inc. Class A common stock, par value \$0.001 per share                                                                                                              | stocks | CS   | BBG001WMKHH5   | BBG001WMKHJ3     |   3.778593e+06 |          53.0054000 |   55.180000 |     51.650000 |     56.298800 |    51.490000 | 1.65852e+12 |        46789 |
| BCOV   | Brightcove, Inc.                                                                                                                                                                | stocks | CS   | BBG000GZH540   | BBG001SP29R0     |   1.028920e+05 |           6.1361000 |    6.370000 |      6.050000 |      6.545000 |     6.030000 | 1.65852e+12 |         2137 |
| CUBE   | CubeSmart                                                                                                                                                                       | stocks | CS   | BBG000HF28Q9   | BBG001SHP0D7     |   5.576460e+05 |          43.6190000 |   43.710000 |     43.620000 |     44.040000 |    43.300000 | 1.65852e+12 |         9198 |
| FDRR   | Fidelity Dividend ETF for Rising Rates                                                                                                                                          | stocks | ETF  | BBG00DR7RKW7   | BBG00DR7RKX6     |   2.907500e+04 |          40.0986000 |   40.230000 |     40.050000 |     40.330000 |    39.797000 | 1.65852e+12 |          298 |
| DXD    | ProShares UltraShort Dow 30                                                                                                                                                     | stocks | ETF  | BBG000PTJ1T4   | BBG001SRDTJ9     |   6.621620e+05 |          48.2448000 |   47.650000 |     48.450000 |     48.995000 |    47.560000 | 1.65852e+12 |         4229 |
| EXPO   | Exponent Inc                                                                                                                                                                    | stocks | CS   | BBG000F31Z34   | BBG001S9CG99     |   1.645200e+05 |          93.7100000 |   94.540000 |     93.810000 |     95.330000 |    92.520000 | 1.65852e+12 |         4383 |
| SFE    | Safeguard Scientifics, Inc.                                                                                                                                                     | stocks | CS   | BBG000BSSN13   | BBG001S5W0J9     |   5.342000e+03 |           4.0925000 |    4.060000 |      4.100000 |      4.100000 |     3.930000 | 1.65852e+12 |           19 |
| MXF    | MEXICO FUND                                                                                                                                                                     | stocks | FUND | BBG000BPN8R9   | BBG001S5TGW3     |   3.561000e+03 |          14.0399000 |   14.030000 |     14.040000 |     14.090000 |    13.978600 | 1.65852e+12 |           28 |
| ORA    | Ormat Technologies, Inc.                                                                                                                                                        | stocks | CS   | BBG000Q5BQ63   | BBG001S9LPW3     |   4.458340e+05 |          79.7973000 |   80.110000 |     80.150000 |     80.990000 |    78.730000 | 1.65852e+12 |         6464 |
| ZBH    | Zimmer Biomet Holdings, Inc.                                                                                                                                                    | stocks | CS   | BBG000BKPL53   | BBG001S7DQJ9     |   1.140931e+06 |         107.4847000 |  108.320000 |    107.000000 |    109.210000 |   106.630000 | 1.65852e+12 |        18005 |
| FWBI   | First Wave BioPharma, Inc. Common Stock                                                                                                                                         | stocks | CS   | BBG00DBBJ4T4   | BBG00DBBJ4V1     |   9.908480e+04 |           5.1549000 |    5.400000 |      5.064000 |      5.667000 |     4.839000 | 1.65852e+12 |         3173 |
| PSCF   | Invesco S&P SmallCap Financials ETF                                                                                                                                             | stocks | ETF  | BBG000QM6ZZ0   | BBG001T7V3D5     |   1.198000e+03 |          50.6515000 |   50.686600 |     50.369200 |     50.790000 |    50.157300 | 1.65852e+12 |           10 |
| FGM    | First Trust Germany AlphaDEX Fund                                                                                                                                               | stocks | ETF  | BBG002N8WN24   | BBG002N8WNT5     |   2.013000e+03 |          33.9515000 |   33.950000 |     33.740000 |     33.980000 |    33.740000 | 1.65852e+12 |           21 |
| THY    | Toews Agility Shares Dynamic Tactical Income ETF                                                                                                                                | stocks | ETF  | BBG00S7P3N01   | BBG00S7P3NR2     |   7.880000e+02 |          23.1209000 |   23.160000 |     23.101000 |     23.160000 |    23.050000 | 1.65852e+12 |            6 |
| SEEL   | Seelos Therapeutics, Inc. Common Stock                                                                                                                                          | stocks | CS   | BBG000BZ9N90   | BBG001S7J019     |   5.590320e+05 |           0.9358900 |    0.960000 |      0.925300 |      0.976000 |     0.908000 | 1.65852e+12 |         2248 |
| UPV    | ProShares Ultra FTSE Europe                                                                                                                                                     | stocks | ETF  | BBG000QTLVQ7   | BBG001T8LKD6     |   1.255000e+03 |          43.7239000 |   43.113900 |     43.113900 |     43.113900 |    43.113900 | 1.65852e+12 |           57 |
| NKE    | Nike, Inc.                                                                                                                                                                      | stocks | CS   | BBG000C5HS04   | BBG001S6NTK2     |   6.039269e+06 |         109.6357000 |  111.930000 |    109.120000 |    111.930000 |   108.750000 | 1.65852e+12 |        81816 |
| EQC    | Equity Commonwealth                                                                                                                                                             | stocks | CS   | BBG000BLG1L7   | BBG001S5S0L1     |   3.794150e+05 |          27.2311000 |   27.240000 |     27.270000 |     27.320000 |    27.060000 | 1.65852e+12 |         5492 |
| SBNY   | Signature Bank                                                                                                                                                                  | stocks | CS   | BBG000M6TR37   | BBG001SGW2S1     |   1.045163e+06 |         176.3908000 |  181.570000 |    175.890000 |    183.910000 |   172.690300 | 1.65852e+12 |        24869 |
| HL     | Hecla Mining Company                                                                                                                                                            | stocks | CS   | BBG000BL5W86   | BBG001S5RXF7     |   6.602485e+06 |           3.9240000 |    3.980000 |      3.820000 |      4.110000 |     3.800000 | 1.65852e+12 |        21796 |
| HCAR   | Healthcare Services Acquisition Corporation Class A Common Stock                                                                                                                | stocks | CS   | BBG00XTHMVV1   | BBG00XTHMVW0     |   8.095110e+05 |           9.8700000 |    9.870000 |      9.870000 |      9.880000 |     9.870000 | 1.65852e+12 |           24 |
| MGTA   | Magenta Therapeutics, Inc. Common Stock                                                                                                                                         | stocks | CS   | BBG00FB2NPZ0   | BBG00FB2NQ15     |   6.731600e+04 |           1.9186000 |    2.060000 |      1.860000 |      2.060000 |     1.840000 | 1.65852e+12 |          619 |
| IHI    | iShares U.S. Medical Devices ETF                                                                                                                                                | stocks | ETF  | BBG000P9YC26   | BBG001SQZT52     |   2.251358e+06 |          52.3738000 |   53.020000 |     52.360000 |     53.290000 |    51.980000 | 1.65852e+12 |        12288 |
| NPAB   | New Providence Acquisition Corp. II Class A Common Stock                                                                                                                        | stocks | CS   | BBG00ZFCX0B0   | BBG00ZFCX155     |   2.103000e+03 |           9.9600000 |    9.960000 |      9.960000 |      9.960000 |     9.960000 | 1.65852e+12 |            5 |
| RISN   | Inspire Tactical Balanced ETF                                                                                                                                                   | stocks | ETF  | BBG00W2RDDW9   | BBG00W2RDFN4     |   1.609700e+04 |          23.9520000 |   23.950000 |     23.926400 |     24.020000 |    23.920000 | 1.65852e+12 |           37 |
| ISVL   | iShares International Developed Small Cap Value Factor ETF                                                                                                                      | stocks | ETF  | BBG00ZMZ2T04   | BBG00ZMZ2TV0     |   6.300000e+02 |          29.9414000 |   30.060000 |     29.681900 |     30.060000 |    29.681900 | 1.65852e+12 |           15 |
| FTAI   | Fortress Transportation and Infrastructure Investors LLC Class A Common Stock                                                                                                   | stocks | CS   | BBG005T0KPY1   | BBG005T0KQ06     |   5.000620e+05 |          20.7821000 |   21.000000 |     20.760000 |     21.340000 |    20.220000 | 1.65852e+12 |         5926 |
| VALQ   | American Century STOXX U.S. Quality Value ETF                                                                                                                                   | stocks | ETF  | BBG00JQ0D7T8   | BBG00JQ0D8W2     |   3.037200e+04 |          47.1265000 |   47.520000 |     47.121600 |     47.560000 |    46.920000 | 1.65852e+12 |           67 |
| TTM    | Tata Motors Limited                                                                                                                                                             | stocks | ADRC | BBG000PVGDH9   | BBG001S5W4B9     |   3.641430e+05 |          28.2488000 |   28.560000 |     28.190000 |     28.585000 |    28.100000 | 1.65852e+12 |         3414 |
| SCVL   | Shoe Carnival Inc                                                                                                                                                               | stocks | CS   | BBG000BF4DG3   | BBG001S6V927     |   2.550090e+05 |          23.1028000 |   23.070000 |     23.090000 |     23.905000 |    22.760000 | 1.65852e+12 |         5700 |
| HARP   | Harpoon Therapeutics, Inc. Common Stock                                                                                                                                         | stocks | CS   | BBG00GTGL021   | BBG00GTGL030     |   1.307080e+05 |           2.5362000 |    2.560000 |      2.400000 |      2.730000 |     2.370000 | 1.65852e+12 |          980 |
| NDP    | TORTOISE ENERGY INDEPENDENCE FUND, INC.                                                                                                                                         | stocks | FUND | BBG002W90ZY5   | BBG002W910P1     |   3.223000e+03 |          27.3235000 |   27.830000 |     27.170000 |     27.830000 |    27.080000 | 1.65852e+12 |           32 |
| PLX    | Protalix BioTherapeutics, Inc. Common Stock                                                                                                                                     | stocks | CS   | BBG000JW08N5   | BBG001SBVMW4     |   1.175420e+05 |           1.0684000 |    1.070000 |      1.060000 |      1.080000 |     1.050000 | 1.65852e+12 |          296 |
| INQQ   | India Internet & Ecommerce ETF                                                                                                                                                  | stocks | ETF  | BBG016CBT4T6   | BBG016CBT5P7     |   1.451000e+03 |          12.6153000 |   12.630000 |     12.541800 |     12.660000 |    12.442000 | 1.65852e+12 |           14 |
| UDMY   | Udemy, Inc. Common Stock                                                                                                                                                        | stocks | CS   | BBG0025DTRN5   | BBG0025DTRP3     |   2.842430e+05 |          12.2303000 |   12.580000 |     12.150000 |     12.610000 |    12.010000 | 1.65852e+12 |         5159 |
| SCHV   | Schwab U.S. Large-Cap Value ETF                                                                                                                                                 | stocks | ETF  | BBG000Q0D5X8   | BBG001T66WQ7     |   3.607330e+05 |          64.2818000 |   64.580000 |     64.290000 |     64.798800 |    63.897900 | 1.65852e+12 |         2412 |
| NARI   | Inari Medical, Inc. Common Stock                                                                                                                                                | stocks | CS   | BBG009J8K7M0   | BBG009J8K7N9     |   7.563960e+05 |          75.8135000 |   79.400000 |     74.160000 |     79.580000 |    73.570000 | 1.65852e+12 |        15468 |
| TYG    | Tortoise Energy Infrastructure Corp.                                                                                                                                            | stocks | FUND | BBG000L1GR60   | BBG001SJTT37     |   5.692300e+04 |          30.2919000 |   30.470000 |     30.180000 |     30.860000 |    30.020000 | 1.65852e+12 |          502 |
| DSKE   | Daseke, Inc. Common Stock                                                                                                                                                       | stocks | CS   | BBG009NWXKB1   | BBG009NWXKC0     |   2.477440e+05 |           6.8189000 |    6.900000 |      6.830000 |      7.100000 |     6.690000 | 1.65852e+12 |         3378 |
| NTST   | NetSTREIT Corp.                                                                                                                                                                 | stocks | CS   | BBG00W5FQPV2   | BBG00W5FQQM0     |   4.073110e+05 |          20.5358000 |   20.600000 |     20.560000 |     20.640000 |    20.380000 | 1.65852e+12 |         4576 |
| ATHM   | Autohome Inc. American Depositary Shares, each representing four Class A Ordinary Shares                                                                                        | stocks | ADRC | BBG005JYTDQ5   | BBG005JYTDP6     |   3.602150e+05 |          36.0766000 |   37.270000 |     35.380000 |     37.270000 |    35.150000 | 1.65852e+12 |         5159 |
| PMT    | PennyMac Mortgage Investment Trust                                                                                                                                              | stocks | CS   | BBG000DKDWS5   | BBG001T1MJR8     |   9.043320e+05 |          14.4829000 |   14.660000 |     14.490000 |     14.780000 |    14.280000 | 1.65852e+12 |         8505 |
| CGCP   | Capital Group Core Plus Income ETF                                                                                                                                              | stocks | ETF  | BBG014YZRQW3   | BBG014YZRRT5     |   4.438970e+05 |          23.6459000 |   23.670000 |     23.655000 |     23.710000 |    23.623100 | 1.65852e+12 |         1235 |
| ITW    | Illinois Tool Works Inc.                                                                                                                                                        | stocks | CS   | BBG000BMBL90   | BBG001S5SDX0     |   7.437500e+05 |         191.6287000 |  191.040000 |    191.560000 |    192.770000 |   190.560000 | 1.65852e+12 |        21342 |
| MDH    | MDH Acquisition Corp.                                                                                                                                                           | stocks | CS   | BBG00YZ58ST4   | BBG00YZ58SW0     |   4.138200e+04 |           9.8435000 |    9.830000 |      9.840000 |      9.850000 |     9.830000 | 1.65852e+12 |           69 |
| CTO    | CTO Realty Growth, Inc.                                                                                                                                                         | stocks | CS   | BBG00Y3M1H59   | BBG00Y3M1H68     |   6.750700e+04 |          21.5073000 |   21.590000 |     21.330000 |     21.850000 |    21.180000 | 1.65852e+12 |         1467 |
| NKSH   | National Bankshares Inc/VA                                                                                                                                                      | stocks | CS   | BBG000BJ9FN7   | BBG001S7T312     |   7.914000e+03 |          31.8510000 |   31.550000 |     31.790000 |     33.329900 |    31.370100 | 1.65852e+12 |          184 |
| DON    | WisdomTree U.S. MidCap Dividend Fund                                                                                                                                            | stocks | ETF  | BBG000BSWGB2   | BBG001SHKG74     |   1.213820e+05 |          40.4854000 |   40.730000 |     40.480000 |     40.830000 |    40.210000 | 1.65852e+12 |          668 |
| BKHY   | BNY Mellon High Yield Beta ETF                                                                                                                                                  | stocks | ETF  | BBG00RYR5TG7   | BBG00RYR5V63     |   5.097800e+04 |          48.2311000 |   48.300000 |     48.110000 |     48.470000 |    47.970000 | 1.65852e+12 |          508 |
| DEEP   | Roundhill Acquirers Deep Value ETF                                                                                                                                              | stocks | ETF  | BBG0074VLYQ5   | BBG0074VLYR4     |   6.470000e+02 |          31.6259000 |   31.680000 |     31.511900 |     31.680000 |    31.320000 | 1.65852e+12 |           29 |
| NIQ    | NUVEEN INTERMEDIATE DURATION QUALITY MUNICIPAL TERM FUND                                                                                                                        | stocks | FUND | BBG003WMJZR2   | BBG003WMJZS1     |   4.318200e+04 |          12.9238000 |   12.940000 |     12.910000 |     12.955000 |    12.880000 | 1.65852e+12 |          130 |
| UBT    | ProShares Ultra 20+ Year Treasury                                                                                                                                               | stocks | ETF  | BBG000Q6X402   | BBG001T6MKQ5     |   4.165300e+04 |          34.6134000 |   34.630000 |     34.760000 |     35.106400 |    34.410000 | 1.65852e+12 |          255 |
| CTRN   | Citi Trends, Inc.                                                                                                                                                               | stocks | CS   | BBG000BRLWY6   | BBG001SH4JL0     |   1.399230e+05 |          25.1594000 |   25.800000 |     25.020000 |     26.230000 |    24.330000 | 1.65852e+12 |         2624 |
| DNN    | Denison Mines Corp                                                                                                                                                              | stocks | CS   | BBG000CX6DQ0   | BBG001S9ZPX7     |   3.346868e+06 |           1.0869000 |    1.120000 |      1.050000 |      1.150000 |     1.050000 | 1.65852e+12 |         4973 |
| TUGN   | STF Tactical Growth & Income ETF                                                                                                                                                | stocks | ETF  | BBG017J189N7   | BBG017J18BH9     |   1.503200e+04 |          22.4874000 |   22.580000 |     22.444100 |     22.580000 |    22.420000 | 1.65852e+12 |           96 |
| RTL    | The Necessity Retail REIT, Inc. Class A Common Stock                                                                                                                            | stocks | CS   | BBG004Z1PW11   | BBG004Z1PW39     |   4.607040e+05 |           7.6291000 |    7.660000 |      7.640000 |      7.690000 |     7.570000 | 1.65852e+12 |         4132 |
| APH    | Amphenol Corporation                                                                                                                                                            | stocks | CS   | BBG000B9YJ35   | BBG001S5NSK6     |   1.924062e+06 |          69.7036000 |   70.000000 |     69.820000 |     70.500000 |    69.270000 | 1.65852e+12 |        24889 |
| JUN    | Juniper II Corp.                                                                                                                                                                | stocks | CS   | BBG00ZXZ1JV1   | BBG00ZXZ1KQ4     |   1.071400e+04 |           9.9897000 |    9.985000 |      9.990000 |      9.990000 |     9.985000 | 1.65852e+12 |            7 |
| CWBR   | CohBar, Inc. Common Stock                                                                                                                                                       | stocks | CS   | BBG0080L6PG6   | BBG007HW6MT9     |   5.876733e+03 |           5.4978000 |    5.700000 |      5.547000 |      5.700000 |     5.430000 | 1.65852e+12 |          227 |
| SDIG   | Stronghold Digital Mining, Inc. Class A Common Stock                                                                                                                            | stocks | CS   | BBG011YPF0K4   | BBG011YPF0R7     |   2.761813e+07 |           3.4683000 |    3.740000 |      2.980000 |      3.770000 |     2.900000 | 1.65852e+12 |       118065 |
| LLAP   | Terran Orbital Corporation                                                                                                                                                      | stocks | CS   | BBG00ZDR71C5   | BBG00ZDR71D4     |   2.283690e+05 |           4.4848000 |    4.630000 |      4.460000 |      4.670000 |     4.410000 | 1.65852e+12 |         2894 |
| BOC    | Boston Omaha Corporation                                                                                                                                                        | stocks | CS   | BBG0021J73K6   | BBG0021J74C3     |   4.167900e+04 |          23.3909000 |   23.810000 |     23.400000 |     23.900000 |    23.100000 | 1.65852e+12 |         1057 |
| QED    | IQ Hedge Event-Driven Tracker ETF                                                                                                                                               | stocks | ETF  | BBG008B8R583   | BBG008B8R592     |   1.000000e+01 |          21.1248000 |   21.075000 |     21.075000 |     21.075000 |    21.075000 | 1.65852e+12 |            6 |
| INZY   | Inozyme Pharma, Inc. Common Stock                                                                                                                                               | stocks | CS   | BBG00J7QJMC1   | BBG00J7QJMD0     |   1.162190e+05 |           3.7701000 |    4.050000 |      3.720000 |      4.050000 |     3.650000 | 1.65852e+12 |         1274 |
| EQT    | EQT CORP                                                                                                                                                                        | stocks | CS   | BBG000BHZ5J9   | BBG001S5QXJ4     |   6.981344e+06 |          42.7967000 |   43.320000 |     42.230000 |     43.940000 |    42.160000 | 1.65852e+12 |        55861 |
| VRT    | Vertiv Holdings Co Class A Common Stock                                                                                                                                         | stocks | CS   | BBG00L2B8KW8   | BBG00L2B8LM7     |   3.593563e+06 |          10.5138000 |   10.440000 |     10.610000 |     10.790000 |    10.345000 | 1.65852e+12 |        23291 |
| REW    | Proshares UltraShort Technology                                                                                                                                                 | stocks | ETF  | BBG000QXJ052   | BBG001SSTD14     |   1.086400e+04 |          19.2149000 |   18.450000 |     19.304000 |     19.420000 |    18.450000 | 1.65852e+12 |          195 |
| TA     | TravelCenters of America LLC                                                                                                                                                    | stocks | CS   | BBG000F71CC6   | BBG001SHR063     |   6.389400e+04 |          40.2222000 |   40.730000 |     40.280000 |     41.970000 |    39.375000 | 1.65852e+12 |         1568 |
| AAOI   | Applied Optoelectronics, Inc.                                                                                                                                                   | stocks | CS   | BBG000D6VW15   | BBG001SG47G4     |   1.878590e+05 |           1.7012000 |    1.820000 |      1.680000 |      1.820000 |     1.640000 | 1.65852e+12 |         1100 |
| VTYX   | Ventyx Biosciences, Inc. Common Stock                                                                                                                                           | stocks | CS   | BBG00VC84K29   | BBG00VC84K47     |   1.018300e+05 |          15.4594000 |   16.000000 |     14.920000 |     16.280000 |    14.920000 | 1.65852e+12 |         2202 |
| WEA    | Western Asset Premier Bond Fund                                                                                                                                                 | stocks | FUND | BBG000BPYDK3   | BBG001SB94R5     |   4.345300e+04 |          10.8168000 |   10.710000 |     10.830000 |     10.840000 |    10.710000 | 1.65852e+12 |          283 |
| HIMX   | Himax Technologies, Inc.                                                                                                                                                        | stocks | ADRC | BBG000R01RJ8   | BBG001SPKXV2     |   2.061514e+06 |           7.1028000 |    7.350000 |      7.060000 |      7.490000 |     6.990000 | 1.65852e+12 |        13668 |
| HCKT   | Hackett Group Inc (The).                                                                                                                                                        | stocks | CS   | BBG000BBLQV7   | BBG001S5VLP7     |   1.096010e+05 |          20.1332000 |   20.370000 |     20.160000 |     20.440000 |    19.950000 | 1.65852e+12 |         2228 |
| FTXN   | First Trust Nasdaq Oil & Gas ETF                                                                                                                                                | stocks | ETF  | BBG00DVWCBN3   | BBG00DVWCBP1     |   9.232700e+04 |          23.7334000 |   23.820000 |     23.490000 |     24.180000 |    23.350000 | 1.65852e+12 |          407 |
| ALRN   | Aileron Therapeutics, Inc. Common Stock                                                                                                                                         | stocks | CS   | BBG002454KR1   | BBG002454KS0     |   2.904470e+05 |           0.2125600 |    0.230100 |      0.205600 |      0.230100 |     0.205600 | 1.65852e+12 |          575 |
| GUNR   | FlexShares Global Upstream Natural Resources Index Fund                                                                                                                         | stocks | ETF  | BBG00243P818   | BBG00243P8S9     |   1.753732e+06 |          38.6184000 |   38.640000 |     38.310000 |     38.970000 |    38.180000 | 1.65852e+12 |         8386 |
| DO     | Diamond Offshore Drilling, Inc.                                                                                                                                                 | stocks | CS   | BBG010JCFCD4   | BBG010JCFCF2     |   1.245980e+06 |           5.4902000 |    5.580000 |      5.410000 |      5.710000 |     5.395000 | 1.65852e+12 |         9432 |
| ZUMZ   | Zumiez Inc.                                                                                                                                                                     | stocks | CS   | BBG000PYX812   | BBG001SGPKJ9     |   2.459270e+05 |          28.0079000 |   28.010000 |     27.970000 |     28.925000 |    27.660000 | 1.65852e+12 |         5679 |
| SPHD   | Invesco S&P 500 High Dividend Low Volatility ETF                                                                                                                                | stocks | ETF  | BBG003H4R9V3   | BBG003H4RBL9     |   1.232256e+06 |          43.7585000 |   43.810000 |     43.780000 |     43.960000 |    43.549900 | 1.65852e+12 |         9135 |
| IPWR   | Ideal Power Inc.                                                                                                                                                                | stocks | CS   | BBG004YSSTM4   | BBG004YSSTN3     |   5.093600e+04 |          10.4245000 |   11.070000 |     10.030000 |     11.264500 |     9.900000 | 1.65852e+12 |          698 |
| ACWV   | iShares MSCI Global Min Vol Factor ETF                                                                                                                                          | stocks | ETF  | BBG0025X38X0   | BBG0025X39Q6     |   1.815800e+05 |          95.5965000 |   95.800000 |     95.610000 |     96.180000 |    95.180000 | 1.65852e+12 |         1500 |
| ANZU   | Anzu Special Acquisition Corp I Class A Common Stock                                                                                                                            | stocks | CS   | BBG00Z6RQKY6   | BBG00Z6RQKZ5     |   1.020000e+02 |           9.8250000 |    9.825000 |      9.825000 |      9.825000 |     9.825000 | 1.65852e+12 |            3 |
| PEAK   | Healthpeak Properties, Inc.                                                                                                                                                     | stocks | CS   | BBG000BKYDP9   | BBG001S5RTS2     |   3.526871e+06 |          26.8264000 |   26.910000 |     26.800000 |     27.125000 |    26.630000 | 1.65852e+12 |        26727 |
| BUI    | BlackRock Utilities, Infrastructure & Power Opportunities Trust                                                                                                                 | stocks | FUND | BBG0021RGFM9   | BBG0021RGFN8     |   3.191000e+04 |          21.5938000 |   21.500000 |     21.530000 |     21.750000 |    21.500000 | 1.65852e+12 |          208 |
| GFI    | Gold Fields Ltd ADR                                                                                                                                                             | stocks | ADRC | BBG000KHT4K7   | BBG001S93ZM2     |   6.660250e+06 |           9.0274000 |    9.120000 |      8.900000 |      9.300000 |     8.845000 | 1.65852e+12 |        33197 |
| LYFE   | 2ndVote Life Neutral Plus ETF                                                                                                                                                   | stocks | ETF  | BBG00Y6T3KL4   | BBG00Y6T3LF9     |   4.520000e+02 |          28.9117000 |   29.319900 |     28.812500 |     29.319900 |    28.775000 | 1.65852e+12 |            8 |
| MUA    | Blackrock Muni Assets Fund, Inc.                                                                                                                                                | stocks | FUND | BBG000BHYBF1   | BBG001S7DTL0     |   4.345800e+04 |          12.1311000 |   12.170000 |     12.100000 |     12.260000 |    12.040000 | 1.65852e+12 |          162 |
| QQMG   | Invesco ESG NASDAQ 100 ETF                                                                                                                                                      | stocks | ETF  | BBG0136JTF54   | BBG0136JTG07     |   2.240000e+03 |          20.3423000 |   20.510000 |     20.200000 |     20.510000 |    20.050000 | 1.65852e+12 |           45 |
| KRBN   | KraneShares Global Carbon Strategy ETF                                                                                                                                          | stocks | ETF  | BBG00WC708F6   | BBG00WC70964     |   4.353960e+05 |          42.8812000 |   43.800000 |     42.710000 |     43.830000 |    42.556500 | 1.65852e+12 |         2686 |
| POWW   | AMMO, Inc. Common Stock                                                                                                                                                         | stocks | CS   | BBG000BYZD23   | BBG001SB6XN8     |   1.153044e+06 |           4.8472000 |    4.880000 |      4.800000 |      4.930000 |     4.790000 | 1.65852e+12 |         5471 |
| BDSX   | Biodesix, Inc. Common Stock                                                                                                                                                     | stocks | CS   | BBG001J2LMV6   | BBG001V0G914     |   7.953800e+04 |           1.9834000 |    2.040000 |      1.980000 |      2.040000 |     1.940000 | 1.65852e+12 |          543 |
| FUTY   | Fidelity MSCI Utilities Index ETF                                                                                                                                               | stocks | ETF  | BBG005FHVX98   | BBG005FHVXB5     |   2.904100e+05 |          44.7647000 |   44.510000 |     44.800000 |     44.960000 |    44.452100 | 1.65852e+12 |         1568 |
| IOVA   | Iovance Biotherapeutics, Inc. Common Stock                                                                                                                                      | stocks | CS   | BBG000FTLBV7   | BBG001T2PM32     |   2.439662e+06 |          11.9616000 |   12.640000 |     11.780000 |     12.850000 |    11.735000 | 1.65852e+12 |        21957 |
| QVMS   | Invesco S&P SmallCap 600 QVM Multi-factor ETF                                                                                                                                   | stocks | ETF  | BBG011FSB4J7   | BBG011FSB5D0     |   3.130000e+02 |          22.0785000 |   22.070000 |     22.146100 |     22.146100 |    22.070000 | 1.65852e+12 |            5 |
| UNF    | Unifirst Corp                                                                                                                                                                   | stocks | CS   | BBG000BW29L1   | BBG001S5X2B2     |   6.715300e+04 |         188.7371000 |  188.980000 |    189.450000 |    189.780000 |   187.331500 | 1.65852e+12 |         2289 |
| PHX    | PHX Minerals Inc.                                                                                                                                                               | stocks | CS   | BBG000C38CT3   | BBG001S6G023     |   1.462030e+05 |           2.9822000 |    3.000000 |      2.950000 |      3.077400 |     2.920000 | 1.65852e+12 |          912 |
| SLY    | SPDR S&P 600 Small Cap ETF (based on S&P SmallCap 600 Index–symbol SML)                                                                                                         | stocks | ETF  | BBG000KMRJ98   | BBG001SPTB69     |   1.063380e+05 |          84.1208000 |   85.220000 |     84.220000 |     85.360200 |    83.509000 | 1.65852e+12 |          635 |
| SQL    | SeqLL Inc. Common stock                                                                                                                                                         | stocks | CS   | BBG0074P1FJ9   | BBG0074P1FK7     |   1.066700e+04 |           0.9622000 |    0.963800 |      0.965200 |      0.980000 |     0.930000 | 1.65852e+12 |           91 |
| MFLX   | First Trust Exchange-Traded Fund VIII First Trust Flexible Municipal High Income ETF                                                                                            | stocks | ETF  | BBG00DX33JB1   | BBG00DX33JC0     |   5.468000e+03 |          16.9922000 |   17.005000 |     17.120000 |     17.120000 |    16.900000 | 1.65852e+12 |           48 |
| SPOK   | Spok Holdings, Inc                                                                                                                                                              | stocks | CS   | BBG000N4KB80   | BBG001SF3J26     |   3.394400e+04 |           6.4452000 |    6.510000 |      6.500000 |      6.530000 |     6.370000 | 1.65852e+12 |          444 |
| ISR    | IsoRay, Inc.                                                                                                                                                                    | stocks | CS   | BBG000BG5HT7   | BBG001S5Q300     |   1.726030e+05 |           0.3206100 |    0.314200 |      0.333000 |      0.335400 |     0.313300 | 1.65852e+12 |          645 |
| SCHW   | The Charles Schwab Corporation                                                                                                                                                  | stocks | CS   | BBG000BSLZY7   | BBG001S5VXD4     |   6.794119e+06 |          63.1404000 |   63.370000 |     62.990000 |     63.885000 |    62.620000 | 1.65852e+12 |        61325 |
| AMRS   | Amyris Inc.                                                                                                                                                                     | stocks | CS   | BBG000DW5XK4   | BBG001T20G05     |   4.401830e+06 |           2.0025000 |    2.270000 |      1.910000 |      2.270000 |     1.900000 | 1.65852e+12 |        13955 |
| QUMU   | Qumu Corp.                                                                                                                                                                      | stocks | CS   | BBG000BG3XJ5   | BBG001S6ZSB1     |   9.182000e+03 |           0.7300200 |    0.740000 |      0.775100 |      0.785100 |     0.639000 | 1.65852e+12 |           84 |
| NIM    | Nuveen Select Maturities Municipal Fund                                                                                                                                         | stocks | FUND | BBG000CWVGL2   | BBG001S74480     |   3.800400e+04 |           9.3253000 |    9.360000 |      9.375000 |      9.380000 |     9.290000 | 1.65852e+12 |          148 |
| XJR    | iShares ESG Screened S&P Small-Cap ETF                                                                                                                                          | stocks | ETF  | BBG00XDJFM42   | BBG00XDJFMZ8     |   4.941200e+04 |          34.0140000 |   34.340000 |     34.150000 |     34.350000 |    33.850100 | 1.65852e+12 |           70 |
| PGC    | Peapack-Gladstone Financial Corp                                                                                                                                                | stocks | CS   | BBG000MMCGW2   | BBG001SG3CC8     |   3.951600e+04 |          30.2835000 |   30.550000 |     30.260000 |     30.550000 |    30.040000 | 1.65852e+12 |          925 |
| INDP   | Indaptus Therapeutics, Inc. Common Stock                                                                                                                                        | stocks | CS   | BBG00ZNLVBC8   | BBG00ZNLVBD7     |   1.413000e+04 |           3.0375000 |    3.000000 |      2.990000 |      3.070000 |     2.980000 | 1.65852e+12 |          148 |
| BOIL   | ProShares Ultra Bloomberg Natural Gas                                                                                                                                           | stocks | ETV  | BBG0024TGRQ2   | BBG0024TGSG1     |   2.230060e+06 |          85.4105000 |   83.970000 |     86.700000 |     88.300000 |    83.621600 | 1.65852e+12 |        24372 |
| DATS   | DatChat, Inc. Common Stock                                                                                                                                                      | stocks | CS   | BBG00K9VQDD6   | BBG00K9VQDF4     |   1.487320e+05 |           1.2014000 |    1.170000 |      1.160000 |      1.250000 |     1.160000 | 1.65852e+12 |          819 |
| OOMA   | Ooma, Inc. Common Stock                                                                                                                                                         | stocks | CS   | BBG001QB7V26   | BBG001V1FP08     |   6.720100e+04 |          11.4974000 |   11.650000 |     11.660000 |     11.660000 |    11.220000 | 1.65852e+12 |         1077 |
| DDWM   | WisdomTree Dynamic Currency Hedged International Equity Fund                                                                                                                    | stocks | ETF  | BBG00BSZ29K3   | BBG00BSZ29J5     |   2.210900e+04 |          28.0831000 |   28.090000 |     28.020000 |     28.270000 |    27.990000 | 1.65852e+12 |          160 |
| BUFT   | FT Cboe Vest Buffered Allocation Defensive ETF                                                                                                                                  | stocks | ETF  | BBG0133TT3L4   | BBG0133TT4G8     |   5.017800e+04 |          18.7588000 |   18.793500 |     18.770000 |     18.793500 |    18.730000 | 1.65852e+12 |          116 |
| CWI    | SPDR MSCI ACWI ex-US ETF                                                                                                                                                        | stocks | ETF  | BBG000Q8TXD5   | BBG001SRXRN6     |   2.895250e+05 |          23.9330000 |   24.000000 |     23.830000 |     24.140000 |    23.750000 | 1.65852e+12 |          956 |
| PMGMU  | Priveterra Acquisition Corp. Units                                                                                                                                              | stocks | UNIT | BBG00Z0HV651   | BBG00Z0HV704     |   5.531000e+03 |           9.9142000 |    9.920000 |      9.860000 |      9.920000 |     9.860000 | 1.65852e+12 |            3 |
| SHW    | The Sherwin-Williams Company                                                                                                                                                    | stocks | CS   | BBG000BSXQV7   | BBG001S5W2F9     |   1.496873e+06 |         259.3985000 |  258.650000 |    259.010000 |    263.235000 |   256.860000 | 1.65852e+12 |        32855 |
| CEV    | Eaton Vance California Municipal Income Trust                                                                                                                                   | stocks | FUND | BBG000BQ1B85   | BBG001SCBLC8     |   2.672200e+04 |          10.7113000 |   10.660000 |     10.750000 |     10.750000 |    10.660000 | 1.65852e+12 |          132 |
| EVBG   | Everbridge, Inc. Common Stock                                                                                                                                                   | stocks | CS   | BBG0022FMPD5   | BBG0022FMPG2     |   4.106260e+05 |          30.1196000 |   31.350000 |     29.910000 |     31.660000 |    29.680000 | 1.65852e+12 |         8711 |
| PESI   | Perma-Fix Environmental Services, Inc.                                                                                                                                          | stocks | CS   | BBG000BGRMX7   | BBG001S75KJ1     |   2.299800e+04 |           5.2191000 |    5.370000 |      5.274800 |      5.370000 |     5.200000 | 1.65852e+12 |          101 |
| DAUG   | FT Cboe Vest U.S. Equity Deep Buffer ETF - August                                                                                                                               | stocks | ETF  | BBG00QQFDYF5   | BBG00QQFDZ53     |   6.794300e+04 |          32.4994000 |   32.495000 |     32.475000 |     32.520000 |    32.470000 | 1.65852e+12 |          139 |
| OCGN   | Ocugen, Inc. Common Stock                                                                                                                                                       | stocks | CS   | BBG00194VJB1   | BBG001TG1LS2     |   5.004289e+06 |           2.4587000 |    2.650000 |      2.410000 |      2.650000 |     2.405000 | 1.65852e+12 |        12943 |
| AXTI   | AXT Inc                                                                                                                                                                         | stocks | CS   | BBG000BHZ0N5   | BBG001S7SPZ7     |   1.083760e+05 |           6.6508000 |    6.790000 |      6.670000 |      6.800000 |     6.570000 | 1.65852e+12 |         1465 |
| LTC    | LTC Properties, Inc.                                                                                                                                                            | stocks | CS   | BBG000BGCCC8   | BBG001S73DC6     |   1.411670e+05 |          39.3646000 |   39.310000 |     39.460000 |     39.520000 |    39.050000 | 1.65852e+12 |         3321 |
| LTH    | Life Time Group Holdings, Inc.                                                                                                                                                  | stocks | CS   | BBG012J3H017   | BBG012J3H0X2     |   1.499090e+05 |          14.0094000 |   14.390000 |     13.950000 |     14.600000 |    13.710000 | 1.65852e+12 |         2674 |
| CXAC   | C5 Acquisition Corporation                                                                                                                                                      | stocks | CS   | BBG0156KRYM8   | BBG0156KRYN7     |   1.875000e+03 |           9.9916000 |    9.980100 |     10.000000 |     10.000000 |     9.980100 | 1.65852e+12 |           13 |
| VOTE   | Engine No. 1 Transform 500 ETF                                                                                                                                                  | stocks | ETF  | BBG011KFPLZ8   | BBG011KFPMT3     |   1.133100e+04 |          45.9578000 |   46.390000 |     45.870000 |     46.390000 |    45.650900 | 1.65852e+12 |          209 |
| SITE   | SiteOne Landscape Supply, Inc.                                                                                                                                                  | stocks | CS   | BBG009T22D49   | BBG009T22D67     |   1.701720e+05 |         126.9240000 |  126.560000 |    126.660000 |    128.940000 |   124.720000 | 1.65852e+12 |         5383 |
| WSBC   | WesBanco Inc                                                                                                                                                                    | stocks | CS   | BBG000BX0BJ9   | BBG001S5XJR8     |   1.625410e+05 |          32.3944000 |   32.380000 |     32.490000 |     32.600000 |    32.120000 | 1.65852e+12 |         3262 |
| LII    | Lennox International Inc.                                                                                                                                                       | stocks | CS   | BBG000BB5B84   | BBG001S5SST2     |   3.432950e+05 |         227.4216000 |  226.170000 |    227.660000 |    230.805000 |   225.030000 | 1.65852e+12 |         9047 |
| FAPR   | FT Cboe Vest U.S. Equity Buffer ETF - April                                                                                                                                     | stocks | ETF  | BBG0101Q5YW4   | BBG0101Q5ZQ8     |   1.616300e+04 |          29.3124000 |   29.476400 |     29.360000 |     29.495000 |    29.230700 | 1.65852e+12 |           65 |
| ONTO   | Onto Innovation Inc.                                                                                                                                                            | stocks | CS   | BBG000BPRN29   | BBG001S5THX0     |   2.714320e+05 |          79.1834000 |   81.180000 |     79.130000 |     81.180000 |    77.930400 | 1.65852e+12 |         6209 |
| BSAC   | Banco Santander-Chile                                                                                                                                                           | stocks | ADRC | BBG000BCZQ13   | BBG001S5VVS2     |   3.744590e+05 |          15.3815000 |   15.520000 |     15.420000 |     15.520000 |    15.205000 | 1.65852e+12 |         3774 |
| LPRO   | Open Lending Corporation Class A Common Stock                                                                                                                                   | stocks | CS   | BBG00VDHLSQ6   | BBG00VDHLTH4     |   7.038420e+05 |          11.0935000 |   11.120000 |     11.050000 |     11.400000 |    10.950000 | 1.65852e+12 |         7951 |
| MIRM   | Mirum Pharmaceuticals, Inc. Common Stock                                                                                                                                        | stocks | CS   | BBG00MH7T2M7   | BBG00MH7T2N6     |   1.019330e+05 |          24.3280000 |   24.590000 |     24.390000 |     24.720000 |    23.700000 | 1.65852e+12 |         2291 |
| AXP    | American Express Company                                                                                                                                                        | stocks | CS   | BBG000BCQZS4   | BBG001S5P034     |   9.298440e+06 |         155.1413000 |  159.010000 |    153.010000 |    160.880000 |   152.620000 | 1.65852e+12 |       103841 |
| WTS    | Watts Water Technologies, Inc. Class A                                                                                                                                          | stocks | CS   | BBG000C4Z6C2   | BBG001S6N6Y7     |   9.576700e+04 |         129.6553000 |  130.730000 |    129.970000 |    130.910000 |   128.335000 | 1.65852e+12 |         3963 |
| RDFN   | Redfin Corporation Common Stock                                                                                                                                                 | stocks | CS   | BBG001Q7HP63   | BBG001V1F423     |   1.727598e+06 |           9.3661000 |    9.920000 |      9.300000 |     10.000000 |     9.175000 | 1.65852e+12 |        15503 |
| XELB   | XCEL BRANDS INC.                                                                                                                                                                | stocks | CS   | BBG000BB0JR1   | BBG001S5PTK2     |   1.169170e+05 |           1.1182000 |    1.160000 |      1.100000 |      1.200000 |     1.060000 | 1.65852e+12 |          243 |
| DWLD   | Davis Select Worldwide ETF                                                                                                                                                      | stocks | ETF  | BBG00FNFQTK3   | BBG00FNFQV91     |   2.609400e+04 |          24.5966000 |   24.890000 |     24.448400 |     24.890000 |    24.330000 | 1.65852e+12 |           32 |
| BNDW   | Vanguard Total World Bond ETF                                                                                                                                                   | stocks | ETF  | BBG00LWSF7T3   | BBG00LWSF8K0     |   3.015200e+04 |          71.2679000 |   71.200000 |     71.250000 |     71.450000 |    71.200000 | 1.65852e+12 |          401 |
| CYA    | Simplify Tail Risk Strategy ETF                                                                                                                                                 | stocks | ETF  | BBG012J36HJ3   | BBG012J36JD5     |   2.846400e+04 |          17.9853000 |   17.690000 |     17.950000 |     18.100000 |    17.690000 | 1.65852e+12 |          144 |
| RYAAY  | Ryanair Holdings plc American Depositary Shares                                                                                                                                 | stocks | ADRC | BBG000J9CBT0   | BBG001SB07J6     |   1.190102e+06 |          72.1686000 |   73.390000 |     70.970000 |     73.570000 |    70.910000 | 1.65852e+12 |        12989 |
| SCHQ   | Schwab Long-Term U.S. Treasury ETF                                                                                                                                              | stocks | ETF  | BBG00PZFJPC3   | BBG00PZFJQM0     |   1.480600e+04 |          41.3634000 |   41.320000 |     41.420000 |     41.568000 |    41.250000 | 1.65852e+12 |          119 |
| WORX   | SC WORX Corp                                                                                                                                                                    | stocks | CS   | BBG00DLMHF89   | BBG00DLMHF70     |   1.646640e+05 |           0.7447800 |    0.800000 |      0.730100 |      0.800000 |     0.716400 | 1.65852e+12 |          397 |
| GGT    | THE GABELLI MULTIMEDIA TRUST INC.                                                                                                                                               | stocks | FUND | BBG000BZNBL6   | BBG001S7K7J3     |   2.701400e+04 |           7.1322000 |    7.170000 |      7.060000 |      7.210000 |     7.055100 | 1.65852e+12 |          107 |
| GSP    | Barclays Bank PLC iPath Exchange Traded Notes due 2036 Linked to GSCI Total Return Index                                                                                        | stocks | ETN  | BBG000JL5K57   | BBG001SPJ643     |   3.998000e+03 |          22.6031000 |   22.660000 |     22.350000 |     22.870000 |    22.340000 | 1.65852e+12 |           24 |
| ABNB   | Airbnb, Inc. Class A Common Stock                                                                                                                                               | stocks | CS   | BBG001Y2XS07   | BBG001Y2XS16     |   4.243932e+06 |         105.1710000 |  108.310000 |    103.970000 |    110.100000 |   102.930000 | 1.65852e+12 |        69562 |
| FMAO   | Farmers & Merchants Bancorp, Inc.                                                                                                                                               | stocks | CS   | BBG000BLPL88   | BBG001S8HL40     |   6.320100e+04 |          30.9051000 |   31.460000 |     30.670000 |     31.490000 |    30.320000 | 1.65852e+12 |         1756 |
| CNBS   | Amplify Seymour Cannabis ETF                                                                                                                                                    | stocks | ETF  | BBG00PSSPB28   | BBG00PSSPBT9     |   4.143400e+04 |           7.4576000 |    7.670000 |      7.430000 |      7.720000 |     7.300100 | 1.65852e+12 |          408 |
| WCLD   | WisdomTree Cloud Computing Fund                                                                                                                                                 | stocks | ETF  | BBG00Q5FMYM0   | BBG00Q5FMZC8     |   3.129430e+05 |          30.2516000 |   31.190000 |     29.790000 |     31.519900 |    29.550000 | 1.65852e+12 |         2405 |
| PGX    | Invesco Preferred ETF                                                                                                                                                           | stocks | ETF  | BBG000TWWFV4   | BBG001T0NSB6     |   3.000736e+06 |          12.8009000 |   12.800000 |     12.810000 |     12.839700 |    12.775000 | 1.65852e+12 |         6083 |
| IYH    | iShares U.S. Healthcare ETF                                                                                                                                                     | stocks | ETF  | BBG000BXX310   | BBG001SFGXR4     |   9.531700e+04 |         273.0074000 |  275.260000 |    272.830000 |    275.260000 |   271.426500 | 1.65852e+12 |         1354 |
| SHYD   | VanEck Short High Yield Muni ETF                                                                                                                                                | stocks | ETF  | BBG005T08329   | BBG005T08310     |   3.569840e+05 |          22.9945000 |   23.010000 |     22.980000 |     23.040000 |    22.903800 | 1.65852e+12 |         1086 |
| AMCX   | AMC Networks Inc. Class A                                                                                                                                                       | stocks | CS   | BBG000H01H92   | BBG001SLR8S3     |   1.844120e+05 |          32.3333000 |   32.540000 |     32.310000 |     33.013800 |    31.970000 | 1.65852e+12 |         4793 |
| CCXI   | ChemoCentryx, Inc.                                                                                                                                                              | stocks | CS   | BBG000PTSB12   | BBG001SRF9K9     |   1.168241e+06 |          22.7355000 |   23.610000 |     22.170000 |     23.840000 |    22.140000 | 1.65852e+12 |        10267 |
| ISRG   | Intuitive Surgical Inc.                                                                                                                                                         | stocks | CS   | BBG000BJPDZ1   | BBG001S7XR78     |   6.588323e+06 |         211.3653000 |  207.440000 |    211.850000 |    217.490000 |   203.310000 | 1.65852e+12 |       107605 |
| IPG    | The Interpublic Group of Companies, Inc.                                                                                                                                        | stocks | CS   | BBG000C90DH9   | BBG001S6RLK5     |   3.668563e+06 |          29.3063000 |   29.550000 |     29.490000 |     29.770000 |    29.040000 | 1.65852e+12 |        33618 |
| QYLG   | Global X Nasdaq 100 Covered Call & Growth ETF                                                                                                                                   | stocks | ETF  | BBG00XH4S633   | BBG00XH4S6Z8     |   1.389000e+04 |          25.2549000 |   25.450000 |     25.210000 |     25.450000 |    25.100000 | 1.65852e+12 |          359 |
| SPWH   | Sportsman’s Warehouse Holdings, Inc.                                                                                                                                            | stocks | CS   | BBG002G8Q1H1   | BBG002G8Q1J9     |   4.245380e+05 |          10.0802000 |   10.280000 |     10.100000 |     10.280000 |     9.950000 | 1.65852e+12 |         4909 |
| GEO    | The GEO Group, Inc.                                                                                                                                                             | stocks | CS   | BBG000GC0TZ3   | BBG001S6VYZ6     |   8.846310e+05 |           6.6038000 |    6.740000 |      6.590000 |      6.810000 |     6.550000 | 1.65852e+12 |         5779 |
| TBCPU  | Thunder Bridge Capital Partners III Inc. Units                                                                                                                                  | stocks | UNIT | BBG00YY9NB71   | BBG00YY9NC24     |   2.586000e+03 |           9.8974000 |    9.890000 |      9.890000 |      9.900000 |     9.890000 | 1.65852e+12 |           16 |
| DMS    | Digital Media Solutions, Inc.                                                                                                                                                   | stocks | CS   | BBG00KHHFX00   | BBG00KHHFXQ2     |   4.698600e+04 |           1.4170000 |    1.340000 |      1.380000 |      1.485100 |     1.290000 | 1.65852e+12 |          413 |
| CATO   | CATO CORP                                                                                                                                                                       | stocks | CS   | BBG000JX7SX1   | BBG001SD32N5     |   9.553200e+04 |          12.7493000 |   12.710000 |     12.870000 |     12.980000 |    12.529100 | 1.65852e+12 |         1711 |
| SKOR   | FlexShares Credit-Scored US Corporate Bond Index Fund                                                                                                                           | stocks | ETF  | BBG007J2RFB4   | BBG007J2RFC3     |   1.276500e+04 |          48.1022000 |   48.080000 |     48.107500 |     48.290000 |    48.010000 | 1.65852e+12 |           97 |
| SHEN   | Shenandoah Telecom Co                                                                                                                                                           | stocks | CS   | BBG000BTC2N0   | BBG001SG37M8     |   2.063300e+05 |          20.0976000 |   20.250000 |     20.280000 |     20.440000 |    19.790000 | 1.65852e+12 |         4773 |
| DEN    | Denbury Inc.                                                                                                                                                                    | stocks | CS   | BBG00XJDVWJ3   | BBG00XJDVXC8     |   2.638100e+05 |          59.9056000 |   60.670000 |     59.190000 |     61.500000 |    59.065000 | 1.65852e+12 |         6882 |
| NFJ    | Virtus Dividend, Interest & Premium Strategy Fund                                                                                                                               | stocks | FUND | BBG000QSY038   | BBG001SNN4Q4     |   5.154970e+05 |          12.5545000 |   12.620000 |     12.510000 |     12.700000 |    12.430000 | 1.65852e+12 |         2551 |
| CCAIU  | Cascadia Acquisition Corp. Unit                                                                                                                                                 | stocks | UNIT | BBG01227QNF1   | BBG01227QP84     |   2.000000e+02 |           9.9100000 |    9.910000 |      9.910000 |      9.910000 |     9.910000 | 1.65852e+12 |            2 |
| VOR    | Vor Biopharma Inc. Common Stock                                                                                                                                                 | stocks | CS   | BBG00NBJRDT6   | BBG00NBJRDV3     |   3.470300e+04 |           4.8485000 |    5.250000 |      4.760000 |      5.250000 |     4.735000 | 1.65852e+12 |          782 |
| CCV    | Churchill Capital Corp V                                                                                                                                                        | stocks | CS   | BBG00YRX3C21   | BBG00YRX3C30     |   3.637000e+03 |           9.8548000 |    9.850000 |      9.850000 |      9.860000 |     9.850000 | 1.65852e+12 |           22 |
| PM     | Philip Morris International Inc.                                                                                                                                                | stocks | CS   | BBG000J2XL74   | BBG001STP9N1     |   5.307586e+06 |          95.5940000 |   94.080000 |     95.930000 |     96.100000 |    94.080000 | 1.65852e+12 |        49768 |
| WIRE   | Encore Wire Corp                                                                                                                                                                | stocks | CS   | BBG000CQCCK6   | BBG001S70TC4     |   2.881280e+05 |         110.2530000 |  113.590000 |    109.330000 |    113.750000 |   108.320000 | 1.65852e+12 |         8953 |
| EW     | Edwards Lifesciences Corp                                                                                                                                                       | stocks | CS   | BBG000BRXP69   | BBG001SF2288     |   2.535041e+06 |         102.8534000 |  103.200000 |    102.580000 |    104.850000 |   101.720000 | 1.65852e+12 |        35804 |
| ODDS   | Pacer BlueStar Digital Entertainment ETF                                                                                                                                        | stocks | ETF  | BBG016PG2349   | BBG016PG2438     |   1.000000e+00 |          17.3600000 |   17.175600 |     17.175600 |     17.175600 |    17.175600 | 1.65852e+12 |            1 |
| APRE   | Aprea Therapeutics, Inc. Common stock                                                                                                                                           | stocks | CS   | BBG00Q6Q3K37   | BBG00Q6Q3KV6     |   1.752590e+05 |           1.1312000 |    1.120000 |      1.130000 |      1.150000 |     1.100000 | 1.65852e+12 |          572 |
| ACIO   | Aptus Collared Income Opportunity ETF                                                                                                                                           | stocks | ETF  | BBG00PNSDFF3   | BBG00PNSDG52     |   1.184500e+04 |          29.6453000 |   29.810000 |     29.670000 |     29.810000 |    29.555000 | 1.65852e+12 |           42 |
| CWST   | Casella Waste Systems Inc                                                                                                                                                       | stocks | CS   | BBG000BT0J38   | BBG001SB5S05     |   1.929150e+05 |          75.3605000 |   75.980000 |     75.260000 |     76.075000 |    74.760000 | 1.65852e+12 |         4204 |
| HOFT   | Hooker Furnishings Corporation Common Stock                                                                                                                                     | stocks | CS   | BBG000BNSKG4   | BBG001S9JGQ3     |   3.656400e+04 |          16.5738000 |   16.780000 |     16.580000 |     16.940000 |    16.440000 | 1.65852e+12 |          858 |
| IIGD   | Invesco Investment Grade Defensive ETF                                                                                                                                          | stocks | ETF  | BBG00LG2GR86   | BBG00LG2GS02     |   8.210000e+02 |          24.5850000 |   24.570000 |     24.600400 |     24.600400 |    24.570000 | 1.65852e+12 |            9 |
| NGM    | NGM Biopharmaceuticals, Inc. Common Stock                                                                                                                                       | stocks | CS   | BBG001J1BNT0   | BBG001V0FS86     |   3.174790e+05 |          15.3683000 |   15.770000 |     15.310000 |     15.770000 |    15.120000 | 1.65852e+12 |         4777 |
| LEMB   | iShares J.P. Morgan EM Local Currency Bond ETF                                                                                                                                  | stocks | ETF  | BBG0025X4HC2   | BBG0025X4J38     |   2.535300e+04 |          33.4513000 |   33.420000 |     33.445000 |     33.550000 |    33.380000 | 1.65852e+12 |          179 |
| AVGO   | Broadcom Inc. Common Stock                                                                                                                                                      | stocks | CS   | BBG00KHY5S69   | BBG00KHY5SY8     |   1.786683e+06 |         511.9210000 |  518.810000 |    512.520000 |    519.790000 |   506.680000 | 1.65852e+12 |        45009 |
| CGGO   | Capital Group Global Growth Equity ETF                                                                                                                                          | stocks | ETF  | BBG015DN3FW0   | BBG015DN3GR4     |   2.851640e+05 |          20.9020000 |   21.130000 |     20.900000 |     21.190000 |    20.790000 | 1.65852e+12 |         1042 |
| WBIY   | WBI Power FactorTM High Dividend ETF                                                                                                                                            | stocks | ETF  | BBG00FFG9CJ8   | BBG00FFG9D88     |   2.405000e+03 |          26.7207000 |   26.825000 |     26.536000 |     26.825000 |    26.536000 | 1.65852e+12 |           28 |
| SGRP   | SPAR Group Inc                                                                                                                                                                  | stocks | CS   | BBG000KGQHC2   | BBG001SCV2B9     |   1.841000e+03 |           1.1601000 |    1.200000 |      1.200000 |      1.200000 |     1.140000 | 1.65852e+12 |           31 |
| IGACU  | IG Acquisition Corp. Unit                                                                                                                                                       | stocks | UNIT | BBG00X71SV62   | BBG00X71SW15     |   1.850000e+02 |           9.9700000 |    9.970000 |      9.970000 |      9.970000 |     9.970000 | 1.65852e+12 |            1 |
| GLDB   | Strategy Shares Gold-Hedged Bond ETF                                                                                                                                            | stocks | ETF  | BBG011387RN3   | BBG011387SH8     |   2.200000e+01 |          19.5845000 |   19.660000 |     19.660000 |     19.660000 |    19.660000 | 1.65852e+12 |            3 |
| DBJP   | Xtrackers MSCI Japan Hedged Equity ETF                                                                                                                                          | stocks | ETF  | BBG001QVXFK8   | BBG001V1QP15     |   2.465000e+03 |          49.1975000 |   49.465100 |     49.163900 |     49.465100 |    49.140000 | 1.65852e+12 |           30 |
| CBLS   | Changebridge Capital Long/Short Equity ETF                                                                                                                                      | stocks | ETF  | BBG00Y49JXG1   | BBG00Y49JYC3     |   3.000000e+00 |          20.2600000 |   20.332400 |     20.332400 |     20.332400 |    20.332400 | 1.65852e+12 |            1 |
| BFEB   | Innovator U.S. Equity Buffer ETF - February                                                                                                                                     | stocks | ETF  | BBG00RHZ11C7   | BBG00RHZ1235     |   4.729000e+03 |          29.6829000 |   29.770900 |     29.675000 |     29.770900 |    29.605000 | 1.65852e+12 |           25 |
| RUFF   | Alpha Dog ETF                                                                                                                                                                   | stocks | ETF  | BBG01315RMG4   | BBG01315RN90     |   1.006000e+03 |          18.4764000 |   18.710000 |     18.431200 |     18.710000 |    18.364000 | 1.65852e+12 |           13 |
| SAM    | Boston Beer Company                                                                                                                                                             | stocks | CS   | BBG000BCZBF1   | BBG001S5VVQ4     |   8.184120e+05 |         342.0595000 |  316.200000 |    356.990000 |    358.000000 |   316.200000 | 1.65852e+12 |        19897 |
| ANGO   | AngioDynamics, Inc.                                                                                                                                                             | stocks | CS   | BBG000BB5621   | BBG001S5RPY3     |   1.186450e+05 |          21.6373000 |   21.890000 |     21.610000 |     22.240000 |    21.380000 | 1.65852e+12 |         3047 |
| SLYV   | SPDR S&P 600 Small Cap Value ETF (based on S&P SmallCap 600 Value Index symbol–CVK)                                                                                             | stocks | ETF  | BBG000C8RKV0   | BBG001SG3JV2     |   1.848720e+05 |          74.9739000 |   75.620000 |     75.060000 |     75.960000 |    74.442800 | 1.65852e+12 |         1759 |
| TFFP   | TFF Pharmaceuticals, Inc. Common Stock                                                                                                                                          | stocks | CS   | BBG00KHKGQ98   | BBG00KHKGQB5     |   2.106750e+05 |           5.4363000 |    5.600000 |      5.510000 |      5.700000 |     5.150000 | 1.65852e+12 |         1454 |
| NDAQ   | Nasdaq, Inc. Common Stock                                                                                                                                                       | stocks | CS   | BBG000F5VVB6   | BBG001SKTBJ6     |   4.148859e+06 |          57.7790000 |   58.023300 |     57.533300 |     58.300000 |    57.160000 | 1.65852e+12 |        27602 |
| EFOI   | Energy Focus, Inc.                                                                                                                                                              | stocks | CS   | BBG000BTLBB3   | BBG001S72R36     |   5.388940e+05 |           0.8829500 |    0.860000 |      0.872000 |      0.940000 |     0.835500 | 1.65852e+12 |         1296 |
| BRT    | BRT Apartments Corp                                                                                                                                                             | stocks | CS   | BBG000BDR7W8   | BBG001S5PCX5     |   2.552900e+04 |          23.0051000 |   23.200000 |     22.890000 |     23.350000 |    22.750000 | 1.65852e+12 |          466 |
| LHCG   | LHC Group LLC                                                                                                                                                                   | stocks | CS   | BBG000G7W420   | BBG001SH4TR2     |   1.357420e+05 |         162.6415000 |  162.250000 |    163.240000 |    163.390000 |   161.510000 | 1.65852e+12 |         3556 |
| PFIS   | Peoples Financial Services Corp.                                                                                                                                                | stocks | CS   | BBG000BPBR63   | BBG001S9KST2     |   5.578000e+03 |          52.6967000 |   53.240000 |     52.510000 |     53.300000 |    52.350000 | 1.65852e+12 |          260 |
| QUS    | SPDR MSCI USA StrategicFactors ETF                                                                                                                                              | stocks | ETF  | BBG008HB6486   | BBG008HB6477     |   1.412800e+05 |         112.0004000 |  113.140000 |    111.980000 |    113.140000 |   111.410000 | 1.65852e+12 |          277 |
| IVDA   | Iveda Solutions, Inc. Common Stock                                                                                                                                              | stocks | CS   | BBG000QWBG75   | BBG001SSRFZ4     |   1.874100e+04 |           1.1450000 |    1.130000 |      1.160000 |      1.200000 |     1.110000 | 1.65852e+12 |           97 |
| AQUA   | Evoqua Water Technologies Corp. Common Stock                                                                                                                                    | stocks | CS   | BBG00DPDYDN4   | BBG00DPDYDP2     |   3.998660e+05 |          35.7954000 |   36.110000 |     35.680000 |     36.500000 |    35.465000 | 1.65852e+12 |         7413 |
| PINC   | Premier, Inc. Class A                                                                                                                                                           | stocks | CS   | BBG000G7QX50   | BBG001SJLD58     |   5.439280e+05 |          37.3336000 |   37.450000 |     37.470000 |     37.655000 |    37.010000 | 1.65852e+12 |         7906 |
| NOVT   | Novanta Inc. Common Stock                                                                                                                                                       | stocks | CS   | BBG000JB24Q5   | BBG001S6MRJ9     |   2.115055e+06 |         150.0330000 |  152.150000 |    150.080000 |    156.055000 |   148.000000 | 1.65852e+12 |        29959 |
| CVX    | Chevron Corporation                                                                                                                                                             | stocks | CS   | BBG000K4ND22   | BBG001S67ZC5     |   5.889480e+06 |         144.7755000 |  145.560000 |    144.190000 |    146.300000 |   143.420000 | 1.65852e+12 |        79386 |
| NSIT   | Insight Enterprises Inc                                                                                                                                                         | stocks | CS   | BBG000DY3K39   | BBG001S7LHJ0     |   2.523820e+05 |          90.6604000 |   91.720000 |     90.530000 |     93.080000 |    89.795000 | 1.65852e+12 |         4356 |
| PWSC   | PowerSchool Holdings, Inc.                                                                                                                                                      | stocks | CS   | BBG00ZXQB522   | BBG00ZXQB5X8     |   2.013680e+05 |          14.2796000 |   14.570000 |     14.360000 |     14.670000 |    14.030000 | 1.65852e+12 |         3709 |
| GDOT   | Green Dot Corporation                                                                                                                                                           | stocks | CS   | BBG000QDJT53   | BBG001T6W7V7     |   2.911490e+05 |          26.5927000 |   27.260000 |     26.610000 |     27.330000 |    26.275000 | 1.65852e+12 |         4284 |
| TURN   | 180 Degree Capital Corp.                                                                                                                                                        | stocks | CS   | BBG000KG4T49   | BBG001SD3FG4     |   1.733000e+03 |           6.0560000 |    6.020000 |      6.050000 |      6.070000 |     6.020000 | 1.65852e+12 |           17 |
| FLFR   | Franklin FTSE France ETF                                                                                                                                                        | stocks | ETF  | BBG00J3MMXJ9   | BBG00J3MMY70     |   5.760000e+02 |          24.8058000 |   24.860000 |     24.810100 |     24.860000 |    24.810100 | 1.65852e+12 |           12 |
| RCLFU  | Rosecliff Acquisition Corp I Unit                                                                                                                                               | stocks | UNIT | BBG00Z1X2KH3   | BBG00Z1X2LB7     |   6.480000e+02 |           9.8400000 |    9.840000 |      9.840000 |      9.840000 |     9.840000 | 1.65852e+12 |            5 |
| BGCP   | BGC Partners, Inc.                                                                                                                                                              | stocks | CS   | BBG000C4MWH4   | BBG001SDL7L6     |   9.228680e+05 |           3.6868000 |    3.740000 |      3.680000 |      3.780000 |     3.650000 | 1.65852e+12 |         5603 |
| FHLTU  | Future Health ESG Corp. Unit                                                                                                                                                    | stocks | UNIT | BBG0127JTBC6   | BBG0127JTC70     |   2.700000e+02 |          10.0907000 |   10.090000 |     10.105000 |     10.105000 |    10.090000 | 1.65852e+12 |            5 |
| BHAC   | Crixus BH3 Acquisition Company Class A Common Stock                                                                                                                             | stocks | CS   | BBG012F020W9   | BBG012F020X8     |   2.249100e+04 |           9.8902000 |    9.890000 |      9.890000 |      9.890000 |     9.889900 | 1.65852e+12 |           36 |
| TDIV   | First Trust Exchange-Traded Fund VI First Trust NASDAQ Technology Dividend Index Fund                                                                                           | stocks | ETF  | BBG00393GQX6   | BBG00393GRN5     |   9.762200e+04 |          52.2299000 |   52.770000 |     52.220000 |     52.770000 |    51.830000 | 1.65852e+12 |          702 |
| SDG    | iShares Trust iShares MSCI Global Sustainable Development Goals ETF                                                                                                             | stocks | ETF  | BBG00CRHS620   | BBG00CRHS639     |   2.571400e+04 |          81.0587000 |   81.460000 |     80.910000 |     81.640000 |    78.571400 | 1.65852e+12 |          234 |
| JOFF   | JOFF Fintech Acquisition Corp. Class A Common Stock                                                                                                                             | stocks | CS   | BBG00YZDSND3   | BBG00YZDSNF1     |   1.301400e+04 |           9.8499000 |    9.850000 |      9.840000 |      9.850000 |     9.840000 | 1.65852e+12 |           29 |
| SNTI   | Senti Biosciences, Inc. Common Stock                                                                                                                                            | stocks | CS   | BBG010WX7ZB3   | BBG010WX8056     |   5.509510e+05 |           1.8954000 |    1.850100 |      1.860000 |      1.980000 |     1.830000 | 1.65852e+12 |         2190 |
| RESP   | WisdomTree U.S. ESG Fund                                                                                                                                                        | stocks | ETF  | BBG000R1RNG0   | BBG001SSZBH4     |   2.200000e+01 |          42.4534000 |   42.566500 |     42.566500 |     42.566500 |    42.566500 | 1.65852e+12 |            3 |
| MEIP   | MEI Pharma, Inc.                                                                                                                                                                | stocks | CS   | BBG000BSGT68   | BBG001SBWG10     |   2.185554e+06 |           0.5192300 |    0.550300 |      0.508100 |      0.550300 |     0.500000 | 1.65852e+12 |         3201 |
| MBNE   | SPDR Nuveen Municipal Bond ESG ETF                                                                                                                                              | stocks | ETF  | BBG016HPRLK3   | BBG016HPRMH5     |   7.550000e+02 |          30.0893000 |   30.090000 |     30.060000 |     30.090000 |    30.060000 | 1.65852e+12 |            8 |
| RHRX   | RH Tactical Rotation ETF                                                                                                                                                        | stocks | ETF  | BBG0134N9CY4   | BBG0134N9FC1     |   5.140000e+02 |          12.0932000 |   12.090000 |     12.142800 |     12.142800 |    12.090000 | 1.65852e+12 |            8 |
| VFVA   | Vanguard U.S. Value Factor ETF                                                                                                                                                  | stocks | ETF  | BBG00K26B596   | BBG00K26B603     |   1.036800e+04 |          95.7777000 |   96.780000 |     95.690000 |     96.780000 |    95.045000 | 1.65852e+12 |          266 |
| SHEL   | Shell plc American Depositary Shares (Each represents two Ordinary shares)                                                                                                      | stocks | ADRC | BBG0147BN6G2   | BBG0147BN6H1     |   3.076278e+06 |          48.8812000 |   48.800000 |     48.790000 |     49.390000 |    48.435000 | 1.65852e+12 |        26876 |
| BSCT   | Invesco BulletShares 2029 Corporate Bond ETF                                                                                                                                    | stocks | ETF  | BBG00Q5D6Z22   | BBG00Q5D6ZV0     |   5.079500e+04 |          18.5639000 |   18.604000 |     18.577300 |     18.660000 |    18.530000 | 1.65852e+12 |          131 |
| SDC    | SmileDirectClub, Inc. Class A Common Stock                                                                                                                                      | stocks | CS   | BBG00PZZ94F4   | BBG00PZZ96M1     |   2.169787e+06 |           1.1195000 |    1.180000 |      1.100000 |      1.180000 |     1.090000 | 1.65852e+12 |         6626 |
| HOG    | Harley-Davidson, Inc.                                                                                                                                                           | stocks | CS   | BBG000BKZTP3   | BBG001S5RTY5     |   1.366145e+06 |          34.6948000 |   35.210000 |     34.610000 |     35.310000 |    34.290000 | 1.65852e+12 |        20109 |
| FMIL   | Fidelity New Millennium ETF                                                                                                                                                     | stocks | ETF  | BBG00V6QF9V7   | BBG00V6QFBM2     |   1.556000e+03 |          27.8667000 |   28.100000 |     27.714300 |     28.100000 |    27.700000 | 1.65852e+12 |           34 |
| CVT    | Cvent Holding Corp. Common Stock                                                                                                                                                | stocks | CS   | BBG00Y1TJTT5   | BBG00Y1TJTV2     |   1.280000e+05 |           5.8682000 |    5.930000 |      5.850000 |      6.010000 |     5.820000 | 1.65852e+12 |         1628 |
| LVO    | LiveOne, Inc. Common Stock                                                                                                                                                      | stocks | CS   | BBG00HNJXM64   | BBG00HNJXM73     |   2.394680e+05 |           1.2033000 |    1.180000 |      1.200000 |      1.240000 |     1.180000 | 1.65852e+12 |         1319 |
| LOCC.U | Live Oak Crestview Climate Acquisition Corp. Units, each consisting of one share of Class A common stock, and one-third of one warrant                                          | stocks | UNIT | BBG00ZK0RR73   | BBG00ZK0RS26     |   1.000000e+02 |           9.7600000 |    9.760000 |      9.760000 |      9.760000 |     9.760000 | 1.65852e+12 |            1 |
| TDY    | Teledyne Technologies Incorporated                                                                                                                                              | stocks | CS   | BBG000BMT9T6   | BBG001SDK336     |   1.396440e+05 |         401.4017000 |  404.890000 |    400.930000 |    406.388400 |   399.370000 | 1.65852e+12 |         5571 |
| ARKO   | ARKO Corp. Common Stock                                                                                                                                                         | stocks | CS   | BBG00YD8K411   | BBG00YD8K420     |   1.971050e+05 |           8.8380000 |    9.020000 |      8.760000 |      9.140000 |     8.760000 | 1.65852e+12 |         2633 |

``` r
time_df = out$time_df
time_df
```

| Company_Name                       | tckr  | Date       | Trading_volume | Volume_wt_avg_price | Open_price | Closing_price | Highest_price | Lowest_price |    Unix_time | Transactions |
|:-----------------------------------|:------|:-----------|---------------:|--------------------:|-----------:|--------------:|--------------:|-------------:|-------------:|-------------:|
| Apple Inc.                         | AAPL  | 2021-07-22 |     1581194431 |            147.2252 |   145.9350 |      148.1900 |      151.6800 |     142.5400 | 1.626926e+12 |     11079500 |
| Apple Inc.                         | AAPL  | 2021-08-22 |     1512015704 |            150.8469 |   148.3100 |      146.0600 |      157.2600 |     145.7600 | 1.629518e+12 |     11071185 |
| Apple Inc.                         | AAPL  | 2021-09-22 |     1734706549 |            143.1875 |   143.8000 |      148.7600 |      149.1700 |     138.2700 | 1.632110e+12 |     12651059 |
| Apple Inc.                         | AAPL  | 2021-10-22 |     1521123082 |            150.6026 |   148.7000 |      157.8700 |      158.6700 |     146.4128 | 1.634702e+12 |     10988977 |
| Apple Inc.                         | AAPL  | 2021-11-22 |     2483951766 |            168.5081 |   157.6500 |      171.1400 |      182.1300 |     156.3600 | 1.637298e+12 |     18157924 |
| Apple Inc.                         | AAPL  | 2021-12-22 |     1603598297 |            175.4116 |   168.2800 |      173.0700 |      182.9400 |     167.4600 | 1.639890e+12 |     13061276 |
| Apple Inc.                         | AAPL  | 2022-01-22 |     2170273792 |            168.3012 |   171.5100 |      172.5500 |      176.6500 |     154.7000 | 1.642482e+12 |     18811323 |
| Apple Inc.                         | AAPL  | 2022-02-22 |     2008854845 |            161.0747 |   171.0300 |      163.9800 |      171.9100 |     150.1000 | 1.645074e+12 |     17139013 |
| Apple Inc.                         | AAPL  | 2022-03-22 |     1598875527 |            172.1755 |   163.5100 |      165.2900 |      179.6100 |     163.0150 | 1.647662e+12 |     12836787 |
| Apple Inc.                         | AAPL  | 2022-04-22 |     2335163426 |            156.4029 |   163.9200 |      149.2400 |      171.5300 |     138.8000 | 1.650254e+12 |     20196798 |
| Apple Inc.                         | AAPL  | 2022-05-22 |     1979093614 |            141.1148 |   146.8500 |      130.0600 |      151.7400 |     129.0400 | 1.652846e+12 |     16000452 |
| Apple Inc.                         | AAPL  | 2022-06-22 |     1468318592 |            140.7204 |   130.0650 |      150.1700 |      150.8600 |     129.8100 | 1.655438e+12 |     10928624 |
| Apple Inc.                         | AAPL  | 2022-07-22 |     1372510065 |            159.7003 |   150.7400 |      172.1000 |      172.1700 |     146.7000 | 1.658030e+12 |     11304305 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2021-07-22 |      594599760 |            135.2892 |   127.8440 |      137.4295 |      138.3625 |     127.4988 | 1.626926e+12 |      1596943 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2021-08-22 |      474783020 |            142.9251 |   137.9695 |      140.8000 |      146.2538 |     137.6075 | 1.629518e+12 |      1456662 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2021-09-22 |      644962280 |            137.9608 |   138.1615 |      143.2370 |      143.6625 |     131.0500 | 1.632110e+12 |      1908559 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2021-10-22 |      741744360 |            145.0314 |   143.3380 |      149.8385 |      150.6148 |     135.4240 | 1.634702e+12 |      2305499 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2021-11-22 |      639204120 |            144.7138 |   149.9754 |      141.7250 |      150.9665 |     140.1500 | 1.637298e+12 |      2121760 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2021-12-22 |      533654820 |            141.8085 |   140.0000 |      139.4805 |      148.3440 |     133.1645 | 1.639890e+12 |      1951440 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2022-01-22 |     1063968780 |            137.6396 |   136.1750 |      137.7380 |      151.5466 |     124.5000 | 1.642482e+12 |      3953735 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2022-02-22 |      753837380 |            131.3872 |   136.2430 |      136.1255 |      137.1135 |     124.9533 | 1.645074e+12 |      2966178 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2022-03-22 |      549816780 |            137.0237 |   136.1635 |      126.7300 |      143.7935 |     126.6010 | 1.647662e+12 |      2172034 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2022-04-22 |      926502060 |            117.8614 |   127.0000 |      116.4730 |      131.3990 |     109.8245 | 1.650254e+12 |      3504346 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2022-05-22 |      829581120 |            110.7138 |   115.0000 |      106.0335 |      119.3470 |     101.8847 | 1.652846e+12 |      2659443 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2022-06-22 |      719245340 |            112.6551 |   106.0335 |      111.7775 |      119.6850 |     105.0460 | 1.655438e+12 |      2110910 |
| Alphabet Inc. Class A Common Stock | GOOGL | 2022-07-22 |      637290733 |            113.5351 |   112.6400 |      121.6800 |      121.6800 |     104.0700 | 1.658030e+12 |      7543377 |
| Microsoft Corp                     | MSFT  | 2021-07-22 |      472721021 |            289.8730 |   283.8400 |      304.3600 |      305.8400 |     282.9500 | 1.626926e+12 |      5617373 |
| Microsoft Corp                     | MSFT  | 2021-08-22 |      401478949 |            301.1109 |   303.2450 |      299.8700 |      305.6500 |     294.0800 | 1.629518e+12 |      4936125 |
| Microsoft Corp                     | MSFT  | 2021-09-22 |      552612933 |            293.2810 |   296.3300 |      308.2300 |      309.3000 |     280.2500 | 1.632110e+12 |      7307952 |
| Microsoft Corp                     | MSFT  | 2021-10-22 |      518412228 |            327.9420 |   309.2100 |      341.2700 |      342.4500 |     306.1100 | 1.634702e+12 |      6657477 |
| Microsoft Corp                     | MSFT  | 2021-11-22 |      644157367 |            332.4192 |   342.6400 |      323.8000 |      349.6700 |     317.2500 | 1.637298e+12 |      8343759 |
| Microsoft Corp                     | MSFT  | 2021-12-22 |      549385946 |            322.0228 |   320.0500 |      310.2000 |      344.3000 |     303.7500 | 1.639890e+12 |      8094197 |
| Microsoft Corp                     | MSFT  | 2022-01-22 |     1007260429 |            301.2363 |   304.0700 |      299.5000 |      315.1200 |     276.0500 | 1.642482e+12 |     14555251 |
| Microsoft Corp                     | MSFT  | 2022-02-22 |      754206539 |            288.7100 |   296.3600 |      300.4300 |      303.1300 |     270.0000 | 1.645074e+12 |     11135200 |
| Microsoft Corp                     | MSFT  | 2022-03-22 |      536338767 |            301.2082 |   298.8900 |      279.8300 |      315.9500 |     279.3200 | 1.647662e+12 |      7102751 |
| Microsoft Corp                     | MSFT  | 2022-04-22 |      800288645 |            274.4439 |   278.9100 |      266.8200 |      293.3000 |     250.0200 | 1.650254e+12 |     10920872 |
| Microsoft Corp                     | MSFT  | 2022-05-22 |      642435254 |            259.9196 |   263.0000 |      244.9700 |      277.6900 |     241.5100 | 1.652846e+12 |      7859777 |
| Microsoft Corp                     | MSFT  | 2022-06-22 |      512373574 |            258.0435 |   244.7000 |      256.7200 |      269.0550 |     244.0300 | 1.655438e+12 |      6150427 |
| Microsoft Corp                     | MSFT  | 2022-07-22 |      497212452 |            271.9176 |   259.7500 |      291.9100 |      291.9100 |     249.5700 | 1.658030e+12 |      6160457 |
| Weyerhaeuser Company               | WY    | 2021-07-22 |       71143588 |             34.3965 |    34.0000 |       34.1600 |       35.8500 |      33.2813 | 1.626926e+12 |       524768 |
| Weyerhaeuser Company               | WY    | 2021-08-22 |       66124940 |             35.5595 |    34.4700 |       36.1700 |       36.7100 |      33.9700 | 1.629518e+12 |       483842 |
| Weyerhaeuser Company               | WY    | 2021-09-22 |       84702165 |             36.5796 |    35.4500 |       36.7700 |       38.2900 |      35.0300 | 1.632110e+12 |       633058 |
| Weyerhaeuser Company               | WY    | 2021-10-22 |       84212085 |             37.2660 |    36.8700 |       37.9400 |       38.8700 |      35.5200 | 1.634702e+12 |       655513 |
| Weyerhaeuser Company               | WY    | 2021-11-22 |       73639947 |             38.9839 |    37.9800 |       39.6400 |       40.3400 |      37.6100 | 1.637298e+12 |       535574 |
| Weyerhaeuser Company               | WY    | 2021-12-22 |       57984875 |             40.1362 |    39.1000 |       40.9800 |       41.8000 |      37.9100 | 1.639890e+12 |       487523 |
| Weyerhaeuser Company               | WY    | 2022-01-22 |       91484053 |             40.2322 |    40.6400 |       41.8600 |       43.0400 |      37.1306 | 1.642482e+12 |       788428 |
| Weyerhaeuser Company               | WY    | 2022-02-22 |      113173595 |             39.0399 |    39.9100 |       39.9900 |       40.5000 |      36.4600 | 1.645074e+12 |       870170 |
| Weyerhaeuser Company               | WY    | 2022-03-22 |       74688957 |             38.5465 |    40.0000 |       39.9000 |       40.5300 |      37.1000 | 1.647662e+12 |       645709 |
| Weyerhaeuser Company               | WY    | 2022-04-22 |       93504442 |             40.3864 |    39.4500 |       39.3800 |       42.8600 |      37.5250 | 1.650254e+12 |       873022 |
| Weyerhaeuser Company               | WY    | 2022-05-22 |      110931546 |             37.5082 |    39.0500 |       32.7700 |       40.3500 |      32.5800 | 1.652846e+12 |       970145 |
| Weyerhaeuser Company               | WY    | 2022-06-22 |       93757037 |             33.9967 |    33.0900 |       34.7800 |       35.2400 |      32.5000 | 1.655438e+12 |       733930 |
| Weyerhaeuser Company               | WY    | 2022-07-22 |       66569673 |             35.6040 |    34.9500 |       36.8900 |       37.2000 |      34.1500 | 1.658030e+12 |       603312 |
| Rayonier Inc.                      | RYN   | 2021-07-22 |       10319422 |             36.9909 |    36.4000 |       36.7400 |       38.3600 |      35.5100 | 1.626926e+12 |       131096 |
| Rayonier Inc.                      | RYN   | 2021-08-22 |       12349725 |             36.9763 |    37.0000 |       37.3100 |       38.4100 |      35.1300 | 1.629518e+12 |       130506 |
| Rayonier Inc.                      | RYN   | 2021-09-22 |       11624066 |             36.3528 |    36.7100 |       36.6300 |       37.9600 |      34.2700 | 1.632110e+12 |       128693 |
| Rayonier Inc.                      | RYN   | 2021-10-22 |       10144414 |             38.9148 |    36.5000 |       40.6100 |       41.0900 |      36.4800 | 1.634702e+12 |       122247 |
| Rayonier Inc.                      | RYN   | 2021-11-22 |       11695521 |             38.9214 |    40.6400 |       39.2100 |       40.9500 |      36.8600 | 1.637298e+12 |       132494 |
| Rayonier Inc.                      | RYN   | 2021-12-22 |        7892198 |             39.4806 |    38.8200 |       39.3000 |       40.9800 |      37.5800 | 1.639890e+12 |       100977 |
| Rayonier Inc.                      | RYN   | 2022-01-22 |       13968870 |             36.9260 |    38.8200 |       38.4700 |       39.0900 |      34.8000 | 1.642482e+12 |       166773 |
| Rayonier Inc.                      | RYN   | 2022-02-22 |       13278922 |             40.4555 |    38.2600 |       41.0800 |       43.4500 |      36.2800 | 1.645074e+12 |       162059 |
| Rayonier Inc.                      | RYN   | 2022-03-22 |       10095816 |             42.0481 |    41.0300 |       44.5900 |       44.6600 |      40.5000 | 1.647662e+12 |       122587 |
| Rayonier Inc.                      | RYN   | 2022-04-22 |       14626640 |             41.1727 |    44.4000 |       38.7200 |       45.8700 |      35.6700 | 1.650254e+12 |       190595 |
| Rayonier Inc.                      | RYN   | 2022-05-22 |       11813034 |             39.3871 |    38.5200 |       36.2200 |       42.0499 |      36.0800 | 1.652846e+12 |       181590 |
| Rayonier Inc.                      | RYN   | 2022-06-22 |       11576302 |             36.8384 |    36.2800 |       34.4500 |       38.6200 |      34.0700 | 1.655438e+12 |       147239 |
| Rayonier Inc.                      | RYN   | 2022-07-22 |       11999127 |             36.5129 |    34.7000 |       37.7700 |       38.4700 |      34.2400 | 1.658030e+12 |       141650 |

``` r
macd_df = out$macd_df
macd_df
```

| Company_Name                       |    timestamp |      value |     signal |  histogram |
|:-----------------------------------|-------------:|-----------:|-----------:|-----------:|
| Apple Inc.                         | 1.658531e+12 |  0.0312818 |  0.2570881 | -0.2258062 |
| Apple Inc.                         | 1.658527e+12 |  0.0751516 |  0.3135396 | -0.2383880 |
| Apple Inc.                         | 1.658524e+12 |  0.1239431 |  0.3731366 | -0.2491935 |
| Apple Inc.                         | 1.658520e+12 |  0.1793252 |  0.4354350 | -0.2561099 |
| Apple Inc.                         | 1.658516e+12 |  0.2209317 |  0.4994625 | -0.2785308 |
| Apple Inc.                         | 1.658513e+12 |  0.2871141 |  0.5690952 | -0.2819811 |
| Apple Inc.                         | 1.658509e+12 |  0.4025380 |  0.6395905 | -0.2370525 |
| Apple Inc.                         | 1.658506e+12 |  0.5226239 |  0.6988536 | -0.1762296 |
| Apple Inc.                         | 1.658502e+12 |  0.6160322 |  0.7429110 | -0.1268788 |
| Apple Inc.                         | 1.658498e+12 |  0.7163979 |  0.7746307 | -0.0582328 |
| Alphabet Inc. Class A Common Stock | 1.658531e+12 | -1.3111350 | -1.2073334 | -0.1038015 |
| Alphabet Inc. Class A Common Stock | 1.658527e+12 | -1.3484506 | -1.1813831 | -0.1670675 |
| Alphabet Inc. Class A Common Stock | 1.658524e+12 | -1.3673014 | -1.1396162 | -0.2276853 |
| Alphabet Inc. Class A Common Stock | 1.658520e+12 | -1.3724921 | -1.0826949 | -0.2897972 |
| Alphabet Inc. Class A Common Stock | 1.658516e+12 | -1.3703538 | -1.0102456 | -0.3601082 |
| Alphabet Inc. Class A Common Stock | 1.658513e+12 | -1.3324193 | -0.9202185 | -0.4122008 |
| Alphabet Inc. Class A Common Stock | 1.658509e+12 | -1.2524023 | -0.8171683 | -0.4352340 |
| Alphabet Inc. Class A Common Stock | 1.658506e+12 | -1.0852185 | -0.7083598 | -0.3768587 |
| Alphabet Inc. Class A Common Stock | 1.658502e+12 | -0.8647006 | -0.6141451 | -0.2505555 |
| Alphabet Inc. Class A Common Stock | 1.658498e+12 | -0.6577167 | -0.5515062 | -0.1062104 |
| Microsoft Corp                     | 1.658531e+12 | -0.5768889 | -0.1695997 | -0.4072893 |
| Microsoft Corp                     | 1.658527e+12 | -0.5197995 | -0.0677774 | -0.4520222 |
| Microsoft Corp                     | 1.658524e+12 | -0.4546447 |  0.0452282 | -0.4998729 |
| Microsoft Corp                     | 1.658520e+12 | -0.3811613 |  0.1701964 | -0.5513577 |
| Microsoft Corp                     | 1.658516e+12 | -0.3123711 |  0.3080358 | -0.6204069 |
| Microsoft Corp                     | 1.658513e+12 | -0.1996626 |  0.4631375 | -0.6628001 |
| Microsoft Corp                     | 1.658509e+12 | -0.0055558 |  0.6288375 | -0.6343934 |
| Microsoft Corp                     | 1.658506e+12 |  0.2529565 |  0.7874359 | -0.5344794 |
| Microsoft Corp                     | 1.658502e+12 |  0.5242612 |  0.9210557 | -0.3967946 |
| Microsoft Corp                     | 1.658498e+12 |  0.8221918 |  1.0202544 | -0.1980626 |
| Weyerhaeuser Company               | 1.658531e+12 |  0.0609735 |  0.0744580 | -0.0134845 |
| Weyerhaeuser Company               | 1.658527e+12 |  0.0615730 |  0.0778291 | -0.0162561 |
| Weyerhaeuser Company               | 1.658524e+12 |  0.0655153 |  0.0818931 | -0.0163778 |
| Weyerhaeuser Company               | 1.658520e+12 |  0.0695942 |  0.0859876 | -0.0163934 |
| Weyerhaeuser Company               | 1.658516e+12 |  0.0737881 |  0.0900859 | -0.0162978 |
| Weyerhaeuser Company               | 1.658513e+12 |  0.0770495 |  0.0941604 | -0.0171108 |
| Weyerhaeuser Company               | 1.658509e+12 |  0.0872188 |  0.0984381 | -0.0112192 |
| Weyerhaeuser Company               | 1.658506e+12 |  0.1030027 |  0.1012429 |  0.0017598 |
| Weyerhaeuser Company               | 1.658502e+12 |  0.1033231 |  0.1008029 |  0.0025202 |
| Weyerhaeuser Company               | 1.658498e+12 |  0.1073201 |  0.1001729 |  0.0071473 |
| Rayonier Inc.                      | 1.658520e+12 |  0.1007150 |  0.1519496 | -0.0512345 |
| Rayonier Inc.                      | 1.658516e+12 |  0.1299322 |  0.1647582 | -0.0348260 |
| Rayonier Inc.                      | 1.658513e+12 |  0.1510795 |  0.1734647 | -0.0223852 |
| Rayonier Inc.                      | 1.658509e+12 |  0.1712917 |  0.1790610 | -0.0077693 |
| Rayonier Inc.                      | 1.658506e+12 |  0.1808529 |  0.1810033 | -0.0001504 |
| Rayonier Inc.                      | 1.658502e+12 |  0.1710805 |  0.1810409 | -0.0099604 |
| Rayonier Inc.                      | 1.658498e+12 |  0.1601916 |  0.1835310 | -0.0233394 |
| Rayonier Inc.                      | 1.658495e+12 |  0.1342906 |  0.1893659 | -0.0550753 |
| Rayonier Inc.                      | 1.658491e+12 |  0.1621509 |  0.2031347 | -0.0409838 |
| Rayonier Inc.                      | 1.658488e+12 |  0.1523340 |  0.2133807 | -0.0610467 |

## Summary Wrapper Function

This function takes the object returned by the wrapper function and
produces all the data for the EDA.

``` r
data_management = function(list_data,...){
  
  # assigning objects with the corresponding list elements from the calls
  df = list_data$df
  
  time_df = list_data$time_df
  
  macd_df = list_data$macd_df
  
  # converting Unix_time to date format
  df<- df %>%
    mutate(Date= as.POSIXct(df$Unix_time/1000,origin = "1970-01-01"))
  
  df$Date<-as.Date(df$Date)
  
  # creating typical price variable = the average among highest, lowest, and
  # closing price
  df<- df %>%
    mutate(Sum_Typical_price = (Closing_price + Highest_price + Lowest_price)/3)
  
  # sub-setting important variables
  # creating the table with the averages for all important variables by
  # each level of market and type variables.
  tab_cate2 <-  df%>%
    select(- c("ticker","name","composite_figi","share_class_figi","Date")) %>%
    group_by(market,type) %>%
    summarise_all(mean)
  
  # creating the price object with the mean of closing price and price range by
  # all combinations of levels of market and type
  df_price = df %>% 
    group_by(market, type) %>%
    summarise(avg_price = mean( Closing_price), price_range = (Highest_price - Lowest_price))
  
  # 
  df_filter_price = df %>%
    filter( Closing_price > 1.5*mean(df$ Closing_price) & Closing_price < max( Closing_price))
  
  time_df_summary <- time_df %>% 
    select(-c( "tckr","Date")) %>% 
    group_by(Company_Name) %>% 
    summarize_all(mean)
  
  time_df$Industry <-ifelse(time_df$tckr=="WY" | time_df$tckr=="WY" , "Real-State", "Technology")
  
  
  output =   list(df = df, tab_cate2 = tab_cate2, df_price = df_price, df_filter_price = df_filter_price, time_df_summary = time_df_summary, time_df = time_df)
  
  return(output)
  
}


output = data_management(list_data = out)
```

## For several tickers data - df

Before proceeding to the detailed EDA of the data sets taken from the
APIs, some minor changes have been made to the data frame df. The
columns have been renamed as

c - Closing_price h - Highest_price l - Lowest_price n - Transactions
o - Open_price t - Unix_time v- Trading_volume vW- Volume_wt_avg_price

Also,the unix time format has been changed to date format.

An additional parameter called Typical price is added to the table. This
is used to determine volume weighted price. Typical price has been
derived as sum of the closing price , highest price and lower price.

``` r
  # df<-df %>% 
  #   rename(
  #     Closing_price=c,
  #     Highest_price=h,
  #     Lowest_price=l, 
  #     Transactions=n,
  #     Open_price=o, 
  #     Unix_time=t,
  #     Trading_volume=v,
  #     Volume_wt_avg_price=vw
  #   )
  # 
  # df<- df %>% mutate(Date= as.POSIXct(df$Unix_time/1000,origin = "1970-01-01"))
  # df$Date<-as.Date(df$Date)
  # 
  # df<- df %>% mutate(Sum_Typical_price= (Closing_price+Highest_price+Lowest_price)/3)
```

##### Contingency Table to count tickers

The 1st contingency table includes total count of tickers based stocks
and otc securities. There are 193 otc based tickers and 678 stock based
tickers.

The 2nd contingency table includes total count of tickers based on type.
Maximum tickers are of CS investment type.

The 3rd contingency table states count of pooled investment based on the
market for the given ticker data set. And also graphs it in a bar plot.
This shows that there were maximum tickers in the stock market for CS
and maximum in OTC for OS. It was noted that tickers/companies with
ADRC, CS,OS and UNIT investments had both OTC markets. Rest of tickers
had just stock market based investments.

``` r
# for categorical and numerical EDA
kable(table(output$df$market), col.names = c("Market Type","Frequency"))
```

| Market Type | Frequency |
|:------------|----------:|
| otc         |       190 |
| stocks      |       683 |

``` r
kable(table(output$df$type), col.names = c("Ticker Type","Frequency"))
```

| Ticker Type | Frequency |
|:------------|----------:|
| ADRC        |        72 |
| CS          |       466 |
| ETF         |       205 |
| ETN         |         5 |
| ETV         |         6 |
| FUND        |        35 |
| OS          |        70 |
| UNIT        |        14 |

``` r
kable(table(output$df$market, output$df$type), caption = "Contingency Table of Market Type by Ticker Type")
```

|        | ADRC |  CS | ETF | ETN | ETV | FUND |  OS | UNIT |
|:-------|-----:|----:|----:|----:|----:|-----:|----:|-----:|
| otc    |   44 |  74 |   0 |   0 |   0 |    0 |  70 |    2 |
| stocks |   28 | 392 | 205 |   5 |   6 |   35 |   0 |   12 |

Contingency Table of Market Type by Ticker Type

``` r
# type vs market
g = ggplot(output$df, aes(x = market))
g + geom_bar(aes(fill = type), position = "dodge") + 
  labs(x = "Type of Investment", title="Number of Market Type Available by Ticker Type") + scale_fill_discrete(name = "Ticker Type")
```

![](README_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

##### Summary Table for all quantitative variables.

The summary table displays mean of all the quantitative parameters based
on two groups- market and type . This uses group_by to group categorical
variables and mean of all the quantitative variables are calculated for
the subsequent groups.

The Trading volume was maximum for OTC/CS type tickers followed by
Stocks/ETV. The mean volume weighted price is maximum for tickers with
maximum price for the given price variables.

``` r
kable(output$tab_cate2)
```

| market | type | Trading_volume | Volume_wt_avg_price | Open_price | Closing_price | Highest_price | Lowest_price |   Unix_time | Transactions | Sum_Typical_price |
|:-------|:-----|---------------:|--------------------:|-----------:|--------------:|--------------:|-------------:|------------:|-------------:|------------------:|
| otc    | ADRC |     104332.227 |           25.319275 |  25.456620 |     25.221296 |     25.595718 |    25.101807 | 1.65852e+12 |    176.97727 |         25.306273 |
| otc    | CS   |   23779311.378 |            4.738820 |   4.740364 |      4.732953 |      4.751451 |     4.721854 | 1.65852e+12 |    181.68919 |          4.735419 |
| otc    | OS   |      31769.329 |            1.775972 |   1.785386 |      1.753709 |      1.801946 |     1.740986 | 1.65852e+12 |     12.71429 |          1.765547 |
| otc    | UNIT |      12464.000 |            5.972900 |   6.000000 |      5.974000 |      6.010000 |     5.954000 | 1.65852e+12 |     23.50000 |          5.979333 |
| stocks | ADRC |    1071487.714 |           21.934853 |  22.135546 |     21.792646 |     22.401757 |    21.625571 | 1.65852e+12 |   6383.03571 |         21.939992 |
| stocks | CS   |    1327825.402 |           42.842359 |  43.212892 |     42.777576 |     43.749069 |    42.264742 | 1.65852e+12 |  11336.96939 |         42.930462 |
| stocks | ETF  |     974294.776 |           39.763325 |  39.941929 |     39.664408 |     40.079720 |    39.474093 | 1.65852e+12 |   3476.44878 |         39.739407 |
| stocks | ETN  |      12649.800 |           33.249500 |  33.221100 |     32.975140 |     33.728000 |    32.844040 | 1.65852e+12 |     61.80000 |         33.182393 |
| stocks | ETV  |    1571428.333 |           42.923583 |  42.481050 |     42.914467 |     43.784433 |    42.266550 | 1.65852e+12 |   7638.00000 |         42.988483 |
| stocks | FUND |      75510.029 |           12.325657 |  12.367143 |     12.294714 |     12.450237 |    12.233200 | 1.65852e+12 |    354.54286 |         12.326050 |
| stocks | UNIT |       2996.083 |            9.923975 |   9.929167 |      9.917083 |      9.931250 |     9.915000 | 1.65852e+12 |     11.66667 |          9.921111 |

``` r
# df_summary<-df%>% select(- c("ticker","name","composite_figi","share_class_figi","Date"))
# tab_cate2<-df_summary%>% group_by(market,type) %>%  summarise_all(mean)
# tab_cate2
```

### Modified Data and Related Plots.

In 1st density plot, Average price and Price range has been derived from
the data and has been used to plot some trends. The count of tickers has
has been same in the stock market in terms of average price.

2nd density plot shows average price based on the type of investment.
This was seen to be maximum for OS type investment.

3rd plot having a box plot summarizes spread of average price for the
two type of markets. OTC markets cover broader price range.

4th Plot or box plot captures average price in terms of the investment
type with ADRC types having broader average price share in the given
time period.

``` r
# quantitative vs market & type
# group by type and market and average

# df_price = df %>% 
#   group_by(market, type) %>%
#   summarise(avg_price = mean( Closing_price), price_range = (Highest_price - Lowest_price))

df_price = output$df_price

h = ggplot(df_price, aes(x = avg_price))

# histograms by market type
h + geom_density(adjust = 0.5, alpha = 0.5, aes(fill = market))+ labs(x = "Type of Investment", y = "Count",title=" Avg Price based on Market")+scale_fill_discrete(name = "Market ")
```

![](README_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

``` r
h + geom_histogram(aes(fill = type, y = ..density..), position = "dodge") + 
  geom_density(adjust = 0.5, alpha = 0.5, aes(fill = type))+ labs(x = "Market", y = "Count/Denstiy",title=" Avg Price based on Market")+scale_fill_discrete(name = "Market ")
```

![](README_files/figure-gfm/unnamed-chunk-4-2.png)<!-- -->

``` r
# box plot by market and type for avg price
h + geom_boxplot(aes(y = market,fill=market)) + coord_flip()+ labs(x = "Avg price", y = "Type",title=" Avg Price based on Type")+scale_fill_discrete(name = "Market ")
```

![](README_files/figure-gfm/unnamed-chunk-4-3.png)<!-- -->

``` r
h + geom_boxplot(aes(y = type,fill=type)) +coord_flip()+ labs(x = "Avg Price", y = "Market",title=" Box Plot showing Avg Price based on Market")+scale_fill_discrete(name = "Market ")
```

![](README_files/figure-gfm/unnamed-chunk-4-4.png)<!-- -->

An Empirical Cumulative distribution function is set up based on market
where data with closing price is filtered at criteria and range of price
is plotted.

The scatter plots summarize the relation between Closing price and
Highest price both in terms of market and investment type. In all the
cases highest price and closing price have shown linear and string
correlation.

``` r
# Empirical CDF by market type - price 50% above price avg up to max price
# df_filter_price = df %>%
#   filter( Closing_price > 1.5*mean(df$ Closing_price) & Closing_price < max( Closing_price))

df_filter_price = output$df_filter_price

h1 = ggplot(df_filter_price, aes(x =  Closing_price))
h1 + stat_ecdf(geom = "step", aes(color = market)) + ylab("ECDF")+labs(title ="ECDF Plot capturing Closing Price based on market")
```

![](README_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

``` r
h1 + stat_ecdf(geom = "step", aes(color = type)) + ylab("ECDF")+labs(title ="ECDF Plot capturing Closing Price based on type")
```

![](README_files/figure-gfm/unnamed-chunk-5-2.png)<!-- -->

``` r
# scatter plot
h1 + geom_point(aes(y = Highest_price)) + facet_wrap(~market) +labs(x= "Highest Price", y= "Closing" ,title ="Closing Price  vs Higest Price grouped by Type")
```

![](README_files/figure-gfm/unnamed-chunk-5-3.png)<!-- -->

``` r
h1 + geom_point(aes(y = Highest_price)) + facet_wrap(~type)+labs(x= "Highest Price", y= "Closing Price" ,title ="Closing Price  vs Higest Price grouped by Market")
```

![](README_files/figure-gfm/unnamed-chunk-5-4.png)<!-- -->

# Timeseries Data frame- Time_df

the columns have been renamed to easily identify the variables.

``` r
# time_df<-time_df %>% 
#   rename(
#     Closing_price=c,
#     Highest_price=h,
#     Lowest_price=l, 
#     Transactions=n,
#     Open_price=o, 
#     Unix_time=t,
#     Trading_volume=v,
#     Volume_wt_avg_price=vw,
#     Date=d
#    )
```

#### Summary Table

This table summarizes the means of the pricing, transactions and volume
for the select technology and timberland companies. There is no relation
b/w two industries but this data is pretty obvious that the Tech giants
price more and have bigger penetration in the trading business. Majority
of these companies have world-wide presence compare to timberland
companies. These tech giants have net worth approx in billions and
having a price in the range of \$ 100+ for given time period is not
surprising. Apple is the company having highest trading volume and
Microsoft had the highest average prices for the given time range.

``` r
# time_df_summary<- select(time_df, -c( "tckr","Date"))
# time_df_summary<-time_df_summary %>% group_by(Company_Name) %>% summarize_all(mean)
# time_df_summary

kable(output$time_df_summary)
```

| Company_Name                       | Trading_volume | Volume_wt_avg_price | Open_price | Closing_price | Highest_price | Lowest_price |   Unix_time | Transactions |
|:-----------------------------------|---------------:|--------------------:|-----------:|--------------:|--------------:|-------------:|------------:|-------------:|
| Alphabet Inc. Class A Common Stock |      700706966 |           131.42652 |  131.27257 |     131.46677 |     138.67452 |    123.21341 | 1.64248e+12 |    2788529.7 |
| Apple Inc.                         |     1797667668 |           156.55936 |  154.63846 |     157.57538 |     165.87077 |    146.84368 | 1.64248e+12 |   14171324.8 |
| Microsoft Corp                     |      606837239 |           294.00985 |  292.38423 |     294.45462 |     309.48962 |    276.53000 | 1.64248e+12 |    8064739.8 |
| Rayonier Inc.                      |       11644927 |            38.53673 |   38.31385 |      38.54615 |      40.76615 |     35.95923 | 1.64248e+12 |     142962.0 |
| Weyerhaeuser Company               |       83224377 |            37.55658 |   37.30462 |      37.78692 |      39.35231 |     35.44361 | 1.64248e+12 |     677307.2 |

##### Bar Plot with Facet Wrap.

The Mean of transactions grouped by Industry type has been shown below
as we have 3 technology companies and 2 timberland companies selected
for the subplots. This shows again transaction wise Technology companies
have a higher trend. In the month of Jan 2022 Technology companies had
maximum transactions and timberland companies showing higher numbers in
the month of May.

``` r
time_df = output$time_df

ggplot(time_df, aes(x=as.factor(Date), y=Transactions,fill=Industry))+ geom_boxplot()+ facet_wrap(~Industry,  nrow=2,scales = "free" )+guides(x = guide_axis(angle = 90))+ labs(x= "Transactions", y= "Date",title=" Mean Transactions for the time period grouped by Industry type")+scale_fill_discrete(name = "Industry Type ")
```

![](README_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

#### Line Plots

The Volume weighted price trend has been captured in the below plot for
each company. Apple as expected showing highest trends line. It saw
lowest numbers in the Jul 2021 and again in Jul 2022. It is interesting
to note all technology companies saw peaks in the month of Nov 2021, Jan
2022 and Apr 2022

``` r
ggplot(time_df, aes(x = Date, y = Transactions, colour =Company_Name, group = Company_Name)) +geom_line() + geom_point()+labs(x= "Date", y= "Volume Weighted Average Price",title=" Volume weighted price for the time range grouped by Companies")
```

![](README_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

##### MACD (Moving Average Convergence/Divergence) - macd_df

This section is just for exploring the concept of technical endpoint of
the API stated. Lot of calculations like macd , moving averages etc are
used to estimate the trend of the stock and get observations which help
in deciding if the given time correct for buying or selling.

MACD (Moving Average Convergence/Divergence) is an oscillator study and
is widely used for assessment of trending characteristics of a given
security. It is calculated as the difference between two price averages,
and is a indicator that provides a signal line which average of that
difference. Interaction of the MACD plot and the signal line often
produce valuable signals for trend analysis.

When the MACD plot goes above the signal line, an uptrend may be
expected, and, when it goes below, a downtrend can be seen. The
difference between the MACD and signal values is plotted as a histogram,
which may sometimes give you an early sign that a crossover is about to
happen. This relation between two can help in determining if
stock/security can be bought or not. The plot shows a trend between the
signal and MACD value for various companies keeping histogram as base.
Slight interaction between value and signal can be seen in the data at
some points. As the data is limited only few points have been captured
for companies. For given data crossover trend can clearly be seen for
Microsoft and Weyerhaeuser.

``` r
g <- ggplot(data =macd_df, aes(x=histogram))
g + geom_line(aes(y=value),color="red")+geom_line(aes(y=signal),color="blue") + facet_wrap(~Company_Name,scales = "free")+guides(x = guide_axis(angle = 90))+ labs(x= "Signal", y= "Value",title=" Signal vs Value")+scale_fill_discrete(name = "Company Name ")
```

![](README_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->
