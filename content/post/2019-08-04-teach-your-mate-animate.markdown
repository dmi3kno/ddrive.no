---
title: Teach your mate animate
author: dmi3kno
date: '2019-08-04'
slug: teach-your-mate-animate
categories:
  - blog
tags:
  - r
  - gganimate
  - magick
  - bunny
---

```r
library(magick)
library(bunny)
library(tidyverse)
#remotes::install_github("dmi3kno/cheese")
library(cheese)
```

I found this old-ish unfinished blogpost, which I thought I would get published anyways because animations is always so fun. We want to make the presentation of otherwise boring plot somewhat fun to watch. I am going to use the dataset from my `cheese` package inspired by the twitter exchange with Colin Fay. We will be plotting a fairly boring price chart for the price of Cheddar cheese in 2001-2002.


```r
cheese_data <- cheese::cheese_price %>% 
  filter(between(lubridate::year(date), 2001,2002),
         category=="Cheddar Barrel, 500lb") %>% 
  mutate(rowid=as.numeric(date))

cheese_data %>% 
  ggplot()+geom_line(aes(as.POSIXct(date), price), color="#e72225")+
  labs(title="Price of Cheddar",
       subtitle = "Traded in 500lb barrels on CME",
       x=NULL, y="Price, $/lb")+
  scale_x_datetime()
```

<img src="/post/2019-08-04-teach-your-mate-animate_files/figure-html/unnamed-chunk-2-1.png" width="672" />

```r
ggsave(here::here("input","cheese_price_chart.png"), width = 7, height = 4)
```

Nothing super remarkable here. Boring theme, no particular insight of what happened in the spring of 2001, which drove the price of cheddar cheese up so much. 

But then you come across this adorable little gif which you think maybe could make your data analyst life a little bit more cheerful! 

<img src="../../input/mouse_riding.gif" style="display: block; margin: auto;" />

Wouldn't it be cool, if we could have this adorable little mouse ride across our chart to make it a little bit more fun! What a cool-but-useless(c) idea!

## Prepare the mouse


```r
remove_bg <-function(img){
  
  mask <- img %>% image_convert(colorspace = "Gray") %>% 
    image_threshold("black", "90%") %>%
    image_threshold("white", "80%") %>% 
    image_negate() %>% image_morphology("Close", "Disk:3") %>% 
    image_fill("red") %>% 
    image_transparent("red")
  
  image_composite(img, mask, "CopyOpacity")
}

msrd_lst <- image_read(here::here("input","mouse_riding.gif")) %>% 
  lapply(remove_bg)
```

First thing we want to do is prepare the mouse for landing in our plot. The mouse is riding on the white background and we need to make that white background transparent. We can not just remove the color, because the mouse's body is also white, so we use the mask trick (like we did with the bunny in one of the earlier posts). The easiest way to create the mask over the object is to make sure it is an enclosed isolated blob of white pixels on a black background. Image morphology to the rescue! I used "Close" morphology to make sure the little gaps in the silhouette of a mouse are completely filled (closed). Then we flood the rest of the image with some easily identifiable color and declare the color transparent. The flooding always starts with pixel (1,1), i.e. from the corner, so the area around the enclosed silhouette of a mouse will be made transparent this way, which is what we wanted all along. We apply the mask by "copying it" to become an opacity layer of our new image. 

The trick about the `gif` images is that when imported to magick they become a stack of images (something like a vector). We can `lapply` a function over it, but remember that it is a list of images now, not a stack anymore.

## Scaling

We want to scale the mouse to be of reasonable size in relation to the chart. Note that I made the line color on the chart match the bicycle frame color, so we can use the same function to extract those. Here we make the color of the chart/bike transparent to move it to the Alpha layer and thin with Erode morphology a bit. We extract the non-black pixels into a data frame.


```r
background <- image_read(here::here("input","cheese_price_chart.png"))

find_line<- function(img){
  img %>% 
  image_convert(matte=FALSE) %>% 
  image_transparent("#e72225", fuzz=20) %>% 
  image_channel("Alpha") %>% image_negate() %>% 
  image_morphology("Erode", "Diamond") %>% 
  image_raster() %>% 
  dplyr::filter(col!="#000000ff")
}

chart_line <-  find_line(background)
bike_line <- find_line(msrd_lst[[1]])
```

Here we just calculate some proportions to make sure the size of the bicycle frame is about 1 month in length on the chart and scale down the mouse stack.


```r
day_px <- diff(range(chart_line$x))/diff(range(as.numeric(cheese_data$date)))
buck_px <- diff(range(chart_line$y))/diff(range(cheese_data$price))
month_px <- day_px*30
axis_px <- diff(range(bike_line$x))

msrd_lst_small <- lapply(msrd_lst, image_scale,
                        paste0(round(month_px/axis_px*100, 2), "%"))
mouse_offset_x <- min(bike_line$x)*month_px/axis_px
```

## Animate!

Now we just need to create a grid of mouse locations and a grid of frames (1 to 8 repeated as many times as we have locations) and sequentially combine the frames.


```r
x_offsets <- seq.int(from=-200, to=2100, by=20)
msrd_lst_ids <- rep_len(seq_along(msrd_lst), length.out = length(x_offsets))

frames_lst <- list()

for(i in seq_along(msrd_lst_ids)){
  x_offset <- x_offsets[i]
  j <- msrd_lst_ids[i]
  frames_lst[[i]] <- image_compose(background, msrd_lst[[j]], 
                                   gravity = "SouthWest", 
                offset = geometry_point(x_offset,-50))
}

stack <- Reduce(c, frames_lst)

stack %>% image_scale("700x") %>%  image_animate() %>% 
  image_write_gif(here::here("input", "mouse_riding_cheese_price_chart.gif"), delay=0.05)
```

```
## [1] "/home/dm0737pe/Projects/ddrive.no/input/mouse_riding_cheese_price_chart.gif"
```

And here's the result


```r
knitr::include_graphics(here::here("input", "mouse_riding_cheese_price_chart.gif"))
```

![](../../input/mouse_riding_cheese_price_chart.gif)<!-- -->

## To be continued

Another idea which I had was to scale the mouse even more and have it ride along the graph, up and down. This would be extremely fun! We probably need to smooth the line a bit, so here's a quick loess smoothing


```r
loess10<-loess(price~rowid, data=cheese_data, span=0.2)
lowess10_pred <- data.frame(
  rowid=seq.int(min(cheese_data$rowid),
                max(cheese_data$rowid), 1), 
  stringsAsFactors = FALSE)

lowess10_pred$date <- as.POSIXct(as.Date(lowess10_pred$rowid, origin="1970-01-01"))
lowess10_pred$pred <- predict(loess10, newdata = lowess10_pred$rowid)

cheese_data %>% 
  ggplot()+geom_line(aes(as.POSIXct(date), price), color="#e72225")+
  geom_line(data=lowess10_pred, aes(date, pred), color="black", inherit.aes = FALSE)+
  labs(title="Price of Cheddar",
       subtitle = "Traded in 500lb barrels on CME",
       x=NULL, y="Price, $/lb")+
  scale_x_datetime()
```

<img src="/post/2019-08-04-teach-your-mate-animate_files/figure-html/unnamed-chunk-9-1.png" width="672" />

Then we would calculate the derivative to find the slope of the curve at every point and locate and rotate the mouse at every frame to have it moving along the curve (without showing the curve, of course)
