---
title: 'Miracles with magick and bunny'
author: dmi3kno
date: '2019-07-29'
slug: miracles-with-magick-and-bunny
categories:
  - blog
tags:
  - r
  - bunny
  - magick
---



Will Chase ([@W_R_Chase](https://twitter.com/W_R_Chase)) made quite a spash last week on twitter when he posted the front slide to his `ggplot2` presentation.

<!--html_preserve-->{{% tweet "1155212225621221376" %}}<!--/html_preserve-->

The image was so well done that it took me a while to realise the cover is not real. Very impressive!

As I was enjoying the picture, I kept thinking about the striking similarity of Will's work with the real-life examples of the Glamour magazine covers, to a large extent driven by that bold title across styled in perfect resemblence of the original. So I got curious which font does Glamour use for their magazine title? After some research I found out that the magazine uses [some variation of Franklin Gothic](https://uk.answers.yahoo.com/question/index?qid=20100121140821AA0lp0h) with custom kerning (the distance between the letters). Frankling Gothic costs money, but I thought, maybe, I can locate a font which is quite similar to the original, so that the user looking at the image won't notice the difference (at least not immediately)? As I was snooping aroung Google Fonts, I came across [Oswald](https://fonts.google.com/specimen/Oswald) which looks pretty close (although noticeably thinner). Well, how close?

## Matching the letters


```r
library(magick)
```

We will be doing all of it, of course, using `magick` - [awesome ImageMagick's Magick++ wrapper](https://github.com/ropensci/magick) developed by Jeroen Ooms and included in the ROpenSci package universe. Let's read in the image directly from Twitter and immediatly convert it to PNG (duh!).


```r
glam <- image_read("https://pbs.twimg.com/media/EAgjZJNWwAAf9gx?format=jpg") %>%
  image_convert(format = "PNG")
glam
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-2-1.png" width="360" />

Next, we'll fiddle a bit with the `location` argument in `image_annotate()` and try to fit at least a few letters on top of the existing. Note that I had to apply *negative* offset on y-axis to lift the text up to where the title is. The shape of the letter "G" is not ideal, "M" is quite a bit thinner and "L" is obviously closer to "G" than my font wants to place them. No wonder people mentioned custom [kerning](https://en.wikipedia.org/wiki/Kerning)!


```r
image_annotate(glam, "GLAM", font = "Oswald", weight = 700, size=180, location = "+10-63")
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-3-1.png" width="360" />

What would it take for me to move the letters a little bit closer together? Well, I guess I would have to place them one by one, which is annoying but doable. I can then position them using negative offset (or whatever it would take) and move them as close as they need to be to match at least the centroid of the letter. Well, wouldn't it be nice if `image_annotate()` had `kerning` argument?

## Introducing bunny: a magick helper!

Well, I had to put something together to help with this situation! Enter `bunny` a [small package](https://github.com/dmi3kno/bunny) of `magick` helper functions. Of course, I first had to design a hex sticker, but with that behind us, we can start making some helpful image processing tools. `bunny` is pretty independent type, loyal only to his `magick` master (i.e. it has no other dependencies).


```r
library(bunny)
```

As of today, `bunny` is only a few days old, but he's already got a tricks up his sleeve, one of which is information function similar in nature to `image_info()`, but for text. The function takes the string and a set of parameters you want to apply it with using `image_annotate()` (such as size, font, style, weight) and outputs a dataframe with one letter per row, showing width and height of every letter, as they would be printed individually, as well as kerning (distance between the letter) when the whole word/phrase is printed at once. 


```r
ii <- bunny::image_annotate_info(text="GLAMOUR", size=180, font="Oswald", weight=700)

ii
#>   letter width height kerning
#> 1      G   106    269       1
#> 2      L    81    269       0
#> 3      A   100    269      -1
#> 4      M   128    269      -1
#> 5      O   106    269      -1
#> 6      U   107    269      -1
#> 7      R   109    269      -1
```

You are supposed to read it like this: 

> "Letter G takes geometry of '106x269' followed by an extra 1 pixel when printed together with a next letter (L), so that letter L would start at geometry '+107+0'"

All of this is, of course assuming "northwest" gravity (i.e. assuming coordinates are counted from top-left corner, which is default in `magick`). Implementing the same with different gravities would be a nightmare I am not yet ready for (..maybe, one day..). Also with this bold font (`weight=700`), kerning is unimpressive. Try plotting the same text in regular (`weight=400`) lowercase and be amazed at how much work went into designing a nice-looking font!

So what you can do with it, is just take these `kerning` values (`dput()` or [`datapasta` them](https://github.com/MilesMcBain/datapasta)) and make your own version, which can be used for spacing out the letters to fit nicely over the originals. 


```r
ii$kerning <- c(-10, 2, 3, -3, -7, -13, 0)
letter_x_pos <- c(0, head(cumsum(ii[["width"]] + ii[["kerning"]]),-1))
```

Yeah, so here's my pre-tidyverse home-made `lag()`, which takes `c(0, head(..., -1))`. And inside of it is just cumulative sum of the letter widths together with respective kerning, so I get kerning-adjusted positioning of the **start** of every letter (hence the `lag`). 

Don't be scared, this took some trial and error, but there was no rocket science involved in arriving at these values, other than eyeballing how the letters would look if we changed them a bit and then ran together with the following code.


```r
canvas <- glam

for (i in seq_along(ii$letter)){
  canvas <- image_annotate(canvas, ii$letter[i], location = geometry_point(10+letter_x_pos[i], -63),
                           weight=700, size=180, font = "Oswald", color="purple")
}
canvas
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-6-1.png" width="360" />

Magick is composed of iteratively "layering" elements onto the image. Normally it uses pipe, so if you would want to plot each letter by hand it would take 7 identical commands that are varying by 1 letter. Also, because consequences of every step "accumulate" (i.e. letters are placed "on top of each other", even though they might not be overlapping), we can not use `apply` family of functions and need to resort to `accumulate` or a good-ol' `for-loop`. Unlike [@MilesMcBain](https://twitter.com/MilesMcBain) I did not take any loop-abstience pledges, so I am free to do what I will here.

Basically we need to take a copy of the `glam` (the original image) and start layering letters on it, one at a time (yes, @JennyBryan's ["row-oriented workflow"](https://github.com/jennybc/row-oriented-workflows)), overwriting the canvas on every iteration. I am adjusting the geometry by the same "+10-63" initial offset and specifying the same parameters I was using when measuring the kerning, plus the font color. You can argue this can be wrapped into a function, but I thought this is interesting enough to see spelled out, unlike the "white magick" happennig inside `image_annotate_info()` (the letters are printed on a white background to avoid accidental over-trimming).

## Title forgery

Ok, so this is all nice an good, but the letters are printed over Hadley's forehead, which is not very respectful, if you ask me ([some models](https://gl-images.condecdn.net/image/No1aXArRl0O/crop/405/f/covers_huda-rgb_p.jpg) allow this to be done to them, but I personally do not think it looks nice). So what can we do about it? `bunny` to the rescue!

So here are two small functions that allow you to place a dot onto a picture (to see where it visibly lands) and then recover color value from that pixel. Again, trial and error, but the objective has been to locate pretty much any point on the title, to sample its color (let me know if you want to train a sophisticated Machine Learning (pardon, AI!) model to segment Glamour maganize covers to locate the titles automatically!).


```r
# bunny::image_plot(glam, "+30+50")
pxcol <- bunny::image_getpixel(glam, "+30+50")
```

What follows is pretty crucial to the success of the whole operation. We want to create **an image mask**. [Image mask](https://www.quora.com/What-is-image-masking-1) is a black-and-white (B/W) image which "masks" non-essential elements and allows only some parts of the image to show. In order to make it, we can declare certain color of the picture "transparent" (which gets reflected on the "Opacity" layer of the image). Then we can recover that B/W image and manipulate it a bit to make sure it tightly covers all the things we want to hide in the image. In this case I am cropping top 180 pixels of the mask and ["dilating"](https://www.imagemagick.org/Usage/morphology/#dilate), which is like "pressing down" on it, so it equally "extends" to the sides. This is done to make sure I cover not only title letters, but also a few pixels around them (e.g. aliacing and compression region). I am using ["Diamond:2" kernel](https://www.imagemagick.org/Usage/morphology/#kernel), but there's a ton more defined in ImageMagick. The [morphology](https://www.imagemagick.org/Usage/morphology/) page has become my favorite and very much recommend you to bookmark it, if you are working with `magick`.

Last few lines remove the opacity (`matte=FALSE`) and invert the image. 

> What we need is a simple 2-color image  where *black* denotes things we want to "black"-list (hide) and *white* denotes the things we want to "allow".

The problem is that morphology functions (and a lot more cool stuff in `magick`) are operating on white-black images, i.e. on images where silhuettes are depicted in *white* and the background is *black*. So if you want to do successful masking, be prepared to work "on the dark side" and then come out to the light by inverting the final mask.


```r
iiw <- image_info(glam)$width
iih <- image_info(glam)$height

mask <- image_transparent(glam, pxcol, fuzz=25) %>%
  image_channel("Opacity") %>%
  image_crop(geometry_area(iiw, 180)) %>%
  image_morphology("Dilate", "Diamond:2") %>%
  image_convert(matte=FALSE) %>%
  image_negate()

mask
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-8-1.png" width="360" />

```r

mask_full <- bunny::image_canvas(glam, "white") %>%
  image_composite(mask)
```

Our mask has been only 180 pixels high. We need a full-page mask, which would cover everything. We manufacture blank image (using `bunny` convenience function, which makes `image_blank()` from specimen image), and place the mask on top of it. This is pretty simple, since no Opacity is involved (yet).

Here comes our second most important trick:

> Once you got a mask you can "apply" it to the colored image using `CopyOpacity` operator. It orders the original pixels which ended up undeneath the *black* regions of the mask to "loose color" (become transparent) and original pixels which ended up under the *white* regions, remain unchanged.

You may need to re-read it. Essentially your mask becomes an `Opacity` layer in the image you are applying it to. Yo! Listen again! **You can "manufacture" transparency with black/white images! How cool is that!**


```r
hadley <- image_composite(glam, mask_full, operator = "CopyOpacity") %>%
  image_flatten() %>%
  image_convert(matte=FALSE)
hadley
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-9-1.png" width="360" />

Vola! The title is gone! Now what?

We want to place our text *under* Hadley's head. So we need to "lift" his head and place the title under it, as simple as that! We will make a cut in the same place we did before (at 180 pixels from the North of the image) and store these parts of Hadle's body in separate variables. Note the use of `gravity` to locate the "bottom" of Hadley. You can also do it traditionally from "northwest" as shown in the comment below.


```r
hadley_head <- image_crop(hadley, geometry_area(iiw,180))
hadley_tail <- image_crop(hadley, geometry_area(iiw,iih-180), gravity="South")
# image_crop(hadley, geometry_area(720,932-180, 0, 180))
```

Remember we have been masking the letters. Now we need to mask everything else! 

>This mask is supposed to cover Hadley's head and letters shown in yellow, so that *the area above Hadley's head* would become transparent (showing the title we will place underneath), while his head needs to remain opaque, retaining its beautiful colors!

In this situation simple thresholding helps. We force "darker" colors become black, `close` the holes with `Diamonds`, tidy up the transparency and invert the mask. **Remember, it should be plain vanilla two-color bitmap.**


```r
hadley_head_mask <- hadley_head %>%
  image_channel("lightness") %>%
  image_threshold("black", "92%") %>%
  image_morphology("Close", "Diamond:2") %>%
  image_flatten() %>%
  image_convert(matte=FALSE) %>%
  image_negate()

hadley_head_mask
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-11-1.png" width="360" />

```r

hadley_head_transparent <- image_composite(hadley_head, hadley_head_mask, operator = "CopyOpacity")

hadley_head_transparent
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-11-2.png" width="360" />

Now the area over Hadley's head is transparent! Let's make a new canvas (the size of the image header we've been masking) and print our custom-kerned letters on it. Yeah, just like that: purple letters on white background!


```r
### copy same code ##############
canvas <- image_canvas(hadley_head_transparent, "white")

for (i in seq_along(ii$letter)){
  canvas <- image_annotate(canvas, ii$letter[i], location = geometry_point(10+letter_x_pos[i], -63),
                           weight=700, size=180, font = "Oswald", color="purple")
}
 canvas
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-12-1.png" width="360" />

Now we have all pieces and can "assemble" our masterpiece. We start with letters, place Hadley's head back and finally land both of these pieces over the original image (it will align against the Northwest, no adjustments needed!). We stiched the picture together, with no seam visible!


```r
hadley_head_lettered <- canvas %>%
  image_composite(hadley_head_transparent, operator = "Over")

glam %>%
  image_composite(hadley_head_lettered, operator = "Over")
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-13-1.png" width="360" />

Oh, well... I liked the magenta color better!

## More black (and white) magick!

Did you like our masking excercise? Let's practice some more! Here's an image of a real-life magician (likely from Russia, that's why he is not smiling) and his rabbit. We want to help the dude pull his main trick and make the rabbit disappear!


```r
fm <- image_read("https://img0.liveinternet.ru/images/attach/c/7/95/129/95129442_3640123_fokusnik.jpg") %>% 
  image_convert("PNG")

fm
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-14-1.png" width="250" />

The situation here is a little bit more grave, since the background color is not white. Also, rabbit color is not magenta, as the case was with Glamour title, but rather very bleak white. There's a big danger of background and foreground co-mingling!

We proceed carefully and pick the point of attack. Here we will be using another super-useful `magick` function called `image_fill()` which flood-fills the image with a given color taking into account `fuzz`, which is like uncertainty argument in relation to a color given in `refcolor` argument. In this case we are filling everything up to 25 fuzz-units from "white". The idea is to then "declare" the flood-color transparent and proceed as we did with the title above.


```r
# image_plot(fm,"+370+250", "red", pointsize = 5)

# this is as far as you can push it
image_fill(fm, "red", "+370+250", fuzz=25, refcolor = "white")
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-15-1.png" width="250" />

It's not bad, but the rabbit is not fully covered. If you increase the fuzz just a little bit, the paint "spills over" and you get the whole image covered. Not good! (Try it ~~at home~~ on your machine!)

OK, let's backtrack and re-consider our strategy. What happens is that "boundaries" of the rabbit are not very well defined. There's simply too much variation within the rabbit and too little contrast with the background. If we could only "highlight" the boundaries a little bit more, so our flooding would stop at the border and not spill into the background.

Luckily enough `magick` has [Canny filter](https://en.wikipedia.org/wiki/Canny_edge_detector), which takes the `geometry` argument and allows detecting (and highlighting) changes of color. It took a little fiddling with the geometry argument, but results look pretty decent. Yes, you *always* want to do things like that on grayscale images.


```r
image_convert(fm, type="Grayscale") %>% 
  image_canny("1x2+5%+15%")
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-16-1.png" width="250" />

Let's see if the edge is "sealing" enough to "hold ~~water~~ paint".


```r
image_convert(fm, type="Grayscale") %>% 
  image_canny("1x2+5%+15%") %>% 
  image_fill("red", "+370+250", fuzz=0, refcolor = "black")
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-17-1.png" width="250" />

Nope! Even with the fuzz of zero the paint is all over the place! Only the top-hat is dry, but we're not after it. Ok, we backtrack once again. Color-filling is not going to work until we "seal" all the small holes in the white lines. How do we do that?

Well, there's another morphology operation called `Close` which literally "closes" openings and holes. Exactly what we need! I first tried it with a small kernel (like `Diamond`) and then noticed something promising! Look at this big fat kernel I got here!


```r
image_convert(fm, type="Grayscale") %>% 
  image_canny("1x2+5%+15%") %>% 
  image_morphology("Close", "Disk:5")
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-18-1.png" width="250" />

It closed the holes and started filling the image. I couldn't be happier! Shall we help it with a little bucket of white paint?


```r
image_convert(fm, type="Grayscale") %>% 
  image_canny("1x2+5%+15%") %>% 
  image_morphology("Close", "Disk:5") %>% 
  image_fill("white", "+370+250", fuzz=20, refcolor = "black")
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-19-1.png" width="250" />

There you go! Awesome. But now our rabbit is "glued" to the top-hat. How to we separate it? Well, we need to "break" those thin lines tracing the edges of the top-hat. Another morphology operator to the rescue! Opposite of `Close` which "closes" or fills the holes, `Open` literally "opens" the holes, makes gaps and breaks parts of the image. Exactly what we need here! We take a thin `Diamond` kernel and make two cuts. Look how nicely the paint is contained within the boundaries of the rabbit!


```r
image_convert(fm, type="Grayscale") %>% 
  image_canny("1x2+5%+15%") %>% 
  image_morphology("Close", "Disk:5") %>% 
  image_fill("white", "+370+250", fuzz=20, refcolor = "black") %>%
  image_morphology("Open", "Diamond", 2) %>% 
  image_fill("red", "+370+250", fuzz=25, refcolor = "white")
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-20-1.png" width="250" />

PUR-R-R-FECT! You know the drill from here: declare transparent, grab `Opacity` channel, flatten and prepare for mating with the original image.


```r
bunny_mask <- image_convert(fm, type="Grayscale") %>% 
  image_canny("1x2+5%+15%") %>% 
  image_morphology("Close", "Disk:5") %>% 
  image_fill("white", "+370+250", fuzz=20, refcolor = "black") %>%
  image_morphology("Open", "Diamond", 2) %>% 
  image_fill("red", "+370+250", fuzz=25, refcolor = "white") %>% 
  image_transparent("red") %>% 
  image_channel("Opacity") %>% 
  image_convert(matte=FALSE) %>% 
  image_negate() 

bunny_gone <- image_composite(fm, bunny_mask, operator = "CopyOpacity")

bunny_gone
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-21-1.png" width="250" />

Bunny is gone. Just a white spot left! We need to cover it up! Let's sample the pixel nest to the outline of the bunny and use that color for filling up the hole. We blur the image a little bit to soften the edges and pour grey paint into it, blurring it a bit more at the end. 


```r
# bunny::image_plot(fm,"+400+215", "red", pointsize = 5)
grey_bg_color <- image_getpixel(fm, "+400+215")

blurred_background <-  bunny_gone %>% 
  image_flatten() %>% 
  image_convert(matte=FALSE) %>% 
  image_blur(8,2) %>% 
  image_fill(color=grey_bg_color, "+370+250", fuzz=25) %>% 
  image_blur(3,1)

blurred_background
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/figure-html/unnamed-chunk-22-1.png" width="250" />

That's it. Only the portion where bunny used to be will be visible, so we coun't care less that the rest of the picture got ruined. Place the magician with the rabbit-shaped hole over this background and save the two images as a GIF.


```r
nfm <- image_composite(blurred_background, bunny_gone, operator = "Over")
c(fm, nfm) %>% image_write_gif(here::here("input", "bunny_gone.gif"))
#> 
Frame 1 (50%)
Frame 2 (100%)
#> Finalizing encoding... done!
#> [1] "/home/dmi3kno/Projects/ddrive.no/input/bunny_gone.gif"
```

<img src="/post/2019-07-29-miracles-with-magick-and-bunny_files/bunny_gone.gif" alt="animated" width="300px"/>

