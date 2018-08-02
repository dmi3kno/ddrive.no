---
title: Reading vintage magazines with `hocr`
author: dmi3kno
date: '2018-08-01'
slug: reading-vintage-magazines-with-hocr
categories:
  - blog
tags:
  - r
  - hocr
---





```r
library(tidyverse)
library(tesseract)
library(pdftools)
library(hocr)
library(here)
library(fs)
library(hunspell)
library(hrbrthemes)
library(patchwork)
```

## Challenge

This post is inspired by recent tweet by Paige Bailey about vintage computer magazines made available for free download on archive.org. A number of people picked up on the idea of checking out some of the old magazines from the time they can remeber starting with computers.
<!--html_preserve-->{{% tweet "1022727838875639812" %}}<!--/html_preserve-->

I decided to pick up a magazine from 1986, probably around the time when I started getting intersted in computers and downloaded one issue of Compute Magazine from October 1986 to play with. Below is the link and the functions to download it.


```r
mag_url <- "https://archive.org/download/1986-10-compute-magazine/Compute_Issue_077_1986_Oct.pdf"

# download the file if not yet done so
go_get_it <- function(url, path){
  if(!dir.exists(here::here(path)))
    dir.create(here::here(path))
  
  if(!file.exists(here::here(path, fs::path_file(mag_url))))
    download.file(url, here::here(path, fs::path_file(url)), mode="wb")  
  return(here::here(path, fs::path_file(url)))
}

# convert the file if not yet done so
go_convert_it <- function(pdf_path, frmt, pg){
  new_path <- paste0(pdf_path,".",frmt)
  if(!file.exists(new_path))
    new_path <- pdftools::pdf_convert(pdf_path, format = frmt, 
                                 filenames=new_path, pages=pg, dpi=300)
  new_path
}

mag_pdf <- go_get_it(mag_url, "input")
mag_png <- go_convert_it(mag_pdf, "png", 2)
```

<img src="/post/2018-08-01-reading-vintage-magazines-with-hocr_files/figure-html/unnamed-chunk-3-1.png" width="300px" />

