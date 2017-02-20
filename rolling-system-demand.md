Rolling System Demand
================

API Request
-----------

``` r
# get API key from user  -------------------------------------------------------
api_key <- getPass("Enter API key")
stopifnot(!is.null(api_key))

# API query data 
report_type <- "BMRS/ROLSYSDEM"            # name of report to request from API
start_datetime <- "2017-02-13 00:00:00"
end_datetime <- "2017-02-20 00:00:00"

# construct query API URL ------------------------------------------------------
service_url <- paste(
    "https://api.bmreports.com:443",
    report_type,
    "v1",
    sep = "/")

query_url <- service_url %>% 
    param_set("APIKey", api_key) %>% 
    param_set("FromDateTime", url_encode(start_datetime)) %>% 
    param_set("ToDateTime", url_encode(end_datetime)) %>% 
    param_set("ServiceType", "xml")  

# connect to API and request data ---------------------------------------------
print(paste0("[", Sys.time(), "] Connected to BM Reports API ..."))
```

    ## [1] "[2017-02-20 11:38:32] Connected to BM Reports API ..."

``` r
xmlfile <- read_xml(query_url)

# tidy up sensitive information -----------------------------------------------
rm(api_key,  query_url)
```

Tidy Data
---------

``` r
# extract raw data from XML document and save ---------------------------------
data_xpaths <- list(
    date = "//settDate", 
    time = "//publishingPeriodCommencingTime", 
    generation = "//fuelTypeGeneration"
)

df_raw <- pmap(list(xpath = data_xpaths), xml_find_all, x = xmlfile) %>% 
    map_df(xml_text) %>% 
    mutate(
           date_time = ymd_hms(paste(date, time), tz = "Europe/London"),
           generation = as.numeric(generation)) %>% 
    select(date_time, generation, -c(date, time))
```

Plot Data
---------

``` r
my_breaks <- seq(from = ymd_hms(start_datetime, tz = 'Europe/London'), 
                 to = ymd_hms(end_datetime, tz = 'Europe/London'),
                 by = '6 hours')

ggplot(df_raw, aes(x = date_time, y = generation)) +
    geom_line(colour = 'red') +
    labs(
        title = "Characteristic 'seasonality' of rolling system demand",
        subtitle = "Seasonality has a period of 24 hours",
        caption = "Data from https://api.bmreports",
        x = "Time",
        y = "Demand (MW)"
    ) +
    scale_x_datetime(date_breaks = "1 day", minor_breaks = my_breaks, 
                     labels = date_format("%a"))
```

![](rolling-system-demand_files/figure-markdown_github/unnamed-chunk-3-1.png)

STL Decompostion
----------------

``` r
ts_raw <- ts(df_raw$generation, frequency = 288)
stl_raw <- stl(ts_raw, s.window = 50)
autoplot(stl_raw)
```

![](rolling-system-demand_files/figure-markdown_github/unnamed-chunk-4-1.png)

Forecast
--------

``` r
fcast <- forecast(ts_raw, method = 'ets')
autoplot(fcast)
```

![](rolling-system-demand_files/figure-markdown_github/unnamed-chunk-5-1.png)