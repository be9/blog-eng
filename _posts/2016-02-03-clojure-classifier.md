---
layout: post
title:  "A Bayesian classifier in Clojure. Part 1"
date:   2016-02-03
comments: true
excerpt: Sometimes you need to classify incoming data. In most interesting cases you have to use machine learning for that. In this series of posts, we'll see how Clojure can be used to build a classifier microservice. The first post goes into details of statistical classification theory and doesn't even mention Clojure (yet!).
---

## Intro

Sometimes you need to classify incoming data. Consider yourself lucky if this
can be done deterministically. In most interesting cases you have to use machine
learning for that. There's been a prominent progress in this field lately: you
might've already heard about
[deep learning](https://en.wikipedia.org/wiki/Deep_learning),
[ImageNet](http://image-net.org/), and friends. On the other hand, not all
classification problems are related to images. Not every problem needs to be
solved with a mysterious artificial neural network, be it shallow or deep. There are
plenty "old-school" machine learning methods applicable to real-world problems.

While working for [Shuttlerock](https://www.shuttlerock.com), I had to solve one of such
problems. Here's the story behind it.

Imagine that you're a fan of sports and there's a local stadium in your city
where sports, well, take place. People come there, practice, watch others and,
most importantly, post to Instagram! You want to create a live wall of Instagram
posts related to the stadium, for that would be very motivational and just fun.

![Stadium](/assets/images/clojure-classifier/stadium.jpg)

Since there's no default hashtag for the things happening at the stadium, you're better off
using [location search](https://www.instagram.com/developer/endpoints/locations/). So you
fire Google Maps, find out the exact coordinates of the stadium, and use location endpoints
to fetch fresh stuff. 

Then you implement the refresh algorithm, open the whole thing on a wide screen
Friday evening and sip a cold beer like every other sports fan does. Everything
is alright, and you are going to boast about it to your friends and neighbors, but... wait, what,
do... DOGS?

![Mean dogs](/assets/images/clojure-classifier/mean_dogs.jpg)

What the hell?! YOU HATE DOGS! Well, apparently some big dog lover lives near
the stadium and posts his photos with almost the same coordinates. It's not
that he even gets his dogs onto the stadium! 

Ok, you add a rule to remove his Instagram account from your show and try to
forget the whole thing. Well, dogs were mean to you in your childhood and you
hate them. Crisis averted!

...Eh, what? It seems that the guy has a whole network of friends who come to
his house and post dogs to Instagram! And these dog people seem to make social
connections at an enormous rate! What do they do, sniff at each other nose?!
It's a new Instagram account every day!

So, you can't ditch location-based search and can't refine it (Instagram API
allows to specify a range, but GPS devices don't give exact coordinates, so you
can't reduce this range down to, like, 1 meter). On the other hand, dogs must be filtered
out.

## Classification

What we came to is a classification task, similar to spam detection. In our
setting sports are "ham", and everything dog-related is "spam". If a cat fan
appears later, it will be "spam" too. How could we establish the difference to be able
to automatically assign a class to incoming posts?

First let's settle on what we are looking at. Classifying images by their contents
(i.e., pixels) is hard, but luckily most Instagram posts contain hashtags which we
can use as an input ("features" in machine learning speak).

The intuition here is that a post with following hashtags:

> `#doggie #Pluto #mylovelydog #Scotch #terrier`

will most certainly contain a dog image, while a post with

> `#stadium #sweat #game #soccer`

is expected to be sports-related.

Now, we are not going to create whitelists and blacklists of words. The machine
learning approach is different. Instead of coming out with rules, we prepare a
[training set](https://en.wikipedia.org/wiki/Test_set) which contains a number
of "posts" (in fact, we only need the hashtag portion, so a post can be trimmed
down to a set of hashtag strings) with a boolean flag attached to each one
(e.g., `true` marks post as ham, and `false` — as spam). Then we let allow a
classifier to "learn", i.e., somehow figure the difference. After learning is
done, the classifier, being presented a post with an unknown class, will tell us if
it's spam or ham. Depending on the approach used, sometimes the
probabilities (e.g., **spam 86%**, **ham 14%**) are also reported.

## Counting words

The task is going to be separated into two subtasks:

1. Single-word reasoning. I.e. what can we say about a post containing some hashtag, like `#soccer`?
2. Combining individual hashtag scores. If it's `#soccer #match`, how about that?

Let's start with the former. Basically, we'd want to calculate the probability of a post being spam
given that there's some `#hashtag` in it.

And here comes the Bayesian approach. There's a well-known
[Bayes' formula](https://en.wikipedia.org/wiki/Bayes%27_theorem) that can give
us exactly what we want:

$$
P(A \mid B) = \dfrac{P(B \mid A)\cdot P(A)}{P(B)}.
$$

Cryptic, huh? Let me describe what it means if you're new to the field. It says
something about probabilities. $$A$$ and $$B$$ are events. $$P(A)$$ is the probability
of event $$A$$ taking place, and $$P(B)$$ is related to $$B$$ in the same fashion.
$$P(A\mid B)$$ is a notion of
[conditional probability](https://en.wikipedia.org/wiki/Conditional_probability), i.e.,
the probability of event $$A$$ in case event $$B$$ takes place.

Let's use the formula in our setting:

$$
p = P(post~is~spam\mid hashtag) = \dfrac{P(hashtag\mid post~is~spam)\cdot P(post~is~spam)}{P(hashtag)}.
$$

Wow, we've just expressed an unknown probability in terms of three other unknown
probabilities! Sounds like progress!

But, in fact, these three can be calculated. What's
$$P(hashtag\mid post~is~spam)$$? Remember, we have a training set which consists of some spam and some
ham posts. What if we just go over all spam posts and count how many of them
contain our `#hashtag`? Let's denote the count as $$N_{spam}(hashtag)$$. Dividing by
$$N_{spam}$$, which is the total number of spam posts in the training set, brings us
the desired value:

$$
P(hashtag \mid post~is~spam) = \dfrac{N_{spam}(hashtag)}{N_{spam}}.
$$

Now let's attack $$P(hashtag)$$. Using [the formula of full probability](https://en.wikipedia.org/wiki/Law_of_total_probability), we can express it in this way:

$$
\begin{align*}
P(hashtag) =~ &P(hashtag \mid post~is~spam)\cdot P(post~is~spam) + \\
&P(hashtag \mid post~is~ham)\cdot P(post~is~ham).
\end{align*}
$$

Calculating conditional probability for ham in a similar fashion, we finally come to the following:

$$
p = \dfrac{ N_{spam}(hashtag)/N_{spam}\cdot P(post~is~spam) }
          { N_{spam}(hashtag)/N_{spam}\cdot P(post~is~spam) + N_{ham}(hashtag)/N_{ham}\cdot P(post~is~ham)}.
$$

Basically, only two unknown probabilities are left here: $$P(post~is~spam)$$ and
$$P(post~is~ham)$$. They are called *prior probabilities*, which means that they
are not related to our post in question, but rather reflect the total proportion
of spam and ham out there. And we better not fixate on any specific proportion
because it's really hard to predict. In particular, the spam/ham ratio in the training set
is a very bad estimate, because we don't know how it was collected. Therefore the
safe way to go would be to assume both priors to be equal to 0.5. The formula gets even simpler:

$$
p = \dfrac{ N_{spam}(hashtag)~/~N_{spam} }
          { N_{spam}(hashtag)~/~N_{spam} + N_{ham}(hashtag)~/~N_{ham}}.
$$

What we just did is estimating the probability of a post containing `#hashtag` to be spam.

## Some Gory Details about $$p$$

There's more to consider about $$p$$. First, what if `#hashtag` was never seen before?
The formula gives us $$0/0$$, which is clearly bad. In this case, we should resort to some predefined
probability, $$p_{unknown}$$. Per research, the best value here is again 0.5.

Another problem is that the formula for $$p$$ doesn't take the total number of hashtag occurrences into
consideration. Only proportion is used. Let's see why it might be important.

Let's imagine that some random hashtag, `#evening`, appeared in one of the spam
posts in our training set. Apparently, it's not relevant because you can do both
things in the evening: play sports (ham) and mess with dogs (spam). However, $$p$$ would be equal
to 1! Let's take another hashtag, `#dog`. Say it appeared 98 times in 100 posts. Here $$p=0.98$$. 
A random word beats a real spam marker, which occurred in most of the spam posts! The whole
system is way too sensitive to rare hashtags. We need a way to make it more robust.

Luckily, there's a solution by [Gary Robinson](http://www.garyrobinson.net/),
presented in his seminal post,
["Spam Detection"](http://radio-weblogs.com/0101454/stories/2002/09/16/spamDetection.html)
and later in LinuxJournal article
["A Statistical Approach to the Spam Problem"](http://www.linuxjournal.com/article/6467?page=0,0).
I won't go into much detail, but rather describe the logic in its essence.
Follow the links if you're interested.

So, basically, if a word is completely missing from our dataset, we assign the
spam probability to 0.5. If the word occurred once in the spam corpus and never
in the ham, the probability should be greater than 0.5, but not much. Because
it's, well, only one occurrence, might be random as discussed earlier. However,
if the word occurred 100 times in a spam corpus with size 100, this ought to be
serious and the probability better be close to 1.

This intuition is captured in the following formula suggested by Gary:

$$
f(w) = \dfrac{s\cdot p_{unknown} + N(w)\cdot p(w)}
             {s+N(w)},
$$

where:

* $$w$$ is the word (hashtag) in question;
* $$p(w)$$ is the same $$p$$ we already know how to calculate (see prev. formula);
* $$s$$ is the strength we want to give to our background information ($$s\le1$$);
* $$N(w)$$ is the number of posts in the training set that contain the word $$w$$ (both spam and ham).
$$N(w)=N_{spam}(w)+N_{ham}(w)$$.

$$f(w)$$ is called *the degree of belief*. It can easily be seen that with small $$N$$'s
$$f(w)$$ is close to $$p_{unknown}$$, while higher $$N$$ values result in $$f(w)$$ approaching $$p(w)$$.
This value, $$f(w)$$, is not a real probability, it's our best guess at it, better than $$p(w)$$ (which is,
by the way, an estimate too!). So we will be using $$f(w)$$ instead of $$p(w)$$ from now on.

## Combining probabilities

So far we were able to calculate the probability of a post being spam given some
hashtag (denoted by $$w$$). But let's remember that there might be multiple
hashtags in one post (we'll name them $$w_1, w_2, \ldots, w_n$$). Accordingly, there will be
$$n$$ probabilities: $$p_1, p_2, \ldots, p_n$$ ($$p_i=f(w_i)$$). What do we do about them? Like, add up, or multiply?

### The Naive Bayes Approach

This method to combine separate probabilities is based on the
premise that the occurrences of $$w_1, w_2, \ldots, w_n$$ are independent
events: words are related to each other. That is, having `#dog #terrier` in a post is a pure
coincidence. The `#terrier` hashtag does not increase the probability of seeing also a `#dog` hashtag.
As you can conclude, that's not exactly true in real-world situations. The classifier is called *naive*
precisely because of that.

Nevertheless, it's still a viable approach. It can be shown (by using the Bayes' formula, equating the
priors and doing some simple math) that the overall spam probability

$$
S = p(post~is~spam\mid w_1, \ldots, w_n)=\dfrac{p_1p_2\cdot\ldots p_n}
          {p_1p_2\cdot\ldots p_n + (1-p_1)(1-p_2)\cdot\ldots (1-p_n)}.
$$

This formula was used by [Paul Graham](http://www.paulgraham.com/) who
[pioneered the statistical approach](http://www.paulgraham.com/spam.html) in
the spam combat.

### The $$\chi^2$$ Approach

As Gary Robinson points out in his
[article](http://www.linuxjournal.com/article/6467?page=0,1), there's
[another approach to combine probabilities](https://en.wikipedia.org/wiki/Fisher%27s_method)
originating from the field of statistics known as meta-analysis. If we calculate
the following value,

$$
-2\sum_{i=1}^n \ln p_i,
$$

the result will have a
[$$\chi^2$$ distribution](https://en.wikipedia.org/wiki/Chi-squared_distribution)
with $$2n$$ degrees of freedom as shown by R. A. Fisher. Therefore we can use
a $$\chi^2$$ table to compute the probability of getting a result as extreme, or
more extreme, than the one obtained. This "combined" probability meaningfully
summarizes all the individual probabilities.

In fact, this Fisher probability is tricky (read the Gary's article with
explanations, especially about the null hypothesis and its verification). What
we can notice from the formula is that the probabilities are multiplied. If we
consider a ham post, most $$p_i$$'s will be close to zero, and the
product will be a very small value. Due to that, the classifier will be very
sensitive to "hammy" words. As for "spammy" words, they will be close to 1 and
thus make less impact on the final product. So the classifier will be less
sensitive to "spammy" words than it is to "hammy" words (which will result in
"false negatives", i.e., spam posts mistakenly considered as ham).

But there's a technique to deal with this bias towards ham. We can just repeat
the Fisher calculation with "reversed" probabilities ($$p'_i=1-p_i$$) and obtain
another indicator, which will be more spam-sensitive. Finally, both indicators
(let's call them $$H$$ and $$S$$) can be united into one:

$$
I = \dfrac{1+H-S}2.
$$

If $$I$$ is close to 0, the conclusion is that the post in question is ham. If $$I$$
is near 1, the conclusion is just the opposite. However, if both $$H$$ and $$S$$ are
near 0 (this might happen if a post contains both "spammy" and "hammy" hashtags), $$I$$ will
be around 0.5 (meaning "unsure"). This is a very handy feature making the $$I$$-based classifier
results much more robust.

## Conclusion

We've discussed some theory behind statistical classification methods.
Next part will show how to implement the $$f(w)$$ and $$\chi^2$$ calculations
in Clojure and build a classification microservice.

Stay tuned!
