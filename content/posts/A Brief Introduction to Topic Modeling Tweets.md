+++
date = "2019-06-21"
title = "A Brief Introduction to Topic Modeling Tweets"
slug = "topic-modeling-brief-intro"
tags = []
categories = []
+++

In this brief tutorial I aim to demonstrate an application of topic
modeling for the recent *state of the nation address* (SONA) in South
Africa (i.e., the state of the union address by the president for
non-South Africans) by looking at a sample of tweets produced before,
during, and after the event.

What is Topic Modeling?
=======================

Topic modeling is one of many approaches used to extract meaning from
textual data. As a point of departure, this approach assumes that words
assume different meanings based upon their use in relation to other
words in a body of text. The meaaning of a word is contingent on the
broader context in which it is used. Topic modeling uses a bag-of-words
approach to achieve this. In contrast to other clustering approaches
(e.g., k-means clustering), topic modeling assigns, each document a
probability of beling to a topic, meaning that a document can belong to
more than a single topic. This probaability is determine through an
iterative Bayesian process (i.e., documents are initially assigned a
random probability of belonging to a topic and the probabilities become
increasingly accurate as more data are processed).

The most common form of topic modeling is \* Latent Dirichlet
Allocation\* (LDA) which involves the following steps:

1.  The researcher specifies a value for *k*, the number of topics in
    the corpus. Later, I briefly describe a process for determining *k*.

2.  Each word in the corpus is randomly assigned to one of the *k*
    topics.

3.  In an iterative process these topic assignments are updated by
    updating the prevalence of the word across the k topics, as well as
    the prevalence of the topics in the document.

This process produces two forms of output:

-   We can identify the words that are most frequently associated with
    each of the topics (beta). This will be demonstrated in this
    tutorial.
-   We can also identify the probability that each document is
    associated with each of the topics (i.e., which tweets correspond to
    which topics) (gamma).

Some brief setup
================

