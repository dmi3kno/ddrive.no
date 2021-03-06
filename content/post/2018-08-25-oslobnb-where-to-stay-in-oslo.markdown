---
title: 'OsloBnB: Where to stay in Oslo?'
author: dmi3kno
date: '2018-08-25'
slug: oslobnb-where-to-stay-in-oslo
categories:
  - blog
tags:
  - r
  - polite
---




```r
library(here)
library(tidyverse)
library(polite)
library(rvest)
library(mapview)
library(sf)
library(raster)
library(fasterize)
library(RPostgreSQL)
library(rpostgis)
library(mapsapi)
library(patchwork)
library(ggmap)
library(ggrepel)
library(plotly)
library(lubridate)
library(hrbrthemes)
```

## Challenge

A friend of mine is coming over to Oslo in couple of weeks time and he asked me if I could recommend a good place for him to stay in Oslo. Nothing popped to my mind, so I answered "any place downtown is good". But is there a way that we could answer this question with data?

My friend is coming in time to visit the meetup, which is usually held in Technology House (Teknologihuset) downtown. I also plan to show him my office, located in Fornebu area, just outside of the city border. We also plan to visit University of Oslo (UiO) and Norwegian Business School (BI). Since these locations are spread out around the city, I would like to recommend him a place to stay where he could find himself in relative proximity to these key locations, but also have enough life and entertainment opportunities around to keep him busy while I am at work ;)

## Data

We will use `polite` package to download [data scraped from AirBNB](insideairbnb.com). First of all we will to scrape the page listing the datasets, to see which files are available. We accomplish it with `bow` followed by `scrape`. The former fetches `robots.txt` and establishes an html session with the server, while the latter performs rate-limited memoised acquistion of the page content. We then can parse the response object with regular functions in `rvest`. 

