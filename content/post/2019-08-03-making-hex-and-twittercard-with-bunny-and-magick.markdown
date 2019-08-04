---
title: Making hex and twittercard with bunny and magick
author: dmi3kno
date: '2019-08-03'
slug: making-hex-and-twittercard-with-bunny-and-magick
categories:
  - blog
tags:
  - r
  - bunny
  - magick
---

There's a lot of talk on twitter about "Hex-driven development" method.

<!--html_preserve-->{{% tweet "1022443077733105665" %}}<!--/html_preserve-->

You don't know what it is? Well, you gotta read on!

## Making hex sticker for your package

Hex stickers have become not only de-facto currency, collectables and bragging items in the #rstats community, but they also serve another very important purpose: keeping package developer motivation up to continue developing or maintaining a package. I don't have the data to back this up, so I am going to speak only from my own experience: making and sharing beautiful hex stickers instill confidence that the package I have written is useful (at least for those who are hunting after the sticker). Producing a sticker (and especially printing it using my own pocket money) creates commitment and emotional attachment to the project that often keeps me going despide difficulties, lack of time and frustration of seemingly endless bugs and confusing github issues.

Many of my colleagues produce stickers in Photoshop, Inkscape, and even [PowerPoint](https://emitanaka.github.io/post/hexsticker/). There are several packages for producing [stickers](https://github.com/GuangchuangYu/hexSticker), but I found them a little too "black-box" and inflexible for my liking. Therefore, [in the best spirit of this community](https://www.ddrive.no/post/rant-about-dependencies/), I decided to make my own toolkit for producing hex stickers.

I have been fascinated lately with `magick` package and [started my own `magick` helper package](https://www.ddrive.no/post/miracles-with-magick-and-bunny/) called `bunny`. Among latest additions to `bunny` there are a few functions for creating hexagon-shaped objects that can be used for designing stickers. Behind the scenes,  `image_canvas_hex()` produces hexagon geometry using awesome [`ggforce` package](https://github.com/thomasp85/ggforce) which, btw, has one of the most impressive and sought-after stickers, so you might as well want to get it installed before going further. Other than that, we will be using `magick` and `bunny` for designing a new sticker for package I have in mind.


```r
# install.packages("ggforce")
library(magick)
library(bunny)
```

## Introducing bbox

Let me introduce you a new package, which as of today has zero functions. In the best traditions of hex-driven development, I am setting out to design a hex before any code is written! 

I have an idea of pulling together functions that deal with manipulation of bounding boxes (hence the name) and geometries from [`hocr`](https://github.com/dmi3kno/hocr/) and [`bunny`](https://github.com/dmi3kno/bunny/). These functions clearly belong together and it is becoming difficult to maintain and manage certain duplication of functionality between these two packages. In `hocr` bounding boxes are used to describe location of the words on the OCR'ed page, while in `bunny` geometries are used for describing points, areas and lines plotted on images. There's enough overlap in functionality to find these helpful functions a new home.

Since the package will be dealing with bounding boxes and rectangular geometries, I was thinking I could build a hex logo around a picture of a frame. Wooden frames are boring and after searching around internet I came across this beautiful picture of "finger frame", which I liked a lot. The picture has [free licence](https://pixabay.com/illustrations/fantasy-fingers-scene-frame-4065903/). I think I am going to use it for my package logo. 


```r
# load and convert to png
p2_clouds <- image_read("https://raw.githubusercontent.com/dmi3kno/bbox/master/data-raw/fantasy-4065903_1920.jpg") %>%  
  image_convert("png")
p2_clouds
```

<img src="/post/2019-08-03-making-hex-and-twittercard-with-bunny-and-magick_files/figure-html/unnamed-chunk-3-1.png" width="960" />

I think the fingers look nice, but they are a little too far from each other. I would love to see them closer together and I don't really need that cloud in the middle, although the watercolor styling and paper-patterned background sure look nice. I used `bunny::image_plot()` to locate the area I want to cut out. Upper hand seems to be contained in the box `1200x900+0+0` (which you need to read as "1200 by 900 pixels starting from upper left corner with 0,0 offset"). Just wanted to remind you that `bunny::image_plot()` takes either point-geometry (which only includes offset in the format `+0+0`) or an area-geometry (in the format we provided). In the former case it plots a point at the specified location and in the latter case it plots a transparent rectangle over the specified area. Anyways, once we know what we want, we can cut the image.


```r
up_hand <- p2_clouds %>% image_crop("1200x900+0+0") %>% 
  image_convert(colorspace = "Gray") %>% 
  image_threshold("white", "40%") %>% 
  image_threshold("black", "30%")

down_hand <- p2_clouds %>% image_crop("1200x1200+750+870") %>% 
  image_convert(colorspace = "Gray") %>% 
  image_threshold("white", "40%") %>% 
  image_threshold("black", "30%")
```

Since the hands are black, I am going to discard the background information and extract the silhouette of the hands in black and white format. Lets convert the image to grayscale and threshold it to remove all other colors. Note that I am thresholding both white and black, essentially first forcing all light colors into white and then all dark colors into black. 

There are some thin elements (white lines) on this image. Let's try to enhance these features using my favorite function in the `magick` package called `image_morphology()`. As I mentioned in my [previous blogpost](https://www.ddrive.no/post/miracles-with-magick-and-bunny/) on `magick`, [morphology page](https://www.imagemagick.org/Usage/morphology/) has become my go-to resource for many cool functions and creative ideas for implementing with `magick`. This time around we will be using "Erosion" and "Smoothing" to first "widen" the gaps a little bit and then smooth the edges of the dark elements. All of these operations are performed in negative mode, so we will `image_negate()` our images first.


```r
up_hand <- up_hand %>%
  image_negate() %>% 
  image_morphology("Erode", "Diamond") %>% 
  image_morphology("Smooth", "Disk:1.2") %>% 
  image_negate() %>% 
  image_transparent("white", 10) %>% 
  image_resize(geometry_size_pixels(700), "Lanczos2")

down_hand <- down_hand %>%
  image_negate() %>% 
  image_morphology("Erode", "Diamond") %>% 
  image_morphology("Smooth", "Disk:1.2") %>% 
  image_negate() %>% 
  image_transparent("white", 10) %>% 
  image_resize(geometry_size_pixels(700), "Lanczos2")

down_hand
```

<img src="/post/2019-08-03-making-hex-and-twittercard-with-bunny-and-magick_files/figure-html/unnamed-chunk-5-1.png" width="350" />

At the end I am making the background transparent and resizing the image using `image_resize()`, which is different from `image_scale()` because it has this cool `filter` attribute which allows specifying image resampling algorithm. I use this function when I care about the quality of the resulting image.

Now hands are ready to plot, but I also want to produce shadows which will be located underneath. I take images, color them grey (remember they are on transparent background, so coloring affects only solid part of the image, which is what we need) and blur them using `image_blur()`. These will be nice soft shadows we can place underneath the hand images.


```r
up_hand_shadow <- up_hand %>% 
    image_colorize(50, "grey") %>% 
    image_blur(20,10)

down_hand_shadow <- down_hand %>% 
    image_colorize(50, "grey") %>% 
    image_blur(20,10)

up_hand_shadow
```

<img src="/post/2019-08-03-making-hex-and-twittercard-with-bunny-and-magick_files/figure-html/unnamed-chunk-6-1.png" width="350" />

## Hex canvas

Time has come for us to put everything together. `bunny` has two functions for producing geometrical elements for a hex sticker: `image_canvas_hex()` and `image_canvas_hexborder()`. First function produces solid hex-shaped canvas with transparent backround and the second one produces only the border which can be placed on top of whatever image or text elements you will decide to add onto your sticker. This will make sure that even if the images or text end up over the edge of the sticker, it will be covered in solid color, when the hexagon frame is put on top of it. Think of `image_canvas_hexborder()` as "toilet seat sheet" which you use to cover the border to keep yourself clean and happy (sorry about the metaphora!). 

For the choice of the color palette, I typically use [coolors.co](https://coolors.co/). This website helps generate color palettes, which you can mix and match to arrive at aesthetically pleasing combinations. I want the background of my sticker to be light and the border to be dark to match to the color of the hands. Let me try some hex codes from the palette I found interesting.


```r
# https://coolors.co/000000-ede6f2-0d4448-c8c8c8-b3b3b3
hex_canvas <- image_canvas_hex(border_color="#0d4448", border_size = 2, fill_color = "#ede6f2")
hex_border <- image_canvas_hexborder(border_color="#0d4448", border_size = 4)
hex_canvas
```

<img src="/post/2019-08-03-making-hex-and-twittercard-with-bunny-and-magick_files/figure-html/unnamed-chunk-7-1.png" width="846" />

This looks easy. We are getting a hexagon of almost 1950 pixels in height. That's more than enought size for printing the 2x2 inch sticker (it is actually 2x1.73 inch, which you [can out find here](http://hexb.in/sticker.html)).

## Composing with gravity

Second set of useful functions is related to arranging images. `bunny` has recently gotten a wrapper over `magick::image_composite()`, which I decided to call `image_compose()`. The only difference is that my function has `gravity` argument that allows you to change origin and alignment of images on the page. `image_annotate()` already has `gravity` and it is such a life-saver! For example, you need to place an element in the center of the page. With `gravity` you can just say `gravity="center"` and never worry about calculating `offset` to make sure the center of your front-layer image matches the center of the background image. `offset` can still be used to finetune and adjust the location of the image in relation to the new `gravity` point. The only minor inconvenience is that this argument is called `offset` in `image_composite()`, while in `image_annotate()` it is called `location`.   

Here's the whole code of our hex sticker:


```r
img_hex <- hex_canvas %>% 
  bunny::image_compose(up_hand_shadow, offset="+40+460", gravity = "northwest")%>% 
  bunny::image_compose(down_hand_shadow, offset="+30+390", gravity = "southeast")%>% 
  bunny::image_compose(up_hand, offset="+20+440", gravity = "northwest")%>% 
  bunny::image_compose(down_hand, offset="+20+380", gravity = "southeast")%>% 
  image_annotate("bbox", size=450, gravity = "center", font = "Aller", color = "#0d4448") %>% 
  bunny::image_compose(hex_border, gravity = "center", operator = "Over")
img_hex
```

<img src="/post/2019-08-03-making-hex-and-twittercard-with-bunny-and-magick_files/figure-html/unnamed-chunk-8-1.png" width="846" />

First we place shadow of the upper hand, with `northwest` gravity (i.e. aligned against the upper left corner). Then comes bottom hand shadow, aligned against the bottom right corner. Next two layers are hands with respective offsets, slightly shifted against the shadows by some 20 pixels to make sure the shadow is visible. Note that `gravity` makes it easy to reason about the location of the elements on the canvas - it is counted in relation to the corners we are placing the hands in. Also note that I am using default compose operator, which is `Atop`. This operator is different from `Over` because it honors the transparency. If edges of the shadows or of the hands "hang over" the edge of the hexagon, they get trimmed (I like to think these parts drop and sink into the transparent background). Finally we place the word `bbox` in the center and "seal" the image with the "toilet seat" border. I am using operator `Over` to make sure that it covers the edges of my hex (and not drop off the cliff of transparency).

That's it! We save two copies of the image: one for StickerMule limited to 1200x1200 pixels (it will have to fit within this bounding box) and one for github (which can be used for showing on README page). Second image goes in to `man\figures` folder of your package and needs to be called `logo.png` to be picked up by, say, `packagedown`. 


```r
img_hex %>%
  image_scale("1200x1200") %>%
  image_write(here::here("input", "bbox_hex.png"), density = 600)

img_hex %>%
  image_scale("200x200") %>%
  image_write(here::here("input", "logo.png"), density = 600)
```

## Github card

By now you have probably heard and noticed that after many complaints from the users Github has made it possible to replace the the monstrous profile pictures with custom twittercards which can be uploaded to repo and used whereever the link to repository is posted on social media. Many R users have embraced the opportunity and produced some impressive visuals.

<!--html_preserve-->{{% tweet "1140601239627010051" %}}<!--/html_preserve-->

The image can be uploaded to a special sections inside your project. You can find it under Settings menu in your repo. This is what github says about the dimensions of the card.

> Images should be at least 640×320px (1280×640px for best display).

Well, `bunny` makes it easy to produce github cards, as well. First of all, let's scale down the newly created logo, which we will place onto the card. I also want to place github logo to indicate the name of the repository where the code will be stored. `bunny` includes logos of all three major `git` providers: github, gitlab and bitbucket. Just type `bunny::github` and get a png of the logo to be used in your image-manipulation projects.


```r
img_hex_gh <- img_hex %>%
  image_scale("400x400")

gh_logo <- bunny::github %>% 
  image_scale("50x50")
```

The main function for producing a card is `image_canvas_ghcard()`. It has all the useful defaults to be compliant with the github card size guidelines, so you only need to specify the background color. The card will automatically set aside the border and return only the "inner area" of the card available for your design experiments. You will have to complete your creative process with `image_border_ghcard()` which will add the border around your image to pad your masterpiece to recommended size.

The rest is easy - you place text and images using same `bunny::image_compose` and `magick::image_annotate` and arrange them on the page as you please. Wrap your work with the `image_border_ghcard()` and save the result.


```r
gh <- image_canvas_ghcard("#ede6f2") %>% 
  image_compose(img_hex_gh, gravity = "East", offset = "+100+0") %>% 
  image_annotate("Frame your world", gravity = "West", location = "+100-30", 
                 color="#0d4448", size=60, font="Aller", weight = 700) %>% 
  image_compose(gh_logo, gravity="West", offset = "+100+40") %>% 
  image_annotate("dmi3kno/bbox", gravity="West", location="+160+45", 
                 size=50, font="Ubuntu Mono") %>% 
  image_border_ghcard("#ede6f2")

gh
```

<img src="/post/2019-08-03-making-hex-and-twittercard-with-bunny-and-magick_files/figure-html/unnamed-chunk-12-1.png" width="640" />

```r
gh %>% 
  image_write(here::here("input", "bbox_ghcard.png"))
```

Your github card is ready! Upload it to repository and start wiring some code!
