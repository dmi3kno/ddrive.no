---
title: Rant about dependencies
author: dmi3kno
date: '2019-08-03'
slug: rant-about-dependencies
categories:
  - blog
tags:
  - r
---

## Modularity

Dependencies in R are direct consequence of modularity. [#tinyverse](http://www.tinyverse.org/) will have to evolve as collection of few completely independent packages. Reducing dependencies means bundling more functions (especially utility functions) into one package. Once you did that, the benefits of continuing working on a well-tooled package vs. writing (and maintaining) a new copy of toolkit are completely not in favour of maverick developers. 

As a community, R is much more fragmented and people, arguably, do many more different things with it, so that centralization and mega-packaging is in many cases difficult. Many R package developers are part-timers, doing software development on the side when they have some time away from doing science (or whatever else they are doing). We as community lack professional software development capabilities to maintain humongous package projects. I will probably not overstate much if I say that compelexity of `dplyr` and `ggplot` have already outgrown single person's ability to keep track of everything at once. That said, I have enormous respect for people like [Romain](https://twitter.com/romain_francois) and [Thomas](https://twitter.com/thomasp85) who have god-like abilities to see forest-for-the-trees in those monstrous projects.

## Development of the language

One of the solutions to modularization (fragmentation) introduced by packages is a more rapid develpment of the base language itself. For example, have Python not developed as fast as it does, we would have seen many more dependencies in a typical Python project, extending basic types and evolving the syntax. We may dislike rapid changes in the language and shrug at co-existence of several competing syntax paradigms in Python (2.7 vs 3.7 until recently), but it is exactly what we are experiencing today with base vs tidyverse, where base R is at version 3.6 and `tidyverse` has bent the syntax so much that it is almost 4.7 by now. That said, `tidyverse` is not adopted by the R Core as the future of R and is unlikely to ever be. I am arguing that we have to be tolerant to co-existence of DSLs in R, since the base language is developing relatively slow and, until today, has prioritized backward compatibility over innovation (and I am not arguing it is a bad decision!). 

One more contributing factor is that R comes from Lisp-family, where inventing syntax and redefinig fundamental types is super easy. In that sence R has a long-term legacy of anarchy and therefore instilling discipline and compliance in this community is much more difficult. I sometimes joke that the extreme German ability to organize themselves and life around them comes partially from their strict, almost stoic language.

## Infrastructure

The R language has grown as much as it did due to CRAN, which introduced discipline and 
*[infrastructure for maintaining]* coherence in the package eco-system. The day when CRAN will be forked will mark the dawn of the language (at least in the form we know if). CRAN is R's enormous competitive advantage and had Python had CRAN we would have had to eat the dust. Python is crippled by its inability to manage dependencies and we can not laugh at Python virtual environments and complain about CRAN policies at the same time.

## Human-centric design

Finally there's a tradeoff between generality and user experience, evident everywhere from database design to shoes. You can design perfect products (database schema, most beautiful shoes), but they would be extremely uncomfortable to use (wear). Big generic packages shift a burden of prior knowledge onto the user. User is now responsible for knowing that images are essentially multi-dimensional arrays, twitter is a graph and animals species are recurrent lists. Now in order to work with biology, analyse tweets or apply filters to Instagram pictures you need to know linear algebra, graph theory and 1500 other things from university-level math. The (questionable) advantage of it is that you can do all of it in one package. 

R's variety and redundancy is its biggest blessing in onboarding newcomers and I cherish and value recent R community focus on creating human-centric APIs and intuitive interfaces. All of this comes at a cost of breaking changes and/or exploding number of packages which (standing on each other's shoulders) try to improve and facilitate better user experience. Many of these efforts are misguided (there's a better and more generic way of doing things) but the cost of failure is nearly non-existent and therefore we have ["package in 20 minutes"](https://resources.rstudio.com/rstudio-conf-2018/you-can-make-a-package-in-20-minutes-jim-hester) blockbuster videos. 

BTW, despite much higher popularity, you will find significantly fewer people in Python community who have written a package or had to reason about maintanable codebase (or even know what reverse dependicies are). In R we are advocating for modularisation already after writing second function (ref [`golem`](https://rtask.thinkr.fr/building-a-shiny-app-as-a-package/) and [Rmd-first](https://rtask.thinkr.fr/when-development-starts-with-documentation/) approaches popularized by ThinkR). 

I was recently amazed by complacency and ability to accept obviously *bs* solutions proposed on Microsoft Power Suite (PowerBI, PowerApps, Flow) user forums! Unempowered users are extremely innovative in finding workarounds, but completely unable to think critically about the fundamental flaws of the tools they are working with. There are some horrible patches proposed in the proprietary software forums, because even most capable users are completely unable to affect the development of the language or framework. By contrast, in R everyone is empowered (and encouraged!) to fork and make her/his own better version of toolkit she/he is using, publish it on CRAN, be certain that it will run not only on her/his own machine and that it will be treated with respect and will be given  equal distribution (and even some free marketing), along with competing product from for-profit organization(s).

## Final thoughts

R has many quirks, and many things we love to complain about, but it is **as free of a community as I can possibly imagine**, a libertarian dream, if you will, where members are encouraged and empowered, and where the language becomes stronger, not weaker, from introduction of competing ideas and frameworks. We should all cherish and support the "backbone" of our community, CRAN, respect and support the effort that goes into maintaining backward compatibility (salute, RCore!) without which we would have quickly descended into anarchy and complete chaos. 

All of this is merely my personal opinion, of course.
