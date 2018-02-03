
rstudio::conf tweets
====================

A repository for tracking [tweets](https://twitter.com/hashtag/rstudioconf?f=tweets&vertical=default&src=hash) about [rstudio::conf 2018](https://www.rstudio.com/conference/).

rtweet
------

To make it easier to request data from Twitter's APIs, install \[if it's not already\] and load [rtweet](http://rtweet.info).

``` r
## install rtweet if not already
if (!requireNamespace("rtweet", quietly = TRUE)) {
  install.packages("rtweet")
}

## load rtweet
library(rtweet)
```

Data
----

Two data collection methods are described in detail below. Hoewver, if you want to skip straight to the data, run the following code:

``` r
## download status IDs file
download.file(
  "https://github.com/mkearney/rstudioconf_tweets/blob/master/data/search-ids.rds?raw=true",
  "rstudioconf_search-ids.rds"
)

## read status IDs fromdownloaded file
ids <- readRDS("rstudioconf_search-ids.rds")

## lookup data associated with status ids
rt <- rtweet::lookup_tweets(ids$status_id)
```

Stream
------

There are two easy ways to get Twitter data filtered by one or more keywords. The first way is to stream the data (using Twitter's stream API). For example, in the code below, a stream is setup to run continuously from the moment its executed until the Saturday at midnight (to roughly coincide with the end of the conference).

``` r
## set stream time
timeout <- as.numeric(
  difftime(as.POSIXct("2018-02-04 00:00:00"),
  Sys.time(), tz = "US/Pacific", "secs")
)

## search terms
rstudioconf <- c("rstudioconf", "rstudio::conf",
  "rstudioconference", "rstudioconference18",
  "rstudioconference2018", "rstudio18",
  "rstudioconf18", "rstudioconf2018",
  "rstudio::conf18", "rstudio::conf2018")

## name of file to save output
json_file <- file.path("data", "stream.json")

## stream the tweets and write to "data/stream.json"
stream_tweets(
  q = paste(rstudioconf, collapse = ","),
  timeout = timeout,
  file_name = json_file,
  parse = FALSE
)

## parse json data and convert to tibble
rt <- parse_stream(json_file)
```

Search
------

The second easy way to gather Twitter data using one or more keywords is to search for the data (using Twitter's REST API). Unlike streaming, searching makes it possible to go back in time. Unfortunately, Twitter sets a rather restrictive cap–roughly nine days–on how far back you can go. Regardless, searching for tweets is often the preferred method. For example, the code below is setup in such a way that it can be executed once \[or even several times\] a day throughout the conference.

``` r
## search terms
rstudioconf <- c("rstudioconf", "rstudio::conf",
  "rstudioconference", "rstudioconference18",
  "rstudioconference2018", "rstudio18",
  "rstudioconf18", "rstudioconf2018",
  "rstudio::conf18", "rstudio::conf2018")

## use since_id from previous search (if exists)
if (file.exists(file.path("data", "search.rds"))) {
  since_id <- readRDS(file.path("data", "search.rds"))
  since_id <- since_id$status_id[1]
} else {
  since_id <- NULL
}

## search for up to 100,000 tweets mentionging rstudio::conf
rt <- search_tweets(
  paste(rstudioconf, collapse = " OR "),
  n = 1e5, verbose = FALSE,
  since_id = since_id,
  retryonratelimit = TRUE
)

## if there's already a search data file saved, then read it in,
## drop the duplicates, and then update the `rt` data object
if (file.exists(file.path("data", "search.rds"))) {

  ## bind rows (for tweets AND users data)
  rt <- do_call_rbind(
    list(rt, readRDS(file.path("data", "search.rds"))))

  ## determine whether each observation has a unique status ID
  kp <- !duplicated(rt$status_id)

  ## only keep rows (observations) with unique status IDs
  users <- users_data(rt)[kp, ]

  ## the rows of users should correspond with the tweets
  rt <- rt[kp, ]

  ## restore as users attribute
  attr(rt, "users") <- users
}

## save the data
saveRDS(rt, file.path("data", "search.rds"))

## save shareable data (only status_ids)
saveRDS(rt[, "status_id"], file.path("data", "search-ids.rds"))
```

Explore
-------

To explore the Twitter data, go ahead and load the [tidyverse](http://tidyverse.org) packages.

``` r
suppressPackageStartupMessages(library(tidyverse))
```

### Tweet frequency over time

In the code below, the data is summarized into a time series-like data frame and then plotted in order depict the frequency of tweets–aggregated using 2-hour intevals–about rstudio::conf over time.

``` r
rt %>%
  filter(created_at > "2018-01-29") %>%
  ts_plot("2 hours", color = "transparent") +
  geom_smooth(method = "loess", se = FALSE, span = .25,
  size = 2, colour = "#0066aa") +
  geom_point(size = 5,
    shape = 21, fill = "#ADFF2F99", colour = "#000000dd") +
  theme_minimal(base_size = 15, base_family = "Roboto Condensed") +
  theme(axis.text = element_text(colour = "#222222"),
    plot.title = element_text(size = rel(1.7), face = "bold"),
    plot.subtitle = element_text(size = rel(1.3)),
    plot.caption = element_text(colour = "#444444")) +
  labs(title = "Frequency of tweets about rstudio::conf over time",
    subtitle = "Twitter status counts aggregated using two-hour intervals",
    caption = "\n\nSource: Data gathered via Twitter's standard `search/tweets` API using rtweet",
    x = NULL, y = NULL)
```

<p align="center">
<img width="100%" height="auto" src="README_files/figure-markdown_github/timefreq-1.png" />
</p>
 

### Positive/negative sentiment

Next, some sentiment analysis of the tweets so far.

``` r
## clean up the text a bit (rm mentions and links)
rt$text2 <- gsub(
  "^RT:?\\s{0,}|#|@\\S+|https?[[:graph:]]+", "", rt$text)
## convert to lower case
rt$text2 <- tolower(rt$text2)
## trim extra white space
rt$text2 <- gsub("^\\s{1,}|\\s{1,}$", "", rt$text2)
rt$text2 <- gsub("\\s{2,}", " ", rt$text2)

## estimate pos/neg sentiment for each tweet
rt$sentiment <- syuzhet::get_sentiment(rt$text2, "syuzhet")

## write function to round time into rounded var
round_time <- function(x, sec) {
  as.POSIXct(hms::hms(as.numeric(x) %/% sec * sec))
}

## plot by specified time interval (1-hours)
rt %>%
  mutate(time = round_time(created_at, 60 * 60)) %>%
  group_by(time) %>%
  summarise(sentiment = mean(sentiment, na.rm = TRUE)) %>%
  mutate(valence = ifelse(sentiment > 0L, "Positive", "Negative")) %>%
  ggplot(aes(x = time, y = sentiment)) +
  geom_smooth(method = "loess", span = .1,
    colour = "#aa11aadd", fill = "#bbbbbb11") +
  geom_point(aes(fill = valence, colour = valence), 
    shape = 21, alpha = .6, size = 3.5) +
  theme_minimal(base_size = 15, base_family = "Roboto Condensed") +
  theme(legend.position = "none",
    axis.text = element_text(colour = "#222222"),
    plot.title = element_text(size = rel(1.7), face = "bold"),
    plot.subtitle = element_text(size = rel(1.3)),
    plot.caption = element_text(colour = "#444444")) +
  scale_fill_manual(values = c(Positive = "#2244ee", Negative = "#dd2222")) +
  scale_colour_manual(values = c(Positive = "#001155", Negative = "#550000")) +
  labs(x = NULL, y = NULL,
    title = "Sentiment (valence) of rstudio::conf tweets over time",
    subtitle = "Mean sentiment of tweets aggregated in one-hour intervals",
    caption = "\nSource: Data gathered using rtweet. Sentiment analysis done using syuzhet")
```

<p align="center">
<img width="100%" height="auto" src="README_files/figure-markdown_github/sentiment-1.png" />
</p>
 

### Semantic networks

The code below provides a quick and dirty visualization of the semantic network (connections via retweet, quote, mention, or reply) found in the data.

``` r
## unlist observations into long-form data frame
unlist_df <- function(...) {
  dots <- lapply(list(...), unlist)
  tibble::as_tibble(dots)
}

## iterate by row
row_dfs <- lapply(
  seq_len(nrow(rt)), function(i)
    unlist_df(from_screen_name = rt$screen_name[i],
      reply = rt$reply_to_screen_name[i],
      mention = rt$mentions_screen_name[i],
      quote = rt$quoted_screen_name[i],
      retweet = rt$retweet_screen_name[i])
)

## bind rows, gather (to long), convert to matrix, and filter out NAs
rdf <- dplyr::bind_rows(row_dfs)
rdf <- tidyr::gather(rdf, interaction_type, to_screen_name, -from_screen_name)
mat <- as.matrix(rdf[, -2])
mat <- mat[apply(mat, 1, function(i) !any(is.na(i))), ]

## get rid of self references
mat <- mat[mat[, 1] != mat[, 2], ]

## filter out users who don't appear in RHS at least 3 times
apps1 <- table(mat[, 1])
apps1 <- apps1[apps1 > 1L]
apps2 <- table(mat[, 2])
apps2 <- apps2[apps2 > 1L]
apps <- names(apps1)[names(apps1) %in% names(apps2)]
mat <- mat[mat[, 1] %in% apps & mat[, 2] %in% apps, ]

## create graph object
g <- igraph::graph_from_edgelist(mat)

## calculate size attribute (and transform to fit)
matcols <- factor(c(mat[, 1], mat[, 2]), levels = names(igraph::V(g)))
size <- table(screen_name = matcols)
size <- (log(size) + sqrt(size)) / 3

## reorder freq table
size <- size[match(names(size), names(igraph::V(g)))]

## plot network
par(mar = c(12, 6, 15, 6))
plot(g,
  edge.size = .4,
  curved = FALSE,
  margin = -.05,
  edge.arrow.size = 0,
  edge.arrow.width = 0,
  vertex.color = "#ADFF2F99",
  vertex.size = size,
  vertex.frame.color = "#003366",
  vertex.label.color = "#003366",
  vertex.label.cex = .8,
  vertex.label.family = "Roboto Condensed",
  edge.color = "#0066aa",
  edge.width = .2,
  main = "")
par(mar = c(9, 6, 9, 6))
title("Semantic network of users tweeting about rstudio::conf",
  adj = 0, family = "Roboto Condensed", cex.main = 6.5)
mtext("Source: Data gathered using rtweet. Network analysis done using igraph",
  side = 1, line = 0, adj = 1.0, cex = 3.8,
  family = "Roboto Condensed", col = "#222222")
mtext("User connections by mentions, replies, retweets, and quotes",
  side = 3, line = -4.25, adj = 0,
  family = "Roboto Condensed", cex = 4.9)
```

<p align="center">
<img width="100%" height="auto" src="README_files/figure-markdown_github/network-1.png" />
</p>
 

Ideally, the network visualization would be an interactive, searchable graphic. Since it's not, I've printed out the node size values below.

``` r
print(as_tibble(sort(size, decreasing = TRUE)), n = length(size))
## # A tibble: 185 x 2
##     screen_name         n
##     <chr>           <dbl>
##   1 hadleywickham   8.99 
##   2 robinson_es     7.77 
##   3 rstudio         7.58 
##   4 drob            6.72 
##   5 RLadiesGlobal   6.47 
##   6 juliasilge      6.34 
##   7 LucyStats       6.23 
##   8 AmeliaMN        5.75 
##   9 JennyBryan      5.30 
##  10 stephhazlitt    5.28 
##  11 d4tagirl        5.28 
##  12 dataandme       5.08 
##  13 visnut          5.01 
##  14 EmilyRiederer   4.99 
##  15 romain_francois 4.91 
##  16 thmscwlls       4.89 
##  17 Voovarb         4.89 
##  18 CivicAngela     4.73 
##  19 sharon000       4.62 
##  20 kearneymw       4.55 
##  21 njogukennly     4.53 
##  22 datapointier    4.51 
##  23 SK_convergence  4.25 
##  24 minebocek       4.21 
##  25 sharlagelfand   4.18 
##  26 CorradoLanera   4.10 
##  27 elhazen         4.08 
##  28 tanyacash21     4.05 
##  29 malco_bearhat   4.05 
##  30 eamcvey         4.00 
##  31 kara_woo        3.95 
##  32 CMastication    3.95 
##  33 old_man_chester 3.89 
##  34 juliesquid      3.89 
##  35 edzerpebesma    3.78 
##  36 thomasp85       3.78 
##  37 astroeringrand  3.72 
##  38 kierisi         3.54 
##  39 gdequeiroz      3.44 
##  40 RLadiesBA       3.44 
##  41 theRcast        3.41 
##  42 _RCharlie       3.37 
##  43 nic_crane       3.27 
##  44 cpsievert       3.23 
##  45 cantoflor_87    3.23 
##  46 dvaughan32      3.19 
##  47 nj_tierney      3.19 
##  48 yutannihilation 3.19 
##  49 ajmcoqui        3.16 
##  50 MangoTheCat     3.12 
##  51 taraskaduk      3.12 
##  52 grrrck          3.12 
##  53 RLadiesMVD      3.04 
##  54 jessenleon      3.00 
##  55 jasongrahn      3.00 
##  56 kevin_ushey     2.96 
##  57 shermstats      2.96 
##  58 Bluelion0305    2.92 
##  59 alice_data      2.92 
##  60 ellisvalentiner 2.92 
##  61 jafflerbach     2.87 
##  62 Denironyx       2.87 
##  63 ntweetor        2.87 
##  64 Dorris_Scott    2.83 
##  65 therriaultphd   2.83 
##  66 bhive01         2.83 
##  67 aindap          2.79 
##  68 sellorm         2.69 
##  69 bizScienc       2.69 
##  70 tnederlof       2.69 
##  71 duto_guerra     2.64 
##  72 daattali        2.64 
##  73 patsellers      2.64 
##  74 BaumerBen       2.64 
##  75 conjja          2.64 
##  76 Blair09M        2.59 
##  77 mfairbrocanada  2.59 
##  78 krlmlr          2.59 
##  79 JonathanZadra   2.54 
##  80 alandipert      2.54 
##  81 plzbeemyfriend  2.54 
##  82 NovasTaylor     2.54 
##  83 sgrifter        2.49 
##  84 jamie_jezebel   2.43 
##  85 RLadiesOrlando  2.43 
##  86 ijlyttle        2.43 
##  87 jarvmiller      2.43 
##  88 sheilasaia      2.43 
##  89 SDanielZafar1   2.38 
##  90 ma_salmon       2.38 
##  91 deekareithi     2.38 
##  92 drvnanduri      2.32 
##  93 danielphadley   2.32 
##  94 millerdl        2.32 
##  95 zevross         2.32 
##  96 just_add_data   2.26 
##  97 chrisderv       2.26 
##  98 hugobowne       2.26 
##  99 ibddoctor       2.19 
## 100 MineDogucu      2.19 
## 101 markroepke      2.19 
## 102 pacocuak        2.19 
## 103 hrbrmstr        2.13 
## 104 seankross       2.13 
## 105 jimhester_      2.06 
## 106 dnlmc           2.06 
## 107 jent103         2.06 
## 108 darokun         2.06 
## 109 OHIscience      2.06 
## 110 hspter          1.98 
## 111 DoITBoston      1.98 
## 112 paylakatel      1.98 
## 113 kyrietree       1.98 
## 114 math_dominick   1.90 
## 115 kpivert         1.90 
## 116 wmlandau        1.90 
## 117 tcbanalytics    1.90 
## 118 RhoBott         1.82 
## 119 simecek         1.82 
## 120 jakethomp       1.82 
## 121 egolinko        1.82 
## 122 revodavid       1.82 
## 123 DJShearwater    1.73 
## 124 mmmpork         1.73 
## 125 LuisDVerde      1.73 
## 126 chrisalbon      1.73 
## 127 n_ashutosh      1.73 
## 128 JonTheGeek      1.64 
## 129 nicoleradziwill 1.64 
## 130 OmniaRaouf      1.64 
## 131 R_by_Ryo        1.64 
## 132 JeanetheFalvey  1.64 
## 133 harry_seunghoon 1.64 
## 134 alichiang13     1.64 
## 135 ParmutiaMakui   1.64 
## 136 RLadiesNYC      1.53 
## 137 _NickGolding_   1.53 
## 138 aaronchall      1.53 
## 139 AllenDowney     1.53 
## 140 lariebyrd       1.53 
## 141 jdblischak      1.53 
## 142 maryclaryf      1.53 
## 143 uncmbbtrivia    1.53 
## 144 msciain         1.41 
## 145 samhinshaw      1.41 
## 146 abresler        1.41 
## 147 zymla           1.41 
## 148 TrestleJeff     1.41 
## 149 Nujcharee       1.41 
## 150 jebyrnes        1.28 
## 151 butterflyology  1.28 
## 152 RobynLBall      1.28 
## 153 runnersbyte     1.28 
## 154 tonyfujs        1.28 
## 155 ledell          1.28 
## 156 kwbroman        1.28 
## 157 VParrillaAixela 1.28 
## 158 rweekly_org     1.28 
## 159 ROfficeHours    1.13 
## 160 sgpln           1.13 
## 161 olgavitek       1.13 
## 162 eirenkate       1.13 
## 163 KurggMantra     1.13 
## 164 RLadiesColumbus 1.13 
## 165 claytonyochum   1.13 
## 166 CaltechChemLib  1.13 
## 167 AriLamstein     1.13 
## 168 canoodleson     1.13 
## 169 dobbleobble     1.13 
## 170 bogdanrau       1.13 
## 171 jhollist        1.13 
## 172 RLadiesTC       1.13 
## 173 ucdlevy         0.944
## 174 ukacz           0.944
## 175 jomilo75        0.944
## 176 EricLeeKrantz   0.702
## 177 benjamingreve   0.702
## 178 harrismcgehee   0.702
## 179 javierluraschi  0.702
## 180 volha_tryputsen 0.702
## 181 bj_bloom        0.702
## 182 jaredlander     0.702
## 183 awunderground   0.702
## 184 Md_Harris       0.333
## 185 s_pearce        0.333
```

### Tidyverse vs. Shiny

This code identifies tweets by topic, detecting mentions of the tidyverse \[packages\] and shiny. It then plots the frequency of those tweets over time.

``` r
rt %>%
  filter(created_at > "2018-02-01") %>%
  mutate(
    text = tolower(text),
    tidyverse = str_detect(
      text, "dplyr|tidyeval|tidyverse|rlang|map|purrr|readr|tibble"),
    shiny = str_detect(text, "shiny|dashboard|interactiv")
  ) %>%
  select(created_at, tidyverse:shiny) %>%
  gather(pkg, mention, -created_at) %>%
  mutate(pkg = factor(pkg, labels = c("Shiny", "Tidyverse"))) %>%
  filter(mention) %>%
  group_by(pkg) %>%
  ts_plot("2 hours") +
  geom_point(shape = 21, size = 3, aes(fill = pkg)) + 
  theme_minimal(base_family = "Roboto Condensed") + 
  scale_x_datetime(timezone = "America/Los_Angelos") + 
  theme(legend.position = "bottom",
    legend.title = element_blank(),
    legend.text = element_text(size = rel(1.1)),
    axis.text = element_text(colour = "#222222"),
    plot.title = element_text(size = rel(1.7), face = "bold"),
    plot.subtitle = element_text(size = rel(1.3)),
    plot.caption = element_text(colour = "#444444")) +
  scale_fill_manual(values = c(Tidyverse = "#2244ee", Shiny = "#dd2222")) +
  scale_colour_manual(values = c(Tidyverse = "#001155", Shiny = "#550000")) +
  labs(x = NULL, y = NULL,
    title = "Frequency of tweets about Tidyverse and Shiny during rstudio::conf",
    subtitle = "Tweet counts aggregated for each topic in two-hour intervals",
    caption = "\nSource: Data gathered using rtweet. Made pretty by ggplot2.")
```

<p align="center">
<img width="100%" height="auto" src="README_files/figure-markdown_github/topics-1.png" />
</p>
 

### Word clouds

I didn't want to add a bunch more code, so here I'm sourcing the prep work/code I used to get word lists.

``` r
source(file.path("R", "words.R"))
```

#### Shiny word cloud

This first word cloud depicts the most popular non-stopwords used in tweets about Shiny.

``` r
par(mar = c(0, 0, 0, 0))
wordcloud::wordcloud(
  shiny$var, shiny$n, min.freq = 3,
  random.order = FALSE,
  random.color = FALSE,
  colors = gg_cols(5)
)
```

<p align="center">
<img width="100%" height="auto" src="README_files/figure-markdown_github/shiny-1.png" />
</p>
 

#### Tidyverse word cloud

The second word cloud depicts the most popular non-stopwords used in tweets about the tidyverse.

``` r
par(mar = c(0, 0, 0, 0))
wordcloud::wordcloud(
  tidyverse$var, tidyverse$n, min.freq = 5,
  random.order = FALSE,
  random.color = FALSE,
  colors = gg_cols(5)
)
```

<p align="center">
<img width="100%" height="auto" src="README_files/figure-markdown_github/tidyverse-1.png" />
</p>
