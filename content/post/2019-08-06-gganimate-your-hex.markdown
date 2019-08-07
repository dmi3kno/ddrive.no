---
title: gganimate your hex
author: dmi3kno
date: '2019-08-06'
slug: gganimate-your-hex
categories:
  - blog
tags:
  - r
  - gganimate
  - magick
---



Today I would like to share a few tips about how you can integrate plotting in `{ggplot2}` with image post-processing in `{magick}`. We will start with simple example and later proceed to animated plots and animating image stacks in `{magick}`.

## Gapminder needs hex

I looked at Jenny Bryan's [`{gapminder}` package](https://github.com/jennybc/gapminder) and discovered that it does not have a hex! I love `{gapminder}` and we at Carpentry@UiO use it extensively for teaching basic data visualization and data wrangling skills. We even have `{purrr}` excercises with `{gapminder}`. What can be better than using one of the most famous plots of all times for a package logo?

![](/post/2019-08-06-gganimate-your-hex_files/gapminder_home_bg_3.jpg)

Let's plot the same image using `{gapminder}` package. We will not be able to visualize data for 2008, but the year before, should work as well (our data is for 1952-2007).


```r
library(gapminder)
library(tidyverse)
library(ggrepel)
library(hrbrthemes)

lbl_countries <- c("Congo, Dem. Rep.", "China", "Japan", "United States", "South Africa")
gapminder2007 <- gapminder %>% filter(year==2007) 

gapminder2007 %>% 
  ggplot(aes(gdpPercap, lifeExp, size = pop, colour = country)) +
  annotate("text", x=4000, y=65, label="2007", size=50, colour="lightgrey", alpha=0.5)+
  geom_point(alpha = 0.7, show.legend = FALSE) +
  geom_label_repel(data=filter(gapminder2007, country %in% lbl_countries), 
                   aes(gdpPercap, lifeExp, label=country), inherit.aes = FALSE)+
  scale_colour_manual(values = country_colors) +
  scale_size(range = c(2, 12)) +
  scale_x_log10(minor_breaks=rep(1:9, 21)*(10^rep(-10:10, each=9))) +
  theme_ipsum_rc(grid_col = "grey90", axis_title_just = 'm')+
  labs( x = 'GDP per capita', y = 'life expectancy')
```

<img src="/post/2019-08-06-gganimate-your-hex_files/figure-html/unnamed-chunk-1-1.png" width="528" />

```r

ggsave(here::here("input", "gapminder2007.png"), width = 5.5, height=5.5)
```

We will use our saved image as a background for logo. Labels are nice and big and colors are bright enough and distinct.

