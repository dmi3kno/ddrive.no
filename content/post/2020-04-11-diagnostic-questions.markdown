---
title: Diagnostic questions and a future of online education
author: Dmytro Perepolkin
date: '2020-04-11'
slug: diagnostic-questions
categories:
  - blog
tags:
  - education
  - bayesian
  - r
---




## Diagnostic questions

I am a big fan of [Greg Wilson's **Teaching Tech Together**](https://teachtogether.tech/) and I have to credit him for sparking my interest in what is called "diagnostic questions", often used in [formative assessments](https://teachtogether.tech/#s:models-formative-assessment). Diagnostic question is a carefully designed (quick to administer) multiple-choice question (MCQ) with one unambiguous correct answer and a number of plausible distractor (incorrect answers) with diagnostic power.

> A distractor is a wrong or less-than-best answer; “plausible” means that it looks like it could be right, while “diagnostic power” means that each of the distractors helps the teacher figure out what to explain next to that particular learner.

In other words, we don't just want random wrong answers, nor do we want to "trick" people with answers that look "almost right", but rather we want plausible answers that are able to surface misconceptions and provide teacher with some signals that certain (identifiable) parts of the material were misunderstood and may require repetition or another round of explaination. Here's what Greg recommends as a strategy for finding "good" wrong answers:

> In order to come up with plausible distractors, think about the questions your learners asked or problems they had the last time you taught this subject. If you haven’t taught it before, think about your own misconceptions, ask colleagues about their experiences, or look at the history of your field: if everyone misunderstood your subject in some way fifty years ago, the odds are that a lot of your learners will still misunderstand it that way today. You can also ask open-ended questions in class to collect misconceptions about material to be covered in a later class, or check question and answer sites like Quora or Stack Overflow to see what people learning the subject elsewhere are confused by.

Outside of data science education, the term ["diagnostic question"](https://medium.com/eedi/what-is-a-diagnostic-question-13bb85c64062) has been popularized by [Craig Barton](http://www.mrbartonmaths.com/blog/diagnostic-questions/), a passionate math teacher from UK, author of [several brilliant books](http://mrbartonmaths.com/books/), including highly-acclaimed [**How I Wish I'd Taught Maths: Lessons learned from research, conversations with experts, and 12 years of mistakes**](https://www.amazon.co.uk/gp/product/1911382497/) and a Head of Education at [Eedi](https://www.eedi.com) a company with a mission of improving teaching and learning world wide. Craig's [diagnosticquestions.com](https://diagnosticquestions.com/) collects and makes freely available thousands of carefully designed formative assessment questions for maths, sciences and computing (including Excel and Python). I very much recommend anyone with the interest in teaching STEM to register at the website and check out the questions contributed by educators from around the world. 

Craig shares [5 Golden Rules](https://medium.com/eedi/what-makes-a-good-diagnostic-question-b760a65e0320) for what makes a good diagnostic question:

1) They should be clear and unambiguous
2) They should test a single skill/concept
3) Students should be able to answer them in less than 10 seconds
4) You should learn something from each incorrect response without the student needing to explain
5) It is not possible to answer the question correctly whilst still holding a key misconception

## Example: Bayes Theorem

