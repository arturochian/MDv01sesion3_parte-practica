---
title: "A tutorial plotting penguins on a map"
author: "David Hood, thoughtfulbloke on GitHub, @thoughtfulnz on Twitter"
output:
  html_document:
    keep_md: yes
    number_sections: yes
    toc: yes
  word_document: default
---



Whenever people ask "what is good data to demonstrate R with?", I reply that they should use a collection of ship's logs data that shows where a ship was at a particular time, and what birds where observed- so there is time, and space, and count data. Along with all the quirks that come with historical data collected by people (the details of which I will leave people to discover for themselves).

As I only ever say "you should use this data", without doing so myself, I thought I would write a tutorial using it for one thing- plotting pictures of penguins on a map. This is mostly to show how you can add any particular images you want to a plot, but talks through the whole process and the thinking around it (which is why I am calling it a tutorial not a demo or walkthrough).

As the data contains time, space, weather, birds, and people, you can build all kinds of tutorials around it. The one major caveat is that, as observations from a ship, it is a convenience sample so should be used for statistical tests expecting independent samples (though there could also be a tutorial of that caveat).

# Have the libraries

As well as using R (from https://cran.r-project.org/) and RStudio Desktop (from https://www.rstudio.com) I am also using some helper libraries. These need to be added to the computer if not already installed, using the `install.packages()` command along with the name of the package. So, for example, if the maps package is not already installed, you need to run the `install.packages("maps")` command.

The packages being used are maps (for a map of New Zealand), readxl (to read in the excel file of the ships log), dplyr (to provide a structure for the tutorial code), stringr(to find which entries are penguins), ggplot2 (to make plots, in this case a map plot), and ggimage (to add an image to a ggplot2 plot).

Once installed, the libraries need to be activated to make the commands in them available in each session you want to use them.


```r
library(maps)
library(readxl)
library(dplyr, quietly=TRUE, warn.conflicts=FALSE)
library(stringr)
library(ggplot2, quietly = TRUE)
library(ggimage)
```

# Get the data

The dataset of at-sea observations of seabirds dating from 1969 to 1990, is held at Te Papa and licensed by Te Papa for re-use under the Creative Commons BY 4.0 International licence. 

https://www.tepapa.govt.nz/learn/research/datasets/sea-observations-seabirds-dataset

The copy we are using comes from the repository data.govt.nz

https://catalogue.data.govt.nz/dataset/at-sea-observations-of-seabirds-1969-to-1990

We are getting the png format penguin image of the Tux logo by Larry Ewing from Wikipedia, licensed with Creative Commons CC0

https://commons.wikimedia.org/wiki/File:Tux.svg

The advantage of png files is that individual pixels can be transparent, allowing images to seem shapes other than rectangles.

We download the files to have local copies while we play with the data and plotting, saving seabirds.xls and tux.png into the folder R is currently paying attention to (the working directory).


```r
# This is a dataset of at-sea observations of seabirds dating from 1969 to 1990.
if(!file.exists("seabirds.xls")){
  download.file("https://catalogue.data.govt.nz/dataset/a99ad31f-9097-43c1-bc21-ee41c687d860/resource/ea3bc86c-5c51-4ffc-bf33-44d034f73251/download/asms_10min_seabird_counts_final.xls", destfile="seabirds.xls", mode="wb")
}

# https://upload.wikimedia.org/wikipedia/commons/thumb/3/35/Tux.svg/204px-Tux.svg.png
# Larry Ewing

if(!file.exists("tux.png")){
  download.file("https://upload.wikimedia.org/wikipedia/commons/thumb/3/35/Tux.svg/204px-Tux.svg.png", destfile="tux.png", mode="wb")
}
```

# Read in the Data

The Excel file has four sheets in it, we read each of them in by name. The "Bird data by record ID" has the added setting guess_max because the initial number of rows read was not enough to correctly guess the column types. One fix would be to set the type of column for all 26 columns, but it was easier just to say "check the full data to work out what is in the columns".


```r
ship_data <- read_excel("seabirds.xls", sheet = "Ship data by record ID")
bird_data <- read_excel("seabirds.xls", sheet = "Bird data by record ID", guess_max=49019)
ship_info <- read_excel("seabirds.xls", sheet = "Ship data codes")
```

```
## New names:
## * `` -> ...2
## * `` -> ...3
```

```r
bird_info <- read_excel("seabirds.xls", sheet = "Bird data codes")
```

```
## New names:
## * `` -> ...2
## * `` -> ...3
```

# Check out the data

Using the glimpse() function, we can examine the structure of the data.


```r
bird_data %>% glimpse()
```

```
## Rows: 49,019
## Columns: 26
## $ RECORD                                                         <dbl> 1, 2...
## $ `RECORD ID`                                                    <dbl> 1083...
## $ `Species common name (taxon [AGE / SEX / PLUMAGE PHASE])`      <chr> "Roy...
## $ `Species  scientific name (taxon [AGE /SEX /  PLUMAGE PHASE])` <chr> "Dio...
## $ `Species abbreviation`                                         <chr> "DIO...
## $ AGE                                                            <chr> NA, ...
## $ WANPLUM                                                        <dbl> NA, ...
## $ PLPHASE                                                        <chr> NA, ...
## $ SEX                                                            <chr> NA, ...
## $ COUNT                                                          <dbl> 6, 2...
## $ NFEED                                                          <dbl> 0, 0...
## $ OCFEED                                                         <chr> "N",...
## $ NSOW                                                           <dbl> 0, 0...
## $ OCSOW                                                          <chr> "N",...
## $ NSOICE                                                         <dbl> 0, 0...
## $ OCSOICE                                                        <chr> "N",...
## $ OCSOSHP                                                        <chr> "N",...
## $ OCINHD                                                         <chr> "N",...
## $ NFLYP                                                          <dbl> 0, 0...
## $ OCFLYP                                                         <chr> "N",...
## $ NACC                                                           <dbl> 6, 2...
## $ OCACC                                                          <chr> "Y",...
## $ NFOLL                                                          <dbl> 0, 0...
## $ OCFOL                                                          <chr> "N",...
## $ OCMOULT                                                        <chr> "U",...
## $ OCNATFED                                                       <chr> "N",...
```

The key columns for this analysis are `Species common name (taxon [AGE / SEX / PLUMAGE PHASE])` (to identify those entries that are penguins) and `RECORD ID` which indicates the matching record in the ship_data information. Because standard column names cannot have things like spaces (and slashes etc) in them, those names are going to need to be referred to with single backticks around them to show "this is all part of the name".

# Find the penguins

Of the 49,019 bird observations, only some are for penguins. To find out which entries we are going to use the str_detect() function, from the stringr package, by finding which common name entries contain the text "penguin" regardless of the case of the letters. Using this information we filter(), from the dplyr package, to keep only the entries we are interested in.

To cut the size of the glimpsed output, we are also selecting, using select() from the dplyr package, only the `Species common name (taxon [AGE / SEX / PLUMAGE PHASE])` and `RECORD ID` columns.


```r
penguins <- bird_data %>%
  select(`Species common name (taxon [AGE / SEX / PLUMAGE PHASE])`, `RECORD ID`) %>%
  filter(str_detect(`Species common name (taxon [AGE / SEX / PLUMAGE PHASE])`,
                    fixed("penguin", ignore_case = TRUE))) %>% glimpse()
```

```
## Rows: 70
## Columns: 2
## $ `Species common name (taxon [AGE / SEX / PLUMAGE PHASE])` <chr> "Little p...
## $ `RECORD ID`                                               <dbl> 2112026, ...
```

Based on looking for "penguin", we can see 70 occasions on which 1 or more penguins were seen.

# Join the peguin information to the ship information

Taking the peguin entries and joining on the ships information, using inner_join() from the dplyr package thought matching Record IDs, gives the ships details at the time the penguins were sighted.

The strategy of reducing the bird_data to just the penguins before matching it to the ship_data was a deliberate choice, as it reduces the amount of work that the join step needs to do since there are not as many entries to join.

For this tutorial, we only want the latitude and longitude of the sightings, so select just those columns.


```r
penguins <- bird_data %>%
  filter(str_detect(`Species common name (taxon [AGE / SEX / PLUMAGE PHASE])`,
                    fixed("penguin", ignore_case = TRUE))) %>% 
  inner_join(ship_data, by="RECORD ID") %>%
  select(LAT, LONG) %>% glimpse()
```

```
## Rows: 70
## Columns: 2
## $ LAT  <dbl> -35.00000, -65.00000, -65.00000, -37.00000, -38.00000, -39.000...
## $ LONG <dbl> 174.0000, 110.0000, 109.0000, 150.0000, 141.0000, 143.0000, 10...
```

That is the data of penguin locations prepared.

# Make a map of New Zealand

To make the map of New Zealand, we get the shape and location of New Zealand from the maps package using the map_data() command.

We plot the map shapes using ggplot() from the ggplot2 package, draw the shapes with the geom_polygon(). The aes() expresses the aesthetics of how aspects of the plot are controlled by things in the declared N.Z. data.

The coord_quickmap() function is a quick way of locking the latitude and longitude onto the map to stop stretching. As New Zealand fits well to a large set of islands vertical rectangle, coord_quickmap works pretty well for making the map. For other outline maps a different way of setting coord is needed.


```r
nz <- map_data("nz")
ggplot() + 
  geom_polygon(data = nz, aes(x=long, y = lat, group = group), fill = "darkgrey", color = "darkgrey") + 
  coord_quickmap()
```

![](tute_files/figure-html/unnamed-chunk-1-1.png)<!-- -->

# Put the penguins on the map

To put the penguins on the map, we use geom_image() from the ggimage package, which is designed to add image files to ggplot based figures.

The aes() for geom_image contains the latitude and longitude to plot each seperate sighting at. The image setting with the filename is outside the aes since we are plotting the same image for each point rather than providing the name of the image file in the data for each point.


```r
ggplot() + 
  geom_polygon(data = nz, aes(x=long, y = lat, group = group), fill = "darkgrey", color = "darkgrey") + 
  coord_quickmap() + 
  geom_image(data=penguins, aes(x=LONG, y=LAT), image="tux.png", size=.02)
```

![](tute_files/figure-html/mapenguin-1.png)<!-- -->

That went horribly wrong.

The main thing that went wrong is a mismatch between what we thought the data contained, and what the data actually contained- while there are a lot of New Zealand sightings, there are not only New Zealand sightings.

A second, minor, problem is that coord_quickmap does a poor job encompassing the extent of the Southern Oceans down to Antartica and seems to be cascading into stretching the images a bit.

One way of fixing this would be changing our goal, picking a world map, and adjusting the coordinate system. But for this tutorial, we will fix things by cutting the data down to that which directly goes to our goal.

# Make it penguins around New Zealand

To restrict the data to penguins near to New Zealand, we will filter() the data to find only those penguin entries at longitude greater than 160


```r
penguins <- bird_data %>%
  filter(str_detect(`Species common name (taxon [AGE / SEX / PLUMAGE PHASE])`,
                    fixed("penguin", ignore_case = TRUE))) %>% 
  inner_join(ship_data, by="RECORD ID") %>%
  filter(LONG > 160) %>%
  select(LAT, LONG) 

ggplot() + 
  geom_polygon(data = nz, aes(x=long, y = lat, group = group), fill = "darkgrey", color = "darkgrey") + 
  coord_quickmap() + 
  geom_image(data=penguins, aes(x=LONG, y=LAT), image="tux.png", size=.05)
```

![](tute_files/figure-html/mapenuin2-1.png)<!-- -->


Finally, we do the annotations to give credit where credit is due, and make it prettier.

In adjusting colours, we are not only using named colours, we are setting colour with #RRGGBB (where RR is the amount of red, GG is the amount of green, and BB is the amount of blue).


```r
ggplot() + 
  geom_polygon(data = nz, aes(x=long, y = lat, group = group), fill = "darkgrey", color = "#888888") + 
  coord_quickmap() + 
  geom_image(data=penguins, aes(x=LONG, y=LAT), image="tux.png", size=.05) +
  ggtitle("Penguin sightings near New Zealand", subtitle="From logbooks of Captain Jenkins, data sourced from Te Papa") +
  xlab("Longitude") + ylab("Latitude") + annotate("text",x=175, y=-47, label="Tux png by Larry Ewing", size=2.5) +
  theme(panel.background = element_rect(fill = '#A5C7E9'))
```

![](tute_files/figure-html/finalmap-1.png)<!-- -->

# Acknowledgements

In using R (version 3.3.3)

R Core Team (2017). R: A language and environment for statistical computing. R Foundation for Statistical Computing, Vienna, Austria. URL https://www.R-project.org/

it would have been a lot more work and more code written without the various helper libraries used:

Original S code by Richard A. Becker, Allan R. Wilks. R version by Ray Brownrigg. Enhancements by Thomas P Minka and Alex Deckmyn. (2016). maps: Draw Geographical Maps. R package version 3.1.1. https://CRAN.R-project.org/package=maps

Hadley Wickham and Jennifer Bryan. readxl: Read Excel Files. http://readxl.tidyverse.org, https://github.com/tidyverse/readxl
  
Hadley Wickham, Romain Francois, Lionel Henry and Kirill Müller (2017). dplyr: A Grammar of Data Manipulation. R package version 0.7.4. https://CRAN.R-project.org/package=dplyr

Hadley Wickham (2017). stringr: Simple, Consistent Wrappers for Common String Operations. R package version 1.2.0. https://CRAN.R-project.org/package=stringr

H. Wickham. ggplot2: Elegant Graphics for Data Analysis. Springer-Verlag New York, 2016

Guangchuang Yu (2017). ggimage: Use Image in 'ggplot2'. R package version 0.1.0. https://CRAN.R-project.org/package=ggimage