Once this is done, it does not take too much time to turn it into a logo, thanks to `{magick}` and `{bunny}` following the steps we discussed [in the previous blogpost](https://www.ddrive.no/post/making-hex-and-twittercard-with-bunny-and-magick/). We first prepare canvas (white hex with purple border), read in our `ggplot2` image and compose it using `magick::image_composite()`. Note that if you installed `magick` earlier than past weekend, you might need to reinstall the latest version from github, since Jeroen has just added `gravity` argument to this function (or you can continue using `bunny::image_compose()`, which is doing the same thing).


```r
# remotes::install_github("ropensci/magick")
library(magick)
# remotes::install_github("dmi3kno/bunny")
library(bunny)

canvas_hex <- bunny::image_canvas_hex(border_color = "#523456", fill_color = "white")
canvas_border <- bunny::image_canvas_hexborder(border_size = 4, border_color = "#523456")

gap07 <- image_read(here::here("input", "gapminder2007.png"))

gap_logo <- image_composite(canvas_hex, gap07, gravity = "center", offset = "-30+0") %>% 
  image_annotate("gapminder", gravity = "center", location = "+10+150", 
                 size=300, font="Aller", color="#523456") %>% 
  image_composite(canvas_border, gravity = "center") 

gap_logo %>% image_scale("500x500") #for screen only
```

<img src="/post/2019-08-06-gganimate-your-hex_files/figure-html/unnamed-chunk-2-1.png" width="217" />

```r

gap_logo %>% 
  image_scale("1200x1200") %>% 
  image_write(here::here("input", "gapminder_logo_big.png"), density = 600)

gap_logo %>% 
  image_scale("200x200") %>% 
  image_write(here::here("input", "gapminder_logo_small.png"), density = 96)
```
The large hex will be useful for color printing and the small hex can be hosted on the github. Wait a second, since we can upload a `png`, what prevents us from posting it as a `gif`??

## Animated hex

The idea is pretty simple, take the plot above, animate it and try to put the animation onto the hex canvas. There are a few gotcha's when it comes down to year annotation on the background. `gganimate` has awesome helper variables, such as `frame_time`, but they are only available within `labs()`. So we have to use `geom_text()` with subsetting to only 1 country (essentially 1 image per year) to avoid overplotting. Another gotcha is that `geom_label_repel()` look really ugly in animation. Labels keep jumping around and produce more noise than guidance. We will switch to regular labels and just "nudge" them a little bit to the side to make sure the data points are clearly visible.

Last, but very important point, we don't want to compress the output from our plotting function into a `.gif` or an `.mp4` file. This will result in image quality loss, and in case of `gifski`, image becomes patchy and really loosing color on individual frames. 

> `gifski` is heavily optimizing filesize and tries to ensure smooth transitions at the expense of individual frame quality.  Long story short, you want to use `file_renderer()`. 

When you are producing `.gif` files directly out of `ggplot`, default `gif` renderer might be fine, but since we will continue doing something with individual frames, we want to pass them to `magick` "as is". Just indicate the folder where all 100+ images will be saved and consider the job of `gganimate` done. 


```r
library(gganimate)

gap_ani <- gapminder %>% 
  ggplot(aes(gdpPercap, lifeExp, size = pop, colour = country)) +
  geom_text(data=. %>%filter(country %in% lbl_countries[1]), 
            aes(x=4000, y=65, label=as.character(floor(year))), 
            size=35, colour="lightgrey", alpha=0.5, inherit.aes = FALSE)+
  geom_point(alpha = 0.7, show.legend = FALSE) +
  geom_label(data=. %>% filter(country %in% lbl_countries), 
                   aes(gdpPercap, lifeExp, label=country), 
             nudge_x = 0.05, nudge_y = 4, inherit.aes = FALSE)+
  scale_colour_manual(values = country_colors) +
  scale_size(range = c(2, 12)) +
  scale_x_log10(minor_breaks=rep(1:9, 21)*(10^rep(-10:10, each=9))) +
  theme_ipsum_rc(grid_col = "grey90",axis_title_just = 'm')+
  labs( x = 'GDP per capita', y = 'life expectancy') +
  transition_time(year) +
  ease_aes('linear')

animate(gap_ani, height=310, width=310, 
        renderer=file_renderer(dir = here::here("input", "snapshots"), overwrite = TRUE))[1:6]
#> [1] "/home/dmi3kno/Projects/ddrive.no/input/snapshots/gganim_plot0001.png"
#> [2] "/home/dmi3kno/Projects/ddrive.no/input/snapshots/gganim_plot0002.png"
#> [3] "/home/dmi3kno/Projects/ddrive.no/input/snapshots/gganim_plot0003.png"
#> [4] "/home/dmi3kno/Projects/ddrive.no/input/snapshots/gganim_plot0004.png"
#> [5] "/home/dmi3kno/Projects/ddrive.no/input/snapshots/gganim_plot0005.png"
#> [6] "/home/dmi3kno/Projects/ddrive.no/input/snapshots/gganim_plot0006.png"
```

Now, what shall we do with this many pictures? We can read them to magick all at once. Just produce a list of file paths and `magick` will read them into a single object, which we will relate to as "image stack". Image stacks consist of memory pointers, so you can not save them, into, for example, an RDS file (I tried!). They are like virtual vectors pulling addresses to memory together, so `magick` would know that these images belong together.  


```r
gap_anim <- list.files(here::here("input", "snapshots"), 
                       pattern = "gganim_plot*", full.names = TRUE) %>%
            image_read()
```

Image stacks are not lists. Rather they can be subset as vectors (using single brackets). Our animation canvas is much smaller, so lets resize the hex size, since we dont intend to use animated version for print. Below is a draft logo using very first frame in the stack, which I subset as `gap_anim[1]`.


```r
canvas_hex_small <- image_resize(canvas_hex, "330x330")
canvas_border_small <- image_resize(canvas_border, "330x330")

image_composite(canvas_hex_small, gap_anim[1], gravity = "center", offset = "-0+10") %>% 
  image_annotate("gapminder", gravity = "center", location = "+5+35", 
                 size=50, font="Aller", color="#523456") %>% 
  image_composite(canvas_border_small, gravity = "center")
```

<img src="/post/2019-08-06-gganimate-your-hex_files/figure-html/unnamed-chunk-5-1.png" width="144" />

This looks a little busy, but OK, since we will have many more frames where data points move upward. Now, we can iterate over "image stack" using `for`-loop, `lapply` or `map`. Either method will do, as long as that at the end we are getting a list. You can also use built-in `magick::image_apply`, but in this case we will be combining two images, so explicit iteration with `lappply` seems to be appropriate (I discovered that `mapply` doesn't work as it tries to coerce `magick` type into something else).


```r
img_lst <- lapply(gap_anim, function(img){
  image_composite(canvas_hex_small, img, gravity = "center", offset = "-0+10") %>% 
    image_annotate("gapminder", gravity = "center", location = "+5+35", 
                   size=50, font="Aller", color="#523456") %>% 
    image_composite(canvas_border_small, gravity = "center")
})
```

Last operation is "converting" the list of images back into the "image stack", which can be done using operator `c()`, wrapped into `Reduce()`. Another possibility is `magick::image_join()` but I discovered that it is significantly slower. Finally, we write the file to disk. Note the last argument in `gifski`, which is responsible for "delay" of individual frames or, generally speaking, for the speed of your video.


```r
Reduce(c, img_lst) %>% 
  image_animate() %>% 
  image_write_gif(here::here("input", "animated_logo.gif"), delay=0.1, progress=FALSE)
#> [1] "/home/dmi3kno/Projects/ddrive.no/input/animated_logo.gif"
```
![](/post/2019-08-06-gganimate-your-hex_files/animated_logo.gif)

What do you say @JennyBryan, can we use this as a package logo?

## Animated twittercard

This is what [Github says](https://help.github.com/en/articles/customizing-your-repositorys-social-media-preview) about the social media preview (aka. twittercard)

> Your image should be a PNG, JPG, or GIF file under 1 MB in size. For the best quality rendering, we recommend keeping the image at 640 by 320 pixels. 

Let's try and fit under 1 MB with animated logo. We probably will have to cut back on number of frames and compress quality further. If we leave 40px for border, that leaves us with 560x240 of "inner space" to play with. Let's try and scale down as well as resample the image stack we still have in the memory to fit under the size boundary.

As with out example above, let's try and make prototype card for single frame. We will need to scale down the images to fit onto the card. 


```r
gap_card <- image_canvas_ghcard(width=640, height = 320, border = "40x40")

image_composite(gap_card, 
                image_resize(img_lst[[1]], "240x240", "Lanczos2"),
                gravity = "East", offset = "+70+0") %>% 
  image_annotate("Update your", font="Aller", size=30, color="#523456",
                 gravity = "West", location="+50-30") %>% 
  image_annotate("worldview!", font="Aller", size=33, weight=700, color="#523456",
                 gravity = "West", location="+50-0") %>% 
  image_annotate("install.packages('gapminder')", font="Ubuntu mono", size=15, color="#0f0a10",
                 gravity = "West", location="+25+40") %>% 
  image_border_ghcard(border = "40x40")
```

<img src="/post/2019-08-06-gganimate-your-hex_files/figure-html/unnamed-chunk-8-1.png" width="320" />

This looks ok. I am not sure how big the file will be, so we will not be adding bells and whistles to the image (like address alternating between CRAN and Jenny's repo).

Right now we have roughly 6-8 frames per single `gapminder` year (100 images for 12 observations). If we drop, say, every third image, things shouldn't go all that bad.  


```r
img_lst_down <- img_lst[seq_along(img_lst)%%3!=0]
length(img_lst_down)
#> [1] 67

gap_card_lst <- lapply(img_lst_down, function(img){
  image_composite(gap_card, 
                image_resize(img, "240x240", "Lanczos2"),
                gravity = "East", offset = "+70+0") %>% 
  image_annotate("Update your", font="Aller", size=30, color="#523456",
                 gravity = "West", location="+50-30") %>% 
  image_annotate("worldview!", font="Aller", size=33, weight=700, color="#523456",
                 gravity = "West", location="+50-0") %>% 
  image_annotate("install.packages('gapminder')", font="Ubuntu mono", size=15, color="#0f0a10",
                 gravity = "West", location="+25+40") %>% 
  image_border_ghcard(border = "40x40")
})
```
So we just need to stich it all together and write to disk.


```r
Reduce(c, gap_card_lst) %>% 
  image_animate() %>% 
  image_write_gif(here::here("input", "animated_card.gif"), delay=0.1, progress=FALSE)
#> [1] "/home/dmi3kno/Projects/ddrive.no/input/animated_card.gif"
file.info(here::here("input", "animated_card.gif"))$size
#> [1] 1001259
```
![](/post/2019-08-06-gganimate-your-hex_files/animated_card.gif)

Yay! Spot on!