First, before we can do anything, we need to load some packages to
enable us to easily do what we want to do -
[pacman](https://cran.r-project.org/web/packages/pacman/index.html) is a
useful package to manage this process. We’ll need a number of general
purpose packages (e.g., tidyverse, dplyr, gpplot2, etc.) as well as some
specific packages for text analysis (e.g., tidytext, snowballC,
topicmodels, tm, ldatuning)

    pacman::p_load(tidyverse, dplyr, rtweet, ggplot2, tidytext, SnowballC, stringr, topicmodels, tm, ldatuning)

Next, we need to link up with twitter. For this, you will need to setup
an account, an API key and an app on the
[twitterdeveloper](https://developer.twitter.com) page. I have
purposefully removed my keys and tokens but this is easy enough to
figure out on your own. I then ran the following code to search for
relevant tweets.

To keep it simple I am going to use [rtweet](https://rtweet.info) to
search for a single hashtag - \#SONA2019 and exclude all retweets. It is
important to be aware that there are a number of restrictions on use of
the twitter API. While most are not relevant for our purposes, for the
searching of tweets, users are limited to 18000 tweets per 15 minutes.
Here I have indicated that, if the limit is hit, the search should try
again and get more. I don’t expect it to need to do this though for this
event. Additionally, the twitter index only includes between 6 and 9
days of tweets, so we won’t be finding anything too old here.

    raw_tweets <- search_tweets(q = "#SONA2019", include_rts = FALSE, retryonratelimit = TRUE)

    save_as_csv(raw_tweets, "SONA2019_tweets.csv")

This search resulted in 17993 tweets being retrieved. The data is messy,
lets just take a look at some of the tweets we have collected. For each
of these tweets we have 90 variables to consider - things like the
tweeter’s handle, the number of retweets, likes, and whether it is a
retweet or not. For our analysis we don’t need most of these fields. We
are really only interested in the text of the tweets for now.

Looks like there is something there, but there are also lots of elements
that won’t be useful for our analysis. We need to do some pre-processing
and cleaning of our dataset. First, lets use the
[tidytext](https://cran.r-project.org/package=tidytext) package to
select only the variables we want and turn our data into a
[tidy](https://en.wikipedia.org/wiki/Tidy_data) format. We are also
using the `anti_join` function to join this with the `stop_words`
dataset and remove these words (the, is, etc.)

I am also loading the stop\_words dataset into our environment from the
[tidytext](https://cran.r-project.org/package=tidytext) package so we
can use this later in our pre-processing of the tweets.

    data("stop_words")
    raw_tweets<-read_twitter_csv("SONA2019_tweets.csv")

    tidy_tweets <- raw_tweets %>%
      select(created_at,text) %>%
      unnest_tokens("word", text) %>%
      anti_join(stop_words)

    ## Joining, by = "word"

Now lets take a look at what our dataset looks like.

    tidy_top_tweets<-
       tidy_tweets %>%
            count(word) %>%
            arrange(desc(n))

    top_20<-tidy_top_tweets[1:20,]

    ggplot(top_20, aes(x=reorder(word, -n), y=n))+
      geom_bar(stat="identity")+
      theme_minimal()+
      scale_y_continuous(expand = c(0, 0), limits = c(0, 20000))+
      theme(axis.text.x = element_text(angle = -45, hjust = 0),
            panel.border = element_blank(),
            axis.line = element_line(colour = "black")) +
      labs(title="State of the nation address 2019 on Twitter",
           subtitle="Tweets containing #SONA2019",
           y = "frequency of word",
           x = "word") +
      guides(fill=FALSE)

![](/images/Topic-Modeling_files/figure-markdown_strict/top-tweets-1-1.png)

That’s not very informative. There is still a lot of junk in our
dataset. Also, there are a number of terms that will appear in nearly
every tweet. These probably won’t be too useful either. Let’s do some
more pre-processing

    tidy_tweets<-tidy_tweets[-grep("\\b\\d+\\b", tidy_tweets$word),]
    tidy_tweets<-tidy_tweets[-grep("http|https|t.co|rt|\\s+", tidy_tweets$word),]
    tidy_tweets<-tidy_tweets[-grep("sona2019|ramaphosa|cyril|cyrilramaphosa|president|speech|south|africa|sona", tidy_tweets$word),]

Now let’s take a look at the dataset and see if it is any better.

    tidy_top_tweets<-
       tidy_tweets %>%
            count(word) %>%
            arrange(desc(n))

    top_20<-tidy_top_tweets[1:20,]

    ggplot(top_20, aes(x=reorder(word, -n), y=n))+
      geom_bar(stat="identity")+
      theme_minimal()+
      scale_y_continuous(expand = c(0, 0), limits = c(0, 1200))+
      theme(axis.text.x = element_text(angle = -45, hjust = 0),
            panel.border = element_blank(),
            axis.line = element_line(colour = "black")) +
      labs(title="State of the nation address 2019 on Twitter",
           subtitle="Tweets containing #SONA2019",
           y = "frequency of word",
           x = "word") +
      guides(fill=FALSE)

![](/images/Topic-Modeling_files/figure-markdown_strict/top-tweets-2-1.png)

That’s better! Now, we could keep pruning and cleaning our dataset until
our hannds go numb, but for now, this will probably do.

The next thing we need to do is create a document-term-matrix. This is a
matrix where:

-   each row represents a single document (a tweet)
-   each column represents a word (term)
-   each value contains the number of appearances of that word (term) in
    that document

<!-- -->

    tidy_DTM<-
      tidy_tweets %>%
      count(created_at, word) %>%
      cast_dtm(created_at, word, n)

Before we can do our topic modeling we need to determine how many topics
are optimal for our dataset. For this we can use the `FindTopicsNumber`
function and see what makes sense for our dataset. There are a number of
parameters that can be tweaked here, but these are generally accepted.

*Warning* depending on the dataset and your machine, this may take quite
a while to compute (upto several hours in some cases).

    result <- FindTopicsNumber(dtm = tidy_DTM , topics = seq(from = 2, to = 40, by = 2),
                               metrics = c("Griffiths2004", "CaoJuan2009", "Arun2010"),
                               method = "Gibbs",
                               control = list(seed = 666),
                               mc.cores = 2L,
                               verbose = TRUE)

Now lets visualise this

    FindTopicsNumber_plot(result)

![](/images/Topic-Modeling_files/figure-markdown_strict/depict-figure-1.png)

It looks like 22 is a good number of topics to go with. Now that we know
this, we can produce our topic model as follows:

    topic_model <- LDA(tidy_DTM, k=22, method = "Gibbs", control=list(alpha = 0.1, seed=456))

    topics <- tidy(topic_model, matrix="beta")

Let’s visualise this and see what the topics look like.

    top_terms <- topics %>%
      group_by(topic) %>%
      top_n(10, beta) %>%
      ungroup() %>%
      arrange(topic, -beta)

    top_terms %>%
      mutate(term = reorder(term, beta)) %>%
      ggplot(aes(term, beta, fill=factor(topic))) +
      geom_col(show.legend = FALSE) +
      facet_wrap(~topic, scales="free", ncol = 2) +
      coord_flip()

![](/images/Topic-Modeling_files/figure-markdown_strict/create_topics_plot-1.png)

What are these topics?

Now that we have identified some topics in the dataset, we need to
interpret them. Importantly, topic models are not a replacement for
human interpretation. Rather, they are a tool to augment human
interpretation, to help make educated guesses about the latent themes or
topics in a dataset.

Looking at these topics, a couple of ideas come to mind:

1.  “economic issues”
2.  “Zuma and the EFF”
3.  “Steve Hofmeyr interview comments”
4.  “Reserve bank mandate”
5.  “jobs issues”

…and so on…

What’s next?
============

Now this was just a very brief analysis of the tweets. We can take this
much further and dig much deeper into the data. The first thing I would
recomend is to us `udpipe` and pull the nouns and adjectives from the
text and see if this improves the topics. Aother anlaysis to consider is
to use the `gamma` estimate instead of the `beta` estimate. This will
enable us to see document-topic relationships i.e., the topics contained
in particular tweets and go from there. Then, the next step would be to
include other contextual information using `structural topic models`.
For this, the [stm](https://cran.r-project.org/package=stm) package is a
good start.