Many of us learned (although arguably a little bit too late) a Laplace conditional probability rule, known as [Bayes Theorem](https://en.wikipedia.org/wiki/Bayes%27_theorem) for discrete events.

![Bayes Theorem](https://upload.wikimedia.org/wikipedia/commons/thumb/1/18/Bayes%27_Theorem_MMB_01.jpg/1600px-Bayes%27_Theorem_MMB_01.jpg)

Although the formula looks "mathy" and even intimidating, the intuition is pretty straightforward and can be applied in a wide variety of situations, from [proving existence of God](https://qz.com/1315731/the-most-important-formula-in-data-science-was-first-used-to-prove-the-existence-of-god/) to [filtering spam emails](https://en.wikipedia.org/wiki/Naive_Bayes_spam_filtering) in your mailbox. There have been multiple attempts to explain "Bayes rule" in an intuitive way: some are [long and tedius](http://yudkowsky.net/rational/bayes), others are short and sweet like this [Bayes Theorem with Lego](https://www.countbayesie.com/blog/2015/2/18/bayes-theorem-with-lego). Bayes rule is a helpful tool in reasoning with conditional probabilities about the events that otherwise look confusing, like, for example, [Monty Hall problem](https://www.statisticshowto.com/probability-and-statistics/monty-hall-problem/).

More straightforward examples of applying Bayes rule involve reasoning about two related but subtly different distinctions: people who are genuinely sick and those testing postitive on a test. Although there's quite a strong temptation to conflate these distinctions, Bayes rule draws our attention to relevant details and helps us keep the reasonining straight.

>   Dangerous fires are rare (1%), but smoke is fairly common (10%) due to barbecues. 90% of dangerous fires make smoke. What is the probability that the fire is dangerous when there's a smoke?

Conventional way of solving these sort of tasks include recognizing that the data that is given is not what is being asked: smoke given fire, not fire given smoke. Once this is identified, another challenge is typically involving finding the value for denominator in the Bayesian formula (i.e. prevalence of smoke) because it is rarely given as straightforward as in the example above. Although rate of fires is usully given explicitly, rate of smoke is typically given indirectly (as rate of smoke in dangerous and beningn fires separately).

`$$P(Fire|Smoke)=\frac{P(Smoke|Fire)P(Fire)}{P(Smoke)}=\frac{90\%\times1\%}{10\%}=9\%$$`

Bayes rule is especially important when "base rate" for the two distinctions is different. There's even a documented [cognitive bias](https://en.wikipedia.org/wiki/Base_rate_fallacy) related to ignoring base rates. The problem can be partially alleviated by using Natural Frequencies in presentation, visualization and solving of problems involving two orthogonal distinctions (Gigerenzer & Hoffrage, 1999).

In the example above, it is important to realize that the problem can be written in the following format.


|         | Fire| &nbsp;&nbsp;No fire|
|:--------|----:|-------------------:|
|Smoke    |    9|                  90|
|No smoke |    1|                 900|

It takes some practice to generate these "natural counts" from the problem definition on the fly. In some cases it is possible to "imagine" the population of data points under consideration and "assign" them to relevant categories.

## Numeracy in surveys

I recently made visualizations for the results of a [questionnaire](https://www.cam.ac.uk/stories/wintoncovid1) by folks at University of Cambridge, which included a section testing numeracy of respondents. Data can be accessed [here](https://osf.io/jnu74/). 


```r
dt <- read_csv("https://osf.io/xubqt/download")

qs <- c("Num1", "Num2a", "Num2b", "Num3") 

q_df <- dt %>% 
  slice(1:2) %>% 
  tibble::rowid_to_column() %>% 
  pivot_longer(-rowid, names_to = "var", 
               values_to = "txt_en") %>% 
  filter(var %in% qs, rowid==1) %>% 
  select(-rowid)

data_df <- dt %>% 
  slice(-1:-3) %>% 
  tibble::rowid_to_column() %>% 
  pivot_longer(cols = GenSocTrust:Politics, 
               names_to = "var", values_to = "code") %>% 
  filter(var %in% qs) %>% 
  mutate(code=ifelse(as.numeric(code)<1, 
                     as.integer(as.numeric(code)*100), 
                     as.integer(code))) %>% 
  count(var, code) %>% 
  group_by(var) %>% 
  mutate(pct=round(n/sum(n),3)) %>% 
  ungroup()
```

## Mens choir

Here's a first question from [the survey](https://www.cam.ac.uk/stories/wintoncovid1) testing people's numeracy:

>Out of 1,000 people in a small town 500 are members of a choir.  Out of these 500 members in the choir 100 are men.  Out of the 500 inhabitants that are not in the choir 300 are men.    What is the probability that a randomly drawn man is a member of the choir?  Please indicate the probability in percent.    ____ %

This is the question that tests skills underlying application of Bayes Theorem. It requires the respondent to map numbers in the problem to the elements of the formula and perform a simple operation. Although it is possible to solve the problem with Bayes rule, there's a more intuitive approach which does not invoke math. It might useful to imagine a city divided into two groups of 500 people, out of which 100 and 300 are carved out, respectively. The final "natural count" table will look like this:


|                | Men| &nbsp;&nbsp;Not men| &nbsp;&nbsp;TOTAL|
|:---------------|---:|-------------------:|-----------------:|
|Choir           | 100|                 400|               500|
|Not choir&nbsp; | 300|                 200|               500|

Or in abbreviated notation (with ^ meaning *not*):


|                | Men| &nbsp;&nbsp;Not men| &nbsp;&nbsp;TOTAL|
|:---------------|---:|-------------------:|-----------------:|
|Choir&nbsp;     |  CM|                 C^M|                 C|
|Not choir&nbsp; | ^CM|                ^C^M|                ^C|

In this table it should be easy to see that the ratio we're after can be calculated from the first column: ratio of choir men to total men, which is 

`$$\frac{\text{CM}}{\text{(CM+^CM)}}=\frac{100}{100+300}=25\%$$`.

What "plausible distractors" can you think for this question? Let's look at the top answers from the actual results of the survey.


| Answer| &nbsp;&nbsp;Frequency|
|------:|---------------------:|
|     10|                 0.182|
|     25|                 0.181|
|     40|                 0.113|
|     20|                 0.111|
|     30|                 0.063|
|     50|                 0.055|
|      5|                 0.033|
|     60|                 0.023|
|     15|                 0.020|
|     33|                 0.016|

Here are some of the plausible but wrong answers to this problem:
- **10:** This would indicate the mistake in the denominator of the ratio above, wrongly dividing by the whole city population, not by men only `CM/ALL`
- **40:** This indicates that the person is calculating ratio of men (100+300) in total population (1000) `(CM+^CM)/ALL`. This is the right answer to the wrong question.
- **20:** Ratio of men singers to total choir members. This one is also a good guess in a sense that the numerator is correct (100) but denominator is wrong (500). `CM/C`
- **50:** This might mean "I don't know" or represent a ratio of all choir members to total population `C/ALL`
- **33:** This looks like `CM/^CM`, which of course, not the right answer, either.

Remarkably, many wrong answers involve incorrect denominator (erroneously dividing by total population or by total choir members) manifesting various wrong mental models. Let's have a look at another assignment.

## Bad mushrooms

>In a forest 20% of mushrooms are red, 50% brown and 30% white.  A red mushroom is poisonous with a probability of 20%.  A mushroom that is not red is poisonous with a probability of 5%.    What is the probability that a poisonous mushroom in the forest is red?    ____ %

This is "different on surface, same in depth" sort of question. A small distracting detail has been thrown in, regarding mushroom color variety. The trick is to disregard this information and collapse "non-red" mushrooms together. Another complication is that the task is formulated in terms of percentages, not absolute counts. Gigerenzer & Hoffrage (1999) predict that this should make it more difficult for people to properly perform Bayesian updating. Let's convert this problem to "natural counts" and adopt (similar) abbreviation:


|              | Poison| &nbsp;&nbsp;Not poison| &nbsp;&nbsp;TOTAL|
|:-------------|------:|----------------------:|-----------------:|
|Red           |     40|                    160|               200|
|Not red&nbsp; |     40|                    760|               800|

Or in abbreviated notation (with ^ meaning *not*)


|              | Poison| &nbsp;&nbsp;Not poison| &nbsp;&nbsp;TOTAL|
|:-------------|------:|----------------------:|-----------------:|
|Red&nbsp;     |     CM|                    C^M|                 C|
|Not red&nbsp; |    ^CM|                   ^C^M|                ^C|

Correct answer is of course

`$$\frac{\text{CM}}{\text{(CM+^CM)}}=\frac{40}{40+40}=50\%$$`.

Let's have a look at the top-10 answers to this question


| Answer| &nbsp;&nbsp;Frequency|
|------:|---------------------:|
|     20|                 0.022|
|      4|                 0.014|
|     50|                 0.012|
|     10|                 0.007|
|      5|                 0.006|
|     80|                 0.005|
|     25|                 0.005|
|     15|                 0.003|
|     95|                 0.003|
|     40|                 0.003|

As we can see, much worse response rate across all answer options. Questionnaire serves this question after three other numeracy questions and the response rate deteriorates with every question indicating significant fatigue. In addition to the "plausible distractor" patterns identified earlier, we have a few more interesting answers here:

- **20:** This one is simply repeating the same number given in the problem (`CM/C` or `C/ALL`). The same applies to **5, 80 and 95**. Those are simply reiteration of the numbers provided in the problem. 
- **4:** This pattern we've seen earlier: dividing by the whole mushroom population, not by poisonous mushrooms only `CM/ALL`
- **10:** No idea where 10 comes from. Let me know if you think of plausible explanation.
- **25:** Adding 20% and 5% together and not knowing what to do next.

Generally, it was probably not a good idea to include this problem for measuring numeracy. Response rate was less than 10% and correct answer was picked by a little over 1% of respondents.

## Five-sided die

>Imagine we are throwing a five-sided die 50 times.  On average, out of these 50 throws how many times would this five-sided die show an odd number (1, 3 or 5)?    ____ out of 50 throws.

Here top-5 results looked as follows:


| Answer| Percent|
|------:|-------:|
|     30|   0.251|
|     25|   0.148|
|      5|   0.060|
|      3|   0.046|
|     50|   0.043|
|     10|   0.043|
|     20|   0.033|
|     35|   0.024|
|     15|   0.019|
|     40|   0.016|

Here, along with correct answer (30), we find a few plausible disractors:
- **25:** This would indicate that the person is misunderstanding the denominator of the ratio above dividing 100 by 1000, not by 400 (number of choir men needs to be related to all men, not all people)
- **40:** This would indicate that the person is returning ratio of men (100+300) in population (1000). This is the right answer to the wrong question.
- **20:** This one is more difficult to grasp, but perhaps it is a difference between men outside and inside choir (300-100) to population (1000)?
- **30:** This must correspond to the share of non-singing men (300) in the city population (1000).

[SSDD problems](https://ssddproblems.com/)

# References

Gigerenzer, G., & Hoffrage, U. (1999). Overcoming difficulties in Bayesian reasoning: A reply to Lewis and Keren (1999) and Mellers and McGraw (1999). Psychological Review, 106(2), 425–430. https://doi.org/10.1037/0033-295X.106.2.425