As you can see from this small thumbnail picture (trying to keep things easy for Hugo), the article talks pretty much about a [**fitbit**](https://www.fitbit.com/no/home) prototype  back in 1986. Pretty amazing that we have not invented anything drastically new, just managed to make the same things smaller. Our challenge is, however, not to build a health-tracking device. We want to meaningfully extract text from this (scanned) page and make it useable for analysis and reproduction.

Needless to say that if we do nothing, `tesseract` will just run across the page and spit out garbage. In current incarnation, `tesseract` is unable to respect text columns and even less so, recognize tabled data. Let's see if we can challenge that.

## OCR

We will use brand new `hocr` package. The package is intended as an add-on to `tesseract` (or, in the future, other systems able to produce hOCR-compliant output). 

> **hOCR** is an open standard of data representation for formatted text obtained from optical character recognition (OCR). The definition encodes text, style, layout information, recognition confidence metrics and other information using Extensible Markup Language (XML) in form of Hypertext Markup Language (HTML) or XHTML. 
>                                                         *Source: [Wikipedia](https://en.wikipedia.org/wiki/HOCR)*

`hocr` has a few useful [functions](https://github.com/dmi3kno/hocr) that allow parcing and tidying of XHTML data.  


```r
bodylink <- ocr(mag_png, HOCR = TRUE) %>% 
  hocr_parse() %>% 
  tidy_tesseract() %>% 
  separate_bbox_cols() 
bodylink
#> # A tibble: 859 x 41
#>    ocrx_word_id ocrx_word_bbox ocrx_word_conf ocrx_word_tag ocrx_word_value
#>           <int> <chr>                   <dbl> <chr>         <chr>          
#>  1            1 300 127 338 1~             26 text          "\u20187"      
#>  2            2 418 127 425 1~             17 text          7              
#>  3            3 489 143 510 1~             61 text          "\\"           
#>  4            4 844 128 1183 ~             35 text          "T\"-\\\\Q\u20~
#>  5            5 1319 175 1326~             50 text          /,             
#>  6            6 1377 131 1429~             27 text          "\u20197\":"   
#>  7            7 1575 130 1591~             38 text          "3\""          
#>  8            8 1607 130 1612~             48 text          "\u2019"       
#>  9            9 1693 130 1732~             28 text          "\u2019i"      
#> 10           10 1751 130 1754~             61 text          '              
#> # ... with 849 more rows, and 36 more variables: ocr_line_id <int>,
#> #   ocr_line_bbox <chr>, ocr_line_xbaseline <dbl>,
#> #   ocr_line_ybaseline <dbl>, ocr_line_xsize <dbl>,
#> #   ocr_line_xdescenders <dbl>, ocr_line_xascenders <dbl>,
#> #   ocr_par_id <int>, ocr_par_lang <chr>, ocr_par_bbox <chr>,
#> #   ocr_carea_id <int>, ocr_carea_bbox <chr>, ocr_page_id <int>,
#> #   ocr_page_image <chr>, ocr_page_bbox <chr>, ocr_page_no <dbl>,
#> #   ocrx_word_bbox_x1 <int>, ocrx_word_bbox_y1 <int>,
#> #   ocrx_word_bbox_x2 <int>, ocrx_word_bbox_y2 <int>,
#> #   ocr_line_bbox_x1 <int>, ocr_line_bbox_y1 <int>,
#> #   ocr_line_bbox_x2 <int>, ocr_line_bbox_y2 <int>, ocr_par_bbox_x1 <int>,
#> #   ocr_par_bbox_y1 <int>, ocr_par_bbox_x2 <int>, ocr_par_bbox_y2 <int>,
#> #   ocr_carea_bbox_x1 <int>, ocr_carea_bbox_y1 <int>,
#> #   ocr_carea_bbox_x2 <int>, ocr_carea_bbox_y2 <int>,
#> #   ocr_page_bbox_x1 <int>, ocr_page_bbox_y1 <int>,
#> #   ocr_page_bbox_x2 <int>, ocr_page_bbox_y2 <int>
```

If the last function in the above pipe looks new to you, it's because it is. The function was released [few hours ago](https://github.com/dmi3kno/hocr/blob/master/NEWS.md) and it acts pretty much like `tidyr::separate`, except it does not need as much guidance because it is limited to work with `bbox` columns. 

### What's in a `bbox`?

`bbox` is a special kind of column generated by hOCR engine that describes **bounding box** around the word, recognized by `tesseract`. It is typically a string of four integers separated by space (or comma, as in output produced by `tesseract::ocr_data`). Every entity in `hOCR` output has a `bbox`: a word, a line, a paragraph, a content area and, ultimately, a page. Here's an example of `bbox` for the page we just imported:


```r
bodylink$ocr_page_bbox[1]
#> [1] "0 0 2313 3230"
```

This means that the page extends from c(0,0) to c(2313, 3230), described as diagonal counting from **top left corner**. `separate_bbox_cols` parsed and added four additional columns for every `bbox` keeping consistent naming, such as `ocr_page_bbox_x1` and `ocr_page_bbox_y2`, which would descibe `x` and `y` coordinate for top-left and bottom-right limits of the document, respectively.

### Document structure and bbox

Looking closer at the picture above you can notice two distinct text columns. It comes to us almost naturally and we do not think twice about the fact that the text location on the page defines the sequence in which it should be consumed (read). For example, we start from the top and read "BODYLINK" then proceed to "CONVERTS YOUR COMMODORE 64/128 INTO A HEALTH AND FITNES SYSTEM". Then text suddently jumps left, but it poses no problem for human comprehension. In fact we percieve all the text that follows as organized into two distinct columns - left-aligned - with a clear separation (white space) between them. Even pictures are positioned to respect that layout, which signals that the text should no longer be read across that white gap, but rather it should start warping down - first in left column, followed by text in right column.

Ok, that's all nice and good for human eye, but can computers see this? It turns out, there's some hope, due to the fact that `tesseract` keeps track of the bounding boxes around the words it finds. Let's see what we can notice from the histogram of top-left word coordinates on horizontal axis (`ocrx_word_bbox_x1`).


```r
bodylink %>%
    ggplot()+
    geom_histogram(aes(x=ocrx_word_bbox_x1))+
  hrbrthemes::theme_ipsum_rc(grid_col = "gray90")
#> `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.
```

<img src="/post/2018-08-01-reading-vintage-magazines-with-hocr_files/figure-html/unnamed-chunk-6-1.png" width="672" />

Oh, we can already see some gaps and spikes! Looks promising. Given that we know about the presence of title, let's go down a few lines and look at the finer resolution of bins. 

How wide should our bins be? Well, we don't want our bins to be more narrow than a single symbol! How wide is that? We will take page width and divide it by character width of one character. Yeah, but we only know width per word! Well, majority of words have beed recognized for us by tesseract, so we can use that by number of characters per bbox.


```r
bodylink %>% 
  mutate(word_nchars=nchar(ocrx_word_value),
         word_bbox_width=ocrx_word_bbox_x2-ocrx_word_bbox_x1,) %>% 
  summarize(avg_nchar=mean(word_nchars), 
            avg_bbox_width=mean(word_bbox_width), 
            page_size=first(ocr_page_bbox_x2)) %>% 
  mutate(avg_char_width=avg_bbox_width/avg_nchar, avg_nchar_page=page_size/avg_char_width)
#> # A tibble: 1 x 5
#>   avg_nchar avg_bbox_width page_size avg_char_width avg_nchar_page
#>       <dbl>          <dbl>     <int>          <dbl>          <dbl>
#> 1      3.96           66.5      2313           16.8           138.
```

Which means that one symbol is around 16px wide and there are roughly 140 of them on the page. This, of course, does not take into account white spaces and margins, but all of that is relatively secondary to the objective we have at the moment, which is undestanding, how many bins we need to have in our updated histogram. I suggest we will be a bit conservative and set number of bins to be roughly 100, which corresponds to a bit more than a character.

## Histograms

Lets go down a few lines and look at the histogram again. This time we can plot both beginnings and the endings of our words (given that we have coordinates for opposite corners).


```r
bodylink %>%
  filter(ocr_line_id>20) %>% {
    ggplot(.)+
    geom_histogram(aes(x=ocrx_word_bbox_x1), bins=100) +
    labs(title="Word Beginnings") +# this `+` is from patchwork
      ggplot(.)+
      geom_histogram(aes(x=ocrx_word_bbox_x2), bins=100)+
      labs(title="Word Endings")
  } & theme_ipsum_rc(grid = FALSE)
```

<img src="/post/2018-08-01-reading-vintage-magazines-with-hocr_files/figure-html/unnamed-chunk-8-1.png" width="672" />

Here I am using somewhat unusual notation for plots. The `+` between the two plots is coming from awesome [`patchwork` package](https://github.com/thomasp85/patchwork) by amazing [Thomas Lin Pedersen](https://twitter.com/thomasp85/), author of ggraph/tidygraph, gganimate/tweenr and the famous R bindings to [lime](https://www.data-imaginist.com/2018/lime-v0-4-the-kitten-picture-edition/). Please, follow him on Twitter, read his [blog](https://www.data-imaginist.com) and support his awesome work on [Patreon](https://www.patreon.com/thomasp85).

The gaps are now more evident and the pattern is more pronounced in word beginnings. It makes sense, since the text columns are left-aligned. We can use the same settings to calculate the break points for bins using `base::hist()` function, which apparently is useful not only for producing quick plots for assessing variable distribution. `hist` produces a rugged list, which we explore by iterating over its elements 


```r
hist_obj <- bodylink %>%
  filter(ocr_line_id>20) %>%
  pull(ocrx_word_bbox_x1) %>%
  hist(breaks=100, plot=FALSE)

map(hist_obj, length)
#> $breaks
#> [1] 107
#> 
#> $counts
#> [1] 106
#> 
#> $density
#> [1] 106
#> 
#> $mids
#> [1] 106
#> 
#> $xname
#> [1] 1
#> 
#> $equidist
#> [1] 1
```

The number of breaks is one element longer than the number of counts (which makes sense, because breaks include both left and right boundary of the interval). We can drop first element and only look at the right limit of the interval. Let's look at word-beginning counts by bin. We are only interested in extremes (spikes and the valley).


```r
tibble(up_to=hist_obj$breaks[-1],
       counts=hist_obj$counts) %>%
  arrange(desc(counts)) %>% {
    list(head(.), tail(.))
  }
#> [[1]]
#> # A tibble: 6 x 2
#>   up_to counts
#>   <int>  <int>
#> 1  1200     29
#> 2   120     20
#> 3   100     14
#> 4   260     14
#> 5  1360     13
#> 6   740     11
#> 
#> [[2]]
#> # A tibble: 6 x 2
#>   up_to counts
#>   <int>  <int>
#> 1   940      0
#> 2  1120      0
#> 3  1140      0
#> 4  1160      0
#> 5  1180      0
#> 6  2180      0
```

So at 1200px the next column starts (we see a spike in counts) and the left column does not begin until 100px. Then it seems there's nothing between 1120px and 1180px and the right margin is likely at 2180px or so.

Lets create a dummy variable classifying the text into these two buckets. We will first mark beginning of the line and the propagate the column assignent to other words on the same line.


```r
bodylink_cn <- bodylink %>%
  mutate(col_num=case_when(
    ocr_line_id>20 & ocrx_word_bbox_x1 < 150 ~ 1L,
    ocr_line_id>20 & between(ocrx_word_bbox_x1, 1150, 1250) ~ 2L,
    TRUE ~ NA_integer_
  )) %>%
  group_by(ocr_line_id) %>%
  tidyr::fill(col_num)
```

What we're doing with the last two commands is we are grouping by line and filling in missing values, effectively spreading the column class until we hit the next column or the end of the line.

## ggplot v3.0.0 (feat. `tidyeval`)

Lets have a look at what we just produced. We will need to define a plotting function, similar to the one shown on `hocr` [README page](https://github.com/dmi3kno/hocr). I have updated by `ggplot` to ver3.0.0 and now can take full advantage of `rlang` and `tidy_eval` in my plotts. Read more about major changes implemented in latest version of in `ggplot` [here](https://www.tidyverse.org/articles/2018/07/ggplot2-3-0-0/).


```r
plot_bboxes <- function(df, x1, y1, x2, y2, clr, fll){
  x1q <- enquo(x1)
  y1q <- enquo(y1)
  x2q <- enquo(x2)
  y2q <- enquo(y2)
  clrq <- enquo(clr)
  fllq <- enquo(fll)

  df %>%
    ggplot(aes(xmin=!!x1q, ymin=!!y1q, xmax=!!x2q, ymax=!!y2q))+
    geom_rect(aes(fill=!!fllq, color=!!clrq), show.legend = FALSE)+
    scale_y_reverse()+
    theme_minimal()+
    theme(panel.grid = element_blank(),
          axis.text = element_text(size = 7),
          legend.text = element_text(size = 7),
          legend.title = element_text(size = 7))
}
```

Here we capture the variables that can be passed to our function by the user and quote them. The variables then get unquoted (for evaluation) where they are placed in respective layers in ggplot. Learn more about tidy evaluation [from Miles Mc Bain](https://milesmcbain.xyz/the-roots-of-quotation/), [Thomas Lumley](http://notstatschat.rbind.io/2018/07/30/quoting-and-macros-in-r/) and read the most recent addition by [Colin Fay](https://colinfay.me/lazyeval/).


```r
bodylink_cn %>%
  mutate(col_num=as.factor(col_num)) %>%
  plot_bboxes(ocrx_word_bbox_x1, ocrx_word_bbox_y1,
              ocrx_word_bbox_x2, ocrx_word_bbox_y2,
              fll=col_num, clr=col_num)
```

<img src="/post/2018-08-01-reading-vintage-magazines-with-hocr_files/figure-html/unnamed-chunk-13-1.png" width="672" />

This alone is a huge improvement over what we had before! As a small note, you have probably noticed some misclassified words on the wrong side of the page. This is because the horizontal lines (baseline) of two columns is not matching. This throws `tesseract` off and it recognizes some words as being on their own line, but on the wron side of the page. In this sense, the fact that `tesseract` has assigned words from two different columns to the same line almost does not mean anything. We would rather have is calculating [line metrics](http://www.how-ocr-works.com/images/resolution.html) for each column separately. Hmmm! Why not?

## Page slicing

Before we split the page in half vertically, perhaps it is worth revisiting our decision to go down a few lines (20 to be precise). We were lucky to hit very close to where the columns begin, but let's see if there's a better place where we could make a page cut and call everything above it a "page header" and everything below it a "text". We will use histograms again


```r
bodylink %>%
  filter(between(ocr_line_id, 18, 22)) %>%
  ggplot()+
  geom_histogram(aes(ocrx_word_bbox_y2))+
  theme_ipsum_rc(grid=FALSE)
#> `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.
```

<img src="/post/2018-08-01-reading-vintage-magazines-with-hocr_files/figure-html/unnamed-chunk-14-1.png" width="672" />

There seems to be an opening at about 1100px and another one a bit earlier at a little over 1000px. Lets remind outselves of the size of the document by looking at the bounding box for the page. `hocr` has a few helper functions, which can facilitate operations with bounding boxes. In particular, converting `bbox` string to [magick-compliant `geometry`](https://github.com/ropensci/magick) or slicing the bounding boxes vertically and horizontally. 


```r
# this is out page size
(page_size <- bodylink$ocr_page_bbox[1])
#> [1] "0 0 2313 3230"
# this is how the same looks in magick package
bbox_to_geometry(page_size)
#> [1] "2313x3230+0+0"
# this is how you can slice the page horizontally (along y axis)
bbox_slice_y(page_size, 121)
#> [[1]]
#> [[1]]$top
#> [1] "0 0 2313 120"
#> 
#> [[1]]$bottom
#> [1] "0 121 2313 3230"
```

The bbox slicer functions produce a list of geometries for each piece left after slicing. Lets slice up our page horizontally, at about 1100px and then vertically (only bottom half of the page) at the "value" (around 1150 px).


```r
slices <- lst(
  top = bbox_slice_y(page_size, 1100)[[1]]$top,
  bottom = bbox_slice_y(page_size, 1100)[[1]]$bottom,
  bottom_left = bbox_slice_x(bottom, 1160)[[1]]$left,
  bottom_right = bbox_slice_x(bottom, 1160)[[1]]$right)

slices
#> $top
#> [1] "0 0 2313 1099"
#> 
#> $bottom
#> [1] "0 1100 2313 3230"
#> 
#> $bottom_left
#> [1] "0 1100 1159 3230"
#> 
#> $bottom_right
#> [1] "1160 1100 2313 3230"
```

We will now define a small helper function that can slice the images for us (using `magick::image_crop`). We will pass it a file name and a geometry we want it to cut for us. `magick` will read the image and return a portion of it as a bitmap (it is actually returning a C++ pointer, dressed as R object of class `magick-image`, but [Jeroen Ooms can tell you much more about it](https://cran.r-project.org/web/packages/magick/vignettes/intro.html)). **The package is a goldmine** and I really recommend anyone diving into some of its super powerful functions.


```r
image_crop <- function(geom, path){
  magick::image_read(path) %>%
    magick::image_crop(geom)
}
```

## Lists, lists, lists

Looking at our list of slices, we really need only three pieces: top and bottom_left/bottom_right. How do we elegantly "select" from the list by name (like you would do with `select(-bottom)`)? Apparently there's no simple answer to this. You can use `magrittr::extract`, which is equivalent to base R `[`, but there's no real equivalent to `purrr::pluck`, which has gracious missing value handler using `.default` argument. Read more about it in this closed, but not entirely forgotten [issue on github](https://github.com/tidyverse/purrr/issues/223). The simplest hack is to select by position (which is fine when we have total of 4 elements, but probaly not an option when deal with complex nested lists).


```r
slices134 <- slices[c(1,3,4)]

imgs <- slices134 %>%
  map(bbox_to_geometry) %>%
  map(image_crop, mag_png)
```

Ha! List of images! Maybe we could produce a list of `hocr` objects? We iterate over the list of images with `purrr::map` and send each one of these images through the familiar `hocr` pipeline. The feature of sending `magick`-images directly to ocr has been released only few days ago, so please install latest and greatest versions of both packages with `remotes::install_github("ropensci/tesseract)` and `remotes::install_github("ropensci/magick")`. 


```r
txt_lst <- map(imgs, ~ocr(.x, HOCR = TRUE) %>%
                 hocr_parse() %>%
                 tidy_tesseract() %>%
                 separate_bbox_cols())
```

There's one more cool feature we could introduce to our dataset. What if we would be able to judge the quaility of OCR, not only by `tesseract` own feelings about it - a confidence index, but more objectively, such as, for example, whether the word is misspelled or not? There are several packages that can automate spell checking process for you, ranging from Peter Norvig's famous [two-liner](http://www.sumsar.net/blog/2014/12/peter-norvigs-spell-checker-in-two-lines-of-r/) to robust and rich toolkits in [`spelling`](https://ropensci.org/technotes/2017/09/07/spelling-release/) and [`hunspell`](https://www.opencpu.org/posts/hunspell-1-2/). We will use the latter, which has a simple `hunspell::hunspell_check` returning a logical vector of correct/incorrect. Lets add that feature to our dataset and use it in plots later on.


```r
txt_lst_spell <- txt_lst %>%
  map(~.x %>% 
        mutate(hun_spell=hunspell_check(ocrx_word_value)))

plot_lst <- map(txt_lst_spell,
                ~plot_bboxes(.x, ocrx_word_bbox_x1, ocrx_word_bbox_y1,
                                 ocrx_word_bbox_x2, ocrx_word_bbox_y2,
                                 fll=ocrx_word_conf, clr = hun_spell))
```

Oh, look, a list of plots! With `purrr`, sky is the limit! We can plot them using amazing `patchwork syntax`


```r
(plot_lst[[1]]) / (plot_lst[[2]] | plot_lst[[3]])
```

<img src="/post/2018-08-01-reading-vintage-magazines-with-hocr_files/figure-html/unnamed-chunk-21-1.png" width="672" />

There are some misspellings in the columns, but it is definitely something we can work with!

## Advanced image processing

We did not pay any attention to image preparation before sending it to OCR. Usually it is a very good idea and things can improve dramatically after image pre-processing, especially if text is located on noisy background. Here we are dealing with nearly ideal page, which hardly needs any enhancement to be readable.


One thing you can't help but notice though, is that `tesseract` is trying very hard to find words in the page header. It seems to notice the edges of the page title almost tracing edges of the huge letters on top of the page. It seems `tesseract` has hard time adjusting for drastic change in font size in the top part of the page, so it basically picks up bits and pieces, intepreting primarily noise. What can we do about it? Well, first of all, could it be that the letters are simply too big for `tesseract` to see? Could it also be that they are not high-contrast enough (written in light blue, remember) and `tesseract` has hard time recongnizing the edges of the letters? Let's see how much things can improve if we apply simple, but very effective trick in pre-OCR image preparation. We take only the top portion of the image.


```r
imgs$top %>%
  magick::image_scale("25%") %>% 
  magick::image_negate() %>%
  magick::image_lat() %>%
  magick::image_negate() %>%
  ocr(HOCR=TRUE) %>%
  hocr_parse() %>%
  tidy_tesseract() %>%
  separate_bbox_cols() %>%
  mutate(hun_spell=hunspell::hunspell_check(ocrx_word_value)) %>% 
  plot_bboxes(ocrx_word_bbox_x1, ocrx_word_bbox_y1,
              ocrx_word_bbox_x2, ocrx_word_bbox_y2,
              fll=ocrx_word_conf, clr=hun_spell)
```

<img src="/post/2018-08-01-reading-vintage-magazines-with-hocr_files/figure-html/unnamed-chunk-22-1.png" width="672" />
Here we downscaled the image 4 times and applied "Local Adaptive Thresholding", a feature, again, available only in development version of `magick`. `tesseract` gave up trying to recognize the title reading "BODYLINK" for us, but it seems like it has picked up the subtitle and even made some attempt at recognizing the lady's face in the picture (just kidding!). Please, take this script for a spin and see how you can improve OCR quality by preparing the image and serving it to `hocr` in digestable pieces.

## Conclusion and final thoughts.

What have we learned today? Well:

- we looked closer at what's avalable in the brand-new [`hocr` package](https://github.com/dmi3kno/hocr). 

- We played around with histograms as a tool to "chop up" (or as `hocr` calls it "slice") the page into pieces. 

- We played enough with lists and marveled at the limits and possibilities of [`purrr`](https://jennybc.github.io/purrr-tutorial/). 

- We were the fascinated by the awesome work of Thomas Lin Pedersen and his [`patchwork` package](https://github.com/thomasp85/patchwork), which makes previously tedious work of combining different `ggplot` charts a breeze.

- We took a peek at image pre-processing and the treasures hiding behind seemingly simple functions in [`magick`](https://github.com/ropensci/magick)

Finally I want to leave you with this idea: Perhaps we could slice the page even more (iteratively), and store smaller pieces and sub-chunks in a nested list: whenever you split a section in two, you record what you have done and save down the pieces as children to current node. Each piece might need its own scaling and pre-processing, but at the end you might have a pretty good shot at individual paragraphs, lines and even words, which need to be then iteratevly combined (assembled) back into text that would make sense as a whole. 

Perhaps slicing horizontally is a safer bet and you will more often than not hit enough histogram-valleys to cut through. Then you analyze adjacent slices and look at the edge word alignments (as we did with columns). 

We did not even start looking at contiguousness of the lines to be recognized as belonging to single column/paragraph. There are inserts in this text both on the left and on the righ, which could be recognized and probably fairly confidently separated from neighboring images by a number of metrics, including text density and amount of non-text populated space. 

This text was super easy, but even here we found some words misspelled, right in the middle of the text. There's another package cooking at the moment wich will deal with confusable characters and offer a toolkit for string distance calculation, smart matching and auto-correction.

Thats it for today! Thank you for hanging it with me! Please, stay tuned for updates!
