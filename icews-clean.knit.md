---
title: "Read and Clean ICEWS Coded Event Data"
author: "Brendan Cooley"
date: \today
bibliography: "/Users/bcooley/Dropbox/References/library.bib"
output: html_document
---



*Last updated 11 April 2018*

Datasets that researchers measure conflict in international relations are generally coarse. The [Correlates of War (COW) Project's](http://www.correlatesofwar.org/) [Militarized Interstate Dispute](http://www.correlatesofwar.org/data-sets/MIDs) data records threats, displays, and uses of military force between 1816 and 2010 and COW's [interstate war](http://cow.dss.ucdavis.edu/data-sets/COW-war) data records conflicts which resulted in at least 1,000 battle deaths. Both MIDs and wars are exceedingly unusual. Yet the "stuff" of international relations happens every day. Governments are bargaining and communicating all the time -- sometimes cooperatively and sometimes conflictually. These interactions almost certaintly contain information about their proclivity to experience armed conflict. New data might help us measure and understand this "stuff" better. 

In recent years, several very-large-n (>1,000,000 observation) dyadic event datasets have become available for public use. An "event" takes the form of "[actor x] undertook [action z] toward [actor y] on [date w]. Natural language processors scrape newswires and map events into preexisting event and actor ontologies. The [Integrated Crisis Early Warning System (ICEWS)](https://dataverse.harvard.edu/dataverse/icews) is one such dataset. You can find a nice discussion of the project's history by Phil Shrodt [here](https://asecondmouse.wordpress.com/2015/03/30/seven-observations-on-the-newly-released-icews-data/). [Andreas Beger](https://andybeger.com/2015/04/08/public-icews-data/) and [David Masad](http://nbviewer.jupyter.org/gist/dmasad/f79ce5abfd4fb61d253b) have nice writeups on what the data look like. It's still pretty rare to see these data used in political science, however. See @Gallop2016, @Minhas2016, @Roberts2017 for notable exceptions.^[Please feel free point out others I'm missing!]

This may be because it's still a little tricky to get these data into a format suitable for empirical analyses. Having struggled myself to clean ICEWS, I figured it'd be worth sharing my experience (working in R). I show three steps in the process here:

1. Grabbing the data from dataverse
2. Converting it 'reduced' form with conflict cooperation scores and COW codes, employing Phil Shrodt's [software](https://github.com/openeventdata/text_to_CAMEO/)
3. Converting the 'reduced' data into date-dyad counts

As always, feel free to send along questions or comments or point out mistakes. That's the point of open research. You can find all the software supporting this [here](https://github.com/brendancooley/icews-clean).

First, get the environment setup

```r
packages <- c('dataverse', 'dplyr', 'zoo', 'lubridate', 'tidyr', 'bibtex', 'knitcitations')
lapply(packages, require, character.only = TRUE)

write.bib(packages)
bib <- read.bib('Rpackages.bib')
```

Then, grab the data straight from Harvard's dataverse

```r
Sys.setenv("DATAVERSE_SERVER" = "dataverse.harvard.edu")
doi <- "doi:10.7910/DVN/28075"
dv <- get_dataset(doi)
```

We can take a look at the files included in the dataverse repository

```r
dvFiles <- dv$files$label
dvFiles
```

```
##  [1] "BBN ACCENT Event Coding Evaluation.updated v01.pdf"
##  [2] "CAMEO.CDB.09b5.pdf"                                
##  [3] "changes.txt"                                       
##  [4] "events.1995.20150313082510.tab.zip"                
##  [5] "events.1996.20150313082528.tab.zip"                
##  [6] "events.1997.20150313082554.tab.zip"                
##  [7] "events.1998.20150313082622.tab.zip"                
##  [8] "events.1999.20150313082705.tab.zip"                
##  [9] "events.2000.20150313082808.tab.zip"                
## [10] "events.2001.20150313082922.tab.zip"                
## [11] "events.2002.20150313083053.tab.zip"                
## [12] "events.2003.20150313083228.tab.zip"                
## [13] "events.2004.20150313083407.tab.zip"                
## [14] "events.2005.20150313083555.tab.zip"                
## [15] "events.2006.20150313083752.tab.zip"                
## [16] "events.2007.20150313083959.tab.zip"                
## [17] "events.2008.20150313084156.tab.zip"                
## [18] "events.2009.20150313084349.tab.zip"                
## [19] "events.2010.20150313084533.tab.zip"                
## [20] "events.2011.20150313084656.tab.zip"                
## [21] "events.2012.20150313084811.tab.zip"                
## [22] "events.2013.20150313084929.tab.zip"                
## [23] "events.2014.20160121105408.tab"                    
## [24] "events.2015.20170206133646.tab"                    
## [25] "events.2016.20180122103653.tab"                    
## [26] "events.2017.20180122111453.tab"                    
## [27] "ICEWS Coded Event Data Read Me.pdf"                
## [28] "ICEWS Events and Aggregations.pdf"                 
## [29] "ICEWS Expanded CAMEO Annotation Guidelines.pdf"
```

We want to get the files prefixed with 'events.' We'll unzip them and put them in a folder called /rawICEWS.

```r
dir.create('rawICEWS', showWarnings = FALSE)

# search through all files in dataverse repo
for (i in dvFiles) {
  # for those that start with events
  if (substr(i, 1, 6) == 'events') {
    dest <- paste0('rawICEWS/', i)
    writeBin(get_file(i, doi), dest)
    # unzip those that are compressed
    if (substr(i, nchar(i) - 2, nchar(i)) == 'zip') {
      unzip(dest, exdir='rawICEWS/')
      file.remove(dest)  # trash zipfile
    }
  }
}

# store list of files in .txt
fNames <- paste0('rawICEWS/', list.files('rawICEWS'))
lapply(fNames, write, 'fNames.txt', append=TRUE)
```

This will take a minute or two. Go get some coffee. Alternatively, you can download the raw data (as of April 2018) from my [Dropbox](https://www.dropbox.com/sh/94coknnsbxjds7b/AABMEsxH6jVlDLRygQn0xRtha?dl=0) and plop it into your /rawICEWS folder in your working directory. The nice thing about this code is that it should dynamically grab new data as the ICEWS project uploads it to dataverse.

Either way, now we have all of our raw data ready to go sitting in /rawICEWS. The raw ICEWS data is clunky on several dimensions. Phil Shrodt provides software to get it into a format that looks recognizable to empirical international relations researchers. If you're interested in the machinery, check it out [here](https://github.com/openeventdata/text_to_CAMEO).

For our purposes, we just need the script `text_to_CAMEO.py` and the ontology files `agentnames.txt` and `countrynames.txt`. It'll take the list of filenames and convert them into "reduced" form. Just navigate to the current working directory and run
```
python text_to_CAMEO.py -c -t fNames.txt
```

After this you should have a bunch of .txt files sitting in your working directory. I moved them over to a /reducedICEWS folder and you can find them [here](https://www.dropbox.com/sh/9h9g5944f4fd606/AADBmYvKVUCXbO7GoPZwEweOa?dl=0). Now we can load these into memory and get to work. 

```r
# helper to replace empty cells with NAs
empty_as_na <- function(x) {
  ifelse(as.character(x)!="", x, NA)
}

reducedFiles <- list.files("reducedICEWS")
events.Y <- list()  # list holding data frames for each year
# for each of the reduced files
for (i in 1:length(reducedFiles)) {
  # append to list
  events.Y[[i]] <- read.delim(paste('reducedICEWS/', reducedFiles[i], sep=""), header = F)
  # convert column names
  colnames(events.Y[[i]]) <- c("date", "sourceName", "sourceCOW", "sourceSec",
                               "tarName", "tarCOW", "tarSec", "CAMEO", "Goldstein", "quad")
  # replace empty cells with NAs
  events.Y[[i]] %>% mutate_if(is.factor, as.character) %>% mutate_all(funs(empty_as_na)) %>% as_tibble() -> events.Y[[i]]
}
# bind everything together
events <- bind_rows(events.Y)

# add year and month fields
events$month <- as.yearmon(events$date)
events$year <- year(events$date)
events <- events %>% select(date, year, month, everything())
```

This will take a little while too (this is BIG data, people!). But once it's done, we can take a look at what we've got:

```r
head(events)
```

```
## # A tibble: 6 x 12
##   date    year month  sourceName sourceCOW sourceSec tarName tarCOW tarSec
##   <chr>  <dbl> <S3: > <chr>          <int> <chr>     <chr>    <int> <chr> 
## 1 1995-… 1995. Jan 1… RUS              365 REB       RUS        365 GOV   
## 2 1995-… 1995. Jan 1… BIH              346 GOV       SRB        345 CVL   
## 3 1995-… 1995. Jan 1… SRB              345 CVL       BIH        346 GOV   
## 4 1995-… 1995. Jan 1… CAN               20 OTH       CAN         20 GOV   
## 5 1995-… 1995. Jan 1… CAN               20 JUD       CAN         20 GOV   
## 6 1995-… 1995. Jan 1… RUS              365 GOV       RUS        365 OTH   
## # ... with 3 more variables: CAMEO <int>, Goldstein <dbl>, quad <int>
```

Now we have COW-coded countries (sourceCOW, tarCOW), the date actors within these countries interacted, and the sector identity of the actors (the data includes government-government interactions but also interactions between subnational actors). The CAMEO/Goldstein/quad fields describe the nature of the interaction under different event categorization systems. See the Shrodt documentation for more detail on the event and sector classifications. The quad score (ranging from 1-4) classifies events as either conflictual or cooperative and as verbal or material in nature. A score of 1 corresponds to verbal cooperation. So the first event can be read as follows: "On Jan. 1, 1995, rebels in Russia verbally cooperated with the government in Russia." The CAMEO scores provide a much richer event classification.

The first question we almost always want to answer -- how many times did each pair of countries experience each type of interaction with each other? The following function returns the directed dyadic matrix of counts, aggregated by 'year' or 'month'.

```r
event.counts <- function(events, agg.date=c('month', 'year'), code=c('quad', 'CAMEO')) {
  counts <- events %>%
    group_by_(agg.date, 'sourceCOW', 'tarCOW', code) %>%
    summarise(n = n()) %>%
    ungroup()  # this seems trivial but screws up a lot of stuff if you don't do it
  output <- spread_(counts, code, 'n')
  output[is.na(output)] <- 0
  return(output)
}

counts <- event.counts(events, 'year', 'quad')
head(counts)
```

```
## # A tibble: 6 x 7
##    year sourceCOW tarCOW   `1`   `2`   `3`   `4`
##   <dbl>     <dbl>  <dbl> <dbl> <dbl> <dbl> <dbl>
## 1 1995.        0.     0. 1440.   60.  357.  259.
## 2 1995.        0.     2.  587.   19.  123.   14.
## 3 1995.        0.    20.   86.    1.   10.    1.
## 4 1995.        0.    31.    1.    0.    0.    0.
## 5 1995.        0.    40.   69.    2.    1.    2.
## 6 1995.        0.    41.   25.    8.    3.    3.
```

COW code 0 is international organizations (IOs), so we can get a sense of how many times IOs are interacting with themselves and other countries in the data from the top of this table. And now that we have the count data we can start doing fun stuff. Over [here](https://github.com/brendancooley/icews-explorer) I've built an app that allows users to scale subsets of the data, following [Lowe](http://dl.conjugateprior.org/preprints/mmfed.pdf).^[This is still in-progress, I've been meaning to go back and update it.] The procedures implemented there allow us to recover what can be thought of as a conflict-cooperation "score" for each dyad-year, based on the underlying count data.

## R Packages

Leeper TJ (2017). _dataverse: R Client for Dataverse 4_. R package
version 0.2.0.

Wickham H, Francois R, Henry L and Müller K (2017). _dplyr: A
Grammar of Data Manipulation_. R package version 0.7.4, <URL:
https://CRAN.R-project.org/package=dplyr>.

Zeileis A and Grothendieck G (2005). "zoo: S3 Infrastructure for
Regular and Irregular Time Series." _Journal of Statistical
Software_, *14*(6), pp. 1-27. doi: 10.18637/jss.v014.i06 (URL:
http://doi.org/10.18637/jss.v014.i06).

Grolemund G and Wickham H (2011). "Dates and Times Made Easy with
lubridate." _Journal of Statistical Software_, *40*(3), pp. 1-25.
<URL: http://www.jstatsoft.org/v40/i03/>.

Wickham H and Henry L (2018). _tidyr: Easily Tidy Data with
'spread()' and 'gather()' Functions_. R package version 0.8.0,
<URL: https://CRAN.R-project.org/package=tidyr>.

Francois R (2017). _bibtex: Bibtex Parser_. R package version
0.4.2, <URL: https://CRAN.R-project.org/package=bibtex>.

Boettiger C (2017). _knitcitations: Citations for 'Knitr' Markdown
Files_. R package version 1.0.8, <URL:
https://CRAN.R-project.org/package=knitcitations>.

## References