The [table](http://insideairbnb.com/get-the-data.html) listing the file names is a little special because it contains a column with hyperlink. We want to make sure to capture both the text and the url it points to. `polite` package has a function (`html_attrs_dfr`) for parsing an html node together with all of its attributes. I recommend checking it out, if you need to download both text and all attributes of any particular page element!



```r
page <- polite::bow("http://insideairbnb.com/get-the-data.html") %>% 
  polite::scrape() %>% 
  rvest::html_nodes(".table.table-hover.table-striped.oslo")

links_df <- dplyr::bind_cols(
  page %>% 
    rvest::html_table(),
  page %>% 
    rvest::html_nodes("tbody tr td a") %>% 
    polite::html_attrs_dfr()
)
```

Once we got our hands on the download links, we can send them one by one through `rip` function which performs `polite` downloading. Note, that files will be saved to the directory we indicate in the "path" argument. We are only interested in the last scraping batch (dated 2018-08-31). `rip` is part of `polite` package and follows it philosoply. It never asks for the same information twice, so it will check if the file we are requestion have been already downloaded and has `overwrite` argument set to `FALSE` by default.


```r
links_df %>% 
  filter(stringr::str_detect(href, "2018-07-31")) %>% 
  pull(href) %>% 
  walk(~bow(.x) %>% rip(path="input"))
```

## Data import and preparation

We will import and prepare a few more datasets. Lets start by definiing places of interest and geocoding them. We will turn this data frame into simple features object using `st_as_sf()`.


```r

places_df <- tibble::tribble(
  ~place, ~lat, ~lon, ~url, 
  "Work", 59.895720, 10.629540, "https://media-cdn.tripadvisor.com/media/photo-s/0b/ee/27/9e/p-20160711-045718-hdr.jpg",
  "UiO", 59.940130, 10.720290,  "https://www.uio.no/english/studies/why-choose-uio/bilder/gsh-970.jpg",
  "BI", 59.948872, 10.768210, "https://nielstorp.no/wp-content/uploads/2015/01/BI.Nydalen.21-480x270.jpg",
  "Meetup", 59.923450, 10.731790, "https://img.gfx.no/1845/1845622/DSC05635.jpg")

places_sf <- places_df %>% 
    st_as_sf(coords=c("lon", "lat"), crs=4326)
```

We will then need data about the city neighbourhoods. Luckily, *insideairbnb.com* hosts geojson with neighbourhood outlines, so we will take that, correct spelling of certain neghbourhoods and enrich the polygons with some data about the listings in each of the neigborhoods. 

In order to do it, we will import AirBnB listings, convert character columns to numeric values and drop records corresponding to "shared rooms", which, I know, would be not of great interest for my friends. We will also look at reasonable prices (under $1000 per night) and accomodations with less than 10 sleeping places. It would be quite lonely for him to wonder around a place, which looks like an empty guesthouse. We will use these observations to create a dataset aggregated to neighbourhood. We want to calculate median price per neighbourhood, number of listings, as well as approximate polygon centroids, which can be later used for placing labels. 


```r
listings_ext_df <- read_csv(here::here("input", "listings.csv.gz")) %>%  # 96 cols
    mutate_at(vars(contains("price")), funs(str_remove_all(., "\\$|,"))) %>% 
    mutate_at(vars(contains("price")), funs(as.numeric))

borough_data <- listings_ext_df %>%
  filter(room_type!="Shared room",
         price<=1e4, beds<=10) %>% 
  group_by(neighbourhood_cleansed) %>% 
  nest() %>% 
  mutate(med_price=map_dbl(data, ~median(.x$price)),
         listing_count=map_int(data, nrow),
         #price_bed_plot=map(data, plot_prices),
         cent_lon=map_dbl(data, ~median(.$longitude)),
         cent_lat=map_dbl(data, ~median(.$latitude)))

oslo_boroughs_sf <- geojsonio::geojson_read(here::here("input", "neighbourhoods.geojson"),
                                            what="sp", stringsAsFactors=FALSE) %>% 
  st_as_sf() %>% 
    mutate(neighbourhood=case_when(
    str_detect(neighbourhood, "^Gr.+kka$") ~ "Grünerløkka",
    str_detect(neighbourhood, "ndre Nordstrand$") ~ "Søndre Nordstrand",
    str_detect(neighbourhood, "stensj") ~ "Østensjø",
    TRUE ~ neighbourhood
  )) %>% left_join(borough_data, by=c("neighbourhood"="neighbourhood_cleansed")) %>% 
  filter(listing_count>20)
```



Let's have a look at the map of Oslo. We will ass points of interest and color the neighbourhoods by listing count. It seems like most of the accomodations are located in areas surrounting the city center: Grunerløkka, Old (Gamle) Oslo, and Frogner. Center of Oslo has relatively few apartments for rent, possibly because most of it consists of administrative buildings and retail outlets. 


```r
osl_grid_map <- get_stamenmap(bbox = as.numeric(st_bbox(oslo_boroughs_sf)), zoom = 12, maptype = "toner-lite")

ggmap(osl_grid_map)+
  geom_sf(data=oslo_boroughs_sf, aes(fill=listing_count), inherit.aes = FALSE, alpha=0.5)+
  geom_text_repel(data=oslo_boroughs_sf, aes(x=cent_lon, y=cent_lat, label=neighbourhood), inherit.aes = FALSE, size=3, color="grey30")+
  geom_point(data=places_df, aes(x=lon, y=lat), inherit.aes = FALSE, size=3)+
  geom_label_repel(data=places_df, aes(x=lon, y=lat, label=place), color="white", fill="grey40", inherit.aes = FALSE, size=3)+
  scale_fill_viridis_c(option = "B")+
  theme_nothing()
```

<img src="/post/2018-08-25-oslobnb-where-to-stay-in-oslo_files/figure-html/unnamed-chunk-6-1.png" width="576" />

Price-wise, average two-bed accomondation costs a little less than 1000 NOK (close to $100) per night. A lot of cheaper accomodation options are actually single rooms and not separate accomodations. We shall target 2-3 bed accomodations, which, as it seems, still possible to get under 1000 NOK/night.


```r
listings_ext_df %>% 
    ggplot(aes(x=beds, y=price))+
    geom_jitter(aes(color=room_type), alpha = 0.5)+
    geom_smooth()+
    scale_y_continuous(limits = c(100,1e4), trans="log10")+
    scale_x_continuous(breaks = seq.int(0,10, by=2), limits = c(0,10))+
    scale_color_viridis_d(option = "D")+
    theme_ipsum_rc(grid_col = "gray90")+
    theme(legend.position = "bottom")+
  labs(x="Number of beds",
       y="Price per night, NOK",
       color="Accomodation type")
```

<img src="/post/2018-08-25-oslobnb-where-to-stay-in-oslo_files/figure-html/unnamed-chunk-7-1.png" width="576" />

## Distances

What could be the best place to stay in the city if one intends to visit each one of these four locations? Let's assume that my friend would make trips from his residence early in the morning. We can create a grid over the city and then calculate travel time from each grid cell to each point of interest using Google Maps services. We will keep only those cells that contain listings, thus dropping remote and unpopulated areas. Google Maps returns distances and travel times for four different means of transportation: walking, bicycling, driving and public transit. We will save the retrieved data to avoid requesting this quite extensive (and expensive) data.


```r
  
listings_ext_sf <- listings_ext_df %>% 
  st_as_sf(coords=c("longitude", "latitude"), crs=4326)

listing_grid_sf <- listings_ext_sf %>% 
  st_make_grid(n=c(15,20), crs = 4326) %>%
  st_sf() %>% mutate(grid_id=1:n())

cells_to_keep <- sapply(st_intersects(listing_grid_sf, listings_ext_sf), function(x) length(x)>0)
#> although coordinates are longitude/latitude, st_intersects assumes that they are planar

listing_grid_centroids_sf <- listing_grid_sf[cells_to_keep,] %>% 
  st_transform(crs=32632) %>% 
  st_centroid() %>% 
  st_transform(crs=4326) %>% 
  mutate(fold=paste0("fold_",1+seq(n())%/%25))
#> Warning in st_centroid.sf(.): st_centroid assumes attributes are constant
#> over geometries of x

get_gdist <- function(fold_id, mode, org, dst){

  Sys.sleep(1)
  fold_sf <- org %>% filter(fold==fold_id)
  
  stopifnot(nrow(fold_sf)<=25)
  stopifnot(nrow(fold_sf)*nrow(dst)<=100)
  
  gdist_obj <- mapsapi::mp_matrix(origins = st_coordinates(fold_sf),
                   destinations = st_coordinates(dst), 
                   departure_time = as.POSIXct("2018-09-05 08:00:00"),
                   mode = mode, 
                   key = Sys.getenv("GOOGLE_MAPS_API_KEY")) 
  
  stopifnot(xml2::xml_text(xml2::xml_find_all(gdist_obj, xpath="./status"))=="OK")
  
  gdist_matrix <- gdist_obj %>% 
    mp_get_matrix(value="duration_s")
  
  colnames(gdist_matrix) <- dst$place
  gdist_matrix %>% 
    as.tibble() %>% 
    mutate(grid_id=fold_sf$grid_id,
           mode=mode)
}

if(!file.exists(here::here("input", "grid_distances.rds"))){
  grid_distances <- crossing(fold_id=unique(listing_grid_centroids_sf$fold), 
                        mode=c("driving", "transit", "walking", "bicycling")) %>% 
    pmap_dfr(get_gdist, org=listing_grid_centroids_sf, dst=places_sf)
  
  write_rds(grid_distances, here::here("input", "grid_distances.rds"))
} 
```

Lets plot distances to points of interest using heatmap. each of the cells corresponds to the area of the city, from which travel time has been measured (using grid cell centroid). Light color corresponds to shorter travel time. As expected bicycling and walking produces pretty smooth color pattern, because very few boundaries hinder the traveller from going straight towards the point of interest. Color patterns on the driving map are a little more fragmented due to major highways and tunnels running through city. The most interesting map is related to public transit. It is most likely, however, that my friend will be using public transportation, as taxi is a luxury service in Oslo, Uber is banned and weather is sometimes a little unpredictable (for bicycling or walking). Public transit shows quite interesting patterns. Let's look at it a little closer.

Note that the `fill` scale varies from panel to panel. This is achieved through individual plotting of each mode of transportation and subsequent arranging of final plot with awesome `patchwork` package. I am using subtitles in the original plots to annotate panels (yes, I am aware of the annotation possibilities of `patchwork`).


```r
grid_distances <- read_rds(here::here("input", "grid_distances.rds"))


grid_df <- listing_grid_sf[cells_to_keep,] %>% 
  left_join(grid_distances, by = "grid_id") %>%
  gather(key=key, value=value, -grid_id, -geometry, -mode) %>% 
  mutate(value=value/60)

plot_grid_df <- function(df){

  ggmap::ggmap(osl_grid_map)+
    geom_sf(data= df, aes(fill=value), inherit.aes = FALSE, color="transparent", alpha=0.8)+
    scale_fill_viridis_c(option = "B", direction = -1)+
    theme_bw(base_family = "Roboto Condensed")+
    coord_sf(ndiscr = 2)+
    facet_wrap(~key, nrow = 1) +
    labs(x=NULL, y=NULL, subtitle=first(df$mode))+
    theme(axis.text.x = element_blank())
}

grid_df %>% 
  split(.$mode) %>% 
  map(plot_grid_df) %>% 
  reduce(`+`)+
  patchwork::plot_layout(ncol=1, guides = "collect")
```

<img src="/post/2018-08-25-oslobnb-where-to-stay-in-oslo_files/figure-html/unnamed-chunk-9-1.png" width="576" />

We are going to assume that median travel time to the points of interest using public transit is a good proxy for convenience of location. We are going to look closer to the grid cells and plot median travel times from corresponding square blocks of the city against median price of listings in those blocks. The size of the bubble will correspond to relative number of listings. Lets annotate cells with less than 20 min median travel time.

It looks like it there's a tradeoff of living closer vs paying more (as expected). However the variance is quite big and it should be possible to find acceptable accomodation within the grid cells that are in convenient proximity to the places we plan to visit.


```r
grid_time_df <- grid_df %>% 
  st_set_geometry(NULL) %>% 
  filter(mode=="transit") %>% 
  group_by(grid_id) %>% 
  summarise(mean_time=mean(value), 
            median_time=median(value), 
            min_time=min(value),
            max_time=max(value))

#glimpse(listings_ext_sf)

listings_grid_df <- listing_grid_sf[cells_to_keep,] %>% 
  st_join(listings_ext_sf) %>% 
  st_set_geometry(NULL) %>% 
  filter(beds<=4) %>% 
  group_by(grid_id) %>% 
  summarise(borough=first(neighbourhood_cleansed),
            median_price=median(price), 
            mean_price=median(price), 
            n_list=n()) %>% 
  left_join(grid_time_df, by="grid_id")
  
  
p <- listings_grid_df %>% 
  ggplot(aes(x=median_time, y=median_price))+
  geom_point(aes(color=borough, size=n_list), show.legend = FALSE) +
  geom_smooth(method=lm)+
  geom_label_repel(data=. %>% filter(median_time<20), 
                  aes(label=borough),
                  family="Roboto Condensed", size=3)+
  theme_ipsum_rc(grid_col = "grey90")+
  scale_color_viridis_d(option="B", end=0.8) +
  scale_y_continuous(limits = c(0,1250))+
  labs(x="Median time to points of interest, min",
       y="Median price per night, NOK",
       color="Neighborhood",
       size="Number of listings")

p
```

<img src="/post/2018-08-25-oslobnb-where-to-stay-in-oslo_files/figure-html/unnamed-chunk-10-1.png" width="576" />

```r
#ggplotly(p)
```

There are total of 7 cells with median travel time of under 20 minutes. Lets bring it back to the map and plot these cells together with points representing individual accomodations. Best place to live is in Blindern/Ullevål area, except there are very few options available in that area. Median travel time is heavily affected by proximity to metro and Ring 3 (highway running around the city). Second best place (with mean travel time of under 20 min) is close to Oslo S. Third location is close to Fagerborg, north of Meetup location. It seems like we've got some neighbourhood recommendations for my friend!

Note that we had to define a 100m "inner" buffer area to make sure grid cell fills are not covering each other. Thanks [@StatnMap](https://statnmap.com/2017-08-10-polygons-tint-band-with-leaflet-and-simple-feature-library-sf/) for this tip!


```r

selected_grids_sf <- listing_grid_sf[cells_to_keep,] %>% 
  right_join(listings_grid_df, by="grid_id") %>% 
  filter(median_time<20) %>% 
  st_transform(crs=32632)

selected_grids_buf_sf <- st_cast(st_buffer(selected_grids_sf, dist = -100)) %>% 
  st_combine() %>% st_sf()

selected_grids_diff_sf <- st_difference(selected_grids_sf, selected_grids_buf_sf) %>% 
  st_cast() %>% st_transform(crs=4326)


osl_map <- get_stamenmap(bbox = as.numeric(st_bbox(places_sf)), zoom = 13, maptype = "toner-lite")

ggmap::ggmap(osl_map)+
  geom_point(data=listings_ext_df, aes(x=longitude, y=latitude), color="darkslateblue", alpha=0.1)+
  geom_sf(data=selected_grids_diff_sf, aes(fill=mean_time),color="transparent", inherit.aes = FALSE)+
  geom_point(data=places_df, aes(x=lon, y=lat), inherit.aes = FALSE, color="violetred", size=5)+
  geom_label_repel(data=places_df, aes(x=lon, y=lat, label=place), color="violetred", inherit.aes = FALSE, size=3)+
  scale_fill_viridis_c(option = "B", direction=-1)+
  theme_minimal(base_family = "Roboto Condensed")+
  labs(x=NULL, y=NULL,
       color="Travel time, min",
       fill="Mean travel time")
```

<img src="/post/2018-08-25-oslobnb-where-to-stay-in-oslo_files/figure-html/unnamed-chunk-11-1.png" width="672" />

## Text analytics... in space

This dataset contains very rich text features, one of which is host's description of rental location. What are the typical things highlighted by hosts in the description/summary fields about their accomodations? We can build a topic model using Latent Dirichlet Allocation (LDA) to try and distill key themes discussed by hosts in the description/summary fields on AirBnB. We will use awesome `text2vec` library which has excellent documentation and some motivating examples to get you started. First we need to make sure texts we will include into topic model are written in the same language (quite a few of the posts are in Norwegian).


```r
library(text2vec)
library(cld2)
library(stopwords)

listings_en_df <- listings_ext_df %>% 
  replace_na(list(summary="", description="")) %>% 
  mutate(txt = paste(summary,description),
         cjk=grepl("[\U4E00-\U9FFF\U3000-\U303F]", txt),
         #cld2=cld2::detect_language(txt,plain_text = FALSE),
         cld3=cld3::detect_language(txt)) %>% 
  filter(!cjk, cld3=="en")


tokens <- listings_en_df$txt %>% 
  tolower %>% 
  word_tokenizer()

it <- itoken(tokens, ids=listings_en_df$id, progressbar=FALSE)

v <- create_vocabulary(it, stopwords = stopwords::stopwords("en")) %>% 
  prune_vocabulary(term_count_min = 10, doc_proportion_max = 0.2)

vectorizer <- vocab_vectorizer(v)

dtm <- create_dtm(it, vectorizer, type="dgTMatrix")
```

LDA is an unsupervised technique which requires specifying number of topics ahead of time. Lets build a model for 10 topics. We will extract top 5 words from each topic and use heatmap to indicate the prevalence of topic in space. We will use a helper function which will build each of the plots finally assembled by `patchwork`.

Note how certain neighborhoods get identified by frequently mentioned keywords. There's clearly identifiable Grunerløkka area, which may be preferred by people seeking to surround themselves with parks and bars. There's also museum/botanical garden area between Tøyen and Grønland, perhaps more appealing to those appreciating the art of Edvard Munch. There's Sørenga/Ekeberg area close to Oslo S, which attracts people with beach view and rooftop terasses. Akerbrygge stands out, as well as Vigelandsparken. There's a recent [post by Julia Silge](https://juliasilge.com/blog/evaluating-stm/) which discusses the optimal number of topics in the model.


```r
n_topics <- 10
set.seed(260)

lda_model <- LDA$new(n_topics=n_topics, doc_topic_prior=0.1, topic_word_prior=0.01)


doc_top_distr <- lda_model$fit_transform(x=dtm, n_iter=1000,
                          convergence_tol = 0.0001, n_check_convergence = 50, 
                          progressbar = FALSE)
#> INFO [2018-09-10 11:48:43] iter 50 loglikelihood = -2942745.040
#> INFO [2018-09-10 11:48:45] iter 100 loglikelihood = -2876817.411
#> INFO [2018-09-10 11:48:47] iter 150 loglikelihood = -2847013.162
#> INFO [2018-09-10 11:48:48] iter 200 loglikelihood = -2832093.573
#> INFO [2018-09-10 11:48:50] iter 250 loglikelihood = -2824146.129
#> INFO [2018-09-10 11:48:52] iter 300 loglikelihood = -2818624.475
#> INFO [2018-09-10 11:48:54] iter 350 loglikelihood = -2815330.266
#> INFO [2018-09-10 11:48:56] iter 400 loglikelihood = -2814211.240
#> INFO [2018-09-10 11:48:58] iter 450 loglikelihood = -2812576.742
#> INFO [2018-09-10 11:49:00] iter 500 loglikelihood = -2811477.631
#> INFO [2018-09-10 11:49:02] iter 550 loglikelihood = -2813359.397
#> INFO [2018-09-10 11:49:02] early stopping at 550 iteration

varnames_df <- lda_model$get_top_words(n=5, topic_number=seq_len(n_topics), lambda=0.2) %>% 
  apply(2,paste0, collapse=", ") %>% 
  as_tibble() %>%
  rename(top5words=value) %>% 
  mutate(key=paste0("V", seq_len(n_topics)))

#lda_model$plot()

listings_en_df <-listings_en_df %>% 
  select(id, longitude, latitude) %>% 
  bind_cols(as_tibble(doc_top_distr)) %>% 
  gather(key, value, -id, -longitude, -latitude) %>% 
  left_join(varnames_df, by="key")

osl_map_words <- get_map(location = "Oslo, Norge", zoom = 13, maptype ="toner-lite") 

plotLDA <- function(df){
ggmap::ggmap(osl_map_words)+
  stat_summary_2d(data=df, aes(x=longitude,y=latitude, z=value), 
                               fun = mean, alpha = 0.5, bins = 20, 
                  inherit.aes = FALSE, show.legend = FALSE)+
  #geom_tile(data=listings_en_df, aes(x=longitude,y=latitude, fill=V7), 
  #                                      alpha = 0.5)+
  scale_fill_gradient(name = "Value", low = "transparent", high = "red") +
  theme_void(base_family = "Roboto Condensed")+
    labs(subtitle=first(df$top5words))
}

listings_en_df %>% 
  split(.$key) %>% 
  map(plotLDA) %>% 
  reduce(`+`) +
  patchwork::plot_layout(ncol=2)
```

<img src="/post/2018-08-25-oslobnb-where-to-stay-in-oslo_files/figure-html/unnamed-chunk-13-1.png" width="672" />

## PostGIS and rasters

Norway is very rich on public data. State Statistic Bureau of Norway hosts a ton of [geo-referenced data](https://www.ssb.no/natur-og-miljo/geodata) aggregared to 250m grid cells. This data has been downloaded and saved in the local PostGIS database. Although grid cells look rectangular in the original crs (epsg 3045), the shape of the cells will change when we reproject it to geographical coordinates (epsg 4326).

We will read a few more files: trade and service analysis (downloaded from ssb.no in ESRI shapefile format), employees and establishments data (in .csv), as well as population (all referencing the same 250m grid). We will import all 


```r
conn <- RPostgreSQL::dbConnect("PostgreSQL", host="localhost", dbname="ssb_geodata", user="postgres", password="postgres")

pgPostGIS(conn)
#> PostGIS extension version 2.4.4 installed.
#> [1] TRUE
# lists geometry columns
#pgListGeom(conn, geog = TRUE)
# lists raster columns
#pgListRast(conn)

# alternative crs is EPSG 25833
hsa_sf<- st_read(here::here("input", "ssb", "HandelServiceAnalyseRuter2017", "AnalyseRuter2017.shp"), crs=3045) %>% 
  filter(KOMMUNENR=="0301")
#> Reading layer `AnalyseRuter2017' from data source `D:\R\ddrive.no\input\ssb\HandelServiceAnalyseRuter2017\AnalyseRuter2017.shp' using driver `ESRI Shapefile'
#> Simple feature collection with 235303 features and 15 fields
#> geometry type:  POLYGON
#> dimension:      XY
#> bbox:           xmin: -75000 ymin: 6451000 xmax: 1098500 ymax: 7937750
#> epsg (SRID):    3045
#> proj4string:    +proj=utm +zone=33 +ellps=GRS80 +towgs84=0,0,0,0,0,0,0 +units=m +no_defs

# employees and establishments
emp_df <- data.table::fread(here::here("input", "ssb", "NOR250M_EST_2017", "NOR250M_EST_2017.csv"), 
                            data.table = FALSE, sep=";", integer64 ="character")

# population
pop_df <- data.table::fread(here::here("input", "ssb", "Ruter250m_beflandet_2018", "Ruter250m_beflandet_2018.csv"), 
                            data.table = FALSE, sep=";", integer64 ="character")

# 0219 Bærum
# 0301 Oslo
q <- "SELECT * FROM rute250land WHERE kommunenr='0301'"
oslo_ruter <- st_read_db(conn, query=q)
#> Warning: 'st_read_db' is deprecated.
#> Use 'st_read' instead.
#> See help("Deprecated")

RPostgreSQL::dbDisconnect(conn)
#> [1] TRUE
```

Let's define the bounding box around the places of interest and use it to limit the extent of the grid for each type of data. We will then turn each of the polygons to raster with the same resolution (using awesome `fasterize` function in the same package) and put them into one stack.


```r
places_bbox_sf <- places_sf %>% 
  st_transform(crs=3045) %>% 
  st_bbox() %>% 
  st_as_sfc()

oslo_est_sf <- oslo_ruter %>% 
  left_join(emp_df, by=c("ssbid"="SSBID250M")) %>% 
  st_intersection(places_bbox_sf)

oslo_pop_sf <- oslo_ruter %>% 
  left_join(pop_df, by=c("ssbid"="ru250m")) %>% 
  st_intersection(places_bbox_sf)

oslo_hsa_sf <- hsa_sf %>% 
  st_intersection(places_bbox_sf)

oslo_est_r <- raster(oslo_est_sf, res=250)
oslo_pop_r <- raster(oslo_pop_sf, res=250)
oslo_hsa_r <- raster(oslo_hsa_sf, res=250)

oslo_est_f <- fasterize(oslo_est_sf, oslo_est_r, field="est_tot", fun="sum")
oslo_emp_f <- fasterize(oslo_est_sf, oslo_est_r, field="emp_tot", fun="sum")
oslo_pop_f <- fasterize(oslo_pop_sf, oslo_pop_r, field="pop_tot", fun="sum")
oslo_hsa_f <- fasterize(oslo_hsa_sf, oslo_hsa_r, field="hsareal", fun="sum")

oslo_ee_s <- stack(oslo_est_f, oslo_emp_f, oslo_pop_f, oslo_hsa_f)
names(oslo_ee_s) <- c("Companies", "Employees", "Populaton", "ShopServiceArea")

plot(oslo_ee_s)
```

<img src="/post/2018-08-25-oslobnb-where-to-stay-in-oslo_files/figure-html/unnamed-chunk-15-1.png" width="672" />

How can we relate these layers to each other? Inspired by another [@StatnMap post](https://statnmap.com/2018-01-27-spatial-correlation-between-rasters/) we can use `raster::focal` to compute correlation in the 5x5 grid segments to highlight patterns in the spatial distributions of these features. Central Oslo seems to be primarily occupied by businesses. It also has much better retail space coverage than western part of Oslo.


```r
tmp_osl_r <- raster(oslo_ee_s, 1)
values(tmp_osl_r) <- 1:ncell(oslo_ee_s)

charts_df <- tibble::tribble(~layer1, ~layer2, ~plotTitle,
                             1, 3, "Residential Areas",
                             3, 4, "Retail Locations",
                             )

osl_map <- get_stamenmap(bbox = as.numeric(st_bbox(places_sf)), zoom = 13, maptype = "toner-lite")

plot_focal <- function(layer1, layer2, plotTitle, tmp_osl_r, oslo_ee_s, osl_map, places_df){

  focal_cor_df <- focal(x=tmp_osl_r, w=matrix(1,5,5),
                     fun=function(x, y=oslo_ee_s){
                       cor(values(y)[x,layer1], values(y)[x,layer2],
                           use="na.or.complete")
                     }) %>%  
    projectRaster(crs="+proj=longlat +datum=WGS84 +no_defs") %>% 
    as("SpatialPixelsDataFrame") %>% 
    as.data.frame() %>% 
    setNames(c("value", "x", "y"))
  
  ggmap::ggmap(osl_map)+
    geom_tile(data=focal_cor_df, aes(x,y, fill=value), alpha=0.4, inherit.aes = FALSE)+
    geom_point(data=places_df, aes(x=lon, y=lat), inherit.aes = FALSE, color="violetred", size=5)+
    geom_label_repel(data=places_df, aes(x=lon, y=lat, label=place), color="violetred", inherit.aes = FALSE, size=3)+
    scale_fill_viridis_c(option="A", direction=1)+
    theme_void(base_family = "Roboto Condensed")+
    labs(subtitle=plotTitle, fill="Correlation")
}

charts_df %>% 
  pmap(plot_focal, tmp_osl_r, oslo_ee_s, osl_map, places_df) %>% 
  reduce(`+`) +
  patchwork::plot_layout(ncol=2)
```

<img src="/post/2018-08-25-oslobnb-where-to-stay-in-oslo_files/figure-html/unnamed-chunk-16-1.png" width="672" />

## Conclusion

Producing spatial anlysis in R is simple, especially when you have data. Luckily, Norway can pride itself for making a ton of spatial data available for a curious data analyst to enjoy. Having performed this analysis, we have something to report back to my friend planning to visit to Oslo. I also hope that you, eager reader of my blog, will soon have something to report back to your boss with awesome visualizations and spatial data insight!
